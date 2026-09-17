---
type: Note
status: Active
category: "[[hpc-infrastructure]]"
related_to:
  - "[[hpc-cluster-operations-index]]"
tags:
  - HPC
  - Lmod
  - Python
  - CUDA
  - HPC-X
  - HDF5
  - NetCDF
  - OpenBLAS
  - C++
  - Runbook
---

# 使用 Lmod 管理 HPC-X、uv、多版本 Python、CUDA 与科学计算库

这份手册记录 `login.acdiost.internal`（管理地址 `198.51.100.11`）在 2026-09-15 建立的软件环境。用户通过 Lmod 选择基础工具链，通过 uv 隔离 Python 项目依赖：

```bash
module load hpcx/2.51
module load uv/0.12.13
module load python/3.12
module load cuda/13
module load cmake/4 ninja/1
module load netcdf-c/4.10.1-gcc13
```

所有应用安装在 `/acdiost/home/apps`，modulefile 安装在 `/acdiost/home/apps/modulefiles`。后者由 `/etc/profile.d/z-hpc-modules.sh` 加入全局 `MODULEPATH`，因此登录节点和计算节点上的登录 shell 都能发现这些模块。

本文既是当前配置清单，也是以后新增版本时的操作手册。命令中的版本、下载地址和驱动要求在升级前必须重新核对，不能机械复用。

## 当前基线

### 平台

```text
操作系统        Ubuntu 24.04，x86_64
Lmod            8.6.19
共享应用目录    /acdiost/home/apps（Lustre）
Modulefiles     /acdiost/home/apps/modulefiles
Slurm gpu-a        RTX 4090 × 8/节点
Slurm gpu-b        RTX 5090 × 8/节点
GPU 驱动        580.173.02（gpu-a、gpu-b 抽样验证一致）
```

`/usr/local` 位于各节点本地 ext4，并非共享文件系统。需要被所有节点使用的软件不能只安装在登录节点的 `/usr/local`；要么通过配置管理逐节点部署，要么像本次 CUDA 13 一样安装到共享 Lustre。

### 已发布模块

```text
hpcx/2.51
uv/0.12.13
cmake/4.4.3
ninja/1.13.2
gcc/13.3.0-system
openblas/0.3.34-gcc13-openmp
eigen/3.4.1
fmt/12.2.0-gcc13
spdlog/1.17.0-gcc13-fmt12.2

python/3.9       -> CPython 3.9.25
python/3.10      -> CPython 3.10.21
python/3.11      -> CPython 3.11.16
python/3.12      -> CPython 3.12.14
python/3.13      -> CPython 3.13.15
python/3.14      -> CPython 3.14.7

cuda/12          -> 节点本地 CUDA Toolkit 12.8.93
cuda/12.8
cuda/13          -> 共享 CUDA Toolkit 13.0.3
cuda/13.0        -> CUDA Toolkit 13.0.3
cuda/13.0.3

hdf5/2.2.0-gcc13-zlib
hdf5/2.2.0-gcc13-hpcx2.51-zlib
netcdf-c/4.10.1-gcc13
netcdf-c/4.10.1-gcc13-hpcx2.51
```

Python 同时保留精确版本模块，例如 `python/3.12.14`。批处理脚本若要求严格复现，优先加载精确版本；交互开发可以使用 `python/3.12` 这种短别名。

## 设计原则

### Lmod 只管理基础运行时

Lmod 管理 HPC-X、uv、Python 解释器、CUDA Toolkit 和编译型公共库。NumPy、PyTorch、TensorFlow 等项目依赖由 uv 写入项目自己的 `.venv`，不能安装到共享基础 Python。

```text
Lmod
├── hpcx/2.51
├── uv/0.12.13
├── python/3.9 ... python/3.14
├── cuda/12.8、cuda/13
├── cmake/4.4.3、ninja/1.13.2
├── gcc/13.3.0-system、openblas/0.3.34-gcc13-openmp
├── eigen/3.4.1、fmt/12.2.0-gcc13、spdlog/1.17.0-gcc13-fmt12.2
└── HDF5、NetCDF-C（串行/并行 ABI 分开）

项目目录
├── pyproject.toml
├── uv.lock
└── .venv/
```

### 驱动与 Toolkit 分离

