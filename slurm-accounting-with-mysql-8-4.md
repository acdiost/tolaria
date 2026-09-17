---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
tags:
  - Slurm
  - MySQL
  - Accounting
  - Tutorial
---
# 从零部署 MySQL 8.4 与 Slurm 记账

完成本文后，Slurm 会把作业、作业步、CPU、内存和 GPU 使用记录通过 `slurmdbd` 写入 MySQL。管理员可以用 `sacct` 查历史作业，用 `sacctmgr` 管理账户、用户、QOS 和资源限制。

本文以当前集群为例：

```
系统          Rocky Linux 9.4，兼容 RHEL 9
控制节点      control / 192.0.2.10
计算节点      compute01 / 192.0.2.20
Slurm         25.11.4，OpenHPC RPM
数据库        MySQL Community Server 8.4 LTS
数据库位置    与 slurmdbd 同机
数据库名      slurm_acct_db
```

所有命令默认由 `root` 执行。`<...>` 是必须替换的变量，不是可以直接用于生产的值。

## 1. 先理解组件关系

```
sacct / sacctmgr / slurmctld
             │
             │ MUNGE 认证，TCP 6819
             ▼
          slurmdbd
             │
             │ MySQL 用户名和密码，TCP 3306
             ▼
 MySQL 8.4 / slurm_acct_db
```

关键点：

- `slurm.conf` 使用 `accounting_storage/slurmdbd`。
- `slurmdbd.conf` 使用 `accounting_storage/mysql`。
- 数据库密码只存在于权限为 `600` 的 `slurmdbd.conf`，不应分发给普通用户。
- Slurm 客户端不直接连接 MySQL。
- Slurm 节点、控制器和 `slurmdbd` 之间使用 MUNGE 认证。

SchedMD 推荐使用 SlurmDBD，而不是让 Slurm 命令直接连接数据库。这样可以集中保护凭据，并在数据库短暂中断时缓存部分记录。

## 2. 系统环境准备

### 2.1 确认操作系统、主机名和时间

```bash
cat /etc/os-release
hostnamectl --static
timedatectl
chronyc tracking
```

作用：

- `cat /etc/os-release`：确认使用 EL9 软件仓库。
- `hostnamectl`：Slurm 对主机名一致性敏感，控制节点应返回 `control`。
- `timedatectl` 和 `chronyc`：防止作业开始、结束和记账时间错乱。

两台节点应能解析彼此：

```
192.0.2.10 control
192.0.2.20 compute01
```

验证：

```bash
getent ahostsv4 control
getent ahostsv4 compute01
```

### 2.2 检查资源

```bash
free -h
df -hT /var/lib/mysql
lsblk -f
```

当前服务器有约 16 GiB 内存和 198 GiB 根文件系统。MySQL 与 Slurm 控制服务共机，因此缓冲池设置为 4 GiB，而不是把大部分内存都交给数据库。

生产环境应为 `/var/lib/mysql` 准备可靠存储，并监控剩余空间。数据库磁盘写满会同时影响作业记账和 Slurm 策略执行。

### 2.3 检查冲突软件

```bash
rpm -qa | grep -Ei 'mysql|mariadb'
```

不要在未做备份和迁移评估时，用 Oracle MySQL RPM 直接覆盖系统已有的 MariaDB。全新主机才按下面步骤安装。

## 3. 安装 MySQL 8.4 LTS

### 3.1 添加官方 Yum 仓库

