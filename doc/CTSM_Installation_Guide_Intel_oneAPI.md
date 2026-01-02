# CTSM 从头安装操作手册

## 基于 Intel oneAPI (2022+) Fortran 编译器 + MPICH4

---

**版本**: CTSM 5.4
**编译器**: Intel oneAPI HPC Toolkit (2022.x 及以上)
**MPI库**: MPICH 4.x
**更新日期**: 2025年12月

---

## 目录

1. [系统要求](#1-系统要求)
2. [Intel oneAPI 安装与配置](#2-intel-oneapi-安装与配置)
3. [依赖库安装](#3-依赖库安装)
4. [CTSM 源代码获取](#4-ctsm-源代码获取)
5. [Python 环境配置](#5-python-环境配置)
6. [CIME 机器配置](#6-cime-机器配置)
7. [创建和运行 Case](#7-创建和运行-case)
8. [常见问题排查](#8-常见问题排查)
9. [附录：环境变量速查表](#9-附录环境变量速查表)

---

## 1. 系统要求

### 1.1 硬件要求

| 项目 | 最低要求 | 推荐配置 |
|------|----------|----------|
| 内存 | 8 GB | 32 GB 以上 |
| 磁盘空间 | 50 GB | 200 GB 以上 |
| CPU | 4 核 | 16 核以上 |

### 1.2 软件要求

| 软件 | 版本要求 |
|------|----------|
| 操作系统 | Linux (RHEL 7+, CentOS 7+, Ubuntu 18.04+) |
| Intel oneAPI | 2022.x 或更高 |
| CMake | 3.10+ |
| GNU Make | 3.81+ |
| Git | 2.0+ |
| Python | 3.8+ (推荐 3.11+) |
| Perl | 5.16+ |
| Perl XML::LibXML | 必需 |

### 1.3 必需的库

- NetCDF-C (4.7.4+)
- NetCDF-Fortran (4.5+)
- PnetCDF (可选，用于并行 I/O)
- HDF5 (1.10+)
- MPICH (4.0+)
- ESMF (Earth System Modeling Framework, 8.0+)
- PIO (Parallel I/O，CTSM 自带)

---

## 2. Intel oneAPI 安装与配置

### 2.1 下载 Intel oneAPI HPC Toolkit

从 Intel 官方网站下载：
```
https://www.intel.com/content/www/us/en/developer/tools/oneapi/hpc-toolkit-download.html
```

### 2.2 安装步骤

#### 方式一：使用包管理器安装（推荐）

**对于 RHEL/CentOS:**

```bash
# 添加 Intel oneAPI 仓库
sudo tee /etc/yum.repos.d/oneAPI.repo << EOF
[oneAPI]
name=Intel® oneAPI repository
baseurl=https://yum.repos.intel.com/oneapi
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://yum.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
EOF

# 安装 HPC Toolkit
sudo yum install intel-hpckit-2022.3
```

**对于 Ubuntu/Debian:**

```bash
# 下载并添加 Intel GPG key
wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
    | gpg --dearmor | sudo tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null

# 添加仓库
echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
    | sudo tee /etc/apt/sources.list.d/oneAPI.list

# 更新并安装
sudo apt update
sudo apt install intel-hpckit-2022.3
```

#### 方式二：离线安装

```bash
# 下载离线安装包后执行
chmod +x l_HPCKit_p_2022.3.0.8751_offline.sh
sudo ./l_HPCKit_p_2022.3.0.8751_offline.sh
```

### 2.3 初始化 Intel oneAPI 环境

在每次使用前，需要初始化环境变量：

```bash
# 初始化 Intel oneAPI 环境（默认安装路径）
# 注意：我们使用 MPICH4 替代 Intel MPI，所以只初始化编译器组件
source /opt/intel/oneapi/setvars.sh --config=""

# 或者只初始化编译器组件（推荐，避免加载 Intel MPI）
source /opt/intel/oneapi/compiler/latest/env/vars.sh
source /opt/intel/oneapi/mkl/latest/env/vars.sh
```

### 2.4 验证安装

```bash
# 检查 Fortran 编译器版本
ifort --version
# 或使用新的 LLVM-based 编译器
ifx --version

# 预期输出示例：
# ifort (IFORT) 2021.7.0 20220726
# Intel(R) Fortran Compiler for applications running on Intel(R) 64, Version 2022.2.0
```

### 2.5 创建环境配置文件

为方便起见，创建一个环境配置脚本：

```bash
cat > ~/intel_ctsm_env.sh << 'EOF'
#!/bin/bash
# Intel oneAPI + MPICH4 environment for CTSM

# 初始化 Intel oneAPI（只加载编译器，不加载 Intel MPI）
source /opt/intel/oneapi/compiler/latest/env/vars.sh
source /opt/intel/oneapi/mkl/latest/env/vars.sh

# 设置 Intel 编译器
export CC=icc
export CXX=icpc
export FC=ifort
export F77=ifort
export F90=ifort

# 如果使用新的 LLVM-based 编译器（推荐用于 2023+）
# export CC=icx
# export CXX=icpx
# export FC=ifx
# export F77=ifx
# export F90=ifx

# MPICH4 路径（安装后设置）
export MPICH_ROOT=/opt/ctsm_libs/mpich4
export PATH=$MPICH_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPICH_ROOT/lib:$LD_LIBRARY_PATH
export MANPATH=$MPICH_ROOT/share/man:$MANPATH

# MPI 编译器包装（MPICH4 风格）
export MPICC=mpicc
export MPICXX=mpicxx
export MPIFC=mpifort
export MPIF90=mpifort
export MPIF77=mpifort

# 编译器标志
export CFLAGS="-O2 -xHost"
export CXXFLAGS="-O2 -xHost"
export FFLAGS="-O2 -xHost -traceback"
export FCFLAGS="-O2 -xHost -traceback"

echo "Intel oneAPI + MPICH4 environment loaded for CTSM"
echo "Fortran Compiler: $(which ifort)"
echo "MPI Fortran: $(which mpifort)"
EOF

chmod +x ~/intel_ctsm_env.sh
```

---

## 3. 依赖库安装

### 3.1 安装顺序

依赖库需要按以下顺序安装：

1. zlib
2. **MPICH4**（MPI 库）
3. HDF5
4. NetCDF-C
5. NetCDF-Fortran
6. PnetCDF (可选)
7. ESMF

### 3.2 设置安装目录

```bash
# 创建安装目录
export CTSM_LIBS=/opt/ctsm_libs
sudo mkdir -p $CTSM_LIBS
sudo chown $USER:$USER $CTSM_LIBS

# 创建源码目录
mkdir -p ~/ctsm_build
cd ~/ctsm_build

# 加载 Intel 编译器环境（暂时不加载 MPI，因为还没安装）
source /opt/intel/oneapi/compiler/latest/env/vars.sh
source /opt/intel/oneapi/mkl/latest/env/vars.sh

export CC=icc
export CXX=icpc
export FC=ifort
export F77=ifort
export F90=ifort
```

### 3.3 安装 zlib

```bash
cd ~/ctsm_build
wget https://zlib.net/zlib-1.3.tar.gz
tar -xzf zlib-1.3.tar.gz
cd zlib-1.3

./configure --prefix=$CTSM_LIBS
make -j$(nproc)
make install
```

### 3.4 安装 MPICH4

```bash
cd ~/ctsm_build
wget https://www.mpich.org/static/downloads/4.2.3/mpich-4.2.3.tar.gz
tar -xzf mpich-4.2.3.tar.gz
cd mpich-4.2.3

# 设置安装目录（注意：确保路径末尾没有多余的斜杠）
export MPICH_INSTALL_DIR=${CTSM_LIBS}/mpich4

# 使用 Intel 编译器编译 MPICH4
CC=icc CXX=icpc FC=ifort F77=ifort \
./configure --prefix=${MPICH_INSTALL_DIR} \
    --enable-shared \
    --enable-static \
    --enable-fast=O2 \
    --enable-fortran=all \
    --with-device=ch4:ofi \
    --enable-romio \
    --enable-cxx

# 编译（注意：不要运行 make check，它需要已安装的 mpicc）
make -j$(nproc)

# 安装
make install

# 设置 MPICH4 环境变量
export MPICH_ROOT=${MPICH_INSTALL_DIR}
export PATH=$MPICH_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPICH_ROOT/lib:$LD_LIBRARY_PATH
export MANPATH=$MPICH_ROOT/share/man:$MANPATH

# 验证安装
which mpifort
mpifort --version
mpicc --version
mpirun --version
```

**关于设备选项的说明**：

`--with-device=ch4:ofi` 使用 libfabric 库进行高性能网络通信，但需要额外的系统依赖：
- **libfabric**: 高性能网络接口库
- **libpciaccess**: PCI 设备访问库（libfabric 的依赖）

如果运行时遇到 `libpciaccess.so.0: cannot open shared object file` 错误，有两种解决方案：

**方案一：安装 libpciaccess 依赖**

```bash
# CentOS/RHEL:
sudo yum install libpciaccess libpciaccess-devel

# Ubuntu/Debian:
sudo apt install libpciaccess0 libpciaccess-dev

# 或使用 conda（如果没有 sudo 权限）:
conda install -c conda-forge libpciaccess
```

**方案二：使用 ch3:sock 设备（推荐用于简单配置）**

如果不需要高性能网络或继续遇到依赖问题，可以使用 TCP/IP 通信的 ch3:sock 设备，它没有额外的系统依赖：

```bash
# 替代配置（使用 TCP/IP 通信，无额外依赖）
make clean  # 如果之前编译过，先清理
CC=icc CXX=icpc FC=ifort F77=ifort \
./configure --prefix=${MPICH_INSTALL_DIR} \
    --enable-shared \
    --enable-static \
    --enable-fast=O2 \
    --enable-fortran=all \
    --with-device=ch3:sock \
    --enable-romio \
    --enable-cxx
make -j$(nproc)
make install
```

ch3:sock 的特点：
- ✓ 无需 libfabric 和 libpciaccess
- ✓ 配置简单，兼容性好
- ✓ 适合单机或小规模集群
- ✗ 性能略低于 ch4:ofi（对于大规模并行计算）

### 3.5 安装 HDF5

```bash
cd ~/ctsm_build
wget https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.14/hdf5-1.14.3/src/hdf5-1.14.3.tar.gz
tar -xzf hdf5-1.14.3.tar.gz
cd hdf5-1.14.3

# 配置 HDF5（启用并行支持，使用 MPICH4）
CC=mpicc FC=mpifort \
./configure --prefix=$CTSM_LIBS \
    --enable-parallel \
    --enable-fortran \
    --enable-shared \
    --with-zlib=$CTSM_LIBS

make -j$(nproc)
make install

# 设置环境变量
export HDF5_DIR=$CTSM_LIBS
export LD_LIBRARY_PATH=$CTSM_LIBS/lib:$LD_LIBRARY_PATH
```

### 3.6 安装 NetCDF-C

```bash
cd ~/ctsm_build
wget https://downloads.unidata.ucar.edu/netcdf-c/4.9.2/netcdf-c-4.9.2.tar.gz
tar -xzf netcdf-c-4.9.2.tar.gz
cd netcdf-c-4.9.2

CC=mpicc \
CPPFLAGS="-I$CTSM_LIBS/include" \
LDFLAGS="-L$CTSM_LIBS/lib" \
./configure --prefix=$CTSM_LIBS \
    --enable-netcdf-4 \
    --enable-parallel4 \
    --enable-shared \
    --disable-dap

make -j$(nproc)
make install

export NETCDF=$CTSM_LIBS
export PATH=$CTSM_LIBS/bin:$PATH
```

### 3.7 安装 NetCDF-Fortran

```bash
cd ~/ctsm_build
wget https://downloads.unidata.ucar.edu/netcdf-fortran/4.6.1/netcdf-fortran-4.6.1.tar.gz
tar -xzf netcdf-fortran-4.6.1.tar.gz
cd netcdf-fortran-4.6.1

CC=mpicc FC=mpifort F77=mpifort \
CPPFLAGS="-I$CTSM_LIBS/include" \
LDFLAGS="-L$CTSM_LIBS/lib" \
LIBS="-lnetcdf -lhdf5_hl -lhdf5 -lz" \
./configure --prefix=$CTSM_LIBS \
    --enable-shared

make -j$(nproc)
make install
```

### 3.8 安装 PnetCDF（可选，用于并行 I/O）

```bash
cd ~/ctsm_build
wget https://parallel-netcdf.github.io/Release/pnetcdf-1.12.3.tar.gz
tar -xzf pnetcdf-1.12.3.tar.gz
cd pnetcdf-1.12.3

CC=mpicc CXX=mpicxx FC=mpifort F77=mpifort \
./configure --prefix=$CTSM_LIBS \
    --enable-shared

make -j$(nproc)
make install

export PNETCDF=$CTSM_LIBS
```

### 3.9 安装 ESMF

```bash
cd ~/ctsm_build
wget https://github.com/esmf-org/esmf/archive/refs/tags/v8.6.0.tar.gz
tar -xzf v8.6.0.tar.gz
cd esmf-8.6.0

# 设置 ESMF 编译环境（使用 MPICH4）
export ESMF_DIR=$PWD
export ESMF_INSTALL_PREFIX=$CTSM_LIBS/esmf
export ESMF_COMPILER=intel
export ESMF_COMM=mpich          # 使用 mpich 而非 intelmpi
export ESMF_NETCDF=nc-config
export ESMF_NETCDF_INCLUDE=$CTSM_LIBS/include
export ESMF_NETCDF_LIBPATH=$CTSM_LIBS/lib
export ESMF_PNETCDF=pnetcdf-config
export ESMF_PNETCDF_INCLUDE=$CTSM_LIBS/include
export ESMF_PNETCDF_LIBPATH=$CTSM_LIBS/lib

# 编译安装
make -j$(nproc)
make install

# 设置 ESMF 环境变量（注意路径包含 mpich）
export ESMFMKFILE=$ESMF_INSTALL_PREFIX/lib/libO/Linux.intel.64.mpich.default/esmf.mk
```

### 3.10 验证依赖库安装

```bash
# 检查 MPICH4
mpifort --version
mpirun --version

# 检查 NetCDF
nc-config --all
nf-config --all

# 检查 HDF5
h5pfc --version

# 检查 ESMF
cat $ESMFMKFILE | grep ESMF_VERSION
```

### 3.11 创建依赖库环境脚本

```bash
cat > $CTSM_LIBS/ctsm_libs_env.sh << 'EOF'
#!/bin/bash
# CTSM Libraries Environment (with MPICH4)

export CTSM_LIBS=/opt/ctsm_libs

# MPICH4 路径
export MPICH_ROOT=$CTSM_LIBS/mpich4
export PATH=$MPICH_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPICH_ROOT/lib:$LD_LIBRARY_PATH
export MANPATH=$MPICH_ROOT/share/man:$MANPATH

# 其他库路径设置
export PATH=$CTSM_LIBS/bin:$PATH
export LD_LIBRARY_PATH=$CTSM_LIBS/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CTSM_LIBS/lib:$LIBRARY_PATH
export CPATH=$CTSM_LIBS/include:$CPATH

# NetCDF
export NETCDF=$CTSM_LIBS
export NETCDF_PATH=$CTSM_LIBS

# HDF5
export HDF5_DIR=$CTSM_LIBS

# PnetCDF
export PNETCDF=$CTSM_LIBS

# ESMF (使用 mpich 路径)
export ESMF_ROOT=$CTSM_LIBS/esmf
export ESMFMKFILE=$ESMF_ROOT/lib/libO/Linux.intel.64.mpich.default/esmf.mk

echo "CTSM libraries environment loaded (with MPICH4)"
echo "MPICH: $MPICH_ROOT"
echo "NETCDF: $NETCDF"
echo "ESMFMKFILE: $ESMFMKFILE"
EOF

chmod +x $CTSM_LIBS/ctsm_libs_env.sh
```

---

## 4. CTSM 源代码获取

### 4.1 克隆 CTSM 仓库

```bash
# 选择安装目录
export CTSMROOT=$HOME/CTSM
cd $HOME

# 克隆 CTSM
git clone https://github.com/ESCOMP/CTSM.git $CTSMROOT
cd $CTSMROOT

# 查看可用的版本标签
git tag -l 'ctsm5.4*'

# 切换到稳定版本（推荐）
git checkout ctsm5.4.0

# 或使用最新开发版
# git checkout master
```

### 4.2 获取外部组件

```bash
cd $CTSMROOT

# 使用 git-fleximod 获取所有子模块
./bin/git-fleximod update

# 检查子模块状态
./bin/git-fleximod status
```

### 4.3 验证目录结构

```bash
ls -la $CTSMROOT

# 应该看到以下主要目录:
# bld/           - 构建配置
# cime/          - CIME 框架
# cime_config/   - CIME 配置
# components/    - 子组件 (CMEPS, CDEPS, MOSART 等)
# doc/           - 文档
# python/        - Python 工具
# src/           - 源代码
# tools/         - 工具脚本
```

---

## 5. Python 环境配置

### 5.1 安装 Conda/Mamba

如果系统没有 Conda，先安装 Miniconda：

```bash
cd ~/ctsm_build
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
source $HOME/miniconda3/bin/activate

# 初始化 conda
conda init bash
source ~/.bashrc

# 安装 mamba（更快的包管理器）
conda install -c conda-forge mamba -y
```

### 5.2 创建 CTSM Python 环境

```bash
cd $CTSMROOT

# 使用提供的脚本创建环境
./py_env_create

# 或手动创建
mamba env create -f python/conda_env_ctsm_py.yml

# 激活环境
conda activate ctsm_pylib
```

### 5.3 安装必需的 Perl 模块

CTSM 的 namelist 构建工具需要 Perl 的 XML::LibXML 模块：

```bash
# 方式一：在 conda 环境中安装（推荐）
conda activate ctsm_pylib
conda install -c conda-forge perl-xml-libxml

# 方式二：使用系统包管理器
# CentOS/RHEL:
sudo yum install perl-XML-LibXML

# Ubuntu/Debian:
sudo apt install libxml-libxml-perl

# 方式三：使用 cpan
cpan XML::LibXML
```

验证安装：
```bash
perl -e "use XML::LibXML; print 'XML::LibXML installed successfully\n';"
```

### 5.4 验证 Python 环境

```bash
conda activate ctsm_pylib

# 检查关键包
python -c "import xarray; print(f'xarray: {xarray.__version__}')"
python -c "import netCDF4; print(f'netCDF4: {netCDF4.__version__}')"
python -c "import scipy; print(f'scipy: {scipy.__version__}')"
```

---

## 6. CIME 机器配置

### 6.1 理解 CIME 配置结构

CIME 使用 XML 文件配置机器、编译器和批处理系统。关键文件位于：

```
$CTSMROOT/ccs_config/machines/
├── config_machines.xml      # 机器定义
├── config_compilers.xml     # 编译器配置
└── config_batch.xml         # 批处理系统配置
```

### 6.2 为自定义机器创建配置

创建 `~/.cime` 目录存放自定义配置：

```bash
mkdir -p ~/.cime
```

#### 6.2.1 创建 config_machines.xml

```bash
cat > ~/.cime/config_machines.xml << 'EOF'
<?xml version="1.0"?>
<config_machines version="2.0">
  <machine MACH="mylinux">
    <DESC>Custom Linux machine with Intel oneAPI + MPICH4</DESC>
    <NODENAME_REGEX>.*</NODENAME_REGEX>
    <OS>LINUX</OS>
    <COMPILERS>intel</COMPILERS>
    <MPILIBS>mpich</MPILIBS>
    <PROJECT>ctsm</PROJECT>
    <SAVE_TIMING_DIR/>
    <CIME_OUTPUT_ROOT>$ENV{HOME}/ctsm_cases</CIME_OUTPUT_ROOT>
    <DIN_LOC_ROOT>$ENV{HOME}/cesm_inputdata</DIN_LOC_ROOT>
    <DIN_LOC_ROOT_CLMFORC>$ENV{HOME}/cesm_inputdata</DIN_LOC_ROOT_CLMFORC>
    <DOUT_S_ROOT>$ENV{HOME}/ctsm_archive/$CASE</DOUT_S_ROOT>
    <BASELINE_ROOT>$ENV{HOME}/ctsm_baselines</BASELINE_ROOT>
    <CCSM_CPRNC>$ENV{HOME}/ctsm_tools/cprnc</CCSM_CPRNC>
    <GMAKE>make</GMAKE>
    <GMAKE_J>8</GMAKE_J>
    <BATCH_SYSTEM>none</BATCH_SYSTEM>
    <SUPPORTED_BY>user</SUPPORTED_BY>
    <MAX_TASKS_PER_NODE>32</MAX_TASKS_PER_NODE>
    <MAX_MPITASKS_PER_NODE>32</MAX_MPITASKS_PER_NODE>
    <PROJECT_REQUIRED>FALSE</PROJECT_REQUIRED>
    <mpirun mpilib="mpich">
      <executable>mpirun</executable>
      <arguments>
        <arg name="ntasks">-np $TOTALPES</arg>
      </arguments>
    </mpirun>
    <mpirun mpilib="mpi-serial">
      <executable/>
    </mpirun>
    <module_system type="none"/>
    <environment_variables>
      <env name="OMP_STACKSIZE">256M</env>
    </environment_variables>
    <environment_variables comp_interface="nuopc">
      <env name="ESMFMKFILE">/opt/ctsm_libs/esmf/lib/libO/Linux.intel.64.mpich.default/esmf.mk</env>
    </environment_variables>
  </machine>
</config_machines>
EOF
```

#### 6.2.2 创建 config_compilers.xml

```bash
cat > ~/.cime/config_compilers.xml << 'EOF'
<?xml version="1.0"?>
<config_compilers version="2.0">
  <compiler COMPILER="intel" MACH="mylinux">
    <CPPDEFS>
      <append> -DFORTRANUNDERSCORE -DNO_R16 -DCPRINTEL </append>
    </CPPDEFS>
    <CFLAGS>
      <base> -O2 -fp-model precise </base>
      <append DEBUG="TRUE"> -g -O0 </append>
    </CFLAGS>
    <FFLAGS>
      <base> -O2 -fp-model precise -convert big_endian -assume byterecl -ftz -traceback -xHost </base>
      <append DEBUG="TRUE"> -g -O0 -check bounds -check pointers -check uninit -fpe0 </append>
    </FFLAGS>
    <FFLAGS_NOOPT>
      <base> -O0 </base>
    </FFLAGS_NOOPT>
    <FC>mpifort</FC>
    <CC>mpicc</CC>
    <CXX>mpicxx</CXX>
    <MPIFC>mpifort</MPIFC>
    <MPICC>mpicc</MPICC>
    <MPICXX>mpicxx</MPICXX>
    <LDFLAGS>
      <base> -L/opt/ctsm_libs/lib -L/opt/ctsm_libs/mpich4/lib </base>
    </LDFLAGS>
    <SLIBS>
      <base> -L/opt/ctsm_libs/lib -lnetcdff -lnetcdf -lpnetcdf -lhdf5_hl -lhdf5 -lz </base>
    </SLIBS>
    <NETCDF_PATH>/opt/ctsm_libs</NETCDF_PATH>
    <PNETCDF_PATH>/opt/ctsm_libs</PNETCDF_PATH>
    <PIO_FILESYSTEM_HINTS>lustre</PIO_FILESYSTEM_HINTS>
    <SUPPORTS_CXX>TRUE</SUPPORTS_CXX>
  </compiler>
</config_compilers>
EOF
```

### 6.3 创建必要的目录

```bash
# 创建输入数据目录
mkdir -p $HOME/cesm_inputdata

# 创建案例输出目录
mkdir -p $HOME/ctsm_cases

# 创建归档目录
mkdir -p $HOME/ctsm_archive

# 创建基线目录
mkdir -p $HOME/ctsm_baselines

# 创建工具目录
mkdir -p $HOME/ctsm_tools
```

### 6.4 验证机器配置

```bash
cd $CTSMROOT/cime/scripts

# 查询可用机器
./query_config --machines

# 应该能看到 mylinux 在列表中
```

---

## 7. 创建和运行 Case

### 7.1 加载环境

```bash
# 加载 Intel oneAPI 环境
source ~/intel_ctsm_env.sh

# 加载依赖库环境
source /opt/ctsm_libs/ctsm_libs_env.sh

# 激活 Python 环境
conda activate ctsm_pylib
```

### 7.2 查询可用 Compsets

```bash
cd $CTSMROOT/cime/scripts

# 列出所有 CLM compsets
./query_config --compsets clm

# 常用 compsets 说明:
# I2000Clm60BgcCrop    - CTSM 6.0 BGC + 作物模式，2000年气候
# I2000Clm60Sp         - CTSM 6.0 卫星物候模式，2000年气候
# I1850Clm60BgcCrop    - CTSM 6.0 BGC + 作物模式，1850年气候
# IHistClm60BgcCrop    - CTSM 6.0 BGC + 作物模式，历史气候
```

### 7.3 查询可用分辨率

```bash
# 列出所有网格
./query_config --grids

# 常用分辨率:
# f09_g17      - 0.9x1.25度（约100km）
# f19_g17      - 1.9x2.5度（约200km）
# f45_g37      - 4x5度（约450km）
# ne30_g17     - SE动态核心 约1度
```

### 7.4 创建新 Case

```bash
cd $CTSMROOT/cime/scripts

# 创建一个简单的测试案例
./create_newcase \
    --case $HOME/ctsm_cases/test_I2000 \
    --compset I2000Clm60BgcCrop \
    --res f45_g37 \
    --mach mylinux \
    --run-unsupported
```

### 7.5 配置 Case

```bash
cd $HOME/ctsm_cases/test_I2000

# 查看当前配置
./xmlquery --listall

# 修改运行配置
./xmlchange STOP_OPTION=ndays
./xmlchange STOP_N=5
./xmlchange NTASKS=4
./xmlchange NTHRDS=1

# 设置调试模式（首次运行推荐）
./xmlchange DEBUG=TRUE

# 配置数据目录
./xmlchange DIN_LOC_ROOT=$HOME/cesm_inputdata
```

### 7.6 设置 Case

```bash
cd $HOME/ctsm_cases/test_I2000

# 运行 case.setup
./case.setup
```

### 7.7 下载输入数据

```bash
# 检查所需数据
./check_input_data

# 下载缺失的数据
./check_input_data --download
```

### 7.8 编译 Case

```bash
cd $HOME/ctsm_cases/test_I2000

# 编译模型（可能需要30-60分钟）
./case.build

# 如果编译失败，使用详细模式重试
./case.build --verbose

# 清理后重新编译
# ./case.build --clean-all
# ./case.build
```

### 7.9 运行 Case

```bash
cd $HOME/ctsm_cases/test_I2000

# 交互式运行（本地机器）
./case.submit

# 或直接运行
./case.run --skip-preview-namelist

# 检查运行状态
tail -f $HOME/ctsm_cases/test_I2000/run/cesm.log.*
```

### 7.10 检查输出

```bash
# 运行完成后检查输出
ls -la $HOME/ctsm_cases/test_I2000/run/

# 历史文件位置
ls $HOME/ctsm_cases/test_I2000/run/*.h0.*.nc

# 使用 ncdump 查看输出
ncdump -h $HOME/ctsm_cases/test_I2000/run/*.h0.*.nc | head -100
```

---

## 8. 常见问题排查

### 8.1 编译错误

#### 问题: "Cannot find module xxx"

```bash
# 检查 NetCDF-Fortran 是否正确安装
nf-config --all

# 确保环境变量正确设置
echo $NETCDF
echo $LD_LIBRARY_PATH
```

#### 问题: Intel 编译器许可证问题

```bash
# 检查许可证状态
ifort -diag-license

# 使用免费的 Intel oneAPI Base 版本不需要许可证
```

#### 问题: ESMF 找不到

```bash
# 确保 ESMFMKFILE 正确设置
echo $ESMFMKFILE
ls -la $ESMFMKFILE

# 检查 ESMF 编译是否正确
grep ESMF_VERSION $ESMFMKFILE
```

### 8.2 运行时错误

#### 问题: 找不到输入数据

```bash
# 检查数据路径
./xmlquery DIN_LOC_ROOT

# 手动下载特定数据
./check_input_data --download --inputdata $HOME/cesm_inputdata
```

#### 问题: MPI 错误

```bash
# 检查 MPI 配置
which mpirun
mpirun --version

# 测试 MPI
mpirun -np 4 hostname
```

#### 问题: libpciaccess.so.0 找不到

如果使用 `--with-device=ch4:ofi` 编译的 MPICH4，运行时可能报错：
```
mpirun: error while loading shared libraries: libpciaccess.so.0: cannot open shared object file: No such file or directory
```

**解决方案一：安装 libpciaccess**

```bash
# 使用系统包管理器（需要 sudo）
sudo yum install libpciaccess libpciaccess-devel  # CentOS/RHEL
sudo apt install libpciaccess0 libpciaccess-dev   # Ubuntu/Debian

# 或使用 conda（无需 sudo）
conda install -c conda-forge libpciaccess
# 然后在运行脚本中添加 conda 库路径：
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
```

**解决方案二：重新编译 MPICH4 使用 ch3:sock**

```bash
cd /path/to/mpich-4.x.x
make clean
./configure --prefix=/your/mpich4/path \
    CC=icc CXX=icpc FC=ifort F77=ifort \
    --with-device=ch3:sock \
    --enable-fortran=all \
    --enable-shared
make -j 8
make install
```

#### 问题: Conda MPI 库冲突

如果同时安装了 conda 和自编译的 MPICH4，可能出现库冲突导致 symbol lookup 错误。

**解决方案：** 在运行脚本中完全重置 LD_LIBRARY_PATH，确保自编译库优先：

```bash
# 在 LSF/PBS/Slurm 提交脚本中
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV

# 完全重置而非追加
export LD_LIBRARY_PATH=/your/mpich4/lib:/your/ctsm_libs/lib:/path/to/intel/lib:/usr/lib64
```

#### 问题: 内存不足

```bash
# 减少并行任务数
./xmlchange NTASKS=2

# 或使用更低分辨率
# 重新创建 case 使用 f45_g37 替代 f09_g17
```

### 8.3 PIO 模块文件不兼容错误

#### 问题: "This module file was not generated by any release of this compiler"

```
error #7013: This module file was not generated by any release of this compiler. [PIO]
```

**原因：** PIO 的 `.mod` 文件是用不同版本的 Intel 编译器编译的，与当前使用的编译器版本不兼容。

**解决方案：**

```bash
# 1. 进入 Case 目录
cd /path/to/your/case

# 2. 彻底清理所有构建产物
./case.build --clean-all

# 3. 删除整个 bld 目录（包括所有缓存的 .mod 文件）
rm -rf bld

# 4. 确保环境干净（避免 conda 干扰）
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV
# 如果 conda 的 MPI 包装在 PATH 中，需要排除
export PATH=$(echo $PATH | tr ':' '\n' | grep -v conda | grep -v miniconda | tr '\n' ':')

# 5. 检查确认使用正确的编译器
which ifort
ifort --version
which mpifort

# 6. 重新运行 case.setup
./case.setup

# 7. 重新编译
./case.build
```

**重要提示：**

- Fortran `.mod` 文件是编译器特定的，不能跨编译器版本使用
- 如果更换了编译器版本，必须完全重新编译所有 Fortran 代码
- 确保 `~/.cime/config_compilers.xml` 中的编译器设置与实际使用的一致

#### 如果仍然失败

检查是否有旧的 PIO 模块文件残留：

```bash
# 搜索所有 PIO 相关的 .mod 文件
find . -name "pio*.mod" -type f 2>/dev/null
find . -name "*.mod" -path "*pio*" -type f 2>/dev/null

# 删除它们
find . -name "pio*.mod" -type f -delete 2>/dev/null
find . -name "*.mod" -path "*pio*" -type f -delete 2>/dev/null

# 重新编译
./case.build
```

### 8.4 常见警告

#### ESMF 警告

大多数 ESMF 警告可以忽略，除非导致运行失败。

#### 浮点数警告

```bash
# 对于调试，启用严格的浮点检查
./xmlchange DEBUG=TRUE

# 生产运行时可关闭
./xmlchange DEBUG=FALSE
```

---

## 9. 附录：环境变量速查表

### 9.1 Intel oneAPI + MPICH4 环境变量

```bash
# Intel 编译器
export CC=icc                # C 编译器
export CXX=icpc              # C++ 编译器
export FC=ifort              # Fortran 编译器
export F77=ifort             # Fortran 77 编译器
export F90=ifort             # Fortran 90 编译器

# 使用新的 LLVM 编译器（Intel oneAPI 2023+）
# export CC=icx
# export CXX=icpx
# export FC=ifx

# MPICH4 路径
export MPICH_ROOT=/opt/ctsm_libs/mpich4
export PATH=$MPICH_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPICH_ROOT/lib:$LD_LIBRARY_PATH

# MPI 包装（MPICH4 风格）
export MPICC=mpicc
export MPICXX=mpicxx
export MPIFC=mpifort
export MPIF90=mpifort
```

### 9.2 库路径环境变量

```bash
export CTSM_LIBS=/opt/ctsm_libs
export PATH=$CTSM_LIBS/bin:$PATH
export LD_LIBRARY_PATH=$CTSM_LIBS/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CTSM_LIBS/lib:$LIBRARY_PATH
export CPATH=$CTSM_LIBS/include:$CPATH
```

### 9.3 NetCDF 环境变量

```bash
export NETCDF=$CTSM_LIBS
export NETCDF_PATH=$CTSM_LIBS
export NETCDF_C_PATH=$CTSM_LIBS
export NETCDF_FORTRAN_PATH=$CTSM_LIBS
```

### 9.4 ESMF 环境变量

```bash
export ESMF_DIR=/path/to/esmf/source
export ESMF_INSTALL_PREFIX=$CTSM_LIBS/esmf
export ESMF_COMPILER=intel
export ESMF_COMM=mpich                    # 使用 mpich 而非 intelmpi
export ESMFMKFILE=$ESMF_INSTALL_PREFIX/lib/libO/Linux.intel.64.mpich.default/esmf.mk
```

### 9.5 CTSM/CIME 环境变量

```bash
export CTSMROOT=$HOME/CTSM
export CIMEROOT=$CTSMROOT/cime
export DIN_LOC_ROOT=$HOME/cesm_inputdata
export CIME_OUTPUT_ROOT=$HOME/ctsm_cases
```

### 9.6 完整环境加载脚本

```bash
cat > ~/load_ctsm_env.sh << 'EOF'
#!/bin/bash
# Complete CTSM Environment Loading Script (Intel oneAPI + MPICH4)

echo "Loading CTSM environment..."

# 1. Load Intel oneAPI (only compiler, not Intel MPI)
source /opt/intel/oneapi/compiler/latest/env/vars.sh 2>/dev/null
source /opt/intel/oneapi/mkl/latest/env/vars.sh 2>/dev/null

# 2. Set Intel compilers
export CC=icc
export CXX=icpc
export FC=ifort
export F77=ifort
export F90=ifort

# 3. Load MPICH4
export CTSM_LIBS=/opt/ctsm_libs
export MPICH_ROOT=$CTSM_LIBS/mpich4
export PATH=$MPICH_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$MPICH_ROOT/lib:$LD_LIBRARY_PATH
export MANPATH=$MPICH_ROOT/share/man:$MANPATH

# 4. Set MPI compilers (MPICH4 style)
export MPICC=mpicc
export MPICXX=mpicxx
export MPIFC=mpifort
export MPIF90=mpifort

# 5. Load other libraries
export PATH=$CTSM_LIBS/bin:$PATH
export LD_LIBRARY_PATH=$CTSM_LIBS/lib:$LD_LIBRARY_PATH
export LIBRARY_PATH=$CTSM_LIBS/lib:$LIBRARY_PATH
export CPATH=$CTSM_LIBS/include:$CPATH

# 6. NetCDF
export NETCDF=$CTSM_LIBS
export NETCDF_PATH=$CTSM_LIBS

# 7. ESMF (with mpich path)
export ESMF_ROOT=$CTSM_LIBS/esmf
export ESMFMKFILE=$ESMF_ROOT/lib/libO/Linux.intel.64.mpich.default/esmf.mk

# 8. CTSM paths
export CTSMROOT=$HOME/CTSM
export CIMEROOT=$CTSMROOT/cime
export DIN_LOC_ROOT=$HOME/cesm_inputdata

# 9. Activate conda environment
conda activate ctsm_pylib 2>/dev/null

echo "Environment loaded successfully!"
echo "  Fortran Compiler: $(which ifort 2>/dev/null || echo 'Not found')"
echo "  MPI Fortran: $(which mpifort 2>/dev/null || echo 'Not found')"
echo "  MPICH: $MPICH_ROOT"
echo "  NetCDF: $NETCDF"
echo "  ESMFMKFILE: $ESMFMKFILE"
echo "  CTSMROOT: $CTSMROOT"
EOF

chmod +x ~/load_ctsm_env.sh
```

---

## 附录：参考资源

### 官方文档

- CTSM 文档: https://escomp.github.io/CTSM/
- CESM 文档: https://www.cesm.ucar.edu/models/cesm2/
- CIME 文档: https://esmci.github.io/cime/
- ESMF 文档: https://earthsystemmodeling.org/docs/

### Intel oneAPI 资源

- Intel oneAPI 文档: https://www.intel.com/content/www/us/en/developer/tools/oneapi/documentation.html
- Fortran 编译器指南: https://www.intel.com/content/www/us/en/docs/fortran-compiler/

### MPICH 资源

- MPICH 官方网站: https://www.mpich.org/
- MPICH 文档: https://www.mpich.org/documentation/guides/
- MPICH 下载: https://www.mpich.org/downloads/

### 社区支持

- CTSM GitHub Issues: https://github.com/ESCOMP/CTSM/issues
- CESM 论坛: https://bb.cgd.ucar.edu/cesm/

---

**文档版本**: 1.0
**最后更新**: 2025年12月
**作者**: CTSM 用户指南
