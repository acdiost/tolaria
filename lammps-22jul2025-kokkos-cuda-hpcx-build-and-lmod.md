---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
  - "[[hpc-software-stack-lmod-python-uv-cuda-hpcx]]"
tags:
  - HPC
  - LAMMPS
  - KOKKOS
  - CUDA
  - HPC-X
  - Lmod
  - Runbook
---

# LAMMPS 22Jul2025 KOKKOS CUDA、HPC-X 编译与 Lmod 发布

这份手册记录 `gpu-node01.acdiost.internal`（管理地址 `198.51.100.20`）上的 LAMMPS 22 Jul 2025 Update 6 GPU/MPI 构建。完成后的用户入口是：

```bash
module load lammps
lmp -h
```

安装使用共享目录 `/acdiost/home/apps`，可执行文件为：

```text
/acdiost/home/apps/lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51/bin/lmp
```

本构建使用 KOKKOS CUDA 后端，不启用传统 GPU package。运行 GPU 输入时使用 `-k on ... -sf kk`。当前 HPC-X/UCX 组合的直接 CUDA device-buffer 通信存在段错误，多 MPI rank 作业必须额外使用：

```text
-pk kokkos gpu/aware off
```

该限制已经过失败与成功对照验证，不能从生产命令中省略。

## 已验证基线

```text
节点              gpu-node01.acdiost.internal
操作系统          Ubuntu 24.04.1 LTS，x86_64
CPU               AMD EPYC 7402
CPU 逻辑核         96
GPU               NVIDIA GeForce RTX 4090 × 8
GPU compute cap.  8.9
GPU 驱动          580.173.02
LAMMPS            22 Jul 2025 - Update 6
内部版本          2025.7.22.6
C/C++ 编译器      GCC/G++ 13.3.0
CUDA Toolkit      12.8.93
KOKKOS            4.6.2
KOKKOS GPU 架构   ADA89 / sm_89
KOKKOS CPU 架构   ZEN2
MPI               HPC-X 2.51 / Open MPI 5.0.10rc2
OpenMP            4.5
Lmod              8.6.19
```

已验证的 KOKKOS execution spaces：

```text
CUDA
OpenMP
Serial
```

已启用的 LAMMPS packages：

```text
KOKKOS
KSPACE
MANYBODY
MOLECULE
RIGID
```

CPU 侧主 FFT 使用内置 KISS FFT；KOKKOS GPU FFT 使用 cuFFT。

## 目录布局

```text
源码
/acdiost/home/apps/lammps-22Jul2025

构建目录
/acdiost/home/apps/lammps-22Jul2025/build-kokkos-cuda12.8-hpcx2.51

安装前缀
/acdiost/home/apps/lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51

Modulefile
/acdiost/home/apps/modulefiles/lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51.lua

默认版本声明
/acdiost/home/apps/modulefiles/lammps/.modulerc.lua
```

安装目录和 modulefile 均归 `apps:apps` 所有。可执行文件权限为 `0755`，modulefile 权限为 `0644`。

## 构建设计

### KOKKOS 而不是传统 GPU package

KOKKOS 提供 CUDA、OpenMP 和 Serial execution spaces，并为常用 pair、neighbor、fix、KSPACE 等组件提供 `/kk` styles。用户通过：

```text
-k on g <每节点 GPU 数> t <每 rank 线程数>
-sf kk
```

启用 KOKKOS GPU 路径。传统 GPU package 没有同时编译，避免同一个发布中维护两套 GPU 后端和两套运行参数。

### 外部 HPC-X MPI

```text
BUILD_MPI=ON
MPI_CXX_COMPILER=<HPC-X 2.51 mpicxx>
```

CMake 使用 KOKKOS 的 `nvcc_wrapper` 编译 C++/CUDA，同时从 HPC-X 获取 MPI include 和 `libmpi.so`。modulefile 使用 `depends_on("hpcx/2.51")`，确保运行时 MPI 与编译时 ABI 一致。

### 精确指定 GPU 与 CPU 架构

```text
Kokkos_ARCH_ADA89=ON
Kokkos_ARCH_ZEN2=ON
```

该构建针对 RTX 4090 和 AMD EPYC 7402。若其他节点采用不同 GPU compute capability 或 CPU 微架构，应创建新的安装前缀和 modulefile，不能只复用当前二进制。

### 基础包集

构建载入 LAMMPS 自带的 `basic.cmake`，启用 KSPACE、MANYBODY、MOLECULE、RIGID，再叠加 `kokkos-cuda.cmake`。这覆盖常见 LJ、EAM、Tersoff、分子拓扑、刚体和长程静电场景，同时不引入 `most.cmake` 中大量外部库。

需要 REAXFF、ML-IAP、PLUMED、HDF5、VORONOI 或其他未启用 package 时，应先核对依赖，再以新版本后缀发布，不能原地替换当前模块。

## 编译步骤

### 1. 准备环境

本次复用了此前安装的 CMake 3.28.3 和 Ninja 1.11.1。以 `apps` 用户执行：

