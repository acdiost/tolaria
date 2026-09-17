---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
tags:
  - HPC
  - Slurm
  - GPU
  - Deployment
---
# Slurm 26.05.4 GPU 集群部署总结

## 1. 部署概况

- 部署日期：2026-09-11
- Slurm 版本：26.05.4
- 集群名称：`cluster`
- 控制节点：`control.acdiost.internal`
- 控制节点管理地址：`198.51.100.10`
- 控制节点集群通信地址：`203.0.113.10`
- 控制节点操作系统：Rocky Linux 9.4（x86_64）
- 首个预登记计算节点：`gpu-node02.acdiost.internal`
- 预登记计算节点地址：`203.0.113.30`
- 预登记 GPU：8 × NVIDIA GeForce RTX 5090
- 默认分区：`gpu5090`
- 安装方式：源码编译安装
- 默认安装前缀：`/usr/local`
- 配置目录：`/etc/slurm`

本阶段已完成控制节点初始化，并已在 `gpu-node02` 上完成 Slurm 26.05.4 编译、MUNGE 密钥同步、configless 配置和 `slurmd` 启动。节点已注册为 `IDLE`，CPU 与单 GPU 作业均验证通过。

## 2. 角色与系统差异

集群各角色允许使用不同 Linux 发行版，但必须使用相同 Slurm 版本，并在各操作系统上分别编译，不能跨系统复制二进制文件。

| 角色 | 操作系统 | 主要功能 |
| --- | --- | --- |
| 控制节点 | Rocky Linux 9.4 | `slurmctld`、`slurmdbd`、`slurmrestd`、Lua、JWT |
| 计算节点 | Ubuntu 24.04 | `slurmd`、cgroup v2、NVML、PMIx、PAM adopt |
| 登录节点 | Ubuntu 24.04 | Slurm 客户端、MUNGE、PMIx，可选 `sackd` |

控制节点没有 GPU，因此未编译 NVML 和 PMIx，不影响 GPU 资源调度。GPU 自动发现由计算节点上的 `slurmd` 和 NVML 插件完成。

## 3. 控制节点编译参数

控制节点实际采用以下参数：

```bash
./configure \
  --sysconfdir=/etc/slurm \
  --with-systemdsystemunitdir=/usr/lib/systemd/system \
  --with-munge=/usr \
  --with-hwloc=/usr \
  --with-lua \
  --enable-slurmrestd \
  --with-jwt=/usr \
  --with-yaml=/usr \
  --with-libcurl=/usr \
  --with-libhttp-parser=/usr \
  --with-llhttp-parser=/usr \
  --disable-debug
```

由于没有指定 `--prefix`，Slurm 使用默认 `/usr/local`：

```
/usr/local/bin
/usr/local/sbin
/usr/local/lib 或 /usr/local/lib64
```

配置检测结果：

- Lua：启用
- `job_submit/lua`：启用
- `slurmrestd`：启用
- JSON-C、JWT、YAML：启用
- libhttp-parser、llhttp：启用
- MUNGE、hwloc：启用
- MySQL 8.4.11：可用
- NVML：未启用，控制节点不需要
- PMIx：未启用，控制节点不执行 MPI 作业

## 4. 已安装的控制节点配置

当前配置文件：

```
/etc/slurm/slurm.conf
/etc/slurm/slurmdbd.conf
/etc/slurm/node.conf
/etc/slurm/partition.conf
/etc/slurm/cgroup.conf
/etc/slurm/gres.conf
/etc/slurm/job_submit.lua
/etc/default/slurmrestd
/etc/tmpfiles.d/slurm.conf
```

初始化前的旧配置已备份到：

```
/etc/slurm/backup-20260911-initial/slurm.conf
```

### 4.1 核心调度配置

关键配置如下：