NVIDIA 驱动属于计算节点操作系统，CUDA Toolkit 属于用户空间工具链。安装额外 Toolkit 时不安装驱动、不重启节点，也不修改 `/usr/local/cuda`。modulefile 始终指向精确目录。

### 常用 Python 共享，特殊版本归用户

Python 3.9—3.14 的常用版本集中安装，避免每个用户重复占用空间。uv 设置为 `manual` 下载模式：不会在作业运行时自动下载解释器，但用户仍可显式执行 `uv python install <版本>`，将特殊版本安装到自己的 home。

## HPC-X 2.51

### 安装与模块位置

```text
安装目录
/acdiost/home/apps/hpcx/hpcx-v2.51-gcc-doca_ofed-ubuntu24.04-cuda13-x86_64

Modulefile
/acdiost/home/apps/modulefiles/hpcx/2.51
```

HPC-X 自带 Tcl modulefile。本次以 `modulefiles/hpcx-ompi` 为基础创建版本化模块，并完成两项修正：

1. 将 `hpcx_dir` 固定为上述真实安装目录，避免复制 modulefile 后错误地从文件位置推导根目录。
2. 将 `$env(HPCX_NCCLNET_PLUGIN_DIR)/lib` 改为 `$hpcx_dir/nccl_spectrum-x_plugin/lib`。原写法在 `module show` 和 `module spider` 阶段会因环境变量尚未真正写入而报错。

模块还包含：

```tcl
conflict hpcx
```

### 使用与验证

```bash
module purge
module load hpcx/2.51

mpirun --version
ucx_info -v
echo "$HPCX_MPI_DIR"

module unload hpcx/2.51
```

已验证的核心版本：

```text
Open MPI  5.0.10rc2
UCX       1.22.0
```

默认使用 Open MPI 5。兼容旧应用时可在加载前选择 Open MPI 4：

```bash
export HPCX_ENABLE_OMPI4=1
module load hpcx/2.51
```

HPC-X 会向 `LD_LIBRARY_PATH` 注入 MPI、UCX、UCC、HCOLL、SHARP 和 NCCL 插件目录。普通 PyTorch 作业不要无条件加载 HPC-X；PyTorch wheel 往往携带自己的 NCCL/CUDA 用户态库，混用可能造成 ABI 或符号冲突。只有明确需要 MPI、UCX、SHARP 或 Spectrum-X 插件时才加载，并针对该组合做计算节点测试。

## uv 0.12.13

### 安装位置

```text
/acdiost/home/apps/uv/0.12.13/bin/uv
/acdiost/home/apps/uv/0.12.13/bin/uvx
/acdiost/home/apps/modulefiles/uv/0.12.13.lua
```

固定版本安装示例：

```bash
version=0.12.13
prefix=/acdiost/home/apps/uv/$version

mkdir -p "$prefix/bin"
curl --proto '=https' --tlsv1.2 -LsSf \
  "https://astral.sh/uv/$version/install.sh" \
  -o "/tmp/uv-install-$version.sh"

UV_INSTALL_DIR="$prefix/bin" \
UV_NO_MODIFY_PATH=1 \
sh "/tmp/uv-install-$version.sh"
```

生产环境不要执行 `uv self update`。升级 uv 时新增版本目录和 modulefile，验证后再通知用户迁移。

### Modulefile 的关键设置

```lua
family("uv")

local root = "/acdiost/home/apps/uv/0.12.13"
local home = os.getenv("HOME")

prepend_path("PATH", pathJoin(root, "bin"))

if home then
    setenv("UV_CACHE_DIR", pathJoin(home, ".cache", "uv"))
end

setenv("UV_PYTHON_DOWNLOADS", "manual")
setenv("UV_NO_MODIFY_PATH", "1")
```

这里没有设置 `UV_NO_MANAGED_PYTHON`。因此用户仍能显式安装特殊 Python；`manual` 只禁止 uv 因项目约束不匹配而自动下载。

每个用户使用 `$HOME/.cache/uv`，避免共享可写缓存的权限和污染问题。项目和缓存都位于 `/acdiost/home` 时，uv 可以使用链接优化；不要把所有用户指向同一个可写缓存目录。

## 多版本 Python

### 共享安装

Python 由 uv 安装在：

```text
/acdiost/home/apps/python/uv-managed/
```

管理员新增版本的通用命令：

