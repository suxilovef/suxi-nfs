# NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期

> 文档状态：待验证  
> 知识阶段：L0 基础设施层  
> 适用范围：Linux NFS 客户端与服务端；NFSv3、NFSv4.0、NFSv4.1；Java 11+。具体行为以目标内核、`nfs-utils` 和实现版本为准  
> 版本：1.1.0  
> 最后更新：2026-07-31  
> 前置文档：[NFS-KB-L0-01-end-to-end-io](NFS-KB-L0-01-end-to-end-io.md)  
> 关联文档：待后续建立（见第 10 节）

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. SUNRPC 总体模型](#2-sunrpc-总体模型)
- [3. XDR 数据表示](#3-xdr-数据表示)
- [4. 一次 RPC 请求的生命周期](#4-一次-rpc-请求的生命周期)
- [5. 重试、重复请求与状态](#5-重试重复请求与状态)
- [6. 生产观测与验证](#6-生产观测与验证)
- [7. Java 工程视角](#7-java-工程视角)
- [8. 生产排障检查清单](#8-生产排障检查清单)
- [9. 小结](#9-小结)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

NFS 不是直接把文件操作塞进 TCP。它依赖 ONC RPC（Open Network Computing Remote Procedure Call）完成远程过程调用，依赖 XDR（External Data Representation）把结构化参数转换为跨平台字节流。

本单元建立从 NFS 操作到网络字节流的模型，回答以下问题：

1. 一个 NFS 请求如何识别“哪个程序、哪个版本、哪个过程”。
2. 为什么抓包中会出现 `XID`、`CALL`、`REPLY`、`AUTH_SYS` 和 NFS 状态码。
3. TCP 上的 RPC record marking 如何划分消息，XDR 如何处理整数、字符串和 padding。
4. 网络重传时，客户端如何匹配响应、处理重复请求和避免重复写入。
5. NFSv3 的普通 RPC 与 NFSv4 `COMPOUND`、NFSv4.1 session/slot 有什么关系。
6. 如何用 `rpcinfo`、`nfsstat`、`nfsiostat` 和抓包区分网络层、RPC 层和 NFS 层故障。

本文描述 Linux SUNRPC/NFS 的通用实现模型，不替代目标发行版的内核文档。涉及协议事实时以 RFC 和实现文档为准，涉及性能结论时必须用目标环境基线验证。

## 2. SUNRPC 总体模型

### 2.1 RPC 标识元组

一次 ONC RPC 调用由以下标识共同确定：

| 字段 | 典型 NFS 值 | 作用 |
| --- | --- | --- |
| Program | `100003` | 标识 NFS 服务程序 |
| Version | `3`、`4` | 标识协议版本 |
| Procedure | `0`、`1` 等 | 标识具体过程；NFSv3 使用独立过程编号 |
| XID | 每次调用的事务 ID | 将响应匹配回发起请求 |
| Transport | TCP/2049，NFSv3 也可能 UDP | 承载 RPC 字节流或数据报 |
| Auth flavor | `AUTH_SYS`、`RPCSEC_GSS` 等 | 表示凭据与验证方式 |

Linux 常见 RPC 程序编号：

| 程序 | 编号 | 主要用途 |
| --- | ---: | --- |
| NFS | `100003` | 文件操作 |
| mountd | `100005` | 主要服务 NFSv3 挂载请求 |
| portmapper/rpcbind | `100000` | 查询 RPC 程序与端口 |
| NLM | `100021` | NFSv3 文件锁 |
| NSM/statd | `100024` | NFSv3 锁状态通知与恢复辅助 |

NFSv4 通常通过 TCP/2049 直接访问 NFS 服务程序，不依赖 NFSv3 那种先查询 mountd 端口的挂载流程；具体是否启用 rpcbind 辅助服务仍取决于发行版和其他 RPC 服务。

### 2.2 CALL 与 REPLY

RPC 消息分为请求 `CALL` 和响应 `REPLY`：

```text
客户端                                      服务端
  |                                           |
  | CALL: XID, program, version, procedure   |
  |------------------------------------------>|
  |                                           | 解码、认证、分发到 nfsd
  |                                           | 执行 NFS 操作和服务端 VFS
  | REPLY: XID, accept/reject, result         |
  |<------------------------------------------|
  | 用 XID 匹配等待中的 RPC 请求               |
```

`XID` 只用于匹配 RPC 消息，不是文件句柄、锁 ID，也不是 NFSv4 的 `stateid`。同一个 TCP 连接中可同时存在多个未完成请求，客户端依靠 XID 和本地 RPC 状态表区分它们。

### 2.3 NFSv3 与 NFSv4 的过程模型

NFSv3 把 `GETATTR`、`LOOKUP`、`READ`、`WRITE` 等定义为独立的 RPC procedure。一次应用调用可能触发多个 NFSv3 RPC。

NFSv4 在 RPC 层通常只有 NFS 程序的通用入口，具体操作放在 `COMPOUND` 数组中：

```text
RPC CALL
  -> NFSv4 COMPOUND
       [PUTFH]
       [LOOKUP "orders"]
       [OPEN "order-001.json"]
       [GETATTR]
  -> COMPOUND REPLY
```

这使客户端能在一次 RPC 中组合多个操作，但每个操作仍有自己的 NFS 语义和失败状态。`COMPOUND` 中某个操作失败时，后续操作通常不会继续执行，排障时要确认失败发生在哪个操作。

### 2.4 RPC 认证不是必然加密

RPC 请求包含 credential 和 verifier：

| Flavor | 典型用途 | 机密性/完整性 |
| --- | --- | --- |
| `AUTH_NONE` | 无身份凭据 | 无 |
| `AUTH_SYS`（AUTH_UNIX） | UID/GID 与组列表 | 不提供加密或密码学完整性 |
| `RPCSEC_GSS` `krb5` | Kerberos 身份认证 | 认证，通常不加密业务 payload |
| `RPCSEC_GSS` `krb5i` | Kerberos 完整性 | 提供完整性校验，增加 CPU/网络开销 |
| `RPCSEC_GSS` `krb5p` | Kerberos 隐私 | 提供加密，延迟和 CPU 开销更高 |

抓包中看见 `AUTH_SYS` 不代表网络安全；生产环境的身份、完整性和保密需求应在 `sec=`、Kerberos、网络隔离和密钥管理层面共同设计。

## 3. XDR 数据表示

### 3.1 基本规则

XDR 的目标是让不同 CPU 架构、字节序和编程语言使用同一线格式。常见规则如下：

- 32 位整数使用大端（network byte order）表示。
- 64 位整数按两个 32 位大端字表示。
- 可变长度 opaque/string/array 前置长度，再跟数据内容。
- 数据按 4 字节边界补零；padding 不属于有效业务数据。
- XDR 结构按字段声明顺序编码，没有 JSON 一样的字段名。

示意：

```text
XDR string "abc"
  00 00 00 03       # 长度 3
  61 62 63 00       # 数据 abc + 1 字节 padding
```

因此抓包中看到的 `00 00 00 03` 可能是字符串长度或数组长度，不能脱离具体 RPC procedure 的 XDR 定义直接解释。

### 3.2 NFSv3 READ/WRITE 的结构化参数

NFSv3 `READ` 请求包含文件句柄、offset 和 count；`WRITE` 请求包含文件句柄、offset、count、稳定性标志和数据。抽象结构如下：

```text
READ3args
  file   : opaque filehandle
  offset : uint64
  count  : uint32

WRITE3args
  file       : opaque filehandle
  offset     : uint64
  count      : uint32
  stable     : enum (UNSTABLE/DATA_SYNC/FILE_SYNC)
  data       : opaque<>  # 长度 + 数据 + padding
```

真实编码还包含 RPC header、认证字段和 procedure 外层结构；上图用于理解字段关系，不应直接当作完整线格式。

### 3.3 NFSv4 COMPOUND 的 XDR 结构

NFSv4 请求通常包含 tag、minor version、操作数量和操作数组：

```text
COMPOUND4args
  tag          : utf8str_cs
  minorversion : uint32
  argarray<>   : nfs_argop4
```

每个 `nfs_argop4` 以操作号开头，后续参数结构由操作号决定。响应类似地包含整体状态、tag 和结果数组。分析 v4 抓包必须先识别 `COMPOUND` 中的操作序列，再解释对应的 XDR 字段。

### 3.4 TCP record marking

RPC over TCP 不是简单地把一条消息写入一次 TCP `send()`。每个 RPC record 由一个或多个 fragment 组成，每个 fragment 前有 4 字节 record marker：

```text
31                    0
+---------------------+
| last-fragment | len |
+---------------------+
```

最高位表示是否为最后一个 fragment，低 31 位表示 fragment 长度。TCP 可能拆分或合并任意字节，不能把一次 `recv()`、一个 TCP segment 或一个 Wireshark packet 当作一条 RPC 消息。

UDP 使用数据报边界，不使用 TCP record marking；但 UDP 丢包、重排和大小限制会改变 RPC 重试与性能特征。

## 4. 一次 RPC 请求的生命周期

### 4.1 从 Java 调用到网络字节流

一次普通 NFS 文件操作可以抽象为：

```text
Java Files.read/write
  -> Linux 系统调用
  -> NFS 客户端决定 NFS procedure
  -> 组装文件句柄、offset、count 等参数
  -> XDR 编码
  -> 分配 XID，填充 RPC CALL 和认证字段
  -> TCP record marking（TCP）或数据报（UDP）
  -> 网络传输
  -> 服务端 SUNRPC 接收与重组
  -> RPC 认证与程序/版本/过程分发
  -> nfsd 执行 NFS 操作和服务端 VFS
  -> XDR 编码 RPC REPLY
  -> 客户端按 XID 匹配等待请求
  -> NFS 客户端更新缓存、状态并返回系统调用
```

RPC 层只负责传输和过程调用框架，文件句柄是否合法、权限是否允许、offset 是否越界、文件是否存在等由 NFS 层返回状态决定。

### 4.2 服务端的分层结果

一个失败可能在多个层次发生，必须区分：

| 层次 | 典型结果 | 说明 |
| --- | --- | --- |
| TCP/UDP | 连接重置、超时、丢包 | RPC 可能根本没有完整到达服务端 |
| RPC 消息 | `MSG_ACCEPTED`、`MSG_DENIED` | 程序版本、认证或 RPC 语法层结果 |
| RPC 认证 | `AUTH_ERROR`、GSS 错误 | 凭据无效、过期、服务主体或时间同步问题 |
| NFS | `NFS3ERR_NOENT`、`NFS4ERR_PERM`、`NFS4ERR_STALE` 等 | 文件系统、权限、句柄和协议状态结果 |
| 本地 VFS/存储 | `EIO`、I/O error、文件系统日志错误 | 服务端本地文件系统或设备故障 |

例如，服务端返回 `NFS4ERR_PERM` 不是网络超时；客户端重试不能解决权限问题。反过来，TCP 连接建立成功也不能证明 NFS procedure 已经被服务端执行。

### 4.3 NFSv4.1 session 与 slot

NFSv4.1 在客户端和服务端之间建立 session。session 建立过程包含：

```text
EXCHANGE_ID
  -> CREATE_SESSION
```

建立 session 后，每个 session-bound 请求通常以 `SEQUENCE` 作为 `COMPOUND` 的第一个操作：

```text
SEQUENCE（slot ID、sequence ID）
  -> 业务操作（READ/WRITE/OPEN/...）
  -> SEQUENCE 结果与业务结果
```

session 为客户端提供 slot。每个 slot 维护 sequence ID，限制并发未完成请求数量，并帮助服务端识别重复请求和恢复状态。slot 数量不足时，应用吞吐可能受客户端或服务端 session 并发限制影响；盲目增加 Java 线程不一定提高 NFS 吞吐。

NFSv4.0 没有 v4.1 session/slot 这一完整机制，但仍有 clientid、lease、stateid、OPEN 和 LOCK 等状态。不能把 v4.1 的 slot 参数直接套用到 v3/v4.0。

## 5. 重试、重复请求与状态

### 5.1 超时与重传

客户端在发送 RPC 后维护等待状态和计时器。出现丢包、服务端处理慢、连接断开或网络分区时，可能发生：

```text
发送 CALL
  -> 等待 REPLY
  -> 超时或 TCP 连接故障
  -> 重传原请求，或重建连接后重新发送
  -> 服务端收到重复请求
  -> 返回缓存结果或再次执行
```

`hard` 挂载通常持续重试并让系统调用保持等待；`soft` 族选项可能向应用返回错误。重试上限、超时时间和错误返回必须与应用超时、线程池隔离和数据重试策略统一设计。

### 5.2 重复请求与操作语义

网络无法让客户端绝对确定“请求未到达”还是“服务端已执行但响应丢失”。因此重试可能遇到重复请求：

- `GETATTR`、`LOOKUP` 等读/查询操作通常可安全重试。
- `WRITE`、`CREATE`、`RENAME` 等操作需要依赖 NFS 协议的稳定性、请求缓存、文件句柄和客户端恢复机制。
- 服务端可能识别重复 RPC 并返回缓存结果，也可能在特定故障窗口重新处理。
- NFS 重试不等于应用层 exactly-once；业务仍需设计幂等键、临时文件和校验机制。

不要把“启用 hard”理解为“自动保证不重复写入”。hard 主要改变故障时的返回/等待语义，不能替代业务幂等设计。

### 5.3 状态恢复与租约

NFSv4 客户端重启、服务端重启或网络分区恢复后，可能触发不同的恢复流程。服务端重启后通常进入 `grace period`，客户端重启或网络分区恢复则主要表现为客户端重新建立身份、租约和状态：

```text
故障发生
  -> clientid/session/stateid 可能失效
  -> 服务端重启：服务端进入 grace period
  -> 客户端重启/网络恢复：客户端进入 recovery
  -> 客户端重新建立身份或 session
  -> OPEN/LOCK/DELEGATION 状态恢复
  -> 服务端结束 grace period
  -> 未能恢复的句柄或锁返回错误
```

`ESTALE`、`NFS4ERR_EXPIRED`、`NFS4ERR_BAD_STATEID` 和锁恢复失败需要结合服务端重启、导出变更、客户端标识和时间线分析，不能只看应用异常堆栈。

## 6. 生产观测与验证

### 6.1 识别 RPC 程序与端口

**执行端：客户端或管理节点**  
**适用范围：主要用于 NFSv3/rpcbind 场景；NFSv4 未在 rpcbind 注册不代表服务不可用**

```bash
NFS_SERVER_IP=192.0.2.10
rpcinfo -p "$NFS_SERVER_IP"
rpcinfo -T tcp "$NFS_SERVER_IP" nfs 3
```

`rpcinfo -p` 查询的是 rpcbind 注册表。NFSv3 通常能看到 mountd、nfs、nlockmgr、status 等程序；NFSv4 常直接访问 TCP/2049，不能因为 `rpcinfo -p` 结果不完整就判定 NFSv4 不可用。

### 6.2 客户端 RPC/NFS 基线

**执行端：客户端**  
**适用范围：Linux NFS 客户端**

```bash
NFS_SERVER_IP=192.0.2.10
nfsstat -m
nfsstat -c
nfsiostat 1 10
cat /proc/self/mountstats
ss -tin "dst ${NFS_SERVER_IP}:2049"
```

观察：

- RPC retrans、超时和错误是否增长；
- READ/WRITE/GETATTR/LOOKUP 的调用量、RTT、执行时间和队列时间；
- TCP 重传、RTT、拥塞窗口和连接状态；
- NFSv4.1 session/slot 相关异常是否与并发上升同时出现。`nfsiostat` 只能提供通用 NFS/RPC 指标，不能直接显示 slot 使用率或 sequence ID；这类问题需要结合抓包、内核日志、`rpcdebug`、tracepoint/eBPF 或服务端实现特定指标确认。

### 6.3 抓包与内核调试

**执行端：客户端或旁路抓包节点**  
**适用范围：已获授权的短时诊断；完整 payload 可能包含业务数据**

```bash
NFS_SERVER_IP=192.0.2.10
timeout 30 tcpdump -i any -nn -s 0 -w /tmp/sunrpc-nfs.pcap "host ${NFS_SERVER_IP} and port 2049"
```

抓包时应按时间线关联 TCP stream、RPC XID、NFS procedure 和应用请求。不要把一个 TCP segment 当成一个 RPC；应使用支持 ONC RPC/NFS 解码的工具按 record/message 解析。

必要时可短时打开内核调试：

```bash
sudo rpcdebug -m nfs -s all
# 采集完成后必须关闭，避免日志和性能影响
sudo rpcdebug -m nfs -c all
```

`rpcdebug -s all` 可能产生大量内核日志，只能在明确授权、有限时间和可控日志容量下使用。

### 6.4 最小验证实验与回滚

实验目标是把一个小文件写入转换为可观察的 RPC/NFS 活动，不验证生产容量或性能上限。

**执行端：客户端**  
**适用范围：独立测试导出；替换测试挂载点和服务端地址**

```bash
NFS_SERVER_IP=192.0.2.10
TEST_DIR=/mnt/nfs-test
TEST_FILE="$TEST_DIR/.nfs-kb-l0-02-$(hostname)-$$"
PCAP_FILE=/tmp/nfs-kb-l0-02-$$.pcap

rpcinfo -T tcp "$NFS_SERVER_IP" nfs 3 || true
nfsstat -c > /tmp/nfs-before.txt
sudo timeout 20 tcpdump -i any -nn -s 0 -w "$PCAP_FILE" "host ${NFS_SERVER_IP} and port 2049" >/tmp/nfs-kb-l0-02-tcpdump.log 2>&1 &
CAPTURE_PID=$!
sleep 2
printf 'sunrpc-xdr-rpc-lifecycle\n' > "$TEST_FILE"
sync -f "$TEST_FILE"
stat "$TEST_FILE"
nfsstat -c > /tmp/nfs-after.txt
diff -u /tmp/nfs-before.txt /tmp/nfs-after.txt || true
wait "$CAPTURE_PID" || true
rm -f "$TEST_FILE"
```

预期观察：客户端 NFS/RPC 计数增加；抓包文件应在写入期间记录 TCP stream 和 RPC CALL/REPLY，具体 procedure 取决于 NFS 版本、缓存命中和写回时机。使用支持 ONC RPC/NFS 的解码器按 XID 关联请求和响应。测试结束后删除临时文件、抓包文件和日志，并关闭 `rpcdebug`。

`sync -f` 依赖支持该选项的 GNU coreutils；若目标发行版不支持，应使用应用自身的 `FileChannel.force(true)` 或等价系统调用完成持久化验证，不要把普通 `sync` 当成 NFS 协议层的 COMMIT 观测替代。

抓包使用 `-s 0` 会保留完整 payload，可能包含业务数据；仅允许在隔离测试导出或经过授权的短时诊断中执行。

当前工作区没有 Linux NFS 环境，本实验属于待执行验证，不构成已验证结论。

## 7. Java 工程视角

### 7.1 RPC 延迟如何表现为 Java 延迟

Java API 不直接暴露 XID 或 NFS procedure，但一次调用的总延迟可能包含：

```text
Java 方法调用
  -> VFS 路径解析和缓存检查
  -> NFS 客户端排队
  -> RPC 编码与发送
  -> 网络 RTT
  -> 服务端 nfsd 排队和本地文件系统
  -> RPC 解码与客户端缓存更新
  -> Java 方法返回
```

应用监控的 p99 延迟只能说明总耗时，不能单独说明是网络、nfsd 还是后端磁盘。需要把 Java trace 时间点与 `nfsiostat`、TCP 和服务端指标对齐。

### 7.2 重试与业务幂等

HTTP 重试、消息消费重试和 NFS RPC 重试可能叠加：

```text
业务请求超时
  -> Java 重试写文件
  -> 原始 NFS RPC 仍在 hard 重试
  -> 两个业务执行可能交错
  -> 文件内容或 rename 顺序出现竞态
```

对上传、归档和任务结果文件，建议使用请求 ID/幂等键命名临时文件，写完并校验后再 rename；不要在应用层无条件重复执行不可幂等的覆盖写。

### 7.3 异常分类

Java 常见异常只是下层状态的映射：

| Java 表现 | 可能的下层来源 | 首要证据 |
| --- | --- | --- |
| `IOException`/`FileSystemException` 或长时间无返回 | RPC/TCP 超时、服务端不可达、hard 重试；具体异常类型取决于内核 errno 与 JDK 映射 | 线程栈、Linux task 状态、`ss`、`nfsiostat` |
| `AccessDeniedException` | AUTH_SYS/Kerberos/UID/GID/ACL/export 权限 | `id`、挂载 `sec`、服务端审计 |
| `NoSuchFileException` | NFS `NOENT`、目录缓存、路径或时序问题 | 双端 `stat`、GETATTR/LOOKUP |
| `FileSystemException: Stale file handle` | `ESTALE`、服务端导出/文件系统变化 | 服务端导出、内核日志、抓包 |
| `ClosedChannelException` | 应用生命周期或并发关闭，不一定是 NFS | Java 代码与线程栈 |

NFS 文件访问发生在 Linux 内核 VFS/SUNRPC 路径中，不是 Java socket 调用；因此不要仅捕获 `SocketTimeoutException` 来处理 NFS 超时。应用还应考虑 hard 挂载下 I/O 长时间不返回的情况。

## 8. 生产排障检查清单

遇到“RPC 超时、NFS 操作偶发失败或应用延迟升高”时：

1. 记录客户端、服务端、挂载点、NFS 版本、故障时间窗口和受影响操作。
2. 先确认 TCP 连接和路由，再确认 RPC 程序/版本，最后分析 NFS 状态码。
3. 用 `nfsstat -m` 和 `findmnt` 固化实际挂载参数，不依赖 `/etc/fstab` 推测。
4. 用 `nfsiostat` 区分 RTT、执行时间、队列时间和 retrans 增长。
5. 用 XID 将抓包中的 CALL/REPLY 与同一时间窗口的应用请求关联。
6. NFSv4 重点检查 COMPOUND 中失败的具体操作、clientid、stateid、session/slot 和 lease。
7. NFSv3 重点检查 rpcbind、mountd、NLM/NSM、UDP/TCP 传输和文件句柄。
8. 权限问题检查认证 flavor、Kerberos 时间同步、UID/GID、export 和 ACL，不用增加重试掩盖。
9. 变更参数前保留基线；调试结束后关闭 `rpcdebug`，清理抓包和临时文件。

常见现象映射：

| 现象 | 优先检查 | 不应直接得出的结论 |
| --- | --- | --- |
| TCP 已连接但 NFS 请求超时 | nfsd/存储队列、RPC 请求状态、网络丢包 | TCP established 不代表 NFS 服务正常 |
| `rpcinfo` 查不到 NFSv4 | rpcbind 注册与端口查询方式 | 不代表 TCP/2049 上的 NFSv4 不可用 |
| 抓包只有 TCP 分片 | record marking、TCP 重组、抓包点 | 一个 segment 不是一个 RPC |
| XID 有响应但应用仍报错 | RPC 已成功、NFS status 为错误 | XID 匹配成功不等于文件操作成功 |
| 重试后文件内容异常 | 业务重试与 RPC 重试叠加、非幂等覆盖写 | hard 不等于 exactly-once |
| NFSv4 锁/打开状态异常 | lease、stateid、session、服务端重启恢复 | 不能只重启 Java 进程解决协议状态 |

## 9. 小结

SUNRPC 是 NFS 的远程调用承载层，XDR 是其跨平台线格式。正确排障必须按以下顺序分层：

```text
TCP/UDP 是否传输
  -> RPC CALL/REPLY 是否完整
  -> XID 是否匹配、是否重试
  -> RPC 认证是否通过
  -> NFS procedure/COMPOUND 哪一步失败
  -> 服务端 VFS、文件系统和存储是否成功
```

需要牢记：

- XID 只匹配 RPC 请求，不等同于文件句柄、stateid 或业务请求 ID。
- TCP segment、`recv()` 次数和抓包 packet 数量都不能直接代表 RPC 消息数量。
- `AUTH_SYS` 提供身份字段而非加密；完整性和隐私需要 RPCSEC_GSS/Kerberos 等机制。
- hard 重试能改变故障返回语义，但不能替代业务幂等和 exactly-once 设计。
- NFSv4.1 的 session/slot、NFSv4 的状态恢复和 NFSv3 的 rpcbind/NLM/NSM 必须按版本分别分析。

## 10. 参考资料与关联文档

### 参考资料

- RFC 1831：RPC: Remote Procedure Call Protocol Specification Version 2
- RFC 1832：XDR: External Data Representation Standard
- RFC 1813：NFS Version 3 Protocol Specification
- RFC 7530：Network File System (NFS) Version 4 Protocol
- RFC 5661：Network File System (NFS) Version 4 Minor Version 1 Protocol
- `man 5 nfs`、`man 8 rpcinfo`、`man 8 nfsstat`、`man 8 mountstats`、`man 8 nfsiostat`
- Linux 内核文档：SUNRPC、NFS client、RPCSEC_GSS 和 VFS 相关文档

### 关联文档

- [NFS-KB-L0-01 Java 文件 I/O 到 NFS 服务端的端到端链路](NFS-KB-L0-01-end-to-end-io.md)
- 待建立：`NFS-KB-L1-01` NFSv3 到 NFSv4.2 的协议演进与版本选型
- 待建立：`NFS-KB-L2-01` 服务端导出与客户端挂载基线
- 待建立：`NFS-KB-L4-01` NFS 性能指标、基线与容量模型

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-07-31 | 1.1.0 | 修正 Java 异常映射、NFSv4.1 时序、恢复语义、session/slot 观测边界和抓包实验 | 基于文档审查结果修订 |
| 2026-07-31 | 1.0.0 | 初始发布 | 建立 SUNRPC、XDR 与 RPC 请求生命周期模型 |