```ini
ClusterName=cluster
SlurmctldHost=control(203.0.113.10)
SlurmUser=slurm

AuthType=auth/munge
CredType=cred/munge
AuthAltTypes=auth/jwt

SlurmctldParameters=enable_configless
PrologFlags=Contain
ProctrackType=proctrack/cgroup
TaskPlugin=task/affinity,task/cgroup

SchedulerType=sched/backfill
SelectType=select/cons_tres
SelectTypeParameters=CR_Core_Memory

GresTypes=gpu
AccountingStorageTRES=gres/gpu
JobAcctGatherType=jobacct_gather/cgroup
JobAcctGatherFrequency=30

AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=control
AccountingStoreFlags=job_comment

JobSubmitPlugins=lua
```

### 4.2 首个计算节点定义

`/etc/slurm/node.conf`：

```ini
NodeName=gpu-node02 NodeAddr=203.0.113.30 CPUs=128 Boards=1 SocketsPerBoard=4 CoresPerSocket=16 ThreadsPerCore=2 RealMemory=500000 Gres=gpu:rtx_5090:8 Parameters=numa_node_as_socket State=UNKNOWN
```

`/etc/slurm/partition.conf`：

```ini
PartitionName=gpu5090 Nodes=gpu-node02 Default=YES MaxTime=INFINITE State=UP
```

`/etc/slurm/gres.conf`：

```ini
NodeName=gpu-node02 AutoDetect=nvml
```

`/etc/slurm/cgroup.conf`：

```ini
CgroupPlugin=autodetect
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes
```

其中 `ConstrainDevices=yes` 用于隔离 GPU 设备访问。

### 4.3 Lua 作业提交插件

已创建最小透传脚本 `/etc/slurm/job_submit.lua`。当前脚本不修改或拒绝作业，只用于确认 Lua 插件能够正常加载。后续可在该脚本中增加分区、时限、账户或 GPU 请求规则。

## 5. MUNGE 初始化

控制节点最初只有 `munge-libs` 和 `munge-devel`，本次补充安装了 MUNGE 运行包，并完成：

- 创建 `munge` 系统账户；
- 生成 `/etc/munge/munge.key`；
- 设置密钥属主为 `munge:munge`；
- 设置密钥权限为 `0400`；
- 启用并启动 `munge.service`；
- 完成 `munge | unmunge` 编解码验证。

计算节点和登录节点必须使用与控制节点完全相同的 `munge.key`，并保持安全权限。

## 6. SlurmDBD 与 MySQL

- MySQL 地址：`127.0.0.1`
- MySQL 端口：`3307`
- Accounting 数据库：`slurm_acct_db`
- 数据库账户：`slurm@127.0.0.1`
- 密码：不记录在本文档中
- SlurmDBD 端口：`6819`

数据库账号已验证具备 `slurm_acct_db.*` 全部权限。`slurmdbd` 成功连接 MySQL 8.4.11 并初始化数据库表。

已登记的 accounting 对象：

```typescript
Cluster: cluster
Account: default
User: root
AdminLevel: Administrator
```

已创建 GPU TRES：

```
gres/gpu
gres/gpumem
gres/gpuutil
```

`/etc/slurm/slurmdbd.conf` 权限为 `0600`，属主为 `slurm:slurm`。

## 7. JWT 与 slurmrestd

JWT 密钥位置：

```javascript
/var/spool/slurmctld/jwt_hs256.key
```

权限为 `0600`，属主为 `slurm:slurm`。该密钥不得下发到计算节点或普通登录节点。

`slurmrestd` 使用独立的非特权账户运行，当前配置：

```
认证插件：jwt
数据解析器：v0.0.45
后端：slurmctld,slurmdbd
监听地址：127.0.0.1:6820
```

已使用短期 JWT 完成真实 API 验证：

```
GET /slurm/v0.0.45/ping   -> HTTP 200
GET /slurmdb/v0.0.45/ping -> HTTP 200
```

REST 服务目前不直接暴露到管理网络。若需要外部访问，建议在前端配置 HTTPS 反向代理、访问控制和审计，而不是直接把明文 HTTP 端口绑定到所有接口。

## 8. 当前服务状态

初始化完成时：

```
munge.service       enabled / active
slurmdbd.service    enabled / active
slurmctld.service   enabled / active
slurmrestd.service  enabled / active
```