```bash
uv=/acdiost/home/apps/uv/0.12.13/bin/uv
install_dir=/acdiost/home/apps/python/uv-managed

"$uv" python install \
  --install-dir "$install_dir" \
  --no-bin \
  --compile-bytecode \
  3.11 3.12
```

`--no-bin` 防止安装器向 apps 用户的个人 bin 目录写入全局命令。安装完成后，uv 会同时创建精确目录和小版本链接，例如：

```text
cpython-3.12.14-linux-x86_64-gnu/
cpython-3.12-linux-x86_64-gnu -> cpython-3.12.14-linux-x86_64-gnu
```

### Python modulefile 模板

精确版本文件示例：

```lua
help([[
CPython 3.12.14, installed centrally with uv.
Use uv or python -m venv for project dependencies.
]])

whatis("CPython 3.12.14 (central uv-managed build)")

family("python")

local root = "/acdiost/home/apps/python/uv-managed/cpython-3.12.14-linux-x86_64-gnu"

prepend_path("PATH", pathJoin(root, "bin"))
prepend_path("MANPATH", pathJoin(root, "share", "man"))

setenv("PYTHONNOUSERSITE", "1")
setenv("PYTHON_ROOT", root)
setenv("PYTHON_VERSION", "3.12.14")
```

文件放置为：

```text
/acdiost/home/apps/modulefiles/python/3.12.14.lua
```

短版本使用符号链接：

```bash
cd /acdiost/home/apps/modulefiles/python
ln -s 3.12.14.lua 3.12.lua
```

使用 `family("python")` 后，一次只能加载一个 Python。modulefile 不设置 `PYTHONHOME` 或 `PYTHONPATH`，因为这两个变量容易破坏 venv 的路径解析；`PYTHONNOUSERSITE=1` 用于阻止 `~/.local/lib/pythonX.Y/site-packages` 意外污染共享解释器。

### 用户工作流

```bash
module purge
module load python/3.12
module load uv/0.12.13

mkdir my-project
cd my-project

uv init
uv python pin 3.12
uv venv --python "$(command -v python)"
uv add numpy
uv run python -c 'import numpy; print(numpy.__version__)'
```

切换 Python 次版本后必须重建 `.venv`：

```bash
module swap python/3.12 python/3.11
uv python pin 3.11
rm -rf .venv
uv sync
```

删除 `.venv` 前要确认当前目录，不能对未解析变量或宽泛路径执行递归删除。

用户确实需要未共享的版本时：

```bash
module load uv/0.12.13
uv python install 3.13
uv python list
```

用户管理的解释器默认位于 `$HOME/.local/share/uv/python`。常用版本应继续加载共享 `python/*` 模块，避免重复安装。

### 版本选择

- Python 3.9 已停止安全维护，只用于无法迁移的旧项目。
- Python 3.11/3.12 是当前科学计算和 GPU 项目的优先选择。
- Python 3.13/3.14 上线项目前，应确认 NumPy、PyTorch、TensorFlow 和自定义扩展是否提供兼容 wheel。

不要对共享基础解释器执行 `pip install`，更不能执行 `sudo pip install`。共享 Python 只提供解释器、标准库和随发行版附带的 pip；项目库全部进入 `.venv`。

## CUDA 12.8 与 13.0.3

### CUDA 12.8 节点本地模块

gpu-a、gpu-b 节点均已有 `/usr/local/cuda-12.8`，抽样验证的 `nvcc` 为 `V12.8.93`。本次没有重复安装 Toolkit，只发布统一模块入口：

```text
/acdiost/home/apps/modulefiles/cuda/12.8.lua
/acdiost/home/apps/modulefiles/cuda/12.lua -> 12.8.lua
```

该模块依赖每个计算节点都有相同路径和兼容内容。新增节点时，必须先配置 `/usr/local/cuda-12.8`，再加入调度。切换版本使用 Lmod，不修改 `/usr/local/cuda`：

```bash
module purge
module load cuda/12.8
nvcc --version

module swap cuda/12.8 cuda/13.0.3
nvcc --version
```

`family("cuda")` 保证一个 shell 中只保留一个 CUDA 模块。

### CUDA 13.0.3 共享安装

#### 为什么使用共享安装

登录节点、gpu-a 和 gpu-b 的 `/usr/local` 都位于各自本地根文件系统。现有 CUDA 12.8 安装在每台机器的 `/usr/local/cuda-12.8`，而新增 CUDA 13 使用共享目录：

