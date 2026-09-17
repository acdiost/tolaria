---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
tags:
  - HPC
  - NFS
  - autofs
  - quota
  - LDAP
---

# 从零部署 NFS、autofs 与 XFS quota 共享家目录

本文说明如何让计算节点 `compute01` 按需挂载控制节点 `control` 上的用户家目录，并使用 LDAP 保证两端 UID/GID 一致、使用 XFS quota 在服务器端限制用户空间。

本次实际部署日期为 2026-08-25，拓扑如下：

| 角色 | 主机名 | 地址 | 功能 |
| --- | --- | --- | --- |
| NFS/LDAP/Slurm 控制节点 | `control` | `192.0.2.10` | 导出 `/home`、执行 quota |
| NFS 客户端/计算节点 | `compute01` | `192.0.2.20` | autofs 按用户挂载 |

最终访问路径保持为 `/home/<用户名>`。用户在 `compute01` 登录或访问家目录时，autofs 才挂载 `control:/home/<用户名>`。

## 1. 工作原理

整个链路依赖四个部分：

1. LDAP 为同一个用户提供固定的 `uidNumber`、`gidNumber` 和 `homeDirectory`。
2. SSSD 让 `control` 与 `compute01` 将该 LDAP 条目解析为相同的 Linux UID/GID。
3. autofs 在访问 `/home/<用户名>` 时发起 NFSv4 挂载。
4. XFS quota 在实际存放数据的 `control` 上计量并拒绝超额写入。

NFS 的 `sec=sys` 根据数字 UID/GID 判断文件身份，所以“两端 UID/GID 一致”不是可选项。仅让用户名相同而数字 ID 不同，会造成属主错乱或权限拒绝。

## 2. 上线前检查

先确认主机名、地址和 Slurm 是否空闲：

```bash
hostname
getent hosts control compute01
squeue -h
sinfo -N -o '%N %T'
```

检查两端 `/home` 是否已有本地数据：

```bash
find /home -mindepth 1 -maxdepth 2 -ls
findmnt /home
```

不要直接把整个远端 `/home` 固定挂到非空的本地 `/home`，否则原目录会被挂载覆盖而暂时不可见。本集群因此采用 autofs 通配符映射，按用户名只挂载一个子目录。

在 `control` 查看文件系统类型和 quota 状态：

```bash
findmnt -no SOURCE,FSTYPE,OPTIONS /
xfs_quota -x -c state /
```

本机 `/home` 位于根 XFS 文件系统 `/dev/mapper/rl-root`，初始状态为 `noquota`。如果 `/home` 是独立文件系统，应将后文的挂载点 `/` 替换为 `/home`，这样配额只统计家目录数据。

## 3. 安装软件包

在两台服务器安装 NFS、autofs 和 quota 工具：

```bash
dnf -y install nfs-utils autofs quota xfsprogs
```

各软件包作用：

| 软件包 | 作用 |
| --- | --- |
| `nfs-utils` | NFS 服务端、客户端、`exportfs` 和挂载工具 |
| `autofs` | 根据访问请求自动挂载并在空闲后卸载 |
| `quota` | 通用 quota 查询工具 |
| `xfsprogs` | 提供 XFS 专用的 `xfs_quota` |

## 4. 在 control 导出 /home

先备份已有配置：

```bash
cp -a /etc/exports /etc/exports.backup-$(date +%F-%H%M%S)
install -d -m 0755 /etc/exports.d
```

新建 `/etc/exports.d/home.exports`：

```exports
/home 192.0.2.20(rw,sync,no_subtree_check,root_squash,sec=sys)
```

选项含义：

| 选项 | 作用 |
| --- | --- |
| `rw` | 允许 compute01 读写 |
| `sync` | 服务端确认数据落盘后再响应，优先保证一致性 |
| `no_subtree_check` | 避免子树检查引起重命名异常和额外开销 |
| `root_squash` | 把客户端 root 映射为匿名身份，防止远端 root 绕过用户权限 |
| `sec=sys` | 使用数字 UID/GID 鉴权，依赖 LDAP/SSSD 一致性 |

客户端地址与左括号之间不能有空格。下面这种写法会改变含义，不要使用：

```text
# 错误示例
/home 192.0.2.20 (rw,sync)
```

加载并启动服务：

```bash
exportfs -rav
systemctl enable --now nfs-server
exportfs -v
systemctl is-active nfs-server
```

如果启用了 firewalld，还需要在内部可信区域开放 NFS。不要为了省事把 NFS 暴露到公网；导出规则和防火墙都应只允许计算节点网络。

## 5. 在 compute01 配置 autofs

建立主映射 `/etc/auto.master.d/home.autofs`：

```text
/home /etc/auto.home --timeout=300
```

建立子映射 `/etc/auto.home`：