控制器验证：

```
Slurmctld(primary) at control is UP
```

节点状态：

```
NODELIST  PARTITION  STATE  CPUS  MEMORY  GRES
gpu-node02   gpu5090    idle   128   500000  gpu:rtx_5090:8
```

## 9. 计算节点编译与下一步

`gpu-node02` 运行 Ubuntu 24.04、cgroup v2，具有 128 个逻辑 CPU、约 512 GB 内存和 8 张 RTX 5090。Slurm 26.05.4 已安装到默认前缀 `/usr/local`。

首次配置失败的原因是将 Rocky/RHEL 的 PAM 路径 `/usr/lib64/security` 用到了 Ubuntu。Ubuntu 24.04 x86_64 的正确路径是 `/usr/lib/x86_64-linux-gnu/security`；同时节点缺少 NUMA 和 PAM 开发库。`ptrace64... no` 是 Linux 上的正常检测结果，不是失败原因。

实际安装的编译依赖：

```bash
apt-get install -y \
  build-essential pkg-config \
  libnuma-dev libpam0g-dev libhwloc-dev \
  libmunge-dev libdbus-1-dev libpmix-dev
```

计算节点实际成功的配置参数：

```bash
./configure \
  --sysconfdir=/etc/slurm \
  --with-systemdsystemunitdir=/usr/lib/systemd/system \
  --enable-cgroupv2 \
  --with-munge=/usr \
  --with-hwloc=/usr \
  --with-nvml=/usr/local/cuda-12.8 \
  --with-pmix=/usr/lib/x86_64-linux-gnu/pmix2 \
  --with-pam_dir=/usr/lib/x86_64-linux-gnu/security \
  --disable-debug
```

NUMA 由 `configure` 自动检测，检测结果为 `numa_available in -lnuma... yes`。NVML 头文件位于 CUDA 12.8 目录，PMIx 使用 Ubuntu 多架构包的实际前缀。计算节点不运行 `slurmrestd`，Lua 作业提交插件也由控制节点执行。

编译和安装：

```bash
make -j32
make install
ldconfig
```

PAM 组件位于 `contribs`，需要单独安装：

```bash
make -C contribs/pam -j32
make -C contribs/pam install
make -C contribs/pam_slurm_adopt -j32
make -C contribs/pam_slurm_adopt install
```

已验证：

- `/usr/local/sbin/slurmd -V` 输出 `slurm 26.05.4`；
- `slurmd -C --parameters=numa_node_as_socket` 识别 128 CPU、4 个 NUMA 调度 socket、16 core/socket、2 thread/core；
- NVML 自动识别 `gpu:nvidia_geforce_rtx_5090:8`；
- `gpu_nvml.so`、`cgroup_v2.so`、`mpi_pmix_v5.so` 已安装；
- `pam_slurm.so` 和 `pam_slurm_adopt.so` 已安装到 Ubuntu PAM 目录。

已完成的计算节点初始化：

1. 确认 `slurm` UID/GID 与控制节点一致，均为 202/202；
2. 安装并启动 MUNGE，同步 `/etc/munge/munge.key`，权限为 `0400 munge:munge`；
3. 创建 `/var/spool/slurmd` 和 `/var/log/slurm`，属主为 `slurm:slurm`；
4. 采用 configless 模式，不在计算节点维护静态 `slurm.conf`；
5. `munge.service` 与 `slurmd.service` 均为 `enabled / active`；
6. 将 IB 网段 `203.0.113.0/24` 永久加入控制节点 firewalld `trusted` 区域；
7. 控制器和计算节点通信均使用 IB 地址。

建议的 `/etc/default/slurmd`：

```bash
SLURMD_OPTIONS="--conf-server=203.0.113.10:6817"
```

首次启动时 NVML 检测到 GPU 的 CPU affinity 按 4 个 NUMA 节点分布，与原 2 socket 调度边界不一致，节点因此进入 `INVALID_REG+DRAIN`。根据 Slurm 26.05 Socket Affinity 文档，对该节点启用 `Parameters=numa_node_as_socket`，并改为 4 socket × 16 core × 2 thread 的调度拓扑。这不改变 2 个物理 CPU 和 128 个逻辑 CPU 的实际硬件数量。