```text
/acdiost/home/apps/cuda/13.0.3
```

这样无需逐节点安装 Toolkit。驱动仍由节点操作系统提供。

#### 安装包与安装方式

```text
安装包
/acdiost/home/apps/packages/cuda_13.0.3_580.126.20_linux.run

大小
约 4.1 GiB

Toolkit 安装后
约 6.6 GiB
```

安装命令：

```bash
pkg=/acdiost/home/apps/packages/cuda_13.0.3_580.126.20_linux.run
dest=/acdiost/home/apps/cuda/13.0.3

"$pkg" \
  --silent \
  --toolkit \
  --toolkitpath="$dest" \
  --no-man-page
```

必须包含 `--toolkit`，不能包含 `--driver`。`--no-man-page` 避免非 root 安装写入 `/usr/share/man`。本次没有修改 `/usr/local/cuda`，它仍指向系统 CUDA 12.8。

#### CUDA modulefile

```lua
help([[
NVIDIA CUDA Toolkit 13.0.3 (toolkit-only shared installation).
]])

whatis("NVIDIA CUDA Toolkit 13.0.3")

family("cuda")

local root = "/acdiost/home/apps/cuda/13.0.3"

setenv("CUDA_HOME", root)
setenv("CUDA_PATH", root)
setenv("CUDA_ROOT", root)
setenv("CUDA_VERSION", "13.0.3")
setenv("CUDACXX", pathJoin(root, "bin", "nvcc"))

prepend_path("PATH", pathJoin(root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib64"))
prepend_path("LIBRARY_PATH", pathJoin(root, "lib64"))
prepend_path("CPATH", pathJoin(root, "include"))
prepend_path("CMAKE_PREFIX_PATH", root)
prepend_path("PKG_CONFIG_PATH", pathJoin(root, "lib64", "pkgconfig"))
```

文件及别名：

```text
/acdiost/home/apps/modulefiles/cuda/13.0.3.lua
/acdiost/home/apps/modulefiles/cuda/13.0.lua -> 13.0.3.lua
/acdiost/home/apps/modulefiles/cuda/13.lua   -> 13.0.3.lua
```

绝对不要将 `lib64/stubs` 加入 `LD_LIBRARY_PATH`。其中的 `libcuda.so` 是链接期桩库，不是可运行的 GPU 驱动库。

#### 驱动与硬件验证

CUDA 13.x 的基础驱动要求是 R580。安装前使用 Slurm 在每类节点上检查，而不是在没有 GPU 的登录节点执行 `nvidia-smi`：

```bash
srun -p gpu-a -N1 -n1 --gres=gpu:1 --time=00:01:00 \
  nvidia-smi --query-gpu=name,driver_version --format=csv,noheader

srun -p gpu-b -N1 -n1 --gres=gpu:1 --time=00:01:00 \
  nvidia-smi --query-gpu=name,driver_version --format=csv,noheader
```

本次两类节点均返回驱动 `580.173.02`。随后用同一个 fat binary 验证：

```bash
module load cuda/13

nvcc -O2 \
  -gencode arch=compute_89,code=sm_89 \
  -gencode arch=compute_120,code=sm_120 \
  cuda-smoke.cu -o cuda-smoke

srun -p gpu-a -N1 -n1 --gres=gpu:1 --time=00:01:00 ./cuda-smoke
srun -p gpu-b -N1 -n1 --gres=gpu:1 --time=00:01:00 ./cuda-smoke
```

RTX 4090 和 RTX 5090 均成功执行 kernel，并返回测试值 `13`。CUDA 13 模块卸载后，shell 中的 `nvcc` 会回落到系统全局 PATH 中的 `/usr/local/cuda-12.8/bin/nvcc`；这不是 CUDA 13 环境残留。

## CMake 4.4.3 与 Ninja 1.13.2

公共构建工具来自上游官方二进制发布包，安装在共享目录：

```text
/acdiost/home/apps/cmake/4.4.3
/acdiost/home/apps/ninja/1.13.2
```

发布的模块及短别名：

```text
cmake/4.4.3  cmake/4.4  cmake/4
ninja/1.13.2 ninja/1.13 ninja/1
```

本次下载包完成 SHA-256 校验：