从 [MySQL Yum Repository](https://dev.mysql.com/downloads/repo/yum/) 下载当前 EL9 仓库 RPM。文件名形式为：

```
mysql84-community-release-el9-<仓库版本>.noarch.rpm
```

安装并检查仓库：

```bash
dnf localinstall ./mysql84-community-release-el9-<仓库版本>.noarch.rpm
dnf repolist enabled | grep mysql

dnf config-manager --disable mysql-9.7-lts-community
dnf config-manager --disable mysql-tools-9.7-lts-community

dnf config-manager --enable mysql-8.4-lts-community
dnf config-manager --enable mysql-tools-8.4-lts-community
```

预期至少看到：

```
mysql-8.4-lts-community
mysql-tools-8.4-lts-community
```

作用：仓库 RPM 只安装 Yum/DNF 仓库定义和签名密钥，不会立即安装数据库。仓库配置包会更新，因此不要在文档中固定一个可能过期的下载版本号。

### 3.2 安装服务端

```bash
dnf install mysql-community-server
rpm -q mysql-community-server mysql-community-client
```

`mysql-community-server` 会同时安装服务端、客户端、公共字符集和客户端库。

### 3.3 首次启动

```bash
systemctl enable --now mysqld
systemctl status mysqld --no-pager
mysql --version
```

首次启动会初始化 `/var/lib/mysql`、生成 TLS 文件并创建 `root@localhost`。临时密码写入错误日志：

```bash
grep 'temporary password' /var/log/mysqld.log
mysql -uroot -p
```

登录后立即修改 root 密码：

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'k77_edHblpJg';
```

不要把临时密码或新密码写入脚本、Git、Shell 命令参数或运维文档。完成后可运行交互式加固工具：

```bash
mysql_secure_installation
```

它用于删除匿名数据库用户、测试数据库和不需要的远程 root 登录。

## 4. 为 Slurm 调整 MySQL

先备份：

```bash
stamp=$(date +%Y%m%d-%H%M%S)
cp -a /etc/my.cnf /etc/my.cnf.backup-$stamp
```

在 `/etc/my.cnf` 的 `[mysqld]` 段加入：

```ini
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid

# slurmdbd 与 MySQL 同机，只监听回环地址最安全
bind-address=127.0.0.1
mysqlx-bind-address=127.0.0.1
skip_name_resolve=ON

# Slurm 依赖事务和回滚
default_storage_engine=InnoDB
innodb_file_per_table=ON

# 当前 16 GiB 共机环境的起始值
innodb_buffer_pool_size=4G
innodb_redo_log_capacity=1G
innodb_lock_wait_timeout=900
max_allowed_packet=64M

# 基础安全与日志保留
binlog_expire_logs_seconds=604800
local_infile=OFF
```

参数作用：

| 参数 | 作用 |
| --- | --- |
| `bind-address` | 限制经典 MySQL 协议监听地址。共机部署无需对外开放 3306。 |
| `skip_name_resolve` | 不对客户端 IP 做反向 DNS，授权表中的 Host 应使用 IP。 |
| `default_storage_engine=InnoDB` | 提供事务、行锁和崩溃恢复。Slurm 需要回滚能力。 |
| `innodb_buffer_pool_size` | 缓存数据页和索引。共享主机不要盲目设置为总内存的 70%。 |
| `innodb_redo_log_capacity` | MySQL 8.0.30+ 的 redo 总容量，减少高写入时的小批次刷盘。 |
| `innodb_lock_wait_timeout=900` | 给 Slurm 升级、清理和复杂事务更多等待时间。 |
| `max_allowed_packet=64M` | 容纳较大的作业环境或脚本记录。 |
| `local_infile=OFF` | 默认禁止客户端请求服务器读取本地文件。 |

检查配置并重启：

```bash
my_print_defaults mysqld
systemctl restart mysqld
systemctl is-active mysqld
ss -lntp | grep 3306
```

验证运行值：

```sql
SELECT
  @@version,
  @@hostname,
  @@bind_address,
  @@innodb_buffer_pool_size,
  @@innodb_redo_log_capacity,
  @@innodb_lock_wait_timeout,
  @@max_allowed_packet,
  @@innodb_default_row_format;
```

如果数据库与 `slurmdbd` 分开部署，才把 `bind-address` 改为数据库内网地址，并通过 firewalld 或安全组只允许 `slurmdbd` 主机访问 3306。不要用 `0.0.0.0` 再配一个完全开放的防火墙。

> 当前实机的 `@@bind_address` 仍为 `0.0.0.0`；上面的 `127.0.0.1` 是从零部署的推荐安全基线。修改现网监听地址会造成数据库短暂重启，应作为单独变更执行并重新验证 `slurmdbd`。

## 5. 创建数据库和最小权限用户

进入 MySQL：

```bash
mysql -uroot -p
```

执行：

```sql
CREATE DATABASE slurm_acct_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

CREATE DATABASE slurm_jobcomp_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

CREATE USER 'slurm'@'127.0.0.1'
  IDENTIFIED BY 'S1urm_2026';

GRANT ALL PRIVILEGES ON slurm_acct_db.*
  TO 'slurm'@'127.0.0.1';

SHOW GRANTS FOR 'slurm'@'127.0.0.1';

---

CREATE USER 'slurmjobcomp'@'localhost'
IDENTIFIED BY 'S1urmj@bc0mp';

GRANT SELECT, INSERT, UPDATE, DELETE
ON `slurm_jobcomp_db`.*
TO 'slurmjobcomp'@'localhost';

SHOW GRANTS FOR 'slurmjobcomp'@'localhost';

---

CREATE USER 'phadagent'@'%'
IDENTIFIED BY 'PhadAgent_2026';

GRANT SELECT, SHOW VIEW
ON `slurm_acct_db`.*
TO 'phadagent'@'%';

GRANT SELECT, SHOW VIEW
ON `slurm_jobcomp_db`.*
TO 'phadagent'@'%';

SHOW GRANTS FOR 'phadagent'@'%';
```

作用：

- 使用独立数据库，方便备份、恢复和权限审计。
- 使用大小写不敏感的 `_ci` 排序规则；Slurm 不支持大小写敏感的数据库列排序规则。
- 数据库用户只拥有 `slurm_acct_db.*`，不拥有全局管理权限。
- `skip_name_resolve=ON` 时使用 `127.0.0.1`，不要写主机名。

验证专用账号：

```bash
mysql -h 127.0.0.1 -u slurm -p slurm_acct_db
```

密码应由密码管理器生成并保存。后面的 `StoragePass` 必须与这里一致。

## 6. 准备 SlurmDBD

当前集群使用 OpenHPC 包：

```bash
dnf install slurm-ohpc slurm-slurmctld-ohpc slurm-slurmdbd-ohpc mariadb-connector-c munge
```

这里出现 `mariadb-connector-c` 是正常的：Slurm 的 MySQL 存储插件可以通过 MariaDB Connector/C 客户端库连接 MySQL 8.4。

确认插件：

```bash
test -f /usr/lib64/slurm/accounting_storage_mysql.so
ldd /usr/lib64/slurm/accounting_storage_mysql.so | grep -Ei 'mysql|maria'
```

Slurm 组件必须共享有效的 MUNGE 信任关系：

```bash
systemctl enable --now munge
munge -n | unmunge
id slurm
```

参与通信的主机上，`slurm` 用户名称和 UID 应保持一致。

## 7. 配置 slurmdbd.conf

创建 `/etc/slurm/slurmdbd.conf`：

```ini
AuthType=auth/munge
SlurmUser=slurm

DbdHost=control
DbdPort=6819

StorageType=accounting_storage/mysql
StorageHost=127.0.0.1
StoragePort=3306
StorageLoc=slurm_acct_db
StorageUser=slurm
StoragePass=<与 MySQL 用户一致的密码>
```

配置作用：

- `AuthType=auth/munge`：Slurm 客户端和控制器连接 DBD 时使用 MUNGE。
- `DbdHost`/`DbdPort`：`slurmdbd` 自己的监听身份和端口。
- `StorageType=accounting_storage/mysql`：只有此文件连接 MySQL。
- `StorageLoc`：目标数据库名。
- `StorageUser`/`StoragePass`：第五节创建的数据库账号。

保护文件：

```bash
chown slurm:slurm /etc/slurm/slurmdbd.conf
chmod 600 /etc/slurm/slurmdbd.conf
restorecon -v /etc/slurm/slurmdbd.conf
```

`slurmdbd.conf` 含明文数据库密码。不要把文件内容完整贴到聊天、工单或日志中。

## 8. 配置 slurm.conf

在控制器使用的 `/etc/slurm/slurm.conf` 加入：

```ini
ClusterName=cluster
SlurmctldHost=control

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=127.0.0.1
AccountingStoragePort=6819

JobAcctGatherType=jobacct_gather/cgroup
JobAcctGatherFrequency=30
```

首次部署时先不要立即启用严格的 `AccountingStorageEnforce`。先建立 cluster、account 和 user association，避免所有用户因没有 association 而无法提交作业。

配置完成后再加入：

```ini
AccountingStorageEnforce=associations,limits,qos
```

含义：

- `associations`：用户必须属于有效的 cluster/account association。
- `limits`：执行 account 和 QOS 限额。
- `qos`：要求作业使用用户被授权的 QOS。

## 9. 先启动数据库层

```bash
systemctl enable --now mysqld
systemctl enable --now slurmdbd
```

检查端口和日志：

```bash
ss -lntp | grep -E ':(3306|6819)[[:space:]]'
systemctl status mysqld slurmdbd --no-pager
journalctl -u slurmdbd -b --no-pager | tail -n 150
```

如果 3306 只监听 `127.0.0.1`，这是共机架构的正确结果。6819 需要能被 Slurm 控制器和管理客户端按设计访问。

## 10. 初始化记账对象，再启动控制器

查看 cluster：

```bash
sacctmgr list cluster
```

不存在时创建：

```bash
sacctmgr add cluster cluster
sacctmgr add account hpc Description='HPC users' Organization='acdiost'
sacctmgr add user <用户名> Account=hpc Cluster=cluster
```

检查：

```bash
sacctmgr show cluster
sacctmgr show account withassoc
sacctmgr show user withassoc
```

确认 cluster 和 association 完整后，启动控制器：

```bash
systemctl enable --now slurmctld
systemctl status slurmctld --no-pager
```

然后在 `slurm.conf` 启用 `AccountingStorageEnforce` 并重新加载：

```bash
scontrol reconfigure
```

## 11. 端到端验收

### 11.1 服务和节点

```bash
systemctl is-active mysqld slurmdbd slurmctld
sinfo -N -l
```

### 11.2 提交作业

```bash
srun --job-name=accounting-smoke-test -N1 -w control /bin/hostname -s
```

### 11.3 查询记账

```bash
sacct -S today -X -o JobID,JobName,User,Account,State,ExitCode,NodeList,AllocTRES
```

成功标准：作业状态为 `COMPLETED`、退出码为 `0:0`，节点名和 TRES 正确。

当前集群已验证的记录包括：

```
control-rename-gpu-test  COMPLETED  control
compute01-after-control-rename COMPLETED  compute01
```

## 12. 常见错误

### `slurmdbd: StorageType plugin not found`

```bash
ls -l /usr/lib64/slurm/accounting_storage_mysql.so
scontrol show config | grep PluginDir
ldd /usr/lib64/slurm/accounting_storage_mysql.so
```

通常是 SlurmDBD 包、插件目录或数据库客户端库缺失。

### `Access denied for user 'slurm'`

检查四项是否完全一致：

```
MySQL 用户名
授权来源地址
slurmdbd.conf 的 StorageUser
slurmdbd.conf 的 StoragePass
```

`skip_name_resolve=ON` 时，`'slurm'@'localhost'` 与 `'slurm'@'127.0.0.1'` 不是同一个授权记录。

### 6819 `Connection refused`

```bash
systemctl status slurmdbd --no-pager
ss -lntp | grep 6819
journalctl -u slurmdbd -b --no-pager | tail -n 100
```

服务刚重启时，进程显示 `active` 到端口真正开始监听之间可能有短暂延迟。验收脚本应检查端口和实际 `sacctmgr` 查询，而不是只检查 systemd 状态。

### 用户提交时报 `Invalid account`

```bash
sacctmgr show user name=<用户名> withassoc
sacctmgr show assoc cluster=cluster account=<账户名>
```

这通常不是 Linux 用户密码问题，而是缺少 Slurm association，或者已经启用严格的 `AccountingStorageEnforce`。

## 13. 备份与维护

一致性备份：

```bash
stamp=$(date +%Y%m%d-%H%M%S)
mysqldump --defaults-extra-file=/root/.my.cnf --single-transaction --routines --events slurm_acct_db > /secure-backup/slurm_acct_db-$stamp.sql
```

注意：

- `/root/.my.cnf` 应为 `600`，避免把密码放在命令参数。
- 备份必须复制到另一台存储设备。
- 升级 MySQL、MariaDB 或 Slurm 前先备份，并分别升级，避免同时切换多个变量。
- 尽早规划 SlurmDBD 的 `Archive*` 和 `Purge*After`，不要等数据库过大后才清理。

## 14. MariaDB 替代说明

Slurm 也支持 MariaDB，`StorageType` 仍写：

```ini
StorageType=accounting_storage/mysql
```

MariaDB 新版本还应确认：

```ini
innodb_snapshot_isolation=OFF
```

不要把 MySQL 数据目录直接交给 MariaDB 打开，也不要仅替换 RPM 后直接启动。数据库产品切换必须采用受支持的逻辑备份、恢复和兼容性验证流程。

## 参考资料

- [MySQL 8.4：使用 Yum 仓库安装](https://dev.mysql.com/doc/refman/8.4/en/linux-installation-yum-repo.html)
- [MySQL 8.4：RPM 安装布局](https://dev.mysql.com/doc/refman/8.4/en/linux-installation-rpm.html)
- [MySQL 8.4：Server System Variables](https://dev.mysql.com/doc/refman/8.4/en/server-system-variables.html)
- [SchedMD：Accounting and Resource Limits](https://slurm.schedmd.com/accounting.html)
- [SchedMD：slurmdbd.conf](https://slurm.schedmd.com/slurmdbd.conf.html)
