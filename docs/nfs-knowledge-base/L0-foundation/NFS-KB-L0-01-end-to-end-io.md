# NFS-KB-L0-01 Java 文件 I/O 到 NFS 服务端的端到端链路

> 文档状态：待验证  
> 知识阶段：L0 基础设施层  
> 适用范围：Linux NFS 客户端与服务端；NFSv3、NFSv4.0、NFSv4.1；Java 11+。具体参数以目标发行版、内核和 `nfs-utils` 版本为准  
> 版本：1.1.0
> 最后更新：2026-07-31  
> 前置文档：无  
> 关联文档：待后续建立（见第 10 节）

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. 总体模型](#2-总体模型)
- [3. 写入链路](#3-写入链路)
- [4. 读取链路](#4-读取链路)
- [5. 一致性、持久化与异常语义](#5-一致性持久化与异常语义)
- [6. 生产观测与验证](#6-生产观测与验证)
- [7. Java 工程视角](#7-java-工程视角)
- [8. 生产排障检查清单](#8-生产排障检查清单)
- [9. 小结](#9-小结)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

本单元回答一个生产中最常见的问题：

> Java 应用执行 `Files.write(path, bytes)` 后，数据经过哪些层，什么时候对其他客户端可见，什么时候真正落到服务端磁盘，哪一层变慢会让 Java 线程卡住？

完成本单元后，应能够：

1. 从 Java API 追踪到 Linux 系统调用、VFS、NFS RPC 和服务端本地文件系统。
2. 区分“应用调用返回”“客户端缓存接收”“服务端确认”“稳定存储完成”四个时间点。
3. 根据 `strace`、`nfsstat`、`mountstats`、`iostat` 和抓包结果判断瓶颈所在层。
4. 解释 NFS 客户端为何可能在网络中断时让 Java 线程长时间阻塞。
5. 识别缓存、一致性、锁和 `fsync` 语义导致的“文件已经写了但看不到”或“返回成功但恢复后丢失”的边界。

本文重点是 Linux 客户端与服务端的实现模型，不把 NFS 当作“远程磁盘协议”的简单封装。生产环境仍需结合具体内核、`nfs-utils`、存储阵列和网络设备验证。

## 2. 总体模型

### 2.1 端到端路径

```text
Java 应用线程
    |
    | java.nio.file.Files / FileChannel / RandomAccessFile
    v
Linux 系统调用层
    | openat2/openat, read, write, fsync, close, fcntl
    v
客户端 VFS
    | inode/dentry/page cache、权限检查、锁、本地回写
    v
Linux NFS 客户端
    | NFS 操作编码、属性缓存、重试、状态恢复
    v
SUNRPC + XDR
    | RPC 请求、认证、NFSv4 COMPOUND、slot/session
    v
TCP/网络设备/交换网络
    | 通常为 TCP/2049；NFSv3 还涉及 mountd/rpcbind 控制面
    v
服务端 SUNRPC + nfsd
    | 解码、权限映射、export 检查、状态管理
    v
服务端 VFS
    | inode、页缓存、writeback、锁、配额
    v
本地文件系统与块设备
    | XFS/ext4/ZFS/阵列/SSD/HDD
    v
稳定介质
```

链路中每一层都有自己的队列、缓存、超时和失败语义。排障时必须先确定是“请求没有发出”“请求在网络中”“服务端正在处理”还是“已经处理但尚未稳定落盘”。

### 2.2 控制面与数据面

| 平面 | 典型内容 | 生产意义 |
| --- | --- | --- |
| 控制面 | NFS 版本协商、挂载、文件句柄、客户端标识、租约、锁恢复 | 失败通常表现为挂载失败、`ESTALE`、锁异常或状态恢复问题 |
| 数据面 | LOOKUP、GETATTR、READ、WRITE、COMMIT、OPEN、CLOSE | 失败通常表现为吞吐下降、延迟升高、I/O 错误或线程阻塞 |

NFSv3 的挂载通常由 `mountd` 和 `rpcbind` 辅助完成，实际文件 I/O 主要发送到 NFS 服务；NFSv4 将挂载命名空间纳入协议本身，通常只需 TCP/2049，具体行为由实现和配置决定。

### 2.3 三个容易混淆的时间点

```text
t0  Java write()/close() 返回
    对普通 buffered I/O，可能只表示客户端已接受数据；不等于稳定落盘

t1  NFS WRITE 响应返回 committed 状态
    可能是 UNSTABLE、DATA_SYNC 或 FILE_SYNC；UNSTABLE 不等于稳定存储完成

t2  COMMIT/服务端文件系统/存储设备完成稳定写入
    仍取决于服务端 export 策略、文件系统、控制器和设备写缓存
```

具备持久化要求的应用应使用 `FileChannel.force(true)`、`fsync` 或等价机制表达意图；这会促使客户端执行相应的刷新和 NFS 稳定写流程，但不能单独证明跨故障域持久化。最终可靠性仍受服务端文件系统、存储控制器和 NFS 导出策略影响。

## 3. 写入链路

### 3.1 Java API 到系统调用

```java
Path path = Path.of("/mnt/nfs/orders/2026-07-31/order-001.json");
Files.writeString(path, payload, StandardCharsets.UTF_8,
    StandardOpenOption.CREATE,
    StandardOpenOption.TRUNCATE_EXISTING,
    StandardOpenOption.WRITE);
```

典型调用链不是固定的一对一映射，但通常包含：

```text
Files.writeString
  -> UnixChannelFactory / FileChannelImpl
  -> openat/openat2
  -> write 或 writev
  -> close
```

`strace` 只能看到本机系统调用，看不到 NFS 操作名称；系统调用返回后，NFS 客户端可能仍在处理缓存和异步回写，必须结合 `mountstats` 或 `nfsstat` 判断协议层行为。

### 3.2 openat 阶段

`openat` 首先经过客户端 VFS：

1. 解析路径中的目录项和挂载点。
2. 对每一级目录执行缓存查找，缓存失效时发出 `LOOKUP`/`GETATTR` 等 NFS 操作。
3. 检查客户端看到的 UID/GID、模式位、ACL 和挂载权限。
4. 创建或打开远端 inode；NFSv4 可能通过 `OPEN`、`CLAIM` 和 stateid 建立打开状态。

NFSv3 没有 NFSv4 的 open state，应用的打开语义主要由客户端本地处理，并通过普通文件操作完成服务端访问。NFSv4 的 `OPEN` 还涉及 clientid、owner、share reservation 和 stateid，因此客户端重启或租约恢复时会出现额外状态流转。

验证路径：

**执行端：客户端**  
**适用范围：Linux，需具备对目标进程的跟踪权限**

```bash
JAVA_PID=12345
strace -ff -ttT -yy -e trace=openat,openat2,write,writev,fsync,fdatasync,close -p "$JAVA_PID"
```

观察 `openat`、`write`、`fsync`、`close` 的耗时；不要假设 `close` 永远是常数时间操作。

### 3.3 write 阶段

对普通 buffered I/O，应用的 `write()` 通常先进入客户端页缓存，之后由 NFS 客户端按实现和挂载参数组织 WRITE 请求。`O_DIRECT`、`O_SYNC`、部分 direct I/O 和内存映射写入可能采用不同路径，不能套用下面的简化流程：

```text
write()
  -> 客户端 page cache 标记 dirty
  -> 按 rsize/wsize 和协议限制切分数据
  -> WRITE RPC
  -> 服务端 nfsd 权限与 export 检查
  -> 服务端 VFS/page cache
  -> 服务端文件系统 writeback
  -> WRITE/COMMIT 响应
```

关键区别：

- `wsize` 是客户端单次 NFS WRITE 请求的目标最大数据量，不等于 Java `write()` 的大小，也不等于实际网络包大小。
- TCP 会进一步分段，不能用 TCP 包数量直接推断 NFS 操作数量。
- `sync` 导出要求服务端按更严格的稳定性语义完成写入；`async` 可能降低延迟，但服务端异常时有数据丢失风险。
- 服务端返回 `UNSTABLE` 时，客户端后续可能发送 `COMMIT`，要求服务端将之前的写入推进到稳定存储。

### 3.4 close、flush 和 COMMIT

Java `close()`、Linux `close()`、NFS `COMMIT` 和后端介质 flush 是不同层次的动作。`close()` 可能触发文件操作的 flush 回调或写回，但不能据此假设一定发出完整的 COMMIT：

```text
Java close
  -> 文件描述符关闭
  -> 可能触发客户端 dirty page 刷新或协议收尾
  -> NFS WRITE / COMMIT
  -> 服务端文件系统 flush
  -> 存储设备确认
```

不能用“`close()` 返回成功”证明数据已经完成跨故障域持久化。具备持久化边界要求的程序可在关键记录完成后调用：

```java
try (FileChannel channel = FileChannel.open(path,
        StandardOpenOption.CREATE,
        StandardOpenOption.WRITE,
        StandardOpenOption.TRUNCATE_EXISTING)) {
    ByteBuffer buffer = StandardCharsets.UTF_8.encode(payload);
    while (buffer.hasRemaining()) {
        channel.write(buffer);
    }
    channel.force(true);
}
```

`force(true)` 增加延迟和服务端压力，不应对每个小片段无条件调用。应根据事务边界批量写入，并通过基准测试确认 `fsync`/`COMMIT` 成本。

## 4. 读取链路

### 4.1 read 的缓存路径

```text
read()
  -> 客户端 page cache 命中？
       | 是：直接复制到 Java 缓冲区
       | 否
       v
  -> READ RPC
  -> 服务端文件系统/page cache
  -> 服务端 READ 响应
  -> 客户端 page cache
  -> 复制到 Java 缓冲区
```

`rsize` 控制单次 READ RPC 的目标大小；Java 缓冲区、顺序访问、客户端 readahead 和服务端缓存都会影响吞吐。小而随机的读取更容易暴露 RTT 和元数据延迟，大而顺序的读取更容易受网络带宽和存储吞吐限制。

### 4.2 属性缓存与目录缓存

客户端会缓存 `GETATTR` 结果以及目录项，以减少每次 `stat()`、`open()` 和目录遍历的 RPC。由此产生 close-to-open 一致性语义，而不是本地文件系统的强一致直觉：

- 一个客户端写入后，自己的后续读取通常可以看到本地修改。
- 另一客户端是否立即看到修改，取决于属性/数据缓存失效和访问时机。
- `actimeo`、`acregmin`、`acregmax`、`acdirmin`、`acdirmax` 会改变缓存窗口，不能只看吞吐而忽略可见性。
- `noac` 会增加 RPC 和延迟，不能当作通用一致性修复方案。

验证挂载实际参数：

**执行端：客户端**  
**适用范围：Linux NFS 客户端**

```bash
nfsstat -m
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS
cat /proc/mounts | awk '$3 == "nfs" || $3 == "nfs4"'
```

## 5. 一致性、持久化与异常语义

### 5.1 一致性不是持久化

“另一客户端能读到”只说明数据在协议和缓存层面可见；“节点掉电后数据仍存在”要求更强的稳定存储链路：

| 问题 | 主要相关因素 | 观测方式 |
| --- | --- | --- |
| 其他客户端何时看见文件 | 属性缓存、目录缓存、close-to-open、`actimeo` | 双客户端交叉读写、`stat`/`getattr` 时间线 |
| Java 写调用何时返回 | 客户端缓存、WRITE 回复、RPC 重试 | `strace`、`mountstats`、应用延迟 |
| 服务端故障后是否保留数据 | `stable`/`UNSTABLE`、COMMIT、文件系统和设备 flush | 故障注入、服务端日志、恢复后校验 |
| 写入者崩溃后文件是否完整 | 分段写、rename 策略、fsync 顺序 | 进程 kill、节点重启和校验和 |

### 5.2 原子更新模式

对配置、索引和小型 JSON 文件，不建议直接覆盖被其他进程读取的文件。更可控的模式是“写临时文件、force 临时文件、rename 为目标文件，必要时 force 父目录”：

```java
Path target = Path.of("/mnt/nfs/config/app.json");
Path temp = target.resolveSibling(".app.json.tmp-" + ProcessHandle.current().pid());

try (FileChannel channel = FileChannel.open(temp,
        StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE)) {
    ByteBuffer buffer = StandardCharsets.UTF_8.encode(payload);
    while (buffer.hasRemaining()) {
        channel.write(buffer);
    }
    channel.force(true);
}

try {
    Files.move(temp, target, StandardCopyOption.ATOMIC_MOVE);
} catch (AtomicMoveNotSupportedException e) {
    Files.deleteIfExists(temp);
    throw e;
}
```

`ATOMIC_MOVE` 是否能在 NFS 上得到目标语义取决于服务端、文件系统和同一文件系统边界；必须在目标环境验证，不能仅依据本地测试结果。使用 `ATOMIC_MOVE` 时，Java 规范允许其他复制选项被忽略，因此不能假设 `REPLACE_EXISTING` 一定生效。若目标文件已存在，必须针对目标服务端验证替换语义；不支持原子移动时示例选择失败并清理临时文件，而不是静默降级为非原子覆盖。需要目录项持久化时，还应单独评估父目录的 `fsync` 能力。

### 5.3 hard 与 soft 的故障语义

生产客户端通常优先使用 `hard`，因为临时网络或服务端故障时继续重试可以避免静默数据损坏；代价是 I/O 系统调用可能长时间阻塞，Java 线程池可能耗尽。`soft`/`softerr`/`softreval` 可能更快返回错误，但应用必须正确处理部分写入、重试和一致性，否则会把基础设施故障转化为数据损坏。

这不是“永远选 hard”的简单结论：应将挂载语义与应用超时、线程池隔离、熔断、优雅停机和数据重试策略一起设计。

## 6. 生产观测与验证

### 6.1 客户端基线

**执行端：客户端**  
**适用范围：Linux NFS 客户端**

```bash
# 挂载与协商参数
nfsstat -m
findmnt -t nfs,nfs4

# NFS 操作计数、错误和 RPC 状态
nfsstat -c
cat /proc/self/mountstats
nfsiostat 1 10

# 块设备和 CPU 基线
iostat -xz 1 10
vmstat 1 10
NFS_SERVER_IP=192.0.2.10
ss -tin "dst ${NFS_SERVER_IP}:2049"
```

重点指标：

- `retrans`、RPC timeout、transport backlog：网络丢包、服务端响应慢或客户端队列问题的线索。
- READ/WRITE/GETATTR/LOOKUP 的调用次数与 RTT/执行时间：使用 `nfsiostat` 区分数据面和元数据面。
- 客户端 CPU、软中断、内存回收和本地磁盘等待：NFS 不是唯一瓶颈。
- TCP 重传、RTT、拥塞窗口和连接数：确认网络是否放大了应用延迟。

### 6.2 服务端基线

**执行端：服务端**  
**适用范围：Linux NFS 服务端，需 root 或等价权限**

```bash
exportfs -v
nfsstat -s
cat /proc/fs/nfsd/threads
iostat -xz 1 10
vmstat 1 10
ss -s
journalctl -u nfs-server --since "10 min ago"
```

关注 nfsd 线程配置与 CPU 是否饱和、后端块设备 `await`/`%util` 是否饱和、文件系统错误、内存压力、网络队列和导出选项是否在变更后发生变化。线程排队需要结合 `nfsiostat`、内核 tracing 或 eBPF 进一步确认，不能仅由 `/proc/fs/nfsd/threads` 的线程数推断。

### 6.3 从网络确认协议操作

**执行端：客户端或旁路抓包节点**  
**适用范围：已获生产抓包授权；抓包可能包含业务数据，必须脱敏并限制保存时间**

```bash
NFS_SERVER_IP=192.0.2.10
timeout 30 tcpdump -i any -nn -s 0 -w /tmp/nfs-trace.pcap "host ${NFS_SERVER_IP} and port 2049"
```

抓包用于确认请求是否发出、TCP 是否重传、请求与响应之间的时间差以及 NFS 操作序列。`-s 0` 会保留完整 payload，可能包含业务数据；生产环境禁止在未评估磁盘、CPU、隐私和合规影响时长期全量抓包。

### 6.4 最小验证实验与回滚

以下实验建议在独立测试导出上执行，不要使用生产目录。实验需要一台服务端和至少两台客户端，或一台客户端配合服务端本地校验。

**执行端：客户端 1**<br>
**适用范围：已挂载的测试 NFS 目录；将变量替换为真实测试路径**

```bash
TEST_DIR=/mnt/nfs-test
TEST_FILE="$TEST_DIR/.nfs-kb-l0-01-$(hostname)-$$"
printf 'nfs-kb-l0-01 %s\n' "$(date -Is)" > "$TEST_FILE"
sync -f "$TEST_FILE"
stat "$TEST_FILE"
sha256sum "$TEST_FILE"
```

**执行端：客户端 2**<br>
**预期观察：** 在一致性缓存窗口内读取同一文件，记录 `stat` 的大小、mtime 和校验和；若与客户端 1 不一致，继续对照挂载参数中的 `actimeo`、客户端时钟和服务端日志。

```bash
TEST_FILE=/mnt/nfs-test/.nfs-kb-l0-01-REPLACE-ME
stat "$TEST_FILE"
sha256sum "$TEST_FILE"
```

**清理/回滚：客户端 1**

```bash
rm -f "$TEST_FILE"
```

实验应同时记录客户端/服务端 OS、内核、NFS 版本、挂载与导出参数、执行时间、`nfsiostat` 输出和实际观察结果。当前工作区没有 Linux NFS 环境，因此本实验仅作为待执行验证步骤，不构成已验证结论。

## 7. Java 工程视角

### 7.1 阻塞点如何传递到线程池

NFS 客户端在内核中等待 RPC 时，Linux task 可能处于 `D`（不可中断睡眠）状态；但 JVM 线程 dump 可能仍显示为 `RUNNABLE`，因为线程正在执行 native I/O。应用层可能只看到：

```text
HTTP 请求堆积
  -> 业务线程阻塞在 Files.read/write/close
  -> Tomcat/Netty/自建线程池耗尽
  -> 健康检查超时
  -> 容器被判定为不健康并重启
```

Java 的 `Future` 超时不会必然中断内核中的 NFS I/O；中断线程后，底层文件描述符和 NFS 请求仍需观察。应同时采集 JVM 线程栈和 Linux task 状态，并结合线程池隔离、请求超时、降级路径和挂载故障语义设计。

### 7.2 FileLock 不等于分布式锁服务

`FileChannel.lock()` 依赖操作系统文件锁实现。在 NFSv3 中通常涉及 NLM/NSM，在 NFSv4 中由协议状态模型承载；网络分区、客户端重启、租约恢复和服务端故障都可能影响锁的可用性。不能把 NFS 文件锁直接当作跨机房、高可用分布式锁。

对关键互斥需求，应明确锁持有者崩溃后的释放方式、网络分区时允许阻塞还是双写、是否需要 fencing，以及是否更适合使用数据库、Redis、ZooKeeper 或专门的协调服务。

### 7.3 mmap、direct I/O 和小文件

- `mmap` 通过缺页异常触发远端读取，访问模式不明显时可能产生大量随机 READ。
- `O_DIRECT`/direct I/O 会绕过部分页缓存，但不等于绕过 NFS 客户端和服务端所有缓存；支持和收益取决于内核及服务端实现。
- 大量小文件会将 LOOKUP、GETATTR、CREATE、REMOVE、RENAME 等元数据操作放大，不能用大文件顺序吞吐基准代表真实应用性能。

## 8. 生产排障检查清单

遇到“Java 访问 NFS 变慢、卡死或返回 I/O 错误”时，按以下证据顺序采集：

1. 记录业务时间线：首个异常请求、影响接口、线程池、实例和文件路径模式。
2. 确认挂载实际参数：NFS 版本、`hard/soft`、`timeo/retrans`、`rsize/wsize`、`actimeo`、`nconnect`。
3. 用 `strace` 或 Java 线程栈确定卡在 `openat`、`read`、`write`、`fsync` 还是 `close`。
4. 在客户端采集 `nfsstat -c`、`mountstats`、TCP 状态、CPU、内存和网络重传。
5. 在服务端采集 `nfsstat -s`、nfsd 线程、导出配置、文件系统、块设备和内核日志。
6. 必要时短时抓包，验证 RPC 请求是否发出、是否重传以及响应延迟。
7. 对照变更记录：挂载变更、内核/nfs-utils 升级、DNS/网络策略、存储阵列和应用发布。
8. 只有在证据明确后才调整参数；变更后用同一负载、同一指标和同一时间窗口回归。

常见现象到首要方向的映射：

| 现象 | 首要检查层 | 不应直接下的结论 |
| --- | --- | --- |
| `open()` 慢、`stat()` 大量变慢 | 目录/属性缓存、元数据 RPC、服务端文件系统 | 不应直接认为网络带宽不足 |
| `write()` 或 `fsync()` 慢 | 服务端稳定写、COMMIT、后端存储、网络 RTT | 不应只增大 `wsize` |
| 所有 Java 线程卡住 | hard 重试、网络分区、服务端不可达、线程池耗尽 | 不应只重启应用实例 |
| 文件一端存在、另一端暂时看不到 | 属性/目录缓存、一致性窗口、路径或权限 | 不应立即认定文件丢失 |
| `ESTALE` | 文件句柄失效、服务端导出/文件系统变化 | 不应通过无限重试掩盖导出变更 |
| 锁冲突或锁恢复失败 | NFS 状态、租约、NLM/NSM、客户端重启 | 不应把 FileLock 当作强一致协调服务 |

## 9. 小结

NFS 文件访问是一条跨用户态、VFS、客户端缓存、NFS 协议、RPC/TCP、服务端 `nfsd`、服务端 VFS 和后端存储的链路。应用看到的 Java 方法耗时只是整条链路的最终表现，排障必须沿层次逐段建立证据。

最重要的工程结论是：

- “写调用返回”“其他客户端可见”“服务端已确认”“稳定介质持久化”不是同一件事。
- NFS 的吞吐、延迟和可靠性由协议参数、缓存策略、网络、nfsd、文件系统和设备共同决定。
- Java 超时和线程中断不能自动消除内核 NFS 阻塞；应用容错必须和挂载故障语义共同设计。
- 任何参数调优都必须有基线、观测指标、回滚路径和同负载验证。

## 10. 参考资料与关联文档

### 参考资料

- RFC 1813：NFS Version 3 Protocol Specification
- RFC 7530：Network File System (NFS) Version 4 Protocol
- RFC 5661：Network File System (NFS) Version 4 Minor Version 1 Protocol
- `man 5 nfs`、`man 5 exports`、`man 8 nfsstat`、`man 8 mount.nfs`
- Linux 内核文档：NFS client、SUNRPC、VFS 和 writeback 相关文档

### 关联文档

后续文档生成后，将以下规划标识替换为相对 Markdown 链接；当前文件尚不存在，因此暂不创建失效链接：

- 待建立：`NFS-KB-L0-02` SUNRPC、XDR 与 RPC 请求生命周期
- 待建立：`NFS-KB-L1-01` NFSv3 到 NFSv4.2 的协议演进与版本选型
- 待建立：`NFS-KB-L2-01` 服务端导出与客户端挂载基线
- 待建立：`NFS-KB-L4-01` NFS 性能指标、基线与容量模型
- 待建立：`NFS-KB-L5-01` NFS 故障树与证据链排障方法

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-07-31 | 1.1.0 | 修正稳定写语义、Java 部分写入、原子移动、观测指标和验证实验 | 基于文档审查结果修订 |
| 2026-07-31 | 1.0.0 | 初始发布 | 建立 Java 文件 I/O 到 NFS 服务端的端到端模型 |
