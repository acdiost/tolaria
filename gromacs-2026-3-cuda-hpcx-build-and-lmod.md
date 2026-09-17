---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
  - "[[hpc-software-stack-lmod-python-uv-cuda-hpcx]]"
tags:
  - HPC
  - GROMACS
  - CUDA
  - HPC-X
  - Lmod
  - Runbook
---

# GROMACS 2026.3 CUDA、HPC-X 编译与 Lmod 发布

这份手册记录 `gpu-node01.acdiost.internal`（管理地址 `198.51.100.20`）上的 GROMACS 2026.3 GPU/MPI 构建。完成后的用户入口是：

```bash
module load gromacs
gmx_mpi --version
```

安装使用共享目录 `/acdiost/home/apps`，可执行文件为：

```text
/acdiost/home/apps/gromacs/2026.3-cuda12.8-hpcx2.51/bin/gmx_mpi
```

本文同时保存本次可复现配置和后续升级检查项。CUDA、GPU 架构、MPI ABI、CPU SIMD 和 GROMACS 源码行为都可能随节点或版本变化，升级时不能只替换版本号后直接重编。

## 已验证基线

```text
节点            gpu-node01.acdiost.internal
操作系统        Ubuntu 24.04.1 LTS，x86_64
CPU             AMD EPYC 7402
CPU 逻辑核       96
GPU             NVIDIA GeForce RTX 4090 × 8
GPU 驱动        580.173.02
GROMACS         2026.3，mixed precision
C/C++ 编译器    GCC/G++ 13.3.0
CUDA Toolkit    12.8.93
CUDA 架构       sm_89
MPI             HPC-X 2.51 / Open MPI 5.0.10rc2
CPU SIMD        AVX2_256、FMA
CPU FFT         FFTW 3.3.10，SSE2/AVX/AVX2/AVX2_128
GPU FFT         cuFFT
Lmod            8.6.19
```

`gmx_mpi --version` 已确认 `MPI (GPU-aware: CUDA)`。双 rank 的 `mpirun -np 2 gmx_mpi --version` 也已通过。

这次只验证了工具链、动态链接和 MPI 启动，没有在用户体系上运行生产 `.tpr`。正式投入前仍应提交一个短 MD 作业，检查 GPU 映射、PME 分配、性能和结果稳定性。

## 目录布局

```text
源码
/acdiost/home/apps/gromacs-2026.3

有效构建目录
/acdiost/home/apps/gromacs-2026.3/build-cuda12.8-hpcx2.51-make

安装前缀
/acdiost/home/apps/gromacs/2026.3-cuda12.8-hpcx2.51

Modulefile
/acdiost/home/apps/modulefiles/gromacs/2026.3-cuda12.8-hpcx2.51.lua

默认版本声明
/acdiost/home/apps/modulefiles/gromacs/.modulerc.lua
```

源码目录中还保留了 `build-cuda12.8-hpcx2.51`。该目录是早期 Ninja 配置尝试，不是有效发布构建；后续增量编译应使用带 `-make` 后缀的目录。

## 构建设计

### 外部 MPI，关闭 thread-MPI

这套构建用于单节点多 GPU，也为多节点运行保留能力，因此使用 HPC-X 的外部 Open MPI：

```text
GMX_MPI=ON
GMX_THREAD_MPI=OFF
```

最终二进制名称是 `gmx_mpi`。modulefile 通过 `depends_on("hpcx/2.51")` 自动加载编译时使用的 MPI，避免运行时误连到其他 `libmpi.so`。

### 只编译 RTX 4090 架构

```text
CMAKE_CUDA_ARCHITECTURES=89
```

这能减少编译时间和 CUDA 代码体积，但生成物针对 Ada `sm_89`。如果同一个模块需要覆盖其他 GPU，必须按实际 GPU compute capability 增加目标列表或拆分模块，不能假定该构建可用于所有节点。

### 由 GROMACS 管理 FFTW

现有 `fftw/3.3.10-gcc13.3.0` 模块可被链接，但配置检测显示它没有 SIMD 支持。最终构建改用：

```text
GMX_BUILD_OWN_FFTW=ON
```

GROMACS 下载 FFTW 3.3.10，并启用 SSE2、AVX、AVX2 和 AVX2_128。该路径不支持 Ninja，所以有效构建必须使用 `Unix Makefiles`。

### 保留单元测试目标

当前这份 2026.3 源码在 `GMX_BUILD_UNITTESTS=OFF` 时仍对不存在的 `nblib-listed-forces-test` 调用 `target_link_libraries`，导致 CMake 配置失败。实际配置使用：

```text
GMX_BUILD_UNITTESTS=ON
BUILD_TESTING=ON
```

这是针对当前源码树的兼容处理。升级到新的源码快照后应重新测试能否关闭测试构建，不要永久照搬。

## 编译步骤

### 1. 安装构建工具

