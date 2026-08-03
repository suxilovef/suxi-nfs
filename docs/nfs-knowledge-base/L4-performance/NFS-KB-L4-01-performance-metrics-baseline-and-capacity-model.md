# NFS-KB-L4-01 NFS 性能指标、基线与容量模型

> 文档状态：待验证  
> 知识阶段：L4 性能与容量  
> 适用范围：Linux NFS 客户端与服务端；RHEL/Rocky/AlmaLinux 8/9、Ubuntu 22.04/24.04 常见环境；NFSv3、NFSv4.0、NFSv4.1、NFSv4.2；Linux knfsd、NAS、云文件服务及 Java 17/21 应用；指标名称和工具字段以目标内核、nfs-utils、sysstat、node_exporter、存储厂商及云监控版本为准  
> 版本：1.0.1
> 最后更新：2026-08-03  
> 前置文档：[NFS-KB-L0-01 端到端 I/O 路径](../L0-foundation/NFS-KB-L0-01-end-to-end-io.md)、[NFS-KB-L0-02 SUNRPC、XDR 与请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)、[NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)  
> 关联文档：[NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)、[NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](../L3-security/NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)、[NFS-KB-L3-03 NFSv4 ACL、SELinux 与多协议权限集成](../L3-security/NFS-KB-L3-03-nfsv4-acl-selinux-and-multiprotocol-permissions.md)

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. NFS 性能模型与指标语义](#2-nfs-性能模型与指标语义)
- [3. 端到端性能路径与基线流程](#3-端到端性能路径与基线流程)
- [4. 配置、命令与指标采集](#4-配置命令与指标采集)
- [5. Java 工程视角](#5-java-工程视角)
- [6. 生产基线、SLI/SLO 与容量模型](#6-生产基线slislo-与容量模型)
- [7. 验证实验与观察指标](#7-验证实验与观察指标)
- [8. 性能证据链与检查清单](#8-性能证据链与检查清单)
- [9. 小结](#9-小结)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

NFS 性能不是一个“MB/s”数字。应用一次 `read()`、`write()`、`fsync()`、`open()`、`rename()` 或 `FileChannel.force()`，可能命中客户端缓存，也可能经过 RPC 排队、TCP、服务端 nfsd、后端文件系统、RAID/对象存储和持久化介质。相同吞吐量下，p99 延迟、重传、排队和数据持久性可能完全不同。

完成本单元后，应能够：

1. 建立应用、客户端 VFS、SUNRPC、网络、服务端 nfsd 与后端存储的分层指标模型。
2. 区分吞吐量、IOPS、RPC ops/s、平均 RTT、平均执行时间、排队时间和应用端分位延迟。
3. 使用 Little's Law 将请求速率、延迟与在途并发关联起来。
4. 建立包含工作负载、缓存状态、协议版本、安全 flavor 和后端介质的可比较基线。
5. 使用 `nfsiostat`、`mountstats`、`nfsstat`、`sar`、`ss`、`iostat`、PSI 和服务端指标定位饱和层。
6. 为 Java 服务设计文件操作级 SLI，而不是只观察 JVM CPU、GC 或接口总耗时。
7. 将业务峰值转换为网络带宽、RPC 并发、服务端 CPU、后端数据与元数据能力需求。
8. 设计容量余量、告警、变更前后对照、HA 节点一致性和多租户隔离策略。
9. 识别“调大 rsize/wsize”“增加 nfsd threads”“关闭同步语义”等缺乏证据的错误调优方式。

本文建立性能测量语言和生产基线，不展开所有调优参数，也不把一次 `fio` 结果当作容量认证。后续性能压测、客户端与服务端调优、网络专项和故障案例必须复用本文指标口径。

### 1.1 性能问题必须先定义

以下描述都不够精确：

- “NFS 很慢”；
- “带宽跑不满”；
- “延迟偶尔高”；
- “Java 写文件卡住”；
- “服务端磁盘没满，所以不是存储问题”。

至少补全：

```text
operation + object type + request size + concurrency
  + cache state + sync/durability semantics
  + NFS version/security flavor + mount/export options
  + client/server/storage topology
  + time window + baseline + percentile/error definition
```

例如：

```text
200 个 Java Pod 通过 NFSv4.1/krb5p 写 64 KiB 临时文件，
每文件 close 前 FileChannel.force(true)，工作日 10:00-10:15，
p99 create-to-force 从 35 ms 升至 420 ms，错误率 0.2%，
同一时段客户端 avg RTT、avg exe 和服务端后端 await 同时上升。
```

这是可分析的问题；“共享盘变慢”不是。

## 2. NFS 性能模型与指标语义

### 2.1 六层观测模型

| 层次 | 关键问题 | 代表指标 | 常见盲区 |
| --- | --- | --- | --- |
| 应用/Java | 用户实际等待多久、什么操作失败 | 操作级 p50/p95/p99、超时、异常、字节数 | 只看接口总耗时 |
| 客户端 VFS/缓存 | 是否命中页缓存、是否在回写或脏页限流 | page cache、dirty/writeback、PSI、系统调用延迟 | 把缓存吞吐当远端能力 |
| NFS/SUNRPC | 哪类操作、请求多大、是否排队或重传 | ops/s、kB/op、avg RTT、avg exe、retrans | 将 RPC 次数等同业务 IOPS |
| 网络/传输 | TCP、RDMA 或 UDP 路径是否丢包、拥塞或受队列限制 | 带宽、RTT、retrans、drops、cwnd/QP、队列 | 只 ping，不看实际 MDS/DS 连接 |
| 服务端/nfsd | 是否有请求积压、线程或 CPU 饱和 | NFS op counters、RPC errors、线程、CPU、run queue | 只看 nfsd CPU 总量 |
| 后端文件系统/介质 | 数据与元数据能否按目标延迟持久化 | IOPS、MB/s、await、queue、cache、元数据延迟 | 把阵列 cache 命中当稳态能力 |

一次慢操作必须尽量在相同时间窗口取得六层证据。仅在问题发生后查看一张服务端 CPU 图，无法证明因果关系。

### 2.2 缓存命中与 RPC 路径

```text
Java read
  -> Linux read()/pread()
     -> client page cache hit
        -> local memory copy，可能没有 READ RPC
     -> cache miss
        -> NFS READ RPC queue
        -> TCP request/response
        -> nfsd + backend read
        -> client page cache
        -> Java buffer
```

因此：

- 应用读取 1 GiB，不代表网络传输 1 GiB；
- `dd` 第二次读取更快，可能只是客户端页缓存命中；
- 服务端存储读吞吐低，不代表 NFS 没有读负载；NAS cache 可能吸收读取；
- `direct=1`、`O_DIRECT` 和 buffered I/O 是不同工作负载，不能混合比较；
- 清空客户端 cache、服务端 cache 和阵列 cache 是三件不同的事。

生产环境禁止为了制造冷缓存执行：

```bash
echo 3 | sudo tee /proc/sys/vm/drop_caches
```

该操作影响整机所有业务，且仍不能清空 NAS、阵列或云文件服务缓存。冷读应使用隔离客户端、超出缓存的工作集、首次读取的新对象或厂商认可的方法验证。

### 2.3 延迟分解

对一个实际发出的同步 RPC，可使用以下概念模型：

```text
T_application
  = T_userspace
  + T_VFS_and_client_queue
  + T_RPC_execution
  + T_copy_and_wakeup

T_RPC_execution
  ~= T_client_RPC_queue + T_round_trip_and_server

nfsiostat avg exe
  ~= request enters RPC layer -> request completes

nfsiostat avg RTT
  ~= request is transmitted -> reply is received

queue indicator
  ~= avg exe - avg RTT
```

`avg exe - avg RTT` 只能作为客户端 RPC 排队/调度线索，不是严格的服务端排队公式。工具字段是累计平均或采样区间平均，不是 p99；不能用它替代应用端分位延迟。

同步写的持久化路径还取决于 NFS 稳定性语义：

```text
write()/FileChannel.write()
  -> client dirty page or WRITE RPC
  -> server may reply UNSTABLE / DATA_SYNC / FILE_SYNC
  -> COMMIT may be required
  -> backend cache/flush/barrier/replication
  -> application fsync()/force() returns
```

只测 buffered `write()` 返回时间，不能证明数据已经稳定落盘。

### 2.4 吞吐量、IOPS 与 RPC ops/s

| 指标 | 定义 | 不能直接说明 |
| --- | --- | --- |
| 应用吞吐 | 应用每秒处理的有效数据量 | 网络或存储实际字节数 |
| NFS kB/s | NFS 挂载点采样到的协议数据速率 | 后端介质吞吐与持久化状态 |
| NFS ops/s | 某类 NFS 操作速率 | 业务文件数或磁盘 IOPS |
| 存储 IOPS | 后端块/对象操作速率 | NFS RPC 数；cache 和合并会改变映射 |
| 文件/s | create/open/write/close/rename 完整事务速率 | 单个协议操作能力 |

常见映射变化：

- 应用多次 4 KiB 写可能被客户端合并为较大的 WRITE；
- 一次 1 MiB 应用读取可能被拆成多个 READ，或通过 readahead 提前读取；
- NFSv4 COMPOUND 可以在一次 RPC 中包含多个 operation；
- 一次文件创建可能涉及 `SEQUENCE`、`PUTFH`、`OPEN`、`GETATTR`、`CLOSE` 等；
- 后端文件系统可能把多个 WRITE 合并为更少的块 I/O；
- 元数据操作可能命中服务端内存而不产生可见磁盘 IOPS。

所以不能把 `nfsstat` operation counter、业务请求数和 `iostat` IOPS 做一比一换算。

### 2.5 Little's Law 与并发

稳态系统满足：

```text
N = lambda * W

N       平均在途请求数
lambda  完成率，单位 ops/s
W       平均响应时间，单位秒
```

示例：稳定完成 `10,000 ops/s`，平均 RPC 执行时间 `5 ms`：

```text
N = 10,000 * 0.005 = 50 outstanding requests
```

当平均延迟升到 `20 ms`，若业务仍要求 `10,000 ops/s`，理论上需要约 200 个在途请求。这个结果只是需求量，不证明客户端实际拥有 200 个可用 RPC/session slot，也不证明请求已经均匀分布到传输连接。客户端 RPC slot、NFSv4.1+ 协商后的 session slot、传输连接、nfsd 并发、CPU 或后端队列中任何一层不足，都会让吞吐停止增长并积累排队。

增加应用线程或虚拟线程只能增加到达率和排队，不能创造服务能力。

### 2.6 工作负载指纹

任何基线必须记录：

| 维度 | 典型取值 | 影响 |
| --- | --- | --- |
| 操作 | READ、WRITE、COMMIT、LOOKUP、GETATTR、OPEN、CLOSE、LOCK、READDIR | 数据与元数据路径不同 |
| 请求大小 | 4 KiB、64 KiB、1 MiB、协商后的 rsize/wsize | RPC 数、CPU、网络效率 |
| 访问模式 | 顺序、随机、追加、覆盖 | readahead、cache、存储队列 |
| 文件规模 | 大文件、海量小文件、深目录、单热目录 | 元数据与锁竞争 |
| 读写比 | 读多、写多、混合 | 网络方向、cache 与持久化压力 |
| 同步语义 | buffered、close、fsync/force、O_SYNC | 延迟和数据安全不同 |
| 缓存状态 | 冷、暖、稳定工作集 | RPC 和后端负载不同 |
| 并发 | 线程、进程、Pod、客户端、挂载数 | slot、锁、队列与公平性 |
| 共享度 | 独占文件、共享读、共享写、锁 | cache 一致性与状态操作 |
| 协议 | v3、v4.0、v4.1、v4.2、pNFS | 操作组合与状态路径不同 |
| 安全 | `sec=sys/krb5/krb5i/krb5p` | CPU、payload 大小和可观测性不同 |
| 拓扑 | 同机架、跨 AZ、跨地域 | RTT、故障域和带宽成本 |

只要其中一个核心维度不同，两次测试就不具备严格可比性。

### 2.7 数据面与元数据面

数据面主要观察：

- READ/WRITE/COMMIT；
- kB/op、kB/s；
- 网络吞吐与重传；
- 客户端 page cache、dirty/writeback；
- 服务端数据 cache 与后端吞吐、IOPS、延迟。

元数据面主要观察：

- LOOKUP、GETATTR、ACCESS、READDIR；
- OPEN/CLOSE、CREATE、REMOVE、RENAME、LINK；
- NFSv4 state、lease、LOCK；
- 目录宽度、深度、负 negative lookup；
- 服务端 inode/dentry/cache、目录锁和元数据介质。

大文件顺序读基线优秀，不能证明百万小文件创建性能合格。

### 2.8 分位延迟、长尾与协调遗漏

平均值会掩盖长尾。例如 99 次 2 ms 和 1 次 2 s 的平均值约 22 ms，但用户已经经历 2 秒卡顿。

生产至少记录：

- p50：稳定中心；
- p95：常见高位；
- p99/p99.9：长尾和容量边界；
- max：用于关联故障，但不单独作为趋势；
- timeout/error：延迟统计之外的失败请求；
- sample count：没有样本量的分位值不可解释。

压测工具如果在前一个请求完成后才发送下一个请求，会在系统卡顿时自动降低到达率，遗漏本应到达的请求，即 coordinated omission。容量测试要明确使用闭环还是开放环模型，并同时报告实际到达率与完成率。

### 2.9 饱和与反压

典型饱和链：

```text
backend latency rises
  -> nfsd requests remain active longer
  -> server run queue / RPC backlog grows
  -> client avg RTT rises
  -> client avg exe and application latency rise
  -> client RPC slots fill
  -> application threads block
  -> timeout/retry/traffic burst may amplify load
```

也可能从客户端、网络或认证层开始。判断瓶颈必须找到最早饱和并形成排队的资源，不能把最后看到高延迟的层误认成根因。

## 3. 端到端性能路径与基线流程

### 3.1 Java 同步写路径

```text
Java business request
  -> Files.newByteChannel / FileChannel.open
  -> OPEN/CREATE + identity/permission checks
  -> FileChannel.write
  -> page cache and/or NFS WRITE
  -> FileChannel.force(true)
  -> remaining WRITE + COMMIT/stable write
  -> backend flush/replication
  -> CLOSE
  -> optional atomic rename
  -> application latency histogram
```

需要分别计时 create/open、write、force、close、rename。只记录方法总耗时会把权限、元数据、数据传输和持久化混在一起。

### 3.2 Java 读取路径

```text
application read latency
  -> open/lookup/getattr/access
  -> page-cache hit?
       yes -> memory copy
       no  -> READ RPC -> network -> nfsd -> backend/cache
  -> readahead useful or wasted?
  -> close
```

读取基线至少区分：首次读取、同客户端重复读取、另一客户端读取和工作集大于缓存后的稳定读取。

### 3.3 元数据事务路径

```text
create temporary file
  -> parent lookup/access
  -> OPEN(CREATE)
  -> SETATTR/ACL inheritance
  -> WRITE/COMMIT
  -> CLOSE
  -> RENAME into final name
  -> parent directory change/cache invalidation
```

小文件业务的单位应是“完成一个业务文件事务的 files/s 和 p99”，不能只报告 WRITE MB/s。

### 3.4 DMA 性能闭环

```text
Define
  -> 明确业务操作、SLO、峰值、数据安全和测试边界
Measure
  -> 同时间采集应用、客户端、RPC、网络、服务端、存储
Analyze
  -> 找到最早饱和资源，验证排队和因果关系
Change
  -> 一次只改变一个变量，保留配置和回滚
Validate
  -> 相同工作负载复测，比较分位、吞吐、错误和资源成本
Baseline
  -> 归档结果、适用范围、容量余量和失效条件
```

### 3.5 基线快照组成

```text
baseline-id
  + timestamp/timezone
  + topology and node roles
  + OS/kernel/nfs-utils/JDK
  + mount/export/security flavor
  + negotiated rsize/wsize/transport
  + workload fingerprint
  + application percentiles/errors
  + mountstats/nfsstat deltas
  + network/transport counters
  + server CPU/nfsd/backend metrics
  + HA node and storage state
  + raw evidence retention location
```

基线不是永久常量。内核、JDK、NAS firmware、网络、目录规模、数据热度或安全 flavor 变化后必须重新建立。

## 4. 配置、命令与指标采集

### 4.1 环境与挂载快照

**执行端：客户端与服务端**  
**适用范围：只读取证；输出可能包含主机、共享、网络和安全配置，按敏感运维数据管理**

```bash
date --iso-8601=seconds
uname -a
cat /etc/os-release
lscpu
free -h

nfsstat -m
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS
cat /proc/fs/nfsfs/volumes 2>/dev/null || true

rpm -q nfs-utils sysstat fio 2>/dev/null || true
dpkg-query -W nfs-common sysstat fio 2>/dev/null || true
java -version 2>&1 || true
```

重点记录实际协商结果，不只记录 fstab 期望值。`nfsstat -m`/`findmnt` 应确认：

- NFS minor version；
- `sec=`；
- TCP 与端口；
- `rsize/wsize`；
- `hard/soft`、timeo/retrans；
- cache、lookupcache、actimeo/ac*；
- `nconnect`、pNFS 或厂商相关选项；
- 挂载源是否经过 DNS、VIP、负载均衡或自动挂载。

### 4.2 客户端 NFS 汇总指标

**执行端：客户端**  
**适用范围：先获取累计计数，再以固定区间采样；不同 nfs-utils 版本字段可能变化**

```bash
nfsstat --client
nfsstat --rpc
nfsiostat 1 10
mountstats iostat 1 10
```

指定挂载点时先检查本机帮助，因为 `nfsiostat` 与 `mountstats` 的参数位置随发行版包装存在差异：

```bash
nfsiostat --help 2>&1 | sed -n '1,160p'
mountstats --help 2>&1 | sed -n '1,160p'
```

常见字段：

| 字段 | 含义 | 解释边界 |
| --- | --- | --- |
| ops/s | 采样区间完成的 NFS 操作速率 | 不等于业务事务/s |
| kB/s | NFS 数据速率 | 不等于后端介质吞吐 |
| kB/op | 平均每操作数据量 | 元数据操作可能接近零 |
| retrans | RPC 重传次数或比例 | 要结合请求量、时间窗和 TCP 证据 |
| avg RTT | 发出到收到响应的平均时间 | 包含网络和服务端路径，不是纯网络 RTT |
| avg exe | RPC 从排队/执行到完成的平均时间 | 与应用 p99 不同 |

比较时必须使用区间增量。开机以来的累计平均会稀释短时故障。

### 4.3 原始 mountstats 证据

**执行端：客户端**  
**适用范围：只读取证；每个进程 mount namespace 视图可能不同，容器场景要在正确 namespace 采集**

```bash
TARGET=/mnt/secure

findmnt -T "$TARGET" -o TARGET,SOURCE,FSTYPE,OPTIONS
sed -n "/ mounted on ${TARGET//\//\\/} /,/^$/p" \
  /proc/self/mountstats
```

如果路径匹配脚本因空格、转义或工具版本失败，直接保留完整 `/proc/self/mountstats`，再使用结构化解析工具处理，不要用脆弱的字段位置做生产采集。

采集前后快照必须带时间戳，并计算 counter delta/rate。counter 回绕、重启、重新挂载、容器迁移和 HA 切换都会造成重置。

#### 4.3.1 RPC slot、session slot 与传输队列证据

Little's Law 计算出的在途需求，必须和以下四类证据对齐：

| 证据 | 要回答的问题 | 解释边界 |
| --- | --- | --- |
| 配置上限 | SUNRPC slot table、连接数等本机上限是多少 | 配置值不是当前占用，也不是 NFSv4.1+ 实际协商值 |
| 协商能力 | NFSv4.1+ fore channel 实际可用 session slot 是多少 | 不同内核未必通过稳定用户态接口完整暴露 |
| 当前压力 | pending、backlog、发送/接收、重传和应用 in-flight 是否同时增长 | 单个累计字段不能直接证明 slot 耗尽 |
| 连接分布 | `nconnect`、trunking、MDS/DS 各有多少 transport，负载是否均匀 | 连接数与 session slot 不是一一对应关系 |

先保留目标内核实际暴露的只读信息：

```bash
TARGET=/mnt/secure

findmnt -T "$TARGET" -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m

sysctl -a 2>/dev/null | grep -E '^sunrpc\..*(slot|tcp|udp)' || true
find /sys/module/sunrpc/parameters -maxdepth 1 -type f -print \
  -exec sed -n '1p' {} \; 2>/dev/null

sed -n "/ mounted on ${TARGET//\//\\/} /,/^$/p" \
  /proc/self/mountstats

perf list 2>/dev/null | grep -Ei 'sunrpc|rpc.*(queue|slot|xprt)' \
  | head -n 100
```

`/proc/self/mountstats` 的 `xprt` 字段可提供发送、接收、pending/backlog 等实现线索，但字段顺序和含义随内核版本、传输类型及 nfs-utils 解析器变化。SUNRPC 模块参数或 sysctl 只代表配置上限。NFSv4.1+ session slot 是客户端与服务端协商结果；若当前内核没有稳定接口显示实际值，应使用目标内核文档、受控 tracepoint/debug 工具或厂商遥测取证，不能用 `tcp_slot_table_entries` 代替协商值。

验证 slot 瓶颈时，应在同一时间窗同时看到应用 in-flight、RPC pending/backlog、完成率与延迟的变化，并在受控环境改变一个并发上限后观察拐点是否移动。`nconnect=N` 只是挂载配置请求；应从 mountstats 的 transport 实例和实际 socket 验证生效数量，并同时报告“每连接”和“该挂载总计”，不能把 N 条连接直接换算成 N 倍并发或吞吐。

### 4.4 NFS operation 分布

**执行端：客户端与服务端**

```bash
nfsstat --client --nfs
nfsstat --server --nfs 2>/dev/null || true

cat /proc/net/rpc/nfs 2>/dev/null || true
cat /proc/net/rpc/nfsd 2>/dev/null || true
```

分析时回答：

1. 数据面是 READ、WRITE 还是 COMMIT 主导？
2. GETATTR/LOOKUP/ACCESS 是否异常放大？
3. OPEN/CLOSE/LOCK 是否与文件事务速率匹配？
4. NFSv4 COMPOUND 中 operation 数和 RPC 数如何变化？
5. 客户端与服务端 counter 增量是否处于同一时间窗？
6. 重试、故障切换或多个客户端是否造成服务端计数更高？

不要跨 NFSv3 与 NFSv4 直接比较某一 operation 占比；协议流程不同。

### 4.5 网络与传输协议

**执行端：客户端与服务端**  
**适用范围：只读取证；以下主路径默认 NFS over TCP，接口和对端必须替换为真实值；RDMA、NFSv3 UDP/辅助 RPC 与 pNFS DS 必须走对应分支**

```bash
NFS_IFACE=eth0
NFS_SERVER=192.0.2.10

ip -s link show dev "$NFS_IFACE"
ethtool "$NFS_IFACE" 2>/dev/null || true
ethtool -S "$NFS_IFACE" 2>/dev/null || true

ss -tin '( sport = :2049 or dport = :2049 )'
nstat -az TcpRetransSegs TcpExtTCPSynRetrans 2>/dev/null || true
sar -n DEV,TCP,ETCP 1 10

ping -c 20 "$NFS_SERVER"
```

先从实际挂载确认 `proto=`，不能因为看到 2049 端口就假定所有数据都走同一条 TCP 连接：

```bash
TARGET=/mnt/secure
NFS_SERVER=192.0.2.10

findmnt -T "$TARGET" -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m

# NFSv3 可能还有动态端口的 rpcbind、mountd、lockd/statd 路径。
rpcinfo -p "$NFS_SERVER" 2>/dev/null || true
ss -uan
nstat -az UdpInDatagrams UdpOutDatagrams UdpInErrors \
  UdpRcvbufErrors UdpSndbufErrors IpInDiscards IpOutDiscards \
  2>/dev/null || true

# proto=rdma 时使用 RDMA 设备、端口、QP 和错误计数；字段依 rdma-core 版本变化。
rdma link show 2>/dev/null || true
rdma resource show qp 2>/dev/null || true
rdma statistic show 2>/dev/null || true
```

解释边界：

- ICMP ping 只提供路径 RTT 线索，不代表 NFS 请求服务时间；
- 主机级 `TcpRetransSegs` 包含其他 TCP 流量，要与 2049 连接、接口和时间窗关联；
- `ss -ti` 中 cwnd、rtt、rto、bytes_retrans 等字段依内核版本变化；
- NFSv3 的 UDP 请求没有 TCP cwnd/retrans 证据，必须结合 UDP socket、IP/UDP 错误、RPC 重传和辅助 RPC 端口；新部署默认优先验证 TCP，只有兼容性要求明确时才保留 UDP 分支；
- `proto=rdma` 不适用 TCP 2049 的拥塞窗口解释，需要检查 RDMA link/QP、端口错误、拥塞与厂商交换网络遥测；
- pNFS 可能让元数据继续连接 MDS，而 READ/WRITE 直接连接一个或多个 DS；必须识别所有 MDS/DS 端点并分别采集连接、接口、方向带宽、错误和路径状态；
- NIC drops 可能发生在物理口、bond、VLAN、虚拟交换机、宿主机或云网络多层；
- 带宽未满仍可能受单流拥塞窗口、RTT、RPC 并发、CPU 或小包速率限制。

不要在生产问题现场未经审批运行大包 flood、`iperf3` 满带宽或修改 MTU。

### 4.6 客户端 OS、缓存与反压

**执行端：客户端**

```bash
vmstat 1 10
pidstat -dru 1 10
sar -r -B -W -q 1 10

grep -E '^(MemAvailable|Cached|Dirty|Writeback|NFS_Unstable):' \
  /proc/meminfo
grep -E '^(nr_dirty|nr_writeback|nr_dirtied|nr_written|pgmajfault) ' \
  /proc/vmstat

cat /proc/pressure/cpu 2>/dev/null || true
cat /proc/pressure/io 2>/dev/null || true
cat /proc/pressure/memory 2>/dev/null || true
```

关注：

- 应用线程 blocked 与系统 load；
- dirty/writeback 是否持续积累；
- `NFS_Unstable` 是否与 COMMIT/稳定写路径相关；
- CPU softirq、system time 与 RPCSEC_GSS 加密开销；
- memory pressure 是否驱逐有效 cache；
- PSI `some/full` 是否与应用长尾同时间发生。

客户端 `iostat` 主要反映本地块设备，不能直接代表远端 NFS 后端；但本地日志盘、容器层或 swap 仍可能影响应用。

### 4.7 服务端 nfsd 与主机资源

**执行端：服务端**  
**适用范围：Linux knfsd；NAS/云服务使用厂商对应指标**

```bash
nfsstat --server
cat /proc/net/rpc/nfsd
cat /proc/fs/nfsd/threads 2>/dev/null || true

vmstat 1 10
mpstat -P ALL 1 10
pidstat -dru 1 10
sar -n DEV,TCP,ETCP 1 10

sudo exportfs -v
```

注意：

- nfsd 内核线程 CPU 可能分散在多个 CPU；只看单进程视图会漏计；
- nfsd thread 数不是越多越好，过多会增加调度、锁和后端队列压力；
- `/proc/net/rpc/nfsd` 是内核接口，字段随内核版本变化，生产解析器必须按版本测试；
- NAS 前端节点 CPU 正常，不代表后端控制器、元数据节点或对象存储正常；
- `exportfs -v` 只用于确认策略，不证明性能。

本单元不直接修改 `/proc/fs/nfsd/threads`。线程调优必须在并发曲线证明服务端执行槽不足、后端仍有余量后进行。

### 4.8 服务端后端存储

**执行端：服务端或存储管理面**  
**适用范围：Linux 本地块后端示例；NAS/云文件服务不能仅依赖 Linux `iostat`**

```bash
lsblk -o NAME,KNAME,TYPE,SIZE,FSTYPE,MOUNTPOINTS,ROTA,HCTL
findmnt -T /srv/nfs -o TARGET,SOURCE,FSTYPE,OPTIONS
iostat -xz 1 10
sar -d 1 10
cat /proc/pressure/io 2>/dev/null || true
```

常见字段：

| 字段 | 含义 | 风险 |
| --- | --- | --- |
| r/s、w/s | 块请求速率 | RAID、cache、合并改变物理 IOPS |
| rkB/s、wkB/s | 块层吞吐 | 不等于介质实际写入量 |
| await | 块请求平均完成时间 | 平均值掩盖长尾 |
| aqu-sz | 平均队列深度 | 要结合设备并行能力 |
| %util | 设备忙碌时间比例 | 多队列 NVMe/阵列上不能简单解释为容量百分比 |

生产还要从存储管理面取得：

- 前端协议延迟与后端介质延迟；
- read/write cache 命中和脏数据；
- RAID、EC、复制、快照和去重开销；
- 元数据节点/卷/聚合池利用率；
- 后台重建、scrub、快照删除、分层迁移；
- QoS 限额、burst credit 与多租户竞争；
- 可用容量和 inode/对象数量，而不只看字节容量。

### 4.9 perf、tracepoint 与 eBPF 边界

**执行端：问题节点**  
**风险：可能增加 CPU、日志和敏感路径暴露；先在测试环境确认事件名、采样率和输出范围**

```bash
sudo perf list | grep -Ei 'nfs|sunrpc|rpc' | head -n 100
sudo perf stat -p "<java-pid>" -e task-clock,cycles,instructions,context-switches \
  -- sleep 30

sudo bpftrace -l 'tracepoint:nfs:*' 2>/dev/null | head -n 100
sudo bpftrace -l 'tracepoint:sunrpc:*' 2>/dev/null | head -n 100
```

事件名和字段不是稳定跨内核 API。先列出本机事件，再设计过滤条件；禁止直接复制依赖不存在字段的 bpftrace 脚本。生产采样应限制 PID、挂载、操作和持续时间，并评估 `krb5p`、容器 namespace 和内核符号限制。

### 4.10 Prometheus 指标与 counter 规则

node_exporter 常见 Linux NFS collector 可能暴露：

```text
node_nfs_rpcs_total
node_nfs_rpc_retransmissions_total
node_nfs_requests_total{method=...}
node_nfsd_requests_total{method=...}
node_nfsd_server_threads
node_nfsd_rpc_errors_total{error=...}
```

名称和 label 必须以目标 node_exporter `/metrics` 为准。先发现：

```bash
curl --fail --silent http://127.0.0.1:9100/metrics \
  | grep -E '^node_nfs(d)?_' \
  | sed -n '1,200p'
```

示例 PromQL 只表达计算意图：

```promql
sum by (instance, proto, method) (
  rate(node_nfs_requests_total[5m])
)
```

```promql
(
  sum by (instance) (
    increase(node_nfs_rpc_retransmissions_total[5m])
  )
  /
  sum by (instance) (
    increase(node_nfs_rpcs_total[5m])
  )
)
and on (instance)
sum by (instance) (
  increase(node_nfs_rpcs_total[5m])
) >= 100
```

规则：

- counter 使用 `rate()`/`increase()`，不用原始值告警；
- node_exporter 的 NFS client 指标通常来自主机级 `/proc/net/rpc/nfs`，没有 mount point label；不能把主机聚合重传率直接归因到某个挂载；
- 重传比率分子和分母必须来自同一实例、时间窗和协议口径；
- 比率使用窗口内 counter 增量相除，并用独立的最小 RPC 样本数过滤低流量窗口；示例中的 `100` 只是演示值，必须按采集间隔、基线和告警灵敏度校准；
- 处理重启和重新挂载造成的 counter reset；
- 不把完整路径、用户、文件名做高基数 label；
- exporter 指标没有应用 p99 和 mountstats avg RTT 时，应部署受控采集器或应用埋点补齐；
- 报警阈值先从业务 SLO 与已验证基线推导，不照抄互联网固定数值。

### 4.11 最小 fio 烟雾测试

**执行端：隔离测试客户端**  
**风险：会在测试根目录下原子创建本次运行的独占目录，并消耗网络和存储；禁止在业务目录、快照关键卷或未知配额下执行**

```bash
set -euo pipefail

TEST_ROOT=/mnt/secure/.nfs-kb-l4-01

umask 077

test -d "$TEST_ROOT"
test ! -L "$TEST_ROOT"

RUN_DIR=$(mktemp -d -- "$TEST_ROOT/fio.XXXXXXXXXX")
RESULT_DIR=$(mktemp -d -- /var/tmp/nfs-kb-l4-01-fio.XXXXXXXXXX)
TEST_FILE="$RUN_DIR/fio-smoke.bin"
RESULT="$RESULT_DIR/write.json"
READ_RESULT="$RESULT_DIR/read.json"

cleanup_fio_smoke() {
  rc=0
  rm -f -- "$TEST_FILE" || rc=1
  rmdir -- "$RUN_DIR" || rc=1
  if (( rc != 0 )); then
    printf 'fio cleanup incomplete: %s\n' "$RUN_DIR" >&2
  fi
  return "$rc"
}
trap cleanup_fio_smoke EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

fio --name=nfs-write-smoke \
  --filename="$TEST_FILE" \
  --rw=write --bs=1m --ioengine=sync --iodepth=1 --numjobs=1 \
  --size=1g --direct=0 --fsync_on_close=1 \
  --group_reporting --eta=never --output-format=json \
  --output="$RESULT"

fio --name=nfs-read-smoke \
  --filename="$TEST_FILE" \
  --rw=read --bs=1m --ioengine=sync --iodepth=1 --numjobs=1 \
  --size=1g --direct=0 \
  --group_reporting --eta=never --output-format=json \
  --output="$READ_RESULT"

rm -f -- "$TEST_FILE"
rmdir -- "$RUN_DIR"
printf 'fio results: %s\n' "$RESULT_DIR"
trap - EXIT INT TERM
```

该实验只验证路径可用、采集链和数量级，不是容量测试：

- `direct=0` 包含客户端 cache 和 writeback；
- `fsync_on_close=1` 只在 close 时同步，不等价于每次业务 `force()`；
- 第二次 read 很可能是暖缓存；
- 单 job、单客户端不能证明集群并发能力；
- 1 GiB 可能小于任一层 cache；
- JSON 中的 clat 百分位是 fio I/O 口径，不是完整业务文件事务口径。

`mktemp -d` 在同一父目录中原子创建独占运行目录，避免“先检查、后创建”的固定路径竞态。清理函数只删除本次运行的已知文件并对独占目录执行 `rmdir`；若目录出现未知对象，清理会失败并保留现场，不使用递归删除。结果文件包含路径、主机和性能信息，应保持私有权限、设置保留期并按运维证据管理。

## 5. Java 工程视角

### 5.1 Java API 与真实操作

| Java 行为 | Linux/NFS 路径 | 性能风险 |
| --- | --- | --- |
| `Files.readAllBytes` | open + 分配内存 + 多次 read + close | 大文件内存和 GC；缓存命中掩盖 RPC |
| `BufferedInputStream` | 用户态缓冲叠加 page cache/readahead | 读取大小与 NFS READ 大小不一一对应 |
| `BufferedOutputStream` | 用户态缓冲后 write | flush 不等于持久化 |
| `FileChannel.force(true)` | fsync 类语义，数据和元数据同步 | COMMIT/后端 flush 长尾 |
| `Files.move(..., ATOMIC_MOVE)` | rename，要求同一服务端文件系统语义 | 元数据、目录锁和目标覆盖延迟 |
| `FileLock` | v3 NLM 或 v4 LOCK/state | 网络往返、恢复和锁竞争 |
| `MappedByteBuffer` | mmap/page fault/writeback | 延迟发生在缺页或回写，不在普通 read/write 计时点 |
| 虚拟线程并发 | 更多阻塞调用可并发挂起 | 不绕过 RPC slot、nfsd 或存储容量 |

`BufferedOutputStream.flush()` 只把 Java 缓冲写入操作系统，不保证 NFS 服务端稳定存储。要求持久性的业务必须定义 `force/fsync`、rename、目录持久化及失败恢复语义，而不是只测 `write()`。

### 5.2 Java 分阶段延迟探针

以下探针只在隔离测试目录执行，测量 create/open、write、force、close、rename 和 delete。单次结果不是基准测试。

```java
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.AtomicMoveNotSupportedException;
import java.nio.file.Files;
import java.nio.file.LinkOption;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.nio.file.StandardOpenOption;

public final class NfsDurabilityLatencyProbe {
    private NfsDurabilityLatencyProbe() {
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1) {
            throw new IllegalArgumentException(
                    "usage: NfsDurabilityLatencyProbe <test-directory>");
        }

        Path directory = Path.of(args[0]);
        if (!Files.isDirectory(directory, LinkOption.NOFOLLOW_LINKS)) {
            throw new IllegalArgumentException(
                    "test path must be an existing non-symlink directory");
        }

        Path runDirectory = Files.createTempDirectory(
                directory,
                ".nfs-kb-l4-01-java-");
        Path temporary = runDirectory.resolve("payload.tmp");
        Path completed = runDirectory.resolve("payload.done");

        FileChannel channel = null;
        try {
            long started = System.nanoTime();
            channel = FileChannel.open(
                    temporary,
                    StandardOpenOption.CREATE_NEW,
                    StandardOpenOption.WRITE);
            printMicros("create-open", started);

            ByteBuffer data = ByteBuffer.allocate(64 * 1024);
            started = System.nanoTime();
            while (data.hasRemaining()) {
                channel.write(data);
            }
            printMicros("write-64k", started);

            started = System.nanoTime();
            channel.force(true);
            printMicros("force-metadata", started);

            started = System.nanoTime();
            channel.close();
            channel = null;
            printMicros("close", started);

            started = System.nanoTime();
            try {
                Files.move(
                        temporary,
                        completed,
                        StandardCopyOption.ATOMIC_MOVE);
                printMicros("atomic-move", started);
            } catch (AtomicMoveNotSupportedException exception) {
                System.out.println("atomic-move=unsupported");
                throw exception;
            }

            started = System.nanoTime();
            Files.delete(completed);
            printMicros("delete", started);
        } finally {
            if (channel != null) {
                channel.close();
            }
            Files.deleteIfExists(temporary);
            Files.deleteIfExists(completed);
            Files.delete(runDirectory);
        }
    }

    private static void printMicros(String operation, long started) {
        long elapsed = System.nanoTime() - started;
        System.out.printf(
                "operation=%s latency_us=%d%n",
                operation,
                elapsed / 1_000L);
    }
}
```

局限：

- `System.nanoTime()` 适合计算单 JVM 内耗时，不用于跨主机时间关联；
- 一次 64 KiB 写不能形成分位分布；
- `force(true)` 的底层稳定性仍由 NFS 版本、服务端和后端保证；
- `ATOMIC_MOVE` 成功表示命名空间原子性，不表示目录项已经跨断电持久化；
- `createTempDirectory` 在测试父目录内原子创建本次运行的独占目录；探针只清理目录内两个已知对象，并以非递归 `Files.delete` 删除空目录；
- finally 清理失败必须被测试框架记录，不能吞掉；
- 正式压测需要 warm-up、固定到达率、直方图和独立采集线程，避免日志反向影响结果。

### 5.3 JFR 与 JVM 证据

**执行端：Java 应用节点**  
**风险：JFR 可能记录路径、类名和调用栈；使用受控文件、持续时间和保留策略**

```bash
: "${JAVA_PID:?set target Java PID}"
JFR_FILE=/var/tmp/nfs-kb-l4-01.jfr

jcmd "$JAVA_PID" VM.version
jcmd "$JAVA_PID" JFR.start \
  name=nfs-kb-l4-01 settings=profile duration=60s \
  filename="$JFR_FILE"
jcmd "$JAVA_PID" JFR.check
```

JFR File Read/Write、Socket、Thread Park、Java Monitor 和 CPU 事件是否启用及阈值依 JDK/settings 变化。JFR 能帮助回答线程在哪里等待，但不会自动标注某次 File Write 对应哪个 NFS RPC。必须使用时间、PID/TID、路径脱敏标识、应用 operation id 和系统指标关联。

### 5.4 应用 SLI 设计

推荐按业务语义埋点：

```text
nfs_file_create_latency_seconds{result,workload_class}
nfs_file_write_bytes_total{workload_class}
nfs_file_force_latency_seconds{result,workload_class}
nfs_file_read_latency_seconds{result,size_bucket,cache_scenario}
nfs_file_rename_latency_seconds{result,workload_class}
nfs_file_operation_errors_total{operation,exception_class}
nfs_file_inflight_operations{operation,workload_class}
```

禁止把真实路径、用户名、租户 ID、文件 ID 放入 metrics label。高基数对象通过受控日志、trace id 或采样事件关联。

Java 端必须同时记录：

- 实际运行 UID/GID、容器和节点；
- mount target 的稳定标识；
- 操作类型和字节数；
- 是否调用 force/fsync；
- 异常类型、errno 映射和超时阶段；
- in-flight、线程池/虚拟线程等待和调用方取消；
- 业务 deadline，而不是无限等待后统一报慢。

生产通常使用 `hard` mount 保障 NFS I/O 在暂时故障后继续重试。在这种语义下，调用方 deadline 到期、`Future.cancel(true)`、线程中断或虚拟线程取消，不保证已经进入内核的 NFS 文件 I/O 立即终止；线程可能继续阻塞，RPC 也可能在调用方已经返回超时结果后继续恢复或完成，具体行为依系统调用、内核和挂载参数而异。因此必须分别记录：

- 调用方观察到的 deadline 超时和取消；
- 实际文件 I/O worker 的在途数、阻塞时长、晚完成和晚失败；
- 超时后是否触发了新的重试请求，以及是否形成重复写、覆盖或临时文件泄漏；
- 有界准入、隔离执行器和故障恢复期间的最大堆积，而不是把线程中断当作可靠的 NFS I/O 终止机制。

### 5.5 常见 Java 性能误判

| 现象 | 可能真实原因 |
| --- | --- |
| `write()` 很快、`close()` 偶尔慢 | 缓冲和 writeback 在 close/flush 阶段兑现 |
| `force()` p99 高 | COMMIT、服务端稳定写、后端 flush 或复制长尾 |
| read 第二次极快 | 客户端 page cache，不是 NFS 带宽提升 |
| 增加线程后吞吐不增 | RPC slot、网络、nfsd、元数据锁或存储饱和 |
| GC 正常但接口卡住 | 线程阻塞在文件系统系统调用 |
| mmap 没有 read 延迟 | 延迟转移到 page fault 或后台 writeback |
| `Files.exists` 很多 | GETATTR/LOOKUP/ACCESS 元数据放大 |
| 失败后高频重试 | 对拥塞或 HA 恢复形成负反馈风暴 |

## 6. 生产基线、SLI/SLO 与容量模型

### 6.1 SLI 分层

| 类别 | 核心 SLI | 用途 |
| --- | --- | --- |
| 用户结果 | 文件事务成功率、deadline 超时率 | 最终可用性 |
| 应用延迟 | create/read/write/force/rename p50/p95/p99 | 用户体验与长尾 |
| 应用负载 | files/s、bytes/s、in-flight、并发客户端 | 到达率和需求 |
| NFS | operation rate、kB/op、avg RTT、avg exe、retrans | 协议层定位 |
| 网络 | 每方向带宽、传输层 retrans/error、drops、RTT/cwnd/QP | 传输能力 |
| 服务端 | nfsd ops、CPU、run queue、RPC errors/threads | 前端服务能力 |
| 后端 | 数据/元数据延迟、IOPS、带宽、queue、cache | 持久化能力 |
| 容量 | 字节、inode/对象、QoS、burst、headroom | 增长与饱和风险 |

SLO 应先定义业务操作。例如：

```text
在正常维护窗口外：
  64 KiB force-and-rename acknowledged transaction success >= 99.95%
  create + write + force + rename p99 <= 150 ms
  read 1 MiB warm-working-set p99 <= 40 ms
```

这些数值只是格式示例，不是通用 NFS 标准。上面的写 SLO 只表示客户端观察到 `force` 与 rename 成功应答，不能单凭该延迟指标命名为 durable transaction。若业务 SLO 要承诺崩溃或断电后的持久性，还必须同时定义并验证：

- 服务端 export 的 `sync` 语义，以及 NFS WRITE/COMMIT 到服务端 stable storage 的实现保证；
- 后端 cache、flush、屏障、复制或 EC 在目标故障模型下的持久性；
- 临时文件 `force(true)` 后，最终 rename 的命名空间更新如何持久化；Java 的 `ATOMIC_MOVE` 只保证可见性切换的原子语义，不自动保证父目录项跨断电持久；
- 服务端进程崩溃、HA 接管、控制器故障和断电恢复后，最终名称、文件长度、内容校验和与旧版本覆盖结果；
- 平台是否提供并支持目录同步或等价事务保证；若不提供，只能按存储服务公开保证定义 SLO，并通过故障注入验证，不能由一次 API 成功返回外推。

因此应把“应答延迟 SLO”和“故障恢复正确性 SLO”分开统计。后者至少报告恢复场景、样本数、丢失/旧版本/双版本结果和恢复时间。

### 6.2 基线矩阵

至少建立：

| 维度 | 基线组合 |
| --- | --- |
| 客户端数 | 1、典型、峰值、故障重连规模 |
| 并发 | 1、逐级增长、峰值、过载停止点 |
| 数据 | 大文件顺序读写、小文件事务、元数据密集 |
| 缓存 | 首次读取、暖缓存、稳定工作集 |
| 持久性 | buffered、close sync、每文件 force、每批次 force |
| 安全 | `sec=sys` 与生产实际 `krb5*` 分别测量 |
| 节点 | 每个 HA 前端、每条网络路径、每个存储池 |
| 时间 | 正常、备份/快照、重建、业务高峰 |

基线报告同时展示吞吐、分位延迟、错误和六层资源指标。只保存峰值吞吐没有诊断价值。

### 6.3 需求模型

定义：

```text
C                 客户端或应用实例数
R_read            每客户端应用读操作/s
R_write           每客户端应用写操作/s
S_read/S_write    每次应用有效字节数
M_read            客户端读 cache miss ratio
F_write           写合并/放大后的协议字节系数
O_wire            协议、安全封装和重传系数
```

初步网络需求：

```text
server_tx_read_bytes/s
  ~= C * R_read * S_read * M_read * O_wire

server_rx_write_bytes/s
  ~= C * R_write * S_write * F_write * O_wire
```

注意全双工链路要分别检查接收和发送方向，不能只把两者相加后与线速比较。

RPC 请求速率需要基于实测拆分/合并系数：

```text
read_RPC_ops/s
  ~= application_read_bytes/s / observed_READ_kB_per_op

write_RPC_ops/s
  ~= application_write_bytes/s / observed_WRITE_kB_per_op
```

元数据需求单独建模：

```text
file_transactions/s
  * observed {LOOKUP, GETATTR, OPEN, CLOSE, CREATE, RENAME, REMOVE} per transaction
```

不要从应用调用次数猜协议比例；用受控基线中的 counter delta 校准。

### 6.4 容量示例

假设：

```text
200 个应用实例
每实例 50 次/s 读取 128 KiB，客户端 miss ratio 30%
每实例 20 次/s 写入 64 KiB，写协议字节系数 1.0
协议、安全和重传预留系数 1.10
```

估算：

```text
read  = 200 * 50 * 128 KiB * 0.30 = 375 MiB/s
write = 200 * 20 *  64 KiB * 1.00 = 250 MiB/s

server send requirement    ~= 375 * 1.10 = 412.5 MiB/s
server receive requirement ~= 250 * 1.10 = 275.0 MiB/s
```

这只是网络入口估算，还没有包含：

- force/COMMIT 和元数据 RPC；
- 后端写放大、复制、EC、快照；
- NAS cache 命中；
- 峰值同时性与重连风暴；
- HA 单节点承载全部流量；
- CPU 加密、校验和及协议处理；
- 文件数、目录和 inode 容量。

后端 IOPS 必须用实测 request size、cache 和写放大校准，不能简单用应用字节数除以 4 KiB。

### 6.5 pNFS 的 MDS/DS 分离容量模型

pNFS 不能沿用“所有 READ/WRITE 都经过一个 NFS server”的入口模型。客户端先经元数据服务器（MDS）获得 layout，随后符合 layout 的数据 I/O 可直接访问一个或多个数据服务器（DS）；元数据、状态、layout 获取/归还/提交仍涉及 MDS。MDS 与 DS 可能物理共址，容量报告仍要先按角色拆开，再映射回共享 CPU、网络和后端资源，避免重复计算或漏算。

定义实测比例：

```text
P_layout_read/write   通过有效 layout 到 DS 的应用数据字节比例
P_mds_read/write      通过 MDS 数据路径的比例，约等于 1 - P_layout_*
D_read_i/D_write_i    第 i 个 DS 承担的 DS 数据字节份额，各方向分别求和为 1
O_mds/O_ds_i          MDS 与各 DS 路径实测的协议、安全和重传系数
```

容量按端点和方向展开：

```text
MDS_send_read_bytes/s
  ~= application_read_bytes/s * P_mds_read * O_mds

MDS_receive_write_bytes/s
  ~= application_write_bytes/s * P_mds_write * O_mds

DS_i_send_read_bytes/s
  ~= application_read_bytes/s * P_layout_read * D_read_i * O_ds_i

DS_i_receive_write_bytes/s
  ~= application_write_bytes/s * P_layout_write * D_write_i * O_ds_i

MDS_metadata_ops/s
  ~= business metadata ops
     + observed LAYOUTGET/LAYOUTRETURN/LAYOUTCOMMIT and state ops
     + recall/recovery/fallback amplification
```

`P_layout_*` 和 `D_*_i` 必须来自基线中的客户端 trace/mountstats、各端点网络计数与存储厂商遥测，不能假设平均条带。文件 layout 的 stripe width/stripe unit、文件大小、访问偏移、热点文件、DS 可用性和多路径策略都会造成倾斜。连接或路径数量也不是容量乘数；应分别验证每个 DS、每条路径以及共享上联的 SLO 拐点。

Linux pNFS 客户端还可能根据服务端在 OPEN 阶段提供的 `mdsthreshold` hint 决定小文件或低 I/O 对象继续走 MDS。该值是协议/实现提供的阈值提示，不是可以假定存在的通用 mount 参数；其暴露方式和实际执行依内核、layout driver 与服务端实现变化。容量测试必须覆盖阈值两侧的文件大小和访问量，并用实际 MDS/DS 字节确认路径。

至少单独验证以下状态：

- 正常 layout 命中时 MDS 元数据能力与每个 DS 的读写能力；
- layout 未授予、过期或被 recall 时的 `LAYOUT*` 突发和业务停顿；
- DS/路径故障、layout recovery，以及实现允许时数据 I/O 回到 MDS 路径后的 MDS 网络与后端容量；
- 单 DS、单 stripe 或单热文件倾斜，而不是只看所有 DS 聚合吞吐；
- MDS 与 DS 共址、共享上联或共享后端池时的物理资源最小值。

只读取证可先保留实际挂载、完整 mountstats、NFSv4 operation delta、SUNRPC/NFS tracepoint 列表、MDS/DS socket 和厂商端点统计。Linux 工具是否展示 DS endpoint、layout 与 `mdsthreshold` 字段依版本变化；没有稳定字段时必须依赖目标内核文档和厂商遥测，不得把 2049 聚合流量全部标成 MDS 流量。

### 6.6 多资源最小值

系统可交付能力近似受最小瓶颈约束：

```text
Capacity = min(
  client concurrency and RPC slots,
  network per-direction bandwidth and packet rate,
  pNFS MDS metadata/fallback and each DS/path capacity,
  server CPU and nfsd concurrency,
  backend data throughput/IOPS,
  backend metadata transactions,
  storage QoS and tenant limits,
  HA degraded-mode capacity
)
```

某层尚有余量不代表系统有余量。10 GbE 未跑满时，元数据节点可能已经饱和；阵列 IOPS 未满时，单热目录锁可能已经成为瓶颈。

### 6.7 余量与降级模式

容量规划必须针对故障状态：

```text
required_capacity
  = peak_business_demand
  * burst_factor
  * growth_factor
  * retry_or_failover_factor

provisioned_capacity
  >= required_capacity / target_max_utilization
```

`target_max_utilization` 不能统一设为 100%。需要根据延迟曲线拐点、恢复时间、扩容周期和业务 SLO 确定。

至少验证：

- 一个 HA 前端失效后剩余节点；
- 一条网络链路或一个存储控制器失效；
- 后端 rebuild/scrub/快照期间；
- Kerberos/KDC 或 DNS 延迟；
- 客户端批量重连与 cache 重新预热；
- 备份、灾备复制和生命周期任务并发。

### 6.8 并发曲线与拐点

正确容量图不是单点：

```text
concurrency: 1 -> 2 -> 4 -> 8 -> 16 -> ...

for each point:
  arrival rate
  completion rate
  p50/p95/p99
  errors/timeouts
  avg RTT/avg exe/retrans
  server CPU/run queue
  backend latency/queue
```

典型拐点：吞吐增幅开始变小，但 p99 和队列快速上升。生产最大负载应低于该拐点并保留故障余量，不应把压测中勉强维持的最大吞吐当可承诺容量。

### 6.9 安全 flavor 成本

`sec=sys`、`krb5`、`krb5i`、`krb5p` 的 CPU、payload 和可观测性不同。生产使用 `krb5p` 时，容量认证必须使用 `krb5p`；不能用 `sec=sys` 测得的吞吐替代。

同时观察：

- 客户端与服务端 CPU cycles/byte；
- GSS context 建立、续期和认证刷新；
- 小 I/O 下加密固定成本；
- 大 I/O 下内存带宽和 crypto 吞吐；
- 抓包因 privacy 无法解码的取证替代方案。

不得为性能绕过安全要求降级 flavor。

### 6.10 稳定性与性能不能交换概念

以下变更可能提高表面吞吐，但改变数据安全或失败语义：

- 服务端 export 从 `sync` 改为 `async`；
- 应用移除 `fsync/force`；
- 客户端从 `hard` 改为 `soft`；
- 缩短超时并在应用高频重试；
- 关闭 Kerberos integrity/privacy；
- 放宽 cache 一致性参数。

性能验收必须保持业务要求的持久性、错误和安全语义。否则测试的是另一个系统。

### 6.11 多租户与公平性

总吞吐正常时，单租户仍可能被饿死。需要按不会造成高基数的稳定 workload class 观察：

- 吞吐和分位延迟；
- in-flight 和队列；
- QoS throttle；
- 热目录、热文件和锁；
- 客户端连接与 source IP 分布；
- 后端卷/池/节点归属。

容量测试应包含一个租户突发时其他租户 SLO，而不只是聚合 MB/s。

### 6.12 变更治理

调优变更记录：

```text
change id
  -> hypothesis
  -> one changed variable
  -> exact old/new value and effective scope
  -> workload and baseline id
  -> expected metric movement
  -> stop conditions
  -> rollback command/config
  -> observed result and side effects
```

需要单独评估的变量包括 `rsize/wsize`、`nconnect`、attribute cache、readahead、RPC slots、nfsd threads、TCP buffer、NIC queue、IRQ affinity、存储 queue/QoS。本文不把任何值作为无条件推荐。

## 7. 验证实验与观察指标

所有实验只在隔离 NFS 导出、专用测试客户端和明确容量配额下执行。当前文档未在真实 Linux/NFS/NAS 环境验证，因此保持“待验证”。

### 7.1 实验零：记录拓扑与变量

| 项目 | 必填值 |
| --- | --- |
| 客户端 | 数量、OS、内核、CPU、内存、NIC、AZ/机架 |
| 服务端 | knfsd/NAS/云服务、版本、前端节点、HA 模式；pNFS 的 MDS/DS 角色与共址关系 |
| 后端 | 文件系统、介质、RAID/EC/复制、cache、QoS |
| NFS | version、minor version、transport、rsize/wsize、`sec=` |
| 网络 | RTT、带宽、MTU、bond/VLAN、交换与防火墙路径 |
| 数据 | 文件数、文件大小、目录宽度、工作集、热度 |
| 负载 | 操作、读写比、同步语义、并发和到达率 |
| 安全 | AUTH_SYS/Kerberos、SELinux、ACL |
| 时间 | 时区、测试窗口、备份/快照/重建状态 |

变量未记录完整时，不发布性能结论。

### 7.2 实验一：空闲基线

**执行端：两端**  
**适用范围：只读取证，无压测流量**

```bash
date --iso-8601=seconds
nfsstat -m
nfsiostat 1 10
vmstat 1 10
sar -n DEV,TCP,ETCP 1 10
```

服务端同时执行：

```bash
nfsstat --server
mpstat -P ALL 1 10
iostat -xz 1 10
```

预期：建立无业务负载时的 counter 增量、后台任务、网络、CPU 和存储噪声。若空闲期已有高 WRITE/COMMIT、重传或存储队列，先解释后台负载再开始实验。

### 7.3 实验二：缓存路径对照

**执行端：隔离客户端**  
**风险：读取专用测试文件；不清理全机 cache**

```bash
TEST_FILE=/mnt/secure/.nfs-kb-l4-01/read-working-set.bin

stat -c 'size=%s path=%n' "$TEST_FILE"

for round in 1 2 3; do
  printf 'round=%s start=%s\n' "$round" "$(date --iso-8601=seconds)"
  dd if="$TEST_FILE" of=/dev/null bs=1M status=progress
done
```

并行采集 `nfsiostat`、接口字节、服务端 NFS READ 与存储读指标。比较：

- 应用读取字节；
- NFS READ kB/s 和 operation delta；
- 网络 server-to-client 字节；
- 后端读取；
- 每轮耗时。

如果第二轮没有 READ RPC，说明主要命中客户端 cache；如果仍有 NFS READ 但后端读低，可能命中服务端/存储 cache。

### 7.4 实验三：buffered write、force/rename 应答与故障恢复

使用第 5.2 节 Java 探针或经过评审的 fio profile，分别测量：

1. write 后立即返回，不调用 `force`；
2. 每文件 `force(true)`；
3. 每 N 个文件批量同步；
4. close 与 rename；
5. 相同有效字节和文件数量。

观察：

| 层 | 指标 |
| --- | --- |
| Java | write/force/close/rename p50/p95/p99 |
| 客户端 | Dirty、Writeback、NFS_Unstable、WRITE/COMMIT |
| RPC | kB/op、avg RTT、avg exe、retrans |
| 服务端 | WRITE/COMMIT、CPU、active/queue 线索 |
| 后端 | write latency、flush、cache、复制、queue |

不得把无 `force` 的结果与 force/rename 应答 SLO 直接比较，也不得把 `force` 和 rename 成功直接当作断电持久性证明。若验收 durable SLO，还要在批准的故障注入环境验证第 6.1 节定义的 export、stable storage、最终命名空间和 HA/崩溃恢复条件。

### 7.5 实验四：元数据事务

**执行端：隔离客户端**  
**风险：在测试根目录内原子创建本次运行的独占目录；必须限制数量并精确清理**

```bash
set -euo pipefail

TEST_ROOT=/mnt/secure/.nfs-kb-l4-01
COUNT=1000
test -d "$TEST_ROOT"
test ! -L "$TEST_ROOT"
command -v perf >/dev/null

umask 077
RUN_DIR=$(mktemp -d -- "$TEST_ROOT/metadata.XXXXXXXXXX")

cleanup_metadata_test() {
  rc=0
  for ((i = 1; i <= COUNT; i++)); do
    file=$(printf '%s/file-%06d' "$RUN_DIR" "$i")
    rm -f -- "$file" || rc=1
  done
  rmdir -- "$RUN_DIR" || rc=1
  if (( rc != 0 )); then
    printf 'metadata cleanup incomplete: %s\n' "$RUN_DIR" >&2
  fi
  return "$rc"
}
trap cleanup_metadata_test EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

export RUN_DIR COUNT

printf 'phase=create files=%s\n' "$COUNT"
perf stat -x ';' -- bash -c '
  set -euo pipefail
  for ((i = 1; i <= COUNT; i++)); do
    printf -v file "%s/file-%06d" "$RUN_DIR" "$i"
    (set -o noclobber; : > "$file")
  done
'

printf 'phase=stat-warm-cache files=%s\n' "$COUNT"
perf stat -x ';' -- bash -c '
  set -euo pipefail
  for ((i = 1; i <= COUNT; i++)); do
    printf -v file "%s/file-%06d" "$RUN_DIR" "$i"
    stat "$file" >/dev/null
  done
'

cleanup_metadata_test
trap - EXIT INT TERM
```

`perf stat` 的 elapsed time 使用单调时钟来源，避免 NTP/人工校时造成 wall clock 跳变；输出按秒解释，不把结果伪装成纳秒精度。该 shell 循环仍包含 Bash、`printf`、`stat` 和 `perf` 开销，只用于功能与指标映射，不用于发布高精度 files/s。create 后立即执行的 stat 是明确的暖 dentry/attribute-cache 场景，不能代表冷查找；正式元数据基准应使用受支持工具或专用单调时钟程序，分别记录 cold/warm 条件及 create/stat/remove 分位延迟。

`mktemp -d` 原子创建独占目录，`noclobber` 阻止覆盖已有对象。清理只枚举本次目录内的确定文件名并使用 `rmdir` 删除空目录；出现未知对象时保留现场，不做递归或通配删除。

重点比较 NFS operation delta，而不是只看 shell 总耗时。

### 7.6 实验五：并发曲线

按 `1、2、4、8、16...` 客户端或 worker 逐级增加，每级满足：

- 工作负载和数据集固定；
- 预热规则固定；
- 到达率/闭环模型明确；
- 运行时间覆盖稳态；
- 每级之间恢复到可比较状态；
- 六层指标同时采集。

停止条件：

- 达到业务安全上限；
- p99 超过 SLO；
- 错误/超时出现；
- 网络、CPU、RPC、后端或 QoS 达到已批准阈值；
- 吞吐不再增长但队列持续上升；
- 影响非测试租户。

容量结论取拐点之前满足 SLO 的最高稳态档位，再扣除 HA、增长和突发余量。

### 7.7 实验六：安全 flavor 对照

在安全团队批准的隔离环境，以完全相同 workload 分别测量允许使用的 flavor。记录：

- 应用吞吐和分位；
- 客户端/服务端 CPU；
- operation 与 bytes；
- GSS context/认证刷新；
- 网络字节与 MTU/分片线索；
- 抓包可观测性差异。

对照的目的，是为生产 flavor 规划容量，不是选择更弱的 flavor。

### 7.8 实验七：Java 与 RPC 时间关联

1. 为每轮生成唯一 operation id，并以固定 workload 参数执行第 5.2 节探针；
2. 记录单调时钟延迟和 wall-clock 开始/结束时间；
3. 同时采集 mountstats interval、JFR、实际 TCP/RDMA/UDP 传输、服务端和存储指标；
4. 对比 force 长尾是否伴随 COMMIT、avg RTT、avg exe 或后端 flush 上升；
5. 对比 open/rename 长尾是否主要伴随元数据 operation；
6. 在另一客户端重复，排除单客户端 CPU、cache 或网络路径。

不能根据时间相近就直接判定因果；应通过改变一个变量或复现实验验证。

### 7.9 实验清理

**执行端：客户端与服务端**  
**风险：这里只列出本文命名规则下的候选独占目录和结果目录；禁止对业务目录递归通配删除**

```bash
find /var/tmp -maxdepth 1 -user "$(id -u)" \
  -name 'nfs-kb-l4-01-fio.*' -type d -ls

find /mnt/secure/.nfs-kb-l4-01 -maxdepth 1 -type d \
  \( -name 'fio.*' -o \
     -name 'metadata.*' -o \
     -name '.nfs-kb-l4-01-java-*' \) -ls
```

上面的 `find` 只列出候选项。逐一核对所有者、路径、测试记录和是否仍被使用后，按脚本保存的精确变量清理；不得把 `find` 输出直接通过管道交给 `rm`。压测结果、JFR、mountstats 和 ACL/身份信息按敏感证据保留或销毁。

### 7.10 必须归档的结果

| 维度 | 结果 |
| --- | --- |
| workload | 操作、大小、并发、到达率、缓存、同步语义 |
| application | 吞吐、p50/p95/p99/max、样本、错误 |
| NFS | operation delta、kB/op、RTT、exe、retrans |
| network | 每方向 bytes/s、retrans、drops、RTT/cwnd |
| client | CPU、PSI、dirty/writeback、in-flight |
| server | CPU、run queue、nfsd、RPC errors |
| backend | 数据/元数据延迟、IOPS、MB/s、queue、cache |
| capacity | 拐点、SLO 档位、headroom、故障模式 |
| evidence | 原始文件、版本、时间、hash、保留位置 |

## 8. 性能证据链与检查清单

### 8.1 先回答十五个问题

1. 慢的是 read、write、force、open、close、lock、rename 还是完整文件事务？
2. 应用 p99、错误率和 in-flight 如何变化？
3. 工作负载大小、并发、客户端数和缓存状态是否变化？
4. 实际 NFS version、`sec=`、rsize/wsize 和 mount options 是否变化？
5. READ/WRITE/COMMIT 与元数据 operation rate 如何变化？
6. avg RTT 和 avg exe 哪个先升高？差值是否扩大？
7. RPC retrans 是否在相同挂载和时间窗上升？
8. 实际 MDS/DS 传输端点是否有 drops/retrans/error？默认检查 2049/TCP；RDMA、NFSv3 UDP 与辅助 RPC 走对应证据链。
9. 客户端 CPU、memory、dirty/writeback、PSI 是否饱和？
10. 服务端 CPU、run queue、nfsd concurrency 或 RPC error 是否变化？
11. 后端数据和元数据延迟、queue、cache、QoS 是否变化？
12. 是否有快照、备份、重建、复制或分层后台任务？
13. 是否只影响某客户端、租户、目录、卷、HA 节点或安全 flavor？
14. DNS、KDC、LDAP/AD、证书或票据续期是否进入关键路径？
15. 最近有哪些应用、内核、挂载、网络、存储或安全变更？

### 8.2 标准采集顺序

```text
freeze changes and identify exact time window
  -> capture application SLI and workload fingerprint
  -> capture mount/version/security and mountstats interval
  -> capture client CPU/cache/PSI and actual MDS/DS transports
  -> capture server nfsd/CPU/network
  -> capture backend data/metadata/QoS/background jobs
  -> compare healthy client/node/volume
  -> form one bottleneck hypothesis
  -> change one variable in controlled environment
  -> reproduce and validate rollback
```

### 8.3 现象到证据映射

| 现象 | 优先证据 | 不应立即执行 |
| --- | --- | --- |
| 应用慢，NFS 指标低 | page cache、锁、权限、DNS/KDC、采集 namespace | 调大 rsize |
| avg exe 高、avg RTT 相对低 | 客户端 RPC queue、CPU、slots、调度 | 增加应用线程 |
| avg RTT 与后端 await 同升 | 服务端/后端 queue、flush、QoS | 修改客户端 TCP buffer |
| retrans 升高 | 实际 MDS/DS 传输、NIC/RDMA/UDP 错误、交换/云网络、服务端过载 | 提高 `retrans` 掩盖 |
| WRITE 快、force 慢 | COMMIT、stable write、后端 flush/复制 | 删除 force |
| 大文件快、小文件慢 | 元数据 op、目录热点、OPEN/CLOSE/LOOKUP | 只扩网络带宽 |
| 单客户端慢 | 客户端 CPU/cache/network/path/namespace | 重启 NFS 服务端 |
| 单 HA 节点慢 | 前端配置、路由、后端归属、firmware | 反复切换 |
| CPU 满、带宽低 | 小 I/O、krb5p、copy、softirq、metadata | 认定没有负载 |
| %util 低、await 高 | 多队列语义、QoS、远端后端、平均值 | 只看 `%util` 下结论 |
| p99 高、平均正常 | 排队、后台任务、GC/调度、尾延迟 | 只扩大采样窗口 |

### 8.4 avg RTT 与 avg exe 四象限

| avg RTT | avg exe | 解释方向 |
| --- | --- | --- |
| 低 | 低 | RPC 路径健康；继续看应用缓存/非 RPC 阶段 |
| 高 | 接近 RTT | 网络、服务端或后端服务时间主导 |
| 低 | 明显更高 | 客户端排队、slot、CPU/调度线索 |
| 高 | 比 RTT 更高很多 | 客户端排队与远端服务时间可能同时存在 |

这是平均值诊断线索，不是根因证明。必须结合应用分位、retrans、服务端和后端证据。

### 8.5 重传分析

```text
RPC retrans rises
  -> confirm counter delta and denominator
  -> identify exact mount/client/server/time
  -> inspect actual TCP/RDMA/UDP MDS/DS transport state
  -> inspect NIC/host/switch/cloud drops
  -> inspect server overload and delayed replies
  -> compare another client/path
  -> distinguish packet loss from slow response/recovery
```

RPC 重传不是“网络一定丢包”的同义词。服务端长时间不响应、连接重建、故障切换和客户端超时策略也可能参与。

### 8.6 容量不足判定

容量不足至少需要：

1. 需求或到达率接近/超过已验证 envelope；
2. 某个资源出现稳定饱和或排队；
3. 吞吐增幅下降而延迟/错误快速上升；
4. 降低负载后指标按预期恢复；
5. 扩大目标瓶颈资源后拐点移动；
6. 排除异常、配置漂移和后台任务。

只有“高峰时慢”不足以证明需要扩容，也可能是限额、热点、重传或周期任务。

### 8.7 生产仪表盘最小集合

```text
row 1: business files/s, bytes/s, success, p50/p95/p99, in-flight
row 2: NFS READ/WRITE/COMMIT/metadata ops, kB/op, avg RTT/exe, retrans
row 3: client CPU/PSI/dirty/writeback and actual MDS/DS transport/NIC
row 4: server CPU/run queue/nfsd/RPC errors/network
row 5: backend data+metadata latency/IOPS/throughput/queue/cache/QoS
row 6: capacity bytes/inodes, HA state, backup/snapshot/rebuild events
```

所有图使用相同时区和对齐的时间范围，并标注部署、配置、HA 和后台任务事件。

### 8.8 告警设计

推荐组合告警，而不是单指标告警：

```text
user-impact alert
  = application p99/error violates SLO
  AND sufficient sample count

network suspicion
  = retrans ratio above validated baseline
  AND matching TCP/RDMA/UDP transport and NIC evidence

backend saturation
  = application or RPC latency rises
  AND backend latency/queue/QoS evidence
```

需要避免：

- 对启动以来累计 retrans 直接告警；
- 将 `avg RTT > 固定值` 作为所有场景统一阈值；
- 没有业务流量时因分母接近零产生比率尖峰；
- 对每文件或每用户建立 metrics label；
- 只报警资源，不报警用户结果；
- 告警触发后自动执行破坏性调优。

### 8.9 生产检查清单

- [ ] 每类关键业务文件操作都有成功率、分位延迟、吞吐和样本量。
- [ ] 基线明确缓存、同步、协议、安全和并发条件。
- [ ] mountstats/nfsstat 使用区间增量而不是累计值直接比较。
- [ ] 已区分业务事务、NFS operation、RPC 和后端 I/O。
- [ ] 已建立客户端 avg RTT/avg exe/retrans 观测。
- [ ] 已建立实际 MDS/DS 传输端点、NIC、交换/云网络证据链，并覆盖 TCP、RDMA 或 NFSv3 UDP/辅助 RPC 的实际分支。
- [ ] 已建立服务端 nfsd、CPU、run queue 和 RPC error 指标。
- [ ] 后端数据、元数据、cache、QoS 和后台任务均可观测。
- [ ] Java `force`、close、rename 与总业务耗时分开计量。
- [ ] 容量模型覆盖读写方向、元数据、RPC 并发和后端写放大。
- [ ] pNFS 容量模型分别覆盖 MDS、每个 DS、layout/条带倾斜、`mdsthreshold`、recall、回退路径和共享物理资源。
- [ ] RPC 容量结论区分配置上限、NFSv4.1+ 协商 slot、当前 pending/backlog 和 `nconnect` 每连接/总计证据。
- [ ] 容量结论来自并发曲线拐点之前满足 SLO 的稳态档位。
- [ ] 已验证 HA 降级、重连风暴、备份和重建场景。
- [ ] 生产 security flavor 已纳入容量认证。
- [ ] 调优一次只改变一个变量，具备停止条件和回滚。
- [ ] 禁止用 drop_caches、soft mount、async export 或安全降级制造好看结果。
- [ ] 原始证据带版本、时间、拓扑、hash 和敏感数据保留策略。

## 9. 小结

1. NFS 性能必须从应用、客户端、RPC、网络、服务端和后端六层同时解释。
2. 应用吞吐、NFS ops/s、RPC 数和后端 IOPS 不是同一个指标。
3. avg RTT、avg exe 和两者差值只能提供平均排队线索，不能替代应用 p99。
4. 缓存、同步语义、请求大小、并发、文件模型、协议和安全 flavor 决定工作负载指纹。
5. Little's Law 将完成率、延迟和在途请求关联起来，可用于检查并发与 slot 需求。
6. 数据面与元数据面必须分别建立基线；大文件吞吐不能代表小文件事务能力。
7. 容量是客户端并发、网络、服务端、后端、QoS 和 HA 降级能力的最小值。
8. pNFS 必须拆分 MDS、每个 DS、layout 与回退路径；聚合吞吐不能证明所有条带和故障模式都有余量。
9. 最大压测吞吐不是生产容量；应取满足 SLO 的拐点前稳态能力并保留余量。
10. Java 必须分别观测 open、write、force、close、rename、delete、调用方 deadline 和实际在途 I/O；缓冲 flush 不等于持久化，取消也不保证终止 hard mount I/O。
11. 调优必须以可验证假设驱动，不能通过改变安全、持久性或错误语义换取表面性能。

## 10. 参考资料与关联文档

### 10.1 参考资料

- RFC 1813 第 3.3.7、3.3.21 节：NFSv3 WRITE、COMMIT 与稳定存储语义
- RFC 7530：NFSv4.0 operation、COMPOUND、缓存和恢复语义
- RFC 8881：NFSv4.1 session、slot、operation 与 pNFS 语义
- RFC 7862：NFSv4.2 扩展能力
- `nfs(5)`、`nfsiostat(8)`、`mountstats(8)`、`nfsstat(8)`：Linux NFS 挂载和客户端/服务端统计
- Linux 内核 NFS client、server、SUNRPC tracepoint 与 `/proc/net/rpc/*` 文档：实现级指标边界
- `iostat(1)`、`sar(1)`、`pidstat(1)`、`mpstat(1)`：sysstat 主机与块设备统计
- `ss(8)`、`ip(8)`、`ethtool(8)`、`nstat(8)`：Linux TCP 与网络接口统计
- fio 官方文档：workload、I/O engine、同步和 latency percentile 语义
- Prometheus node_exporter NFS/NFSd collector 文档：Linux NFS exporter 指标
- OpenJDK JFR、`jcmd`、`FileChannel` 与 `Files` API 文档：Java 文件 I/O 和运行时观测
- 目标 NAS/云文件服务的性能、QoS、cache、HA、计费和监控官方文档

### 10.2 关联文档

- [NFS-KB-L0-01 端到端 I/O 路径](../L0-foundation/NFS-KB-L0-01-end-to-end-io.md)
- [NFS-KB-L0-02 SUNRPC、XDR 与请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)
- [NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)
- [NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)
- [NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)
- [NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](../L3-security/NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)
- [NFS-KB-L3-03 NFSv4 ACL、SELinux 与多协议权限集成](../L3-security/NFS-KB-L3-03-nfsv4-acl-selinux-and-multiprotocol-permissions.md)
- 待建立：`NFS-KB-L4-02` NFS 性能基准测试与压测方法
- 待建立：`NFS-KB-L4-03` 客户端、服务端、网络与存储调优
- 待建立：`NFS-KB-L5-01` NFS 性能骤降与长尾延迟排障

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-03 | 1.0.1 | 修复实验路径竞态、低流量重传率、持久性 SLO、slot/传输取证与元数据计时；补全 pNFS 容量模型和 Java hard mount deadline 边界 | 文档复核发现固定文件 TOCTOU、单 server 容量假设、`force + rename` 持久性外推及跨协议证据链不完整 |
| 2026-08-03 | 1.0.0 | 初始发布 | 建立 NFS 六层性能模型、指标字典、Java SLI、生产基线、容量计算、验证实验和性能证据链 |
