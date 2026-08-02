# NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型

> 文档状态：待验证  
> 知识阶段：L3 安全与身份  
> 适用范围：Linux NFS 客户端与服务端；RHEL/Rocky/AlmaLinux 8/9、Ubuntu 22.04/24.04 常见环境；NFSv3、NFSv4.0、NFSv4.1、NFSv4.2；`sec=sys`、Linux DAC、POSIX ACL、常见 NFSv4 owner 映射和 SELinux 集成边界；具体行为以目标内核、nfs-utils、NSS/SSSD、后端文件系统及存储厂商实现为准  
> 版本：1.1.1  
> 最后更新：2026-08-02  
> 前置文档：[NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)、[NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)  
> 关联文档：[NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)、[NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期](../L2-deployment/NFS-KB-L2-02-systemd-autofs-mount-lifecycle.md)、[NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. AUTH_SYS 与身份映射原理](#2-auth_sys-与身份映射原理)
- [3. 端到端权限判定流程](#3-端到端权限判定流程)
- [4. 配置、命令与权限模型](#4-配置命令与权限模型)
- [5. Java 工程视角](#5-java-工程视角)
- [6. 生产实践与风险边界](#6-生产实践与风险边界)
- [7. 验证实验与观察指标](#7-验证实验与观察指标)
- [8. 排障证据链与检查清单](#8-排障证据链与检查清单)
- [9. 小结](#9-小结)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

NFS 权限问题不能只看客户端 `ls -l`，也不能只看服务端 `/etc/exports`。一次文件访问至少同时经过：客户端进程凭据、RPC 安全 flavor、服务端 squash、数字身份解析、文件模式位或 ACL、LSM/SELinux、导出只读属性和后端文件系统状态。

完成本单元后，应能够：

1. 解释 AUTH_SYS 在 RPC 中携带什么，以及它为什么不是强身份认证。
2. 区分 NFSv3 数字 UID/GID、NFSv4 `owner@domain` 属性映射和 Kerberos 主体映射。
3. 从客户端进程的有效 UID、主 GID、附加组追踪到服务端 VFS 权限检查。
4. 正确设计 `root_squash`、`all_squash`、`anonuid/anongid`、setgid 目录和 POSIX ACL。
5. 识别 LDAP/SSSD、容器 user namespace、Kubernetes `runAsUser/fsGroup` 对 NFS 数字身份的影响。
6. 区分 export 拒绝、DAC/ACL 拒绝、SELinux AVC、只读导出和 NFSv4 idmapping 故障。
7. 从 Java `AccessDeniedException` 建立客户端、网络、服务端和后端文件系统的完整证据链。

本文深入 `sec=sys/AUTH_SYS` 和 Linux 权限模型。RPCSEC_GSS/Kerberos 的主体、票据、GSS context、完整性与隐私保护将在后续 L3 文档单独展开。NFSv4 ACL 和 Labeled NFS 在本文只定义边界，不把所有存储厂商的 ACL 转换行为泛化为 Linux 通用行为。

## 2. AUTH_SYS 与身份映射原理

### 2.1 AUTH_SYS 凭据内容

AUTH_SYS（历史上也称 AUTH_UNIX）由 RFC 5531 定义。RPC 请求的 credential 典型包含：

| 字段 | 含义 | 安全边界 |
| --- | --- | --- |
| stamp | 客户端生成的时间相关标记 | 不是防重放安全令牌 |
| machinename | 客户端声明的主机名 | 不是经过密码学认证的主机身份 |
| uid | 发起系统调用进程的有效用户 ID | 服务端信任客户端内核提供的数字值 |
| gid | 有效主组 ID | 同样是客户端声明的数字值 |
| gids | 附加组 ID 列表 | 经典 AUTH_SYS 编码上限为 16 个组 |

```text
Java/Linux 进程
  euid=20001 egid=30001 groups=[30001,30002]
          |
          v
Linux NFS 客户端构造 RPC AUTH_SYS credential
  uid=20001 gid=30001 gids=[30002,...]
          |
          v
服务端 nfsd 接收并构造请求凭据
          |
          v
export squash -> VFS DAC/ACL -> LSM/SELinux -> 后端文件系统
```

AUTH_SYS 不携带密码，不证明请求者确实拥有某个 UID，也不提供消息完整性、机密性或可靠防重放。能够控制受信客户端内核或构造 RPC 流量的主体，可能伪造数字身份。因此 `sec=sys` 的安全边界必须包含：受控客户端、网络隔离、最小导出范围、root squash 和服务端文件权限。

`secure` 导出选项只要求传统客户端使用特权源端口，它不能把 AUTH_SYS 变成密码学认证；NAT、容器、CAP_NET_BIND_SERVICE 和受控主机被攻陷时都可能削弱这一假设。

### 2.2 NFSv3：权限核心是数字 UID/GID

NFSv3 文件属性直接携带数字 `uid` 和 `gid`。客户端把数字值显示成用户名时，会查询本机 NSS；因此：

```text
服务端 inode owner = UID 20001
客户端 A：getent passwd 20001 -> appsvc
客户端 B：getent passwd 20001 -> legacy
客户端 C：查不到 20001       -> ls 显示 20001
```

三个客户端看到的名字不同，但服务端权限检查面对的是同一个数字 UID 20001。反过来，两端都存在名为 `appsvc` 的用户，如果 UID 分别是 20001 和 21001，它们不是同一 NFS 身份。

### 2.3 NFSv4：owner 属性与请求认证是两条链

NFSv4 的 `owner` 和 `owner_group` 文件属性采用 UTF-8 字符串，典型形式是 `name@domain`。Linux 客户端和服务端可能通过 `nfsidmap`、`rpc.idmapd`、keyring、NSS 或实现内置的数字模式完成名称与本地 UID/GID 的转换。

必须区分：

| 链路 | 解决的问题 | 典型故障 |
| --- | --- | --- |
| RPC 认证凭据 | 当前请求以谁的身份访问 | AUTH_SYS UID/GID、Kerberos principal |
| owner 属性映射 | 文件属主如何在协议和本机之间表示 | 显示 `nobody`、数字 ID、`name@domain` 不一致 |

NFSv4 属性显示为 `nobody` 不等于每一次数据访问都使用匿名身份；反之，`ls -l` 能显示正确用户名也不证明 AUTH_SYS 可信。必须同时检查挂载 `sec=`、实际请求凭据、服务端 squash 和 inode 权限。

### 2.4 Linux NFSv4 idmapping 组件

常见组件关系如下：

```text
NFSv4 owner/owner_group string
          |
          v
Linux kernel idmap upcall / numeric-id mode
          |
          +--> nfsidmap + request-key + keyring cache
          |
          +--> rpc.idmapd（取决于内核、角色和发行版实现）
          |
          v
/etc/idmapd.conf Domain + NSS(getent/SSSD/LDAP/local files)
          |
          v
local UID/GID or unmapped anonymous identity
```

不能只检查 `rpc.idmapd` 是否 running。较新的 Linux 可能主要通过 `nfsidmap` 和 keyring 完成客户端映射，也可能因数字 ID 模式而不走预期 upcall。以下内核参数用于确认实现状态，但并非所有内核都暴露相同路径：

**执行端：两端**  
**适用范围：Linux NFSv4；只读检查**

```bash
for p in \
  /sys/module/nfs/parameters/nfs4_disable_idmapping \
  /sys/module/nfsd/parameters/nfs4_disable_idmapping; do
    if [ -r "$p" ]; then
        printf '%s=' "$p"
        cat "$p"
    fi
done

test -r /etc/idmapd.conf && sed -n '/^\[General\]/,/^\[/p' /etc/idmapd.conf
command -v nfsidmap >/dev/null && sudo nfsidmap -l || true
```

预期观察：记录客户端和服务端是否使用数字 ID 模式、双方 Domain 配置、当前 idmap key。keyring 可见范围受执行用户、session、内核和发行版实现影响，`nfsidmap -l` 空输出不能单独证明没有 idmap cache。不要为了消除 `nobody` 直接修改内核模块参数；模式变更会影响现有挂载、缓存和所有 NFSv4 业务，必须在兼容性测试后通过持久化配置统一实施。

### 2.5 NSS、LDAP、SSSD 的位置

LDAP/AD/SSSD 本身不改变 AUTH_SYS wire format。它们通过 NSS 为本机提供用户名、UID、GID 和附加组查询，并可能参与 NFSv4 name mapping 或服务端 `manage-gids`。

```text
LDAP 中的 uidNumber/gidNumber
        |
        v
SSSD/NSS on client ----> 进程 euid/egid/groups ----> AUTH_SYS
SSSD/NSS on server ----> owner 名称解析、manage-gids、运维审计
```

生产要求是稳定且全局唯一的 `uidNumber/gidNumber`，不是“用户名看起来一样”。目录服务不可用或缓存过期还可能使 `getent` 变慢，从而放大登录、应用启动、NFSv4 映射或服务端组重建延迟。

### 2.6 附加组与 16 组边界

RFC 5531 的 AUTH_SYS `gids` 列表最多包含 16 项。用户属于大量组时，客户端不能在经典凭据中完整发送所有附加组，可能出现“前几个组有权限、后面的组无权限”。组排序和具体选择由客户端实现决定，不能依赖某个业务组总能落入前 16 项。

Linux nfs-utils 的 `rpc.mountd --manage-gids` 可让服务端根据 UID 通过 NSS 重建附加组，从而绕过客户端列表不完整问题。但它引入新的生产依赖：

- 服务端必须能通过 NSS/SSSD/LDAP得到权威组列表；
- 客户端与服务端数字身份必须一致；
- 目录服务延迟会进入权限路径；
- 缓存和组变更生效时间必须验证；
- 某些 NAS 或非 Linux 服务端没有等价能力。

Kerberos 解决的是强身份认证，不自动消除所有本地组映射与 ACL 设计问题。

## 3. 端到端权限判定流程

### 3.1 文件操作状态链

```text
1. Java Files.newByteChannel("/mnt/app/a")
          |
2. JVM -> openat(2)
          |
3. 客户端 VFS/NFS 使用进程 fsuid/fsgid/附加组
          |
4. NFS RPC 使用 sec=sys 或 RPCSEC_GSS
          |
5. 服务端定位 export、检查 ro/rw 与客户端范围
          |
6. root_squash/all_squash/anonuid/anongid 转换凭据
          |
7. 服务端 VFS 检查 inode owner/mode/POSIX ACL 或实现 ACL
          |
8. LSM/SELinux、只读文件系统、不可变属性、配额等继续检查
          |
9. 成功执行或返回 NFS3ERR_ACCES/NFS4ERR_ACCESS 等状态
          |
10. 客户端转换为 EACCES/EPERM/EROFS，Java 抛出异常
```

这是一条分析顺序，不表示每个内核 hook 都严格串行或只执行一次。排障目标是确定拒绝发生在哪一层，而不是看到 `Permission denied` 就修改目录为 `0777`。

### 3.2 export 与文件系统权限是叠加关系

| 层次 | 示例 | 能否被下一层放宽 |
| --- | --- | --- |
| export 客户端范围 | `10.10.0.0/16` | 不能靠 chmod 绕过 |
| export 读写属性 | `ro`/`rw` | `ro` 不能靠目录写权限绕过 |
| 安全 flavor | `sec=sys`/`krb5i` | 客户端必须使用服务端允许的 flavor |
| squash | `root_squash`/`all_squash` | 决定进入 VFS 的有效身份 |
| DAC/ACL | owner、mode、ACL mask | root squash 后仍按匿名身份检查 |
| SELinux/LSM | type enforcement | chmod 不能绕过 AVC |
| 后端状态 | 只读文件系统、immutable、配额 | export `rw` 也不能放宽 |

### 3.3 Linux DAC 判定

简化的权限选择逻辑：

```text
请求 fsuid == inode uid ?
  -> 使用 owner 权限
否则是否匹配 inode gid 或有效 ACL group？
  -> 使用 group/ACL 与 ACL mask 的交集
否则
  -> 使用 other 权限
```

目录权限需要特别区分：

| 权限 | 目录语义 |
| --- | --- |
| read | 读取目录项名称 |
| write | 创建、删除、重命名目录项，通常还需要 execute |
| execute | 路径穿越和访问已知名称 |

能够读取文件内容不等于能在其目录中 rename；能够写文件内容也不等于能删除文件，删除主要由父目录权限、sticky bit 和 ACL 决定。

### 3.4 POSIX ACL 与 mask

POSIX ACL 中的 `mask::` 限制 named user、named group 和 `group::` 的最大有效权限。下面的 ACL 即使写着 `group:appwriters:rwx`，若 mask 是 `r-x`，有效权限仍没有 write：

```text
user::rwx
group::r-x
group:appwriters:rwx        #effective:r-x
mask::r-x
other::---
```

default ACL 只影响目录下新创建对象，不会追溯修改已有文件。setgid 目录让新文件通常继承目录 GID，但最终 mode 仍受创建 mode、umask、default ACL、应用显式 chmod 和服务端实现影响。

### 3.5 squash 转换

| export 选项 | 凭据转换 | 生产定位 |
| --- | --- | --- |
| `root_squash` | 客户端 UID/GID 0 请求映射为匿名身份 | 默认安全基线 |
| `no_root_squash` | 保留客户端 root 语义 | 高风险，默认禁止 |
| `all_squash` | 所有请求映射为匿名身份 | 匿名投递、单一服务身份场景可评估 |
| `anonuid/anongid` | 指定匿名数字 UID/GID | 必须使用专用、稳定、最小权限身份 |
| `no_all_squash` | 非 root 用户保持原数字身份 | 常见默认行为 |

默认匿名 ID 常见为 65534，但用户名可能是 `nobody` 或 `nfsnobody`，不能硬编码名称。应以 `exportfs -v`、`getent passwd 65534`、目标发行版和存储实现为准。

### 3.6 SELinux 与 NFS

在 Linux NFS 服务端，nfsd 最终访问本地后端文件系统，SELinux 可以在 DAC/ACL 允许后继续拒绝。客户端看到的通常只有 `EACCES`，根因证据存在服务端 AVC 日志。

普通 NFS 挂载不等于客户端获得服务端本地 SELinux label 的完整语义。Labeled NFS 是独立能力，依赖 NFS 版本、内核、export、客户端策略和存储实现，不能因为两端都启用 SELinux 就假定标签端到端传播。

## 4. 配置、命令与权限模型

### 4.1 建立两端身份基线

**执行端：两端**  
**适用范围：只读检查；将 `appsvc` 和路径替换为实际测试对象**

```bash
TEST_USER=appsvc
TEST_PATH=/mnt/app

uname -r
id "$TEST_USER"
getent passwd "$TEST_USER"
getent group "$(id -gn "$TEST_USER")"
getent initgroups "$TEST_USER" 2>/dev/null || true
namei -om "$TEST_PATH" 2>/dev/null || true
stat -c 'path=%n uid=%u gid=%g mode=%a type=%F' "$TEST_PATH"
getfacl -cp "$TEST_PATH" 2>/dev/null || true
```

预期观察：客户端进程使用的 UID/GID、服务端 inode 的数字 owner/group、附加组和每一级目录穿越权限可以对应。不要只保存用户名输出；必须保存数字 ID。

### 4.2 AUTH_SYS 导出基线

`/etc/exports` 测试示例：

```exports
/srv/nfs/app  10.10.0.0/16(rw,sync,root_squash,no_subtree_check,sec=sys)
```

**适用范围：** 受控企业网络中的兼容场景，客户端主机由同一安全域管理，UID/GID 具备统一治理。  
**预期效果：** 非 root 业务用户保留数字 UID/GID，客户端 root 被 squash；服务端继续执行 DAC/ACL/SELinux。  
**风险：** AUTH_SYS 可被受控客户端上的特权主体伪造，不适合跨不可信租户或不可信客户端边界。  
**回滚：** 恢复经审批的上一版 exports；在变更窗口重新加载并验证单个目标导出，不能无评估地放宽为 `no_root_squash`。

**执行端：服务端**  
**风险：`exportfs -ra` 会重新加载全部导出，应在测试和变更窗口执行**

```bash
sudo exportfs -v
sudo exportfs -ra
sudo exportfs -v
sudo journalctl -u nfs-server -u nfs-kernel-server --since "5 min ago" --no-pager
```

### 4.3 `all_squash` 专用身份示例

当业务明确要求所有客户端写入都映射为单一服务身份，可评估：

```exports
/srv/nfs/dropbox  10.10.20.0/24(rw,sync,all_squash,anonuid=22000,anongid=22000,no_subtree_check,sec=sys)
```

实施前必须确认服务端 UID/GID 22000 是专用非登录身份、目录仅授予所需权限、没有与现有用户冲突。

| 项目 | 要求 |
| --- | --- |
| 适用场景 | 单一应用身份、匿名投递、无需按用户审计 |
| 风险 | 所有用户在文件系统层不可区分，审计粒度下降 |
| 验证 | 不同客户端用户创建文件后，服务端 owner/group 均为 22000 |
| 回滚 | 恢复原 squash 策略前先评估既有文件 owner，避免新旧身份都失去访问 |

`no_root_squash` 不是解决容器写入、安装程序 chown 或 root 运维便利性的通用方案。它允许受信任范围内任意客户端 root 以服务端 root 语义操作导出，可能修改 owner、权限和业务数据。生产默认禁止；确有遗留需求时应使用独立导出、最小客户端范围、只读优先、网络隔离、审计和限时回滚。

### 4.4 setgid 目录与 POSIX ACL 基线

**执行端：服务端**  
**适用范围：支持 POSIX ACL 的后端文件系统；只创建并修改固定名称测试子目录**  
**风险：ACL 修改立即影响所有客户端；禁止把 `EXPORT_DIR` 指向导出根或已有业务目录**

```bash
EXPORT_ROOT=/srv/nfs/app
EXPORT_DIR="$EXPORT_ROOT/.nfs-kb-l3-01-acl-baseline"

test -d "$EXPORT_ROOT" || { echo "export root not found" >&2; exit 1; }
test ! -e "$EXPORT_DIR" || { echo "test directory already exists" >&2; exit 1; }
getent group appwriters >/dev/null || { echo "group appwriters not found" >&2; exit 1; }

sudo install -d -o root -g appwriters -m 2770 "$EXPORT_DIR"
umask 077
ACL_BACKUP=$(mktemp /var/tmp/nfs-kb-l3-01-acl.XXXXXX)

sudo getfacl -p "$EXPORT_DIR" > "$ACL_BACKUP"
sudo setfacl -m g:appwriters:rwx,m::rwx "$EXPORT_DIR"
sudo setfacl -d -m u::rwx,g::rwx,g:appwriters:rwx,m::rwx,o::--- "$EXPORT_DIR"

getfacl -p "$EXPORT_DIR"
printf 'Record this ACL backup path for rollback: %s\n' "$ACL_BACKUP"
```

预期效果：目录保持 setgid，新对象继承目标 GID；default ACL 为新对象提供基线，ACL mask 不截断 `appwriters` 写权限。

回滚不要直接执行 `setfacl -b`，它会删除全部扩展 ACL。应审查备份后恢复：

**执行端：服务端**  
**风险：ACL restore 会覆盖目标当前 ACL；必须填写并核对刚才记录的绝对备份路径**

```bash
EXPORT_DIR=/srv/nfs/app/.nfs-kb-l3-01-acl-baseline
ACL_BACKUP=/var/tmp/nfs-kb-l3-01-acl.REPLACE_WITH_RECORDED_SUFFIX

test "$EXPORT_DIR" = /srv/nfs/app/.nfs-kb-l3-01-acl-baseline || exit 1
test -d "$EXPORT_DIR" || { echo "test directory not found" >&2; exit 1; }
test -s "$ACL_BACKUP" || { echo "ACL backup not found" >&2; exit 1; }
sudo setfacl --restore="$ACL_BACKUP"
getfacl -p "$EXPORT_DIR"
```

### 4.5 服务端 `manage-gids`

当用户附加组超过 AUTH_SYS 上限且服务端拥有权威 NSS 数据，可在目标 nfs-utils 版本评估 `/etc/nfs.conf`：

```ini
[mountd]
manage-gids = y
```

**适用范围：** Linux knfsd/nfs-utils，服务端 NSS/SSSD/LDAP 可用且 UID/GID 全局一致。  
**预期效果：** 服务端根据请求 UID 重建附加组，而不是只使用客户端传入的有限组列表。  
**风险：** 目录服务延迟、缓存不一致或 UID 冲突会进入权限检查链；组变更生效时间可能变化。  
**回滚：** 恢复 `manage-gids = n` 或上一版配置，只重启目标 mountd 服务，并验证现有挂载和权限；服务名依发行版确认。

**执行端：服务端**  
**风险：服务重启可能影响 NFSv3 mount 控制面；先确认服务名和变更窗口**

```bash
rpc.mountd --help 2>&1 | grep -E 'manage-gids|manage_gids' || true
grep -Rni 'manage-gids' /etc/nfs.conf /etc/nfs.conf.d 2>/dev/null || true
systemctl list-unit-files | grep -E 'nfs-mountd|rpc-mountd|nfs-server|nfs-kernel-server'
```

不要在未建立 LDAP/SSSD 延迟基线前启用。NAS、云文件服务和非 Linux NFS 服务端应查厂商是否提供等价组展开能力。

### 4.6 NFSv4 idmap 配置检查

`/etc/idmapd.conf` 常见配置：

```ini
[General]
Domain = example.com
```

**适用范围：** 实际启用 NFSv4 name mapping 的客户端和服务端。  
**预期效果：** 双方对无显式域的 NFSv4 owner/owner_group 使用相同映射域。  
**风险：** 单边修改 Domain 或切换数字映射模式可能让既有 owner 显示为匿名身份，并污染缓存。  
**回滚：** 恢复双方上一版配置，按目标发行版流程重载映射组件并在维护窗口清理缓存，不重启全部 NFS 业务。

清理 idmap key 会影响本机所有相关 NFSv4 挂载，只能在记录现状并确认影响后执行：

**执行端：客户端或服务端对应节点**  
**风险：高；以下命令会清理 NFSv4 idmap 缓存，不得在未评估的生产节点执行**

```bash
sudo nfsidmap -c
```

清理后必须重新读取测试文件 owner/group，观察新 key、日志和 NSS 查询；缓存清理不是 Domain、NSS 或 UID 冲突的根因修复。

### 4.7 SELinux 只读取证

**执行端：服务端；必要时客户端也执行**  
**适用范围：SELinux enabled；只读诊断**

```bash
getenforce
ls -ldZ /srv/nfs /srv/nfs/app 2>/dev/null || true
getsebool -a 2>/dev/null | grep -E '(^|_)nfs|use_nfs|virt_use_nfs|container_use_nfs' || true
sudo ausearch -m AVC,USER_AVC -ts recent 2>/dev/null | tail -n 100
```

不要用 `setenforce 0`、长期 permissive、`chcon` 临时标签或全局放宽 `nfs_export_all_rw` 作为常规修复。应根据发行版 SELinux policy，为目标路径建立最小持久化 `semanage fcontext` 规则并使用 `restorecon`，同时记录原规则、AVC、验证和回滚。不同发行版 policy type/boolean 不同，本文不提供可无条件复制的放宽命令。

### 4.8 NFSv4 ACL 与 POSIX ACL 边界

NFSv4 ACL 支持 allow/deny ACE、继承标记和 `OWNER@`、`GROUP@`、`EVERYONE@` 等主体，表达能力不同于 POSIX ACL。Linux knfsd、后端文件系统和 NAS 可能执行转换、裁剪或厂商扩展。

**执行端：两端**  
**适用范围：只读检查；工具存在且目标导出声明支持 NFSv4 ACL**

```bash
TEST_PATH=/mnt/app
getfacl -p "$TEST_PATH" 2>/dev/null || true
if command -v nfs4_getfacl >/dev/null; then
    nfs4_getfacl "$TEST_PATH"
fi
```

`getfacl` 的输出不能证明完整 NFSv4 ACL 语义；`nfs4_getfacl` 可用也不证明后端保留所有 ACE。任何跨 NAS、Linux、SMB/NFS 双协议访问的 ACL 设计都要做逐项读回和访问矩阵验证。

## 5. Java 工程视角

### 5.1 Java 没有独立的 NFS 用户身份

Java 文件 API 最终使用 JVM 进程在 Linux 上的凭据。标准 JDK 没有可移植 API 完整返回 Linux effective UID、fsuid、主 GID 和附加组；`System.getProperty("user.name")` 只是名称信息，不能用于证明内核访问身份。

```text
systemd/Kubernetes 启动 JVM
  -> Linux uid/gid/groups/capabilities
  -> Java Files/FileChannel
  -> openat/read/write/fchmod/fchown
  -> NFS AUTH_SYS credential
```

应用身份变更后只修改 LDAP 或 `/etc/group` 通常不会改变已经运行 JVM 的附加组。需要按变更流程重启或重新登录进程，并用 `/proc/<pid>/status` 的 `Uid:`、`Gid:`、`Groups:` 验证实际凭据。

**执行端：客户端**  
**适用范围：只读检查目标 JVM**

```bash
PID=$(pgrep -n -f 'java.*myapp')
test -n "$PID" || { echo "JVM not found" >&2; exit 1; }
grep -E '^(Name|Uid|Gid|Groups|Cap)' "/proc/$PID/status"
readlink -f "/proc/$PID/exe"
```

### 5.2 Java 异常与 Linux/NFS 语义

| Java 表现 | Linux/NFS 方向 | 必须验证 |
| --- | --- | --- |
| `AccessDeniedException` | `EACCES`/`EPERM`、ACL、squash、SELinux | 进程数字身份、父目录、服务端 AVC |
| `FileSystemException: Read-only file system` | `EROFS`、export `ro`、后端只读 | `findmnt`、`exportfs -v`、服务端 mount |
| `NoSuchFileException` | 通常对应 `ENOENT`，目标或某个路径组件不存在 | 路径、缓存、rename 时间线；父目录缺少 execute 通常更接近 `AccessDeniedException` |
| `FileAlreadyExistsException` | `CREATE_NEW` 冲突 | 是否为并发正常分支 |
| owner 显示 `nobody` | NFSv4 name mapping 或匿名文件 | 数字 owner、Domain、idmap cache、squash |

同一个 NFS status 经过客户端内核和 JDK 后可能呈现为相近异常。必须记录 `exception.getFile()`、`getReason()`、操作阶段、JVM 主机、PID、挂载 source/options 和时间戳。

### 5.3 `Files.isWritable()` 不是授权证明

`Files.isWritable(path)` 只能进行当时的可写性判断；路径状态、ACL、父目录、export 状态和网络可能随后变化。它还不能替代真实 create/rename/fsync 测试。不要采用：

```text
if (Files.isWritable(path)) {
    // 假定后续一定可以写
}
```

正确设计是直接执行目标操作，处理明确异常，并让业务写入具备幂等、临时文件、同目录 rename 和校验机制。

### 5.4 最小权限诊断程序

下面程序只适用于专用测试目录。它记录进程名称信息、文件数字 UID/GID/mode 和写入阶段；它不能证明 AUTH_SYS 强身份安全。

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.charset.StandardCharsets;
import java.nio.file.AccessDeniedException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.Set;

public final class NfsPermissionProbe {
    private NfsPermissionProbe() {
    }

    public static void main(String[] args) throws IOException {
        Path dir = args.length == 1
                ? Path.of(args[0])
                : Path.of("/mnt/app/.nfs-kb-l3-01");
        Path file = dir.resolve("java-permission-test");

        System.out.printf("user.name=%s dir=%s%n",
                System.getProperty("user.name"), dir);

        try {
            Files.createDirectories(dir);
            ByteBuffer buffer = ByteBuffer.wrap("permission-test\n"
                    .getBytes(StandardCharsets.UTF_8));
            try (FileChannel channel = FileChannel.open(file,
                    Set.of(StandardOpenOption.CREATE_NEW, StandardOpenOption.WRITE))) {
                while (buffer.hasRemaining()) {
                    channel.write(buffer);
                }
                channel.force(true);
            }

            Object uid = Files.getAttribute(file, "unix:uid");
            Object gid = Files.getAttribute(file, "unix:gid");
            Object mode = Files.getAttribute(file, "unix:mode");
            System.out.printf("created=%s uid=%s gid=%s mode=%s%n",
                    file, uid, gid, Integer.toOctalString((Integer) mode));
        } catch (AccessDeniedException e) {
            System.err.printf("denied file=%s other=%s reason=%s%n",
                    e.getFile(), e.getOtherFile(), e.getReason());
            throw e;
        } catch (IOException e) {
            System.err.printf("io-error type=%s message=%s%n",
                    e.getClass().getName(), e.getMessage());
            throw e;
        }
    }
}
```

`unix:*` 属性是 Linux/Unix provider 扩展，不是所有 JDK 平台都支持。实验结束应由具备权限的测试流程删除文件，并在服务端核对数字 owner/group 和 ACL。

### 5.5 创建、rename 和删除需要不同权限

Java 原子发布常用“同目录临时文件 + `FileChannel.force(true)` + rename”。其权限依赖包括：

| 操作 | 主要权限对象 |
| --- | --- |
| 创建临时文件 | 父目录 write + execute、default ACL、export rw |
| 写入和 force | 文件写权限、配额、后端可写 |
| rename | 源/目标父目录 write + execute、sticky bit、ACL |
| 删除 | 父目录权限和 sticky bit，不仅是文件自身 mode |
| 修改 owner | 服务端权限、squash、capability；普通用户通常受限 |

因此“能够覆盖已有文件但不能创建新文件”或“能写不能 rename”并不矛盾。应分别复现具体系统调用阶段。

## 6. 生产实践与风险边界

### 6.1 安全等级选择

| 信任边界 | 推荐起点 | AUTH_SYS 是否足够 |
| --- | --- | --- |
| 同一受控主机群、单租户、隔离网络 | `sec=sys` + root squash + UID/GID 治理 | 可评估，仍需审计和最小权限 |
| 多团队但客户端由统一基础设施控制 | 强制数字身份治理，敏感路径评估 Kerberos | 仅靠 AUTH_SYS 风险较高 |
| 不可信客户端、跨租户、跨组织 | RPCSEC_GSS/Kerberos 或其他强认证存储 | 不足 |
| 需要链路完整性/机密性 | `krb5i/krb5p` 或网络层等价保护 | 不足 |
| 互联网或弱隔离网络 | 不直接暴露 NFS | 不足 |

### 6.2 数字身份治理基线

生产必须维护：

- UID/GID 分配范围和唯一性；
- 服务账号、人员账号、容器账号的分区；
- 禁止回收后立即复用 UID/GID；
- LDAP/SSSD 缓存、离线和故障策略；
- 客户端与服务端 `nsswitch.conf` 数据源差异；
- NFSv4 Domain 和数字 idmapping 模式；
- 账号变更后进程重启与附加组生效流程。

删除用户名不会删除 inode 上的数字 owner。未来新用户复用同一 UID，会继承旧文件访问权，这是身份生命周期风险，不是 NFS 缓存问题。

### 6.3 容器与 Kubernetes

容器内用户名不决定 NFS 身份，最终以客户端 Linux 内核向 NFS 发出的数字凭据为准。

```text
Pod securityContext
  runAsUser: 20001
  runAsGroup: 30001
  fsGroup: 30002
        |
        v
容器进程 Linux credentials
        |
        v
节点 NFS 客户端 AUTH_SYS
        |
        v
服务端 UID/GID/ACL
```

生产注意：

1. `fsGroup` 是否应用、是否递归 chown、`fsGroupChangePolicy` 是否生效取决于 volume plugin/CSI driver 和版本。
2. `root_squash` 下 kubelet 或 initContainer 以 root 对 NFS 执行 chown 可能失败，这是安全基线的预期结果。
3. 优先在服务端预创建目录和 ACL，或使用 CSI/存储平台支持的身份和目录供应机制。
4. 不要为了让 Pod 启动成功改为 `no_root_squash` 或 `0777`。
5. user namespace/remap 可能让容器 UID 0 在节点上成为非零 UID；必须实测最终 wire UID，不能按容器内 `id` 推断。

### 6.4 SELinux 风险边界

权限故障时临时关闭 SELinux会删除关键证据并扩大攻击面。正确路径是：

```text
保存 AVC
  -> 确认目标进程 domain、文件 type 和请求权限
  -> 判断是否为错误标签、缺少受控 boolean 或真实策略违规
  -> 使用持久化最小规则修复
  -> enforcing 下复测
```

`audit2allow` 生成的规则不能未经审查直接加载；它只根据观测到的拒绝生成允许建议，不理解业务最小权限。

### 6.5 SMB/NFS 双协议访问

同一数据集同时通过 SMB 和 NFS 暴露时，Windows SID、NFSv4 ACL、POSIX UID/GID 和 POSIX ACL 之间的转换由存储实现决定。Linux `chmod/setfacl` 可能重写或裁剪 SMB ACL，反之亦然。必须使用存储厂商的 identity mapping 和 ACL 模式，不把 Linux knfsd 结论直接套用 NAS。

### 6.6 禁止的通用修复

| 错误修复 | 为什么危险 |
| --- | --- |
| `chmod -R 777` | 破坏最小权限、ACL mask 和审计边界，递归操作还可能长时间阻塞 NFS |
| `chown -R` | 大规模元数据写、身份破坏、不可逆覆盖原 owner |
| `no_root_squash` | 把客户端 root 提升为服务端 root 语义 |
| 全局 `all_squash` | 所有用户失去区分，可能破坏既有 owner 和审计 |
| 关闭 SELinux | 隐藏根因并扩大系统权限 |
| 只清理 idmap/SSSD cache | 缓存会重建，不能修复 Domain 或 UID 冲突 |
| 反复重试 Java 写入 | 权限拒绝通常不是瞬时错误，可能放大日志和负载 |

## 7. 验证实验与观察指标

以下实验必须使用隔离测试导出和脱敏账号。当前无真实 Linux NFS 实验环境，因此本文保持“待验证”。

### 7.1 实验一：同名不同 UID

**执行端：两台测试客户端**  
**前提：不创建生产账号；使用已有测试身份和测试目录**

```bash
TEST_USER=appsvc
CLIENT_ID=${CLIENT_ID:?set CLIENT_ID to client-a or client-b}
case "$CLIENT_ID" in
    client-a|client-b) ;;
    *) echo "invalid CLIENT_ID" >&2; exit 1 ;;
esac
TEST_FILE="/mnt/app/.nfs-kb-l3-01-uid-$CLIENT_ID"

id "$TEST_USER"
getent passwd "$TEST_USER"
test ! -e "$TEST_FILE" || { echo "test file already exists" >&2; exit 1; }
sudo -u "$TEST_USER" sh -c \
  'set -C; umask 0027; printf "%s\n" "$(id)" > "$1"' sh "$TEST_FILE"
stat -c 'uid=%u gid=%g mode=%a name=%n' "$TEST_FILE"
ls -ln "$TEST_FILE"
```

在客户端 A 设置 `CLIENT_ID=client-a`，客户端 B 设置 `CLIENT_ID=client-b`。两个客户端必须创建不同 inode，不能通过覆盖同一文件比较 owner。

**执行端：服务端**  
**适用范围：与客户端测试文件对应的后端路径；只读核对**

```bash
for TEST_FILE in \
  /srv/nfs/app/.nfs-kb-l3-01-uid-client-a \
  /srv/nfs/app/.nfs-kb-l3-01-uid-client-b; do
    stat -c 'uid=%u gid=%g mode=%a name=%n' "$TEST_FILE"
    getent passwd "$(stat -c %u "$TEST_FILE")" || true
    getent group "$(stat -c %g "$TEST_FILE")" || true
done
```

记录三端的数字 UID/GID 和名称解析。预期结论以数字身份为准；用户名差异只影响显示和本机运维语义。

### 7.2 实验二：root squash

**执行端：测试客户端**  
**风险：只操作专用测试文件；确认导出启用 `root_squash`**

客户端：

```bash
TEST_DIR=/mnt/app/.nfs-kb-l3-01-root-squash
umask 077
ERROR_LOG=$(mktemp /var/tmp/nfs-kb-l3-01-root-squash.XXXXXX)

mkdir_rc=0
sudo mkdir "$TEST_DIR" 2>"$ERROR_LOG" || mkdir_rc=$?
write_rc=not-run
if [ "$mkdir_rc" -eq 0 ]; then
    write_rc=0
    sudo sh -c 'set -C; printf root-test > "$1/client-root-file"' \
      sh "$TEST_DIR" 2>>"$ERROR_LOG" || write_rc=$?
fi

printf 'mkdir_rc=%s write_rc=%s error_log=%s\n' \
  "$mkdir_rc" "$write_rc" "$ERROR_LOG"
cat "$ERROR_LOG"
rm -f "$ERROR_LOG"
```

**执行端：服务端**  
**适用范围：只读取证；不要修改匿名身份或文件 owner**

```bash
TEST_DIR=/srv/nfs/app/.nfs-kb-l3-01-root-squash
stat -c 'uid=%u gid=%g mode=%a name=%n' "$TEST_DIR" "$TEST_DIR/client-root-file" 2>/dev/null || true
sudo exportfs -v | sed -n '/\/srv\/nfs\/app/,+2p'
```

预期观察：客户端 root 不应自动获得服务端 root 权限；能否创建取决于匿名身份对父目录的权限。若文件存在，其 owner 应按实际匿名映射验证，不能只看名字 `nobody`。

### 7.3 实验三：ACL mask 与 default ACL

**执行端：服务端**  
**风险：只使用测试目录；先保存 ACL 备份**

服务端设置测试 ACL：

```bash
TEST_DIR=/srv/nfs/app/.nfs-kb-l3-01-acl
test ! -e "$TEST_DIR" || { echo "test directory already exists" >&2; exit 1; }
getent group appwriters >/dev/null || { echo "group appwriters not found" >&2; exit 1; }
sudo install -d -o root -g appwriters -m 2770 "$TEST_DIR"
umask 077
TEST_ACL_BACKUP=$(mktemp /var/tmp/nfs-kb-l3-01-test-acl.XXXXXX)
sudo getfacl -p "$TEST_DIR" > "$TEST_ACL_BACKUP"
sudo setfacl -m g:appwriters:rwx,m::r-x "$TEST_DIR"
sudo setfacl -d -m u::rwx,g::rwx,g:appwriters:rwx,m::rwx,o::--- "$TEST_DIR"
getfacl -p "$TEST_DIR"
printf 'Record this test ACL backup path for cleanup: %s\n' "$TEST_ACL_BACKUP"
```

**执行端：客户端**  
**适用范围：使用 `appwriters` 组成员验证 ACL mask**

```bash
TEST_USER=appsvc
TEST_DIR=/mnt/app/.nfs-kb-l3-01-acl
id "$TEST_USER"
umask 077
ERROR_LOG=$(mktemp /var/tmp/nfs-kb-l3-01-acl-mask.XXXXXX)
touch_rc=0
sudo -u "$TEST_USER" touch "$TEST_DIR/mask-test" 2>"$ERROR_LOG" || touch_rc=$?
printf 'touch_rc=%s error_log=%s\n' "$touch_rc" "$ERROR_LOG"
cat "$ERROR_LOG"
rm -f "$ERROR_LOG"
```

预期观察：access ACL mask 为 `r-x` 时，named group 的写权限被截断；default ACL 不会放宽当前目录自身的 access ACL。修复 mask 后，新文件应继承预期 ACL/GID。

本步骤中 `touch_rc` 应为非零；若为零，必须重新检查实际进程 Groups、ACL mask、目录 owner 和服务端读回结果，不能把它记录为实验通过。

### 7.4 实验四：超过 16 个附加组

**执行端：测试客户端与服务端**  
**风险：不要为实验批量修改生产组；使用已有测试目录服务账号**

```bash
TEST_USER=groupheavy
GROUPS=$(id -G "$TEST_USER")
printf 'user=%s group_count=%s groups=%s\n' \
  "$TEST_USER" "$(printf '%s\n' "$GROUPS" | wc -w)" "$GROUPS"
getent initgroups "$TEST_USER" 2>/dev/null || true
```

选择一个仅由较后附加组授权的测试目录，分别在 `manage-gids` 关闭和受控开启时执行创建测试，记录：客户端 `id`、抓包 AUTH_SYS gids、服务端 NSS 结果、mountd 配置、访问结果和查询延迟。不得仅凭“组数量大于 16”直接认定根因。

### 7.5 实验五：NFSv4 owner 映射

**执行端：服务端**  
**风险：只创建固定名称测试文件；若文件已存在则停止，不覆盖现场**

```bash
TEST_USER=appsvc
TEST_FILE=/srv/nfs/app/.nfs-kb-l3-01-idmap
test ! -e "$TEST_FILE" || { echo "test file already exists" >&2; exit 1; }
sudo install -o "$(id -u "$TEST_USER")" -g "$(id -g "$TEST_USER")" \
  -m 0640 /dev/null "$TEST_FILE"
stat -c 'uid=%u gid=%g user=%U group=%G' "$TEST_FILE"
```

**执行端：NFSv4 客户端**  
**适用范围：只读取证；使用与服务端测试文件对应的挂载路径**

```bash
TEST_FILE=/mnt/app/.nfs-kb-l3-01-idmap
nfsstat -m
ls -ln "$TEST_FILE"
ls -l "$TEST_FILE"
stat -c 'uid=%u gid=%g user=%U group=%G' "$TEST_FILE"
getent passwd "$(stat -c %u "$TEST_FILE")" || true
getent group "$(stat -c %g "$TEST_FILE")" || true
command -v nfsidmap >/dev/null && sudo nfsidmap -l || true
journalctl -b | grep -Ei 'nfsidmap|idmapd|nfs4.*id' | tail -n 100
```

预期观察：区分“数字 UID 正确但本机无名称”“NFSv4 owner 映射失败”“文件本身由匿名身份创建”三种不同情况。只有在记录双方 Domain、数字模式、NSS 和缓存后，才能决定是否清理 idmap cache。

### 7.6 AUTH_SYS 抓包验证

**执行端：隔离测试客户端或受控镜像点**  
**风险：抓包可能包含文件名、UID/GID、Kerberos 元数据或业务信息；限制时长、权限和保存位置，完成后按安全流程清理**

```bash
NFS_SERVER=192.0.2.10
umask 077
CAPTURE=$(mktemp /var/tmp/nfs-kb-l3-01-authsys.XXXXXX.pcap)

sudo timeout 30s tcpdump -i any -s 0 -w "$CAPTURE" \
  "host $NFS_SERVER and port 2049"
sudo chmod 0600 "$CAPTURE"
printf 'Record this capture path for cleanup: %s\n' "$CAPTURE"

if command -v tshark >/dev/null; then
    tshark -r "$CAPTURE" -Y 'rpc.auth_flavor == 1' \
      -T fields -e frame.time -e ip.src -e ip.dst -e rpc.xid
fi
```

预期观察：确认目标请求使用 AUTH_SYS flavor，并将 XID、时间戳与客户端操作对应。字段名称随 Wireshark 版本变化；若要提取 UID/GID 和 gids，先用 `tshark -G fields | grep -i auth` 确认本机字段，不在生产脚本中硬编码未经验证的字段名。

### 7.7 实验清理与证据归档

先保存实验拓扑、两端数字身份、挂载参数、export、ACL、AVC、抓包摘要和实际结果，再清理专用测试对象。清理命令只列出本文创建的固定测试名称，不使用递归通配删除。

**执行端：测试客户端**  
**风险：只清理已记录的本机抓包；NFS 测试对象由服务端按后端路径清理，避免客户端 root 被 squash 后清理失败**

```bash
CAPTURE=${CAPTURE:-}
if [ -n "$CAPTURE" ]; then
    case "$CAPTURE" in
        /var/tmp/nfs-kb-l3-01-authsys.*.pcap) sudo rm -f -- "$CAPTURE" ;;
        *) echo "unexpected capture path" >&2; exit 1 ;;
    esac
fi
```

**执行端：服务端**  
**风险：先核对 ACL 备份内容；测试目录不为空时停止并人工确认，不使用 `rm -rf`**

```bash
TEST_ACL_DIR=/srv/nfs/app/.nfs-kb-l3-01-acl
ACL_BASELINE_DIR=/srv/nfs/app/.nfs-kb-l3-01-acl-baseline

if [ -n "${TEST_ACL_BACKUP:-}" ]; then
    case "$TEST_ACL_BACKUP" in
        /var/tmp/nfs-kb-l3-01-test-acl.*)
            sudo setfacl --restore="$TEST_ACL_BACKUP" || exit 1
            ;;
        *) echo "unexpected test ACL backup path" >&2; exit 1 ;;
    esac
fi
if [ -n "${ACL_BACKUP:-}" ]; then
    case "$ACL_BACKUP" in
        /var/tmp/nfs-kb-l3-01-acl.*)
            sudo setfacl --restore="$ACL_BACKUP" || exit 1
            ;;
        *) echo "unexpected ACL backup path" >&2; exit 1 ;;
    esac