```text
* -fstype=nfs4,rw,hard,_netdev,nosuid,nodev control:/home/&
```

这里的 `*` 匹配用户名，`&` 代入同一个名字。例如访问 `/home/alice` 会挂载 `control:/home/alice`。

重要选项：

| 选项 | 作用 |
| --- | --- |
| `hard` | 服务暂时不可用时持续重试，避免应用误以为数据已经写入 |
| `_netdev` | 将它标记为依赖网络的文件系统 |
| `nosuid` | 不允许共享目录中的 SUID/SGID 位提升权限 |
| `nodev` | 不解释共享目录中的设备文件 |

校验映射并启动 autofs：

```bash
automount -m
systemctl enable --now autofs
systemctl is-active autofs
```

`automount -m` 如果显示 SSSD 中没有 `auto.master` 条目，但同时正确列出文件映射 `/etc/auto.home`，不影响本次本地文件映射工作。

## 6. 在根 XFS 文件系统启用用户 quota

先备份并修改 `control` 的 `/etc/fstab`：

```fstab
/dev/mapper/rl-root / xfs defaults,uquota 0 0
```

校验 fstab：

```bash
findmnt --verify --verbose
```

对于独立的 `/home` 文件系统，维护窗口内卸载再挂载即可启用 quota。当前集群的 `/home` 位于根 XFS 上；根文件系统在读取 fstab 前就已挂载，仅修改 fstab 并重启后仍可能显示 `noquota`。因此还需把同一选项加入内核启动参数：

```bash
cp -a /etc/default/grub /etc/default/grub.backup-$(date +%F-%H%M%S)
grubby --update-kernel=ALL --args='rootflags=uquota'
grubby --info=DEFAULT
```

确认默认启动项的 `args` 中已有 `rootflags=uquota`，并确认没有运行中的作业，然后重启：

```bash
test -z "$(squeue -h)"
systemctl reboot
```

重连后必须看到 `usrquota`、Accounting 和 Enforcement 均开启：

```bash
cat /proc/cmdline
findmnt -no SOURCE,FSTYPE,OPTIONS /
xfs_quota -x -c state /
```

本次实际结果：

```text
/dev/mapper/rl-root xfs ... usrquota
User quota state on /
  Accounting: ON
  Enforcement: ON
```

## 7. 创建 LDAP 测试用户

下面示例使用：

```text
用户名：nfstest
UID/GID：21001
家目录：/home/nfstest
Shell：/bin/bash
```

先检查数字 ID 和 DN 未被占用：

```bash
ldapsearch -LLL -Y EXTERNAL \
  -H 'ldapi://%2Frun%2Fslapd-acdiost.socket' \
  -b 'dc=acdiost,dc=internal' \
  '(|(uidNumber=21001)(gidNumber=21001)(uid=nfstest))' dn uid uidNumber gidNumber
```

用 `openssl rand -hex 12` 生成随机测试密码。将同名 `posixGroup` 加到 `ou=Groups`，再将含有 `inetOrgPerson`、`posixAccount` 和 `shadowAccount` 的用户条目加到 `ou=People`。不要把明文密码写入教程、Git 仓库或 shell 历史。

用户条目的关键属性如下：

```ldif
dn: uid=nfstest,ou=People,dc=acdiost,dc=internal
objectClass: inetOrgPerson
objectClass: posixAccount
objectClass: shadowAccount
uid: nfstest
cn: NFS Test User
sn: User
uidNumber: 21001
gidNumber: 21001
homeDirectory: /home/nfstest
loginShell: /bin/bash
userPassword: <随机强密码>
```

新增后刷新两端 SSSD 缓存并核对 ID：

```bash
sss_cache -E
getent passwd nfstest
id nfstest

ssh root@compute01 'sss_cache -E; getent passwd nfstest; id nfstest'
```

两端必须都解析为 `21001:21001`。然后仅在 NFS 服务端创建真正的家目录：

```bash
install -d -o 21001 -g 21001 -m 0700 /home/nfstest
```

## 8. 设置用户 quota

本次给测试用户设置 100 MiB 软限制、120 MiB 硬限制，并设置 inode 限制：

```bash
xfs_quota -x \
  -c 'limit bsoft=100m bhard=120m isoft=10000 ihard=12000 nfstest' /

xfs_quota -x -c 'quota -h nfstest' /
xfs_quota -x -c 'report -h -u' /
```

软限制允许用户在宽限期内临时超额；硬限制会立即拒绝继续分配磁盘块。因为当前 `/home` 与系统目录同处根文件系统，该用户在根 XFS 上拥有的所有文件都会计入配额。生产环境更推荐让 `/home` 使用独立 XFS 文件系统。

## 9. 端到端验收

### 9.1 身份和密码

