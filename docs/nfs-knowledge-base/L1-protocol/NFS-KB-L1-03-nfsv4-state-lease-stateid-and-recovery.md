# NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复

> 文档状态：待验证  
> 知识阶段：L1 协议原理  
> 适用范围：Linux NFSv4.0、NFSv4.1、NFSv4.2 客户端与服务端；具体恢复、delegation、session 和锁行为以目标内核、`nfs-utils`、NAS/存储实现为准  
> 版本：1.1.0  
> 最后更新：2026-08-01  
> 前置文档：[NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](NFS-KB-L1-01-protocol-evolution-and-version-selection.md)、[NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)  
> 关联文档：待后续建立（见第 10 节）

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. NFSv4 状态总览](#2-nfsv4-状态总览)
- [3. clientid、lease 与伪文件系统](#3-clientidlease-与伪文件系统)
- [4. OPEN、stateid 与文件共享](#4-openstateid-与文件共享)
- [5. LOCK、delegation 与状态恢复](#5-lockdelegation-与状态恢复)
- [6. 服务端重启与网络分区](#6-服务端重启与网络分区)
- [7. 生产观测与验证](#7-生产观测与验证)
- [8. Java 工程视角](#8-java-工程视角)
- [9. 排障检查清单](#9-排障检查清单)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

NFSv4 将打开、锁、客户端身份和租约纳入协议状态。与 NFSv3 的独立文件操作相比，NFSv4 的一次 `READ` 或 `LOCK` 可能依赖之前建立的 clientid、session、stateid 和 lease。理解这些对象的生命周期，是定位以下生产现象的前提：

- 挂载仍显示存在，但读写长期阻塞；
- `NFS4ERR_BAD_STATEID`、`NFS4ERR_EXPIRED`、`NFS4ERR_STALE_CLIENTID`；
- 服务端重启后锁恢复慢、文件被拒绝访问；
- VIP/DNS 切换后客户端无法恢复状态；
- Java `FileLock` 在某些节点成功、另一些节点失败。

本文重点解释 NFSv4 状态对象之间的关系，不把 NFSv4 锁当作通用分布式协调服务，也不把客户端重新连上 TCP 误认为协议状态已经恢复。

## 2. NFSv4 状态总览

### 2.1 状态对象关系

```text
客户端实现
  |
  | EXCHANGE_ID（v4.1+）/ SETCLIENTID（v4.0）
  v
clientid
  |
  | lease：客户端需要持续刷新状态的时间窗口
  v
OPEN
  |
  | open stateid：表示某个 open owner 对文件的打开状态
  v
LOCK
  |
  | lock stateid：表示 lock owner 在文件上的锁状态
  v
READ/WRITE/LOCKU/CLOSE
  携带相应 stateid，服务端据此校验状态
```

NFSv4.1/v4.2 还会在 clientid 之上建立 session，使用 slot 和 `SEQUENCE` 管理请求序列；NFSv4.0 没有 v4.1 的 session/slot 模型，但仍有 clientid、lease、open stateid 和 lock stateid。

### 2.2 “有状态”不等于每个请求都必须 OPEN

NFSv4 的状态依赖是分层的：

| 操作 | 典型状态依赖 | 说明 |
| --- | --- | --- |
| `GETATTR`/`LOOKUP` | 文件句柄、权限和命名空间 | 通常不需要 open stateid |
| `OPEN` | clientid、open owner、文件句柄 | 创建或更新 open state |
| `READ`/`WRITE` | 文件句柄和适用的 stateid | 必须携带 stateid；可能来自 OPEN/LOCK，也可能是协议定义的 special stateid |
| `LOCK` | clientid、open/lock owner、open stateid、lock stateid | 需要锁状态一致 |
| `LOCKU` | lock stateid、锁范围 | 释放指定锁范围 |
| `CLOSE` | open stateid、seqid | 关闭 open state |
| `RENEW`（v4.0） | clientid | v4.0 显式刷新租约 |
| `SEQUENCE`（v4.1+） | session、slot、sequence ID | session-bound 请求通常借此维持租约并管理请求序列 |

具体是否使用 delegation、special stateid 或 compound 中隐式状态，取决于 NFS 版本和客户端实现。分析抓包时必须把当前操作与此前状态建立过程关联。

## 3. clientid、lease 与伪文件系统

### 3.1 NFSv4 伪文件系统

NFSv4 使用服务端伪文件系统向客户端呈现统一命名空间：

```text
server:/                 <- NFSv4 伪根，通常对应 fsid=0
├── export-a/
├── export-b/
└── projects/
```

客户端的 `server:/path` 是在伪根命名空间中解析的路径，不一定等于服务端本地目录的绝对路径。`fsid=0`、导出层级、`crossmnt`、`nohide` 和服务端的伪根实现决定了客户端可见范围。

**执行端：服务端**  
**适用范围：Linux NFS 服务端；生产变更前先在测试导出验证**

```bash
sudo exportfs -v
sudo findmnt -T /export/app
sudo cat /etc/exports
```

`exportfs -v` 展示当前生效的导出参数；`findmnt -T /export/app` 用于确认导出路径所在的本地文件系统。两者都不能只替代对伪根层级、`fsid=0` 和客户端实际命名空间的验证，不能只读取 `/etc/exports` 推断服务端已经加载的伪根和安全选项。

### 3.2 clientid 的建立

NFSv4.0 使用 `SETCLIENTID`/`SETCLIENTID_CONFIRM` 建立或确认客户端身份；NFSv4.1 及后续 minor version 使用 `EXCHANGE_ID`，随后可通过 `CREATE_SESSION` 建立 session。

```text
NFSv4.0：SETCLIENTID -> SETCLIENTID_CONFIRM -> OPEN/LOCK
NFSv4.1：EXCHANGE_ID -> CREATE_SESSION -> SEQUENCE + OPEN/LOCK
```

clientid 不是 Java 进程 ID，也不是 TCP 连接 ID。客户端重启、客户端标识变化、主机克隆导致的身份重复，可能使服务端拒绝旧状态或触发恢复流程。虚拟机模板、容器网络和节点克隆必须避免复用会影响 NFS 身份恢复的持久化标识。

### 3.3 lease 与 RENEW

服务端为客户端状态维护 lease。NFSv4.0 客户端通过有效状态请求或 `RENEW` 刷新租约；NFSv4.1/v4.2 通常通过包含 `SEQUENCE` 的 session-bound 请求维持租约，不应把 `RENEW` 当作 v4.1+ 的常规刷新流程。在租约窗口内，服务端保留 OPEN/LOCK 等状态。

```text
clientid 建立
  -> OPEN/LOCK 创建状态
  -> v4.0：有效状态请求或 RENEW
  -> v4.1+：SEQUENCE/session-bound 请求活动
  -> 网络中断超过租约/恢复窗口
  -> 服务端可能释放状态
  -> 客户端收到 EXPIRED/BAD_STATEID 等错误
```

租约时间不是应用超时，也不是 TCP keepalive。调大 Java HTTP 超时不会延长 NFS lease；反之，TCP 连接保持也不等于 NFS 状态仍然有效。

## 4. OPEN、stateid 与文件共享

### 4.1 OPEN 的参与者

一个 NFSv4 `OPEN` 至少涉及：

| 对象 | 作用 |
| --- | --- |
| clientid | 标识 NFS 客户端身份 |
| open owner | 标识客户端内某个打开者；用于 OPEN 序列和状态归属 |
| filehandle | 标识目标文件或目录 |
| share access | 读、写或读写访问意图 |
| share deny | 对其他打开者施加的拒绝模式 |
| open stateid | 服务端返回的打开状态凭据 |

`OPEN` 的 share access/deny 不是 POSIX 模式位，也不等同于 Java `FileChannel` 的所有共享语义。服务端可能根据 NFSv4 share reservation 拒绝冲突打开，返回 `NFS4ERR_SHARE_DENIED`。

### 4.2 stateid 的使用

stateid 是服务端发给客户端的状态凭据，通常用于证明某个 `READ`、`WRITE`、锁操作或布局操作属于已建立状态。它不是永久对象，可能因 `CLOSE`、租约过期、服务端恢复或客户端状态重建而失效。

```text
OPEN -> open stateid
LOCK -> lock stateid
READ/WRITE -> 携带适用 stateid
LOCKU/CLOSE -> 消费或更新对应状态
```

收到 `NFS4ERR_BAD_STATEID` 时，不能简单无限重试原 RPC。应确认客户端是否已经执行恢复、服务端是否重启、对应 stateid 是否属于当前 clientid，以及是否需要重新 OPEN/LOCK。

### 4.3 OPEN 的 create mode 与原子创建

NFSv4 `OPEN` 可以同时表达打开和创建，支持 `OPEN4_CREATE` 以及 `GUARDED4`、`UNCHECKED4`、`EXCLUSIVE4` 等创建语义。与 NFSv3 的 `CREATE` 不同，v4 的 OPEN 还会建立 open state。

```text
COMPOUND
  PUTFH(parent)
  OPEN(CLAIM_NULL, create mode, share access/deny)
  GETFH
  GETATTR
```

创建响应丢失后，客户端必须依赖 open owner、seqid、create verifier 和服务端重复请求处理，避免把一次业务创建误判为失败并重复创建。

## 5. LOCK、delegation 与状态恢复

### 5.1 LOCK 与 lock owner

NFSv4 锁由协议内建的 `LOCK`、`LOCKT`、`LOCKU` 操作承载。锁状态通常关联：

```text
clientid
  -> open owner
  -> open stateid
  -> lock owner
  -> lock stateid + byte range
```

锁操作需要维护 seqid 和状态恢复。客户端崩溃、服务端重启、网络分区或进程更换 lock owner 后，原 lock stateid 可能不再有效。

### 5.2 delegation

服务端可将某些文件的读或写 delegation 授予客户端，让客户端在 delegation 有效期间减少与服务端交互。delegation 不是永久缓存许可：

- 服务端可通过回收请求要求客户端归还 delegation；
- 其他客户端访问可能触发 recall；
- 客户端失联时服务端需要等待恢复窗口或收回状态；
- delegation 与属性/数据缓存、锁和 lease 共同影响可见性。

生产排障不能看到客户端 RPC 变少就断定网络或服务端正常，可能是 delegation 命中；也不能把 delegation 当作跨客户端强一致保证。

### 5.3 服务端 grace period

服务端重启后通常进入 grace period，用于允许旧客户端 reclaim OPEN/LOCK 等状态。它是服务端恢复阶段，不是客户端的普通业务超时。

```text
服务端重启
  -> 服务端进入 grace period
  -> 客户端重建 clientid/session
  -> reclaim OPEN/LOCK
  -> v4.1+ 客户端发送 RECLAIM_COMPLETE
  -> 服务端结束 grace period
  -> 新状态请求恢复正常
```

如果客户端错过恢复窗口，服务端可能不再接受 reclaim；此时客户端需要重新建立状态，应用层可能看到锁丢失、I/O 错误或状态异常。NFSv4.1/v4.2 客户端完成允许的状态回收后，需要通过 `RECLAIM_COMPLETE` 告知服务端该客户端的 reclaim 阶段结束；该操作不等于服务端全局 grace period 立即结束。具体 grace 时间和日志字段以实现为准。

## 6. 服务端重启与网络分区

### 6.1 客户端 recovery 状态机

```text
正常
  -> 发现 RPC 超时/连接断开
  -> 客户端重连并重试
  -> 检测 lease/session/stateid 状态
  -> 执行 clientid/session recovery
  -> reclaim OPEN/LOCK/delegation
  -> v4.1+ 发送 RECLAIM_COMPLETE
  -> recovery 完成
  -> 返回正常 I/O
```

hard 挂载可能让应用线程在 recovery 完成前持续等待。soft 选项可能更早返回错误，但会增加部分操作失败和数据一致性风险。恢复期间不要只重启 Java 进程；重启应用可能制造新的 open owner 或重复业务写入，反而增加状态和幂等复杂度。

### 6.2 常见协议错误

| 错误 | 典型含义 | 优先检查 |
| --- | --- | --- |
| `NFS4ERR_EXPIRED` | clientid/lease 已过期 | 服务端重启、网络分区、客户端恢复时间 |
| `NFS4ERR_STALE_CLIENTID` | clientid 不再被服务端接受 | 客户端身份、服务端状态库、恢复流程 |
| `NFS4ERR_BAD_STATEID` | stateid 不匹配或已失效 | OPEN/LOCK 恢复、clientid、服务端重启 |
| `NFS4ERR_OLD_STATEID` | 使用了旧序列或旧状态 | 重试顺序、并发恢复、session/slot |
| `NFS4ERR_STALE_STATEID` | stateid 所属状态已失效 | lease、CLOSE、状态恢复 |
| `NFS4ERR_GRACE` | 服务端仍处于恢复宽限期 | 服务端 grace 日志、reclaim 时序 |
| `NFS4ERR_DELAY` | 服务端暂时不能完成请求 | recovery、资源压力、实现重试策略 |
| `NFS4ERR_SHARE_DENIED` | OPEN 的 share reservation 冲突 | 其他客户端打开模式、share deny |

日志中的 `ESTALE` 仍可能指 filehandle 失效，不应与 `BAD_STATEID` 混为一谈；前者偏向对象句柄/命名空间，后者偏向协议状态凭据。

## 7. 生产观测与验证

### 7.1 固化实际挂载与服务端状态

**执行端：客户端**  
**适用范围：Linux NFSv4 客户端；授权诊断窗口**

```bash
NFS_MOUNT=/mnt/nfs4-test
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
cat /proc/self/mountstats
mountpoint "$NFS_MOUNT"
```

**执行端：服务端**  
**适用范围：Linux NFS 服务端；需 root 或等价权限**

```bash
sudo exportfs -v
sudo nfsstat -s
sudo journalctl -k --since "10 min ago" | grep -Ei 'nfs|nfsd|stateid|lease|grace|lock'
```

### 7.2 抓包关注点

**执行端：客户端或旁路抓包节点**  
**适用范围：测试环境或已授权短时抓包；完整 payload 可能包含业务数据**

```bash
NFS_SERVER_IP=192.0.2.10
PCAP_FILE=/tmp/nfs4-state-$(date +%s).pcap
timeout 30 tcpdump -i any -nn -s 0 -w "$PCAP_FILE" "host ${NFS_SERVER_IP} and port 2049"
```

抓包分析应按 NFSv4 `COMPOUND` 操作序列确认：

- `SETCLIENTID`/`SETCLIENTID_CONFIRM` 或 `EXCHANGE_ID`；
- `CREATE_SESSION`、`SEQUENCE`（v4.1+）；
- `PUTFH`、`OPEN`、`CLOSE`、`LOCK`、`LOCKU`；v4.0 的 `RENEW`；
- `RECLAIM_COMPLETE`（v4.1+ 状态回收完成）；
- `CB_RECALL` 等 callback（启用 delegation 时）；
- 响应中的 NFS 状态码、stateid、clientid 和 sequence 关系。

### 7.3 最小状态实验

**执行端：客户端 1 与客户端 2**  
**适用范围：独立测试导出；不要在生产挂载上执行重启和锁争用实验**

先准备一个最小 Java 锁程序，两个客户端使用同一份源码即可：

```java
import java.nio.channels.FileChannel;
import java.nio.channels.FileLock;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;

public class HoldLock {
    public static void main(String[] args) throws Exception {
        Path path = Path.of(args[0]);
        long seconds = Long.parseLong(args[1]);
        try (FileChannel channel = FileChannel.open(path,
                StandardOpenOption.CREATE,
                StandardOpenOption.WRITE)) {
            System.out.printf("trying lock path=%s%n", path);
            try (FileLock lock = channel.lock(0L, 1L, false)) {
                System.out.printf("locked path=%s valid=%s%n", path, lock.isValid());
                Thread.sleep(seconds * 1000L);
                System.out.printf("unlocking path=%s valid=%s%n", path, lock.isValid());
            }
        }
    }
}
```

客户端 1 创建测试文件并持有 120 秒锁：

```bash
TEST_DIR=/mnt/nfs4-test
TEST_FILE="$TEST_DIR/.nfs-kb-l1-03-lock-test"
printf 'nfs4-state-test\n' > "$TEST_FILE"
javac HoldLock.java
java HoldLock "$TEST_FILE" 120
```

客户端 2 在客户端 1 持锁期间尝试获取同一字节范围的排他锁：

```bash
TEST_FILE=/mnt/nfs4-test/.nfs-kb-l1-03-lock-test
javac HoldLock.java
date -Is
timeout 30 java HoldLock "$TEST_FILE" 10
date -Is
```

预期观察：客户端 2 在客户端 1 持锁期间阻塞，或在 `timeout` 到期后被终止；客户端 1 释放锁后，客户端 2 应能获得锁。若在隔离环境中测试服务端重启或网络中断，应在客户端 1 持锁期间执行故障注入，同时记录服务端 grace、客户端 recovery/reclaim、v4.1+ `RECLAIM_COMPLETE`、Java 输出、线程状态和错误码。

测试结束后清理：

```bash
TEST_FILE=/mnt/nfs4-test/.nfs-kb-l1-03-lock-test
rm -f "$TEST_FILE" HoldLock.class
```

本实验需要保存客户端/服务端版本、挂载选项、锁范围、clientid/session 日志、服务端 grace 时间、恢复耗时和清理结果。当前工作区没有 Linux NFSv4 环境，本节仅为待执行验证方案。

## 8. Java 工程视角

### 8.1 FileLock 与 NFSv4 lock state

`FileChannel.lock()` 在 NFSv4 上通常映射到协议 `LOCK`/`LOCKU`，但 Java API 不暴露 clientid、open owner、lock owner、stateid、lease 和 reclaim 状态。网络分区时，Java 线程可能长期阻塞；恢复后原锁可能被重新声明、失效或返回异常。

应用不能仅依据 `FileLock` 对象存在来判断锁永远有效。关键业务需要记录锁持有者、租约/心跳、fencing 和恢复后的重新校验。

### 8.2 Java 文件替换与 OPEN 状态

`Files.move`/rename 可能与其他客户端的 OPEN、delegation、属性缓存和读取并发交互。推荐使用临时文件、完整写入、`force(true)`、校验和和同目录 rename，并在 NFSv4 目标环境验证 `ATOMIC_MOVE`、目标替换和跨客户端可见性。

### 8.3 线程池、超时与恢复

Java `Future`、CompletableFuture 或 HTTP 超时不能保证取消内核中的 NFS recovery。应用应隔离 NFS I/O 线程池，并在服务端恢复时避免无限创建新任务。健康检查不应只读取 NFS 路径，否则共享存储故障可能把探针线程也拖入 hard I/O 等待。

## 9. 排障检查清单

遇到 NFSv4 锁异常、I/O 卡顿或状态错误时：

1. 固化客户端/服务端 OS、内核、`nfs-utils`、NFS minor version、挂载参数和服务端地址。
2. 用 `findmnt`、`nfsstat -m` 确认实际版本、`hard/soft`、`timeo/retrans`、`nconnect`、`sec` 和缓存参数。
3. 区分 filehandle 错误（`ESTALE`）与 stateid/clientid 错误（`BAD_STATEID`、`EXPIRED`、`STALE_CLIENTID`）。
4. 检查服务端是否重启、是否处于 grace period、客户端是否完成 recovery/reclaim。
5. NFSv4.1/v4.2 检查 `EXCHANGE_ID`、`CREATE_SESSION`、`SEQUENCE`、slot 和 trunking；不要用 v4.0 的判断方式替代。
6. 锁问题检查 open owner、lock owner、share deny、锁范围、delegation recall 和 fencing。
7. 用抓包、内核日志和 NFS 操作统计关联错误发生的 COMPOUND 操作，而不是只看 Java 异常。
8. 变更服务端导出、VIP/DNS、状态目录或挂载参数前，保留证据、恢复窗口和回滚方案。

常见现象映射：

| 现象 | 优先检查 | 不应直接得出的结论 |
| --- | --- | --- |
| TCP 已连接但读写卡住 | lease/session/state recovery、服务端 grace、nfsd/存储队列 | TCP connected 不代表 NFS state 有效 |
| `BAD_STATEID` | OPEN/LOCK 状态恢复、clientid、stateid 生命周期 | 无限重试原 RPC 可以修复 |
| `EXPIRED`/`STALE_CLIENTID` | 租约、客户端身份、服务端状态库和恢复时序 | 只重启 Java 进程即可恢复 |
| `SHARE_DENIED` | 其他客户端 OPEN 的 share access/deny | 直接修改 POSIX mode 位即可解决 |
| 锁在一端存在、另一端不存在 | grace/reclaim、delegation、fencing、网络分区 | Java `FileLock` 对象就是全局真相 |
| RPC 数量减少但延迟正常 | delegation、缓存命中、应用负载下降 | NFS 服务端没有执行任何状态管理 |

## 10. 参考资料与关联文档

### 参考资料

- RFC 7530：Network File System (NFS) Version 4 Protocol
- RFC 5661：Network File System (NFS) Version 4 Minor Version 1 Protocol
- RFC 8881：Network File System (NFS) Version 4 Minor Version 1 Protocol（更新版）
- `man 5 nfs`、`man 5 exports`、`man 8 nfsstat`、`man 8 mount.nfs`
- Linux 内核文档：NFS client recovery、state management、SUNRPC 和 VFS

### 关联文档

- [NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](NFS-KB-L1-01-protocol-evolution-and-version-selection.md)
- [NFS-KB-L1-02 NFSv3 协议流程与状态边界](NFS-KB-L1-02-nfsv3-protocol-and-state-boundaries.md)
- [NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)
- 待建立：`NFS-KB-L2-01` 服务端导出与客户端挂载基线
- 待建立：`NFS-KB-L5-01` NFS 故障树与证据链排障方法

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-01 | 1.1.0 | 修正 v4.0/v4.1 租约刷新边界、补充 RECLAIM_COMPLETE、精确 READ/WRITE stateid 语义并补齐 Java 锁实验 | 基于文档审查结果修订 |
| 2026-08-01 | 1.0.0 | 初始发布 | 建立 NFSv4 状态、租约、stateid 与恢复模型 |