fi

sudo rm -f /srv/nfs/app/.nfs-kb-l3-01-uid-client-a
sudo rm -f /srv/nfs/app/.nfs-kb-l3-01-uid-client-b
sudo rm -f /srv/nfs/app/.nfs-kb-l3-01-idmap
sudo rm -f /srv/nfs/app/.nfs-kb-l3-01-root-squash/client-root-file
sudo rm -f "$TEST_ACL_DIR/mask-test"
sudo rm -f /srv/nfs/app/.nfs-kb-l3-01/java-permission-test

cleanup_ok=1
for dir in \
  /srv/nfs/app/.nfs-kb-l3-01-root-squash \
  "$TEST_ACL_DIR" \
  /srv/nfs/app/.nfs-kb-l3-01 \
  "$ACL_BASELINE_DIR"; do
    if [ -d "$dir" ] && ! sudo rmdir "$dir"; then
        echo "directory not empty; retain backups and inspect: $dir" >&2
        cleanup_ok=0
    fi
done

if [ "$cleanup_ok" -eq 1 ]; then
    if [ -n "${TEST_ACL_BACKUP:-}" ]; then sudo rm -f -- "$TEST_ACL_BACKUP"; fi
    if [ -n "${ACL_BACKUP:-}" ]; then sudo rm -f -- "$ACL_BACKUP"; fi