```bash
getent passwd nfstest
ssh root@compute01 'getent passwd nfstest'

ldapwhoami -x -H ldaps://ldap.acdiost.internal \
  -D 'uid=nfstest,ou=People,dc=acdiost,dc=internal' -W
```

成功时 `ldapwhoami` 返回该用户 DN，且不会提示认证失败。

### 9.2 autofs 和共享写入

在 `compute01`：

```bash
runuser -u nfstest -- bash -c \
  'echo nfs-autofs-ldap-ok > /home/nfstest/end-to-end.txt'

findmnt -T /home/nfstest
runuser -u nfstest -- cat /home/nfstest/end-to-end.txt
```

实际挂载结果应类似：

```text
/home/nfstest control:/home/nfstest nfs4 rw,nosuid,nodev,...
```

回到 `control` 检查相同文件：

```bash
runuser -u nfstest -- cat /home/nfstest/end-to-end.txt
stat -c '%u:%g %a %n' /home/nfstest/end-to-end.txt
```

本次结果为属主 `21001:21001`，说明 LDAP、SSSD、autofs 和 NFS 四层已贯通。客户端 root 无法进入用户的 `0700` 目录，证明 `root_squash` 生效。

### 9.3 quota 强制执行

在 `compute01` 以测试用户写入超过硬限制的文件：

```bash
runuser -u nfstest -- dd if=/dev/zero \
  of=/home/nfstest/quota-fill.bin bs=1M count=130 status=progress
```

预期命令非零退出并出现：

```text
超出磁盘限额
```

测试后清理并确认用量回落：

```bash
runuser -u nfstest -- rm -f /home/nfstest/quota-fill.bin
ssh root@control sync
xfs_quota -x -c 'quota -h nfstest' /
```

## 10. 日常检查

在 `control`：

```bash
systemctl is-active nfs-server
exportfs -v
xfs_quota -x -c state /
xfs_quota -x -c 'report -h -u' /
sinfo -N -o '%N %T'
```

在 `compute01`：

```bash
systemctl is-active autofs sssd slurmd
automount -m
findmnt -t nfs,nfs4
journalctl -u autofs -n 50 --no-pager
```

## 11. 常见故障

### 访问 /home/用户 时卡住

检查 `control` 是否可解析和连通、`nfs-server` 是否运行，以及导出是否授权了 compute01 的实际来源地址：

```bash
getent hosts control
showmount -e control
exportfs -v
journalctl -u nfs-server -n 100 --no-pager
```

### 文件显示为数字属主

这是 SSSD 没有解析到相同 LDAP UID/GID。先比较两端：

```bash
getent passwd <用户名>
id <用户名>
sssctl domain-status acdiost.internal
```

不要用本地 `useradd` 创建一个“同名但 UID 不同”的补丁账户。

### quota 命令有设置但写入不受限

检查挂载选项和 enforcement：

```bash
findmnt -no OPTIONS /
xfs_quota -x -c state /
```

如果仍是 `noquota`，检查默认启动项和实际内核命令行是否都含 `rootflags=uquota`。XFS quota 必须从首次挂载时启用，普通 remount 通常不能补开。

### root 在 compute01 无法读取用户目录

这通常是正确现象：`root_squash` 加上家目录 `0700` 会阻止客户端 root 冒充服务端 root。需要诊断用户文件时，应在 `control` 本机操作，或明确切换为对应用户；不要改成 `no_root_squash`。

## 12. 回滚

回滚前确认没有用户正在使用共享目录：

```bash
squeue -h
findmnt -t nfs,nfs4
```

在 `compute01` 停止 autofs，并恢复或删除本次映射文件：

```bash
systemctl disable --now autofs
rm -f /etc/auto.master.d/home.autofs /etc/auto.home
```

在 `control` 取消本次 `/home` 导出：

```bash
rm -f /etc/exports.d/home.exports
exportfs -rav
systemctl disable --now nfs-server
```

如需关闭 quota，先删除用户限制，再从 fstab 和启动参数中同时移除 quota 选项，并在维护窗口重启：

```bash
xfs_quota -x -c 'limit bsoft=0 bhard=0 isoft=0 ihard=0 nfstest' /
grubby --update-kernel=ALL --remove-args='rootflags=uquota'
```

不要在未确认影响范围时删除 LDAP 用户或其家目录。LDAP 条目删除和 `/home/<用户>` 数据删除是两个独立且有破坏性的动作。

## 相关文档

- [[ldap-389ds-single-master-configuration|从零部署 389 Directory Server 单主 LDAP]]
- [[ldap-sssd-client-and-login-troubleshooting|从零配置 LDAP 客户端、SSSD 与 Linux 登录]]
- [[slurm-accounting-with-mysql-8-4|从零部署 MySQL 8.4 与 Slurm 记账]]
- [[hpc-cluster-validation-and-rollback|集群验收、备份与回滚手册]]