以 root 执行：

```bash
apt-get install -y cmake ninja-build
```

GROMACS 2026.3 要求 CMake 3.28；Ubuntu 24.04 提供的 CMake 3.28.3 满足要求。虽然安装了 Ninja，但本配置因内置 FFTW 使用 Unix Makefiles。

系统还需要可用的 GCC/G++、GNU Make、Perl、Python 和常规 libc 开发环境。本节点在开始编译前已经具备这些依赖。

### 2. 切换到 apps 并加载 MPI

```bash
su - apps
source /etc/profile.d/lmod.sh
module purge
module load hpcx/2.51

command -v mpicc
command -v mpicxx
mpicc --showme:version
```

本次解析到的 MPI wrapper 位于：

```text
/acdiost/home/apps/hpcx/hpcx-v2.51-gcc-doca_ofed-ubuntu24.04-cuda13-x86_64/ompi5/bin/
```

### 3. 配置

```bash
src=/acdiost/home/apps/gromacs-2026.3
bld=/acdiost/home/apps/gromacs-2026.3/build-cuda12.8-hpcx2.51-make
prefix=/acdiost/home/apps/gromacs/2026.3-cuda12.8-hpcx2.51

mkdir -p "$bld"

cmake -S "$src" -B "$bld" \
  -G "Unix Makefiles" \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX="$prefix" \
  -DCMAKE_C_COMPILER="$(command -v mpicc)" \
  -DCMAKE_CXX_COMPILER="$(command -v mpicxx)" \
  -DCMAKE_CUDA_COMPILER=/usr/local/cuda-12.8/bin/nvcc \
  -DCMAKE_CUDA_ARCHITECTURES=89 \
  -DGMX_GPU=CUDA \
  -DGMX_MPI=ON \
  -DGMX_THREAD_MPI=OFF \
  -DGMX_BUILD_OWN_FFTW=ON \
  -DGMX_BUILD_UNITTESTS=ON \
  -DBUILD_TESTING=ON \
  -DREGRESSIONTEST_DOWNLOAD=OFF
```

配置日志中应至少出现：

```text
GROMACS library will use OpenMPI 5.0.10
Compiling GROMACS for CUDA architectures: 89
MPI_SUPPORTS_CUDA_AWARE_DETECTION - Success
Detected best SIMD instructions for this CPU - AVX2_256
The GROMACS-managed build of FFTW 3 will configure with:
--enable-sse2;--enable-avx;--enable-avx2
```

### 4. 编译并安装

```bash
cmake --build "$bld" --parallel 48 --target install
```

48 路并行在本节点的 96 个逻辑 CPU、约 503 GiB 内存环境下完成。复用到较小节点时应降低并行度，避免同时运行的 C++/CUDA 编译进程耗尽内存。

## Lmod 配置

### Modulefile

`/acdiost/home/apps/modulefiles/gromacs/2026.3-cuda12.8-hpcx2.51.lua` 内容如下：

```lua
help([[GROMACS 2026.3 with CUDA 12.8 and HPC-X Open MPI 5.0.10]])
whatis("GROMACS 2026.3 molecular dynamics package")
whatis("GPU: CUDA 12.8, architecture sm_89 (RTX 4090)")
whatis("MPI: HPC-X 2.51 / Open MPI 5")

conflict("gromacs")
depends_on("hpcx/2.51")

local root = "/acdiost/home/apps/gromacs/2026.3-cuda12.8-hpcx2.51"
local cuda = "/usr/local/cuda-12.8"

prepend_path("PATH", pathJoin(root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))
prepend_path("LD_LIBRARY_PATH", pathJoin(cuda, "lib64"))
prepend_path("LIBRARY_PATH", pathJoin(root, "lib"))
prepend_path("CPATH", pathJoin(root, "include"))
prepend_path("PKG_CONFIG_PATH", pathJoin(root, "lib/pkgconfig"))
prepend_path("CMAKE_PREFIX_PATH", root)
prepend_path("MANPATH", pathJoin(root, "share/man"))

setenv("GMXBIN", pathJoin(root, "bin"))
setenv("GMXLDLIB", pathJoin(root, "lib"))
setenv("GMXDATA", pathJoin(root, "share/gromacs"))
setenv("GMX_VERSION", "2026.3")
setenv("GROMACS_ROOT", root)
```

### 默认版本

`/acdiost/home/apps/modulefiles/gromacs/.modulerc.lua`：

```lua
module_version("gromacs/2026.3-cuda12.8-hpcx2.51", "default")
```

因此以下两种加载方式等价：

```bash
module load gromacs
module load gromacs/2026.3-cuda12.8-hpcx2.51
```

生产作业为了可复现，建议使用完整版本名。

### 全局 MODULEPATH

`/etc/lmod/modulespath` 已包含：

```text
/acdiost/home/apps/modulefiles
```