```bash
su - apps
source /etc/profile.d/lmod.sh
module purge
module load hpcx/2.51

export PATH=/usr/local/cuda-12.8/bin:$PATH
export CUDA_PATH=/usr/local/cuda-12.8
export NVCC_WRAPPER_DEFAULT_COMPILER=g++
```

检查工具链：

```bash
cmake --version
nvcc --version
mpicxx --showme:version
mpicxx --showme:command
```

### 2. 配置

```bash
src=/acdiost/home/apps/lammps-22Jul2025
bld=/acdiost/home/apps/lammps-22Jul2025/build-kokkos-cuda12.8-hpcx2.51
prefix=/acdiost/home/apps/lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51

mkdir -p "$bld"

cmake -S "$src/cmake" -B "$bld" \
  -G Ninja \
  -C "$src/cmake/presets/basic.cmake" \
  -C "$src/cmake/presets/kokkos-cuda.cmake" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$prefix" \
  -DBUILD_MPI=ON \
  -DMPI_CXX_COMPILER="$(command -v mpicxx)" \
  -DKokkos_ARCH_ZEN2=ON \
  -DKokkos_ARCH_ADA89=ON \
  -DKokkos_ENABLE_OPENMP=ON \
  -DKokkos_ENABLE_CUDA=ON \
  -DKokkos_ENABLE_SERIAL=ON \
  -DFFT_KOKKOS=CUFFT \
  -DBUILD_OMP=ON \
  -DBUILD_TESTING=OFF
```

配置日志中的关键结果应为：

```text
LAMMPS Version:   2025.7.22.6
C++ Compiler:     .../lib/kokkos/bin/nvcc_wrapper
Enabled packages: KOKKOS;KSPACE;MANYBODY;MOLECULE;RIGID
Kokkos Devices:   CUDA;CUDA_LAMBDA;OPENMP;SERIAL
Kokkos Arch:      ADA89;ZEN2
Kokkos FFT:       CUFFT
MPI libraries:    .../hpcx/.../ompi5/lib/libmpi.so
```

### 3. 编译安装

```bash
cmake --build "$bld" --parallel 48 --target install
```

本节点有 96 个逻辑 CPU 和约 503 GiB 内存，48 路并行编译成功。复用到资源较小的节点时应降低并行度，尤其是多个 `nvcc_wrapper` 进程并发时。

## Lmod 配置

### Modulefile

`/acdiost/home/apps/modulefiles/lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51.lua`：

```lua
help([[LAMMPS 22 Jul 2025 Update 6 with KOKKOS, CUDA 12.8, OpenMP, and HPC-X MPI.

Single-GPU example:
  lmp -k on g 1 t 6 -sf kk -in in.lammps

Multi-rank CUDA jobs on the current HPC-X/UCX stack must add:
  -pk kokkos gpu/aware off
]])
whatis("LAMMPS 22 Jul 2025 Update 6 molecular dynamics package")
whatis("Accelerator: KOKKOS 4.6.2, CUDA sm_89 (RTX 4090), OpenMP")
whatis("MPI: HPC-X 2.51 / Open MPI 5")
whatis("Packages: KOKKOS KSPACE MANYBODY MOLECULE RIGID")

conflict("lammps")
depends_on("hpcx/2.51")

local root = "/acdiost/home/apps/lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51"
local cuda = "/usr/local/cuda-12.8"

prepend_path("PATH", pathJoin(root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(cuda, "lib64"))
prepend_path("CMAKE_PREFIX_PATH", root)
prepend_path("MANPATH", pathJoin(root, "share/man"))

setenv("LAMMPS_ROOT", root)
setenv("LAMMPS_POTENTIALS", pathJoin(root, "share/lammps/potentials"))
setenv("LAMMPS_BENCH", pathJoin(root, "share/lammps/bench"))
setenv("LAMMPS_VERSION", "22Jul2025-Update6")
```

### 默认版本

`/acdiost/home/apps/modulefiles/lammps/.modulerc.lua`：

```lua
module_version("lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51", "default")
```

交互环境可以简写：

```bash
module load lammps
```

生产作业建议固定完整版本：

```bash
module load lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51
```

共享 modulefile 路径已由 `/etc/lmod/modulespath` 中的以下记录全局发布：

```text
/acdiost/home/apps/modulefiles
```

## 验证

### 模块、版本和动态链接

```bash
module purge
module load lammps
module -t list
command -v lmp
lmp -h

ldd "$(command -v lmp)" | grep -E 'libmpi|libcudart|libgomp'
```

本次验证得到：

```text
libmpi.so.40  -> HPC-X 2.51 ompi5/lib/libmpi.so.40
libcudart.so.12 -> /usr/local/cuda-12.8/lib64/libcudart.so.12
libgomp.so.1  -> Ubuntu 系统 libgomp
```

`lmp -h` 报告：

```text
LAMMPS 22 Jul 2025 - Update 6
KOKKOS package API: CUDA OpenMP Serial
Kokkos library version: 4.6.2
KOKKOS FFT library = cuFFT
Installed packages: KOKKOS KSPACE MANYBODY MOLECULE RIGID
```