else
    exit 1
fi
```

在新 shell 执行清理时，应先把实验输出中记录的路径赋给 `TEST_ACL_BACKUP`、`ACL_BACKUP` 和 `CAPTURE`；未运行相应实验则保持变量为空。若 `rmdir` 失败，表示目录仍有测试文件、打开引用或非预期内容，应停止自动清理并核对现场。只有 ACL restore 成功后才能删除备份；抓包删除应遵循组织的数据留存和介质清理规范，普通 `rm` 不保证底层介质不可恢复。

### 7.8 必须记录的观察指标

| 维度 | 证据 | 目的 |
| --- | --- | --- |
| 进程身份 | `/proc/PID/status`、`id` | 确认真实 euid/gid/groups |
| NSS | `getent` 结果与耗时 | 确认数字名称和组来源 |
| 挂载 | `findmnt`、`nfsstat -m` | 确认版本、source、`sec=`、ro/rw |
| export | `exportfs -v` | 确认生效客户端范围和 squash |
| inode | `stat`、`ls -ln` | 确认数字 owner/mode |
| ACL | `getfacl`、ACL mask | 确认有效而非名义权限 |
| idmap | Domain、模块参数、key、日志 | 确认 NFSv4 owner 映射链 |
| SELinux | enforce、context、AVC | 确认 LSM 拒绝 |
| RPC | auth flavor、UID/GID/gids、NFS status | 确认 wire credential 和返回状态 |
| Java | 异常类型、reason、阶段、PID | 连接业务现象与系统证据 |

## 8. 排障证据链与检查清单

### 8.1 先按症状分类

| 症状 | 优先判断 |
| --- | --- |
| mount 阶段 `access denied by server` | export 客户端范围、伪根、`sec=`，不是目录 chmod |
| mount 成功但 open 返回 permission denied | RPC 身份、squash、父目录、ACL、SELinux |
| root 也无法写 | `root_squash`、匿名身份权限、export/backend ro |
| 文件显示 `nobody` | 匿名创建还是 NFSv4 owner 映射失败 |
| 同名用户跨节点权限不同 | 数字 UID/GID、附加组、NSS 缓存 |
| 用户属于组但仍无权限 | 进程 Groups、16 组限制、ACL mask、目录 execute |
| 容器 root 不能 chown | root squash、user namespace、CSI/fsGroup 行为 |
| chmod 777 仍拒绝 | SELinux、export ro、后端 ro、immutable、父目录 |
| Java 偶发 AccessDenied | 发布竞态、目录 ACL、进程身份变更、服务端切换 |

### 8.2 标准取证命令

**执行端：客户端**  
**适用范围：只读诊断；将 PID、用户和路径替换为目标对象；路径解析命令仍可能触发 automount 或在 hard NFS 上阻塞**

```bash
PID=12345
TEST_USER=appsvc
NFS_MOUNT=/mnt/app
TEST_PATH=/mnt/app/problem-file

