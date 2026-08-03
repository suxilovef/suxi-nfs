# NFS-KB-L2-02 systemd、fstab、autofs 与挂载生命周期

> 文档状态：待验证  
> 知识阶段：L2 部署与配置  
> 适用范围：Linux NFS 客户端；RHEL/Rocky/AlmaLinux 8/9、Ubuntu 22.04/24.04 常见环境；systemd 管理的 NFSv3、NFSv4.0、NFSv4.1、NFSv4.2 挂载；具体行为以目标 systemd、autofs、nfs-utils、内核及发行版文档为准  
> 版本：1.1.2
> 最后更新：2026-08-03
> 前置文档：[NFS-KB-L2-01 服务端导出与客户端挂载生产基线](NFS-KB-L2-01-export-and-mount-baseline.md)  
> 关联文档：[NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)

## 目录

- [1. 学习目标与问题边界](#1-学习目标与问题边界)
- [2. systemd 挂载模型与依赖语义](#2-systemd-挂载模型与依赖语义)
- [3. 端到端挂载生命周期](#3-端到端挂载生命周期)
- [4. fstab 与 systemd 配置设计](#4-fstab-与-systemd-配置设计)
- [5. autofs 状态机与配置设计](#5-autofs-状态机与配置设计)
- [6. Java 工程视角](#6-java-工程视角)
- [7. 生产实践与风险边界](#7-生产实践与风险边界)
- [8. 验证实验与观察指标](#8-验证实验与观察指标)
- [9. 排障证据链与检查清单](#9-排障证据链与检查清单)
- [10. 小结](#10-小结)
- [11. 参考资料与关联文档](#11-参考资料与关联文档)

## 1. 学习目标与问题边界

NFS 挂载生命周期不是一条 `mount` 命令，而是配置解析、unit 生成、网络依赖、名称解析、RPC 建连、协议协商、内核挂载、业务访问、故障恢复和卸载之间的完整状态链。生产问题通常出现在这些边界，而不是出现在 `/etc/fstab` 的语法本身。

完成本单元后，应能够：

1. 从 `/etc/fstab` 反推出 systemd 生成的 `.mount`、`.automount` unit 及其依赖关系。
2. 区分“网络管理器已启动”“网络被判定 online”“DNS 可用”“NFS 服务端可达”四种不同状态。
3. 根据业务启动依赖、访问频率和故障目标，在直接挂载、systemd automount 和 autofs 之间做出选择。
4. 解释挂载超时、RPC 重试、业务 I/O 阻塞和卸载超时为何属于不同阶段。
5. 设计可验证、可回滚且不会误伤同机其他远程挂载的变更流程。
6. 从 Java 线程栈、systemd unit、内核 NFS 指标和网络证据构建同一条故障时间线。

本文聚焦客户端挂载编排与生命周期。服务端导出参数、Kerberos、NFS 性能调优和协议恢复的完整内容由其他专题展开。

## 2. systemd 挂载模型与依赖语义

### 2.1 `/etc/fstab` 不是 systemd 下的最终运行对象

在 systemd 系统中，`systemd-fstab-generator` 在启动早期和 manager reload 时读取 `/etc/fstab`，把条目转换为临时 `.mount` unit；使用 `x-systemd.automount` 时还会生成同名路径对应的 `.automount` unit。生成文件通常位于 `/run/systemd/generator/` 及相关 generator 目录，不能直接编辑，因为重启或重新生成会覆盖它们。该行为以 `systemd-fstab-generator(8)` 和 `systemd.mount(5)` 为依据。

```text
/etc/fstab
    |
    v
systemd-fstab-generator
    |
    +--> mnt-app.automount  --监听路径访问-->  mnt-app.mount
    |                                               |
    |                                               v
    |                                          /sbin/mount.nfs
    |                                               |
    |                                               v
    +------------------------------------------ Linux NFS client
                                                    |
                                                    v
                                               NFS server
```

mount unit 名必须由绝对挂载路径转义得到。例如 `/mnt/app-data` 对应 `mnt-app\x2ddata.mount`，不能靠人工替换字符猜测。

**执行端：客户端**  
**适用范围：systemd 管理的 Linux 客户端；只读检查**

```bash
MOUNT_PATH=/mnt/app-data
MOUNT_UNIT=$(systemd-escape --path --suffix=mount "$MOUNT_PATH")
AUTO_UNIT=$(systemd-escape --path --suffix=automount "$MOUNT_PATH")
printf 'mount=%s\nautomount=%s\n' "$MOUNT_UNIT" "$AUTO_UNIT"

systemctl cat "$MOUNT_UNIT" 2>/dev/null || true
systemctl cat "$AUTO_UNIT" 2>/dev/null || true
```

预期观察：unit 名与路径严格对应；如果条目来自 `fstab`，`systemctl cat` 会指出生成来源。不存在 automount 配置时，`.automount` unit 不存在属于正常结果。

### 2.2 `.automount` 与 `.mount` 是两个不同状态对象

`.automount` active 仅表示内核 autofs 触发点已建立，不表示远端 NFS 已挂载。只有路径被访问并且 `.mount` 成功后，`findmnt` 才能看到真正的 NFS 文件系统；这是 `systemd.automount(5)` 的触发模型，不应把 automount unit 状态当作数据面健康状态。

| 状态 | `.automount` | `.mount` | 路径含义 |
| --- | --- | --- | --- |
| 尚未启动 | inactive | inactive | 普通本地目录或不存在 |
| 等待访问 | active (waiting) | inactive | autofs 触发点，尚未连接 NFS |
| 正在触发 | active (running/waiting) | activating | 访问者等待挂载作业 |
| 已挂载 | active | active (mounted) | 路径由 NFS 文件系统覆盖 |
| 挂载失败 | 通常仍可保持 active | failed | 当前访问失败；重试行为须按目标 systemd 实测 |
| 空闲卸载后 | active | inactive | 回到等待访问状态 |

因此，以下检查回答的是不同问题：

**执行端：客户端**  
**适用范围：systemd 管理的 Linux 客户端；只读状态检查**

```bash
systemctl is-active mnt-app.automount  # 触发点是否存在
systemctl is-active mnt-app.mount      # NFS 是否已挂载
findmnt --mountpoint /mnt/app          # 被动检查当前是否已有挂载
grep -F ' /mnt/app ' /proc/self/mountinfo || true
```

`mountpoint`、`findmnt --target`、`stat` 和 `ls` 可能访问路径并触发 automount。验证“访问前”状态时，应先使用 unit 状态和 `/proc/self/mountinfo`；需要执行路径访问触发时，单独记录触发前后的状态。

### 2.3 本地与远程挂载分类

systemd 会根据文件系统类型判断远程挂载，并将其纳入 `remote-fs.target`。`nfs`、`nfs4` 通常会被识别为网络文件系统；`_netdev` 可以强制把不易识别的条目按网络挂载处理。对于 NFS，保留 `_netdev` 仍有助于表达运维意图，但不能把它误解为“持续检测网络可用”。

远程 mount unit 通常会获得与以下 target 的自动依赖：

```text
network.target
      |
network-online.target
      |
remote-fs-pre.target
      |
<path>.mount
      |
remote-fs.target
```

具体 `Wants=`、`Requires=`、`After=`、`Before=` 会受 `nofail`、`noauto`、automount 和 systemd 版本影响，必须检查生成 unit，不能只按图推断。

### 2.4 `network-online.target` 的真实边界

`network-online.target` 是启动阶段的一次性同步点，不是在线状态监视器，也不保证以下条件成立。该边界来自 `systemd.special(7)` 对 target 的定义，实际等待行为还取决于具体 wait-online 服务：

- 指定网卡已经获得期望地址；
- 到 NFS 服务端的路由已经收敛；
- DNS 正反向解析正常；
- 防火墙、VPN、SD-WAN 或负载均衡路径已经可用；
- NFS 服务端已经对外提供服务。

它是否真正等待网络，由网络栈对应的 wait-online 服务决定。常见实现包括：

**执行端：客户端**  
**适用范围：使用 NetworkManager 或 systemd-networkd 的 Linux 客户端；只读检查**

```bash
systemctl status NetworkManager-wait-online.service --no-pager
systemctl status systemd-networkd-wait-online.service --no-pager
systemctl is-enabled NetworkManager-wait-online.service 2>/dev/null || true
systemctl is-enabled systemd-networkd-wait-online.service 2>/dev/null || true
```

不要为了“修复 NFS 启动慢”直接禁用 wait-online。它可能影响同机数据库、远程磁盘和依赖网络的其他服务。应先确定等待的是哪块接口、配置是否完成、服务是否确实需要同步启动。

### 2.5 三类超时不能混为一谈

| 阶段 | 代表参数或机制 | 控制对象 | 不能解决的问题 |
| --- | --- | --- | --- |
| systemd 挂载作业 | `x-systemd.mount-timeout=` / unit `TimeoutSec=` | `mount.nfs` 挂载作业等待时间 | 已挂载后的普通读写卡顿 |
| NFS RPC | `timeo`、`retrans`、`hard/soft` | 内核 NFS RPC 重传和错误返回语义 | systemd 启动目标关系 |
| 应用请求 | HTTP/RPC deadline、线程池、熔断 | 业务请求生命周期 | 取消已经陷入不可中断内核等待的文件 I/O |

`x-systemd.device-timeout=` 面向设备出现等待，只能在 `/etc/fstab` 中使用；它不是 NFS 服务端连接超时参数，不应拿来解决网络挂载问题。

## 3. 端到端挂载生命周期

### 3.1 启动阶段

直接挂载的典型启动序列如下：

```text
systemd manager 启动
  -> generator 读取 fstab
  -> 建立 remote-fs.target 依赖图
  -> 等待网络 online 同步点
  -> 启动 <path>.mount
  -> mount.nfs 解析服务端名称
  -> 建立 RPC 连接并协商 NFS 版本/安全 flavor
  -> 内核安装 superblock 和挂载对象
  -> mount unit 进入 active (mounted)
  -> 依赖该路径的业务服务启动
```

任一环节失败都可能表现成“服务启动失败”，但根因可能分别是错误 unit 名、DNS、路由、防火墙、服务端导出、认证或协议协商。排障时必须记录失败发生在哪一跳。

### 3.2 systemd automount 按需触发

```text
启动 .automount
  -> 内核建立 autofs 触发点
  -> 启动过程继续，不访问 NFS
  -> 进程第一次 lookup/open/stat 该路径
  -> 访问线程等待
  -> systemd 启动同路径 .mount
       -> 成功：放行访问，后续请求进入 NFS
       -> 失败：访问返回错误或等待至作业失败
  -> 可选 idle timeout
       -> 无活跃引用：卸载 .mount，保留 .automount
       -> 仍 busy：保持挂载，稍后再评估
```

automount 把故障从“系统启动阶段”移动到“首次路径访问阶段”。它降低启动耦合，但不会消除 DNS、网络或 NFS 故障。若第一次访问发生在 Java 请求线程，用户仍可能承受完整挂载延迟。

### 3.3 运行期故障

挂载已经 active 后，服务端不可达不会自动让 `.mount` 立刻变成 failed。VFS 挂载对象仍存在，业务线程可能在内核 RPC 重试中阻塞，systemd 看到的 unit 仍是 active。

```text
systemctl: active (mounted)
          !=
NFS 数据面健康
```

因此，健康检查至少要分层：

| 层次 | 检查内容 | 示例 |
| --- | --- | --- |
| 编排层 | unit/挂载点存在 | `systemctl`, `findmnt` |
| 协议层 | RPC 是否有超时和重传 | `nfsiostat`, `mountstats`, `nfsstat` |
| 数据层 | 受控路径能否完成真实 I/O | 带 deadline 的只读探针或专用测试文件 |
| 业务层 | Java 请求延迟、线程池和错误率 | APM、线程 dump、业务指标 |

探针必须有隔离线程、超时预算和并发上限。不要让高频健康检查在 NFS 故障时创建无限阻塞线程。

### 3.4 停机与卸载阶段

mount unit 会参与关机卸载顺序。业务服务如果通过明确依赖绑定挂载，应先停止业务、关闭文件和 mmap，再卸载 NFS。没有依赖关系时，关机阶段可能出现应用仍在写而系统开始卸载，或挂载 busy 导致停机延长。

安全顺序是：

```text
停止入口流量
  -> 停止产生新 NFS I/O
  -> 等待在途任务并关闭文件/锁/mmap
  -> 停止业务服务
  -> 验证挂载不再 busy
  -> 正常 umount
  -> 停止网络
```

`umount -l` 只从当前命名空间延迟分离挂载，不等于远端 I/O 已完成；`umount -f` 对 NFS 的行为依赖内核和状态，可能导致应用错误或未完成写入。二者都不是常规关机策略，只能作为故障处置手段，在确认业务停止、数据风险和恢复路径后使用。

## 4. fstab 与 systemd 配置设计

### 4.1 先按业务依赖分类

| 类型 | 业务特征 | 建议起点 |
| --- | --- | --- |
| 启动强依赖 | 没有 NFS 数据就不得启动，错误启动比延迟启动更危险 | 直接 mount + 业务 unit 显式依赖 |
| 启动弱依赖 | 主功能可启动，NFS 功能可延后恢复 | `nofail` + automount + 业务降级 |
| 低频大量路径 | 数百或数千共享按需访问 | autofs map |
| 高频低延迟 | 每个请求都访问固定共享 | 直接 mount，避免首次触发和反复过期 |
| 运维工具路径 | 偶发人工访问，不能阻塞开机 | systemd automount 或 autofs |

同一挂载路径只能由一种生命周期管理器负责。不要让手写 `.mount`、`fstab` automount 和 autofs 同时管理 `/mnt/app`。

### 4.2 启动强依赖示例

`/etc/fstab`：

```fstab
nfs.example.com:/app  /srv/app-data  nfs4  nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2,_netdev,x-systemd.mount-timeout=45s  0  0
```

业务 unit drop-in：

```ini
# /etc/systemd/system/myapp.service.d/nfs.conf
[Unit]
RequiresMountsFor=/srv/app-data
After=network-online.target
```

`RequiresMountsFor=` 会为访问该路径所需的 mount unit 增加 requirement 和 ordering 依赖。它比仅写 `After=mnt-app.mount` 更不易因路径转义出错，也能覆盖路径层级上的挂载依赖。

**执行端：客户端**  
**风险：修改启动依赖可能导致业务无法启动；先在测试节点验证**  
**回滚：移除 drop-in 和对应 fstab 变更，执行 `systemctl daemon-reload`，恢复经审批的原挂载方式**

```bash
sudo systemd-analyze verify myapp.service
systemctl cat myapp.service
systemctl show myapp.service -p Requires -p After
systemctl list-dependencies myapp.service --all
```

注意：`RequiresMountsFor=` 是 unit 配置指令，不一定作为 `systemctl show -p` 的独立运行时属性列出；应通过 `systemctl cat`、`Requires=` 和依赖图确认其效果。`After=network-online.target` 只表示排序，不会自动拉起 target；NFS mount 自动依赖和网络管理器配置才决定实际等待关系。不要用这一行替代挂载 unit 的依赖验证。

### 4.3 启动弱依赖与按需挂载示例

```fstab
nfs.example.com:/archive  /mnt/archive  nfs4  nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2,_netdev,nofail,x-systemd.automount,x-systemd.mount-timeout=30s,x-systemd.idle-timeout=15min  0  0
```

选项语义：

| 选项 | 作用 | 生产边界 |
| --- | --- | --- |
| `_netdev` | 按网络挂载参与依赖生成 | 不代表远端持续健康 |
| `nofail` | 降低该挂载对启动 target 的强制约束 | 业务仍须处理路径不可用 |
| `x-systemd.automount` | 访问时触发对应 `.mount` | 首次访问承担挂载延迟 |
| `x-systemd.mount-timeout=30s` | 限制单次挂载作业等待 | 不限制挂载后的 I/O |
| `x-systemd.idle-timeout=15min` | 尝试卸载空闲挂载 | busy 时不能卸载；过短会导致抖动 |

不要将 `noauto` 与 `x-systemd.automount` 组合使用。二者对 generator 是否生成、是否启用 mount/automount unit 的影响容易造成版本差异和运维误判；需要按需挂载时只使用 `x-systemd.automount`，并通过生成的 unit 和目标级启动命令验证。

### 4.4 变更和验证 `/etc/fstab`

直接执行 `mount -a` 会对所有符合条件但尚未挂载的条目产生影响，不适合作为无边界的生产验证命令。推荐先静态检查，再只操作目标 unit。

**执行端：客户端**  
**风险：`daemon-reload` 会重新加载全部 unit 配置，但不会自动重启已有服务；启动目标 unit 会访问远端**

```bash
sudo findmnt --verify --verbose
sudo systemctl daemon-reload

MOUNT_PATH=/mnt/archive
MOUNT_UNIT=$(systemd-escape --path --suffix=mount "$MOUNT_PATH")
AUTO_UNIT=$(systemd-escape --path --suffix=automount "$MOUNT_PATH")

systemctl cat "$MOUNT_UNIT"
systemctl cat "$AUTO_UNIT"
systemctl show "$MOUNT_UNIT" -p After -p Before -p Wants -p Requires \
  -p TimeoutUSec -p JobTimeoutUSec -p JobRunningTimeoutUSec
sudo systemctl start "$AUTO_UNIT"
systemctl status "$AUTO_UNIT" "$MOUNT_UNIT" --no-pager
```

`findmnt --verify` 能发现部分语法和可用性问题，但不能证明 DNS、权限、协议协商和故障恢复正确。修改已经挂载的 fstab 选项也不会自动改变现有内核挂载；需要评估 `mount -o remount` 是否支持目标选项，或在维护窗口卸载后重新挂载。

`TimeoutUSec` 用于观察 mount unit 的执行超时；`JobTimeoutUSec` 和 `JobRunningTimeoutUSec` 属于 systemd job 层，不能单独证明 `mount.nfs` 已按预期超时。最终应结合 unit journal 中的实际开始、失败和结束时间确认。

### 4.5 NFS `bg` 与 systemd 的交互

传统 NFS `bg` 表示首次挂载失败后让 `mount.nfs` 在后台重试。systemd 环境会对 `bg` 做兼容转换；systemd 版本常见实现会组合无限挂载作业超时、大量重试、`fg` 和 `nofail` 语义。结果可能与运维人员直觉不同，并造成长期后台重试或弱化启动失败信号。

生产 systemd 环境建议明确使用：

- automount 控制“何时发起挂载”；
- `x-systemd.mount-timeout=` 控制挂载作业预算；
- `nofail` 或业务 unit requirement 明确启动失败策略；
- 监控和恢复流程控制后续重试。

若遗留条目使用 `bg`，必须查看目标 systemd 生成的 unit 和 `systemd.mount(5)` 文档后再迁移，不能机械删除。

### 4.6 独立 `.mount` unit 的适用场景

复杂依赖可以使用原生 unit，但 mount unit 文件名必须与 `Where=` 路径完全匹配。

```ini
# /etc/systemd/system/srv-app\x2ddata.mount
[Unit]
Description=Application NFS data
Wants=network-online.target
After=network-online.target

[Mount]
What=nfs.example.com:/app
Where=/srv/app-data
Type=nfs4
Options=nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2,_netdev
TimeoutSec=45

[Install]
WantedBy=remote-fs.target
```

**执行端：客户端**  
**适用范围：需要版本管理复杂 unit 依赖的场景**  
**风险：不要为同一路径同时保留 fstab 条目；冲突会导致来源不清或启动失败**  
**回滚：disable 该 unit，恢复原 fstab 条目，daemon-reload 后只启动目标挂载**

```bash
MOUNT_PATH=/srv/app-data
systemd-escape --path --suffix=mount "$MOUNT_PATH"
sudo systemd-analyze verify "/etc/systemd/system/$(systemd-escape --path --suffix=mount "$MOUNT_PATH")"
```

## 5. autofs 状态机与配置设计

### 5.1 autofs 与 systemd automount 的区别

二者都使用内核 autofs 机制进行按需触发，但配置、map 能力和管理进程不同。

| 能力 | systemd automount | autofs daemon |
| --- | --- | --- |
| 配置入口 | fstab 或 `.automount` unit | master map + direct/indirect map |
| 固定少量路径 | 简洁 | 可以，但管理成本更高 |
| 大量/动态路径 | 能力有限 | 支持 wildcard、LDAP 等 map 来源 |
| 依赖关系 | 原生 systemd unit 图 | 由 autofs 服务和 map 管理 |
| 日志入口 | 对应 `.automount/.mount` journal | `autofs.service` journal，加 mount helper 日志 |
| 空闲过期 | `TimeoutIdleSec` / fstab idle timeout | master/map timeout |

选择原则：固定少量路径优先 systemd automount；大量、分层、动态或集中目录映射优先 autofs。

### 5.2 direct map 示例

`/etc/auto.master.d/nfs.autofs`：

```text
/-  /etc/auto.nfs  --timeout=600
```

`/etc/auto.nfs`：

```text
/mnt/archive  -fstype=nfs4,nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2  nfs.example.com:/archive
/mnt/reports  -fstype=nfs4,nfsvers=4.1,proto=tcp,hard,timeo=600,retrans=2  nfs.example.com:/reports
```

`/-` 表示 direct map，key 本身就是绝对挂载路径。它适合离散且固定的生产路径。

**适用范围：** 已安装 `autofs` 的 RHEL/Rocky/AlmaLinux 8/9 或 Ubuntu 22.04/24.04；具体 master map 自动包含目录以发行版配置为准。  
**预期效果：** 访问指定 direct key 时按需调用 `mount.nfs`，空闲约 600 秒后尝试卸载。  
**验证：** 使用 `automount -m`、`systemctl status autofs`、`findmnt --target` 和对应 journal 检查 map 与实际挂载。  
**回滚：** 恢复上一版 map，reload `autofs`；停止使用目标路径并确认不 busy 后再正常卸载，不批量强制卸载其他路径。

### 5.3 indirect map 与 wildcard

`/etc/auto.master.d/projects.autofs`：

```text
/projects  /etc/auto.projects  --timeout=600
```

`/etc/auto.projects`：

```text
*  -fstype=nfs4,nfsvers=4.1,proto=tcp,hard  nfs.example.com:/projects/&
```

访问 `/projects/team-a` 时，`*` 匹配 `team-a`，`&` 替换为该 key，最终挂载 `nfs.example.com:/projects/team-a`。

通配 map 会把客户端输入拼入远端路径。生产使用前必须限制可用命名空间、验证服务端导出边界，并防止用户借由可控 key 访问未授权路径。AUTH_SYS 本身不提供可信身份边界；强安全场景需要结合 Kerberos、目录权限和网络隔离。

**适用范围：** 仅适用于远端路径命名空间已经受控、且服务端导出边界可验证的测试或生产场景。  
**预期效果：** 访问 `/projects/<key>` 时按需映射到服务端 `/projects/<key>`。  
**验证与回滚：** 先使用非敏感测试 key 验证 `automount -m`、journal、`findmnt` 和权限；恢复旧 map 后 reload，已挂载路径按 direct map 的回滚流程处理。

### 5.4 autofs 生命周期

```text
autofs.service 启动
  -> 读取 master map
  -> 建立 direct/indirect 触发点
  -> 进程访问 key
  -> automount daemon 查询 map
  -> 调用 mount.nfs
       -> 成功：返回目录内容
       -> 失败：访问报错，日志记录原因
  -> 空闲计时
       -> 无 cwd/open fd/mmap/子挂载等引用：卸载
       -> busy：延后过期
```

“空闲”不是“没有新请求”。进程当前工作目录、打开文件、mmap、文件锁、bind mount 或容器 mount namespace 都可能使挂载保持 busy。

### 5.5 加载、验证和回滚 map

**执行端：客户端**  
**适用范围：已安装 autofs；测试节点优先**  
**风险：reload 会重新读取全部 map；已挂载条目通常不会因 map 修改自动按新参数重挂**

```bash
sudo automount -m
sudo systemctl reload autofs
systemctl status autofs --no-pager
journalctl -u autofs --since "5 min ago" --no-pager

# 访问前后分别检查；ls 本身会触发按需挂载
findmnt --target /mnt/archive
timeout 35s ls -ld /mnt/archive
findmnt --target /mnt/archive -o TARGET,SOURCE,FSTYPE,OPTIONS
```

`timeout` 只能约束命令进程的等待时间，不保证底层硬挂载 I/O 已取消，也不应作为数据一致性保证。验证 map 变更时，应先在测试 key 上操作；要让已挂载条目采用新参数，必须确认不再 busy，并在维护窗口正常卸载后重新触发。

回滚步骤：恢复上一个 map 版本，执行 `automount -m` 检查、reload autofs；若错误条目已挂载，停止使用该路径并正常卸载，再访问旧 key 验证。不要为了应用 map 变更批量强制卸载全部 autofs 路径。

## 6. Java 工程视角

### 6.1 看似无害的元数据调用也会触发挂载

以下 Java API 可能触发路径解析，从而启动 automount：

```java
Path path = Path.of("/mnt/archive/report.json");

Files.exists(path);
Files.readAttributes(path, BasicFileAttributes.class);
Files.newDirectoryStream(path.getParent());
FileChannel.open(path, StandardOpenOption.READ);
```

`Files.exists()` 返回 `false` 既可能表示文件不存在，也可能表示状态无法确定。不能把它用作 NFS 健康性的唯一依据，更不能据此自动创建或覆盖业务文件。

### 6.2 应用超时不等于内核 I/O 取消

当线程进入 `open(2)`、`read(2)`、`write(2)`、`fsync(2)`、`fcntl(2)` 等调用后，NFS `hard` 重试可能使线程长时间停留在内核。`Future.cancel(true)`、`Thread.interrupt()` 或 HTTP 请求超时不保证底层文件 I/O 立即终止。虚拟线程可以降低线程对象成本，但不会改变 NFS 协议重试和系统调用取消语义。

生产措施应包括：

- NFS I/O 使用有界并发隔离，不能和核心请求线程池无上限混用；
- 就绪探针和业务 I/O 分开，避免探针雪崩；
- 入口 deadline 到期后停止创建新任务，但仍监控未退出的 native I/O；
- 用线程 dump、`/proc/<pid>/stack`、mountstats 和网络指标确认阻塞位置；
- 把“服务端恢复后线程能否继续”纳入故障演练，而不是遇到阻塞就切换为 `soft`。

### 6.3 服务启动依赖与健康状态

对于强依赖 NFS 的 Java 服务，可用 `RequiresMountsFor=` 保证挂载先于 JVM 启动，但这只能证明 mount 作业成功。服务启动后仍需区分：

```text
liveness：JVM 是否应该被重启
readiness：是否接受需要 NFS 的新流量
dependency health：受控 NFS 操作是否在预算内完成
```

不要让 liveness 探针同步访问 NFS。NFS 故障时反复杀 JVM，可能中断恢复中的锁和状态、扩大启动风暴，并增加服务端恢复压力。更合理的策略通常是 readiness 摘流、限制依赖探针并保留 JVM 供诊断和恢复。

### 6.4 WatchService 与跨客户端变更

Linux 上 Java `WatchService` 通常依赖本机内核通知机制。另一台 NFS 客户端对文件的修改不一定产生本客户端可见的 inotify 事件，因此不能把 WatchService 当作跨客户端可靠消息总线。需要可靠通知时，应使用消息系统、数据库版本号或周期性带校验的扫描机制。

### 6.5 最小化诊断代码

下面的代码用于受控测试目录，记录操作阶段和耗时；它不是强制取消 NFS I/O 的方案：

```java
import java.io.IOException;
import java.nio.ByteBuffer;
import java.nio.channels.FileChannel;
import java.nio.file.Path;
import java.nio.file.StandardOpenOption;
import java.util.concurrent.TimeUnit;

Path probe = Path.of("/mnt/archive/.health/read-only-probe");
long started = System.nanoTime();
try (FileChannel channel = FileChannel.open(probe, StandardOpenOption.READ)) {
    ByteBuffer buffer = ByteBuffer.allocate(1);
    int read = channel.read(buffer);
    long elapsedMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started);
    System.out.printf("stage=read result=%d elapsedMs=%d%n", read, elapsedMs);
} catch (IOException e) {
    long elapsedMs = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - started);
    System.err.printf("stage=open-or-read elapsedMs=%d error=%s%n", elapsedMs, e);
}
```

需要同时记录实际挂载参数、客户端主机、JVM 版本、NFS 服务端地址和同时间窗的 RPC 指标，单独一条 Java 异常不足以定位根因。

## 7. 生产实践与风险边界

### 7.1 方案选择矩阵

| 需求 | 直接 mount | systemd automount | autofs |
| --- | --- | --- | --- |
| JVM 启动前必须可用 | 推荐 | 不推荐依赖首次业务访问 | 通常不推荐 |
| 开机不能被 NFS 阻塞 | 配合 `nofail`，但业务需降级 | 推荐 | 推荐 |
| 首次访问延迟敏感 | 推荐 | 需预热 | 需预热 |
| 固定 1 至 10 个路径 | 推荐 | 推荐 | 可用 |
| 数百个动态 key | 不适合 | 不适合 | 推荐 |
| 需要 systemd 精确 unit 依赖 | 推荐 | 推荐 | 需围绕 autofs.service 设计 |

### 7.2 `nofail` 的风险边界

`nofail` 只调整系统启动对该挂载失败的容忍，不会自动让业务安全降级。如果 NFS 路径在挂载失败时露出底层本地目录，应用可能把数据写入本地根文件系统，等 NFS 恢复挂载后这些文件又被覆盖隐藏，形成典型的“文件丢失”假象。

防护措施：

1. 本地挂载点由 root 持有，并禁止业务用户写入。
2. 应用启动或每次写入前验证目标文件系统身份，而非只验证目录存在。
3. 对弱依赖功能显式降级，不回退到同路径本地写。
4. 监控根分区异常增长和 NFS mount identity。

**执行端：客户端**

```bash
MOUNT_PATH=/mnt/archive
mountpoint -q "$MOUNT_PATH" || { echo "not mounted" >&2; exit 1; }
findmnt --target "$MOUNT_PATH" -n -o FSTYPE,SOURCE,OPTIONS
stat -f -c 'fstype=%T fsid=%i' "$MOUNT_PATH"
grep -F " $MOUNT_PATH " /proc/self/mountinfo || true
```

`stat -f` 的文件系统类型和 fsid 只能作为辅助证据，不能单独证明路径来自预期 NFS 服务端；生产检查必须结合 `findmnt` 的 source/options 与 `/proc/self/mountinfo`。

### 7.3 空闲卸载的风险边界

过短的 idle timeout 可能造成：

- 周期任务每次都承担 DNS、TCP、RPC 和挂载协商延迟；
- 多进程同时访问时出现挂载抖动；
- 监控扫描不断触发本应空闲的挂载；
- 服务端切换后第一次请求才暴露恢复问题。

需要记录触发次数、挂载耗时、卸载失败和首次访问 p99，再决定 timeout。高频固定共享通常无需空闲卸载。

### 7.4 DNS、VIP 与主机名

挂载时解析出的地址、NFSv4 客户端状态身份和现有 TCP 连接不会因为 DNS TTL 到期就自动完成无缝切换。DNS 改址不是完整的 NFS 高可用协议。VIP、集群文件系统、服务端恢复目录、租约/stateid 恢复和 fencing 必须整体设计。

任何“修改 `/etc/hosts` 后重启 mount 就能切换”的方案，都要验证打开文件、锁、未提交写入和服务端身份连续性。

### 7.5 生产变更最小边界

推荐流程：

```text
保存目标条目和生成 unit 证据
  -> 静态校验新配置
  -> 测试节点/测试路径验证
  -> 确认业务 drain 与回滚条件
  -> daemon-reload
  -> 只启动或重启目标 unit
  -> 验证 mount identity、权限、读写、锁和指标
  -> 观察一个完整业务周期
```

不要直接重启 `remote-fs.target`、批量执行 `umount -a -t nfs` 或重启 autofs 后强制清理全部路径。这些操作可能同时影响同机无关业务。

## 8. 验证实验与观察指标

以下实验必须在测试客户端和测试导出执行。当前文档未在真实 Linux NFS 环境验证，因此状态为“待验证”。

### 8.1 实验一：验证 generator 和依赖图

**执行端：客户端**  
**前提：已准备测试 fstab 条目；不要使用生产业务路径**

```bash
sudo findmnt --verify --verbose
sudo systemctl daemon-reload

MOUNT_PATH=/mnt/archive
MOUNT_UNIT=$(systemd-escape --path --suffix=mount "$MOUNT_PATH")
AUTO_UNIT=$(systemd-escape --path --suffix=automount "$MOUNT_PATH")

systemctl cat "$MOUNT_UNIT"
systemctl cat "$AUTO_UNIT"
systemctl show "$MOUNT_UNIT" -p Id -p LoadState -p ActiveState -p SubState \
  -p After -p Before -p Wants -p Requires -p TimeoutUSec
systemctl list-dependencies remote-fs.target --all
```

记录：systemd 版本、生成 unit 内容、依赖关系、timeout 的实际属性名和 fstab 来源。不同 systemd 版本的 `systemctl show` 属性可能不同，应以 `systemctl show --all` 为准确认。

### 8.2 实验二：观察按需触发状态转换

**执行端：客户端**

```bash
MOUNT_PATH=/mnt/archive
MOUNT_UNIT=$(systemd-escape --path --suffix=mount "$MOUNT_PATH")
AUTO_UNIT=$(systemd-escape --path --suffix=automount "$MOUNT_PATH")

sudo systemctl start "$AUTO_UNIT"
systemctl is-active "$AUTO_UNIT"
systemctl is-active "$MOUNT_UNIT" || true
findmnt --target "$MOUNT_PATH"

START_NS=$(date +%s%N)
timeout 35s stat "$MOUNT_PATH"
RC=$?
END_NS=$(date +%s%N)
printf 'rc=%s elapsed_ms=%s\n' "$RC" "$(( (END_NS - START_NS) / 1000000 ))"

systemctl status "$AUTO_UNIT" "$MOUNT_UNIT" --no-pager
findmnt --target "$MOUNT_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
```

预期观察：访问前 automount active 而 mount inactive；`stat` 触发挂载；成功后 mount active。若 `findmnt --target` 在访问前显示 `autofs`，应区分它与访问后出现的 `nfs4`。

### 8.3 实验三：服务端不可达故障注入

**执行端：隔离测试客户端**  
**风险：禁止在共享生产网络修改防火墙或关闭生产服务；故障注入必须仅影响测试客户端到测试服务端的受控流量**

步骤：

1. 记录正常挂载耗时和 mountstats。
2. 用实验网络策略阻断测试客户端到测试服务端 TCP/2049；NFSv3 还要考虑 rpcbind、mountd、lockd 和 statd。
3. 在未挂载的 automount 路径执行带外层 watchdog 的 `stat`。
4. 在已挂载路径执行只读操作，比较挂载作业 timeout 与 `hard` RPC 阻塞。
5. 恢复网络，观察原线程是否继续、RPC retrans 是否停止、mount unit 是否需要 reset/restart。

**执行端：隔离测试客户端**  
**适用范围：故障注入期间的只读观察；命令中的主机名必须替换为测试服务端**

观察命令：

```bash
nfsiostat 1 10
nfsstat -c
cat /proc/self/mountstats
journalctl -b -u "$(systemd-escape --path --suffix=mount /mnt/archive)" --no-pager
ss -tn dst nfs.example.com
```

回滚：撤销仅针对测试流量的网络规则，验证 TCP/2049 和必要 RPC 端口恢复；结束测试进程前保存线程栈、journal、mountstats 和抓包时间线。

### 8.4 实验四：防止写入底层本地目录

**执行端：测试客户端**  
**风险：仅使用空的测试挂载点；卸载前确认没有业务进程使用**

```bash
MOUNT_PATH=/mnt/nfs-kb-l2-02
sudo install -d -o root -g root -m 0555 "$MOUNT_PATH"

# 未挂载时，业务用户写入必须失败；成功表示存在本地误写风险
if sudo -u nobody sh -c 'printf test > "$1/local-write"' sh "$MOUNT_PATH"; then
    echo "ERROR: local directory is writable" >&2
    rm -f "$MOUNT_PATH/local-write"
    exit 1
else
    echo "OK: local directory is not writable"
fi

# 挂载测试导出后，按服务端授权验证写入
mountpoint "$MOUNT_PATH"
findmnt --target "$MOUNT_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
```

预期观察：NFS 未挂载时，业务身份不能向底层目录写入；挂载后权限由导出和远端文件系统共同决定。

### 8.5 必须采集的指标

| 维度 | 指标或证据 | 用途 |
| --- | --- | --- |
| systemd | unit active/substate、job duration、journal | 判断编排阶段 |
| automount | 触发次数、首次访问延迟、idle 卸载结果 | 判断按需挂载成本 |
| NFS RPC | ops、timeouts、retrans、avg RTT、avg exe | 判断协议与服务端等待 |
| TCP | connect 失败、重传、RTO、连接地址 | 判断网络路径 |
| DNS | 查询耗时、返回地址、失败码 | 判断名称解析 |
| Java | 阶段耗时、线程池 active/queue、线程栈 | 判断应用放大效应 |
| 主机 | 根分区容量、挂载点 fs type/source | 发现本地误写 |

## 9. 排障证据链与检查清单

### 9.1 先回答六个问题

1. 路径由 fstab、原生 mount unit、systemd automount 还是 autofs 管理？
2. 当前看到的是 autofs 触发点，还是真正的 NFS mount？
3. 故障发生在开机、首次访问、稳定运行、服务端切换还是关机？
4. 阻塞的是 mount helper，还是已挂载文件系统上的 NFS RPC？
5. unit active 是否与实际数据面健康矛盾？
6. NFS 缺失时，应用是否写入了底层本地目录？

### 9.2 分层取证命令

**执行端：客户端**  
**适用范围：只读诊断；`stat`/`ls` 可能触发 automount，执行前先判断是否允许**

```bash
MOUNT_PATH=/mnt/archive
MOUNT_UNIT=$(systemd-escape --path --suffix=mount "$MOUNT_PATH")
AUTO_UNIT=$(systemd-escape --path --suffix=automount "$MOUNT_PATH")

# 配置与来源
findmnt --fstab --evaluate --target "$MOUNT_PATH"
systemctl cat "$MOUNT_UNIT" "$AUTO_UNIT" 2>/dev/null || true
systemctl show "$MOUNT_UNIT" "$AUTO_UNIT" \
  -p LoadState -p ActiveState -p SubState -p Result -p After -p Wants -p Requires 2>/dev/null

# 当前挂载身份；避免先用 ls 触发
findmnt --target "$MOUNT_PATH" -o TARGET,SOURCE,FSTYPE,OPTIONS
grep -F " $MOUNT_PATH " /proc/self/mountinfo || true

# 日志与启动关键路径
journalctl -b -u "$MOUNT_UNIT" -u "$AUTO_UNIT" --no-pager
systemd-analyze critical-chain remote-fs.target

# NFS 与网络
nfsstat -m
nfsiostat 1 5
rpcinfo -T tcp nfs.example.com nfs 4
getent ahosts nfs.example.com
ss -ton dst nfs.example.com
```

NFSv3 不能只检查 `nfs 4` 或 TCP/2049，还要按服务端固定端口策略检查 rpcbind、mountd、lockd 和 statd。Kerberos 场景还需检查 GSS daemon、凭据、DNS 和时间同步。

### 9.3 症状到证据的映射

| 症状 | 优先证据 | 常见根因方向 |
| --- | --- | --- |
| 开机等待很久 | wait-online、mount unit journal、critical-chain | 网络同步、DNS、服务端不可达、挂载重试 |
| automount active 但目录访问卡住 | `.mount` 状态、mount helper、DNS/RPC | 触发点正常，实际挂载失败 |
| mount active 但 Java 请求卡住 | mountstats、线程栈、TCP、服务端指标 | 运行期 RPC 重试或后端存储慢 |
| 重启后目录里“少了文件” | mount identity、根分区、底层目录 | NFS 未挂载期间误写本地目录 |
| idle timeout 不卸载 | `fuser`/`lsof`、cwd、mmap、子挂载、namespace | 路径仍 busy |
| reload autofs 后参数未变 | `findmnt` 实际 options、活动 mount | 旧挂载未卸载重建 |
| 关机卡在 unmount | 应用停止顺序、busy 引用、RPC 状态 | 仍有打开文件或服务端不可达 |

`lsof +D` 会递归遍历目录，可能非常慢并触发大量 NFS 操作。故障期优先使用挂载点级别的 `fuser -vm`、`findmnt`、进程 cwd/fd 定向检查，避免无边界扫描。

### 9.4 恢复动作的顺序

```text
停止新流量或隔离受影响功能
  -> 保存 unit/journal/mountstats/线程栈/网络证据
  -> 恢复 DNS、路由、端口或服务端
  -> 观察 hard RPC 是否自然恢复
  -> 对失败的 mount 作业执行目标级 reset-failed/restart
  -> 验证 mount identity 和真实 I/O
  -> 恢复业务流量
```

仅对目标 unit 操作：

**执行端：客户端**  
**风险：重启 mount unit 可能中断仍在运行的 NFS I/O；确认业务已摘流且目标路径无活动引用后执行**

```bash
MOUNT_UNIT=$(systemd-escape --path --suffix=mount /mnt/archive)
sudo systemctl reset-failed "$MOUNT_UNIT"
sudo systemctl restart "$MOUNT_UNIT"
systemctl status "$MOUNT_UNIT" --no-pager
```

重启 mount 前必须确认没有进程使用该路径，并评估未提交写入、文件锁和 NFSv4 状态恢复。若 mount 由 automount 管理，通常应保留 `.automount`，只恢复对应 `.mount`；具体重试行为需在目标 systemd 版本验证。

### 9.5 发布前检查清单

- [ ] 已明确路径唯一管理者，没有 fstab、原生 unit 与 autofs 冲突。
- [ ] 已检查 generator 生成的 `.mount/.automount` 内容和真实依赖。
- [ ] 已区分强依赖、弱依赖、按需访问和动态 map 场景。
- [ ] 已验证 network-online 的具体 provider 和目标接口策略。
- [ ] 已分别定义挂载作业、NFS RPC 和应用请求的超时边界。
- [ ] 已防止 NFS 缺失时业务写入底层本地目录。
- [ ] 已验证启动、首次访问、运行期断网、服务端恢复和关机顺序。
- [ ] 已记录 Java 阻塞、线程池隔离、readiness 和 liveness 策略。
- [ ] 所有变更都有目标级操作、观察指标和回滚步骤。
- [ ] 未把 `soft`、强制卸载、lazy unmount 或批量重启当成通用恢复方案。

## 10. 小结

1. systemd 下的真实运行对象是路径对应的 `.mount` 和 `.automount` unit，`fstab` 是生成输入。
2. automount 只把远端依赖推迟到路径访问，不会消除 NFS 故障；首次访问者会承担挂载延迟。
3. `network-online.target` 是启动同步点，不是 DNS、路由或 NFS 服务健康保证。
4. systemd mount timeout、NFS RPC 重试和 Java 请求 timeout 属于三个不同控制层。
5. `nofail` 必须配合业务降级和底层目录防写，否则可能造成隐蔽的本地误写。
6. autofs 适合大量动态路径；固定少量路径通常由 systemd automount 更直接地管理。
7. unit active 不等于 NFS 数据面健康，必须用 mount identity、RPC、网络和真实 I/O 共同验证。
8. 生产恢复应先保存证据、恢复依赖，再对单个目标 unit 操作，避免扩大影响范围。

## 11. 参考资料与关联文档

### 11.1 参考资料

- `systemd.mount(5)`：mount unit、自动依赖、`_netdev`、`nofail` 和 NFS `bg` 兼容语义
- `systemd.automount(5)`：automount unit、路径触发与 idle timeout
- `systemd-fstab-generator(8)`：fstab 到 unit 的生成规则
- `systemd.unit(5)`：`RequiresMountsFor=` 和依赖关系
- `systemd.special(7)`：`network-online.target`、`remote-fs-pre.target`、`remote-fs.target`
- `fstab(5)`、`nfs(5)`、`mount.nfs(8)`：挂载字段与 NFS 客户端选项
- `autofs(5)`、`auto.master(5)`、`automount(8)`：autofs map、daemon 和超时语义
- Linux 内核 NFS 客户端文档及目标发行版 systemd/autofs/NFS 管理指南

### 11.2 关联文档

- [NFS-KB-L2-01 服务端导出与客户端挂载生产基线](NFS-KB-L2-01-export-and-mount-baseline.md)
- [NFS-KB-L1-03 NFSv4 状态、租约、stateid 与恢复](../L1-protocol/NFS-KB-L1-03-nfsv4-state-lease-stateid-and-recovery.md)
- [NFS-KB-L3-01 AUTH_SYS、身份映射与 Linux 权限模型](../L3-security/NFS-KB-L3-01-auth-sys-identity-mapping-and-linux-permissions.md)
- [NFS-KB-L4-01 NFS 性能指标、基线与容量模型](../L4-performance/NFS-KB-L4-01-performance-metrics-baseline-and-capacity-model.md)
- 待建立：`NFS-KB-L5-01` NFS 挂载卡死与不可中断 I/O 排障

## 变更记录

| 日期 | 版本 | 变更内容 | 证据或原因 |
| --- | --- | --- | --- |
| 2026-08-03 | 1.1.2 | 将已发布的 L4-01 加入关联文档 | 知识库交叉引用校验 |
| 2026-08-02 | 1.1.1 | 将已发布的 L3-01 加入关联文档链接 | 知识库交叉引用校验 |
| 2026-08-02 | 1.1.0 | 修正 systemd timeout 验证属性、禁止混用 `noauto` 与 automount、修复本地误写实验退出逻辑，补充 automount 触发副作用、autofs 配置回滚、Java import、挂载身份证据和执行端标注 | 基于文档审查结果修订；依据 systemd/autofs man pages 复核 |
| 2026-08-02 | 1.0.0 | 初始发布 | 建立 systemd、fstab、autofs 与 NFS 挂载生命周期生产知识基线 |