```text
cmake-4.4.3-linux-x86_64.tar.gz
d6c83076c575bc00b823522ac974bda66d0af05d6ddc30e739c12385cf32c6cc

ninja-linux.zip 1.13.2
5749cbc4e668273514150a80e387a957f933c6ed3f5f11e03fb30955e2bbead6
```

GCC 13.3 下的 CMake + Ninja 最小 C 项目已完成配置、编译和运行验证。用户通常这样加载：

```bash
module purge
module load cmake/4.4.3 ninja/1.13.2
cmake --version
ninja --version
```

生产构建脚本应加载精确版本，短别名主要用于交互环境。

## GCC/GFortran、OpenBLAS 与 C++ 基础库

### GCC 13.3.0 系统工具链

登录节点、8 台 gpu-a 和 4 台 gpu-b 使用相同的 Ubuntu GCC/G++ 13.3.0。为补齐 Fortran，所有 13 台机器安装了以下系统包：

```text
gfortran-13                    13.3.0-6ubuntu2~24.04.1
gfortran-13-x86-64-linux-gnu  13.3.0-6ubuntu2~24.04.1
libgfortran-13-dev             13.3.0-6ubuntu2~24.04.1
libgfortran5                   14.2.0-4ubuntu2~24.04.1
```

安装过程没有升级或删除其他包。Lmod 模块提供统一编译器变量：

```bash
module purge
module load gcc/13.3.0-system

echo "$CC"   # /usr/bin/gcc-13
echo "$CXX"  # /usr/bin/g++-13
echo "$FC"   # /usr/bin/gfortran-13
```

模块及别名：

```text
gcc/13.3.0-system
gcc/13.3 -> 13.3.0-system
gcc/13   -> 13.3.0-system
```

该模块使用 `family("compiler")`，并主动冲突 NVHPC 模块。现有 NVHPC modulefile 没有声明相同 family，因此切换编译器时仍应先执行 `module purge`，不要在同一 shell 中叠加 GCC 与 NVHPC。

### OpenBLAS 0.3.34

安装目录和模块：

```text
/acdiost/home/apps/openblas/0.3.34-gcc13-openmp
openblas/0.3.34-gcc13-openmp
openblas/0.3.34 -> 0.3.34-gcc13-openmp
openblas/0.3    -> 0.3.34-gcc13-openmp
```

构建特性：

```text
DYNAMIC_ARCH=1
USE_OPENMP=1
USE_THREAD=1
NO_AFFINITY=1
CC=gcc-13
FC=gfortran-13
```

这套构建包含 BLAS、CBLAS、LAPACK 与 LAPACKE。`NO_AFFINITY=1` 防止 OpenBLAS 自行绑核干扰 Slurm。模块不会设置线程数，作业脚本应按申请到的 CPU 设置：

```bash
module purge
module load gcc/13 openblas/0.3

export OMP_NUM_THREADS="$SLURM_CPUS_PER_TASK"
export OMP_PROC_BIND=close
export OMP_PLACES=cores
./application
```

不要同时设置相互矛盾的 `OPENBLAS_NUM_THREADS` 和 `OMP_NUM_THREADS`。本版本采用 OpenMP，优先统一使用 `OMP_NUM_THREADS`。

源码包及本次计算的校验值：

```text
/acdiost/home/apps/packages/OpenBLAS-0.3.34.tar.gz
cd7e129868320cc2d033afa920e31202dfe0b8066a5b66661900ccc0f197dfed
```

OpenBLAS 通过其 BLAS/CBLAS 测试集和 126 项单元测试。运行时动态内核选择验证结果：

```text
登录节点  SkylakeX
gpu-a 抽样   Zen
gpu-b 抽样   Cooperlake
```

### Eigen、fmt 与 spdlog

| 模块 | 安装内容 | 依赖关系 |
| --- | --- | --- |
| `eigen/3.4.1` | Eigen 3.4.1 头文件和 CMake/pkg-config 元数据 | 无 |
| `fmt/12.2.0-gcc13` | fmt 12.2.0 共享库 | GCC 13 ABI |
| `spdlog/1.17.0-gcc13-fmt12.2` | spdlog 1.17.0 共享库 | 自动加载 `fmt/12.2.0-gcc13` |

对应短别名为 `eigen/3`、`fmt/12` 和 `spdlog/1`；生产构建仍建议使用表中的精确名称。

归档 SHA-256：