端到端验证结果：

```typescript
CPU job: gpu-node02.acdiost.internal
GPU job: CUDA_VISIBLE_DEVICES=0
GPU visible in job: 1 x NVIDIA GeForce RTX 5090
Final node state: IDLE
```

单 GPU 作业中 `nvidia-smi -L` 只能看到已分配的 GPU，证明 `ConstrainDevices=yes` 和 GPU cgroup 隔离已生效。

## 10. 常用检查命令

```bash
systemctl status munge slurmdbd slurmctld slurmrestd
scontrol ping
scontrol show config
sinfo -N -l
sacctmgr show cluster
sacctmgr show account
sacctmgr show user
sacctmgr show tres
ss -lntp | grep -E '6817|6819|6820'
journalctl -u slurmctld -u slurmdbd -u slurmrestd --since today
```

计算节点加入后：

```bash
slurmd -C
slurmd -G
scontrol show node gpu-node02
srun -N1 -w gpu-node02 hostname
srun -N1 -w gpu-node02 --gres=gpu:1 nvidia-smi -L
```

## 11. 当前待办事项

- 初始化登录节点；
- 按实际硬件逐批加入其余 GPU 节点；
- 根据实际用户和项目建立 Slurm account、QOS 与关联；
- 外部开放 REST API 前部署 HTTPS 和访问控制。

## 12. Ansible 计算节点添加 Playbook

控制节点已创建：

```
/etc/ansible/playbook/add_compute_node.yml
/etc/ansible/playbook/README.md
```

Playbook 适用于 inventory 中已定义 `ansible_host` 和 `ib_address` 的 Ubuntu/Debian x86_64 NVIDIA GPU 计算节点。它会自动完成依赖安装、本地源码编译、PAM 模块安装、MUNGE 密钥同步、IB configless 环境配置以及 CPU/NUMA/GPU 检测，然后打印建议的 `node.conf`、`gres.conf` 和 `partition.conf` 配置行。

2026-09-14 安全调整：Playbook 已移除对控制节点配置文件的回写、`scontrol reconfigure`、节点 RESUME、`slurmd` 自动启动和作业测试。管理员必须审核 playbook 输出，手工更新控制端配置后，再重载和启动节点。

每次必须显式指定目标，例如：

```bash
ansible gpu-node03 -m ping

ansible-playbook /etc/ansible/playbook/add_compute_node.yml \
  -e slurm_target=gpu-node03
```

多台节点可传入 inventory 组名；playbook 使用 `serial: 1` 逐台处理，避免并发改写控制端配置。默认加入已存在的 `gpu5090` 分区，不同 GPU、CUDA 或拓扑的节点应在 inventory host vars 中覆盖 `slurm_partition`、`slurm_cuda_prefix`、`slurm_node_parameters`、`slurm_real_memory_mb` 和 `slurm_gpu_type`。

Playbook 已通过 `--syntax-check`、`--list-hosts` 和 `--list-tasks` 检查。为避免 SSH 认证配置失误导致节点锁定，它只安装 `pam_slurm` 与 `pam_slurm_adopt` 模块，不自动修改 `/etc/pam.d/sshd`。

### 12.1 gpu-node01 旧版 Playbook 测试结果

2026-09-14 确认之前发起的旧版 Playbook 已完整执行，其结果为：

```
NodeName=gpu-node01
NodeAddr=203.0.113.20
CPUs=96
Sockets=8
CoresPerSocket=6
ThreadsPerCore=2
RealMemory=500000
Gres=gpu:rtx_4090:8
Parameters=numa_node_as_socket
Partition=gpu-a
State=IDLE
Slurm=26.05.4
```

`gpu-node01` 的 `munge` 和 `slurmd` 已设为开机自启并保持 active。该状态为旧版测试的已完成结果，本次修改未回滚现有节点或控制端配置。