# 先从 mountinfo 被动确认当前状态，再执行会解析路径的命令
grep -F " $NFS_MOUNT " /proc/self/mountinfo || true
findmnt --mountpoint "$NFS_MOUNT" -o TARGET,SOURCE,FSTYPE,OPTIONS

grep -E '^(Uid|Gid|Groups|Cap)' "/proc/$PID/status"
id "$TEST_USER"
getent passwd "$(stat -c %u "$TEST_PATH")" || true
getent group "$(stat -c %g "$TEST_PATH")" || true
namei -om "$TEST_PATH"
stat -c 'uid=%u gid=%g mode=%a type=%F name=%n' "$TEST_PATH"
getfacl -p "$TEST_PATH" 2>/dev/null || true
nfsstat -m
journalctl -k --since "10 min ago" | grep -Ei 'nfs|rpc|idmap' | tail -n 100
```

**执行端：服务端**  
**适用范围：只读诊断；将后端路径替换为与客户端同一 filehandle 对应的对象**

```bash
BACKEND_PATH=/srv/nfs/app/problem-file

sudo exportfs -v
namei -om "$BACKEND_PATH"
stat -c 'uid=%u gid=%g mode=%a type=%F name=%n' "$BACKEND_PATH"
getfacl -p "$BACKEND_PATH" 2>/dev/null || true
lsattr -d "$BACKEND_PATH" 2>/dev/null || true
findmnt --target "$BACKEND_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
getenforce 2>/dev/null || true
sudo ausearch -m AVC,USER_AVC -ts recent 2>/dev/null | tail -n 100
```

### 8.3 权限判定时间线

```text
T0 Java 异常：操作、路径、PID、主机
  -> T1 客户端进程 Uid/Gid/Groups
  -> T2 实际 mount source/version/sec/options
  -> T3 AUTH_SYS/RPCSEC_GSS 和 wire credential
  -> T4 export 客户端匹配、ro/rw、squash、anon IDs
  -> T5 服务端路径每级 owner/mode/ACL/mask
  -> T6 SELinux AVC、后端 ro/immutable/quota
  -> T7 NFS status -> errno -> Java exception