```text
eigen-3.4.1.tar.gz
b93c667d1b69265cdb4d9f30ec21f8facbbe8b307cf34c0b9942834c6d4fdbe2

fmt-12.2.0.tar.gz
8b852bb5aa6e7d8564f9e81394055395dd1d1936d38dfd3a17792a02bebd7af0

spdlog-1.17.0.tar.gz
d8862955c6d74e5846b3f580b1605d2428b11d97a410d86e2fb13e857cd3a744
```

fmt 通过 21/21 项上游测试。spdlog 的上游测试会在线拉取 Catch2，集群外网克隆被中止；随后关闭该在线测试依赖完成构建，并用实际 C++ 程序同时验证 Eigen 行列式、fmt 格式化和 spdlog 输出。`libspdlog.so` 已确认动态链接共享 `libfmt.so.12`，没有混入内置 fmt 副本。

使用示例：

```bash
module purge
module load gcc/13 eigen/3.4.1 spdlog/1.17.0-gcc13-fmt12.2

pkg-config --cflags eigen3 spdlog fmt
pkg-config --libs spdlog fmt
```

## HDF5 2.2.0 与 NetCDF-C 4.10.1

### ABI 变体命名

串行库和 HPC-X 并行库使用不同安装目录与模块名，防止链接时无意混用 MPI ABI：

| 模块 | 能力 | 自动依赖 |
| --- | --- | --- |
| `hdf5/2.2.0-gcc13-zlib` | C/C++、HL、zlib，串行 | 无 |
| `hdf5/2.2.0-gcc13-hpcx2.51-zlib` | C、HL、zlib、Parallel HDF5 | `hpcx/2.51` |
| `netcdf-c/4.10.1-gcc13` | NetCDF-4/HDF5，串行 | 串行 HDF5 zlib 变体 |
| `netcdf-c/4.10.1-gcc13-hpcx2.51` | NetCDF-4、parallel4 | 并行 HDF5 zlib 变体 |

Fortran bindings、DAP、NCZarr、S3、HDF4 和 PnetCDF 没有包含在本轮构建中。DAP/NCZarr/S3 需要补齐 curl、libxml2 等依赖后再作为单独变体发布，不应悄悄改变现有模块功能。

### 为什么保留两组 HDF5 2.2.0

最初构建的以下两个模块未启用 zlib：

```text
hdf5/2.2.0-gcc13
hdf5/2.2.0-gcc13-hpcx2.51
```

NetCDF-C 配置阶段明确拒绝无 zlib 的 HDF5。为遵守共享软件“不可原地覆盖”的原则，本次新增 `-zlib` 目录和模块，没有修改或删除原目录。新项目和 NetCDF-C 应使用 `-zlib` 变体；旧变体仅在应用明确不需要 deflate 时使用。

HDF5 源码包校验值：

```text
/acdiost/home/apps/packages/hdf5-2.2.0.tar.gz
1a1ab8209b35586fbc1aa279ba76d102130b95badcb20ca329587219112d8c16
```

zlib 变体的最终头文件和动态链接均已验证：

```text
H5_HAVE_ZLIB_H=1
H5_HAVE_FILTER_DEFLATE=1
libhdf5.so -> libz.so.1
```

### 用户使用方式

串行应用：

```bash
module purge
module load netcdf-c/4.10.1-gcc13

nc-config --has-nc4       # yes
nc-config --has-parallel4 # no
cc app.c $(nc-config --cflags) $(nc-config --libs)
```

并行应用：

```bash
module purge
module load netcdf-c/4.10.1-gcc13-hpcx2.51

nc-config --has-nc4       # yes
nc-config --has-parallel4 # yes
mpicc app.c $(nc-config --cflags) $(nc-config --libs)
```

加载 NetCDF-C 会自动加载匹配的 HDF5；并行变体还会间接加载 `hpcx/2.51`。不要再手工叠加另一种 HDF5。`family("hdf5")` 和 `family("netcdf")` 会在 `module swap` 时同步切换相应变体。

### 验证记录与边界

- HDF5 串行初始变体通过 2569 项启用测试；37 项由构建配置禁用。
- NetCDF-C 串行变体通过 199/199 项测试。
- NetCDF-C 并行变体在关闭上游 MPI 多进程测试后，通过其余 199/199 项测试；`nc-config --has-parallel4` 返回 `yes`。
- `nc_create_par` 已成功创建并关闭有效 HDF5/NetCDF 文件。