### 单 GPU smoke test

```bash
module purge
module load lammps

export OMP_NUM_THREADS=6
export OMP_PROC_BIND=spread
export OMP_PLACES=threads

mpirun -np 1 lmp \
  -k on g 1 t 6 \
  -sf kk \
  -in "$LAMMPS_BENCH/in.lj" \
  -log none
```

32,000 原子、100 步内置 LJ 算例成功完成。日志明确包含：

```text
will use up to 1 GPU(s) per node
pair lj/cut/kk
attributes: full, newton off, kokkos_device
pair build: full/bin/kk/device
bin: kk/device
Total wall time: 0:00:00
```

这些字段证明 pair 和 neighbor 构建实际走 KOKKOS device，而不是只编译进了 CUDA 支持。

### 8 GPU smoke test

```bash
module purge
module load lammps

export OMP_NUM_THREADS=6
export OMP_PROC_BIND=spread
export OMP_PLACES=threads

mpirun -np 8 lmp \
  -k on g 8 t 6 \
  -sf kk \
  -pk kokkos gpu/aware off \
  -in "$LAMMPS_BENCH/in.lj" \
  -log none
```

已验证结果：

```text
will use up to 8 GPU(s) per node
using 6 OpenMP thread(s) per MPI task
2 by 2 by 2 MPI processor grid
kokkos_device
full/bin/kk/device
Loop time of 0.0655082 on 48 procs for 100 steps with 32000 atoms
Total wall time: 0:00:01
```

该算例用于验证 8 GPU、MPI 域分解和 CUDA kernel 路径，不是性能基准。32,000 原子不足以有效衡量八张 GPU 的强扩展性能。

## GPU-aware MPI 兼容问题

### 现象

双 MPI rank、双 GPU 在默认 KOKKOS GPU-aware 模式下发生 SIGSEGV：

```text
Signal: Segmentation fault (11)
Failing at address: ...
libuct.so.0(uct_mm_ep_am_bcopy...)
libucp.so.0(ucp_tag_send_nbx...)
mca_pml_ucx.so(mca_pml_ucx_send...)
MPI_Send
```

调用栈显示错误位于 HPC-X 的 UCX PML/共享内存传输路径。单 rank GPU 算例正常，增加以下参数后双 GPU 和八 GPU 均正常：

```text
-pk kokkos gpu/aware off
```

### 当前处理

`gpu/aware off` 让 KOKKOS 在 MPI 通信前把 device 数据暂存到 host memory，绕开 UCX 对 device pointer 的错误处理。它优先保证正确性，但会增加 host-device copy，可能降低多 GPU 或多节点通信性能。

modulefile 的帮助文本已经记录该必需参数：

```bash
module help lammps
```

在重新验证以下任一变化前，不应删除该参数：

- HPC-X/UCX 升级；
- 改用与 CUDA 12.8 明确匹配的 MPI/UCX 构建；
- CUDA Toolkit 或驱动升级；
- LAMMPS/KOKKOS 升级；
- UCX CUDA memory-type 检测和 transport 配置修复。

## Slurm 作业模板

```bash
#!/bin/bash
#SBATCH --job-name=lammps-test
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --cpus-per-task=6
#SBATCH --gres=gpu:8
#SBATCH --time=00:10:00

set -euo pipefail
source /etc/profile.d/lmod.sh
module purge
module load lammps/22Jul2025-kokkos-cuda12.8-hpcx2.51

export OMP_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export OMP_PROC_BIND=spread
export OMP_PLACES=threads

mpirun -np "$SLURM_NTASKS" lmp \
  -k on g 8 t "$OMP_NUM_THREADS" \
  -sf kk \
  -pk kokkos gpu/aware off \
  -in in.lammps
```

分区、账户、GPU GRES 名称和 MPI 启动方式应按集群 Slurm 配置补充。若调度系统为每个 rank 单独设置 `CUDA_VISIBLE_DEVICES`，应先用短作业检查 LAMMPS 的 GPU 映射，再提交生产任务。

## 运维检查表

1. 用 `nvidia-smi` 核对目标节点 GPU 数量、型号、compute capability 和驱动。
2. 确认每个目标节点存在兼容的 `/usr/local/cuda-12.8/lib64/libcudart.so.12`。
3. `module load lammps` 后确认同时加载 `hpcx/2.51`。
4. 用 `ldd` 确认 `libmpi.so` 与 `libcudart.so` 没有指向其他版本。
5. 用 `lmp -h` 核对版本、KOKKOS backends 和 enabled packages。
6. 单 GPU 运行内置 LJ smoke test，并检查 `kokkos_device`。
7. 多 GPU 必须带 `-pk kokkos gpu/aware off`，并用 `set -o pipefail` 保留真实退出码。
8. 用用户真实输入检查所需 pair/fix/compute style 是否已编译。
9. 通过短 Slurm 作业核对 GPU 映射、MPI rank、OpenMP 绑定和性能。
10. 新版本使用新安装前缀和 modulefile；验证通过后再调整默认版本。
