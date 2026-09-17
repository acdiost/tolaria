---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
tags:
  - HPC
  - Runbook
  - Backup
---

# 集群验收、备份与回滚手册

每次修改 MySQL、Slurm 或 LDAP 后，都按“配置语法—服务状态—认证—最小作业—记账”顺序验收。只检查 `systemctl active` 不足以证明链路可用。

## 变更前备份

```bash
stamp=$(date +%Y%m%d-%H%M%S)
cp -a /etc/slurm/slurm.conf /etc/slurm/slurm.conf.backup-$stamp
cp -a /etc/slurm/slurmdbd.conf /etc/slurm/slurmdbd.conf.backup-$stamp
cp -a /etc/sssd/sssd.conf /etc/sssd/sssd.conf.backup-$stamp
```

LDAP 使用在线备份：

```bash
dsctl acdiost db2bak
dsctl acdiost backups
```

MySQL 使用一致性备份；凭据从受限选项文件读取，不直接放入命令：

```bash
mysqldump --defaults-extra-file=/root/.my.cnf \
  --single-transaction --routines --events \
  slurm_acct_db > /secure-backup/slurm_acct_db-$stamp.sql
```

备份目录应限制权限，并复制到另一台存储设备。

## 验收清单

### 服务

```bash
systemctl is-active mysqld slurmdbd slurmctld dirsrv@acdiost sssd
ssh compute01 systemctl is-active slurmd sssd
```

### Slurm 节点与 GPU

```bash
sinfo -N -l
scontrol show node control -o
scontrol show node compute01 -o
srun -N1 -w control --gres=gpu:grid_v100d-1q:1 nvidia-smi -L
srun -N1 -w compute01 --gres=gpu:grid_v100d-1q:1 nvidia-smi -L
```

### 记账

```bash
sacctmgr list cluster
sacctmgr show assoc
sacct -S today -X -o JobID,JobName,User,State,NodeList,AllocTRES
```

### LDAP 和 TLS

```bash
dsconf acdiost config get \
  nsslapd-allow-anonymous-access \
  nsslapd-require-secure-binds

openssl s_client -brief \
  -connect ldap.acdiost.internal:636 \
  -servername ldap.acdiost.internal \
  -CAfile /etc/openldap/certs/acdiost-ca.crt </dev/null

sssctl domain-status acdiost.internal
ssh compute01 sssctl domain-status acdiost.internal
```

匿名查询必须失败；管理员和 SSSD 服务账号的加密绑定必须成功。

## 回滚原则

1. 一次只回滚一个组件。
2. 先恢复配置文件，再验证权限和 SELinux 标签。
3. 按依赖顺序重启：数据库 → `slurmdbd` → `slurmctld` → `slurmd`。
4. LDAP 配置回滚后，重新验证匿名访问、TLS 和 SSSD 绑定，不能只看进程状态。
5. 恢复数据库前先停止写入，并保留失败现场的备份。

恢复普通配置文件示例：

```bash
cp -a /etc/slurm/slurm.conf.backup-<时间戳> /etc/slurm/slurm.conf
restorecon -v /etc/slurm/slurm.conf
slurmctld -t
systemctl restart slurmctld
```

LDAP 数据库恢复属于破坏性操作，必须先确认备份名、维护窗口和恢复目标，再停止实例并执行 `bak2db`。不要在目录仍有写入时直接恢复。

## 已知非本次故障

两台主机的 `mcelog.service` 当前为 failed。它不影响本次 LDAP、SSSD 或 Slurm 验收，但应单独确认 CPU 平台是否支持 `mcelog`；不要把它与 Slurm 节点 `INVAL` 混为一谈。