登录 shell 会初始化 Lmod，因此用户无需手工执行 `module use`。如果非登录 shell 找不到 `module` 函数，可以显式执行：

```bash
source /etc/profile.d/lmod.sh
```

## 验证

### 模块与版本

```bash
su - apps
module purge
module load gromacs
module -t list
command -v gmx_mpi
gmx_mpi --version
```

关键结果：

```text
Loaded modules:      hpcx/2.51, gromacs/2026.3-cuda12.8-hpcx2.51
GROMACS version:     2026.3
MPI library:         MPI (GPU-aware: CUDA)
GPU support:         CUDA
SIMD instructions:   AVX2_256
CPU FFT library:     fftw-3.3.10-sse2-avx-avx2-avx2_128
GPU FFT library:     cuFFT
CUDA compiler:       NVIDIA 12.8.93
CUDA targets:        89
CUDA driver:         13.0
CUDA runtime:        12.80
```

### 动态链接

```bash
ldd "$(command -v gmx_mpi)" | grep -E 'libgromacs|libmpi|libcudart|libfftw'
```

`libgromacs_mpi.so.11` 应来自 GROMACS 安装前缀，`libmpi.so.40` 应来自 HPC-X 2.51。若 MPI 指向 `/usr/lib` 或其他 module 的路径，应先处理模块污染，不能直接提交作业。

### MPI 启动

```bash
mpirun -np 2 gmx_mpi --version
```

该命令已成功执行。它验证 MPI 进程可启动，但不替代包含 GPU kernel、域分解和通信的短 MD 验证。

## 作业使用

### 单节点 8 GPU 示例

```bash
module purge
module load gromacs/2026.3-cuda12.8-hpcx2.51

mpirun -np 8 gmx_mpi mdrun \
  -deffnm md \
  -ntomp 6 \
  -nb gpu \
  -pme gpu \
  -bonded gpu
```

这里使用 8 个 MPI rank、每 rank 6 个 OpenMP 线程，对应节点的 48 个物理核心。GROMACS 会根据可见设备自动进行 GPU task mapping。实际最佳 rank/thread/PME 配置依赖体系规模，必须用真实输入做基准测试。

当前构建的 `gmx_mpi --version` 显示 `Multi-GPU FFT: none`。这不影响普通多 GPU 短程非键计算，但 GPU PME/FFT 的扩展方式受限；大型多节点体系应特别观察 PME rank 是否成为瓶颈。

### Slurm 模板

```bash
#!/bin/bash
#SBATCH --job-name=gmx-test
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --cpus-per-task=6
#SBATCH --gres=gpu:8
#SBATCH --time=00:10:00

set -euo pipefail
source /etc/profile.d/lmod.sh
module purge
module load gromacs/2026.3-cuda12.8-hpcx2.51

export OMP_NUM_THREADS="$SLURM_CPUS_PER_TASK"

mpirun -np "$SLURM_NTASKS" gmx_mpi mdrun \
  -deffnm md \
  -ntomp "$OMP_NUM_THREADS" \
  -nb gpu \
  -pme gpu \
  -bonded gpu \
  -nsteps 1000
```

先用短作业检查日志中的 GPU 数量、rank 映射、PME 分配和性能。确认无误后再移除 `-nsteps 1000` 或改为生产步数。分区、账户和 GPU GRES 名称按集群实际策略补充。

## 已知限制

本构建未启用或未检测到：

```text
Torch / neural-network potential
HDF5
hwloc
NVSHMEM
多 GPU FFT
```

其中 Torch、HDF5 和 NVSHMEM 属于可选能力；有明确需求时应创建新的安装前缀和 modulefile，不要原地覆盖当前版本。

CUDA Toolkit 位于节点本地的 `/usr/local/cuda-12.8`，GROMACS 安装和 modulefile 位于共享 Lustre。其他计算节点必须存在兼容的 `/usr/local/cuda-12.8/lib64`，并有足够新的 NVIDIA 驱动，否则共享模块能被发现但程序仍会在启动时失败。

## 升级检查表

1. 用 `nvidia-smi` 核对目标节点的 GPU 型号和驱动。
2. 用 `nvcc --version` 核对 Toolkit，并按 GPU 更新 `CMAKE_CUDA_ARCHITECTURES`。
3. 加载目标 HPC-X 后记录 `mpicc --showme:version` 和底层 GCC 版本。
4. 使用新的源码、构建目录、安装前缀和 modulefile 版本；不覆盖当前安装。
5. 重新检查 `GMX_BUILD_UNITTESTS=OFF` 是否仍触发 NB-LIB CMake 错误。
6. 检查配置输出中的 CUDA-aware MPI、SIMD 和 FFTW 优化。
7. 执行 `gmx_mpi --version`、动态链接检查和双 rank MPI smoke test。
8. 使用真实 `.tpr` 提交短 GPU 作业，再决定是否将新版本设为默认。
