# NFS-KB-L2-01 服务端导出与客户端挂载生产基线

> 文档状态：待验证  
> 知识阶段：L2 部署与配置  
> 适用范围：Linux NFS 服务端与客户端；RHEL/Rocky/AlmaLinux 8/9、Ubuntu 22.04/24.04 常见环境；NFSv3、NFSv4.0、NFSv4.1、NFSv4.2；具体参数以目标内核、`nfs-utils`、systemd、发行版文档和存储厂商兼容矩阵为准  
> 版本：1.1.4\
> 最后更新：2026-08-03\
> 前置文档：[NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)、[NFS-KB-L1-02 NFSv3 协议流程与状态边界](../L1-protocol/NFS-KB-L1-02-nfsv3-protocol-and-state-boundaries.md)、[NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)  
> 关联文档：[NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期](NFS-KB-L2-02-systemd-autofs-mount-lifecycle.md)、[NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](../L3-security/NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)、[NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](../L3-security/NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)

## 目录

- [1. 学习目标与生产边界](#1-学习目标与生产边界)
- [2. 部署决策模型](#2-部署决策模型)
- [3. 服务端导出基线](#3-服务端导出基线)
- [4. 客户端挂载基线](#4-客户端挂载基线)
- [5. systemd、fstab 与 autofs](#5-systemdfstab-与-autofs)
- [6. 网络、防火墙与 DNS](#6-网络防火墙与-dns)
- [7. 生产验证与观察指标](#7-生产验证与观察指标)
- [8. Java 工程视角](#8-java-工程视角)
- [9. 排障检查清单](#9-排障检查清单)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与生产边界

NFS 部署的核心不是“能 mount 上”，而是让导出边界、身份权限、缓存一致性、故障语义、性能参数和回滚路径都可解释、可观测、可验证。

完成本单元后，应能够：

1. 为 NFSv3、NFSv4.x 选择合适的导出路径、伪根、`fsid` 和安全边界。
2. 区分 `sync/async`、`root_squash/no_root_squash`、`subtree_check/no_subtree_check`、`secure/insecure` 的生产风险。
3. 设计客户端挂载参数，包括版本、传输、`hard/soft`、`timeo/retrans`、`rsize/wsize`、`nconnect`、缓存参数和安全参数。
4. 使用 systemd、`fstab` 或 autofs 管理挂载生命周期，避免启动顺序、网络抖动和挂载卡死放大故障。
5. 用统一命令验证导出、挂载、权限、锁、持久化和性能基线。

本文提供生产基线模板，不是所有业务的最终配置。任何参数都必须在目标 OS、内核、NFS 版本、存储实现、网络拓扑和业务 I/O 模式下验证。

## 2. 部署决策模型

### 2.1 从业务语义反推挂载策略

```text
业务需求
  -> 共享范围：单客户端 / 多客户端 / 跨集群
  -> 数据语义：只读 / 覆盖写 / 追加写 / 原子替换 / 文件锁
  -> 故障偏好：等待恢复 / 快速失败 / 降级只读
  -> 安全边界：UID/GID / Kerberos / root squash / 网络隔离
  -> 性能模型：小文件元数据 / 大文件吞吐 / 混合随机 I/O
  -> 部署参数：exports + mount options + systemd/autofs + 监控
```

不要从参数开始设计。先确认业务是否允许 I/O 长时间阻塞、是否需要多客户端互斥、是否能接受缓存可见性延迟，以及服务端故障时更偏向数据安全还是快速返回错误。

### 2.2 推荐基线矩阵

| 场景 | 推荐起点 | 风险边界 |
| --- | --- | --- |
| 新建企业共享目录 | NFSv4.1/4.2、TCP、`hard`、`sync`、`root_squash` | 需验证锁恢复、权限映射、应用线程池隔离 |
| 只读发布目录 | NFSv4.1 或 v3、`ro`、合理属性缓存 | 发布流程要使用原子替换和版本目录 |
| 老旧客户端兼容 | NFSv3、固定 mountd/statd/lockd 端口 | 防火墙、NLM/NSM、AUTH_SYS 风险更高 |
| Kubernetes RWX | CSI 明确支持的版本，通常优先 v4.1+ | Pod UID/GID、root squash、重启恢复必须实测 |
| 强身份与合规 | v4.x + RPCSEC_GSS/Kerberos | keytab、时间同步、DNS、性能开销和运维复杂度 |
| 大吞吐顺序 I/O | v4.1+，评估 `nconnect`/pNFS/大 `rsize/wsize` | 后端存储和网络可能先饱和 |

## 3. 服务端导出基线

### 3.1 `/etc/exports` 结构

NFS 导出项由导出路径、客户端匹配规则和选项组成：

```exports
/srv/nfs/app  10.10.0.0/16(rw,sync,root_squash,no_subtree_check,sec=sys)
```

基本规则：

- 客户端匹配应尽量使用明确网段、主机名或 netgroup，不使用裸 `*` 作为生产默认。
- 选项括号前不能有空格；`host(options)` 与 `host (options)` 语义不同，后者可能被解析为不同默认规则。
- 导出目录应位于清晰的文件系统边界上，便于容量、快照、配额、备份和故障隔离。
- 修改 `/etc/exports` 后必须通过 `exportfs -ra` 或等效服务管理命令加载，并用 `exportfs -v` 验证生效配置。

**执行端：服务端**  
**适用范围：Linux NFS 服务端；在测试导出验证后再应用到生产；`exportfs -ra` 会重新加载全部导出，应在变更窗口执行**

```bash
sudo exportfs -v
sudo exportfs -ra
sudo exportfs -v
sudo journalctl -u nfs-server --since "5 min ago"
```

生产环境建议先用配置管理或测试节点校验 `/etc/exports` 语法和目标客户端范围，再在变更窗口执行 `exportfs -ra`。如果只需临时新增或移除单个导出，可评估 `exportfs -o ... host:/path` 或 `exportfs -u host:/path` 等更小范围操作，但最终仍应回写配置文件并验证一致性。

### 3.2 NFSv4 伪根与 `fsid=0`

NFSv4 客户端从服务端伪根进入命名空间。常见导出结构：

```exports
/srv/nfs        10.10.0.0/16(ro,fsid=0,sync,root_squash,no_subtree_check,crossmnt,sec=sys)
/srv/nfs/app    10.10.0.0/16(rw,sync,root_squash,no_subtree_check,sec=sys)
/srv/nfs/logs   10.10.0.0/16(rw,sync,root_squash,no_subtree_check,sec=sys)
```

客户端看到的是伪根下的路径：

```bash
NFS_SERVER=nfs.example.com
sudo mount -t nfs4 -o nfsvers=4.1 "${NFS_SERVER}:/app" /mnt/app
```

`fsid=0` 的伪根通常应只承载命名空间，不直接作为业务可写目录。`crossmnt` 会让客户端穿越伪根下的本地挂载点，可能暴露超出预期的子文件系统；只有在明确需要跨挂载点导出时才启用，并用客户端实际遍历结果验证可见范围。伪根的 `sec=` 必须与子导出访问路径兼容：例如子导出使用 Kerberos 时，客户端仍需要用允许的安全 flavor 通过伪根和中间目录，否则可能在到达目标导出前被拒绝。

### 3.3 核心导出选项

| 选项 | 推荐基线 | 语义与风险 |
| --- | --- | --- |
| `rw`/`ro` | 按业务最小权限 | 发布目录优先 `ro`，写目录才开放 `rw` |
| `sync` | 生产默认优先 `sync` | 降低服务端异常时数据丢失风险，可能增加写延迟 |
| `async` | 仅限明确可丢失或可重建数据 | 服务端崩溃可能丢失已确认写入，需业务接受 |
| `root_squash` | 生产默认启用 | 将客户端 root 映射为匿名用户，降低横向破坏风险 |
| `no_root_squash` | 默认禁止 | 客户端 root 可在服务端以 root 语义访问，高风险 |
| `all_squash` | 多租户/匿名写入可评估 | 所有用户映射为匿名用户，需配合 `anonuid/anongid` |
| `no_subtree_check` | 常见生产默认 | 导出整个文件系统或稳定目录时减少路径检查开销和 rename 问题 |
| `subtree_check` | 仅特定子目录安全需求评估 | 可能引入 rename/文件句柄相关问题和额外开销 |
| `secure` | 默认 | 要求客户端使用特权源端口，安全价值有限但兼容传统假设 |
| `insecure` | 仅明确需要时启用 | 允许非特权源端口，常见于某些容器/NAT 场景，需网络隔离 |
| `sec=sys` | 基础环境常见 | AUTH_SYS 信任客户端 UID/GID，不提供密码学保护 |
| `sec=krb5/krb5i/krb5p` | 强身份场景评估 | 增加安全性和运维复杂度，性能需压测 |

`async`、`no_root_squash`、`insecure` 属于高风险选项。若确需使用，必须记录业务原因、隔离边界、回滚方式和验证指标，不能作为通用优化项。

### 3.4 目录、权限与匿名用户

导出参数不能替代 Linux 文件系统权限。服务端本地目录的 owner、mode、ACL、SELinux 上下文和配额仍然生效。

**执行端：服务端**

```bash
EXPORT_DIR=/srv/nfs/app
sudo mkdir -p "$EXPORT_DIR"
sudo chown root:app "$EXPORT_DIR"
sudo chmod 2770 "$EXPORT_DIR"
sudo getfacl "$EXPORT_DIR" 2>/dev/null || true
sudo ls -ldZ "$EXPORT_DIR" 2>/dev/null || sudo ls -ld "$EXPORT_DIR"
```

`root_squash` 下客户端 root 通常映射为匿名 UID/GID。若匿名写入目录未授权，会出现客户端 root 也无法写入，这是期望的安全行为，而不是 NFS 故障。

## 4. 客户端挂载基线

### 4.1 明确版本与传输

生产挂载应显式指定 NFS 版本和传输协议，避免客户端/服务端升级后默认协商变化。

```bash
NFS_SERVER=nfs.example.com
sudo mount -t nfs4 -o nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2 \
  "${NFS_SERVER}:/app" /mnt/app
```

验证实际协商：

```bash
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
cat /proc/mounts | awk '$3 == "nfs" || $3 == "nfs4"'
```

### 4.2 `hard`、`soft` 与超时

| 选项 | 行为 | 生产建议 |
| --- | --- | --- |
| `hard` | RPC 超时后持续重试，系统调用可能长期阻塞 | 数据可靠性优先场景默认评估 |
| `soft` | 达到重试条件后向应用返回错误 | 只用于应用能正确处理部分失败和重试的场景 |
| `softerr` | 与 `soft` 类似，但错误返回更偏向 `ETIMEDOUT` | 需目标内核验证 |
| `timeo` | RPC 超时基准，单位和退避受协议/实现影响 | 不按应用 HTTP 超时机械换算 |
| `retrans` | 重传次数相关参数 | 与 `hard/soft` 共同决定故障表现 |

Java 应用常把 NFS I/O 放在请求线程中；`hard` 故障可能耗尽线程池，`soft` 故障可能暴露部分写入和数据损坏风险。两者都需要应用层超时、线程池隔离和幂等策略。

### 4.3 `rsize/wsize`、`nconnect` 与缓存

| 参数 | 作用 | 注意事项 |
| --- | --- | --- |
| `rsize/wsize` | 单次 READ/WRITE RPC 目标大小 | 默认通常较合理；调大不一定提升小文件性能 |
| `nconnect` | 单挂载使用多个 TCP 连接 | 需内核和服务端支持；会改变连接、队列和故障表现 |
| `actimeo` | 属性缓存总控 | 降低可见性延迟会增加 GETATTR/LOOKUP |
| `acregmin/acregmax` | 普通文件属性缓存时间 | 影响文件大小、mtime 等可见性 |
| `acdirmin/acdirmax` | 目录属性缓存时间 | 影响目录遍历和新文件可见性 |
| `noac` | 尽量禁用属性缓存并改变写入行为 | 延迟和 RPC 放大明显，不能作为通用一致性方案 |
| `noatime` | 避免访问时间更新 | 减少部分元数据写，需业务确认不依赖 atime |

调优顺序应是先测业务 I/O 模型，再看 `nfsiostat`、`mountstats`、服务端磁盘和网络指标，最后调整参数。不要只通过增大 `rsize/wsize` 或启用 `nconnect` 解决所有慢 I/O。

### 4.4 安全挂载参数

`sec=` 必须与服务端导出匹配：

```bash
# AUTH_SYS
sudo mount -t nfs4 -o nfsvers=4.1,sec=sys nfs.example.com:/app /mnt/app

# Kerberos 认证、完整性或隐私保护，需提前完成域、keytab 和时间同步
sudo mount -t nfs4 -o nfsvers=4.1,sec=krb5i nfs.example.com:/secure /mnt/secure
```

Kerberos 相关配置属于独立安全主题；本篇只强调部署边界：DNS、正反向解析、时间同步、主体名、keytab、服务端导出 `sec=` 和客户端凭据必须整体验证。

## 5. systemd、fstab 与 autofs

### 5.1 `/etc/fstab` 基线

直接写入 `fstab` 的挂载可能在启动阶段阻塞系统。生产建议使用 systemd 网络依赖和 automount 降低启动风险：

```fstab
nfs.example.com:/app  /mnt/app  nfs4  nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2,_netdev,nofail,x-systemd.automount,x-systemd.idle-timeout=600  0  0
```

关键点：

- `_netdev` 表示网络文件系统；
- `nofail` 避免启动时因挂载失败进入紧急模式；
- `x-systemd.automount` 让访问路径时触发挂载，避免启动路径强依赖 NFS；
- `x-systemd.idle-timeout` 可在空闲后卸载，需确认业务能接受首次访问延迟。

### 5.2 systemd 验证

**执行端：客户端**

```bash
sudo systemctl daemon-reload
MOUNT_PATH=/mnt/app
AUTO_UNIT=$(systemd-escape -p --suffix=automount "$MOUNT_PATH")
MOUNT_UNIT=$(systemd-escape -p --suffix=mount "$MOUNT_PATH")
sudo systemctl start "$AUTO_UNIT" || sudo systemctl start "$MOUNT_UNIT"
systemctl status "$AUTO_UNIT" "$MOUNT_UNIT" --no-pager || true
findmnt /mnt/app
journalctl -b -u "$AUTO_UNIT" -u "$MOUNT_UNIT" --no-pager
```

不要只执行 `mount -a` 后认为启动链路已验证；还要测试重启、网络延迟可用、DNS 暂时失败和服务端不可达。避免在生产机器上重启整个 `remote-fs.target`，它可能影响同机其他远程挂载。

### 5.3 autofs 适用边界

autofs 适合大量不常访问的挂载点，按需挂载和空闲卸载。对于高频低延迟业务路径，autofs 首次访问延迟、过期卸载和并发触发行为需要压测。

```text
/etc/auto.master.d/nfs.autofs
  /- /etc/auto.nfs --timeout=600

/etc/auto.nfs
  /mnt/app -fstype=nfs4,nfsvers=4.1,hard,timeo=600,retrans=2 nfs.example.com:/app
```

修改后验证：

```bash
sudo systemctl reload autofs
ls -ld /mnt/app
findmnt /mnt/app
```

## 6. 网络、防火墙与 DNS

### 6.1 NFSv4.x 网络基线

NFSv4.x 通常以 TCP/2049 为主，但 Kerberos、DNS、NTP、监控和管理流量仍需纳入依赖。启用 delegation、callback 或高可用切换时，还要区分版本：NFSv4.0 可能涉及服务端到客户端的 callback 连接，NFSv4.1+ 使用 session backchannel；防火墙和负载均衡策略必须按目标版本和实现验证。

```text
客户端 -> 服务端 TCP/2049
NFSv4.0 callback -> 按实现和 callback 配置验证
NFSv4.1+ backchannel -> 通常复用 session 连接，仍需抓包和厂商文档确认
客户端/服务端 -> DNS
客户端/服务端 -> NTP/chrony
Kerberos 场景 -> KDC/LDAP/AD
监控与日志 -> 指标和审计系统
```

### 6.2 NFSv3 网络基线

NFSv3 可能需要固定并放行：

- TCP/UDP 2049：NFS；
- rpcbind/portmapper：通常 111；
- mountd：建议固定端口；
- lockd/nlockmgr：建议固定端口；
- statd/NSM：建议固定端口。

只开放 2049 可能导致 v3 数据面可达但挂载、锁或恢复失败。应以 `rpcinfo -p`、防火墙配置和抓包共同验证。

**执行端：客户端或管理节点**

```bash
NFS_SERVER=nfs.example.com
rpcinfo -p "$NFS_SERVER"
rpcinfo -T tcp "$NFS_SERVER" nfs 3
rpcinfo -T udp "$NFS_SERVER" nfs 3 || true
```

### 6.3 DNS 与客户端身份

NFSv4/Kerberos/HA 场景对名称更敏感：

- 服务端名称应稳定，避免客户端缓存旧地址；
- Kerberos 服务主体依赖正确的 FQDN；
- VIP/DNS 切换必须验证 NFSv4 状态恢复；
- 客户端克隆或容器化环境需避免身份重复影响 clientid/recovery。

## 7. 生产验证与观察指标

### 7.1 服务端验证

**执行端：服务端**

```bash
sudo exportfs -v
sudo nfsstat -s
sudo ss -lntup | grep -E ':2049|:111' || true
sudo rpcinfo -p localhost
sudo journalctl -u nfs-server -u nfs-kernel-server --since "10 min ago"
```

验证点：导出路径、客户端匹配、`sec=`、`root_squash`、`sync/async`、NFS 版本、服务端监听、NFSv3 动态 RPC 端口和日志错误。RHEL/Rocky/AlmaLinux 常见服务名是 `nfs-server`，Ubuntu 常见服务名是 `nfs-kernel-server`；具体以 `systemctl list-units '*nfs*'` 为准。

### 7.2 客户端验证

**执行端：客户端**

```bash
NFS_MOUNT=/mnt/app
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
mountpoint "$NFS_MOUNT"
id

# 仅在已确认这是可写测试导出时执行写入验证
TEST_FILE="$NFS_MOUNT/.nfs-kb-l2-01-write-test"
printf 'nfs write test %s\n' "$(date -Is)" > "$TEST_FILE"
stat "$TEST_FILE"
rm -f "$TEST_FILE"
```

若导出是只读或当前用户不应具备写权限，写入失败是预期结果，应记录错误码而不是直接判定 NFS 故障。若测试 root squash，应分别用 root 和业务用户执行，并确认服务端文件 owner/group 是否符合预期。

### 7.3 双客户端语义验证

**执行端：客户端 1 与客户端 2**  
**适用范围：测试导出；不要在生产目录制造锁冲突或故障注入**

客户端 1：

```bash
NFS_MOUNT=/mnt/app
TEST_FILE="$NFS_MOUNT/.nfs-kb-l2-01-visibility"
printf 'client1 %s\n' "$(date -Is)" > "$TEST_FILE"
sync -f "$TEST_FILE"
stat "$TEST_FILE"
```

客户端 2：

```bash
TEST_FILE=/mnt/app/.nfs-kb-l2-01-visibility
stat "$TEST_FILE"
cat "$TEST_FILE"
rm -f "$TEST_FILE"
```

记录两端的时间、挂载参数、属性缓存配置、文件大小、mtime、校验和和错误码。`sync -f` 依赖 GNU coreutils 支持；若目标系统不支持，应改用应用级 `FileChannel.force(true)` 或等价单文件 `fsync` 测试，不建议用普通 `sync` 作为 fallback，因为它会触发更宽范围的系统同步。`sync -f` 也不等于完整证明 NFS COMMIT 和服务端稳定介质；稳定写需要结合抓包、服务端日志和故障注入验证。

### 7.4 性能基线

**执行端：客户端与服务端**

```bash
# 客户端
nfsiostat 1 10
nfsstat -c
cat /proc/self/mountstats

# 服务端
nfsstat -s
iostat -xz 1 10
vmstat 1 10
```

生产基线至少记录：p50/p95/p99、READ/WRITE/GETATTR/LOOKUP/COMMIT、RPC retrans、TCP 重传、服务端 nfsd、后端磁盘、CPU、内存、网络和应用线程池。

## 8. Java 工程视角

### 8.1 挂载语义会直接影响线程模型

Java 代码看到的是普通文件 API，但挂载参数决定故障时线程行为：

| 挂载语义 | Java 表现 | 工程措施 |
| --- | --- | --- |
| `hard` + 服务端不可达 | 线程可能长期卡在 native I/O | 独立 I/O 线程池、降级路径、健康检查隔离 |
| `soft` 返回错误 | `IOException` 或短写风险 | 幂等、校验、重试边界和补偿 |
| 属性缓存较长 | `Files.exists/size` 可能短暂滞后 | 版本文件、rename、应用级校验 |
| `noac` | RPC 放大、延迟升高 | 只在明确一致性实验或特殊场景使用 |
| `root_squash` | root 写入被拒绝或 owner 变为匿名用户 | 容器 UID/GID 与服务端权限统一规划 |

### 8.2 推荐应用写入模式

对需要跨客户端读取的业务文件，建议：

```text
写临时文件
  -> 循环写完整内容
  -> FileChannel.force(true)
  -> 同目录 rename
  -> 读取端校验版本/校验和
```

不要用共享 NFS 目录承载高频细粒度协调状态。若业务需要强一致锁、任务队列或元数据事务，应优先评估数据库、消息队列或协调服务。

## 9. 排障检查清单

遇到挂载失败、权限异常、卡顿或数据可见性问题时：

1. 固化实际挂载参数：`findmnt`、`nfsstat -m`、`/proc/mounts`。
2. 服务端确认生效导出：`exportfs -v`，不要只看 `/etc/exports`。
3. 区分 NFSv3 控制面、NFSv4 伪根、数据面、锁服务和后端存储。
4. 权限问题同时检查 UID/GID、ACL、`root_squash`、匿名 UID/GID、SELinux 和 `sec=`。
5. 卡顿问题按 Java 线程栈 -> `nfsiostat` -> TCP -> 服务端 nfsd -> 后端磁盘定位。
6. 可见性问题检查属性缓存、目录缓存、rename 模式、双客户端时间线和应用校验。
7. 锁问题检查 NFS 版本、NLM/NSM 或 NFSv4 stateid、服务端 grace、客户端 recovery。
8. 任何高风险选项变更都要有基线、灰度、回滚和故障演练记录。

常见误区：

| 误区 | 正确判断 |
| --- | --- |
| `mount` 成功就代表生产可用 | 还要验证权限、锁、缓存、恢复、性能和启动行为 |
| `async` 是通用性能优化 | 它改变服务端确认语义，可能造成确认后数据丢失 |
| `no_root_squash` 方便容器写入 | 它让客户端 root 获得高危权限，应优先修正 UID/GID |
| `noac` 可以解决所有一致性问题 | 它会放大 RPC 和延迟，且不替代应用协议设计 |
| `/etc/fstab` 写上就完成部署 | 还要验证 systemd 启动顺序、网络延迟和服务端不可达 |

## 10. 参考资料与关联文档

### 参考资料

- `man 5 exports`、`man 5 nfs`、`man 8 exportfs`、`man 8 mount.nfs`
- Linux `nfs-utils` 文档与发行版 NFS 管理文档
- RFC 1813：NFS Version 3 Protocol Specification
- RFC 7530：Network File System (NFS) Version 4 Protocol
- RFC 5661 / RFC 8881：NFSv4.1 协议

### 关联文档

- [NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)
- [NFS-KB-L1-02 NFSv3 协议流程与状态边界](../L1-protocol/NFS-KB-L1-02-nfsv3-protocol-and-state-boundaries.md)
- [NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)
- [NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期](NFS-KB-L2-02-systemd-autofs-mount-lifecycle.md)
- [NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](../L3-security/NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)
- [NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](../L3-security/NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)
- [NFS-KB-L4-01 NFS 性能指标、基线与容量模型](../L4-performance/NFS-KB-L4-01-performance-metrics-baseline-and-capacity-model.md)

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-03 | 1.1.4 | 将已发布的 L4-01 加入关联文档 | 知识库交叉引用校验 |
| 2026-08-02 | 1.1.3 | 将已发布的 L3-02 加入关联文档 | 知识库交叉引用校验 |
| 2026-08-02 | 1.1.2 | 将已发布的 L3-01 加入关联文档，并拆分 Kerberos 后续主题引用 | 知识库交叉引用校验 |
| 2026-08-02 | 1.1.1 | 将已发布的 L2-02 加入关联文档链接，修正“待建立”引用 | 知识库交叉引用校验 |
| 2026-08-01 | 1.1.0 | 收紧 exportfs 变更边界、补充 NFSv4 callback/backchannel、crossmnt/sec 风险、NFSv3 端口检查、服务名差异和 sync 验证边界 | 基于文档审查结果修订 |
| 2026-08-01 | 1.0.0 | 初始发布 | 建立 NFS 服务端导出与客户端挂载生产基线 |
