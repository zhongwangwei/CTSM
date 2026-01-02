# CTSM 模型运行手册

## 基于 Intel oneAPI + MPICH4 环境

---

## 目录

1. [运行前准备](#1-运行前准备)
2. [创建 Case](#2-创建-case)
3. [配置 Case](#3-配置-case)
4. [编译 Case](#4-编译-case)
5. [运行模型](#5-运行模型)
6. [检查输出](#6-检查输出)
7. [常见问题与解决方案](#7-常见问题与解决方案)
8. [LSF 批处理提交](#8-lsf-批处理提交)

---

## 1. 运行前准备

### 1.1 设置环境变量

**重要：每次运行前都需要设置干净的环境！**

```bash
# 清除 Conda 环境变量（避免库冲突）
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV

# 设置库路径（根据实际安装路径修改）
export CTSM_LIBS=/share/home/dq013/software/ctsm_libs
export MPICH_ROOT=$CTSM_LIBS/mpich4

# 设置 PATH
export PATH=$MPICH_ROOT/bin:$CTSM_LIBS/bin:$PATH

# 设置 LD_LIBRARY_PATH（完全重置，不要追加）
export LD_LIBRARY_PATH=$MPICH_ROOT/lib:$CTSM_LIBS/lib:/share/home/dq013/software/intel/oneapi/compiler/2022.0.2/linux/compiler/lib/intel64_lin:/share/home/dq013/software/intel/oneapi/mkl/2022.0.2/lib/intel64:/usr/lib64

# 设置 NetCDF 路径
export NETCDF=$CTSM_LIBS

# 激活 Python 环境（仅用于 CIME 脚本）
conda activate ctsm_pylib
```

### 1.2 验证环境

```bash
# 检查编译器
which ifort
ifort --version

# 检查 MPI
which mpirun
mpirun --version
which mpifort

# 检查 NetCDF
nc-config --all
nf-config --all
```

---

## 2. 创建 Case

### 2.1 进入 CIME scripts 目录

```bash
cd /share/home/dq013/zhwei/CTSM/CTSM/cime/scripts
```

### 2.2 查询可用配置

```bash
# 查看可用 compsets
./query_config --compsets clm

# 查看可用分辨率
./query_config --grids

# 查看可用机器
./query_config --machines
```

### 2.3 常用 Compsets

| Compset | 描述 |
|---------|------|
| I2000Clm60Sp | CTSM 6.0 卫星物候，2000年气候 |
| I2000Clm60BgcCrop | CTSM 6.0 BGC + 作物，2000年气候 |
| IHistClm60Sp | CTSM 6.0 卫星物候，历史气候 |
| IHistClm60BgcCrop | CTSM 6.0 BGC + 作物，历史气候 |
| I1850Clm60Sp | CTSM 6.0 卫星物候，1850年气候 |

### 2.4 常用分辨率

| 分辨率 | 描述 | 内存需求 |
|--------|------|----------|
| f45_g37 | 4x5度 (~450km) | 低 |
| f19_g17 | 1.9x2.5度 (~200km) | 中 |
| f09_g17 | 0.9x1.25度 (~100km) | 高 |

### 2.5 创建新 Case

```bash
# 示例：创建一个简单测试 case
./create_newcase \
    --case /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test \
    --res f45_g37 \
    --compset I2000Clm60Sp \
    --mach mylinux \
    --compiler intel \
    --run-unsupported
```

---

## 3. 配置 Case

### 3.1 进入 Case 目录

```bash
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test
```

### 3.2 基本配置

```bash
# 运行时长设置
./xmlchange STOP_OPTION=ndays    # 单位：ndays, nmonths, nyears
./xmlchange STOP_N=5             # 运行 5 天

# 并行设置
./xmlchange NTASKS=4             # MPI 进程数
./xmlchange NTHRDS=1             # 每个进程的 OpenMP 线程数

# 调试模式（首次运行建议关闭以避免严格浮点检查）
./xmlchange DEBUG=FALSE

# 输入数据目录
./xmlchange DIN_LOC_ROOT=/share/home/dq013/zhwei/CTSM/CTSM_data/cesm_inputdata

# 冷启动（从初始状态开始）
./xmlchange CLM_FORCE_COLDSTART=on

# PIO I/O 类型（推荐 netcdf）
./xmlchange PIO_TYPENAME=netcdf
```

### 3.3 高级配置选项

```bash
# 查看所有配置
./xmlquery --listall

# 查看特定配置
./xmlquery STOP_OPTION STOP_N NTASKS

# 重启设置
./xmlchange CONTINUE_RUN=FALSE   # FALSE=新运行，TRUE=继续运行
./xmlchange RESUBMIT=0           # 自动重新提交次数

# 历史文件输出频率
./xmlchange HIST_OPTION=nmonths
./xmlchange HIST_N=1
```

### 3.4 运行 case.setup

```bash
./case.setup
```

### 3.5 预览 Namelists

```bash
./preview_namelists
```

---

## 4. 编译 Case

### 4.1 确保环境干净

```bash
# 在编译前设置环境
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV
export LD_LIBRARY_PATH=/share/home/dq013/software/ctsm_libs/mpich4/lib:/share/home/dq013/software/ctsm_libs/lib:/share/home/dq013/software/intel/oneapi/compiler/2022.0.2/linux/compiler/lib/intel64_lin:/share/home/dq013/software/intel/oneapi/mkl/2022.0.2/lib/intel64:/usr/lib64
export PATH=/share/home/dq013/software/ctsm_libs/mpich4/bin:/share/home/dq013/software/ctsm_libs/bin:$PATH
```

### 4.2 编译

```bash
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test

# 首次编译
./case.build

# 如果需要重新编译
./case.build --clean-all
./case.build
```

### 4.3 修复 PIO PnetCDF 支持（如需要）

如果遇到 "Bad IO type" 错误（iotype=1），需要手动启用 PnetCDF：

```bash
# 找到 PIO 构建目录
PIO_DIR=$(find ./bld -type d -name "pio2" | head -1)
cd $PIO_DIR

# 启用 PnetCDF
cmake . -DWITH_PNETCDF=TRUE \
    -DPnetCDF_PATH=/share/home/dq013/software/ctsm_libs \
    -DPnetCDF_C_LIBRARY=/share/home/dq013/software/ctsm_libs/lib/libpnetcdf.so \
    -DPnetCDF_C_INCLUDE_DIR=/share/home/dq013/software/ctsm_libs/include

make -j4

# 验证
grep "_PNETCDF" config.h
# 应该看到: #define _PNETCDF

# 返回 case 目录重新链接
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test
./case.build
```

---

## 5. 运行模型

### 5.1 交互式运行（测试用）

```bash
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test

# 设置环境
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV
export LD_LIBRARY_PATH=/share/home/dq013/software/ctsm_libs/mpich4/lib:/share/home/dq013/software/ctsm_libs/lib:/share/home/dq013/software/intel/oneapi/compiler/2022.0.2/linux/compiler/lib/intel64_lin:/share/home/dq013/software/intel/oneapi/mkl/2022.0.2/lib/intel64:/usr/lib64
export PATH=/share/home/dq013/software/ctsm_libs/mpich4/bin:$PATH

# 进入 run 目录
cd run

# 运行
mpirun -np 4 ../bld/cesm.exe 2>&1 | tee run_output.txt
```

### 5.2 使用 case.submit

```bash
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test
./case.submit
```

---

## 6. 检查输出

### 6.1 查看日志文件

```bash
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test/run

# 主日志
cat cesm.log.* | tail -50

# 陆面模型日志
cat lnd.log.* | tail -50

# 大气模型日志
cat atm.log.* | tail -50

# 耦合器日志
cat cpl.log.* | tail -50
```

### 6.2 查看输出文件

```bash
# 历史文件
ls -la *.h0.*.nc

# 重启文件
ls -la *.r.*.nc

# 使用 ncdump 查看变量
ncdump -h *.h0.*.nc | head -100
```

### 6.3 检查运行是否成功

```bash
# 查看时间信息
grep "model date" lnd.log.* | tail -5

# 检查是否有错误
grep -i "error\|abort\|fail" *.log.* | tail -20
```

---

## 7. 常见问题与解决方案

### 7.1 Bad IO type (err_num = -500)

**原因：** PIO 编译时未启用 PnetCDF 或 NetCDF 支持

**解决方案：**
```bash
# 检查 PIO 配置
find ./bld -name "config.h" -path "*/pio/*" -exec grep "_PNETCDF\|_NETCDF" {} \;

# 如果缺少支持，按照 4.3 节修复 PIO
```

### 7.2 Floating Invalid 错误

**原因：** DEBUG 模式启用了严格浮点检查

**解决方案：**
```bash
./xmlchange DEBUG=FALSE
./case.build --clean-all
./case.build
```

### 7.3 Segmentation Fault

**可能原因：**
1. 内存不足
2. 库版本不兼容
3. 数据文件问题

**解决方案：**
```bash
# 减少并行任务数
./xmlchange NTASKS=2

# 使用更低分辨率
# 重新创建 case 使用 f45_g37

# 设置环境变量忽略异常
export FOR_IGNORE_EXCEPTIONS=1
export KMP_INIT_AT_FORK=FALSE
```

### 7.4 找不到输入数据

**解决方案：**
```bash
# 检查数据路径
./xmlquery DIN_LOC_ROOT

# 下载缺失数据
./check_input_data --download
```

### 7.5 PIO2 retry NETCDF 警告

**说明：** 这是正常的降级行为，PIO 从 netcdf4p 降级到基本 netcdf

**如果想消除警告：**
```bash
./xmlchange PIO_TYPENAME=netcdf
./preview_namelists
```

### 7.6 Conda MPI 库冲突

**症状：** symbol lookup error 或加载了错误的 libmpi

**解决方案：**
```bash
# 完全重置环境变量
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV

# 重置 LD_LIBRARY_PATH（不要追加，直接设置）
export LD_LIBRARY_PATH=/your/mpich4/lib:/your/ctsm_libs/lib:/intel/lib:/usr/lib64
```

---

## 8. LSF 批处理提交

### 8.1 创建提交脚本

```bash
cat > /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test/run_ctsm.lsf << 'EOF'
#!/bin/bash
#BSUB -J ctsm_run
#BSUB -q normal
#BSUB -n 4
#BSUB -o ctsm_%J.out
#BSUB -e ctsm_%J.err
#BSUB -R "span[ptile=4]"

# 避免 Conda 库冲突
unset CONDA_PREFIX
unset CONDA_DEFAULT_ENV

# 设置库路径
export LD_LIBRARY_PATH=/share/home/dq013/software/ctsm_libs/mpich4/lib:/share/home/dq013/software/ctsm_libs/lib:/share/home/dq013/software/intel/oneapi/compiler/2022.0.2/linux/compiler/lib/intel64_lin:/share/home/dq013/software/intel/oneapi/mkl/2022.0.2/lib/intel64:/usr/lib64

export PATH=/share/home/dq013/software/ctsm_libs/mpich4/bin:/share/home/dq013/software/ctsm_libs/bin:$PATH

# 进入运行目录
CASEROOT=/share/home/dq013/zhwei/CTSM/ctsm_cases/my_test
RUNDIR=$CASEROOT/run
EXEROOT=$CASEROOT/bld

cd $RUNDIR

# 运行模型
mpirun -np 4 $EXEROOT/cesm.exe

echo "CTSM run completed at $(date)"
EOF
```

### 8.2 提交作业

```bash
cd /share/home/dq013/zhwei/CTSM/ctsm_cases/my_test
bsub < run_ctsm.lsf
```

### 8.3 查看作业状态

```bash
# 查看队列
bjobs

# 查看作业详情
bjobs -l <job_id>

# 取消作业
bkill <job_id>
```

---

## 附录：快速参考

### 常用 xmlchange 命令

```bash
# 运行时长
./xmlchange STOP_OPTION=ndays STOP_N=5

# 并行设置
./xmlchange NTASKS=4 NTHRDS=1

# 调试模式
./xmlchange DEBUG=FALSE

# 重启设置
./xmlchange CONTINUE_RUN=TRUE

# 历史输出
./xmlchange HIST_OPTION=nmonths HIST_N=1

# 冷启动
./xmlchange CLM_FORCE_COLDSTART=on

# PIO 类型
./xmlchange PIO_TYPENAME=netcdf
```

### 环境变量快速设置

```bash
# 一行命令设置所有环境
unset CONDA_PREFIX && unset CONDA_DEFAULT_ENV && \
export LD_LIBRARY_PATH=/share/home/dq013/software/ctsm_libs/mpich4/lib:/share/home/dq013/software/ctsm_libs/lib:/share/home/dq013/software/intel/oneapi/compiler/2022.0.2/linux/compiler/lib/intel64_lin:/share/home/dq013/software/intel/oneapi/mkl/2022.0.2/lib/intel64:/usr/lib64 && \
export PATH=/share/home/dq013/software/ctsm_libs/mpich4/bin:/share/home/dq013/software/ctsm_libs/bin:$PATH
```

---

**文档版本**: 1.0
**最后更新**: 2026年1月
**作者**: CTSM 用户指南