登录节点上的 HPC-X 默认 UCX 无法完成本机多进程初始化；强制 `ob1` 后并行 I/O 能完成，但两进程作业卡在 `MPI_Finalize`。因此这不是跨节点生产验收。获得有效 Slurm 账户/分区后，必须在计算节点补做：

```bash
srun -p <有效分区> -N2 -n2 --time=00:05:00 bash -lc '
  module purge
  module load netcdf-c/4.10.1-gcc13-hpcx2.51
  ./netcdf-parallel-smoke
'
```

### Fortran 编译器已补齐，库绑定待发布

`gfortran-13` 现已在登录节点、gpu-a 和 gpu-b 全量部署，`gcc/13.3.0-system` 模块也已发布。HPC-X 的 `mpifort` 可以使用系统 GFortran。现有 HDF5/NetCDF 模块仍是原来的 C/C++ 变体，没有被原地覆盖。

下一步应使用新目录新增 HDF5 Fortran 与 NetCDF-Fortran 变体，模块名同时携带 GCC、HPC-X 和 Fortran 信息。不要修改现有 `hdf5/2.2.0-*-zlib` 和 `netcdf-c/4.10.1-*` 目录。

## Apptainer 暂缓事项

当前系统设置 `kernel.apparmor_restrict_unprivileged_userns=1`，`apps` 用户执行 rootless user namespace 会失败。直接改 AppArmor、启用 setuid 安装或放宽所有节点策略都属于集群安全决策，本次没有执行。

安装 Apptainer 前需要先确定一种模式：

1. rootless：通过受控 AppArmor profile 允许 Apptainer 使用 user namespace，并在所有计算节点验证；
2. setuid：采用官方推荐的特权安装布局，完成安全审计并限制可写路径；
3. 调度隔离：仅在指定容器分区/节点启用，先做试点。

策略确定后才能发布 `apptainer/<版本>` 模块。不能只在登录节点安装并宣称集群可用。

## 下一批公共库候选

按复用率和 ABI 风险排序，建议后续分批加入：

1. HDF5 Fortran 与 NetCDF-Fortran：GFortran 阻塞已解除，应新增串行和 HPC-X 并行变体。
2. Boost 与 protobuf：C++ ABI 影响较大，按真实应用需求选择版本并进行依赖闭包测试。
3. BLIS：仅在需要与 OpenBLAS 做性能对照时加入，避免默认提供多个 BLAS 造成链接歧义。
4. hwloc、PMIx、libfabric：仅在明确需要自建 MPI 栈时加入，避免与 HPC-X 自带组件混用。
5. oneAPI MKL/oneMKL：需先确认许可、CPU 平台与线程运行时策略。
6. GDAL、PROJ、GEOS：地学用户有明确需求时成套构建，避免系统库与共享库交叉链接。

Python 生态中的 NumPy、SciPy、pandas、JupyterLab、PyTorch、TensorFlow、JAX、CuPy、Transformers 等继续由各项目的 uv 环境管理，不创建全局共享 site-packages。

## Slurm 作业模板

### 普通 Python 项目

```bash
#!/bin/bash
#SBATCH --job-name=python-job
#SBATCH --partition=gpu-a
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G

set -euo pipefail

module purge
module load python/3.12.14
module load uv/0.12.13

cd /path/to/project
uv run --frozen --no-sync python train.py
```

提交作业前在登录节点执行一次：

```bash
module load python/3.12.14 uv/0.12.13
cd /path/to/project
uv sync --frozen
```

作业使用 `--no-sync`，防止多个任务同时修改共享 `.venv`。

### 编译 CUDA 扩展

```bash
module purge
module load python/3.12.14
module load uv/0.12.13
module load cuda/13.0.3

cd /path/to/project
uv sync --frozen
uv run python -m build
```

如果只是运行官方 PyTorch wheel，而不编译 CUDA 扩展，通常不需要加载 `cuda/*`。wheel 自带的 CUDA 用户态库版本与系统 Toolkit 可以不同；最终兼容边界由节点驱动和 wheel 决定。

## 验收清单

每次新增或更新模块后，至少执行以下检查：

```bash
module -t avail hpcx uv python cuda cmake ninja gcc openblas eigen fmt spdlog hdf5 netcdf-c
module spider <模块>/<版本>
module show <模块>/<版本>
```

