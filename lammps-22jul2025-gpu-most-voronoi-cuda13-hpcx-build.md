---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
  - "[[hpc-software-stack-lmod-python-uv-cuda-hpcx]]"
  - "[[lammps-22jul2025-kokkos-cuda-hpcx-build-and-lmod]]"
tags:
  - HPC
  - LAMMPS
  - GPU
  - CUDA
  - HPC-X
  - VORONOI
  - MOLECULE
  - Lmod
  - Runbook
---
# LAMMPS 22Jul2025 GPU、MOST、VORONOI CUDA 13 编译与 Lmod 发布

本文以一个示例 HPC 集群为背景，记录 LAMMPS 22 Jul 2025 Update 6 构建的完整过程。本次构建启用传统 `GPU` package、`most.cmake` 包集合，并显式设置 `PKG_VORONOI=yes`；最终包清单也确认包含 `MOLECULE` 和 `EXTRA-MOLECULE`。


登录节点及共享软件目录：

```
登录节点       apps@login.example.internal
共享软件目录   /opt/hpc/apps
源码包         /opt/hpc/apps/packages/lammps-stable.tar.gz
源码版本       LAMMPS 22Jul2025 Update 6 / 2025.7.22.6
```

## 最终交付

```
安装前缀
/opt/hpc/apps/lammps/22Jul2025/gcc-13.3.0/hpcx-2.51/cuda-13.0.3/sm86-sm89-sm120

构建目录
/opt/hpc/apps/lammps/22Jul2025/_build/gcc-13.3.0/hpcx-2.51/cuda-13.0.3/sm86-sm89-sm120

源码目录
/opt/hpc/apps/lammps/22Jul2025/_src/lammps-22Jul2025

Modulefile
/opt/hpc/apps/modulefiles/lammps/22Jul2025-gcc13.3-hpcx2.51-gpu-most-cuda13.0.3-sm86-sm89-sm120.lua
```

用户入口：

```bash
module load lammps/22Jul2025-gcc13.3-hpcx2.51-gpu-most-cuda13.0.3-sm86-sm89-sm120
lmp -sf gpu -pk gpu 1 -in in.lj
```

本次没有修改 LAMMPS 的默认 module，也没有覆盖已有 Kokkos 构建。

## 源码与校验

安装包内容确认其顶层目录为 `lammps-22Jul2025/`。归档校验值：

```
SHA-256  <LAMMPS_ARCHIVE_SHA256>
文件      /opt/hpc/apps/packages/lammps-stable.tar.gz
```

同一版本源码已存在于规范 `_src/` 目录，因此直接复用，没有重复解包或创建第二份源码树。

## 工具链与依赖

| 角色 | module | 实际路径 / 版本 |
| --- | --- | --- |
| C 编译器 | `gcc/13.3.0-system` + HPC-X wrapper | `ompi5/bin/mpicc` → GCC 13.3.0 |
| C++ 编译器 | `gcc/13.3.0-system` + HPC-X wrapper | `ompi5/bin/mpicxx` → G++ 13.3.0 |
| Fortran 编译器 | `gcc/13.3.0-system` | `/usr/bin/gfortran-13` |
| MPI | `hpcx/2.51` | Open MPI 5.0.10，MPI API 3.1 |
| CUDA | `cuda/13.0.3` | `/opt/hpc/apps/cuda/13.0.3` |
| CMake | `cmake/4.4.3` | `/opt/hpc/apps/cmake/4.4.3/bin/cmake` |
| Ninja | `ninja/1.13.2` | `/opt/hpc/apps/ninja/1.13.2/bin/ninja` |
| Eigen | `eigen/3.4.1` | `/opt/hpc/apps/eigen/3.4.1/gcc-13.3.0/nompi` |
| FFT | 内置 KISS FFT | 不依赖外部 FFTW |
| Voro++ | 无可复用 module | 0.4.6，构建目录内静态编译并嵌入 |

首次配置时，CMake 因没有找到 Eigen 自动取了 Eigen 3.4.0。依据依赖复用规范，随后显式设置 `DOWNLOAD_EIGEN3=OFF` 与 `Eigen3_DIR`，将最终构建修正为复用现有 Eigen 3.4.1，并增量重编所有受影响目标。最终配置日志显示 `Eigen3` 已被发现。

## 配置命令

非登录 shell 必须先加载集群模块初始化脚本：

