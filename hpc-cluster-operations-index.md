---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
tags:
  - HPC
  - Slurm
  - LDAP
  - MySQL
---
# HPC 集群运维文档索引

这组文档记录 `acdiost.internal` 集群在 2026-08-25 完成的 Slurm 记账、节点 `INVAL` 修复、LDAP/SSSD 部署，以及 NFS/autofs 共享家目录与 XFS quota。

## 当前拓扑

| 角色 | 主机名 | 地址 | 关键服务 |
| --- | --- | --- | --- |
| 控制节点 | `control` | `192.0.2.10` | `slurmctld`、`slurmdbd`、MySQL、389 DS、SSSD、NFS、XFS quota |
| 计算节点 | `compute01` | `192.0.2.20` | `slurmd`、SSSD、autofs、NFS 客户端 |

## 文档导航

- [[slurm-accounting-with-mysql-8-4|从零部署 MySQL 8.4 与 Slurm 记账]]
- [[slurm-node-inval-troubleshooting|Slurm 节点 INVAL 排障与修复]]
- [[ldap-389ds-single-master-configuration|从零部署 389 Directory Server 单主 LDAP]]
- [[ldap-sssd-client-and-login-troubleshooting|从零配置 LDAP 客户端、SSSD 与 Linux 登录]]
- [[nfs-autofs-xfs-quota-shared-home|从零部署 NFS、autofs 与 XFS quota 共享家目录]]
- [[hpc-cluster-validation-and-rollback|集群验收、备份与回滚手册]]
- [[hpc-software-stack-lmod-python-uv-cuda-hpcx|使用 Lmod 管理 HPC-X、uv、多版本 Python、CUDA 与科学计算库]]

## 已验证状态

```
MySQL       8.4.11
Slurm       25.11.4
389 DS      2.8.0
SSSD        2.9.8
NFS         4.2（compute01 按用户自动挂载）
XFS quota   用户计费与强制执行已开启
control     idle, 1 × grid_v100d-1q
compute01          idle, 2 × grid_v100d-1q
```

已完成的验收作业：

```
1  codex-gpu-test       COMPLETED  control-old（更名前）
2  codex-compute01-gpu-test    COMPLETED  compute01
3  ldap-identity-test   COMPLETED  compute01
```

## 日常快速检查

```bash
systemctl is-active mysqld slurmdbd slurmctld dirsrv@acdiost sssd
systemctl is-active nfs-server
exportfs -v
xfs_quota -x -c state /
sinfo -N -l
sacct -S today -X -o JobID,JobName,User,State,NodeList
sssctl domain-status acdiost.internal
```

在 `compute01` 上检查：

```bash
systemctl is-active slurmd sssd autofs
findmnt -t nfs,nfs4
scontrol show node compute01 -o
```
