# NFS-KB-L1-02 NFSv3 协议流程与状态边界

> 文档状态：待验证  
> 知识阶段：L1 协议原理  
> 适用范围：Linux NFSv3 客户端与服务端；TCP/UDP 传输；`nfs-utils`、NLM/NSM、AUTH_SYS/RPCSEC_GSS 的具体行为以目标实现为准  
> 版本：1.1.0  
> 最后更新：2026-08-01  
> 前置文档：[NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](NFS-KB-L1-01-protocol-evolution-and-version-selection.md)、[NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)  
> 关联文档：待后续建立（见第 10 节）

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. NFSv3 总体模型](#2-nfsv3-总体模型)
- [3. 挂载与文件句柄](#3-挂载与文件句柄)
- [4. 读写与一致性](#4-读写与一致性)
- [5. 锁与状态边界](#5-锁与状态边界)
- [6. 故障、重试与错误码](#6-故障重试与错误码)
- [7. 生产验证与观察指标](#7-生产验证与观察指标)
- [8. Java 工程视角](#8-java-工程视角)
- [9. 排障检查清单](#9-排障检查清单)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

NFSv3 常被称为“无状态 NFS”，但生产系统中仍然存在文件句柄、客户端缓存、TCP 连接、锁服务、服务端请求缓存和本地文件系统状态。本单元要建立准确边界：哪些状态属于 NFSv3 基本文件操作，哪些状态由 NLM/NSM 或实现扩展提供。

完成本单元后，应能够：

1. 从 `mount` 追踪到 `rpcbind`、`mountd`、根 filehandle 和后续 NFSv3 RPC。
2. 解释 filehandle 如何定位对象，以及导出/文件系统变化为何导致 `ESTALE`。
3. 区分 READ/WRITE/COMMIT 的协议语义、客户端缓存与服务端稳定写入。
4. 解释 NLM/NSM 与 NFSv3 基本文件操作的边界，以及锁恢复的失败模式。
5. 用命令、抓包和日志判断问题发生在挂载控制面、NFS 数据面、锁服务还是后端存储。

本文不把 NFSv3 作为新建系统的默认推荐；版本选择需参照 [L1-01](NFS-KB-L1-01-protocol-evolution-and-version-selection.md) 的兼容性、安全和运维矩阵。

## 2. NFSv3 总体模型

### 2.1 控制面与数据面

```text
控制面
  rpcbind/portmapper -> 查询程序端口
  mountd             -> 将导出路径转换为根 filehandle
  NLM/NSM             -> 文件锁与锁状态恢复（可选但常见）

数据面
  NFS program 100003 version 3
    -> LOOKUP / GETATTR / READ / WRITE / COMMIT / CREATE / REMOVE / RENAME ...
    -> 服务端 nfsd、VFS、本地文件系统和存储
```

NFSv3 文件 I/O 通常使用 NFS 程序 `100003` 的 version 3。挂载时可能先访问 `rpcbind` 和 `mountd`，但挂载完成后，数据面请求通常直接发送到 NFS 服务端口；锁请求则发送到 NLM，状态通知由 NSM/statd 辅助。

### 2.2 常用 procedure

| Procedure | 作用 | 常见失败线索 |
| --- | --- | --- |
| `NULL` | 连通性探测，无 NFS 参数 | 只能证明 RPC procedure 可达 |
| `GETATTR/SETATTR` | 获取或修改文件属性 | 属性缓存、权限、时间戳和截断语义 |
| `LOOKUP/ACCESS` | 查找名称、检查访问权限 | 路径、UID/GID、目录权限和缓存 |
| `READLINK/READ` | 读取符号链接或文件数据 | 文件句柄、后端读取和网络 |
| `WRITE/COMMIT` | 写入并推进到稳定存储 | 稳定级别、write verifier、空间和重试 |
| `CREATE/MKDIR/SYMLINK/MKNOD` | 创建文件系统对象 | create mode/verifier、权限、配额 |
| `REMOVE/RMDIR` | 删除文件或目录 | 权限、目录非空、只读和并发操作 |
| `RENAME/LINK` | 改名或创建硬链接 | 同文件系统边界、目标冲突和缓存可见性 |
| `READDIR/READDIRPLUS` | 枚举目录；PLUS 同时返回属性/句柄 | 大目录、cookie/cookie verifier、元数据延迟 |
| `FSSTAT/FSINFO/PATHCONF` | 容量、传输偏好与文件系统约束 | 配额、最大文件、名称长度和实现差异 |

一次 Java `open` 或 `Files.move` 可能触发多个 procedure；不能把业务方法名直接等同于单个 NFS RPC。

## 3. 挂载与文件句柄

### 3.1 NFSv3 挂载流程

```text
mount -t nfs -o vers=3 server:/export/app /mnt/app
  -> rpcbind GETPORT(mountd)
  -> mountd.MNT(/export/app)
  <- 根 filehandle + auth flavor 列表
  -> rpcbind GETPORT(nfs)（若端口未固定）
  -> 建立 NFSv3 客户端挂载
  -> LOOKUP/GETATTR/READ/WRITE ...
```

实际步骤由 `mount.nfs`、内核和配置决定；管理员应以 `findmnt`、`nfsstat -m` 和 `rpcinfo` 结果为准，而不是假设所有发行版端口流程完全相同。

**执行端：客户端**  
**适用范围：Linux NFSv3 测试挂载；使用隔离测试目录**

```bash
NFS_SERVER=nfs.example.com
TEST_MOUNT=/mnt/nfs-v3-test
sudo mkdir -p "$TEST_MOUNT"
sudo mount -t nfs -o vers=3,proto=tcp "${NFS_SERVER}:/export/app" "$TEST_MOUNT"
findmnt -t nfs -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
```

**清理/回滚：客户端**  
**执行条件：全部读写、锁和故障测试完成，且测试挂载点无进程占用**

```bash
TEST_MOUNT=/mnt/nfs-v3-test
sudo umount "$TEST_MOUNT"
```

### 3.2 filehandle 的语义

服务端通过 filehandle 标识文件或目录对象。客户端通常通过 `LOOKUP` 从父目录和名称得到子对象的 filehandle，后续 READ/WRITE/GETATTR 携带该句柄，而不是每次都发送完整路径。

```text
路径 /export/app/orders/a.json
  -> LOOKUP "orders" 得到目录 filehandle
  -> LOOKUP "a.json" 得到文件 filehandle
  -> READ/WRITE 携带文件 filehandle + offset
```

filehandle 不是永久不变的业务 ID。服务端重建文件系统、改变导出边界、替换后端对象或某些存储故障恢复后，旧句柄可能失效，客户端收到 `ESTALE`（stale file handle）。无限重试不能修复失效句柄，通常需要确认服务端导出和文件系统变化，再重新解析路径或重新挂载。

### 3.3 `rpcbind`、固定端口与防火墙

NFSv3 的控制面可能涉及多个 RPC 程序和动态端口。生产环境应固定 `mountd`、`statd`、锁服务端口并在防火墙中显式放行；不要只放行 TCP/2049 就认为 v3 挂载和锁一定可用。具体端口配置是发行版/服务管理问题，需参考目标系统 `nfs.conf`、systemd unit 和 `rpcinfo -p`。

**执行端：客户端或管理节点**  
**适用范围：允许访问服务端 rpcbind 的诊断节点；结果受防火墙和 RPC 注册策略影响**

```bash
NFS_SERVER=nfs.example.com
rpcinfo -p "$NFS_SERVER"
rpcinfo -T tcp "$NFS_SERVER" nfs 3
rpcinfo -T tcp "$NFS_SERVER" mountd 3 || true
```

## 4. 读写与一致性

### 4.1 READ 流程

```text
应用 read()
  -> 客户端页缓存命中？
      | 是：复制到应用缓冲区
      | 否
      v
  -> NFSv3 READ(filehandle, offset, count)
  -> 服务端 nfsd/VFS/页缓存/本地文件系统
  -> READ reply(data, eof, attributes)
  -> 客户端缓存并返回应用
```

`rsize` 控制客户端请求的目标大小，但实际 READ 大小还受协议、内核、服务端和缓存影响。随机小读更容易受 RTT 和元数据延迟影响，顺序大读更容易受网络和后端吞吐限制。

### 4.2 WRITE 与稳定性

NFSv3 `WRITE` 请求包含 offset、count、数据和请求稳定级别 `stable`；响应通过 `committed` 报告服务端实际达到的稳定级别。客户端必须检查响应，不能只根据请求参数推断结果：

| `stable`/`committed` 值 | 响应含义 | 注意事项 |
| --- | --- | --- |
| `UNSTABLE` | 服务端可能只写入缓存 | 客户端可能随后发送 `COMMIT` |
| `DATA_SYNC` | 数据已达到稳定介质要求，但元数据可能未完全同步 | 仍需结合服务端文件系统语义 |
| `FILE_SYNC` | 数据和相关元数据达到稳定存储要求 | 不等于跨站点灾备完成 |

客户端 `write()` 返回、服务端 `WRITE` 响应、`COMMIT` 完成和存储设备 flush 是不同时间点。`sync`/`async` 导出、服务端文件系统、RAID 控制器和设备写缓存都会影响最终耐久性。

WRITE 和 COMMIT 响应还包含服务端 `write verifier`。客户端保留未稳定写入期间的 verifier；如果服务端重启或丢失易失缓存导致 verifier 改变，客户端必须把相关数据重新写入，而不能把新的 COMMIT 响应当作旧数据已经稳定。

### 4.3 属性缓存与 close-to-open

NFSv3 客户端缓存文件属性和目录项，以减少 GETATTR/LOOKUP。多个客户端之间通常呈现 close-to-open 风格的一致性，而不是每次读取都强制访问服务端：

- 写入客户端可能从自己的缓存立即读到新数据；
- 其他客户端何时看见更新取决于属性和数据缓存失效；
- `actimeo`、`acregmin/max`、`acdirmin/max` 会改变缓存窗口；
- `noac` 减少属性缓存但增加 RPC 和延迟，不能作为通用一致性修复。

需要强可见性时，应明确应用协议（例如临时文件 + `fsync` + rename + 版本校验），并在双客户端测试中验证，不能只依赖降低缓存时间。

### 4.4 COMMIT 与客户端崩溃

当客户端收到 `UNSTABLE` 写入结果，可能批量发送 `COMMIT`。客户端在 COMMIT 前崩溃、网络中断或服务端异常时，已返回成功的应用写入不必然等于稳定介质中已有完整数据。需要可靠边界的 Java 程序应在事务点使用 `FileChannel.force(true)`，并通过故障注入验证服务端存储。

## 5. 锁与状态边界

### 5.1 NFSv3 基本操作与 NLM/NSM

NFSv3 基本文件操作不携带 NFSv4 的 OPEN/stateid/lease。`FileChannel.lock()` 等锁语义通常由 NLM（Network Lock Manager）承载，NSM/statd 用于重启通知和恢复辅助：

```text
Java FileLock
  -> POSIX byte-range lock（Linux 通常经 fcntl）
  -> NLM LOCK/UNLOCK/TEST
  -> NSM/statd 监测客户端/服务端重启
  -> 锁恢复或失败
```

不同内核、锁实现和网络拓扑下的恢复行为可能不同。锁服务端口、statd 状态目录、主机名解析和防火墙必须纳入部署基线。`nolock`、`local_lock=` 等挂载选项会改变锁是在本地模拟还是通过 NLM 协调；多客户端共享写场景不得在未验证语义时启用本地锁。

### 5.2 锁故障模式

| 现象 | 可能原因 | 首要证据 |
| --- | --- | --- |
| `lock()` 长时间阻塞 | 对端不可达、NLM 重试、网络分区 | 线程栈、`rpcinfo`、网络连接、内核日志 |
| 锁看似丢失 | 客户端/服务端重启、statd 状态丢失、主机身份变化 | statd 日志、状态目录、服务端时间线 |
| 两端都认为持有锁 | 网络分区、fencing 缺失、实现恢复窗口 | 双端日志、网络分区记录、业务写入校验 |
| `ENOLCK` 或协议错误 | 锁服务不可用、资源耗尽、版本实现差异 | NLM/NSM 日志、内核参数、服务端负载 |

NLM 锁不应替代跨机房、高可用协调服务。关键互斥需要定义持有者失效、网络分区、fencing 和恢复后的仲裁策略。

## 6. 故障、重试与错误码

### 6.1 重试边界

NFSv3 over TCP 仍可能因为服务端处理慢、连接断开或网络分区触发 RPC 重试。`hard` 通常让 I/O 持续等待并重试，`soft` 族可能较早向应用返回错误；两者都不能提供业务 exactly-once。

```text
CALL(XID=0x1234)
  -> 响应丢失
  -> 客户端重传相同请求或重建连接
  -> 服务端可能返回缓存结果或再次处理
```

读操作通常更容易重试；WRITE、CREATE、RENAME 等操作必须结合稳定写、服务端重复请求缓存和应用幂等策略分析。

### 6.2 创建、目录遍历与重复执行

NFSv3 `CREATE` 定义三种创建模式：

| 模式 | 语义 | 重试注意 |
| --- | --- | --- |
| `UNCHECKED` | 可创建或修改已存在对象属性 | 重复请求可能作用于已存在文件 |
| `GUARDED` | 目标存在则返回 `EXIST` | 适合要求“仅创建一次”的调用，但响应丢失仍需判定结果 |
| `EXCLUSIVE` | 使用 create verifier 支持可重试的独占创建 | 客户端重试时必须保持 verifier 一致 |

`READDIR/READDIRPLUS` 使用 cookie 和 cookie verifier 继续目录遍历。目录在分页期间发生修改、服务端重启或 verifier 不匹配时，客户端可能需要从头重新读取；应用不能把 NFS 目录 cookie 当作永久游标。

`RENAME` 在同一服务端文件系统命名空间内具有协议定义的原子目录项更新语义，但响应丢失后重试仍需检查源、目标和服务端实际结果。跨文件系统移动不是一个 NFSv3 RENAME，通常需要复制再删除。

### 6.3 常见 NFSv3 错误

| NFSv3 状态 | 常见含义 | 排查方向 |
| --- | --- | --- |
| `NFS3ERR_NOENT` | 文件/目录不存在 | 路径、目录缓存、并发删除/rename |
| `NFS3ERR_ACCES` | 权限拒绝 | AUTH_SYS UID/GID、ACL、export、root squash |
| `NFS3ERR_EXIST` | 创建目标已存在 | 并发创建、独占标志、应用幂等 |
| `NFS3ERR_NOSPC` | 空间不足 | 文件系统、配额、后端池 |
| `NFS3ERR_DQUOT` | 配额超限 | 用户/组/项目配额 |
| `NFS3ERR_STALE` | filehandle 失效 | 导出、文件系统、后端对象变化 |
| `NFS3ERR_ROFS` | 只读文件系统 | 服务端导出、文件系统故障恢复 |
| `NFS3ERR_IO` | 服务端 I/O 错误 | 服务端日志、文件系统、块设备 |

应用日志中的 `EIO`、`ESTALE`、`EACCES` 可能是这些协议状态映射后的结果；需要同时保存服务端和客户端证据。

## 7. 生产验证与观察指标

### 7.1 实际挂载与服务检查

**执行端：客户端**  
**适用范围：已授权的测试或生产诊断窗口**

```bash
NFS_SERVER=nfs.example.com
NFS_MOUNT=/mnt/app
findmnt -t nfs -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
mountpoint "$NFS_MOUNT"
rpcinfo -p "$NFS_SERVER"
ss -tin "dst ${NFS_SERVER}:2049"
```

### 7.2 观察 NFS 操作与后端

**执行端：客户端与服务端**  
**适用范围：需 root 的服务端指标仅在授权窗口采集**

```bash
# 客户端
nfsstat -c
nfsiostat 1 10
cat /proc/self/mountstats

# 服务端
nfsstat -s
exportfs -v
iostat -xz 1 10
vmstat 1 10
journalctl -u nfs-server --since "10 min ago"
```

重点观察：READ/WRITE/GETATTR/LOOKUP/COMMIT 调用量和延迟、RPC retrans/timeout、TCP 重传、nfsd CPU/线程、后端 `await`/`%util`、空间/配额和内核 I/O 错误。

### 7.3 最小功能实验

**执行端：客户端**  
**适用范围：独立测试导出；不要在生产目录执行删除和锁故障实验**

```bash
TEST_DIR=/mnt/nfs-v3-test
TEST_FILE="$TEST_DIR/.nfs-kb-l1-02-$(hostname)-$$"
printf 'nfsv3-read-write-commit\n' > "$TEST_FILE"
sync -f "$TEST_FILE"
stat "$TEST_FILE"
sha256sum "$TEST_FILE"
mv "$TEST_FILE" "$TEST_FILE.renamed"
rm -f "$TEST_FILE.renamed"
```

预期观察：文件创建、属性读取、数据读取、rename 和删除均成功；同时记录 `nfsstat` 前后计数和 `nfsiostat`。`sync -f` 是否可用取决于 GNU coreutils，不能作为所有发行版的必备命令。

该实验只验证基本文件功能，不能单独证明客户端发送了 `COMMIT`、服务端返回的 `committed` 级别或 write verifier 恢复逻辑。验证稳定写语义时，应在隔离环境配合 NFS 抓包、服务端日志和服务端重启故障注入，并比较 WRITE/COMMIT 响应中的 `committed` 与 verifier；生产环境禁止未经评估直接执行此类故障注入。

## 8. Java 工程视角

### 8.1 API 到 NFSv3 procedure

| Java API | 可能触发的 NFSv3 操作 | 工程注意 |
| --- | --- | --- |
| `Files.exists`/`Files.size` | GETATTR、LOOKUP | 属性缓存可能让结果短暂滞后 |
| `Files.newInputStream` | LOOKUP、GETATTR、READ | 大量小读暴露 RTT 和缓存开销 |
| `Files.write` | LOOKUP/GETATTR/CREATE/SETATTR、WRITE、COMMIT | 操作组合取决于文件是否存在、是否截断、打开选项和客户端缓存 |
| `Files.move` | RENAME、GETATTR | 目标存在、跨文件系统和缓存需实测 |
| `FileChannel.lock` | NLM LOCK/TEST/UNLOCK | 网络分区时可能阻塞或恢复失败 |

NFSv3 没有协议级 OPEN procedure；Java 的打开语义由客户端 VFS 和后续 NFS 操作实现。应用日志需要同时记录路径、操作类型、耗时、线程池和实例，才能与 NFS procedure 计数对齐。

当挂载使用 `nolock` 或 `local_lock=` 将锁限制在客户端本地时，`FileChannel.lock()` 只能协调同一客户端上的进程，不能提供多客户端互斥。共享写应用必须验证实际挂载参数和 NLM 通信，不能仅以 Java 返回了 `FileLock` 判断分布式锁已经建立。

### 8.2 线程池与 hard 挂载

服务端不可达时，NFSv3 `hard` 挂载可能让 Java 线程长时间等待。JVM 线程 dump 可能显示 `RUNNABLE`，Linux task 可能处于 `D` 状态。业务超时或 `Future.cancel` 不保证立即终止内核 I/O，应通过线程池隔离、请求级降级和独立健康检查避免共享线程池被拖垮。

## 9. 排障检查清单

遇到 NFSv3 挂载失败、慢 I/O、锁异常或 `ESTALE` 时：

1. 固化客户端 OS、内核、`nfs-utils`、挂载参数和服务端地址。
2. 用 `rpcinfo -p` 验证 rpcbind 中的 nfs、mountd、nlockmgr、status 注册；用 `ss` 验证实际连接。
3. 挂载失败先查 mountd/export/path/firewall，再查 NFS 数据面。
4. I/O 变慢按 `strace`/线程栈 -> `nfsiostat` -> TCP -> 服务端 nfsd/磁盘顺序定位。
5. 锁问题单独检查 NLM/NSM、statd 状态目录、主机名解析、时间线和 fencing。
6. `ESTALE` 重点调查导出重载、文件系统替换、后端故障恢复和句柄生命周期，不要无限重试。
7. `NOSPC/DQUOT/ROFS/IO` 需要服务端文件系统、配额、块设备和内核日志证据。
8. 任何调整 `soft`、`async`、`noac`、锁参数或防火墙端口前，保留基线、风险评估和回滚步骤。

## 10. 参考资料与关联文档

### 参考资料

- RFC 1813：NFS Version 3 Protocol Specification
- RFC 1831：RPC: Remote Procedure Call Protocol Specification Version 2
- RFC 1832：XDR: External Data Representation Standard
- `man 5 nfs`、`man 5 exports`、`man 8 mount.nfs`、`man 8 rpcinfo`、`man 8 nfsstat`
- Linux 内核文档：NFS client、NLM/NSM、SUNRPC、VFS 和 writeback

### 关联文档

- [NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](NFS-KB-L1-01-protocol-evolution-and-version-selection.md)
- [NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)
- 待建立：`NFS-KB-L1-03` NFSv4 状态、租约、stateid 与锁恢复
- 待建立：`NFS-KB-L2-01` 服务端导出与客户端挂载基线
- 待建立：`NFS-KB-L5-01` NFS 故障树与证据链排障方法

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-01 | 1.1.0 | 拆分挂载回滚、修正 Java procedure 映射、补充稳定写实验边界和本地锁限制 | 基于文档审查结果修订 |
| 2026-08-01 | 1.0.0 | 初始发布 | 建立 NFSv3 协议流程与状态边界模型 |