Python 与 uv：

```bash
module purge
module load python/3.12 uv/0.12.13

python --version
pip --version
uv --version
uv python find 3.12 --system

testdir=$(mktemp -d /tmp/uv-test.XXXXXX)
uv venv --python "$(command -v python)" "$testdir/.venv"
"$testdir/.venv/bin/python" --version
```

HPC-X：

```bash
module purge
module load hpcx/2.51
mpirun --version
ucx_info -v
module unload hpcx/2.51
```

CUDA 必须同时验证编译和真实 GPU 运行：

```bash
module purge
module load cuda/13.0.3
nvcc --version

srun -p gpu-a -N1 -n1 --gres=gpu:1 --time=00:01:00 ./cuda-smoke
srun -p gpu-b -N1 -n1 --gres=gpu:1 --time=00:01:00 ./cuda-smoke
```

公共构建与数据格式库：

```bash
module purge
module load cmake/4.4.3 ninja/1.13.2
cmake --version
ninja --version

module purge
module load netcdf-c/4.10.1-gcc13
nc-config --has-nc4
nc-config --has-parallel4

module swap netcdf-c/4.10.1-gcc13 netcdf-c/4.10.1-gcc13-hpcx2.51
nc-config --has-parallel4
h5pcc -showconfig | grep 'Parallel HDF5'
```

GFortran、OpenBLAS 与 C++ 公共库：

```bash
module purge
module load gcc/13 openblas/0.3
gfortran-13 --version
pkg-config --modversion openblas

module purge
module load gcc/13 eigen/3 spdlog/1
pkg-config --modversion eigen3 fmt spdlog
ldd "$SPDLOG_ROOT/lib/libspdlog.so" | grep libfmt
```

检查卸载时，验证模块设置的变量消失、PATH 回到加载前状态。不要简单地以“还能找到 python/nvcc”为失败依据，因为系统 PATH 可能有后备版本。

## 升级与回滚

### 不原地覆盖

所有升级使用新目录和新 modulefile：

```text
/acdiost/home/apps/uv/<新版本>
/acdiost/home/apps/python/uv-managed/cpython-<精确版本>-linux-x86_64-gnu
/acdiost/home/apps/cuda/<新版本>
/acdiost/home/apps/openblas/<版本>-<编译器>-<线程模型>
/acdiost/home/apps/eigen/<版本>
/acdiost/home/apps/fmt/<版本>-<编译器>
/acdiost/home/apps/spdlog/<版本>-<编译器>-<fmt版本>
/acdiost/home/apps/hdf5/<版本>-<编译器>-<功能变体>
/acdiost/home/apps/netcdf-c/<版本>-<编译器>-<MPI变体>
/acdiost/home/apps/modulefiles/<软件>/<新版本>.lua
```

先发布精确版本并完成验证，再调整短版本符号链接。正在运行的生产脚本应加载精确版本，不依赖短别名。

### 回滚短别名

若新版本验证失败，将短别名恢复到旧 modulefile 即可。例如：

```bash
cd /acdiost/home/apps/modulefiles/cuda
ln -sfn 13.0.3.lua 13.lua
```

执行前必须用 `readlink -f` 确认当前目标，并保留旧精确版本文件。不要删除仍可能被作业引用的安装目录。

### 清理策略

1. 先查询 Slurm 历史和团队使用情况。
2. 取消短别名，不立即删除精确版本。
3. 保留至少一个迁移窗口。
4. 确认无脚本、modulefile 或软链接引用后再删除。
5. CUDA 大型安装包是否保留由软件归档策略决定；删除前确认可重新取得并保存校验信息。

## 日常快速检查

```bash
module -t avail hpcx uv python cuda cmake ninja gcc openblas eigen fmt spdlog hdf5 netcdf-c

module load python/3.12 uv/0.12.13
python --version
uv --version
uv cache dir

module purge
module load cuda/13
nvcc --version

module purge
module load netcdf-c/4.10.1-gcc13
nc-config --all

sinfo -h -o '%P|%N|%G|%t'
```

出现环境异常时先执行 `module purge`，再只加载复现问题所必需的模块。记录 `module -t list`、`env | sort`、解释器/编译器绝对路径和 Slurm 节点名，比只记录版本号更容易定位路径污染。
