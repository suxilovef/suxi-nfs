# NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护

> 文档状态：待验证  
> 知识阶段：L3 安全与身份  
> 适用范围：Linux NFS 客户端与服务端；RHEL/Rocky/AlmaLinux 8/9、Ubuntu 22.04/24.04 常见环境；NFSv3、NFSv4.0、NFSv4.1、NFSv4.2；MIT Kerberos 或 Active Directory Kerberos 常见集成；`sec=krb5/krb5i/krb5p`；具体行为以目标内核、nfs-utils、krb5-libs、gssproxy、KDC/AD、NAS 和厂商兼容矩阵为准  
> 版本：1.0.2\
> 最后更新：2026-08-03\
> 前置文档：[NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)、[NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)  
> 关联文档：[NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)、[NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)、[NFS-KB-L3-03 NFSv4 ACL、SELinux 与多协议权限集成](NFS-KB-L3-03-nfsv4-acl-selinux-and-multiprotocol-permissions.md)

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. RPCSEC_GSS 与 Kerberos 原理](#2-rpcsec_gss-与-kerberos-原理)
- [3. 端到端认证与访问流程](#3-端到端认证与访问流程)
- [4. 配置、命令与参数说明](#4-配置命令与参数说明)
- [5. Java 工程视角](#5-java-工程视角)
- [6. 生产实践与风险边界](#6-生产实践与风险边界)
- [7. 验证实验与观察指标](#7-验证实验与观察指标)
- [8. 排障证据链与检查清单](#8-排障证据链与检查清单)
- [9. 小结](#9-小结)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

Kerberos NFS 不是在 mount 命令后增加一个 `sec=krb5p` 就完成。生产链路至少包含 DNS、时间同步、realm、KDC、用户 TGT、NFS 服务 principal、keytab、Linux GSS upcall、principal 到本地身份映射、export flavor、DAC/ACL 和凭据续期。

完成本单元后，应能够：

1. 解释 Kerberos TGT、service ticket、GSS context 与 RPCSEC_GSS context 的关系。
2. 准确区分 `krb5`、`krb5i`、`krb5p` 的认证、完整性和隐私边界。
3. 设计 `nfs/FQDN@REALM` principal、DNS alias/VIP、keytab 和高可用节点关系。
4. 区分用户 credential cache、客户端 machine credential 和服务端 keytab。
5. 解释 Linux kernel、`rpc.gssd`、`gssproxy`、nfsd 与 KDC 的端到端调用链。
6. 防止 `sec=sys` fallback、多 flavor 导出和错误 HA 切换造成安全降级。
7. 判断 Java 服务为什么在 `klist` 正常时仍可能收到 `AccessDeniedException`。
8. 建立覆盖 DNS、时钟、票据、kvno、keytab、GSS、principal mapping 和文件权限的排障时间线。

本文不会替代企业 Kerberos/AD 域设计规范，也不讲 KDC 主从、PKINIT、完整 LDAP 架构或通用 JAAS 开发。所有 principal 创建、SPN 注册、keytab 导出和密钥轮换必须由 realm 管理流程完成；示例只展示 NFS 集成所需的对象和验证方式，不生成真实密钥。

## 2. RPCSEC_GSS 与 Kerberos 原理

### 2.1 Kerberos 对象关系

| 对象 | 典型表示 | 作用 | 保存位置 |
| --- | --- | --- | --- |
| realm | `EXAMPLE.COM` | Kerberos 管理与信任域 | KDC、`krb5.conf`、AD |
| 用户 principal | `appsvc@EXAMPLE.COM` | 表示用户或服务账号身份 | KDC 数据库，客户端获得票据 |
| NFS 服务 principal | `nfs/nfs01.example.com@EXAMPLE.COM` | 标识目标 NFS 服务 | KDC/AD + 服务端 keytab |
| TGT | `krbtgt/EXAMPLE.COM@EXAMPLE.COM` | 向 KDC申请其他 service ticket | 用户 credential cache |
| service ticket | `nfs/nfs01.example.com@EXAMPLE.COM` | 向目标 NFS 服务证明身份 | 用户/machine credential cache |
| keytab | principal、kvno、enctype 和长期密钥 | 服务端解密/验证 service ticket；也可供非交互主体取票 | root 或受控代理可读文件/安全凭据存储 |
| GSS context | initiator、acceptor、session key、sequence/window state | 为 RPC 调用提供认证、完整性或隐私服务 | 内核/GSS daemon 运行时状态 |

principal 名字区分大小写语义由 Kerberos 实现和目录策略决定，realm 通常大写，但不能靠字符串外观判断身份相同。服务 principal 的 hostname 必须与客户端实际请求和 Kerberos canonicalization 规则一致。

### 2.2 `krb5`、`krb5i`、`krb5p`

| NFS `sec=` | RPCSEC_GSS service | 提供身份认证 | 保护 NFS payload 完整性 | 加密 NFS payload | 典型成本 |
| --- | --- | --- | --- | --- | --- |
| `krb5` | `rpc_gss_svc_none` | 是 | 否；不能把 payload 当作防篡改 | 否 | 最低 GSS 数据面开销 |
| `krb5i` | `rpc_gss_svc_integrity` | 是 | 是，使用 MIC/完整性封装 | 否 | CPU、带宽和延迟增加 |
| `krb5p` | `rpc_gss_svc_privacy` | 是 | 是 | 是，payload wrap/unwrap | 通常开销最高 |

`krb5` 仍通过 GSS context 和 RPC verifier 建立已认证调用关系，但 NFS 数据 payload 不获得 `krb5i/krb5p` 级别的完整性或隐私保护。TCP 校验和不是对抗主动攻击者的密码学完整性。

`krb5p` 保护 NFS RPC payload，不等于保护客户端内存、服务端后端磁盘、备份、副本、日志或应用在 NFS 之外发送的数据。Kerberos也不替代 Linux DAC、ACL、SELinux、配额和导出最小权限。

### 2.3 票据与 GSS context 不是同一个生命周期

```text
用户或 machine principal
  -> TGT：可申请 service ticket
  -> NFS service ticket：面向 nfs/FQDN
  -> GSS security context：客户端与 NFS 服务之间的运行时上下文
  -> RPCSEC_GSS DATA 请求：带 context handle、sequence 和 service
```

TGT 到期、service ticket 到期和已建立 GSS context 到期可能发生在不同时间。实现可以在可用凭据存在时续建 context，但不能假定已有 mount 或打开文件会永久可用。生产监控要同时观察 credential cache、GSS 日志和实际 NFS I/O。

### 2.4 RPCSEC_GSS 建立过程

RFC 2203 定义的典型过程：

```text
客户端 kernel NFS
  -> 触发 GSS upcall
  -> rpc.gssd 或 gssproxy 查找 initiator credential
  -> 向 KDC 获取/读取 nfs/server service ticket
  -> NFS RPCSEC_GSS INIT / CONTINUE_INIT
  -> 服务端 nfsd + GSS acceptor 使用 keytab 接受 context
  -> 返回 context handle
  -> 后续 DATA 请求使用 sequence number 和 krb5/krb5i/krb5p
  -> context 过期或错误时 DESTROY/重新建立
```

RPCSEC_GSS sequence number 用于调用标识、重放检测和 context window 控制；协议允许窗口内请求乱序传输或处理，不能把它理解为业务 I/O 排序机制，也不能把它等同于 NFSv4 session slot/sequence ID。RPCSEC_GSS context、NFSv4 clientid/session/stateid 是不同状态层，故障恢复时可能分别重建。

### 2.5 用户凭据、machine credential 与服务端 keytab

| 凭据 | 代表谁 | 典型用途 | 常见误区 |
| --- | --- | --- | --- |
| 用户 TGT/ccache | 当前 UID 对应用户 principal | 按用户访问 NFS | root 的 `klist` 看不到业务 UID cache |
| 客户端 machine credential | 客户端主机 principal | root、挂载初始化或实现特定操作 | 不能自动代表所有业务用户 |
| 服务端 NFS keytab | `nfs/server@REALM` acceptor 密钥 | 接受 service ticket | 不是给客户端用户复制使用的 keytab |

Linux `rpc.gssd` 查找 credential cache 的规则受 nfs-utils 版本、ccache 类型、UID、realm 和配置影响。某些发行版通过 gssproxy/KCM 集中代理。必须基于目标系统验证，不使用“`klist` 有票就一定能访问 NFS”的简化结论。

### 2.6 principal 到本地 UID/GID

Kerberos证明 principal 身份后，服务端仍需把它映射为本地凭据，才能执行 VFS DAC/ACL 检查。

```text
appsvc@EXAMPLE.COM
       |
       v
GSS principal mapping / idmapping / NSS / SSSD / storage identity mapper
       |
       v
local uid=20001 gid=30001 groups=[...]
       |
       v
inode mode + ACL + SELinux
```

认证成功但 principal 无法映射、映射到错误 UID、缺少附加组或 ACL 不允许时，客户端仍会得到 `Permission denied`。Kerberos 强认证不会自动赋予文件权限，也不会自动统一 LDAP 中的 `uidNumber/gidNumber`。

### 2.7 DNS、canonicalization 与 SPN

客户端 mount source、DNS CNAME、反向解析、`dns_canonicalize_hostname`、`rdns` 和服务端实际 principal 共同决定请求哪个 `nfs/hostname@REALM`。典型风险：

- 客户端 mount `nfs-vip.example.com`，keytab 只有 `nfs/node01.example.com`；
- CNAME 被 canonicalize 为另一名称，但 KDC 没有对应 SPN；
- AD 中同一 SPN 重复注册到两个对象；
- PTR 返回不受控名称，客户端请求错误 principal；
- HA 切换到节点后，新节点没有相同 kvno 的 acceptor key。

不要用 `/etc/hosts` 临时伪造名称作为生产 Kerberos 修复。应先确定稳定服务名、canonicalization 规则和 SPN 所有权，再设计 DNS/VIP。

## 3. 端到端认证与访问流程

### 3.1 用户访问完整流程

```text
1. 用户 appsvc 登录或凭据管理器取票
   -> 获得 TGT，存入该 UID 可访问的 ccache

2. Java 进程 open("/mnt/secure/file")
   -> VFS -> NFS client，使用进程 fsuid

3. NFS client 需要 nfs/nfs.example.com GSS context
   -> kernel upcall -> rpc.gssd/gssproxy

4. GSS daemon 找到 appsvc 的 TGT/credential
   -> 从 KDC 获得 NFS service ticket

5. RPCSEC_GSS INIT
   -> server nfsd/GSS acceptor 使用 keytab 验证 ticket

6. principal appsvc@EXAMPLE.COM 映射本地 UID/GID/groups

7. export sec=、ro/rw、squash + VFS DAC/ACL + SELinux

8. RPCSEC_GSS DATA 使用 krb5/krb5i/krb5p
   -> 成功或返回认证/权限/NFS 状态

9. Linux errno -> Java IOException/AccessDeniedException
```

### 3.2 挂载成功不等于业务用户有票

mount helper 以 root 执行时，可能通过 machine credential、服务端 flavor 协商或已有 root credential 完成挂载。NFS 超级块建立后，每个用户的数据访问仍可能需要对应 UID 的 Kerberos credential。

```text
systemctl: mnt-secure.mount active
root: ls 成功
appsvc: open -> EACCES
```

这通常不是 mount 丢失，而是业务 UID ccache、principal mapping 或文件权限问题。验证必须用运行 JVM 的真实 UID 执行 I/O。

### 3.3 NFSv4 伪根与 SECINFO

NFSv4 客户端从伪根逐级遍历命名空间，并可通过 SECINFO 获知路径允许的 security flavor。伪根、中间目录和目标导出的 flavor 必须形成可遍历路径。

```text
server:/                 sec=krb5p
  -> /secure             sec=krb5p
       -> /team-a        sec=krb5p
```

如果伪根只允许一种客户端无法使用的 flavor，客户端到达目标子导出之前就会失败。相反，伪根允许 `sys`、敏感子导出允许 `krb5p` 时，是否满足组织的端到端安全策略必须单独评估，不能只看最终目录。

### 3.4 NFSv3 边界

NFSv3 也可结合 RPCSEC_GSS，但控制面和辅助服务更多：rpcbind、mountd、NLM/NSM 及固定端口策略都要验证。mountd 返回的认证 flavor、NFS 数据 RPC 和锁相关身份必须一致。新建强身份环境通常优先评估 NFSv4.1/4.2，以减少外部协议和端口复杂度，但不能绕过厂商兼容矩阵。

### 3.5 错误分层

| 阶段 | 典型错误 | 根因方向 |
| --- | --- | --- |
| DNS/SPN | server not found in Kerberos database | 请求 principal 不存在、canonicalization 错误 |
| KDC | cannot contact any KDC | DNS SRV、路由、防火墙、KDC 可用性 |
| 时间 | clock skew too great / ticket not yet valid | chrony/NTP、虚拟机时间跳变 |
| keytab | KRB_AP_ERR_MODIFIED / decrypt integrity check failed | 错 principal、kvno、密钥或重复 SPN |
| client credential | no credentials / key not available | TGT 缺失、ccache 不可见、UID 不匹配 |
| GSS context | context expired / sequence error | 票据、context 生命周期、重放或切换 |
| identity mapping | authenticated but unmapped | principal 到 UID/GID/NSS 映射 |
| VFS | permission denied | mode、ACL、squash、SELinux、后端 ro |

## 4. 配置、命令与参数说明

### 4.1 环境基线清单

**执行端：两端**  
**适用范围：只读检查；输出包含域、principal 和主机信息，应按安全规范保存**

```bash
hostname -f
uname -r
timedatectl status
chronyc tracking 2>/dev/null || true
chronyc sources -v 2>/dev/null || true
getent hosts nfs.example.com
getent ahosts nfs.example.com

rpm -q nfs-utils krb5-libs gssproxy 2>/dev/null || true
dpkg-query -W nfs-common nfs-kernel-server krb5-user gssproxy 2>/dev/null || true

systemctl list-unit-files | grep -E 'rpc-gssd|rpc-svcgssd|gssproxy|nfs-client|nfs-server'
```

记录实际 FQDN、地址、时间偏差、package 版本和 GSS unit。不同发行版可能使用 `rpc-gssd.service`、`gssproxy.service`、`rpc-svcgssd.service` 或由 `nfs-client.target/nfs-server.service` 拉起，不能按服务名是否存在单点判断。

### 4.2 Kerberos 客户端配置边界

`/etc/krb5.conf` 最小结构示例：

```ini
[libdefaults]
    default_realm = EXAMPLE.COM
    dns_lookup_kdc = true
    rdns = false
    dns_canonicalize_hostname = false
```

**适用范围：** 仅用于说明与 NFS相关的配置项；具体值由组织 Kerberos/DNS 架构决定。  
**预期效果：** 客户端能够确定 realm、KDC 和 NFS service principal 名称。  
**风险：** 机械设置 `rdns=false` 或关闭 canonicalization 可能破坏现有 HTTP、LDAP、SSH 和跨 realm 业务；启用 DNS KDC discovery 则依赖可信 DNS。  
**回滚：** 恢复配置管理中的上一版文件，使用 `kinit/kvno` 和现有 Kerberos 业务回归；不要只验证 NFS。

生产必须明确：

- realm 与 DNS domain 的映射；
- KDC discovery 使用静态配置还是 DNS SRV；
- hostname 是否 canonicalize，CNAME/PTR 是否参与；
- permitted/default enctypes 是否由安全基线统一管理；
- credential cache 类型和默认位置；
- cross-realm trust/referral 规则。

不要为了兼容旧设备重新启用已禁用的弱加密算法。加密类型变更属于 realm 级安全决策，应核对 KDC、客户端库、keytab 和存储设备共同支持范围。

### 4.3 principal 与 keytab 需求清单

服务端至少需要与客户端使用的稳定服务名匹配的 NFS service principal，例如：

```text
nfs/nfs.example.com@EXAMPLE.COM
```

HA/VIP 场景可能同时需要：

```text
nfs/nfs-vip.example.com@EXAMPLE.COM
nfs/node01.example.com@EXAMPLE.COM
nfs/node02.example.com@EXAMPLE.COM
```

是否需要节点 principal 取决于客户端实际 mount 名称、管理访问和厂商实现。不得为“保险”无边界生成 principal，也不得把服务 keytab 分发到非授权主机。若 HA/VIP 设计要求多个候选节点共同接受同一个 `nfs/vip@REALM`，这些节点必须通过受控流程获得相同服务 principal 的有效密钥集合，并统一记录 principal、kvno、enctype、节点清单、投递时间和回滚窗口。

**执行端：服务端**  
**适用范围：只读取证；不输出密钥材料**  
**风险：principal、kvno、enctype 和 keytab 路径仍属于敏感基础设施元数据，输出应受控**

```bash
KEYTAB=/etc/krb5.keytab
sudo stat -c 'path=%n owner=%U group=%G mode=%a size=%s' "$KEYTAB"
sudo klist -kte "$KEYTAB"
```

预期观察：存在目标 `nfs/FQDN@REALM`，kvno 与 KDC 当前版本一致，加密类型满足策略，keytab 不对无关用户或容器可读。`klist -k` 只列元数据，不证明 keytab 中密钥与 KDC 一致；必须通过真实 GSS accept 验证。

本文不提供 `kadmin addprinc/ktadd` 可复制命令，原因是：

- `ktadd` 的默认行为可能轮换 principal 密钥并提高 kvno，使其他节点旧 keytab 立即失效；
- AD SPN 必须唯一绑定到正确对象；
- keytab 落地路径、传输、权限、审计和销毁需要安全流程；
- HA 节点的 kvno 切换必须与旧票据有效期协调。

### 4.4 服务端 export 基线

NFSv4 伪根和敏感导出只允许 `krb5p` 的示例：

```exports
/srv/nfs         10.10.0.0/16(ro,fsid=0,sync,root_squash,no_subtree_check,sec=krb5p)
/srv/nfs/secure  10.10.0.0/16(rw,sync,root_squash,no_subtree_check,sec=krb5p)
```

**适用范围：** 客户端、服务端、KDC、DNS、时间同步和 principal mapping 已完成验证的敏感 NFSv4 导出。  
**预期效果：** 伪根到目标路径全部要求 RPCSEC_GSS privacy service。  
**风险：** 未持有有效凭据的用户和节点将失去访问；`krb5p` 可能增加 CPU、延迟和网络开销。  
**回滚：** 恢复上一版受控导出或回退业务流量；禁止临时添加 `sec=sys` 作为紧急 fallback。若业务接受的降级只到 `krb5i`，必须事先定义独立审批和时限。

**执行端：服务端**  
**风险：`exportfs -ra` 重新加载全部导出，只能在测试和变更窗口执行**

```bash
sudo exportfs -v
sudo exportfs -ra
sudo exportfs -v
sudo journalctl -u nfs-server -u nfs-kernel-server \
  -u gssproxy -u rpc-svcgssd --since "10 min ago" --no-pager
```

多 flavor 语法如 `sec=krb5p:krb5i:krb5:sys` 会允许客户端选择较弱 flavor。除非有明确兼容需求、数据分类和 flavor-specific 权限设计，否则不要在同一敏感导出开放 `sys`。排序不等于不可降级的强制协商。

### 4.5 客户端挂载基线

```fstab
nfs.example.com:/secure  /mnt/secure  nfs4  nfsvers=4.1,proto=tcp,hard,sec=krb5p,_netdev,x-systemd.mount-timeout=45s  0  0
```

**适用范围：** systemd 管理、NFSv4.1 和 `krb5p` 已验证的客户端。  
**预期效果：** 客户端只以 privacy flavor 挂载，不自动退回 `krb5i/krb5/sys`。  
**风险：** 启动时 machine credential、DNS、KDC 或 GSS daemon 不可用可能导致挂载失败；已挂载后的用户 I/O 仍依赖各 UID credential。  
**回滚：** 恢复上一版 fstab 和目标 mount unit；降级 flavor 必须按安全事件处理，不通过删除 `sec=` 依赖默认协商。

**执行端：客户端**  
**风险：只启动目标 mount unit；不要执行无边界 `mount -a` 或重启全部 remote-fs**

```bash
MOUNT_PATH=/mnt/secure
MOUNT_UNIT=$(systemd-escape --path --suffix=mount "$MOUNT_PATH")

sudo findmnt --verify --verbose
sudo systemctl daemon-reload
sudo systemctl start "$MOUNT_UNIT"
systemctl status "$MOUNT_UNIT" --no-pager
findmnt --mountpoint "$MOUNT_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
```

验证输出必须明确包含 `sec=krb5p`。如果只显示 `sec=krb5` 或 `sec=sys`，不能把 mount 成功判定为安全验证通过。

### 4.6 用户 credential cache 验证

**执行端：客户端，以目标业务 UID 执行**  
**适用范围：先被动读取票据状态；principal、票据时间、realm 和 service 名称属于敏感元数据**

```bash
TEST_USER=appsvc
sudo -u "$TEST_USER" klist -ef
```

如果需要验证该 UID 能否获取目标 NFS service ticket，再执行主动验证：

**执行端：客户端，以目标业务 UID 执行**  
**风险：`kvno` 会联系 KDC、获取 service ticket 并写入该 UID 的 ccache；执行前先保存原始 `klist -ef` 输出**

```bash
sudo -u "$TEST_USER" kvno nfs/nfs.example.com@EXAMPLE.COM
sudo -u "$TEST_USER" klist -ef
```

预期观察：TGT 有效、NFS service ticket 可获得、kvno 与服务端 keytab/KDC一致。执行前确认测试用户允许取票且 ccache 可写。

`sudo -u` 只切换进程身份，不保证进入 Java 服务原有的登录会话、session keyring 或 KCM collection，也不保证继承该服务的 `KRB5CCNAME`。若目标使用 `KEYRING:`、`KCM:` 或自定义 `FILE:` cache，应通过组织批准的凭据代理或与服务相同的受控执行上下文取证；不能因为上述命令看不到票据就断言业务进程没有票据，也不能导出完整进程环境到工单。

不要在命令行使用 `kinit principal password`、`echo password | kinit` 或把密码写入脚本。测试环境需要交互取票时使用：

**执行端：隔离测试客户端，以测试用户执行**  
**风险：创建临时 TGT；终端、审计和 ccache 必须受控，结束后执行 `kdestroy`**

```bash
kinit testuser@EXAMPLE.COM
klist -ef
kvno nfs/nfs.example.com@EXAMPLE.COM
```

密码应由 `kinit` 的安全交互提示读取，不能出现在命令参数、shell history、环境变量或文档中。

### 4.7 GSS daemon 与日志

**执行端：两端**  
**适用范围：只读诊断；不存在的 unit 报错属于实现差异**

```bash
systemctl status rpc-gssd gssproxy rpc-svcgssd --no-pager 2>/dev/null || true
systemctl show rpc-gssd gssproxy rpc-svcgssd \
  -p Id -p LoadState -p ActiveState -p SubState -p FragmentPath 2>/dev/null || true
journalctl -b -u rpc-gssd -u gssproxy -u rpc-svcgssd \
  -u nfs-server -u nfs-kernel-server --no-pager | tail -n 300
```

不要在生产直接停止 gssproxy、以前台 debug 模式替换 rpc.gssd，或开启无边界 kernel RPC debug。这些操作可能让所有 Kerberos NFS context 建立失败或产生大量含 principal 的日志。需要 debug 时使用隔离节点、限定时间、保存前后基线并明确恢复 unit。

### 4.8 principal 到本地身份映射

**执行端：服务端**  
**适用范围：只读取证；将 principal 和本地用户替换为脱敏测试对象**

```bash
TEST_USER=appsvc
getent passwd "$TEST_USER"
id "$TEST_USER"
test -r /etc/idmapd.conf && sed -n '1,220p' /etc/idmapd.conf
command -v nfsidmap >/dev/null && sudo nfsidmap -l || true
journalctl -b | grep -Ei 'gss|idmap|principal|nfs.*auth' | tail -n 200
```

不要在生产文档或工单中粘贴完整 realm 映射规则、真实 principal 列表和 keytab 内容。映射修复后必须使用业务 principal 创建文件，并在服务端核对数字 UID/GID、附加组和 ACL，不能只看用户名显示。

## 5. Java 工程视角

### 5.1 JVM 不直接执行 NFS Kerberos 协议

Java `Files`、`FileChannel` 和 `FileLock` 通过 Linux 系统调用进入 kernel NFS client。RPCSEC_GSS context 通常由 kernel、rpc.gssd/gssproxy 和系统 credential cache处理，而不是由 JVM 的 JAAS `Subject` 直接提供。

```text
Java Files.read/write
  -> Linux VFS/NFS client
  -> kernel GSS upcall
  -> rpc.gssd/gssproxy
  -> system ccache/keyring/KCM
  -> KDC + NFS server
```

应用使用 JAAS 登录获得的 `KerberosTicket` 主要供 JVM 内 GSS/Kerberos API 使用。除非有明确的操作系统集成机制，否则它不会自动出现在 Linux kernel NFS client 可见的 credential cache 中。

### 5.2 `KRB5CCNAME` 的边界

给 Java service 设置 `KRB5CCNAME` 只影响继承该环境变量的用户态进程。rpc.gssd/gssproxy 通常是独立 system service，不继承应用环境；它如何查找 ccache 取决于 nfs-utils/gssproxy 配置和 cache 类型。

因此以下配置本身不能证明 NFS 可用：

```ini
[Service]
Environment=KRB5CCNAME=FILE:/run/myapp/krb5cc
```

必须验证：

- JVM 进程 UID 与 ccache owner；
- ccache 类型是否被 rpc.gssd/gssproxy 支持；
- systemd runtime directory 和 SELinux policy；
- TGT 是否可续期、谁负责续期和重新取票；
- 服务重启、节点重启和票据到期后的行为。

### 5.3 systemd Java 服务凭据模型

可选模型：

| 模型 | 优点 | 风险与条件 |
| --- | --- | --- |
| 人员登录后用户 ccache | 符合按用户访问 | 不适合无人值守服务，session 退出可能销毁 cache |
| 服务账号 + KCM/KEYRING | 生命周期与 UID 集成 | 依赖 KCM/SSSD/gssproxy 实现和 HA 设计 |
| 服务账号 keytab 自动取票 | 可无人值守 | 长期密钥暴露、轮换和 ticket renewal 必须治理 |
| machine credential | 运维简单 | 身份粒度是主机，不一定满足业务审计；实现限制 |

优先让专用凭据代理读取 keytab并向 kernel/GSS 提供短期凭据，不让 JVM 直接持有可导出的长期 keytab。具体方案由发行版、gssproxy/SSSD 和组织 Kerberos 平台决定。

### 5.4 票据过期对 Java 的表现

| 系统状态 | Java 可能表现 | 证据 |
| --- | --- | --- |
| 无 TGT/ccache 不可见 | `AccessDeniedException`、`EACCES` | 业务 UID `klist`、rpc.gssd 日志 |
| service ticket 无法获取 | 首次访问或 context 重建失败 | `kvno`、KDC/DNS日志 |
| context 到期且不可续建 | 已运行服务在某时刻突然失败 | GSS context/credential 时间线 |
| principal 映射失败 | 认证后仍被拒绝或显示匿名 | 服务端 idmap/NSS 日志、inode owner |
| keytab kvno 不一致 | 多节点部分成功、部分失败 | node、SPN、kvno、`KRB_AP_ERR_MODIFIED` |

不要把权限异常加入无界重试。票据故障通常持续到凭据、KDC、DNS、时间或映射恢复；高频重试会放大 GSS upcall、KDC 和日志负载。

### 5.5 Java 最小 I/O 探针

下面程序不调用 JAAS，只验证运行 JVM 的 Linux UID 是否能够通过 kernel NFS/RPCSEC_GSS 完成真实 create/write/force/delete。只在专用测试目录执行。

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.StandardCharsets;
import java.nio.file.AccessDeniedException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.time.Instant;
import java.util.Set;
import java.util.UUID;

public final class NfsKerberosProbe {
    private NfsKerberosProbe() {
    }

    public static void main(String[] args) throws IOException {
        if (args.length != 1) {
            throw new IllegalArgumentException("usage: NfsKerberosProbe <test-directory>");
        }

        Path dir = Path.of(args[0]);
        Path file = dir.resolve(".nfs-kb-l3-02-java-"
                + UUID.randomUUID() + ".tmp");
        byte[] payload = ("time=" + Instant.now() + "\n")
                .getBytes(StandardCharsets.UTF_8);

        System.out.printf("stage=start user=%s path=%s time=%s%n",
                System.getProperty("user.name"), file, Instant.now());
        IOException cleanupFailure = null;
        try {
            ByteBuffer buffer = ByteBuffer.wrap(payload);
            try (FileChannel channel = FileChannel.open(file,
                    Set.of(StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE))) {
                while (buffer.hasRemaining()) {
                    channel.write(buffer);
                }
                channel.force(true);
            }
            System.out.printf("stage=created uid=%s gid=%s time=%s%n",
                    Files.getAttribute(file, "unix:uid"),
                    Files.getAttribute(file, "unix:gid"), Instant.now());
        } catch (AccessDeniedException e) {
            System.err.printf("stage=denied file=%s reason=%s time=%s%n",
                    e.getFile(), e.getReason(), Instant.now());
            throw e;
        } finally {
            try {
                Files.deleteIfExists(file);
            } catch (IOException cleanupError) {
                cleanupFailure = cleanupError;
                System.err.printf("stage=cleanup-failed error=%s time=%s%n",
                        cleanupError, Instant.now());
            }
        }
        if (cleanupFailure != null) {
            throw cleanupFailure;
        }
    }
}
```

UUID 文件名用于避免并发探针互相覆盖或删除对方文件，但测试目录仍必须专用并限制单实例并发。探针删除失败会使进程返回失败，便于自动化捕获残留文件；如果同时存在主 I/O 异常，仍以主异常为主，清理失败通过日志保留。生产运行时还应配置外部 watchdog；Java deadline/interrupt 不保证取消已进入 kernel 的 hard NFS I/O。

### 5.6 Java 调试风险

`-Dsun.security.krb5.debug=true` 只调试 JVM 内 Kerberos/JAAS，不一定覆盖 kernel NFS GSS 路径，而且日志可能包含 principal、realm、KDC、票据元数据和配置。不要在生产长期启用，也不要把完整输出粘贴到公开工单。

## 6. 生产实践与风险边界

### 6.1 flavor 选择

| 数据分类 | 推荐起点 | 说明 |
| --- | --- | --- |
| 受控网络、只要求强身份 | `krb5` 可评估 | payload 无强完整性/隐私，需风险接受 |
| 防篡改要求 | `krb5i` | 明文可见但有完整性保护 |
| 敏感数据、跨不完全可信网络 | `krb5p` | payload 加密；仍需静态数据和端点安全 |
| 不可信客户端 | Kerberos + 最小 export + ACL/SELinux | 强身份不等于可信端点 |

新建敏感系统通常以 `krb5p` 为安全基线候选，再通过压测和容量规划确认。不能因为性能不足自动降级到 `sys`；应先定位 CPU、加密算法、网络、单连接、后端存储和客户端并发瓶颈。

### 6.2 禁止隐式降级

```text
期望：krb5p
  -> 失败时拒绝访问并告警

禁止：krb5p 失败
  -> 自动重挂 krb5i
  -> 再失败自动重挂 krb5
  -> 最后使用 sec=sys
```

任何降级都改变数据保护属性，必须是事先审批、带时限、带数据范围、带监控和可回滚的安全事件。应用不得在 mount 失败后自行执行不同 `sec=` 的 mount 命令。

### 6.3 keytab 安全基线

- keytab 是长期密钥材料，不是普通配置文件；
- 不进入 Git、镜像、JAR、日志、工单、备份明文或聊天记录；
- Kubernetes Secret 的 base64 不是加密，仍需 etcd 加密、RBAC、投递和轮换治理；
- 只允许 nfsd/GSS代理所需身份读取，默认 root:root 0600 只是起点；
- 使用受控安全通道投递，记录 principal、kvno、节点、时间和审批，不记录密钥；
- 轮换后保留旧 key 的时间应覆盖必要 ticket/context 生命周期，具体由 realm 和实现验证；
- 删除旧 key 前验证所有 HA 节点和客户端路径。

不建议让 Java 业务容器直接挂载宿主机 `/etc/krb5.keytab`。若业务自身也需要 Kerberos，使用独立最小 principal 和专用 keytab，不能复用 NFS 服务端 acceptor key。

### 6.4 高可用与 key rotation

HA 服务名 `nfs-vip.example.com` 对应的 SPN、keytab、kvno、NFSv4 server identity、恢复目录和 fencing 必须统一设计。

```text
KDC/AD SPN ownership
  -> 新 kvno 安全生成
  -> 所有候选服务节点预置新 key
  -> 验证新旧 ticket 接受范围
  -> 灰度切流与 GSS/NFS 状态恢复
  -> 超过旧 ticket/context 所需窗口
  -> 删除旧 key
```

只更新一个节点 keytab会导致连接到该节点的请求成功，其他节点出现 `KRB_AP_ERR_MODIFIED`。只切 VIP 不代表 NFSv4 stateid、锁和 GSS context 都能连续恢复。

### 6.5 时间同步

Kerberos 默认允许的 clock skew 常见为数分钟，但具体由 realm policy 决定。不要把“误差小于 5 分钟”写成通用 SLA；生产应把时钟偏差控制在远低于策略阈值的范围，并监控：

- NTP/chrony source 数量和 stratum；
- offset、frequency、leap status；
- 虚拟机 suspend/resume 和宿主机时间跳变；
- KDC、客户端、服务端时区显示与 UTC 实际时间；
- 证书、日志和 Kerberos 时间线一致性。

不要使用手工 `date -s` 在运行中修复大偏差，时间回拨/跳变可能影响 JVM、数据库、租约、日志和票据。

### 6.6 KDC、DNS 和依赖容量

Kerberos NFS 的依赖不仅是 TCP/2049：

| 依赖 | 常见端口/协议 | 说明 |
| --- | --- | --- |
| KDC | TCP/UDP 88 | 大响应可能使用 TCP；以 realm 配置为准 |
| kpasswd | TCP/UDP 464 | 仅需要密码/密钥管理的主体使用 |
| DNS | TCP/UDP 53 | A/AAAA/PTR/SRV 与 canonicalization |
| NTP/chrony | UDP 123 或组织时间服务 | 时钟同步 |
| LDAP/AD/SSSD | 环境相关 | principal 到用户、组和策略映射 |
| NFSv4 | TCP 2049 | RPCSEC_GSS context 和数据请求 |

KDC 短暂不可用时，已有 ticket/context 可能继续工作，新 context 或到期重建失败。必须测试故障窗口，而不是只检查端口可达。

### 6.7 性能边界

`krb5i/krb5p` 可能改变 CPU、内存复制、包大小、网络带宽和 p99 延迟。压测至少区分：

- 首次 GSS context 建立与稳定 context 复用；
- 小文件元数据和大文件顺序 I/O；
- 每用户/每 UID context 数量；
- 单客户端、多客户端和 HA 节点；
- 客户端/服务端 CPU、softirq、加密热点；
- `krb5`、`krb5i`、`krb5p` 同条件对比。

性能对比只能在隔离测试导出临时允许多 flavor，测试结束恢复强制 flavor。不要为基准测试在生产敏感导出开放 `sec=sys`。

### 6.8 禁止的通用修复

| 错误动作 | 风险 |
| --- | --- |
| export 增加 `sec=sys` | 直接绕过 Kerberos 强身份和数据保护 |
| 把 keytab chmod 0644 | 长期密钥可被本机其他用户读取 |
| 将服务端 keytab 复制给客户端/JVM | acceptor 密钥扩散，可伪装服务端 |
| 删除并重建 SPN | kvno/对象所有权变化，现有节点和票据失效 |
| `kdestroy -A` 全局清理 | 影响同 UID/节点其他 Kerberos 业务，破坏证据 |
| 关闭 DNS canonicalization 解决单个主机 | 可能破坏其他 Kerberos 服务 |
| 关闭时间校验或放大 clock skew | 扩大重放窗口并掩盖时钟故障 |
| 无限重试 mount/I/O | 放大 KDC、GSS upcall、线程和日志压力 |

## 7. 验证实验与观察指标

以下实验只能在隔离 Kerberos realm 或受控测试 principal、测试 keytab 和测试导出中执行。不得把真实生产 keytab、密码、principal 清单或抓包提交到知识库。

### 7.1 实验一：DNS、时间和 service principal 预检

**执行端：客户端**  
**适用范围：先执行被动 DNS、时间和当前票据检查；使用脱敏测试服务名**

```bash
NFS_NAME=nfs.example.com
NFS_PRINCIPAL=nfs/nfs.example.com@EXAMPLE.COM

hostname -f
getent ahosts "$NFS_NAME"
timedatectl show -p NTPSynchronized -p TimeUSec -p Timezone
chronyc tracking 2>/dev/null || true
klist -ef
```

确认需要主动验证 service ticket 获取能力时，再执行：

**执行端：客户端**  
**风险：`kvno` 会联系 KDC 并修改当前 ccache；如果目标是保留故障现场，先保存原始票据状态**

```bash
kvno "$NFS_PRINCIPAL"
```

记录 canonical service name、解析地址、客户端时间、TGT、service ticket kvno 和执行 UID。`kvno` 成功只证明客户端可获得 ticket，不证明服务端 keytab 能解密。

### 7.2 实验二：服务端 keytab 和 acceptor

**执行端：服务端**  
**风险：只列 keytab 元数据；不得复制、base64、上传或输出密钥**

```bash
KEYTAB=/etc/krb5.keytab
sudo stat -c 'owner=%U group=%G mode=%a size=%s path=%n' "$KEYTAB"
sudo klist -kte "$KEYTAB"
systemctl status gssproxy rpc-svcgssd nfs-server nfs-kernel-server \
  --no-pager 2>/dev/null || true
```

客户端随后用测试 principal 访问 `sec=krb5p` 测试导出，服务端观察 journal。只有真实 GSS context 接受成功，才能证明 SPN、kvno、enctype 和 keytab 密钥匹配。

### 7.3 实验三：业务 UID 凭据可见性

**执行端：客户端，以 JVM 运行 UID 执行**  
**适用范围：已有受控测试 TGT；不在命令中包含密码**

```bash
TEST_USER=appsvc
TEST_PATH=/mnt/secure/.nfs-kb-l3-02-user

sudo -u "$TEST_USER" klist -ef
sudo -u "$TEST_USER" kvno nfs/nfs.example.com@EXAMPLE.COM

write_rc=0
sudo -u "$TEST_USER" sh -c \
  'set -C; umask 0077; printf "%s\n" "$(id)" > "$1"' \
  sh "$TEST_PATH" || write_rc=$?
printf 'write_rc=%s\n' "$write_rc"
```

服务端核对测试文件的 local UID/GID、principal mapping、ACL 和 GSS 日志。`write_rc=0` 才表示完整链路成功；非零时按第 8 节分层，不通过增加 `sec=sys` 重试。若实际服务使用 KCM、KEYRING 或自定义 cache，本例中的 `sudo -u` 可能创建不同会话，必须再用与 JVM 服务相同的受控凭据上下文验证。

### 7.4 实验四：无票据与票据恢复

**执行端：隔离测试客户端，在专用测试用户的真实登录/服务会话中执行**  
**风险：`kdestroy` 会销毁当前活动 cache；禁止复用生产业务 UID、root，或承载其他 Kerberos 业务的 KCM/KEYRING cache collection**

```bash
BASELINE_FILE=/mnt/secure/.nfs-kb-l3-02-baseline

kinit testuser@EXAMPLE.COM
klist -ef
kvno nfs/nfs.example.com@EXAMPLE.COM

test -r "$BASELINE_FILE"
dd if="$BASELINE_FILE" of=/dev/null bs=4k count=1 status=none

kdestroy
klist -s
printf 'klist_after_destroy_rc=%s\n' "$?"

read_after_destroy_rc=0
dd if="$BASELINE_FILE" of=/dev/null bs=4k count=1 status=none \
  || read_after_destroy_rc=$?
printf 'read_after_destroy_rc=%s\n' "$read_after_destroy_rc"
```

实验前必须证明该测试 UID 没有其他 Kerberos 工作负载，并由服务端管理员预先创建只读基线文件，避免客户端 root squash 或权限差异影响结论。先在相同会话和 cache 下完成一次 `sec=krb5*` NFS I/O；否则 `klist` 成功不等于 rpc.gssd/gssproxy 能发现该 cache。密码只允许由 `kinit` 安全交互提示读取。

销毁票据后执行一次低频 NFS 读取并记录“成功或失败”以及 rpc.gssd/gssproxy 日志。已有 GSS context 仍有效时，读取可以继续成功；这不是 `kdestroy` 失效。若要观察新 context 建立失败，应在隔离节点等待 context 到期，或使用经验证的全新测试 UID/会话，不能通过重启生产 GSS/NFS 服务强制清缓存。

随后通过安全交互 `kinit` 恢复 TGT，执行 `kvno` 和同一读取，记录 context 重建及业务恢复。实验结束再次 `kdestroy`，不保留测试票据。该实验不能替代生产 KCM/KEYRING 会话的完整到期与续期演练。

### 7.5 实验五：flavor 强制与禁止降级

**执行端：客户端与服务端**  
**适用范围：独立测试导出；生产敏感导出不得临时开放 `sys`**

验证矩阵：

| 服务端只允许 | 客户端 mount `sec=` | 预期 |
| --- | --- | --- |
| `krb5p` | `krb5p` | 有有效凭据时成功 |
| `krb5p` | `krb5i` | 失败，不自动升级/降级 |
| `krb5p` | `krb5` | 失败 |
| `krb5p` | `sys` | 失败 |

**执行端：客户端**

```bash
NFS_SOURCE=nfs.example.com:/secure
MOUNT_PATH=/mnt/secure-flavor-test
sudo mkdir -p "$MOUNT_PATH"

for flavor in krb5p krb5i krb5 sys; do
    sudo umount "$MOUNT_PATH" 2>/dev/null || true
    mount_rc=0
    sudo mount -t nfs4 -o "nfsvers=4.1,proto=tcp,hard,sec=$flavor" \
      "$NFS_SOURCE" "$MOUNT_PATH" || mount_rc=$?
    printf 'flavor=%s mount_rc=%s\n' "$flavor" "$mount_rc"
    if [ "$mount_rc" -eq 0 ]; then
        findmnt --mountpoint "$MOUNT_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
        nfsstat -m | sed -n "/$MOUNT_PATH/,+5p"
    fi
done

sudo umount "$MOUNT_PATH" 2>/dev/null || true
findmnt --mountpoint /mnt/secure -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
```

预期只有 `krb5p` 在有效凭据存在时成功，`krb5i/krb5/sys` 均失败。必须保存每次 mount 返回码、实际 mount options 和服务端 `exportfs -v`。只看到 I/O 成功而未确认实际 flavor，不算通过。测试结束后确认未残留测试挂载点。

### 7.6 实验六：ticket/context 到期演练

使用 KDC 管理员创建短生命周期、可审计的专用测试 principal，不修改 realm 全局 ticket policy。记录：

```text
T0 获取 TGT 和 service ticket
T1 首次建立 GSS context并成功 I/O
T2 service ticket 到期前后持续低频 I/O
T3 TGT 到期但 context 可能仍存活
T4 context 需要重建时的错误
T5 续期/重新取票后恢复
```

同时采集 `klist -ef`、rpc.gssd/gssproxy journal、NFS mountstats、Java异常和 KDC 日志。不要通过修改系统时钟模拟过期，它会污染 NFS lease、JVM、日志和其他协议。

### 7.7 实验七：HA 与 kvno 故障演练

**执行端：隔离 HA 测试环境**  
**风险：需要 realm 管理员、NFS 管理员和网络管理员共同执行；不得在生产轮换窗口外修改 SPN/keytab**

验证：

1. VIP 名称对应的 SPN 唯一且归属正确。
2. 每个候选节点 keytab 都包含目标 principal 和相同有效 kvno。
3. 切换前后 `krb5p` 新建 context 成功。
4. 已打开文件、锁和 NFSv4 stateid 按 HA 设计恢复。
5. 旧 ticket 在轮换窗口内按设计接受或明确失败。
6. KRB_AP_ERR_MODIFIED、GSS context、锁恢复和业务错误率均受监控。

### 7.8 抓包验证

**执行端：隔离测试客户端或受控镜像点**  
**风险：Kerberos 抓包包含 principal、realm、ticket、SPN、时间和网络拓扑；即使 `krb5p` 隐藏 NFS payload，pcap 仍是敏感材料**

```bash
NFS_SERVER=192.0.2.20
umask 077
CAPTURE=$(mktemp --suffix=.pcap /var/tmp/nfs-kb-l3-02-gss.XXXXXX)

sudo timeout 30s tcpdump -i any -s 0 -w "$CAPTURE" \
  "host $NFS_SERVER and (port 2049 or port 88)"
sudo chmod 0600 "$CAPTURE"
printf 'Record this capture path for cleanup: %s\n' "$CAPTURE"

if command -v tshark >/dev/null; then
    tshark -r "$CAPTURE" -Y 'rpc || kerberos' \
      -T fields -e frame.time -e ip.src -e ip.dst -e rpc.xid
fi
```

RPC auth flavor 数值 6 表示 RPCSEC_GSS。上面的显示过滤器只做 RPC/Kerberos 初筛，不据此证明 flavor 或 protection service。Wireshark 字段和 dissector 随版本变化；用 `tshark -G fields | grep -Ei 'rpc.*auth.*flavor|rpc.*gss|kerberos'` 确认本机字段，再判断 auth flavor、GSS service、context 和错误。不要尝试把 keytab 提供给通用抓包平台解密生产流量。

### 7.9 实验清理

**执行端：测试客户端**  
**风险：只清理专用测试 principal 的 ticket、固定测试文件和已记录抓包；先保存脱敏证据**

```bash
TEST_USER=appsvc
sudo -u "$TEST_USER" rm -f /mnt/secure/.nfs-kb-l3-02-user

# 仅当该 UID 的 cache 专用于本实验时执行
# sudo -u "$TEST_USER" kdestroy

if [ -n "${CAPTURE:-}" ]; then
    case "$CAPTURE" in
        /var/tmp/nfs-kb-l3-02-gss.*.pcap) sudo rm -f -- "$CAPTURE" ;;
        *) echo "unexpected capture path" >&2; exit 1 ;;
    esac
fi
```

如果客户端无法清理测试文件，应在确认后端路径和文件名后由服务端管理员删除，不能通过 `no_root_squash` 或递归 chmod 清理。测试 keytab 和 principal 的撤销/销毁由 realm 管理流程执行，知识库脚本不自动删除 KDC 对象。

### 7.10 必须记录的指标

| 维度 | 证据 | 目的 |
| --- | --- | --- |
| DNS/SPN | mount 名称、A/AAAA/PTR、canonical name | 确认请求哪个 principal |
| 时间 | client/server/KDC offset 与同步状态 | 排除 skew 和时间跳变 |
| ccache | 类型、owner UID、TGT/service ticket 生命周期 | 确认 initiator credential |
| keytab | principal、kvno、enctype、节点、权限 | 确认 acceptor 准备状态 |
| GSS | daemon、context 建立/到期/错误 | 定位 RPCSEC_GSS 层 |
| mount/export | 实际 `sec=`、版本、source、服务端 flavor | 防止安全降级 |
| identity | principal -> local UID/GID/groups | 连接认证与 VFS 权限 |
| NFS | status、retrans、mountstats、state recovery | 区分协议和安全故障 |
| Java | PID/UID、操作阶段、异常、线程和时间 | 定位应用影响 |
| 性能 | context setup、CPU、吞吐、p99 | 评估 flavor 成本 |

## 8. 排障证据链与检查清单

### 8.1 先回答十个问题

1. 客户端实际 mount source 是 hostname、CNAME 还是 VIP？
2. 客户端请求的准确 `nfs/FQDN@REALM` 是什么？
3. 运行 Java 的 UID 能否看到有效 TGT 和 service ticket？
4. rpc.gssd/gssproxy 是否支持并能读取当前 ccache 类型？
5. 服务端 keytab 是否包含正确 principal、kvno 和 enctype？
6. 客户端、服务端、KDC 时间是否在策略范围内？
7. 实际 mount 和 export flavor 是否都是期望的 `krb5/krb5i/krb5p`？
8. principal 是否映射到正确 local UID/GID/groups？
9. DAC/ACL/SELinux/后端文件系统是否允许？
10. 故障是否只发生在某个 HA 节点、某个 UID 或票据到期后？

### 8.2 客户端标准取证

**执行端：客户端**  
**适用范围：先执行被动状态检查；`klist/kvno` 必须以目标业务 UID 执行**

```bash
TEST_USER=appsvc
NFS_MOUNT=/mnt/secure
NFS_PRINCIPAL=nfs/nfs.example.com@EXAMPLE.COM

grep -F " $NFS_MOUNT " /proc/self/mountinfo || true
findmnt --mountpoint "$NFS_MOUNT" -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m

id "$TEST_USER"
sudo -u "$TEST_USER" klist -ef
sudo -u "$TEST_USER" kvno "$NFS_PRINCIPAL"

hostname -f
getent ahosts nfs.example.com
timedatectl show -p NTPSynchronized -p TimeUSec
chronyc tracking 2>/dev/null || true

systemctl status rpc-gssd gssproxy --no-pager 2>/dev/null || true
journalctl -b -u rpc-gssd -u gssproxy --no-pager | tail -n 300
```

`kvno` 会获取 service ticket，改变 ccache。若目标是保存故障现场，应先执行 `klist -ef` 并记录，再决定是否用 `kvno` 主动修复/验证。

这里的 `sudo -u` 同样不能保证进入目标服务的 KCM collection、session keyring 或自定义 FILE cache。必须把“目标 JVM 的 UID/会话/cache 类型”和“诊断命令实际使用的 cache”分别记录；不一致时，该命令结果只能证明诊断上下文，不能代表 JVM 上下文。

### 8.3 服务端标准取证

**执行端：服务端**  
**适用范围：只列 keytab 元数据和日志；输出按敏感信息管理**

```bash
KEYTAB=/etc/krb5.keytab
BACKEND_PATH=/srv/nfs/secure

sudo exportfs -v
sudo stat -c 'owner=%U group=%G mode=%a size=%s path=%n' "$KEYTAB"
sudo klist -kte "$KEYTAB"
hostname -f
timedatectl show -p NTPSynchronized -p TimeUSec
chronyc tracking 2>/dev/null || true

namei -om "$BACKEND_PATH"
stat -c 'uid=%u gid=%g mode=%a name=%n' "$BACKEND_PATH"
getfacl -p "$BACKEND_PATH" 2>/dev/null || true
getenforce 2>/dev/null || true
sudo ausearch -m AVC,USER_AVC -ts recent 2>/dev/null | tail -n 100

systemctl status gssproxy rpc-svcgssd nfs-server nfs-kernel-server \
  --no-pager 2>/dev/null || true
journalctl -b -u gssproxy -u rpc-svcgssd -u nfs-server \
  -u nfs-kernel-server --no-pager | tail -n 300
```

### 8.4 错误到根因映射

| 现象/日志 | 第一检查点 | 不应立即执行 |
| --- | --- | --- |
| `Server not found in Kerberos database` | mount 名、canonicalization、SPN | 新建一批猜测 principal |
| `Cannot contact any KDC` | DNS SRV、路由、88/TCP+UDP、KDC | 添加 `sec=sys` |
| `Clock skew too great` | 三方 UTC、chrony、时间跳变 | 放大 realm skew |
| `KRB_AP_ERR_MODIFIED` | SPN 唯一性、节点、kvno、keytab | 删除重建 SPN |
| `No credentials were supplied` | 业务 UID ccache、rpc.gssd 搜索 | 给 keytab chmod 0644 |
| mount 成功、业务 UID失败 | 用户 ticket、UID、mapping、ACL | 重挂载全部 NFS |
| `krb5p` 吞吐下降 | 两端 CPU、context setup、I/O 模型 | 降级 `sys` |
| 仅一个 HA 节点失败 | 节点 keytab/kvno/SPN、时钟 | 反复切 VIP |
| 票据到期后失败 | TGT renewal、ccache、context 重建 | 无限重试 Java I/O |

### 8.5 `KRB_AP_ERR_MODIFIED` 分析

此错误通常表示服务端无法用预期密钥解密 ticket，常见原因：

```text
客户端请求 nfs/name@REALM
  -> DNS/canonicalization 指向节点 B
  -> KDC 返回 SPN 当前 kvno=N ticket
  -> 节点 B keytab 只有 kvno=N-1 或另一 principal 密钥
  -> accept 失败：KRB_AP_ERR_MODIFIED
```

取证必须把客户端 service ticket、KDC SPN 对象、请求命中的节点和节点 keytab 放在同一时间线。不要在未确认对象所有权前运行 `ktadd`，它可能再次提高 kvno。

### 8.6 mount 成功但 Java 失败

```text
mount unit active
  -> root/machine context 可用
  -> JVM UID 无 TGT、ccache 不可见或 principal mapping失败
  -> Java open -> AccessDeniedException
```

检查 `/proc/<jvm-pid>/status` Uid/Gid/Groups，以该 UID 执行 `klist`，再看 rpc.gssd/gssproxy 日志。不要使用管理员 shell 的 `klist` 代替业务 UID。

### 8.7 安全恢复顺序

```text
阻止新的敏感写流量
  -> 保存 mount/export、业务 UID ccache、DNS、时间、日志、node、kvno 证据
  -> 修复唯一根因：DNS/SPN、时间、ticket、keytab、GSS daemon 或 mapping
  -> 用目标业务 UID 执行 kvno 和最小 I/O
  -> 确认实际 sec= 未降级
  -> 验证 HA 节点和票据续期
  -> 恢复流量并观察完整票据周期
```

恢复过程中不删除原 keytab、不清理共享 ccache、不新增 `sec=sys`，直到证据已保存且安全负责人批准明确变更。

### 8.8 生产检查清单

- [ ] 已确定稳定 NFS 服务名、DNS canonicalization 和唯一 SPN 所有权。
- [ ] 服务端每个候选节点拥有正确 principal、kvno、enctype 和受控 keytab权限。
- [ ] 客户端、服务端、KDC 时间同步和偏差监控已建立。
- [ ] 运行 JVM 的真实 UID 能获得且续期所需 TGT/service ticket。
- [ ] rpc.gssd/gssproxy 支持当前 ccache 类型和查找路径。
- [ ] principal 到 local UID/GID/groups 映射已通过真实文件验证。
- [ ] 伪根、中间路径和目标 export flavor 可连续遍历。
- [ ] 实际 mount options 与 export 都强制期望 `sec=`，不存在隐式 `sys` fallback。
- [ ] keytab 不进入代码仓库、镜像、日志、工单或无加密备份。
- [ ] key rotation 覆盖 HA 所有节点、旧票据窗口、回滚和审计。
- [ ] 已测试 KDC/DNS短时不可用、ticket/context 到期和 HA 切换。
- [ ] Java readiness、线程隔离和错误处理不会放大 GSS/KDC 故障。
- [ ] `krb5i/krb5p` 性能已在业务 I/O 模型下建立 p95/p99 基线。

## 9. 小结

1. Kerberos ticket 证明 principal，RPCSEC_GSS context 为 NFS RPC 提供认证、完整性或隐私服务。
2. `krb5` 只提供强身份基础，`krb5i` 增加 payload 完整性，`krb5p` 增加完整性和隐私。
3. 用户 TGT、客户端 machine credential、服务端 NFS keytab 是三个不同凭据对象。
4. mount 成功不代表每个业务 UID 都有可见且可续期的 Kerberos credential。
5. JAAS ticket 和 `KRB5CCNAME` 不会自动进入 kernel NFS 的 GSS credential 路径。
6. DNS、SPN、keytab、kvno、时间和 HA 节点必须作为同一个服务身份设计。
7. Kerberos认证成功后仍要经过 principal mapping、DAC/ACL、SELinux和后端文件系统检查。
8. `sec=sys` fallback 是安全降级，不能作为 Kerberos 故障的临时通用修复。
9. keytab 是长期密钥材料；生成、分发、轮换和销毁必须由 realm 安全流程治理。
10. 企业级排障必须关联业务 UID、ccache、service ticket、GSS context、节点 keytab、实际 flavor 和 Java 异常时间线。

## 10. 参考资料与关联文档

### 10.1 参考资料

- RFC 2203：RPCSEC_GSS Protocol Specification
- RFC 2623：NFSv2/v3 Security Issues and the NFS Protocol's Use of RPCSEC_GSS and Kerberos V5
- RFC 4120：The Kerberos Network Authentication Service (V5)
- RFC 2743、RFC 2744：GSS-API 框架与 C bindings
- RFC 7530、RFC 8881：NFSv4.0/NFSv4.1 security flavor、SECINFO 与 RPCSEC_GSS 集成
- RFC 7862：NFSv4.2 扩展及相关安全考虑
- `exports(5)`、`nfs(5)`、`mount.nfs(8)`：`sec=krb5/krb5i/krb5p` 和挂载语义
- `rpc.gssd(8)`、`rpc.svcgssd(8)`、`gssproxy(8)`：Linux NFS GSS credential 与 acceptor 实现
- `krb5.conf(5)`、`kinit(1)`、`klist(1)`、`kvno(1)`、`kdestroy(1)`：Kerberos配置、票据与验证
- MIT Kerberos、Microsoft AD Kerberos 和目标发行版 NFS/Kerberos 官方文档
- Linux kernel NFS/RPCSEC_GSS 文档及目标 NAS/云文件服务兼容矩阵
- Kubernetes Secret、CSI driver、gssproxy/SSSD/KCM 官方文档：容器凭据投递与生命周期边界

### 10.2 关联文档

- [NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)
- [NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)
- [NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)
- [NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期](../L2-deployment/NFS-KB-L2-02-systemd-autofs-mount-lifecycle.md)
- [NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)
- [NFS-KB-L3-03 NFSv4 ACL、SELinux 与多协议权限集成](NFS-KB-L3-03-nfsv4-acl-selinux-and-multiprotocol-permissions.md)

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-03 | 1.0.2 | 将已发布的 L3-03 加入关联文档 | 知识库交叉引用校验 |
| 2026-08-02 | 1.0.1 | 修正 RPCSEC_GSS sequence 语义、HA keytab 分发边界、kvno 主动验证标注、票据销毁实验、flavor 验证命令和 Java 探针清理失败处理，补充 RFC 2623/RFC 7862 参考 | 基于发布前复查发现的问题修订 |
| 2026-08-02 | 1.0.0 | 初始发布 | 建立 RPCSEC_GSS、Kerberos、keytab、GSS context、Java 凭据和生产排障安全基线 |