```bash
. /etc/profile.d/hpc-modules.sh
module purge
module load gcc/13.3.0-system hpcx/2.51 cuda/13.0.3 \
  cmake/4.4.3 ninja/1.13.2 eigen/3.4.1
```

最终配置命令：

```shellscript
src=/opt/hpc/apps/lammps/22Jul2025/_src/lammps-22Jul2025
bld=/opt/hpc/apps/lammps/22Jul2025/_build/gcc-13.3.0/hpcx-2.51/cuda-13.0.3/sm86-sm89-sm120
prefix=/opt/hpc/apps/lammps/22Jul2025/gcc-13.3.0/hpcx-2.51/cuda-13.0.3/sm86-sm89-sm120

cmake -C "$src/cmake/presets/most.cmake" \
  -S "$src/cmake" \
  -B "$bld" \
  -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$prefix" \
  -DCMAKE_C_COMPILER="$(command -v mpicc)" \
  -DCMAKE_CXX_COMPILER="$(command -v mpicxx)" \
  -DBUILD_MPI=ON \
  -DBUILD_OMP=ON \
  -DPKG_GPU=ON \
  -DGPU_API=cuda \
  -DGPU_ARCH=sm_89 \
  -DCUDA_BUILD_MULTIARCH=ON \
  -DCUDPP_OPT=OFF \
  -DPKG_VORONOI=yes \
  -DFFT=KISS \
  -DCMAKE_CXX_STANDARD=17 \
  -DDOWNLOAD_EIGEN3=OFF \
  -DEigen3_DIR=/opt/hpc/apps/eigen/3.4.1/gcc-13.3.0/nompi/share/eigen3/cmake
```

首次下载上游辅助资源较慢时，按服务器规范仅在当前会话使用临时代理，没有写入 shell 配置：

```bash
export http_proxy=http://proxy.example.internal:3128
export https_proxy=http://proxy.example.internal:3128
```

## 编译与安装

```bash
cmake --build "$bld" -j 96
cmake --install "$bld"
```

初始构建共有 2349 个 Ninja 目标并成功完成。修正 Eigen 复用后，1523 个受影响目标重新编译并成功安装。CUDA 13 针对 LAMMPS 源码中的旧 `double4` 类型给出弃用警告，但没有编译错误。

## 最终启用包

最终生成的 `styles/lmpinstalledpkgs.h` 共列出 68 个包：

```
AMOEBA ASPHERE BOCS BODY BPM BROWNIAN CG-DNA CG-SPICA CLASS2
COLLOID COLVARS COMPRESS CORESHELL DIELECTRIC DIFFRACTION DIPOLE
DPD-BASIC DPD-MESO DPD-REACT DPD-SMOOTH DRUDE EFF ELECTRODE
EXTRA-COMMAND EXTRA-COMPUTE EXTRA-DUMP EXTRA-FIX EXTRA-MOLECULE
EXTRA-PAIR FEP GPU GRANULAR INTERLAYER KSPACE LEPTON MACHDYN
MANYBODY MC MEAM MESONT MISC ML-IAP ML-POD ML-SNAP ML-UF3 MOFFF
MOLECULE OPENMP OPT ORIENT PERI PHONON PLUGIN POEMS QEQ REACTION
REAXFF REPLICA RHEO RIGID SHOCK SPH SPIN SRD TALLY UEF VORONOI YAFF
```

特别核对：

```
GPU             已启用
VORONOI         已启用
MOLECULE        已启用
EXTRA-MOLECULE  已启用
```

因此该发布可以使用 LAMMPS 的分子拓扑相关样式；具体输入仍需保证所用 style 属于上述已编译包或 LAMMPS 核心。

## GPU 架构核对

LAMMPS 传统 GPU package 使用旧 FindCUDA 流程。CUDA 13 下，`CUDA_BUILD_MULTIARCH=ON` 被转成：

```
nvcc -arch=all
```

实际 fatbin 架构并集为：

```
sm_75 sm_80 sm_86 sm_87 sm_88 sm_89 sm_90
sm_100 sm_103 sm_110 sm_120 sm_121
```

其中明确包含集群所需：

```
gpu-sm89-partition   示例 sm_89 GPU
gpu-sm120-partition  示例 sm_120 GPU
```