```

只要任一时间点来自不同主机、不同 PID、不同路径或不同时间窗口，结论就可能错误。尤其不要拿交互式 shell 的 `id` 代替已运行 JVM 的 `/proc/PID/status`。

### 8.4 常见错误结论

| 错误结论 | 正确分析 |
| --- | --- |
| 用户名相同就是同一用户 | NFS AUTH_SYS 以数字 UID/GID 为核心 |
| `ls -l` 显示正确就没有 idmap 问题 | 名称显示和请求认证是两条链 |
| root 一定能写 | root squash 后按匿名身份检查 |
| 组里有用户就一定能写 | 还要检查进程 Groups、16 组边界和 ACL mask |
| export `rw` 就允许写 | DAC/ACL、SELinux、后端 ro 仍可拒绝 |
| chmod 777 能证明是 SELinux | 它只破坏权限，不能提供完整根因证据 |
| 清 idmap cache 就是修复 | Domain/NSS/模式错误会再次产生同样结果 |
| Java 重试能解决 AccessDenied | 稳定权限拒绝应修复身份或策略，不应重试放大 |

### 8.5 恢复顺序

```text
摘除受影响写流量
  -> 保存进程身份、mount、export、inode、ACL、AVC 和抓包证据
  -> 修复唯一根因：UID/GID、组、ACL mask、Domain、export 或 SELinux policy
  -> 仅清理必要的 NSS/idmap 缓存
  -> 重启需要重新获得附加组的目标进程
  -> 用同一业务身份执行最小 create/read/rename/delete 测试
  -> 核对服务端数字 owner 和审计
  -> 恢复流量并观察完整业务周期
