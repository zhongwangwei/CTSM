# CTSM 从头安装完整操作手册

## 目录

1. [概述](#1-概述)
2. [系统要求](#2-系统要求)
3. [依赖软件安装](#3-依赖软件安装)
   - [3.1 编译器安装](#31-编译器安装)
   - [3.2 MPI 安装](#32-mpi安装)
   - [3.3 HDF5 安装](#33-hdf5安装)
   - [3.4 NetCDF-C 安装](#34-netcdf-c安装)
   - [3.5 NetCDF-Fortran 安装](#35-netcdf-fortran安装)
   - [3.6 PnetCDF 安装（可选）](#36-pnetcdf安装可选)
   - [3.7 PIO 安装](#37-pio安装)
   - [3.8 ESMF 安装](#38-esmf安装)
4. [Python 环境配置](#4-python环境配置)
5. [CTSM 获取与配置](#5-ctsm获取与配置)
6. [创建和运行案例](#6-创建和运行案例)
7. [常见问题排查](#7-常见问题排查)
8. [参考资源](#8-参考资源)

---

## 1. 概述

CTSM（Community Terrestrial Systems Model）是社区陆面系统模型，是 CESM（Community Earth System Model）的陆面组件。本手册提供从零开始的完整安装指南，包括所有依赖库的编译安装。

**CTSM 5.3 系列主要特点：**
- 使用 NUOPC/ESMF 驱动框架
- 使用 CMEPS（Community Mediator for Earth Prediction Systems）作为耦合器
- 使用 CDEPS（Community Data Models for Earth Prediction System）提供大气强迫数据
- 使用 `git-fleximod` 管理子模块（替代了旧的 `manage_externals`）

---

## 2. 系统要求

### 2.1 硬件要求
- **内存**：至少 8GB RAM（建议 16GB 以上）
- **磁盘空间**：至少 50GB 可用空间
- **处理器**：多核 CPU（建议 4 核以上）

### 2.2 操作系统
- Linux（推荐 CentOS 7+、Ubuntu 18.04+、RHEL 7+）
- macOS（需要 Xcode Command Line Tools）

### 2.3 软件版本要求

| 软件 | 最低版本 | 推荐版本 |
|------|----------|----------|
| GCC/gfortran | 8.0 | 10.0+ |
| CMake | 3.10 | 3.20+ |
| GNU Make | 3.8 | 4.0+ |
| MPI (OpenMPI/MPICH) | 3.0 | 4.0+ |
| HDF5 | 1.10 | 1.12+ |
| NetCDF-C | 4.7.4 | 4.9+ |
| NetCDF-Fortran | 4.5 | 4.6+ |
| ESMF | 8.4 | 8.6+ |
| Python | 3.9 | 3.13 |

---

## 3. 依赖软件安装

### 3.0 准备工作

首先创建安装目录结构：

```bash
# 设置安装根目录（可根据需要修改）
export INSTALL_ROOT=/opt/ctsm-libs
export BUILD_ROOT=/tmp/ctsm-build

# 创建目录
sudo mkdir -p $INSTALL_ROOT
sudo chown $USER:$USER $INSTALL_ROOT
mkdir -p $BUILD_ROOT

# 设置并行编译核心数
export MAKE_JOBS=$(nproc)
```

### 3.1 编译器安装

#### 3.1.1 Ubuntu/Debian 系统

```bash
# 安装 GCC、G++、GFortran
sudo apt update
sudo apt install -y build-essential gfortran

# 安装其他必需工具
sudo apt install -y cmake git wget curl m4 zlib1g-dev libcurl4-openssl-dev

# 验证安装
gcc --version
gfortran --version
cmake --version
```

#### 3.1.2 CentOS/RHEL 系统

```bash
# 安装开发工具组
sudo yum groupinstall -y "Development Tools"

# 安装 GFortran 和其他工具
sudo yum install -y gcc-gfortran cmake3 git wget curl m4 zlib-devel libcurl-devel

# 在 CentOS 8+ 使用 dnf
sudo dnf install -y gcc gcc-c++ gcc-gfortran cmake git wget curl m4 zlib-devel libcurl-devel

# 验证安装
gcc --version
gfortran --version
```

#### 3.1.3 macOS 系统

```bash
# 安装 Xcode 命令行工具
xcode-select --install

# 使用 Homebrew 安装 GFortran
brew install gcc cmake wget

# 验证安装
gfortran --version
```

### 3.2 MPI安装

MPI（Message Passing Interface）用于并行计算。推荐使用 OpenMPI 或 MPICH。

#### 3.2.1 OpenMPI 源码安装

```bash
cd $BUILD_ROOT

# 下载 OpenMPI（使用稳定版本）
wget https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.6.tar.gz
tar xzf openmpi-4.1.6.tar.gz
cd openmpi-4.1.6

# 配置（启用 Fortran 支持）
./configure --prefix=$INSTALL_ROOT/openmpi-4.1.6 \
    --enable-mpi-fortran=all \
    --enable-mpi-cxx \
    --enable-shared \
    --with-zlib

# 编译安装
make -j $MAKE_JOBS
make install

# 设置环境变量
export MPI_ROOT=$INSTALL_ROOT/openmpi-4.1.6
export PATH=$MPI_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPI_ROOT/lib:$LD_LIBRARY_PATH
export MANPATH=$MPI_ROOT/share/man:$MANPATH
```

#### 3.2.2 MPICH 源码安装（替代方案）

```bash
cd $BUILD_ROOT

# 下载 MPICH
wget https://www.mpich.org/static/downloads/4.1.2/mpich-4.1.2.tar.gz
tar xzf mpich-4.1.2.tar.gz
cd mpich-4.1.2

# 配置
./configure --prefix=$INSTALL_ROOT/mpich-4.1.2 \
    --enable-fortran=all \
    --enable-cxx \
    --enable-shared

# 编译安装
make -j $MAKE_JOBS
make install

# 设置环境变量
export MPI_ROOT=$INSTALL_ROOT/mpich-4.1.2
export PATH=$MPI_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPI_ROOT/lib:$LD_LIBRARY_PATH
```

#### 3.2.3 验证 MPI 安装

```bash
# 检查版本
mpicc --version
mpif90 --version
mpirun --version

# 运行简单测试
cat > test_mpi.f90 << 'EOF'
program test_mpi
  use mpi
  implicit none
  integer :: ierr, rank, size
  call MPI_Init(ierr)
  call MPI_Comm_rank(MPI_COMM_WORLD, rank, ierr)
  call MPI_Comm_size(MPI_COMM_WORLD, size, ierr)
  print *, 'Hello from rank', rank, 'of', size
  call MPI_Finalize(ierr)
end program
EOF

mpif90 test_mpi.f90 -o test_mpi
mpirun -np 4 ./test_mpi
rm test_mpi test_mpi.f90
```

### 3.3 HDF5安装

HDF5 是 NetCDF-4 的底层存储格式库。

```bash
cd $BUILD_ROOT

# 下载 HDF5
wget https://github.com/HDFGroup/hdf5/releases/download/hdf5_1.14.4.3/hdf5-1.14.4-3.tar.gz
tar xzf hdf5-1.14.4-3.tar.gz
cd hdf5-1.14.4-3

# 配置（启用并行和 Fortran 支持）
CC=mpicc FC=mpif90 ./configure --prefix=$INSTALL_ROOT/hdf5-1.14.4 \
    --enable-fortran \
    --enable-parallel \
    --enable-shared \
    --with-zlib

# 编译安装
make -j $MAKE_JOBS
make install

# 设置环境变量
export HDF5_ROOT=$INSTALL_ROOT/hdf5-1.14.4
export PATH=$HDF5_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$HDF5_ROOT/lib:$LD_LIBRARY_PATH
```

#### 验证 HDF5 安装

```bash
# 检查版本
h5dump --version

# 检查并行支持
h5pcc -showconfig | grep -i parallel
```

### 3.4 NetCDF-C安装

NetCDF（Network Common Data Form）是 CTSM 的核心数据格式。

```bash
cd $BUILD_ROOT

# 下载 NetCDF-C
wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar xzf netcdf-c-4.9.2.tar.gz
cd netcdf-c-4.9.2

# 设置依赖路径
export CPPFLAGS="-I$HDF5_ROOT/include"
export LDFLAGS="-L$HDF5_ROOT/lib"

# 配置
CC=mpicc ./configure --prefix=$INSTALL_ROOT/netcdf-c-4.9.2 \
    --enable-netcdf-4 \
    --enable-shared \
    --enable-parallel-tests \
    --disable-dap

# 编译安装
make -j $MAKE_JOBS
make check  # 运行测试（可选但推荐）
make install

# 设置环境变量
export NETCDF_C_ROOT=$INSTALL_ROOT/netcdf-c-4.9.2
export PATH=$NETCDF_C_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF_C_ROOT/lib:$LD_LIBRARY_PATH
```

#### 验证 NetCDF-C 安装

```bash
# 检查版本和配置
nc-config --version
nc-config --all

# 确认 NetCDF-4 和并行支持
nc-config --has-nc4    # 应显示 yes
nc-config --has-parallel4  # 应显示 yes
```

### 3.5 NetCDF-Fortran安装

NetCDF-Fortran 提供 Fortran 接口，是 CTSM 所必需的。

```bash
cd $BUILD_ROOT

# 下载 NetCDF-Fortran
wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/netcdf-fortran-4.6.1.tar.gz
tar xzf netcdf-fortran-4.6.1.tar.gz
cd netcdf-fortran-4.6.1

# 设置依赖路径
export CPPFLAGS="-I$NETCDF_C_ROOT/include -I$HDF5_ROOT/include"
export LDFLAGS="-L$NETCDF_C_ROOT/lib -L$HDF5_ROOT/lib"
export LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz"

# 配置
CC=mpicc FC=mpif90 F77=mpif77 ./configure --prefix=$INSTALL_ROOT/netcdf-fortran-4.6.1 \
    --enable-shared

# 编译安装
make -j $MAKE_JOBS
make check  # 运行测试（可选但推荐）
make install

# 设置环境变量
export NETCDF_FORTRAN_ROOT=$INSTALL_ROOT/netcdf-fortran-4.6.1
export PATH=$NETCDF_FORTRAN_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF_FORTRAN_ROOT/lib:$LD_LIBRARY_PATH

# 设置统一的 NETCDF 变量（CTSM 使用）
export NETCDF=$INSTALL_ROOT/netcdf
mkdir -p $NETCDF/lib $NETCDF/include $NETCDF/bin
ln -sf $NETCDF_C_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_C_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_C_ROOT/bin/* $NETCDF/bin/
ln -sf $NETCDF_FORTRAN_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_FORTRAN_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_FORTRAN_ROOT/bin/* $NETCDF/bin/
```

#### 验证 NetCDF-Fortran 安装

```bash
# 检查版本
nf-config --version
nf-config --all

# 测试 Fortran 程序
cat > test_netcdf.f90 << 'EOF'
program test_netcdf
  use netcdf
  implicit none
  integer :: ncid, status
  print *, 'NetCDF-Fortran version: ', nf90_inq_libvers()
  status = nf90_create('test.nc', NF90_CLOBBER, ncid)
  if (status == NF90_NOERR) then
    print *, 'Successfully created test.nc'
    status = nf90_close(ncid)
  end if
end program
EOF

mpif90 test_netcdf.f90 -o test_netcdf \
    -I$NETCDF/include -L$NETCDF/lib -lnetcdff -lnetcdf
./test_netcdf
rm -f test_netcdf test_netcdf.f90 test.nc
```

### 3.6 PnetCDF安装（可选）

PnetCDF 提供额外的并行 I/O 支持，可以提高大规模模拟的 I/O 性能。

```bash
cd $BUILD_ROOT

# 下载 PnetCDF
wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz
tar xzf pnetcdf-1.12.3.tar.gz
cd pnetcdf-1.12.3

# 配置
CC=mpicc FC=mpif90 CXX=mpicxx ./configure --prefix=$INSTALL_ROOT/pnetcdf-1.12.3 \
    --enable-shared \
    --enable-fortran

# 编译安装
make -j $MAKE_JOBS
make install

# 设置环境变量
export PNETCDF_ROOT=$INSTALL_ROOT/pnetcdf-1.12.3
export PATH=$PNETCDF_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$PNETCDF_ROOT/lib:$LD_LIBRARY_PATH
```

### 3.7 PIO安装

PIO（Parallel I/O）是 CESM/CTSM 的并行 I/O 库。

```bash
cd $BUILD_ROOT

# 下载 PIO（从 CESM 的 PIO 仓库）
git clone https://github.com/NCAR/ParallelIO.git pio
cd pio
git checkout pio2_6_2  # 使用稳定版本

# 创建构建目录
mkdir build && cd build

# 配置 CMake
CC=mpicc FC=mpif90 cmake .. \
    -DCMAKE_INSTALL_PREFIX=$INSTALL_ROOT/pio-2.6.2 \
    -DPIO_ENABLE_FORTRAN=ON \
    -DPIO_ENABLE_TIMING=OFF \
    -DNetCDF_C_PATH=$NETCDF_C_ROOT \
    -DNetCDF_Fortran_PATH=$NETCDF_FORTRAN_ROOT \
    -DHDF5_PATH=$HDF5_ROOT \
    -DWITH_PNETCDF=OFF  # 如果安装了 PnetCDF，改为 ON 并添加路径

# 编译安装
make -j $MAKE_JOBS
make install

# 设置环境变量
export PIO_ROOT=$INSTALL_ROOT/pio-2.6.2
export LD_LIBRARY_PATH=$PIO_ROOT/lib:$LD_LIBRARY_PATH
```

### 3.8 ESMF安装

ESMF（Earth System Modeling Framework）是 NUOPC 驱动的基础框架。

```bash
cd $BUILD_ROOT

# 下载 ESMF
git clone https://github.com/esmf-org/esmf.git
cd esmf
git checkout v8.6.1  # 使用稳定版本

# 设置 ESMF 环境变量
export ESMF_DIR=$PWD
export ESMF_INSTALL_PREFIX=$INSTALL_ROOT/esmf-8.6.1
export ESMF_COMM=openmpi  # 或 mpich，取决于你安装的 MPI
export ESMF_COMPILER=gfortran
export ESMF_NETCDF=nc-config
export ESMF_NETCDF_INCLUDE=$NETCDF/include
export ESMF_NETCDF_LIBPATH=$NETCDF/lib
export ESMF_PIO=external
export ESMF_PIO_LIBPATH=$PIO_ROOT/lib
export ESMF_PIO_INCLUDE=$PIO_ROOT/include

# 编译
make -j $MAKE_JOBS

# 安装
make install

# 设置运行时环境变量
export ESMF_ROOT=$INSTALL_ROOT/esmf-8.6.1
export ESMFMKFILE=$ESMF_ROOT/lib/libg/Linux.gfortran.64.openmpi.default/esmf.mk
export PATH=$ESMF_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$ESMF_ROOT/lib:$LD_LIBRARY_PATH
```

#### 验证 ESMF 安装

```bash
# 运行 ESMF 系统测试
cd $ESMF_DIR
make check

# 检查 esmf.mk 文件
cat $ESMFMKFILE | head -20
```

### 3.9 创建环境配置文件

将所有环境变量保存到配置文件，便于后续使用：

```bash
cat > $INSTALL_ROOT/ctsm-env.sh << 'EOF'
#!/bin/bash
# CTSM 依赖库环境配置

export INSTALL_ROOT=/opt/ctsm-libs

# MPI
export MPI_ROOT=$INSTALL_ROOT/openmpi-4.1.6
export PATH=$MPI_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPI_ROOT/lib:$LD_LIBRARY_PATH
export MANPATH=$MPI_ROOT/share/man:$MANPATH

# HDF5
export HDF5_ROOT=$INSTALL_ROOT/hdf5-1.14.4
export PATH=$HDF5_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$HDF5_ROOT/lib:$LD_LIBRARY_PATH

# NetCDF
export NETCDF_C_ROOT=$INSTALL_ROOT/netcdf-c-4.9.2
export NETCDF_FORTRAN_ROOT=$INSTALL_ROOT/netcdf-fortran-4.6.1
export NETCDF=$INSTALL_ROOT/netcdf
export PATH=$NETCDF/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF/lib:$LD_LIBRARY_PATH

# PnetCDF (可选)
# export PNETCDF_ROOT=$INSTALL_ROOT/pnetcdf-1.12.3
# export PATH=$PNETCDF_ROOT/bin:$PATH
# export LD_LIBRARY_PATH=$PNETCDF_ROOT/lib:$LD_LIBRARY_PATH

# PIO
export PIO_ROOT=$INSTALL_ROOT/pio-2.6.2
export LD_LIBRARY_PATH=$PIO_ROOT/lib:$LD_LIBRARY_PATH

# ESMF
export ESMF_ROOT=$INSTALL_ROOT/esmf-8.6.1
export ESMFMKFILE=$ESMF_ROOT/lib/libg/Linux.gfortran.64.openmpi.default/esmf.mk
export PATH=$ESMF_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$ESMF_ROOT/lib:$LD_LIBRARY_PATH

echo "CTSM environment loaded successfully"
echo "  MPI: $(mpirun --version 2>&1 | head -1)"
echo "  NetCDF: $(nc-config --version)"
echo "  ESMF: $ESMF_ROOT"
EOF

chmod +x $INSTALL_ROOT/ctsm-env.sh
```

使用方法：
```bash
source /opt/ctsm-libs/ctsm-env.sh
```

---

## 4. Python环境配置

CTSM 的工具和测试需要 Python 环境。

### 4.1 安装 Miniconda/Anaconda

```bash
# 下载 Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# 安装
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3

# 初始化
$HOME/miniconda3/bin/conda init bash
source ~/.bashrc

# 验证
conda --version
```

### 4.2 创建 CTSM Python 环境

CTSM 提供了自动创建 Python 环境的脚本：

```bash
cd /path/to/CTSM

# 使用 CTSM 提供的脚本创建环境
./py_env_create

# 或指定使用 mamba（更快）
./py_env_create --mamba

# 激活环境
conda activate ctsm_pylib
```

### 4.3 手动创建环境（备选方案）

如果自动脚本不工作，可以手动创建：

```bash
# 创建环境
conda create -n ctsm_pylib python=3.13

# 激活环境
conda activate ctsm_pylib

# 安装依赖
conda install -c conda-forge \
    dask xarray tqdm scipy netcdf4 requests \
    xesmf numba pylint black cartopy matplotlib \
    pytest coverage

# 安装文档构建工具
pip install sphinx sphinx_rtd_theme rst2pdf sphinxcontrib_programoutput sphinx-mdinclude
```

### 4.4 验证 Python 环境

```bash
conda activate ctsm_pylib

# 检查关键包
python -c "import netCDF4; print('netCDF4:', netCDF4.__version__)"
python -c "import xarray; print('xarray:', xarray.__version__)"
python -c "import dask; print('dask:', dask.__version__)"
```

---

## 5. CTSM获取与配置

### 5.1 克隆 CTSM 仓库

```bash
# 克隆仓库
git clone https://github.com/ESCOMP/CTSM.git my_ctsm
cd my_ctsm

# 查看可用版本
git tag | grep ctsm

# 切换到特定版本（可选）
git checkout ctsm5.3.0

# 获取所有子模块
./bin/git-fleximod update
```

### 5.2 子模块说明

`git-fleximod update` 会下载以下组件：

| 组件 | 说明 |
|------|------|
| cime | CESM 基础设施代码 |
| ccs_config | 配置文件（网格、compsets、机器） |
| cmeps | 耦合器（NUOPC 驱动） |
| cdeps | 数据模型 |
| share | 共享代码 |
| fates | 生态系统模型 |
| cism | 冰盖模型 |
| mosart | 河流传输模型 |
| mizuRoute | 河流路由模型 |
| rtm | 河流传输模型 |
| parallelio | 并行 I/O 库 |
| mpi-serial | 串行 MPI 库 |

### 5.3 配置机器设置

对于非 NCAR 支持的机器，需要创建机器配置文件。

#### 5.3.1 检查现有支持的机器

```bash
cd cime/scripts
./query_config --machines
```

#### 5.3.2 创建新机器配置

在 `$HOME/.cime/` 目录下创建配置文件：

```bash
mkdir -p $HOME/.cime
```

创建 `config_machines.xml`：

```xml
<?xml version="1.0"?>
<config_machines version="2.0">
  <machine MACH="my_machine">
    <DESC>My local Linux machine</DESC>
    <OS>LINUX</OS>
    <COMPILERS>gnu</COMPILERS>
    <MPILIBS>openmpi</MPILIBS>
    <NODENAME_REGEX>my_machine.*</NODENAME_REGEX>
    <CIME_OUTPUT_ROOT>/scratch/ctsm_output</CIME_OUTPUT_ROOT>
    <DIN_LOC_ROOT>/data/cesm/inputdata</DIN_LOC_ROOT>
    <DIN_LOC_ROOT_CLMFORC>/data/cesm/inputdata/atm/datm7</DIN_LOC_ROOT_CLMFORC>
    <DOUT_S_ROOT>/scratch/ctsm_archive</DOUT_S_ROOT>
    <BASELINE_ROOT>/scratch/ctsm_baseline</BASELINE_ROOT>
    <CCSM_CPRNC>/usr/local/bin/cprnc</CCSM_CPRNC>
    <GMAKE>make</GMAKE>
    <GMAKE_J>8</GMAKE_J>
    <BATCH_SYSTEM>none</BATCH_SYSTEM>
    <SUPPORTED_BY>me</SUPPORTED_BY>
    <MAX_TASKS_PER_NODE>16</MAX_TASKS_PER_NODE>
    <MAX_MPITASKS_PER_NODE>16</MAX_MPITASKS_PER_NODE>
    <mpirun mpilib="openmpi">
      <executable>mpirun</executable>
      <arguments>
        <arg name="num_tasks">-np {{ total_tasks }}</arg>
      </arguments>
    </mpirun>
  </machine>
</config_machines>
```

创建 `config_compilers.xml`：

```xml
<?xml version="1.0"?>
<config_compilers version="2.0">
  <compiler COMPILER="gnu" MACH="my_machine">
    <NETCDF_PATH>/opt/ctsm-libs/netcdf</NETCDF_PATH>
    <PIO_FILESYSTEM_HINTS>gpfs</PIO_FILESYSTEM_HINTS>
    <ESMF_LIBDIR>/opt/ctsm-libs/esmf-8.6.1/lib</ESMF_LIBDIR>
    <PNETCDF_PATH>/opt/ctsm-libs/pnetcdf-1.12.3</PNETCDF_PATH>
    <SLIBS>-L$(NETCDF_PATH)/lib -lnetcdf -lnetcdff</SLIBS>
    <FFLAGS>
      <append> -fallow-argument-mismatch -fallow-invalid-boz </append>
    </FFLAGS>
    <CMAKE_OPTS>
      <append> -DCMAKE_Fortran_FLAGS="-fallow-argument-mismatch" </append>
    </CMAKE_OPTS>
  </compiler>
</config_compilers>
```

---

## 6. 创建和运行案例

### 6.1 加载环境

```bash
# 加载依赖库环境
source /opt/ctsm-libs/ctsm-env.sh

# 激活 Python 环境
conda activate ctsm_pylib

# 进入 CTSM 目录
cd /path/to/my_ctsm
```

### 6.2 查询可用配置

```bash
cd cime/scripts

# 查看可用 compsets
./query_config --compsets clm

# 查看可用网格
./query_config --grids

# 查看可用机器
./query_config --machines
```

### 6.3 创建案例

```bash
cd cime/scripts

# 创建案例（示例：2000年条件，BGC+Crop，0.9x1.25度分辨率）
./create_newcase --case ~/cases/test_case \
    --res f09_g17_gl4 \
    --compset I2000Clm60BgcCrop \
    --mach my_machine \
    --run-unsupported
```

常用 compsets：
- `I2000Clm60Sp`：卫星表型，简单模式
- `I2000Clm60BgcCrop`：BGC + 作物
- `I1850Clm60BgcCrop`：1850年工业化前条件
- `IHistClm60BgcCrop`：历史瞬态模拟

### 6.4 配置案例

```bash
cd ~/cases/test_case

# 修改运行长度
./xmlchange STOP_OPTION=ndays,STOP_N=5

# 查看/修改其他设置
./xmlquery --listall

# 设置案例
./case.setup
```

### 6.5 修改 namelist

编辑 `user_nl_clm` 文件进行用户自定义设置：

```bash
# 编辑 namelist
vim user_nl_clm

# 示例内容：
# hist_nhtfrq = 0, -24      ! 输出频率：月平均，日平均
# hist_mfilt = 1, 30        ! 每个文件的输出次数
# hist_fincl1 = 'GPP', 'NPP', 'NEE', 'TLAI'  ! 额外输出变量
```

### 6.6 编译模型

```bash
./case.build

# 如果需要清理重新编译
./case.build --clean-all
./case.build
```

### 6.7 提交运行

```bash
# 直接运行（无作业调度器）
./case.submit

# 或手动运行
./case.run --skip-preview-namelist
```

### 6.8 查看输出

```bash
# 运行日志
ls -la timing/

# 输出数据
ls -la /scratch/ctsm_output/test_case/run/

# 归档数据
ls -la /scratch/ctsm_archive/test_case/
```

---

## 7. 常见问题排查

### 7.1 NetCDF 相关错误

**问题**：`NetCDF: HDF error`

**解决方案**：确保 HDF5 和 NetCDF 使用相同的编译器和 MPI 库编译。

```bash
# 检查 NetCDF 配置
nc-config --all
nf-config --all

# 确认 HDF5 路径
h5dump --version
```

**问题**：找不到 `netcdf.inc` 或 `netcdf.mod`

**解决方案**：
```bash
# 确保 NETCDF 环境变量正确设置
export NETCDF=/opt/ctsm-libs/netcdf
ls $NETCDF/include/netcdf.inc
ls $NETCDF/include/netcdf.mod
```

### 7.2 MPI 相关错误

**问题**：MPI 初始化失败

**解决方案**：
```bash
# 检查 MPI 是否正确安装
mpirun --version

# 测试 MPI
mpirun -np 2 hostname

# 检查防火墙设置（可能阻止 MPI 通信）
```

### 7.3 ESMF 相关错误

**问题**：找不到 `esmf.mk`

**解决方案**：
```bash
# 确保 ESMFMKFILE 设置正确
export ESMFMKFILE=/opt/ctsm-libs/esmf-8.6.1/lib/libg/Linux.gfortran.64.openmpi.default/esmf.mk

# 验证文件存在
ls -la $ESMFMKFILE
```

### 7.4 编译错误

**问题**：GFortran 10+ 参数不匹配错误

**解决方案**：在编译器标志中添加：
```bash
export FFLAGS="-fallow-argument-mismatch -fallow-invalid-boz"
```

**问题**：内存不足

**解决方案**：减少并行编译数量：
```bash
./case.build -j 2
```

### 7.5 输入数据下载失败

**问题**：无法下载输入数据

**解决方案**：
```bash
# 手动下载数据
cd /data/cesm/inputdata
wget --recursive --no-parent https://svn-ccsm-inputdata.cgd.ucar.edu/trunk/inputdata/

# 或使用 Globus 传输
# 参考：https://www.cesm.ucar.edu/models/cesm2/cesm-input-data
```

---

## 8. 参考资源

### 官方文档
- CTSM 用户指南：https://escomp.github.io/CTSM/
- CESM 论坛：https://bb.cgd.ucar.edu/cesm/forums/ctsm-clm-mosart-rtm.134/
- CIME 文档：http://esmci.github.io/cime/

### GitHub 资源
- CTSM 仓库：https://github.com/ESCOMP/CTSM
- CTSM Wiki：https://github.com/ESCOMP/CTSM/wiki
- 问题追踪：https://github.com/ESCOMP/CTSM/issues

### 依赖库文档
- NetCDF：https://www.unidata.ucar.edu/software/netcdf/
- HDF5：https://www.hdfgroup.org/solutions/hdf5/
- ESMF：https://earthsystemmodeling.org/
- OpenMPI：https://www.open-mpi.org/
- MPICH：https://www.mpich.org/

### 技术支持
- Email：ctsm-software@ucar.edu
- 讨论区：https://github.com/ESCOMP/CTSM/discussions

---

## 附录 A：完整安装脚本

以下是一个完整的自动化安装脚本示例：

```bash
#!/bin/bash
# CTSM 依赖库完整安装脚本
# 使用方法：bash install_ctsm_deps.sh

set -e  # 遇到错误立即退出

export INSTALL_ROOT=/opt/ctsm-libs
export BUILD_ROOT=/tmp/ctsm-build
export MAKE_JOBS=$(nproc)

mkdir -p $INSTALL_ROOT $BUILD_ROOT

echo "=== Installing CTSM dependencies ==="
echo "Install root: $INSTALL_ROOT"
echo "Build root: $BUILD_ROOT"
echo "Parallel jobs: $MAKE_JOBS"

# 1. OpenMPI
echo "=== Building OpenMPI ==="
cd $BUILD_ROOT
wget -q https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.6.tar.gz
tar xzf openmpi-4.1.6.tar.gz && cd openmpi-4.1.6
./configure --prefix=$INSTALL_ROOT/openmpi-4.1.6 --enable-mpi-fortran=all --enable-shared
make -j $MAKE_JOBS && make install
export PATH=$INSTALL_ROOT/openmpi-4.1.6/bin:$PATH
export LD_LIBRARY_PATH=$INSTALL_ROOT/openmpi-4.1.6/lib:$LD_LIBRARY_PATH

# 2. HDF5
echo "=== Building HDF5 ==="
cd $BUILD_ROOT
wget -q https://github.com/HDFGroup/hdf5/releases/download/hdf5_1.14.4.3/hdf5-1.14.4-3.tar.gz
tar xzf hdf5-1.14.4-3.tar.gz && cd hdf5-1.14.4-3
CC=mpicc FC=mpif90 ./configure --prefix=$INSTALL_ROOT/hdf5-1.14.4 \
    --enable-fortran --enable-parallel --enable-shared
make -j $MAKE_JOBS && make install
export HDF5_ROOT=$INSTALL_ROOT/hdf5-1.14.4
export PATH=$HDF5_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$HDF5_ROOT/lib:$LD_LIBRARY_PATH

# 3. NetCDF-C
echo "=== Building NetCDF-C ==="
cd $BUILD_ROOT
wget -q https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar xzf netcdf-c-4.9.2.tar.gz && cd netcdf-c-4.9.2
CPPFLAGS="-I$HDF5_ROOT/include" LDFLAGS="-L$HDF5_ROOT/lib" \
    CC=mpicc ./configure --prefix=$INSTALL_ROOT/netcdf-c-4.9.2 \
    --enable-netcdf-4 --enable-shared --disable-dap
make -j $MAKE_JOBS && make install
export NETCDF_C_ROOT=$INSTALL_ROOT/netcdf-c-4.9.2
export PATH=$NETCDF_C_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF_C_ROOT/lib:$LD_LIBRARY_PATH

# 4. NetCDF-Fortran
echo "=== Building NetCDF-Fortran ==="
cd $BUILD_ROOT
wget -q https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/netcdf-fortran-4.6.1.tar.gz
tar xzf netcdf-fortran-4.6.1.tar.gz && cd netcdf-fortran-4.6.1
CPPFLAGS="-I$NETCDF_C_ROOT/include -I$HDF5_ROOT/include" \
LDFLAGS="-L$NETCDF_C_ROOT/lib -L$HDF5_ROOT/lib" \
LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz" \
    CC=mpicc FC=mpif90 ./configure --prefix=$INSTALL_ROOT/netcdf-fortran-4.6.1 --enable-shared
make -j $MAKE_JOBS && make install
export NETCDF_FORTRAN_ROOT=$INSTALL_ROOT/netcdf-fortran-4.6.1

# Create unified NETCDF directory
export NETCDF=$INSTALL_ROOT/netcdf
mkdir -p $NETCDF/lib $NETCDF/include $NETCDF/bin
ln -sf $NETCDF_C_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_C_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_C_ROOT/bin/* $NETCDF/bin/
ln -sf $NETCDF_FORTRAN_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_FORTRAN_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_FORTRAN_ROOT/bin/* $NETCDF/bin/

echo "=== Installation complete ==="
echo "Please add the following to your ~/.bashrc:"
echo "source $INSTALL_ROOT/ctsm-env.sh"
```

---

## 附录 B：版本兼容性矩阵

| CTSM 版本 | ESMF 版本 | NetCDF-C | NetCDF-F | Python |
|-----------|-----------|----------|----------|--------|
| 5.3.x     | 8.4+      | 4.7.4+   | 4.5+     | 3.9+   |
| 5.2.x     | 8.2+      | 4.7+     | 4.5+     | 3.8+   |
| 5.1.x     | 8.0+      | 4.6+     | 4.5+     | 3.7+   |

---

*本手册最后更新：2024年12月*
*适用于 CTSM 5.3 系列*