GPU package 会用 `bin2c` 将 fatbin 转为字节数组后嵌入最终可执行文件，因此直接对 `bin/lmp` 执行 `cuobjdump --list-elf` 不会列出这些内部镜像。本次对构建目录中嵌入前的 `cuda_compile_fatbin_*.fatbin` 执行 `cuobjdump --list-elf` 完成核验。

## Lmod 发布

Module 名称：

```
lammps/22Jul2025-gcc13.3-hpcx2.51-gpu-most-cuda13.0.3-sm86-sm89-sm120
```

其依赖：

```
gcc/13.3.0-system
hpcx/2.51
cuda/13.0.3
eigen/3.4.1
```

加载验证：

```
which lmp
→ /opt/hpc/apps/lammps/22Jul2025/gcc-13.3.0/hpcx-2.51/cuda-13.0.3/sm86-sm89-sm120/bin/lmp

LAMMPS_ROOT
→ /opt/hpc/apps/lammps/22Jul2025/gcc-13.3.0/hpcx-2.51/cuda-13.0.3/sm86-sm89-sm120

CUDA_HOME
→ /opt/hpc/apps/cuda/13.0.3
```

没有修改 `/opt/hpc/apps/modulefiles/lammps/.modulerc.lua`，因此不会改变现有默认版本。

## Slurm 示例

安装前缀内已生成 `run.slurm`。以下为 sm_89 GPU 分区的单卡示例，CPU、内存和资源名称必须按目标集群策略调整：

```bash
#!/bin/bash
#SBATCH -J lammps-gpu-most
#SBATCH -p <GPU_SM89_PARTITION>
#SBATCH -A <SLURM_ACCOUNT>
#SBATCH --gres=gpu:<GPU_SM89_RESOURCE>:1
#SBATCH -n 1 -c 12
#SBATCH --mem=65536M
#SBATCH --time=00:30:00
#SBATCH -o slurm-%j.out

set -euo pipefail
. /etc/profile.d/hpc-modules.sh
module purge
module load lammps/22Jul2025-gcc13.3-hpcx2.51-gpu-most-cuda13.0.3-sm86-sm89-sm120

export OMP_NUM_THREADS="${SLURM_CPUS_PER_TASK}"
srun --mpi=pmix lmp -sf gpu -pk gpu 1 -in in.lj
```

提交前必须把尖括号中的分区、账号和 GPU 资源占位符替换为实际值。

## 验证结果与限制

已完成：

- CMake 配置、Ninja 编译和安装成功。
- 最终包清单静态核验包含 `GPU`、`VORONOI`、`MOLECULE`、`EXTRA-MOLECULE`。
- `ldd bin/lmp | grep 'not found'` 无输出。
- 动态链接命中 HPC-X `libmpi.so.40`、系统 `libgomp.so.1` 和 `libcuda.so.1`。
- 新 module 在 `module avail` 中可见，`module load` 后 `which lmp` 命中新前缀。
- GPU fatbin 包含 `sm_89` 与 `sm_120`。
- `BUILD.md`、`run.slurm` 和 Lua modulefile 均已生成；示例部署中归属软件服务账号和共享软件组。
- `sbatch --test-only run.slurm` 只返回 `Invalid account or account/partition combination specified`，没有其他 CPU/GPU/内存策略错误。

尚未完成：

- 登录节点没有可用 GPU。该传统 GPU package 构建在静态初始化阶段调用 CUDA，`lmp -h` 会报 `Cuda driver error 100`，因此不能把登录节点运行标记为通过。
- 软件服务账号当时没有 Slurm association，无法在 sm_89 与 sm_120 两类计算节点提交作业。
- 管理员分配 account 后，需分别在两类 GPU 节点执行最小算例，确认传统 GPU package 真实运行。

## 后续验收建议

获得有效 account 后，先复制安装前缀内的 `run.slurm` 和一个小型 `in.lj` 到作业目录，在 sm_89 GPU 分区执行：

```bash
sbatch run.slurm
```

sm_120 GPU 分区的单卡示例应调整为：

```
#SBATCH -p <GPU_SM120_PARTITION>
#SBATCH --gres=gpu:<GPU_SM120_RESOURCE>:1
#SBATCH -n 1 -c 16
#SBATCH --mem=65536M
```

验收日志至少应保留 LAMMPS 版本、GPU device 名称、使用的 package/suffix、MPI rank 数、OpenMP 线程数、步数完成情况和退出码，并将结果补充到安装前缀的 `BUILD.md`。
