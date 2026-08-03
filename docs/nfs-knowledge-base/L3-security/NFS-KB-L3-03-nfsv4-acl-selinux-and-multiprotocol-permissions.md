# NFS-KB-L3-03 NFSv4 ACL、SELinux 与多协议权限集成

> 文档状态：待验证  
> 知识阶段：L3 安全与身份  
> 适用范围：Linux NFS 客户端与服务端；RHEL/Rocky/AlmaLinux 8/9、Ubuntu 22.04/24.04 常见环境；NFSv4.0、NFSv4.1、NFSv4.2；NFSv3 仅讨论兼容边界；Linux knfsd、支持 NFSv4 ACL 的 NAS/云文件服务、SELinux、Labeled NFS 及 NFS/SMB 多协议共享；具体行为以目标内核、nfs-utils、nfs4-acl-tools、SELinux policy、Samba/AD、后端文件系统和存储厂商兼容矩阵为准  
> 版本：1.0.2  
> 最后更新：2026-08-03  
> 前置文档：[NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)、[NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)  
> 关联文档：[NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)、[NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. NFSv4 ACL、SELinux 与多协议权限原理](#2-nfsv4-aclselinux-与多协议权限原理)
- [3. 端到端权限判定与变更流程](#3-端到端权限判定与变更流程)
- [4. 配置、命令与参数说明](#4-配置命令与参数说明)
- [5. Java 工程视角](#5-java-工程视角)
- [6. 生产实践与风险边界](#6-生产实践与风险边界)
- [7. 验证实验与观察指标](#7-验证实验与观察指标)
- [8. 排障证据链与检查清单](#8-排障证据链与检查清单)
- [9. 小结](#9-小结)
- [10. 参考资料与关联文档](#10-参考资料与关联文档)

## 1. 学习目标与问题边界

NFSv4 ACL 不是“更长的 chmod”，SELinux 也不是传统权限的替代品。生产中的一次访问可能同时受 RPC 身份、NFSv4 ACE、POSIX mode、服务端 ACL 映射、SELinux type enforcement、导出属性、后端文件系统和 SMB/Windows DACL 约束。

完成本单元后，应能够：

1. 解释 NFSv4 ACE 的 `type:flags:who:permissions` 四段模型。
2. 区分 `ALLOW`、`DENY`、`AUDIT`、`ALARM`，并理解 `aclsupport` 决定实现能力。
3. 解释 `OWNER@`、`GROUP@`、`EVERYONE@`、命名用户和命名组的身份语义。
4. 正确分析 ACE 顺序、未决权限位、`DELETE` 与 `DELETE_CHILD`、目录遍历和继承。
5. 判断 `chmod`、POSIX ACL、NFSv4 ACL 之间是否发生有损转换。
6. 区分服务端 SELinux、客户端挂载标签和 NFSv4.2 Labeled NFS。
7. 设计 NFS/SMB 多协议共享的 ACL 权威源、SID 与 UID/GID 映射和变更治理。
8. 解释 Java NIO 为什么通常不能直接读取 Linux NFSv4 ACE，以及权限变更 API 的风险。
9. 建立覆盖协议 ACL、mode、身份映射、SELinux、SMB DACL 和实际 I/O 的证据链。

本文不重复 AUTH_SYS、Kerberos、POSIX ACL 基础，也不把 Windows ACL、Samba、AD 或 SELinux 通用知识完整展开。重点是这些机制进入 NFS 权限链后如何组合、如何失真以及如何验证。

## 2. NFSv4 ACL、SELinux 与多协议权限原理

### 2.1 权限模型不是互相替代

| 层次 | 回答的问题 | 典型对象 | 不能替代 |
| --- | --- | --- | --- |
| RPC security flavor | 请求者身份是否可信、payload 是否受保护 | `AUTH_SYS`、`krb5/krb5i/krb5p` | 文件授权 |
| NFSv4 ACL | 某主体对某对象允许或拒绝哪些动作 | ordered ACE list | SELinux、export |
| POSIX mode/ACL | Linux VFS 的所有者、组、其他和命名主体权限 | `mode`、`system.posix_acl_*` | NFSv4 全部语义 |
| SELinux/LSM | 主体域是否可访问目标类型 | subject type、object type、class、permission | DAC/NFS ACL |
| export policy | 哪些客户端、flavor、读写模式可进入导出 | `/etc/exports`、NAS export policy | inode 权限 |
| SMB/Windows DACL | SID 对对象的 Windows 权限 | ordered ACE、继承、owner | NFS 身份映射 |

最终访问是多层条件的交集。某一层允许，不代表最终允许；某一层拒绝，客户端通常只看到 `EACCES`、`EPERM` 或协议状态码。

### 2.2 NFSv4 ACL 属性

NFSv4 `acl` 属性是有序的 `nfsace4` 数组。每个 ACE 包含：

```text
type       ALLOW / DENY / AUDIT / ALARM
flag       继承、组标识、审计成功/失败等
accessmask 文件或目录动作位
who        特殊主体或 user@domain / group@domain
```

NFSv4 `aclsupport` 属性用于声明当前文件系统支持哪些 ACE type：

| 能力位 | 含义 | 工程边界 |
| --- | --- | --- |
| `ALLOW` | 允许 ACE | 几乎所有 ACL 实现的基础 |
| `DENY` | 拒绝 ACE | 部分 POSIX 映射后端无法无损表达 |
| `AUDIT` | 成功/失败访问审计 | 不等同于 Linux auditd 或 NAS 审计日志 |
| `ALARM` | 访问告警 | 实现支持较少，不能默认可用 |

协议允许服务器只支持其中一部分。设置前必须查询实际能力；不能因为客户端工具接受字符串就假设服务端能够持久化和执行。

### 2.3 ACE type 与顺序

访问判定针对“本次请求需要的权限位集合”逐位处理：

```text
requested = {READ_DATA, READ_ATTRIBUTES, ...}
undecided = requested

按 ACL 顺序遍历匹配主体的 ACE：
  ALLOW -> 从 undecided 中批准该 ACE 覆盖的位
  DENY  -> 若命中仍未决权限位，则拒绝对应访问

全部需要的权限位均获准 -> 允许
遍历结束仍有未决位      -> 拒绝
```

因此不能使用“DENY 永远优先”或“ALLOW 永远覆盖 DENY”的简化规则。ACE 顺序、主体是否匹配以及当时仍未决的权限位共同决定结果。服务端还可能对 ACL 进行规范化或拒绝无法表达的顺序。

`AUDIT` 和 `ALARM` 用于审计/告警语义，不应被当作授予访问的 ACE。客户端看到审计 ACE，不代表服务端或后端已经产生可检索日志。

### 2.4 特殊主体与命名主体

| `who` | 含义 | 常见误区 |
| --- | --- | --- |
| `OWNER@` | 当前对象 owner | 不是字符串名为 OWNER 的用户 |
| `GROUP@` | 当前对象 owning group | ACE 通常带 group identifier flag |
| `EVERYONE@` | 所有主体，包括 owner 和 owning group | 不是传统 UNIX “other” 的简单同义词 |
| `user@domain` | 命名用户 | 依赖 NFSv4 name/domain 到本地身份映射 |
| `group@domain` | 命名组 | 必须带组标识 flag，且依赖组成员解析 |

NFSv4 ACL 中的名字与 RPC 请求的认证身份是两条相关但不同的链。`sec=sys` 请求携带数字 UID/GID，`sec=krb5*` 请求携带可认证 principal；服务端仍需将它们映射到能够匹配 ACE 的本地身份。

### 2.5 文件与目录权限位

同一个 access mask 位在文件和目录上可能有不同含义：

| 权限概念 | 文件 | 目录 |
| --- | --- | --- |
| `READ_DATA / LIST_DIRECTORY` | 读取文件数据 | 列出目录项 |
| `WRITE_DATA / ADD_FILE` | 写已有文件 | 在目录创建文件 |
| `APPEND_DATA / ADD_SUBDIRECTORY` | 追加文件 | 在目录创建子目录 |
| `EXECUTE` | 执行文件 | 遍历目录 |
| `DELETE` | 删除当前对象 | 删除当前目录对象 |
| `DELETE_CHILD` | 通常无意义 | 删除目录中的子对象 |
| `READ_ATTRIBUTES` | 读取基础属性 | 读取目录属性 |
| `WRITE_ATTRIBUTES` | 修改基础属性/时间等 | 修改目录属性 |
| `READ_ACL` | 读取 ACL | 读取 ACL |
| `WRITE_ACL` | 修改 ACL | 修改 ACL |
| `WRITE_OWNER` | 修改 owner | 修改 owner |
| `SYNCHRONIZE` | Windows 兼容语义，NFS 实现处理存在差异 | 同左 |

删除文件通常取决于“对象的 `DELETE`”或“父目录的 `DELETE_CHILD`”，而不是文件自身的 `WRITE_DATA`。按照 RFC 8881 第 6.2.1.3.2 节，只要其中一条路径明确允许，服务端就应允许删除，即使另一条路径显式拒绝；如果两处均未明确允许或拒绝且父目录没有 sticky bit，服务端应按父目录 `ADD_FILE` 执行兼容 UNIX write 语义的回退判定。父目录设置 `MODE4_SVTX` 后，还可能要求操作者拥有父目录或目标对象，部分实现还会考虑目标是否可写。

重命名至少需要从源父目录移除目录项，并在目标父目录增加普通文件或子目录目录项；覆盖既有目标时还涉及目标删除语义。对应的核心权限通常是源父目录 `DELETE_CHILD`、目标父目录 `ADD_FILE/ADD_SUBDIRECTORY`，以及覆盖目标所需的删除权限。sticky bit、同文件系统限制、目录类型兼容性和后端实现策略仍会继续约束结果。

### 2.6 ACE flags 与继承

| flag | 作用 |
| --- | --- |
| `FILE_INHERIT` | 新建普通文件继承 ACE |
| `DIRECTORY_INHERIT` | 新建子目录继承 ACE |
| `NO_PROPAGATE_INHERIT` | 只向下一层传播，不继续向更深层传播 |
| `INHERIT_ONLY` | ACE 只作为继承模板，不直接作用于当前目录 |
| `IDENTIFIER_GROUP` | `who` 是组而不是用户 |
| `SUCCESSFUL_ACCESS` | AUDIT/ALARM 针对成功访问 |
| `FAILED_ACCESS` | AUDIT/ALARM 针对失败访问 |
| `INHERITED` | `ACE4_INHERITED_ACE` 表示继承结果；仅属于 NFSv4.1+ 的 `dacl/sacl` 自动继承模型，旧 `acl` 属性中必须清除此位，NFSv4.0 不定义此位 |

继承通常发生在对象创建时。修改父目录 ACL 不保证既有子对象自动重算；Windows GUI 中的“替换所有子对象权限”也是递归变更操作，不是协议自动继承。常用 Linux `nfs4_getfacl` 读取旧 `acl` 属性时，不能依靠 `ACE4_INHERITED_ACE` 判断 ACE 来源；必须结合创建前后对照、`dacl/sacl` 能力或厂商原生 ACL 接口验证。

### 2.7 `mode` 与 NFSv4 ACL 的相互影响

NFSv4 同时定义 `mode` 和 `acl` 属性。服务器必须让两者保持不冲突，但允许不同实现采取不同策略：

- 从 ACL 推导低九位 mode；
- `chmod` 时修改 ACL，使其不再授予超出 mode 的权限；
- 丢弃无法保留的命名 ACE、DENY 或继承信息，重建仅表示 mode 的 ACL；
- 拒绝无法映射的 ACL；
- 在支持 `mode_set_masked` 的版本/实现中按协议规则限制修改范围。

`ls -l` 只能显示 mode 摘要，无法完整表达命名 ACE、DENY、继承和 `DELETE_CHILD`。在存在 NFSv4 ACL 的共享上执行 `chmod -R`，可能是破坏性 ACL 重写，不是普通权限修正。

### 2.8 Linux knfsd 与后端文件系统映射

Linux knfsd 常需要在 NFSv4 ACL 与本地文件系统 ACL 模型之间转换。ext4/XFS 常见后端主要原生支持 POSIX ACL，而不是完整 NFSv4 ordered ACL；无法无损表达的 DENY、复杂顺序或继承语义可能被拒绝、规范化或转换。

```text
NFSv4 client acl attribute
        |
        v
Linux nfsd ACL translator
        |
        +--> 可映射 -> POSIX mode/default ACL/xattr
        |
        +--> 不可映射 -> 拒绝、收紧或实现特定转换
```

NAS、ZFS、NTFS security style 或专有文件系统可能原生保存 NFSv4/Windows 风格 ACL。不能把 Linux ext4/XFS 的行为推广到所有存储，也不能把某家 NAS 的 `chmod` 结果推广到 knfsd。

### 2.9 SELinux 是额外的强制访问控制

服务端 nfsd 最终访问后端 inode 时，SELinux 可以在 DAC/NFS ACL 允许后继续拒绝：

```text
RPC identity + export + NFS ACL/DAC allow
                 |
                 v
SELinux subject type -> object type -> class/permission
                 |
          allow or AVC denial
```

客户端通常只收到 `EACCES`，真正原因在服务端 AVC。`chmod 777`、增加 ALLOW ACE 或关闭 root squash 都不能修复 SELinux policy 拒绝。

普通 NFS 客户端挂载往往使用挂载级标签或统一远程文件类型；这与 NFSv4.2 `sec_label` 每对象安全标签不同。

### 2.10 Labeled NFS 与挂载 `context=` 不是一回事

| 模式 | 标签粒度 | 标签来源 | 适用边界 |
| --- | --- | --- | --- |
| 服务端 SELinux | 服务端 inode/后端对象 | 服务端 xattr 和 policy | 保护服务端访问 |
| 客户端挂载 `context=` | 整个挂载统一标签 | 客户端 mount option | 本地单一消费者隔离 |
| NFSv4.2 Labeled NFS | 每对象 `sec_label` | 协议属性 + 双端 MAC policy | 需要客户端、服务端和 LFS 兼容 |

`context=` 不会给每个远端文件写入独立标签，也不能表达多级安全。Labeled NFS 需要 NFSv4.2、`FATTR4_SEC_LABEL`、兼容的 Label Format Specifier、双端策略和受保护的传输；主流 NAS/发行版的支持程度差异很大，必须实测。

### 2.11 NFS/SMB 多协议 ACL

Windows DACL、NFSv4 ACL 和 POSIX ACL 都有“主体、权限、继承”的概念，但不是一一等价：

| 语义 | Windows/SMB | NFSv4 | POSIX ACL |
| --- | --- | --- | --- |
| 主体 | SID | `who` 字符串/特殊主体 | UID/GID |
| 顺序 | ordered ACE | ordered ACE | owner/named/group/mask/other 固定结构 |
| DENY | 原生 | 协议原生，服务端可选支持 | 无通用 ordered DENY |
| 继承 | 丰富 flags | 丰富 flags | default ACL，表达能力较小 |
| 删除控制 | object/parent 权限 | `DELETE/DELETE_CHILD` | 主要由父目录 write+execute |
| 审计 ACE | SACL | AUDIT/ALARM 可选 | 不属于 POSIX ACL |

多协议共享必须先确定 ACL 权威源：

1. NFSv4/Windows ACL 原生权威，POSIX mode 只是投影；
2. POSIX ACL 权威，SMB/NFSv4 只能使用可逆子集；
3. NAS 专有统一安全模型，由厂商完成 SID、UID/GID 和 ACL 转换。

没有明确权威源时，Windows Explorer、`chmod`、`setfacl`、`nfs4_setfacl` 和 NAS 管理界面会成为相互覆盖的多个写入者。

## 3. 端到端权限判定与变更流程

### 3.1 一次 Java 读取的权限链

```text
Java Files.newInputStream()
  -> JVM 进程 fsuid/fsgid/groups
  -> Linux VFS / NFS client
  -> AUTH_SYS 或 RPCSEC_GSS
  -> 服务端 principal/UID/GID 映射
  -> export ro/rw、squash、security flavor
  -> NFSv4 ordered ACL / 服务端 ACL 映射
  -> mode/POSIX ACL
  -> SELinux/LSM
  -> 后端文件系统、快照、只读、immutable、配额
  -> NFS status -> errno -> Java exception
```

排障必须沿调用链逐层收集证据。客户端 `Files.isReadable()`、`ls -l` 或 Windows GUI 任意一个视图都不足以证明最终访问。

### 3.2 ACL 读取链

```text
nfs4_getfacl on NFSv4 mount
  -> client/provider requests the acl attribute
  -> server returns native or synthesized ACL
  -> client tool renders ACE strings

aclsupport capability evidence
  -> a client/probe explicitly requests GETATTR aclsupport
  -> or vendor capability API/documentation reports supported ACE types
  -> packet decode confirms the returned capability bitmap

getfacl on server backend
  -> reads POSIX ACL/xattr
  -> may only show translation target
```

`nfs4_getfacl` 的成功输出不能证明它查询了 `aclsupport`；常用 Linux CLI 通常不直接展示该属性。两份 ACL 输出不同也不一定表示损坏，可能是协议 ACL 与后端存储模型的投影差异。需要把 ACL 回读、服务端或厂商 capability、协议级 `aclsupport` 证据和实际 I/O 分开记录。

### 3.3 ACL 变更链

```text
nfs4_setfacl / Windows ACL editor / NAS API / chmod
  -> ACL authority receives change
  -> canonicalize / map / reject
  -> update mode or backend ACL/xattr
  -> change attribute / cache invalidation
  -> other protocol reads projected ACL
```

变更成功只表示服务端接受请求，不保证再次读取时字节级一致。服务端可能重排 ACE、补充特殊主体、合并权限或删除无法存储的字段。验收应比较“访问语义”和“重新读取结果”，不能只检查命令退出码。

### 3.4 多协议身份链

```text
AD user SID
  -> SSSD/winbind/NAS identity mapper
  -> stable uidNumber/gidNumber or internal numeric ID
  -> NFSv4 who / local UID/GID
  -> SMB DACL and NFS ACL refer to the same identity
```

如果 SID 到 UID/GID 的映射范围、算法或目录属性改变，同一个用户可能在 NFS 和 SMB 视图中成为不同主体。ACL 文本中名字相同也不能证明底层 SID/UID 一致。

### 3.5 创建、删除和重命名

| 操作 | 关键权限位置 | 常见误判 |
| --- | --- | --- |
| 创建文件 | 父目录 `ADD_FILE` + 遍历 | 只看新文件 mode |
| 创建目录 | 父目录 `ADD_SUBDIRECTORY` + 遍历 | 只看父目录 write |
| 读取文件 | 文件 `READ_DATA` + 路径逐级 `EXECUTE` | 文件 ACL 允许就一定能读 |
| 删除文件 | 文件 `DELETE` 或父目录 `DELETE_CHILD`；两者均未决时还可能回退到父目录 `ADD_FILE`；另检查 sticky bit | 文件不可写就不能删除 |
| rename | 源父目录 `DELETE_CHILD`、目标父目录 `ADD_FILE/ADD_SUBDIRECTORY`、覆盖目标删除语义、sticky bit | 只检查源文件 owner |
| 修改 ACL | `WRITE_ACL` + 服务端 policy | owner 必然能任意改 ACL |
| 修改 owner | `WRITE_OWNER` + chown policy | 有 `WRITE_ACL` 就能 chown |

### 3.6 SELinux 拒绝链

```text
NFS client gets EACCES
  -> export and ACL appear correct
  -> server ausearch shows AVC
  -> identify source domain, target type, class, permission
  -> correct labeling or approved policy
  -> repeat exact I/O
```

不要把 `setenforce 0` 作为修复。它最多是受控诊断实验，而且会改变整机安全状态并影响证据。

## 4. 配置、命令与参数说明

### 4.1 客户端与服务端能力基线

**执行端：两端**  
**适用范围：只读取证；工具缺失、属性不支持或 unit 不存在属于环境差异**

```bash
uname -r
nfsstat -m
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS

command -v nfs4_getfacl || true
command -v nfs4_setfacl || true
rpm -q nfs4-acl-tools nfs-utils policycoreutils-python-utils 2>/dev/null || true
dpkg-query -W nfs4-acl-tools nfs-common policycoreutils 2>/dev/null || true

getenforce 2>/dev/null || true
sestatus 2>/dev/null || true
```

记录客户端与服务端 OS、内核、NFS minor version、mount `sec=`、后端文件系统、ACL 工具版本、SELinux mode 和存储厂商 ACL 模式。

### 4.2 NFSv4 ACL 只读取证

**执行端：客户端**  
**适用范围：NFSv4 挂载；输出包含用户、组和访问策略，按敏感信息管理**

```bash
TEST_PATH=/mnt/secure/team-a

stat -c 'mode=%a uid=%u gid=%g type=%F path=%n' "$TEST_PATH"
nfs4_getfacl "$TEST_PATH"
getfacl -p "$TEST_PATH" 2>/dev/null || true
```

预期观察：

- `nfs4_getfacl` 返回 ordered ACE；
- `getfacl` 可能失败、只返回 POSIX 视图或由客户端/服务端转换；
- `stat` mode 只是 ACL 的摘要投影；
- 重新读取的 ACE 顺序可能与写入文本不同。

### 4.3 ACE 文本格式

`nfs4-acl-tools` 常见文本格式：

```text
A::OWNER@:rwatTnNcCy
A:g:GROUP@:rtncy
A::EVERYONE@:rtncy
```

字段：

```text
A : flags : who : permissions
|     |       |         |
|     |       |         +-- 权限字符或工具支持的 R/W/X 别名
|     |       +------------ 特殊主体或 name@domain
|     +-------------------- f/d/n/i/g 等 flag
+-------------------------- A/D/U/L 等 ACE type
```

权限字符和别名以目标 `nfs4_acl(5)`、`nfs4_setfacl(1)` 为准。特别注意：工具的通用 `W` 别名可能扩展为写数据、写属性、写 ACL 等多个权限，不等于“只允许修改文件内容”。

### 4.4 受控 ACL 变更与恢复

以下操作只能针对隔离测试对象。

**执行端：客户端**  
**风险：`nfs4_setfacl -S` 会完整替换 ACL；备份包含身份和授权信息，必须使用 0600 权限并及时清理**

```bash
set -euo pipefail

TEST_PATH=/mnt/secure/.nfs-kb-l3-03-acl-test-dir
TEST_USER=acltest
NFSV4_IDMAP_DOMAIN=example.com  # 必须替换为 nfsidmap -d 或存储配置确认的值
ACL_WHO="${TEST_USER}@${NFSV4_IDMAP_DOMAIN}"
BACKUP=$(mktemp /var/tmp/nfs-kb-l3-03-acl.XXXXXX)
RESTORED=$(mktemp /var/tmp/nfs-kb-l3-03-acl-restored.XXXXXX)
chmod 0600 "$BACKUP"
chmod 0600 "$RESTORED"

restore_required=0
cleanup_acl_test() {
  rc=$?
  trap - EXIT INT TERM

  if (( restore_required )); then
    if nfs4_setfacl -S "$BACKUP" "$TEST_PATH" &&
       nfs4_getfacl "$TEST_PATH" > "$RESTORED"; then
      restore_required=0
    else
      printf 'ACL restore failed; retain backup: %s\n' "$BACKUP" >&2
      rc=1
    fi
  fi

  if (( restore_required == 0 )); then
    rm -f -- "$BACKUP" "$RESTORED"
  fi
  exit "$rc"
}
trap cleanup_acl_test EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

nfs4_getfacl "$TEST_PATH" > "$BACKUP"
test -s "$BACKUP"

# 请求是否部分落盘由服务端决定，因此在发出写请求前标记为需要恢复
restore_required=1
nfs4_setfacl -a "A::${ACL_WHO}:RX" "$TEST_PATH"
nfs4_getfacl "$TEST_PATH"

# 验证结束后恢复完整 ACL
nfs4_setfacl -S "$BACKUP" "$TEST_PATH"
nfs4_getfacl "$TEST_PATH" > "$RESTORED"
restore_required=0
diff -u "$BACKUP" "$RESTORED" || true

rm -f -- "$BACKUP" "$RESTORED"
trap - EXIT INT TERM
```

`RX` 示例用于读取和遍历测试，不授予写 ACL 或写 owner。`NFSV4_IDMAP_DOMAIN` 是 NFSv4 owner/idmapping 使用的 DNS 风格域，不是 Kerberos realm；例如两者可能分别为 `example.com` 和 `EXAMPLE.COM`。必须通过 `nfsidmap -d`、`/etc/idmapd.conf` 或存储端身份配置确认，禁止从 Kerberos realm 直接推导。

该保护模式必须复用于本文其他 ACL 写实验。服务端可能规范化 ACL，因此 `diff` 只用于显示差异；恢复验收还必须检查 mode、实际 I/O 和多协议视图。只有恢复和回读成功后才能删除备份；恢复失败时必须保留 0600 备份并停止后续实验。

恢复后比较：

1. ACE 顺序和主体；
2. mode 是否变化；
3. 目标用户实际读取/遍历；
4. 服务端后端 POSIX ACL 或原生 ACL；
5. SMB 视图是否变化。

### 4.5 继承 ACE

示例含义：

```text
A:fd:acltest@example.com:RX
```

`f` 表示普通文件继承，`d` 表示子目录继承。这里的 `example.com` 是脱敏的 NFSv4 idmap domain，不是 Kerberos realm。命名组需要 `g` flag，例如 `A:fdg:appteam@example.com:RX`。

**执行端：客户端**  
**风险：继承只在隔离测试目录验证；不要对生产树执行递归 ACL 替换**

完整、可执行并包含异常恢复的继承实验见第 7.4 节。该实验在修改父 ACL 前创建基线对象，修改后创建新文件和新目录，分别验证非追溯行为、文件读取、目录遍历、父 ACL 恢复以及固定对象清理。不要把单条 `nfs4_setfacl -a` 从事务脚本中拆出后直接用于生产目录。

### 4.6 `chmod` 影响验证

**执行端：客户端**  
**风险：`chmod` 可能重写命名 ACE、DENY 和继承信息；仅对隔离测试对象执行**

```bash
set -euo pipefail

TEST_PATH=/mnt/secure/.nfs-kb-l3-03-chmod
ACL_BEFORE=$(mktemp /var/tmp/nfs-kb-l3-03-before.XXXXXX)
ACL_AFTER=$(mktemp /var/tmp/nfs-kb-l3-03-after.XXXXXX)
ACL_RESTORED=$(mktemp /var/tmp/nfs-kb-l3-03-restored.XXXXXX)
chmod 0600 "$ACL_BEFORE" "$ACL_AFTER" "$ACL_RESTORED"

restore_required=0
cleanup_chmod_test() {
  rc=$?
  trap - EXIT INT TERM

  if (( restore_required )); then
    if nfs4_setfacl -S "$ACL_BEFORE" "$TEST_PATH" &&
       nfs4_getfacl "$TEST_PATH" > "$ACL_RESTORED"; then
      restore_required=0
    else
      printf 'ACL restore failed; retain backup: %s\n' "$ACL_BEFORE" >&2
      rc=1
    fi
  fi

  if (( restore_required == 0 )); then
    rm -f -- "$ACL_BEFORE" "$ACL_AFTER" "$ACL_RESTORED"
  fi
  exit "$rc"
}
trap cleanup_chmod_test EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

nfs4_getfacl "$TEST_PATH" > "$ACL_BEFORE"
test -s "$ACL_BEFORE"
MODE_BEFORE=$(stat -c '%a' "$TEST_PATH")

restore_required=1
chmod 0750 "$TEST_PATH"
nfs4_getfacl "$TEST_PATH" > "$ACL_AFTER"
diff -u "$ACL_BEFORE" "$ACL_AFTER" || true

nfs4_setfacl -S "$ACL_BEFORE" "$TEST_PATH"
nfs4_getfacl "$TEST_PATH" > "$ACL_RESTORED"
MODE_RESTORED=$(stat -c '%a' "$TEST_PATH")
restore_required=0

diff -u "$ACL_BEFORE" "$ACL_RESTORED" || true
printf 'mode_before=%s mode_restored_from_acl=%s\n' \
  "$MODE_BEFORE" "$MODE_RESTORED"

rm -f -- "$ACL_BEFORE" "$ACL_AFTER" "$ACL_RESTORED"
trap - EXIT INT TERM
```

本实验将 ACL 定义为权威源，因此只恢复 ACL，并把恢复后 mode 作为验收结果；禁止在 ACL 恢复后再次执行 `chmod "$MODE_BEFORE"`。如果目标系统以 mode 为权威，应在单独实验中先执行 `chmod`，接受服务端由 mode 生成的最终 ACL，并重新建立 ACL 基线；不能同时承诺恢复原 ACL 和原 mode。生产迁移必须预先确定唯一权威源，并以 ACL 回读、mode 和实际 I/O 共同验收。

### 4.7 服务端后端 ACL 与 xattr

**执行端：服务端**  
**适用范围：只读取证；xattr 名称和 ACL 内容可能暴露安全模型，不上传完整输出**

```bash
BACKEND_PATH=/srv/nfs/secure/team-a

stat -c 'mode=%a uid=%u gid=%g type=%F path=%n' "$BACKEND_PATH"
getfacl -p "$BACKEND_PATH" 2>/dev/null || true
getfattr --absolute-names -m '^(system|security|trusted)\.' \
  "$BACKEND_PATH" 2>/dev/null || true
findmnt -T "$BACKEND_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
```

只列 xattr 名称，不使用 `getfattr -d` 导出二进制 Windows ACL、SELinux label 或厂商元数据。不同后端可能使用 `system.posix_acl_*`、`security.NTACL`、`system.nfs4_acl` 或专有存储。

### 4.8 SELinux 服务端取证

**执行端：服务端**  
**适用范围：只读取证；AVC 可能包含路径、进程和用户信息**

```bash
BACKEND_PATH=/srv/nfs/secure

getenforce
sestatus
ls -ldZ "$BACKEND_PATH"
matchpathcon "$BACKEND_PATH" 2>/dev/null || true
getsebool -a | grep -E '(^|_)(nfs|use_nfs|export).*-->' || true
sudo ausearch -m AVC,USER_AVC -ts recent | tail -n 200
sudo journalctl -t setroubleshoot --since "30 min ago" --no-pager \
  2>/dev/null || true
```

预期观察：实际 label 是否与 policy 默认匹配、是否存在 nfsd 或后端服务相关 AVC，以及拒绝的 class/permission。

### 4.9 SELinux 持久标签示例

以下示例只用于独立实验目录。`public_content_rw_t` 可能允许多个服务域访问，不是所有生产 NFS 导出的默认最小权限类型。

**执行端：服务端**  
**风险：改变 SELinux 持久文件上下文；必须先确认发行版 policy、共享服务范围和回滚结果**

```bash
LAB_PATH=/srv/nfs/.nfs-kb-l3-03-selinux
LAB_RULE='/srv/nfs/\.nfs-kb-l3-03-selinux(/.*)?'

sudo semanage fcontext -a -t public_content_rw_t "$LAB_RULE"
sudo restorecon -RFv "$LAB_PATH"
ls -ldZ "$LAB_PATH"

# 实验结束后删除本次规则，并恢复父目录默认标签
sudo semanage fcontext -d "$LAB_RULE"
sudo restorecon -RFv "$LAB_PATH"
```

若目标 policy 没有该类型或需要专用类型，应停止并由 SELinux policy 管理者设计模块。不要使用 `chcon -R` 作为生产持久配置。

### 4.10 SELinux boolean 风险

发行版可能提供 `nfs_export_all_rw`、`httpd_use_nfs`、`virt_use_nfs`、`use_nfs_home_dirs` 等 boolean。

**执行端：服务端或对应客户端**  
**适用范围：先只读取证**

```bash
getsebool nfs_export_all_ro 2>/dev/null || true
getsebool nfs_export_all_rw 2>/dev/null || true
getsebool httpd_use_nfs 2>/dev/null || true
getsebool virt_use_nfs 2>/dev/null || true
getsebool use_nfs_home_dirs 2>/dev/null || true
```

`setsebool -P ... on` 会持久改变整个策略域，不是单目录授权。只有在 policy 文档、影响范围、审批、监控和回滚都明确时才能启用；更窄的文件类型或自定义 policy 通常更可控。

### 4.11 客户端挂载级 `context=`

隔离应用节点可以评估统一挂载标签：

```fstab
nfs.example.com:/secure  /mnt/secure  nfs4  nfsvers=4.1,hard,sec=krb5p,context=system_u:object_r:httpd_sys_content_t:s0,_netdev  0  0
```

**适用版本与范围：** SELinux 已启用且 `mount.nfs` 支持安全上下文挂载选项的 Linux 客户端；单一 SELinux 消费域、所有远端对象可共享同一客户端标签。  
**预期效果：** 客户端本地 VFS 把整个挂载视为指定 context。  
**风险：** 不能表达每文件标签；同一挂载被多个应用使用时可能过度授权；标签错误可导致整个挂载不可访问；同一 NFS export 的多个挂载可能共享 superblock，后续挂载不能假定可以覆盖首个挂载的安全选项。  
**回滚：** 停止使用该挂载的应用，恢复上一版 fstab，完整卸载并重新挂载目标 mount unit，然后验证实际 mount options 与 `ls -Z`。SELinux `context=` 不能通过普通 remount 改成另一个 context；不能把 `mount -o remount` 当作可靠回滚。

变更前先使用 `findmnt -S nfs.example.com:/secure -o TARGET,SOURCE,OPTIONS` 检查相同 export 的所有现有挂载。不要为了获得不同标签而无评审地添加 `nosharecache`；它会改变 NFS 数据与属性缓存共享方式。MLS/MCS context 如果包含逗号，还必须按目标 `mount(8)`/`fstab(5)` 规则正确引用，避免被拆成多个 mount option。

不要把 `context=` 误写成 Labeled NFS，也不要与存储端 per-file label 结论混用。

### 4.12 Labeled NFS 能力边界

**执行端：两端**  
**适用范围：只读取证；工具和内核字段依发行版实现**

```bash
uname -r
nfsstat -m
findmnt -t nfs,nfs4 -o TARGET,SOURCE,FSTYPE,OPTIONS

# 客户端应检查 CONFIG_NFS_V4_SECURITY_LABEL，服务端检查 CONFIG_NFSD_V4_SECURITY_LABEL
grep -E '^CONFIG_(NFS|NFSD)_V4_SECURITY_LABEL=' \
  "/boot/config-$(uname -r)" 2>/dev/null || true
zgrep -E '^CONFIG_(NFS|NFSD)_V4_SECURITY_LABEL=' \
  /proc/config.gz 2>/dev/null || true

grep -w security_label /proc/mounts 2>/dev/null || true
```

这些命令只能证明静态前置条件，不能单独证明 `FATTR4_SEC_LABEL` 工作。真实验证必须使用专用 NFSv4.2 测试对象和经 SELinux 管理员批准的完整测试 context：

```bash
set -euo pipefail

TEST_DIR=/mnt/secure/.nfs-kb-l3-03-labeled
TEST_FILE="$TEST_DIR/label-probe"
: "${APPROVED_CONTEXT:?set an SELinux-admin-approved full context}"

test ! -e "$TEST_FILE" && test ! -L "$TEST_FILE"
cleanup_label_probe() {
  rm -f -- "$TEST_FILE"
}
trap cleanup_label_probe EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

touch "$TEST_FILE"
stat -c 'before=%C path=%n' "$TEST_FILE"
chcon "$APPROVED_CONTEXT" "$TEST_FILE"
stat -c 'after=%C path=%n' "$TEST_FILE"

# 在另一台兼容客户端再次读取 label 后，删除本文创建的固定测试对象
rm -f -- "$TEST_FILE"
trap - EXIT INT TERM
```

`chcon` 成功本身仍不够；必须从另一台客户端回读，并确认服务端后端、HA 节点和备份链保留相同 label。若任一步失败，不得通过关闭 SELinux 或改用挂载 `context=` 冒充 per-file label 支持。最终证据必须覆盖：

1. 服务端声明并返回 `sec_label`；
2. 双端理解相同 LFS/policy；
3. 每对象标签可读取、创建、变更和缓存失效；
4. 标签传输使用满足安全要求的 RPC flavor；
5. 不识别 label 或 LFS 时按协议拒绝，而不是静默降级。

在可解码且经过批准的隔离环境，可用 `tcpdump -s 0 -w <capture.pcap> host <server> and tcp port 2049` 保存协议证据，并在 Wireshark 中核对 NFSv4.2 `GETATTR/SETATTR` 的 `FATTR4_SEC_LABEL`。`sec=krb5p` 会保护 RPC payload，抓包通常无法直接解码属性；不得为了抓包而降低生产挂载的 security flavor，应改用双端标签回读、受控服务端 tracing 或厂商诊断接口。

### 4.13 SMB 侧只读检查

**执行端：SMB 管理客户端**  
**适用范围：已有受控 Kerberos ticket；Samba 工具选项随版本变化，先检查本机帮助**

```bash
smbcacls --help | sed -n '1,180p'
smbcacls //smb.example.com/share team-a/test \
  --use-kerberos=required
```

旧版 Samba 可能使用 `-k` 等不同参数，必须以本机帮助为准。不要把密码放入命令行。输出中的 SID、域、继承和权限属于敏感授权数据。

### 4.14 多协议配置证据

**执行端：Samba/NAS 管理端**  
**适用范围：只读取证；隐藏密码、密钥、真实共享名和完整 idmap 范围**

```bash
testparm -s 2>/dev/null | grep -Ei \
  'security|realm|idmap|vfs objects|acl_xattr|inherit|map acl|dos filemode'
wbinfo --online-status 2>/dev/null || true
getent passwd acltest
id acltest
```

需要记录但不在公开文档展示：

- SID 到 UID/GID 映射模式和稳定性；
- Samba `vfs_acl_xattr`、NAS security style 或厂商统一 ACL 模式；
- 谁可以通过 SMB 修改 owner、DACL 和继承；
- NFS `chmod/chown/nfs4_setfacl` 是否被允许；
- ACL 保存位置、HA 复制和备份恢复方式。

## 5. Java 工程视角

### 5.1 Java API 能看到什么

| Java API | Linux NFS 常见含义 | 风险 |
| --- | --- | --- |
| `PosixFileAttributeView` | mode、owner、group | 看不到完整 NFSv4 ACE |
| `AclFileAttributeView` | provider 可选 ACL 视图 | Linux NFS provider 常不支持；非空也需验证语义 |
| `Files.isReadable/isWritable` | provider 的可访问性提示 | 有 TOCTOU，不是最终授权证明 |
| `Files.setPosixFilePermissions` | 类似 chmod | 可能重写或收紧 NFSv4 ACL |
| `Files.setOwner` | chown | 依赖 `WRITE_OWNER`、服务端策略和身份映射 |
| `Files.delete/move` | 删除/rename | 主要受父目录和目标目录 ACL 影响 |

JDK 没有通用、可移植的 NFSv4 ACE API。`AclFileAttributeView` 的类型看起来接近 Windows ACL，但是否映射到服务端 NFSv4 ACL由文件系统 provider 决定。

### 5.2 Java 只读能力探针

```java
import java.io.IOException;
import java.nio.file.FileStore;
import java.nio.file.Files;
import java.nio.file.LinkOption;
import java.nio.file.Path;
import java.nio.file.attribute.AclFileAttributeView;
import java.nio.file.attribute.PosixFileAttributeView;
import java.nio.file.attribute.PosixFileAttributes;

public final class NfsAclCapabilityProbe {
    private NfsAclCapabilityProbe() {
    }

    public static void main(String[] args) throws IOException {
        if (args.length != 1) {
            throw new IllegalArgumentException(
                    "usage: NfsAclCapabilityProbe <path>");
        }

        Path path = Path.of(args[0]);
        FileStore store = Files.getFileStore(path);

        boolean posixSupported =
                store.supportsFileAttributeView("posix");
        boolean aclSupported =
                store.supportsFileAttributeView("acl");

        System.out.printf(
                "path=%s store=%s type=%s posix=%s acl=%s%n",
                path, store.name(), store.type(),
                posixSupported, aclSupported);

        PosixFileAttributeView posixView =
                Files.getFileAttributeView(
                        path,
                        PosixFileAttributeView.class,
                        LinkOption.NOFOLLOW_LINKS);
        if (posixView != null) {
            PosixFileAttributes attrs = posixView.readAttributes();
            System.out.printf(
                    "owner=%s group=%s permissions=%s%n",
                    attrs.owner().getName(),
                    attrs.group().getName(),
                    attrs.permissions());
        }

        AclFileAttributeView aclView =
                Files.getFileAttributeView(
                        path,
                        AclFileAttributeView.class,
                        LinkOption.NOFOLLOW_LINKS);
        if (aclView == null) {
            System.out.println(
                    "acl-view=unsupported; use nfs4_getfacl/vendor API");
        } else {
            System.out.printf(
                    "acl-view-owner=%s ace-count=%d%n",
                    aclView.getOwner().getName(),
                    aclView.getAcl().size());
        }
    }
}
```

`acl-view=unsupported` 不表示服务端没有 NFSv4 ACL，只表示当前 Java provider 不暴露该视图。探针输出中的 owner/group 名称仍受客户端 NSS/idmapping 影响。

### 5.3 不要用 `isWritable` 做授权决策

`Files.isWritable(path)` 之后，ACL、ticket、SELinux、父目录或服务端状态都可能变化：

```text
T0 Files.isWritable(path) -> true
T1 管理员修改 ACL / ticket 到期 / HA 切换
T2 Files.newOutputStream(path) -> AccessDeniedException
```

业务代码应直接执行目标操作，捕获异常并记录阶段、路径类型、运行 UID、mount、时间和关联请求。预检查只能改善提示，不能替代实际系统调用。

### 5.4 Java 权限变更风险

以下代码在 NFSv4 ACL 共享上可能造成有损转换：

```java
Files.setPosixFilePermissions(path, permissions);
Files.setOwner(path, owner);
```

风险：

- `setPosixFilePermissions` 触发服务端 mode-to-ACL 同步；
- 命名 ACE、DENY、继承和 SMB DACL 可能被删除或重排；
- `setOwner` 改变 `OWNER@` 语义；
- 在多协议共享中可能覆盖 Windows 管理员设置；
- 应用回滚只保存 mode，无法恢复原 ACL。

生产 Java 服务默认不应拥有 `WRITE_ACL` 或 `WRITE_OWNER`。权限变更应由独立治理工具完成，并保存 ACL 语义级备份。

### 5.5 Java 删除与 rename

`AccessDeniedException` 出现在 delete/move 时，应优先检查父目录：

```text
Files.delete(file)
  -> file DELETE
     or parent DELETE_CHILD
  -> parent EXECUTE/traverse
  -> sticky/mode/ACL
  -> SELinux unlink permission

Files.move(source, target)
  -> source parent remove
  -> target parent add/replace
  -> target existing object delete
```

不要因为 Java 能写文件内容就推断它能 rename 或删除。

### 5.6 缓存与一致性

客户端可能缓存属性、目录项和 ACCESS 结果。ACL 变更后的短时间内，`ls`、`Files.isReadable` 和实际 I/O 的观察顺序可能不同。验证时应记录 mount `actimeo/ac*` 参数、变更时间、客户端、服务端 change attribute 和真实操作结果。

不建议为排障直接把生产挂载改成 `noac`；它会显著改变一致性和性能行为。应在隔离测试挂载验证缓存假设。

## 6. 生产实践与风险边界

### 6.1 先确定 ACL 权威源

| 场景 | 推荐权威源 | 禁止的并行写入者 |
| --- | --- | --- |
| Linux 单协议 knfsd + ext4/XFS | POSIX mode/ACL，可映射的 NFSv4 子集 | Windows DACL 管理工具 |
| 原生 NFSv4 ACL NAS | NAS/NFSv4 ACL | 无治理的 chmod/setfacl |
| SMB 主导的部门共享 | Windows DACL/NAS security style | Java chmod、Linux 递归 setfacl |
| 真正双协议共享 | 厂商统一 ACL 模型 | 任何绕过统一模型的本地后端修改 |

权威源必须写入架构文档、运维权限、自动化工具和审计流程。不能只依赖团队口头约定。

### 6.2 使用可逆 ACL 子集

如果后端只能保存 POSIX ACL，应限制 NFSv4 ACL 使用：

- 以 ALLOW 为主；
- 避免依赖 ACE 顺序的复杂 DENY；
- 谨慎使用继承组合；
- 不依赖 AUDIT/ALARM；
- 通过 owner/group/named user/named group/mask 可表达；
- 每种 ACL 模板都做写入、回读和实际 I/O 验证。

需要完整 Windows/NFSv4 ACL 时，应选择原生支持的存储，不用脚本模拟复杂 DENY。

### 6.3 禁止无备份的 `chmod -R`

`chmod -R`、`setfacl -Rb`、Windows“替换所有子对象权限”和 NAS 递归 ACL 应视为批量数据变更：

- 可能遍历数百万对象；
- 不具备原子性；
- 中途失败形成混合权限状态；
- 改变 ctime/change attribute 和缓存；
- 可能删除命名 ACE、继承和 SMB DACL；
- 回滚需要逐对象 ACL 备份，而不是一个目录 mode。

生产执行前必须有快照、对象清单、速率限制、断点、进度指标和回滚演练。

### 6.4 ACL 备份也是敏感数据

ACL 备份暴露：

- 用户、组、SID、realm 和组织结构；
- 管理员、服务账号和特权路径；
- 数据分类与访问边界；
- 继承和审计策略。

备份应加密、最小权限、设置保留期，不进入 Git、普通日志或公开工单。恢复时要验证身份映射仍然稳定。

### 6.5 身份映射必须长期稳定

多协议环境要冻结：

- AD SID 与 uidNumber/gidNumber；
- winbind/SSSD/NAS idmap 方法；
- 自动分配范围和冲突策略；
- domain、realm、NFSv4 idmap domain；
- 删除用户、重建账号和 SIDHistory 处理；
- 跨域组与嵌套组展开规则。

改变 idmap 算法可能让所有既有 ACL 指向错误主体。迁移不能只对比用户名文本。

### 6.6 SELinux 最小授权

优先顺序：

1. 修正错误文件标签；
2. 使用发行版已有的窄范围类型和接口；
3. 为特定应用编写受控 policy 模块；
4. 最后才评估全局 boolean。

禁止动作：

| 动作 | 风险 |
| --- | --- |
| `setenforce 0` 作为长期修复 | 关闭整机 MAC |
| `setsebool -P nfs_export_all_rw on` 无评审启用 | 扩大全局可写导出能力 |
| `chcon -R` 作为持久配置 | restorecon/relabel 后丢失 |
| 直接使用 `audit2allow -M` 全量生成策略 | 把攻击或误配置行为固化为允许 |
| 给共享树统一宽松类型 | 跨应用越权 |

### 6.7 Labeled NFS 适用边界

只有在以下条件同时满足时才进入生产评估：

- 强制要求每对象 MAC label；
- 客户端和服务端均支持目标 NFSv4.2/LFS；
- 标签语义在所有节点一致；
- pNFS、HA、快照、备份和灾备保留 label；
- RPC 认证、完整性和隐私符合安全要求；
- 标签缓存、delegation 和失效行为完成演练；
- NAS/云服务明确声明兼容。

普通业务共享不应仅为了“更安全”盲目启用 Labeled NFS。

### 6.8 多协议变更控制

ACL 变更必须记录：

```text
request id
  -> identity/SID/UID
  -> source protocol and tool
  -> object scope
  -> before ACL + mode + label
  -> requested semantic change
  -> server canonicalized ACL
  -> NFS/SMB actual I/O matrix
  -> rollback evidence
```

同一维护窗口内不要同时运行 Windows ACL 迁移、Linux chmod、NAS policy 转换和 idmap 变更。

### 6.9 高可用与灾备

HA/DR 必须复制的不只是文件数据：

- NFSv4/native ACL；
- POSIX ACL xattr；
- Windows security descriptor/SID；
- SELinux label；
- owner/group 数字身份；
- idmap 配置和目录依赖；
- NAS security style；
- ACL 审计记录。

只复制文件内容和 mode 会造成灾备切换后静默越权或大面积拒绝。

### 6.10 容器与 Kubernetes

`runAsUser`、`runAsGroup`、`fsGroup` 只能影响数字身份和部分卷权限准备，不会自动创建 NFSv4 ACE，也不会解决 Kerberos principal、SELinux type 或 SMB SID 映射。

风险：

- kubelet 递归 `chown/chmod` 可能破坏 ACL；
- `fsGroupChangePolicy` 只适用于支持 fsGroup ownership 管理的卷；CSI 驱动声明 `VOLUME_MOUNT_GROUP` 后由驱动处理，kubelet 不再递归修改，且该字段不再控制驱动行为；
- NFS 导出启用 root squash 时，kubelet 或 CSI 的递归 `chown` 可能因服务端身份映射而失败；不能为绕过失败临时启用 `no_root_squash`；
- 宿主机 SELinux mount label 与容器 MCS label 可能拒绝；
- 多 Pod 共享同一 NFS 路径时，统一 `context=` 可能跨租户；
- CSI driver 是否支持 mountOptions、Kerberos 和 Labeled NFS 要单独验证。

## 7. 验证实验与观察指标

所有实验只在隔离测试共享执行。测试用户、组、SID、路径和 ACL 备份均使用脱敏对象；生产 ACL 不进入知识库。

### 7.1 实验一：ACL 能力和三视图基线

**执行端：客户端与服务端**  
**适用范围：只读取证**

客户端：

```bash
TEST_PATH=/mnt/secure/.nfs-kb-l3-03-baseline

nfsstat -m
stat -c 'mode=%a uid=%u gid=%g path=%n' "$TEST_PATH"
nfs4_getfacl "$TEST_PATH"
getfacl -p "$TEST_PATH" 2>/dev/null || true
```

服务端：

```bash
BACKEND_PATH=/srv/nfs/secure/.nfs-kb-l3-03-baseline

stat -c 'mode=%a uid=%u gid=%g path=%n' "$BACKEND_PATH"
getfacl -p "$BACKEND_PATH" 2>/dev/null || true
getfattr --absolute-names -m '^(system|security|trusted)\.' \
  "$BACKEND_PATH" 2>/dev/null || true
```

记录客户端 NFSv4 ACL、客户端 mode、服务端 mode/POSIX ACL 和后端类型，建立转换基线。该实验只能证明 ACL 是否可读取及三个视图如何映射，不能证明 `aclsupport` 的 ACE type 位图；后者必须按第 3.2 节另行取得协议或厂商 capability 证据。

### 7.2 实验二：命名用户 ALLOW

**执行端：客户端**  
**风险：只修改专用测试对象；ACL 备份使用 0600**

```bash
set -euo pipefail

TEST_PATH=/mnt/secure/.nfs-kb-l3-03-allow
TEST_USER=acltest
NFSV4_IDMAP_DOMAIN=example.com  # 替换为已确认的 NFSv4 idmap domain
ACL_WHO="${TEST_USER}@${NFSV4_IDMAP_DOMAIN}"
BACKUP=$(mktemp /var/tmp/nfs-kb-l3-03-allow.XXXXXX)
chmod 0600 "$BACKUP"

restore_required=0
restore_acl() {
  if (( restore_required )); then
    nfs4_setfacl -S "$BACKUP" "$TEST_PATH"
    nfs4_getfacl "$TEST_PATH"
    restore_required=0
  fi
}
trap restore_acl EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

nfs4_getfacl "$TEST_PATH" > "$BACKUP"
test -s "$BACKUP"
restore_required=1
nfs4_setfacl -a "A::${ACL_WHO}:R" "$TEST_PATH"
nfs4_getfacl "$TEST_PATH"

sudo -u "$TEST_USER" dd if="$TEST_PATH" of=/dev/null \
  bs=4k count=1 status=none

restore_acl
trap - EXIT INT TERM
rm -f -- "$BACKUP"
```

预期：目标用户能读取；恢复并回读后权限回到基线。恢复失败时脚本会以非零状态退出并保留 `$BACKUP`，必须停止实验并人工恢复。若挂载使用 `sec=krb5*`，`sudo -u` 可能无法进入业务用户原有 KCM/KEYRING/FILE cache，应在目标用户的真实受控会话中执行读取。若失败，依次检查 identity mapping、路径遍历、ACE 顺序、mode 映射、SELinux 和 export。

### 7.3 实验三：`chmod` 有损转换

使用第 4.6 节命令，对比：

1. 命名 ACE 是否保留；
2. DENY 是否被删除或重排；
3. 继承 flags 是否保留；
4. mode 是否与预期一致；
5. SMB DACL 是否变化；
6. 恢复 ACL 后 mode 是否再次变化。

每种目标存储和 security style 都要独立记录，不共享结论。

### 7.4 实验四：继承与既有对象

**执行端：客户端**  
**风险：只操作固定测试目录，不递归修改生产树**

```bash
set -euo pipefail

PARENT=/mnt/secure/.nfs-kb-l3-03-inherit
TEST_USER=acltest
NFSV4_IDMAP_DOMAIN=example.com  # 替换为已确认的 NFSv4 idmap domain
ACL_WHO="${TEST_USER}@${NFSV4_IDMAP_DOMAIN}"
BEFORE_FILE="$PARENT/file-before-acl"
AFTER_FILE="$PARENT/file-after-acl"
AFTER_DIR="$PARENT/dir-after-acl"
BACKUP=$(mktemp /var/tmp/nfs-kb-l3-03-inherit.XXXXXX)
BEFORE_ACL=$(mktemp /var/tmp/nfs-kb-l3-03-before-object.XXXXXX)
BEFORE_ACL_AFTER=$(mktemp /var/tmp/nfs-kb-l3-03-before-object-after.XXXXXX)
chmod 0600 "$BACKUP" "$BEFORE_ACL" "$BEFORE_ACL_AFTER"

for path in "$BEFORE_FILE" "$AFTER_FILE" "$AFTER_DIR"; do
  test ! -e "$path" && test ! -L "$path"
done

restore_required=0
before_created=0
after_file_created=0
after_dir_created=0
cleanup_inheritance_test() {
  rc=$?
  trap - EXIT INT TERM

  if (( restore_required )); then
    if nfs4_setfacl -S "$BACKUP" "$PARENT" &&
       nfs4_getfacl "$PARENT" >/dev/null; then
      restore_required=0
    else
      printf 'parent ACL restore failed; retain backup: %s\n' \
        "$BACKUP" >&2
      rc=1
    fi
  fi

  if (( restore_required == 0 )); then
    if (( after_file_created )); then
      rm -f -- "$AFTER_FILE" || rc=1
    fi
    if (( after_dir_created )); then
      rmdir -- "$AFTER_DIR" || rc=1
    fi
    if (( before_created )); then
      rm -f -- "$BEFORE_FILE" || rc=1
    fi
    rm -f -- "$BACKUP" "$BEFORE_ACL" "$BEFORE_ACL_AFTER" || rc=1
  fi
  exit "$rc"
}
trap cleanup_inheritance_test EXIT
trap 'exit 130' INT
trap 'exit 143' TERM

touch "$BEFORE_FILE"
before_created=1
nfs4_getfacl "$BEFORE_FILE" > "$BEFORE_ACL"

nfs4_getfacl "$PARENT" > "$BACKUP"
test -s "$BACKUP"
restore_required=1
nfs4_setfacl -a "A:fd:${ACL_WHO}:RX" "$PARENT"

# 既有对象不应仅因父 ACL 改变而自动获得继承 ACE
nfs4_getfacl "$BEFORE_FILE" > "$BEFORE_ACL_AFTER"
diff -u "$BEFORE_ACL" "$BEFORE_ACL_AFTER" || true

touch "$AFTER_FILE"
after_file_created=1
mkdir "$AFTER_DIR"
after_dir_created=1

nfs4_getfacl "$PARENT"
nfs4_getfacl "$AFTER_FILE"
nfs4_getfacl "$AFTER_DIR"

# stat/ls -ld 不验证 READ_DATA 或目录遍历，必须执行真实读取与目录访问
sudo -u "$TEST_USER" dd if="$AFTER_FILE" of=/dev/null \
  bs=4k count=1 status=none
sudo -u "$TEST_USER" sh -c \
  'cd "$1" && ls -A . >/dev/null' sh "$AFTER_DIR"

nfs4_setfacl -S "$BACKUP" "$PARENT"
nfs4_getfacl "$PARENT" >/dev/null
restore_required=0

# 父 ACL 恢复不会追溯删除新对象已经继承的 ACE
nfs4_getfacl "$AFTER_FILE"
nfs4_getfacl "$AFTER_DIR"

rm -f -- "$AFTER_FILE"
after_file_created=0
rmdir -- "$AFTER_DIR"
after_dir_created=0
rm -f -- "$BEFORE_FILE"
before_created=0
rm -f -- "$BACKUP" "$BEFORE_ACL" "$BEFORE_ACL_AFTER"
trap - EXIT INT TERM
```

预期结果：变更前对象 ACL 不自动改变；变更后文件和目录获得可识别的继承结果；目标用户能够真实读取文件并进入、列举目录；父 ACL 恢复后，新对象的继承 ACE 仍然存在，直到固定测试对象被删除。若使用 `sec=krb5*`，必须在具备目标用户有效 credential cache 的受控会话执行 I/O，不能仅依赖 `sudo -u`。

### 7.5 实验五：删除与 rename 权限

**执行端：客户端**  
**适用范围：专用测试目录和测试用户**

```bash
TEST_USER=acltest
SRC_DIR=/mnt/secure/.nfs-kb-l3-03-src
DST_DIR=/mnt/secure/.nfs-kb-l3-03-dst

delete_rc=0
sudo -u "$TEST_USER" rm "$SRC_DIR/delete-me" || delete_rc=$?

move_rc=0
sudo -u "$TEST_USER" mv "$SRC_DIR/move-me" "$DST_DIR/moved" \
  || move_rc=$?

printf 'delete_rc=%s move_rc=%s\n' "$delete_rc" "$move_rc"
```

测试管理员必须为每个用例重新创建 `delete-me` 和 `move-me`，并确认目标 `moved` 不存在。每次 ACL 变更都复用第 4.4 节的备份、异常恢复和回读模式；记录源/目标目录 ACL、文件 ACL、mode、sticky bit、返回码和服务端日志。不要只检查文件自身 write 权限。

至少覆盖以下独立矩阵，不能用一次成功或失败概括删除语义：

| 用例 | 目标对象 | 父目录 | 其他条件 | 预期关注点 |
| --- | --- | --- | --- | --- |
| A | `DELETE` ALLOW | `DELETE_CHILD` DENY | 无 sticky bit | 对象 ALLOW 是否仍允许删除 |
| B | `DELETE` DENY | `DELETE_CHILD` ALLOW | 无 sticky bit | 父目录 ALLOW 是否仍允许删除 |
| C | `DELETE` 未决 | `DELETE_CHILD` 未决，`ADD_FILE` ALLOW | 无 sticky bit | 是否执行 UNIX write 兼容回退 |
| D | 同 C | 同 C | 设置 `MODE4_SVTX` | owner 与实现策略如何改变结果 |
| E | 源目录允许移除 | 目标目录分别允许/拒绝 `ADD_FILE` 或 `ADD_SUBDIRECTORY` | 目标不存在 | rename 的源、目标目录判定 |
| F | 同 E | 目标目录允许增加 | 目标已存在 | 覆盖目标的删除权限和类型兼容性 |

不同后端可能拒绝无法表示的 DENY。若 ACL 写入后回读已经被转换，就必须记录“该后端无法构造此测试用例”，不能继续把结果解释为 RFC 原生 ACL 行为。

### 7.6 实验六：SELinux enforcing 对照

**执行端：服务端**  
**风险：不切换 permissive；通过正确标签和受控 policy 验证**

```bash
getenforce
ls -ldZ /srv/nfs/.nfs-kb-l3-03-selinux
sudo ausearch -m AVC,USER_AVC -ts recent | tail -n 200
```

实验步骤：

1. 保持 Enforcing；
2. 执行目标 NFS I/O 并记录时间；
3. 收集 AVC；
4. 只修正实验路径持久 label 或批准的 policy；
5. 重复完全相同的 I/O；
6. 确认没有新增 AVC。

### 7.7 实验七：NFS/SMB 交叉矩阵

| 操作 | NFS 客户端 | SMB 客户端 | 需要比较 |
| --- | --- | --- | --- |
| 创建文件 | 是 | 是 | owner、SID、UID/GID、继承 ACL |
| 修改内容 | 是 | 是 | 数据权限与锁 |
| chmod/DACL | 受控 | 受控 | 对方协议 ACL 是否变化 |
| rename | 是 | 是 | 父目录权限 |
| 删除 | 是 | 是 | `DELETE/DELETE_CHILD` 映射 |
| 修改 owner | 受控 | 受控 | SID/UID 与 `OWNER@` |

每次只允许一个协议修改 ACL，另一协议只读取和执行 I/O。否则无法确定哪个写入者造成转换。

### 7.8 实验八：Java capability 与真实操作

**执行端：客户端，以 JVM 运行 UID 执行**

```bash
javac NfsAclCapabilityProbe.java
java NfsAclCapabilityProbe /mnt/secure/.nfs-kb-l3-03-baseline
```

记录：

- provider 是否暴露 posix/acl view；
- Java owner/group 与命令行显示是否一致；
- `AclFileAttributeView` 不支持时是否错误地触发应用 fallback chmod；
- 真实 create/read/write/delete/rename 的异常类型和时间。

### 7.9 实验清理

**执行端：客户端与服务端**  
**风险：先恢复 ACL，再按固定对象清理；禁止递归通配删除**

清理前确认：

- 所有 `nfs4_setfacl -S` 恢复成功；
- 测试对象 ACL 已回读；
- SELinux fcontext 实验规则已删除；
- 没有残留挂载级 `context=`；
- SMB DACL 已恢复；
- ACL 备份已按安全流程销毁；
- 只删除本文创建的固定测试对象。

先只列出可能残留的临时备份，不使用通配符直接删除：

```bash
find /var/tmp -maxdepth 1 -user "$(id -u)" \
  -name 'nfs-kb-l3-03-*' -type f -ls
```

逐一核对每个文件对应的实验、目标路径及 ACL 是否已经恢复；恢复失败所保留的备份不得删除。删除与 rename 实验只清理已确认由本文创建的固定对象：

```bash
SRC_DIR=/mnt/secure/.nfs-kb-l3-03-src
DST_DIR=/mnt/secure/.nfs-kb-l3-03-dst

rm -f -- "$SRC_DIR/delete-me" "$SRC_DIR/move-me" "$DST_DIR/moved"
```

其他实验脚本已经在成功恢复后通过保存的精确变量删除自身临时文件和新建对象。不得执行 `rm -f /var/tmp/nfs-kb-l3-03-*`，也不得使用递归通配删除测试树。普通文件删除不能保证覆盖 SSD、快照、日志文件系统或备份中的旧数据；ACL 备份的彻底销毁应遵循组织的数据介质、快照和备份保留策略。

客户端 root 受 root squash 影响时，由对象 owner 或服务端管理员按后端固定路径清理，不能临时启用 `no_root_squash`。

### 7.10 必须记录的指标

| 维度 | 证据 | 目的 |
| --- | --- | --- |
| NFS | version、`sec=`、mount options、status | 确认协议路径 |
| ACL | 写入前、写入后、回读、ACE 顺序 | 识别规范化和丢失 |
| mode | 每次变更前后 mode | 观察 ACL 投影 |
| identity | principal、UID/GID、SID、groups | 证明主体一致 |
| backend | filesystem、security style、xattr 名称 | 确认存储模型 |
| SELinux | mode、context、AVC、policy | 定位 MAC 拒绝 |
| SMB | DACL、owner、inheritance | 多协议对照 |
| Java | provider view、operation、exception、PID/UID | 应用影响 |
| cache | change time、客户端、重试时间 | 区分缓存 |
| HA/DR | 节点、ACL/label 复制结果 | 防止切换差异 |

## 8. 排障证据链与检查清单

### 8.1 先回答十二个问题

1. 存储的 ACL 权威源是什么？
2. 客户端实际使用 NFSv4 哪个 minor version 和 `sec=`？
3. 服务端 `aclsupport`/厂商 ACL capability 支持哪些 ACE type？
4. 后端是 POSIX ACL、原生 NFSv4 ACL、Windows DACL 还是专有模型？
5. ACL 是否在最近被 `chmod`、`setfacl`、Windows GUI 或 NAS API 修改？
6. `who`、SID、UID/GID 是否指向同一主体？
7. 拒绝操作是 read、create、delete、rename、chmod、chown 还是 ACL 修改？
8. 父目录每一级是否允许遍历和目标目录操作？
9. SELinux 是否 Enforcing，是否存在同时间 AVC？
10. 问题是否仅出现在 NFS、SMB、某客户端或某 HA 节点？
11. ACL 回读是否被服务端重排、收紧或丢失？
12. Java 是否错误使用 POSIX API 修改 NFSv4/SMB ACL？

### 8.2 客户端标准取证

**执行端：客户端**  
**适用范围：先被动读取证**

```bash
TEST_PATH=/mnt/secure/team-a/file

findmnt -T "$TEST_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
nfsstat -m
namei -om "$TEST_PATH"
stat -c 'mode=%a uid=%u gid=%g type=%F path=%n' "$TEST_PATH"
nfs4_getfacl "$TEST_PATH"
getfacl -p "$TEST_PATH" 2>/dev/null || true
id acltest
```

实际操作验证必须以目标业务 UID 执行，并记录返回码；不要先执行 chmod 或 ACL 修复污染现场。

### 8.3 服务端标准取证

**执行端：服务端**  
**适用范围：只读取证；ACL、xattr、SID 和 AVC 输出按敏感信息管理**

```bash
BACKEND_PATH=/srv/nfs/secure/team-a/file

sudo exportfs -v
findmnt -T "$BACKEND_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
namei -om "$BACKEND_PATH"
stat -c 'mode=%a uid=%u gid=%g type=%F path=%n' "$BACKEND_PATH"
getfacl -p "$BACKEND_PATH" 2>/dev/null || true
getfattr --absolute-names -m '^(system|security|trusted)\.' \
  "$BACKEND_PATH" 2>/dev/null || true
ls -lZ "$BACKEND_PATH" 2>/dev/null || true
sudo ausearch -m AVC,USER_AVC -ts recent | tail -n 200
```

### 8.4 多协议标准取证

**执行端：Samba/NAS 管理端**

```bash
testparm -s 2>/dev/null | grep -Ei \
  'security|realm|idmap|vfs objects|acl_xattr|inherit|map acl|dos filemode'
getent passwd acltest
id acltest
smbcacls //smb.example.com/share team-a/file \
  --use-kerberos=required
```

同时从 NAS 管理面导出脱敏 ACL 模式、security style、SID 映射和最近 ACL 变更审计。不能只看 Windows 显示名称。

### 8.5 现象到根因映射

| 现象 | 第一检查点 | 不应立即执行 |
| --- | --- | --- |
| `ls -l` 有权限但读取失败 | 命名/DENY ACE、路径遍历、SELinux | `chmod 777` |
| ACL 写入后回读变化 | 后端映射、canonicalization、`aclsupport` | 重复覆盖 |
| `chmod` 后 Windows 权限消失 | mode-to-ACL 转换、ACL 权威源 | 继续递归 chmod |
| 新文件权限错误、旧文件正常 | inheritance flags、创建协议、umask | 递归重设全部对象 |
| NFS 成功、SMB 失败 | SID 映射、SMB DACL、security style | 重建用户 |
| SMB 成功、NFS 失败 | NFS `who`/UID mapping、export、SELinux | 开放 `sec=sys` |
| 能写但不能删除 | `DELETE/DELETE_CHILD`、sticky、父目录 | 给文件 write |
| rename 跨目录失败 | 两个父目录 ACL、目标对象 delete | 重试 Java move |
| 仅 Enforcing 失败 | AVC、label、domain policy | 长期 permissive |
| 仅某 HA 节点失败 | ACL/xattr/label/idmap 配置复制 | 反复切换 |
| Java ACL view 为空 | provider 不支持 | 自动 fallback chmod |

### 8.6 ACL 回读变化分析

```text
client sends ordered NFSv4 ACL
  -> server checks aclsupport
  -> backend cannot represent some ACE
  -> reject OR canonicalize OR map to POSIX subset
  -> GETATTR returns synthesized ACL
```

需要判断：

- 请求是否返回成功；
- 哪些 ACE/type/flag/permission 丢失；
- 最终 ACL 是否至少与原请求一样严格；
- mode 如何变化；
- 实际 I/O 是否符合预期；
- SMB DACL 是否同步变化。

### 8.7 SELinux 分析

AVC 取证字段：

| 字段 | 含义 |
| --- | --- |
| `scontext` | 发起访问的主体域 |
| `tcontext` | 目标对象类型 |
| `tclass` | file、dir、lnk_file 等对象类别 |
| `permissive` | 拒绝发生时是否真正 enforcing |
| permission | read、write、add_name、remove_name、rename 等 |

先检查标签是否错误，再判断 policy 缺口。`audit2why` 可辅助解释，但 `audit2allow` 输出不能未经评审直接加载。

### 8.8 Java `AccessDeniedException` 分析

```text
Java operation
  -> exact syscall class: open/create/unlink/rename/chmod/chown
  -> current PID UID/GID/groups
  -> mount and NFS security flavor
  -> ordered ACL + mode
  -> parent directory ACL
  -> SELinux AVC
  -> SMB/NAS concurrent ACL change
```

必须记录操作类型。把所有 `AccessDeniedException` 统一解释为“文件不可写”会错过父目录、ACL 管理权限和 SELinux 拒绝。

### 8.9 安全恢复顺序

```text
冻结 ACL 和身份映射写入
  -> 保存 ACL/mode/label/SID/UID/export/mount 证据
  -> 确认 ACL 权威源
  -> 修复唯一目标层：identity / ACL / mode / SELinux / export
  -> 回读服务端最终 ACL
  -> NFS 与 SMB 实际 I/O 矩阵
  -> Java 真实业务操作
  -> 观察缓存和 HA 节点
  -> 恢复变更入口
```

不要同时修改 ACL、mode、owner、SELinux 和 export，否则即使恢复，也无法证明根因。

### 8.10 生产检查清单

- [ ] 已确定 ACL 权威源和允许的写入工具。
- [ ] 已确认服务端支持的 ACE type、flags 和 permission 子集。
- [ ] 已验证 `chmod`、POSIX ACL 和 NFSv4 ACL 的转换行为。
- [ ] 已建立 SID、principal、UID/GID 的稳定映射。
- [ ] 命名用户、命名组、`OWNER@/GROUP@/EVERYONE@` 语义已实测。
- [ ] 创建、删除、rename、ACL 修改和 owner 修改均有测试。
- [ ] 继承只对新对象还是可递归传播已明确。
- [ ] SELinux label、boolean 和 policy 使用最小授权。
- [ ] 未把挂载 `context=` 误认为 Labeled NFS。
- [ ] Labeled NFS 的 NFSv4.2、LFS、双端 policy 和 HA 支持已验证。
- [ ] NFS/SMB 双协议 ACL 变更矩阵已验证。
- [ ] Java 不使用 POSIX API 无治理修改 ACL 权威源。
- [ ] ACL、SID、label 和 xattr 已纳入备份、HA 和灾备。
- [ ] 批量 ACL 变更具备快照、进度、限速和逐对象回滚。
- [ ] ACL 备份和审计材料按敏感数据管理。

## 9. 小结

1. NFSv4 ACL 是有序 ACE 列表，判定依赖 type、flags、who、permissions 和顺序。
2. `DENY` 不是无条件全局优先；访问按仍未决权限位逐项判断。
3. `DELETE/DELETE_CHILD`、父目录遍历和 rename 权限是 Java 文件操作常见盲区。
4. `mode` 只是 ACL 投影；`chmod` 可能有损重写 NFSv4 或 Windows ACL。
5. Linux knfsd、POSIX ACL 后端、原生 NFSv4 ACL NAS 的行为不能互相泛化。
6. SELinux 在 ACL/DAC 之后继续强制检查，AVC 才是服务端拒绝证据。
7. 挂载 `context=` 是统一客户端标签，不等于 NFSv4.2 Labeled NFS。
8. 多协议共享必须定义唯一 ACL 权威源并稳定映射 SID、principal 和 UID/GID。
9. Java 标准 ACL API 不保证暴露 NFSv4 ACE，权限探测不能代替真实 I/O。
10. 企业级权限修复必须可回读、可实测、可回滚，并覆盖 ACL、mode、label、身份和多协议视图。

## 10. 参考资料与关联文档

### 10.1 参考资料

- RFC 7530 第 5.9、6.2 节：NFSv4.0 `user@dns_domain`、ACL、`aclsupport`、mode 与 ACL 交互
- RFC 8881 第 6.2.1.3.2、6.2.1.4、6.4.3 节：删除判定、ACE flags、NFSv4.1 `dacl/sacl` 和自动继承
- RFC 7862 第 9.4、12.2.4 节：Labeled NFS 能力发现与 `FATTR4_SEC_LABEL`
- RFC 7204：Requirements for Labeled NFS
- RFC 7569：Security Label Format Selection Registry
- `nfs4_acl(5)`、`nfs4_getfacl(1)`、`nfs4_setfacl(1)`：Linux NFSv4 ACL 工具格式与操作
- `nfs(5)`、`exports(5)`、`nfsd(8)`：挂载、导出和 Linux NFS 实现边界
- `acl(5)`、`getfacl(1)`、`setfacl(1)`：POSIX ACL 模型
- SELinux Project、目标发行版 SELinux/NFS 官方文档：label、boolean、policy 和 AVC
- Samba `smbcacls(1)`、`smb.conf(5)`、Winbind/SSSD 官方文档：SMB ACL 与身份映射
- 目标 NAS/云文件服务 ACL、security style、多协议和灾备兼容矩阵

### 10.2 关联文档

- [NFS-KB-L1-01 NFSv3 到 NFSv4.2 的协议演进与版本选型](../L1-protocol/NFS-KB-L1-01-protocol-evolution-and-version-selection.md)
- [NFS-KB-L2-01 服务端导出与客户端挂载生产基线](../L2-deployment/NFS-KB-L2-01-export-and-mount-baseline.md)
- [NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期](../L2-deployment/NFS-KB-L2-02-systemd-autofs-mount-lifecycle.md)
- [NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)
- [NFS-KB-L3-02 RPCSEC_GSS 与 Kerberos 认证、完整性和隐私保护](NFS-KB-L3-02-rpcsec-gss-kerberos-authentication-integrity-and-privacy.md)
- [NFS-KB-L4-01 NFS 性能指标、基线与容量模型](../L4-performance/NFS-KB-L4-01-performance-metrics-baseline-and-capacity-model.md)

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-03 | 1.0.2 | 将已发布的 L4-01 加入关联文档 | 知识库交叉引用校验 |
| 2026-08-03 | 1.0.1 | 分离 NFSv4 idmap domain 与 Kerberos realm，修正 `aclsupport` 读取链、NFSv4.1 继承标志和删除/rename 语义，重构 ACL 回滚、继承、Labeled NFS 与清理实验，补充 SELinux mount 和 Kubernetes 边界 | 基于发布前协议、安全与可恢复性复查结果修订；RFC 7530、RFC 8881、RFC 7862 |
| 2026-08-03 | 1.0.0 | 初始发布 | 建立 NFSv4 ACL、SELinux、Labeled NFS、多协议权限、Java 语义和生产排障基线 |
