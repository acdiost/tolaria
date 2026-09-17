---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
tags:
  - Slurm
  - Troubleshooting
  - GPU
---

# Slurm 节点 INVAL 排障与修复

`INVAL` 表示控制器配置的节点资源与 `slurmd` 实际上报信息不一致。不要直接反复执行 `scontrol update State=RESUME`；先找出 CPU、内存、GRES 或配置来源的差异，否则节点会立即回到无效状态。

## 本次处理结果

```text
control   IDLE  gpu:grid_v100d-1q:1
compute01        IDLE  gpu:grid_v100d-1q:2
```

本次修复涉及以下配置一致性问题：

- 为控制节点和计算节点明确声明 CPU、内存和 GPU 数量。
- 配置 `gres.conf` 与 NVIDIA 设备自动探测。
- 计算节点通过 configless 模式从控制器获取 Slurm 配置。
- 清理不适用的 cgroup 配置并补齐 `job_container.conf` 的基础路径。
- 配置生效后重启 `slurmd`，再恢复节点状态。

## 第一步：读取控制器给出的原因

```bash
sinfo -R
scontrol show node control -o
scontrol show node compute01 -o
journalctl -u slurmctld -b --no-pager | tail -n 150
journalctl -u slurmd -b --no-pager | tail -n 150
```

重点看 `Reason=`，并比较以下字段：

- `CPUTot`、Sockets、CoresPerSocket、ThreadsPerCore
- `RealMemory`
- `Gres` 与 `CfgTRES`
- `NodeAddr`、`NodeHostName`
- 控制器与节点的 Slurm 版本

## 第二步：从节点获取硬件事实

```bash
slurmd -C
nvidia-smi -L
ls -l /dev/nvidia*
```

不要凭经验填写 `NodeName`。以 `slurmd -C` 的 CPU 拓扑和可用内存为起点，再为系统和守护进程预留少量内存。

当前 `node.conf`：

```ini
NodeName=control CPUs=8 Boards=1 SocketsPerBoard=1 CoresPerSocket=8 ThreadsPerCore=1 RealMemory=15736 Gres=gpu:grid_v100d-1q:1
NodeName=compute01 CPUs=8 Boards=1 SocketsPerBoard=1 CoresPerSocket=8 ThreadsPerCore=1 RealMemory=15736 Gres=gpu:grid_v100d-1q:2
```

## 第三步：检查 GRES

控制器发布的 `/etc/slurm/gres.conf`：

```ini
AutoDetect=nvidia
NodeName=control Name=gpu Type=grid_v100d-1q File=/dev/nvidia0
```

检查控制器识别结果：

```bash
slurmd -G
scontrol show node control -o
scontrol show node compute01 -o
```

如果配置的 GPU 数量、类型或设备文件与自动探测结果不一致，Slurm 会将节点 drain 或 invalid。GPU 类型字符串也应在 `slurm.conf`、`node.conf`、`gres.conf` 和提交参数中保持一致。

## 第四步：确认 configless 节点

`compute01` 当前使用：

```ini
SLURMD_OPTIONS="--conf-server=192.0.2.10:6817"
```

缓存目录：

```text
/var/spool/slurmd/conf-cache/
```

检查：

```bash
systemctl cat slurmd
cat /etc/sysconfig/slurmd
find /var/spool/slurmd/conf-cache -maxdepth 1 -type f -print
```

configless 模式下，计算节点没有 `/etc/slurm/gres.conf` 并不一定是错误；应检查控制器下发到缓存目录的实际文件。

## 第五步：cgroup 与作业容器

当前配置：

```ini
# /etc/slurm/cgroup.conf
CgroupPlugin=autodetect
ConstrainCores=yes
ConstrainDevices=yes
ConstrainRAMSpace=yes
ConstrainSwapSpace=yes

# /etc/slurm/job_container.conf
AutoBasePath=true
BasePath=/var/spool/slurm/job_container/%n
# /etc/slurm/partition.conf
PartitionName=normal Nodes=control,compute01 Default=YES MaxTime=24:00:00 State=UP
```

配置语法先用 Slurm 自身命令验证，再重启服务：

```bash
slurmctld -t
systemctl restart slurmctld
systemctl restart slurmd
```

## 恢复与验收

确认硬件上报与配置一致后：

```bash
scontrol update NodeName=control State=RESUME
scontrol update NodeName=compute01 State=RESUME
sinfo -N -l
```

提交最小 GPU 作业：

```bash
srun -N1 -w control --gres=gpu:grid_v100d-1q:1 nvidia-smi -L
srun -N1 -w compute01 --gres=gpu:grid_v100d-1q:1 nvidia-smi -L
```

随后检查记账：

```bash
sacct -S today -X -o JobID,JobName,State,NodeList,AllocTRES
```

## 参考资料

- [SchedMD：GRES Scheduling](https://slurm.schedmd.com/gres.html)
- [SchedMD：Troubleshooting Guide](https://slurm.schedmd.com/troubleshoot.html)
