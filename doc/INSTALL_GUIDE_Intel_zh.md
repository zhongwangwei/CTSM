# CTSM 从头安装完整操作手册（Intel 编译器版）

## 目录

1. [概述](#1-概述)
2. [系统要求](#2-系统要求)
3. [Intel oneAPI 安装与配置](#3-intel-oneapi安装与配置)
4. [依赖软件安装](#4-依赖软件安装)
   - [4.1 Intel MPI 配置](#41-intel-mpi配置)
   - [4.2 HDF5 安装](#42-hdf5安装)
   - [4.3 NetCDF-C 安装](#43-netcdf-c安装)
   - [4.4 NetCDF-Fortran 安装](#44-netcdf-fortran安装)
   - [4.5 PnetCDF 安装（可选）](#45-pnetcdf安装可选)
   - [4.6 PIO 安装](#46-pio安装)
   - [4.7 ESMF 安装](#47-esmf安装)
5. [Python 环境配置](#5-python环境配置)
6. [CTSM 获取与配置](#6-ctsm获取与配置)
7. [创建和运行案例](#7-创建和运行案例)
8. [常见问题排查](#8-常见问题排查)
9. [参考资源](#9-参考资源)

---

## 1. 概述

本手册提供使用 **Intel oneAPI 编译器套件** 从零开始安装 CTSM（Community Terrestrial Systems Model）的完整指南。Intel 编译器在高性能计算环境中广泛使用，具有出色的优化性能和对 Fortran 的良好支持。

**使用 Intel 编译器的优势：**
- 优秀的代码优化和向量化能力
- 与 Intel MPI 的无缝集成
- 对 Fortran 2008/2018 标准的完整支持
- Intel MKL（数学核心库）提供高性能数学运算
- 在 Intel 处理器上有显著的性能优势

**Intel oneAPI 组件说明：**
- **Intel oneAPI Base Toolkit**：包含核心工具（dpcpp、Intel MKL 等）
- **Intel oneAPI HPC Toolkit**：包含 Fortran/C++ 编译器和 Intel MPI

---

## 2. 系统要求

### 2.1 硬件要求
- **内存**：至少 16GB RAM（编译 ESMF 时需要较多内存）
- **磁盘空间**：至少 80GB 可用空间（Intel oneAPI 约 20GB）
- **处理器**：Intel 64 位处理器（推荐）或 AMD64 兼容处理器

### 2.2 操作系统
- Linux（推荐 CentOS 7+、Ubuntu 18.04+、RHEL 7+、SUSE 15+）
- Intel oneAPI 2024 支持的发行版

### 2.3 软件版本要求

| 软件 | 最低版本 | 推荐版本 |
|------|----------|----------|
| Intel oneAPI HPC Toolkit | 2023.0 | 2024.0+ |
| CMake | 3.16 | 3.25+ |
| GNU Make | 3.8 | 4.0+ |
| HDF5 | 1.12 | 1.14+ |
| NetCDF-C | 4.8 | 4.9+ |
| NetCDF-Fortran | 4.5 | 4.6+ |
| ESMF | 8.5 | 8.6+ |
| Python | 3.9 | 3.11+ |

---

## 3. Intel oneAPI安装与配置

### 3.1 安装前准备

```bash
# 确保系统已安装基本开发工具
# Ubuntu/Debian
sudo apt update
sudo apt install -y build-essential cmake git wget curl m4 zlib1g-dev libcurl4-openssl-dev

# CentOS/RHEL
sudo yum groupinstall -y "Development Tools"
sudo yum install -y cmake git wget curl m4 zlib-devel libcurl-devel
```

### 3.2 安装 Intel oneAPI

#### 方法一：使用 APT/YUM 仓库安装（推荐）

**Ubuntu/Debian 系统：**

```bash
# 添加 Intel GPG 密钥
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
    | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

# 添加仓库
echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
    | sudo tee /etc/apt/sources.list.d/oneAPI.list

# 更新并安装
sudo apt update

# 安装 HPC Toolkit（包含 Fortran 编译器和 Intel MPI）
sudo apt install -y intel-hpckit

# 或者只安装必需的组件
sudo apt install -y intel-oneapi-compiler-fortran \
                    intel-oneapi-compiler-dpcpp-cpp \
                    intel-oneapi-mpi-devel \
                    intel-oneapi-mkl-devel
```

**CentOS/RHEL 系统：**

```bash
# 添加 Intel 仓库
cat << EOF | sudo tee /etc/yum.repos.d/oneAPI.repo
[oneAPI]
name=Intel® oneAPI repository
baseurl=https://yum.repos.intel.com/oneapi
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
EOF

# 安装 HPC Toolkit
sudo yum install -y intel-hpckit

# 或者只安装必需的组件
sudo yum install -y intel-oneapi-compiler-fortran \
                    intel-oneapi-compiler-dpcpp-cpp \
                    intel-oneapi-mpi-devel \
                    intel-oneapi-mkl-devel
```

#### 方法二：离线安装包安装

```bash
# 下载安装包（从 Intel 官网）
# https://www.intel.com/content/www/us/en/developer/tools/oneapi/hpc-toolkit-download.html

# 下载 HPC Toolkit（约 3GB）
wget https://registrationcenter-download.intel.com/akdlm/IRC_NAS/1b2baedd-a757-4a79-8abb-a5bf15adae9a/l_HPCKit_p_2024.0.0.49589_offline.sh

# 运行安装程序
sudo sh l_HPCKit_p_2024.0.0.49589_offline.sh -a --silent --eula accept

# 默认安装路径：/opt/intel/oneapi
```

### 3.3 配置 Intel oneAPI 环境

```bash
# 加载 Intel oneAPI 环境（每次使用前执行）
source /opt/intel/oneapi/setvars.sh

# 或者只加载特定组件
source /opt/intel/oneapi/compiler/latest/env/vars.sh
source /opt/intel/oneapi/mpi/latest/env/vars.sh
source /opt/intel/oneapi/mkl/latest/env/vars.sh
```

### 3.4 验证 Intel 编译器安装

```bash
# 加载环境
source /opt/intel/oneapi/setvars.sh

# 检查编译器版本
ifort --version    # Intel Fortran 编译器
icx --version      # Intel C 编译器（新版）
icpx --version     # Intel C++ 编译器（新版）

# 传统编译器（如果可用）
icc --version      # Intel C 编译器（经典版）
icpc --version     # Intel C++ 编译器（经典版）

# 检查 MPI
mpiifx --version   # Intel MPI Fortran 包装器
mpiicx --version   # Intel MPI C 包装器

# 检查 MKL
echo $MKLROOT
```

### 3.5 创建 Intel 环境配置文件

```bash
# 创建环境配置脚本
cat > ~/intel_env.sh << 'EOF'
#!/bin/bash
# Intel oneAPI 环境配置

# 加载 Intel oneAPI
source /opt/intel/oneapi/setvars.sh --force

# 设置编译器变量
export CC=icx
export CXX=icpx
export FC=ifx
export F77=ifx
export F90=ifx

# 对于旧版本 Intel 编译器，使用：
# export CC=icc
# export CXX=icpc
# export FC=ifort
# export F77=ifort
# export F90=ifort

# MPI 编译器包装器
export MPICC=mpiicx
export MPICXX=mpiicpx
export MPIFC=mpiifx
export MPIF77=mpiifx
export MPIF90=mpiifx

# 对于旧版本：
# export MPICC=mpiicc
# export MPICXX=mpiicpc
# export MPIFC=mpiifort

echo "Intel oneAPI environment loaded"
echo "  Fortran: $(ifx --version 2>&1 | head -1)"
echo "  C:       $(icx --version 2>&1 | head -1)"
echo "  MPI:     $(mpiifx --version 2>&1 | head -1)"
EOF

chmod +x ~/intel_env.sh
```

---

## 4. 依赖软件安装

### 4.0 准备工作

```bash
# 加载 Intel 环境
source ~/intel_env.sh

# 设置安装目录
export INSTALL_ROOT=/opt/ctsm-libs-intel
export BUILD_ROOT=/tmp/ctsm-build-intel

# 创建目录
sudo mkdir -p $INSTALL_ROOT
sudo chown $USER:$USER $INSTALL_ROOT
mkdir -p $BUILD_ROOT

# 设置并行编译核心数
export MAKE_JOBS=$(nproc)

# 确认编译器
echo "CC=$CC, FC=$FC, MPIFC=$MPIFC"
```

### 4.1 Intel MPI配置

Intel oneAPI 自带 Intel MPI，无需单独安装，只需正确配置。

```bash
# 验证 Intel MPI
which mpiifx
which mpiicx
mpirun --version

# 测试 MPI
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

mpiifx test_mpi.f90 -o test_mpi
mpirun -np 4 ./test_mpi
rm -f test_mpi test_mpi.f90
```

### 4.2 HDF5安装

使用 Intel 编译器编译 HDF5：

```bash
cd $BUILD_ROOT

# 下载 HDF5
wget https://github.com/HDFGroup/hdf5/releases/download/hdf5_1.14.4.3/hdf5-1.14.4-3.tar.gz
tar xzf hdf5-1.14.4-3.tar.gz
cd hdf5-1.14.4-3

# 配置（使用 Intel MPI 编译器包装器）
CC=mpiicx FC=mpiifx ./configure --prefix=$INSTALL_ROOT/hdf5-1.14.4 \
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

# 验证
h5pfc --version
h5dump --version
```

### 4.3 NetCDF-C安装

```bash
cd $BUILD_ROOT

# 下载 NetCDF-C
wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar xzf netcdf-c-4.9.2.tar.gz
cd netcdf-c-4.9.2

# 设置依赖路径
export CPPFLAGS="-I$HDF5_ROOT/include"
export LDFLAGS="-L$HDF5_ROOT/lib"
export CFLAGS="-O3 -xHost"

# 配置
CC=mpiicx ./configure --prefix=$INSTALL_ROOT/netcdf-c-4.9.2 \
    --enable-netcdf-4 \
    --enable-shared \
    --enable-parallel-tests \
    --disable-dap

# 编译安装
make -j $MAKE_JOBS
make check  # 可选，运行测试
make install

# 设置环境变量
export NETCDF_C_ROOT=$INSTALL_ROOT/netcdf-c-4.9.2
export PATH=$NETCDF_C_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF_C_ROOT/lib:$LD_LIBRARY_PATH

# 验证
nc-config --version
nc-config --has-nc4
nc-config --has-parallel4
```

### 4.4 NetCDF-Fortran安装

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
export FCFLAGS="-O3 -xHost"
export FFLAGS="-O3 -xHost"

# 配置
CC=mpiicx FC=mpiifx F77=mpiifx ./configure --prefix=$INSTALL_ROOT/netcdf-fortran-4.6.1 \
    --enable-shared

# 编译安装
make -j $MAKE_JOBS
make check  # 可选
make install

# 设置环境变量
export NETCDF_FORTRAN_ROOT=$INSTALL_ROOT/netcdf-fortran-4.6.1
export PATH=$NETCDF_FORTRAN_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF_FORTRAN_ROOT/lib:$LD_LIBRARY_PATH

# 创建统一的 NETCDF 目录
export NETCDF=$INSTALL_ROOT/netcdf
mkdir -p $NETCDF/lib $NETCDF/include $NETCDF/bin
ln -sf $NETCDF_C_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_C_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_C_ROOT/bin/* $NETCDF/bin/
ln -sf $NETCDF_FORTRAN_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_FORTRAN_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_FORTRAN_ROOT/bin/* $NETCDF/bin/

# 验证
nf-config --version
```

#### 验证 NetCDF 安装

```bash
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

mpiifx test_netcdf.f90 -o test_netcdf \
    -I$NETCDF/include -L$NETCDF/lib -lnetcdff -lnetcdf
./test_netcdf
rm -f test_netcdf test_netcdf.f90 test.nc
```

### 4.5 PnetCDF安装（可选）

```bash
cd $BUILD_ROOT

# 下载 PnetCDF
wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz
tar xzf pnetcdf-1.12.3.tar.gz
cd pnetcdf-1.12.3

# 配置
CC=mpiicx FC=mpiifx CXX=mpiicpx ./configure --prefix=$INSTALL_ROOT/pnetcdf-1.12.3 \
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

### 4.6 PIO安装

```bash
cd $BUILD_ROOT

# 下载 PIO
git clone https://github.com/NCAR/ParallelIO.git pio
cd pio
git checkout pio2_6_2

# 创建构建目录
mkdir build && cd build

# 配置 CMake
CC=mpiicx FC=mpiifx cmake .. \
    -DCMAKE_INSTALL_PREFIX=$INSTALL_ROOT/pio-2.6.2 \
    -DPIO_ENABLE_FORTRAN=ON \
    -DPIO_ENABLE_TIMING=OFF \
    -DNetCDF_C_PATH=$NETCDF_C_ROOT \
    -DNetCDF_Fortran_PATH=$NETCDF_FORTRAN_ROOT \
    -DHDF5_PATH=$HDF5_ROOT \
    -DWITH_PNETCDF=OFF \
    -DCMAKE_C_COMPILER=mpiicx \
    -DCMAKE_Fortran_COMPILER=mpiifx

# 编译安装
make -j $MAKE_JOBS
make install

# 设置环境变量
export PIO_ROOT=$INSTALL_ROOT/pio-2.6.2
export LD_LIBRARY_PATH=$PIO_ROOT/lib:$LD_LIBRARY_PATH
```

### 4.7 ESMF安装

ESMF 编译是最复杂的部分，需要仔细配置。

```bash
cd $BUILD_ROOT

# 下载 ESMF
git clone https://github.com/esmf-org/esmf.git
cd esmf
git checkout v8.6.1

# 设置 ESMF 构建环境变量
export ESMF_DIR=$PWD
export ESMF_INSTALL_PREFIX=$INSTALL_ROOT/esmf-8.6.1

# Intel 编译器配置
export ESMF_COMPILER=intel
export ESMF_COMM=intelmpi

# 对于新版 Intel oneAPI（ifx/icx），可能需要：
export ESMF_COMPILER=intel
export ESMF_F90COMPILEROPTS="-O2 -fp-model precise"
export ESMF_CXXCOMPILEROPTS="-O2 -fp-model precise"

# NetCDF 配置
export ESMF_NETCDF=nc-config
export ESMF_NETCDF_INCLUDE=$NETCDF/include
export ESMF_NETCDF_LIBPATH=$NETCDF/lib

# PIO 配置
export ESMF_PIO=external
export ESMF_PIO_LIBPATH=$PIO_ROOT/lib
export ESMF_PIO_INCLUDE=$PIO_ROOT/include

# 可选：使用 PnetCDF
# export ESMF_PNETCDF=pnetcdf-config
# export ESMF_PNETCDF_INCLUDE=$PNETCDF_ROOT/include
# export ESMF_PNETCDF_LIBPATH=$PNETCDF_ROOT/lib

# 编译
make -j $MAKE_JOBS

# 如果使用新版 ifx/icx 遇到问题，尝试串行编译
# make

# 安装
make install

# 设置运行时环境变量
export ESMF_ROOT=$INSTALL_ROOT/esmf-8.6.1
# 注意：路径可能因版本和编译器而异，请检查实际路径
export ESMFMKFILE=$(find $ESMF_ROOT -name "esmf.mk" | head -1)
export PATH=$ESMF_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$ESMF_ROOT/lib:$LD_LIBRARY_PATH

# 验证
echo "ESMFMKFILE=$ESMFMKFILE"
ls -la $ESMFMKFILE
```

#### ESMF 编译注意事项

如果遇到编译错误，尝试以下方法：

```bash
# 方法1：使用经典 Intel 编译器
export ESMF_COMPILER=intel
export ESMF_F90COMPILER=ifort
export ESMF_CXXCOMPILER=icpc
export ESMF_CCOMPILER=icc

# 方法2：降低优化级别
export ESMF_OPTLEVEL=2
export ESMF_F90COMPILEOPTS="-O2"
export ESMF_CXXCOMPILEOPTS="-O2"

# 方法3：禁用某些功能
export ESMF_LAPACK=internal
export ESMF_XERCES=OFF

# 清理并重新编译
make clean
make -j $MAKE_JOBS
```

### 4.8 创建完整环境配置文件

```bash
cat > $INSTALL_ROOT/ctsm-intel-env.sh << 'EOF'
#!/bin/bash
# CTSM 依赖库环境配置（Intel 编译器版）

# ============================================
# Intel oneAPI 环境
# ============================================
source /opt/intel/oneapi/setvars.sh --force 2>/dev/null

# 设置编译器变量
export CC=icx
export CXX=icpx
export FC=ifx
export F77=ifx
export F90=ifx

# MPI 编译器包装器
export MPICC=mpiicx
export MPICXX=mpiicpx
export MPIFC=mpiifx
export MPIF77=mpiifx
export MPIF90=mpiifx

# ============================================
# CTSM 依赖库路径
# ============================================
export INSTALL_ROOT=/opt/ctsm-libs-intel

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
export ESMFMKFILE=$(find $ESMF_ROOT -name "esmf.mk" 2>/dev/null | head -1)
export PATH=$ESMF_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$ESMF_ROOT/lib:$LD_LIBRARY_PATH

# ============================================
# 验证环境
# ============================================
echo "=========================================="
echo "CTSM Intel Environment Loaded"
echo "=========================================="
echo "  Intel Fortran: $(ifx --version 2>&1 | head -1)"
echo "  Intel C:       $(icx --version 2>&1 | head -1)"
echo "  Intel MPI:     $(mpiifx --version 2>&1 | head -1)"
echo "  NetCDF:        $(nc-config --version 2>/dev/null)"
echo "  HDF5:          $(h5dump --version 2>&1 | head -1)"
echo "  ESMFMKFILE:    $ESMFMKFILE"
echo "=========================================="
EOF

chmod +x $INSTALL_ROOT/ctsm-intel-env.sh
```

---

## 5. Python环境配置

Python 环境配置与 GNU 版本相同，详见 [GNU版本手册](INSTALL_GUIDE_zh.md#4-python环境配置)。

```bash
# 安装 Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
$HOME/miniconda3/bin/conda init bash
source ~/.bashrc

# 创建 CTSM Python 环境
cd /path/to/CTSM
./py_env_create
conda activate ctsm_pylib
```

---

## 6. CTSM获取与配置

### 6.1 克隆 CTSM 仓库

```bash
git clone https://github.com/ESCOMP/CTSM.git my_ctsm_intel
cd my_ctsm_intel
./bin/git-fleximod update
```

### 6.2 配置机器设置（Intel 编译器）

在 `$HOME/.cime/` 目录下创建配置文件：

```bash
mkdir -p $HOME/.cime
```

创建 `config_machines.xml`：

```xml
<?xml version="1.0"?>
<config_machines version="2.0">
  <machine MACH="my_intel_machine">
    <DESC>My local Linux machine with Intel compilers</DESC>
    <OS>LINUX</OS>
    <COMPILERS>intel</COMPILERS>
    <MPILIBS>intelmpi</MPILIBS>
    <NODENAME_REGEX>.*</NODENAME_REGEX>
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
    <MAX_TASKS_PER_NODE>32</MAX_TASKS_PER_NODE>
    <MAX_MPITASKS_PER_NODE>32</MAX_MPITASKS_PER_NODE>
    <mpirun mpilib="intelmpi">
      <executable>mpirun</executable>
      <arguments>
        <arg name="num_tasks">-np {{ total_tasks }}</arg>
      </arguments>
    </mpirun>
    <environment_variables>
      <env name="OMP_STACKSIZE">256M</env>
      <env name="KMP_STACKSIZE">256M</env>
    </environment_variables>
  </machine>
</config_machines>
```

创建 `config_compilers.xml`：

```xml
<?xml version="1.0"?>
<config_compilers version="2.0">
  <compiler COMPILER="intel" MACH="my_intel_machine">
    <NETCDF_PATH>/opt/ctsm-libs-intel/netcdf</NETCDF_PATH>
    <ESMF_LIBDIR ENV="ESMF_ROOT">/opt/ctsm-libs-intel/esmf-8.6.1/lib</ESMF_LIBDIR>
    <PNETCDF_PATH>/opt/ctsm-libs-intel/pnetcdf-1.12.3</PNETCDF_PATH>
    <PIO_FILESYSTEM_HINTS>gpfs</PIO_FILESYSTEM_HINTS>

    <!-- Intel Fortran 编译器标志 -->
    <FFLAGS>
      <append> -qno-opt-dynamic-align -fp-model precise -convert big_endian -assume byterecl -ftz -traceback -assume realloc_lhs -xHost </append>
    </FFLAGS>

    <!-- 调试模式标志 -->
    <FFLAGS_DEBUG>
      <append> -O0 -g -check uninit -check bounds -check pointers -fpe0 -check noarg_temp_created </append>
    </FFLAGS_DEBUG>

    <!-- Intel C 编译器标志 -->
    <CFLAGS>
      <append> -qno-opt-dynamic-align -fp-model precise -xHost </append>
    </CFLAGS>

    <!-- 链接库 -->
    <SLIBS>
      <append> -L$(NETCDF_PATH)/lib -lnetcdf -lnetcdff </append>
    </SLIBS>

    <!-- CMake 选项 -->
    <CMAKE_OPTS>
      <append> -DCMAKE_C_COMPILER=mpiicx -DCMAKE_Fortran_COMPILER=mpiifx </append>
    </CMAKE_OPTS>

    <!-- 如果使用旧版 Intel 编译器 -->
    <!--
    <SFC>ifort</SFC>
    <SCC>icc</SCC>
    <SCXX>icpc</SCXX>
    <MPIFC>mpiifort</MPIFC>
    <MPICC>mpiicc</MPICC>
    <MPICXX>mpiicpc</MPICXX>
    -->

    <!-- 新版 Intel oneAPI 编译器 -->
    <SFC>ifx</SFC>
    <SCC>icx</SCC>
    <SCXX>icpx</SCXX>
    <MPIFC>mpiifx</MPIFC>
    <MPICC>mpiicx</MPICC>
    <MPICXX>mpiicpx</MPICXX>

  </compiler>
</config_compilers>
```

---

## 7. 创建和运行案例

### 7.1 加载环境

```bash
# 加载 Intel CTSM 环境
source /opt/ctsm-libs-intel/ctsm-intel-env.sh

# 激活 Python 环境
conda activate ctsm_pylib

# 进入 CTSM 目录
cd /path/to/my_ctsm_intel
```

### 7.2 创建案例

```bash
cd cime/scripts

# 创建案例
./create_newcase --case ~/cases/test_intel \
    --res f09_g17_gl4 \
    --compset I2000Clm60BgcCrop \
    --mach my_intel_machine \
    --compiler intel \
    --run-unsupported
```

### 7.3 配置和编译

```bash
cd ~/cases/test_intel

# 修改运行参数
./xmlchange STOP_OPTION=ndays,STOP_N=5

# 设置案例
./case.setup

# 编辑 namelist（可选）
vim user_nl_clm

# 编译
./case.build
```

### 7.4 提交运行

```bash
./case.submit
```

---

## 8. 常见问题排查

### 8.1 Intel 编译器相关问题

**问题**：找不到 `ifx` 或 `ifort`

**解决方案**：
```bash
# 确保加载了 Intel 环境
source /opt/intel/oneapi/setvars.sh

# 检查编译器路径
which ifx
which ifort

# 如果只有旧版编译器，使用 ifort 代替 ifx
export FC=ifort
```

**问题**：Intel MPI 初始化失败

**解决方案**：
```bash
# 设置 Intel MPI 环境变量
export I_MPI_PMI_LIBRARY=/usr/lib64/libpmi.so
export FI_PROVIDER=sockets  # 或 tcp

# 对于 Slurm 作业调度器
export I_MPI_PMI_LIBRARY=/usr/lib64/libpmi2.so
```

### 8.2 编译标志兼容性问题

**问题**：新版 Intel 编译器（ifx/icx）不识别某些旧标志

**解决方案**：
```bash
# 替换旧标志
# -xCORE-AVX2 -> -march=core-avx2 或 -xHost
# -ip -> 通常可以省略
# -fp-model source -> -fp-model=precise
```

### 8.3 ESMF 编译问题

**问题**：ESMF 编译失败

**解决方案**：
```bash
# 使用经典 Intel 编译器
export ESMF_COMPILER=intel
export ESMF_F90COMPILER=ifort
export ESMF_CXXCOMPILER=icpc

# 或降低并行编译数量
make -j 4

# 或串行编译
make
```

### 8.4 NetCDF 链接问题

**问题**：运行时找不到 NetCDF 库

**解决方案**：
```bash
# 检查库路径
ldd /path/to/executable | grep netcdf

# 确保 LD_LIBRARY_PATH 包含 NetCDF 路径
export LD_LIBRARY_PATH=$NETCDF/lib:$LD_LIBRARY_PATH

# 或使用 rpath 编译
export LDFLAGS="-Wl,-rpath,$NETCDF/lib"
```

### 8.5 性能优化建议

```bash
# Intel 特定优化
export OMP_NUM_THREADS=4
export OMP_STACKSIZE=256M
export KMP_STACKSIZE=256M
export KMP_AFFINITY=granularity=fine,compact,1,0

# 使用 Intel MKL 作为 BLAS/LAPACK
export MKL_NUM_THREADS=4
export MKL_DYNAMIC=FALSE
```

---

## 9. 参考资源

### Intel oneAPI 文档
- Intel oneAPI 官网：https://www.intel.com/content/www/us/en/developer/tools/oneapi/overview.html
- Intel Fortran 编译器文档：https://www.intel.com/content/www/us/en/docs/fortran-compiler/
- Intel MPI 文档：https://www.intel.com/content/www/us/en/docs/mpi-library/

### CTSM 文档
- CTSM 用户指南：https://escomp.github.io/CTSM/
- CTSM GitHub：https://github.com/ESCOMP/CTSM

### 依赖库文档
- NetCDF：https://www.unidata.ucar.edu/software/netcdf/
- HDF5：https://www.hdfgroup.org/solutions/hdf5/
- ESMF：https://earthsystemmodeling.org/

---

## 附录 A：Intel 编译器完整安装脚本

```bash
#!/bin/bash
# CTSM 依赖库完整安装脚本（Intel 编译器版）
# 使用方法：source intel_env.sh && bash install_ctsm_deps_intel.sh

set -e

# 检查 Intel 环境
if ! command -v ifx &> /dev/null && ! command -v ifort &> /dev/null; then
    echo "Error: Intel Fortran compiler not found. Please run:"
    echo "  source /opt/intel/oneapi/setvars.sh"
    exit 1
fi

export INSTALL_ROOT=/opt/ctsm-libs-intel
export BUILD_ROOT=/tmp/ctsm-build-intel
export MAKE_JOBS=$(nproc)

# 设置编译器
if command -v ifx &> /dev/null; then
    export FC=mpiifx
    export CC=mpiicx
    export CXX=mpiicpx
else
    export FC=mpiifort
    export CC=mpiicc
    export CXX=mpiicpc
fi

mkdir -p $INSTALL_ROOT $BUILD_ROOT

echo "=== Installing CTSM dependencies (Intel) ==="
echo "Install root: $INSTALL_ROOT"
echo "Compiler: $FC"

# 1. HDF5
echo "=== Building HDF5 ==="
cd $BUILD_ROOT
wget -q https://github.com/HDFGroup/hdf5/releases/download/hdf5_1.14.4.3/hdf5-1.14.4-3.tar.gz
tar xzf hdf5-1.14.4-3.tar.gz && cd hdf5-1.14.4-3
CC=$CC FC=$FC ./configure --prefix=$INSTALL_ROOT/hdf5-1.14.4 \
    --enable-fortran --enable-parallel --enable-shared
make -j $MAKE_JOBS && make install
export HDF5_ROOT=$INSTALL_ROOT/hdf5-1.14.4
export PATH=$HDF5_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$HDF5_ROOT/lib:$LD_LIBRARY_PATH

# 2. NetCDF-C
echo "=== Building NetCDF-C ==="
cd $BUILD_ROOT
wget -q https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar xzf netcdf-c-4.9.2.tar.gz && cd netcdf-c-4.9.2
CPPFLAGS="-I$HDF5_ROOT/include" LDFLAGS="-L$HDF5_ROOT/lib" \
    CC=$CC ./configure --prefix=$INSTALL_ROOT/netcdf-c-4.9.2 \
    --enable-netcdf-4 --enable-shared --disable-dap
make -j $MAKE_JOBS && make install
export NETCDF_C_ROOT=$INSTALL_ROOT/netcdf-c-4.9.2
export PATH=$NETCDF_C_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$NETCDF_C_ROOT/lib:$LD_LIBRARY_PATH

# 3. NetCDF-Fortran
echo "=== Building NetCDF-Fortran ==="
cd $BUILD_ROOT
wget -q https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/netcdf-fortran-4.6.1.tar.gz
tar xzf netcdf-fortran-4.6.1.tar.gz && cd netcdf-fortran-4.6.1
CPPFLAGS="-I$NETCDF_C_ROOT/include -I$HDF5_ROOT/include" \
LDFLAGS="-L$NETCDF_C_ROOT/lib -L$HDF5_ROOT/lib" \
LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz" \
    CC=$CC FC=$FC ./configure --prefix=$INSTALL_ROOT/netcdf-fortran-4.6.1 --enable-shared
make -j $MAKE_JOBS && make install

# Create unified NETCDF directory
export NETCDF=$INSTALL_ROOT/netcdf
mkdir -p $NETCDF/lib $NETCDF/include $NETCDF/bin
ln -sf $NETCDF_C_ROOT/lib/* $NETCDF/lib/
ln -sf $NETCDF_C_ROOT/include/* $NETCDF/include/
ln -sf $NETCDF_C_ROOT/bin/* $NETCDF/bin/
ln -sf $INSTALL_ROOT/netcdf-fortran-4.6.1/lib/* $NETCDF/lib/
ln -sf $INSTALL_ROOT/netcdf-fortran-4.6.1/include/* $NETCDF/include/

echo "=== Installation complete ==="
echo "Please source: $INSTALL_ROOT/ctsm-intel-env.sh"
```

---

## 附录 B：Intel 编译器版本对照

| Intel oneAPI 版本 | 经典编译器 | LLVM 编译器 | 推荐用于 CTSM |
|------------------|-----------|------------|---------------|
| 2024.0+ | ifort, icc, icpc | ifx, icx, icpx | ifx/icx |
| 2023.x | ifort, icc, icpc | ifx, icx, icpx | ifort/icc |
| 2022.x | ifort, icc, icpc | ifx, icx | ifort/icc |
| 2021.x | ifort, icc, icpc | (beta) | ifort/icc |

**注意**：
- 经典编译器（ifort/icc）从 2024.0 开始被标记为 deprecated
- 新版 LLVM 编译器（ifx/icx）可能与某些旧代码不兼容
- 建议先尝试新编译器，如遇问题再回退到经典编译器

---

*本手册最后更新：2024年12月*
*适用于 CTSM 5.3 系列 + Intel oneAPI 2024*