```

不要在证据保存前递归 chmod/chown、切换 `no_root_squash`、关闭 SELinux 或清理全局缓存，这些动作会改变现场并扩大影响。

### 8.6 生产检查清单

- [ ] 已记录客户端进程实际 Uid/Gid/Groups，而非只记录用户名。
- [ ] 已确认 NFS 版本、`sec=`、source、ro/rw 和真实挂载身份。
- [ ] 已用 `exportfs -v` 验证生效配置，而非只查看 `/etc/exports`。
- [ ] 已确认 root/all squash 和实际匿名数字 UID/GID。
- [ ] 已检查服务端每一级目录 execute、inode mode、ACL 和 mask。
- [ ] 已区分 NFSv4 owner 显示映射与 RPC 请求认证。
- [ ] 已确认 NSS/SSSD/LDAP 数字身份一致性和查询延迟。
- [ ] 用户组较多时已验证 AUTH_SYS 组上限与 `manage-gids` 边界。
- [ ] 已检查 SELinux AVC、后端只读、immutable 和配额。
- [ ] 容器场景已核对最终节点凭据、CSI 和 `fsGroup` 行为。
- [ ] Java 异常已关联操作阶段、PID、主机和同一时间窗口证据。
- [ ] 修复方案没有使用 `0777`、递归 chown、`no_root_squash` 或关闭 SELinux。

## 9. 小结

1. AUTH_SYS 传递客户端声明的数字 UID/GID 和有限附加组，不提供密码学身份、完整性或机密性。
2. NFSv3 的 owner 核心是数字 ID；NFSv4 owner 属性映射和请求认证必须分开分析。
3. 服务端权限是 export、squash、DAC/ACL、SELinux和后端状态的叠加结果。
4. `root_squash` 是生产默认安全基线；`no_root_squash` 不能作为容器或权限故障的通用修复。
5. POSIX ACL mask 决定 named user/group 的有效权限，default ACL 只影响新对象。
6. 超过 16 个附加组时应验证 wire credential 和服务端 `manage-gids`，不能只看客户端 `id`。
7. LDAP/SSSD 为本机提供数字身份和组数据，但不把 AUTH_SYS 变成强认证。
8. Java 使用 JVM 进程的 Linux 凭据；权限变化需要核对运行中 PID，而不是交互式 shell。
9. 企业级权限排障必须保留数字身份、mount、export、inode、ACL、AVC 和 RPC 同一时间线。

## 10. 参考资料与关联文档

### 10.1 参考资料

- RFC 5531：RPC Version 2；AUTH_SYS credential 和附加组编码边界
- RFC 1813：NFS Version 3；数字 UID/GID、访问状态与文件属性
- RFC 7530、RFC 8881：NFSv4.0/NFSv4.1；owner、owner_group 和安全模型
- `exports(5)`：`root_squash`、`all_squash`、`anonuid/anongid`、`sec=`
- `rpc.mountd(8)`、`nfs.conf(5)`：`manage-gids` 实现与配置
- `idmapd.conf(5)`、`nfsidmap(8)`、`request-key.conf(5)`：Linux NFSv4 name mapping
- `credentials(7)`、`path_resolution(7)`、`acl(5)`、`getfacl(1)`、`setfacl(1)`：Linux 进程身份、路径和 POSIX ACL
- Linux 内核 NFS 客户端/服务端文档与目标发行版 NFS 管理指南
- 目标发行版 SELinux NFS 文档、`selinux(8)`、`ausearch(8)`、`semanage-fcontext(8)`
- Kubernetes Security Context 与目标 CSI driver 文档：`runAsUser`、`runAsGroup`、`fsGroup` 的实现边界

### 10.2 关联文档

- [NFS-KB-L0-02 SUNRPC、XDR 与 RPC 请求生命周期](../L0-foundation/NFS-KB-L0-02-sunrpc-xdr-request-lifecycle.md)
- [NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)
- [NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)
- [NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期](../L2-deployment/NFS-KB-L2-02-systemd-autofs-mount-lifecycle.md)
- [NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)
- 待建立：`NFS-KB-L3-03` NFSv4 ACL、SELinux 与多协议权限集成

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-02 | 1.1.1 | 将已发布的 L3-02 加入关联文档 | 知识库交叉引用校验 |
| 2026-08-02 | 1.1.0 | 收紧 ACL 示例范围和备份权限，重构双客户端、root squash 与 ACL 实验，补全 Java 短写处理、异常映射、清理和 automount 取证边界 | 基于文档审查结果修订 |
| 2026-08-02 | 1.0.0 | 初始发布 | 建立 AUTH_SYS、数字身份、squash、ACL、SELinux、容器和 Java 权限排障生产知识基线 |
