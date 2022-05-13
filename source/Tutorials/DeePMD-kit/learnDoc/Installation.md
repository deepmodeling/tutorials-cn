# 简易安装

DeePMD-kit有很多简单的安装方式，你可以按需选择。

完成安装流程后，文件中会出现两个已经编译好的程序：DeePMD-kit(`dp`)和LAMMPS(`lmp`)，你可以输入`dp -h`和`lmp -h`来获取帮助信息。考虑到并行训练模型和运行LAMMPS的需求，`mpirun`也将同时被编译。

- 安装离线软件包

- 使用conda安装

- 使用docker安装


## 利用离线软件包安装

CPU和GPU版本的离线软件包都在可以[the Releases page](https://github.com/deepmodeling/deepmd-kit/releases)中找到。

由于Github对文件大小的限制，部分安装包被拆分成两个文件，你可以在下载之后进行合并操作。
```bash
cat deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh.0 deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh.1 > 
deepmd-kit-2.0.0-cuda11.3_gpu-Linux-x86_64.sh
```
## 利用conda安装
DeePMD-kit可以利用[Conda](https://github.com/conda/conda)安装，首先需要安装[Anaconda](https://www.anaconda.com/products/individual#download-section)或[Miniconda](https://docs.conda.io/en/latest/miniconda.html)	。

安装Anaconda或Miniconda之后，你可以创建一个包含CPU版本的DeePMD-kit和LAMMPS的虚拟环境：
```bash
conda create -n deepmd deepmd-kit=*=*cpu libdeepmd=*=*cpu lammps-dp -c https://conda.deepmodeling.org
```
或者创建一个同时包含[CUDA Toolkit](https://docs.nvidia.com/deploy/cuda-compatibility/index.html#binary-compatibility__table-toolkit-driver)的GPU虚拟环境：
```bash
conda create -n deepmd deepmd-kit=*=*gpu libdeepmd=*=*gpu lammps-dp cudatoolkit=11.3 horovod -c https://conda.deepmodeling.org
```
CUDA Toolkit版本可以按需从10.1或11.3中选择。

安装指定版本的DeePMD-kit，例如2.0.0：
```bash
conda create -n deepmd deepmd-kit=2.0.0=*cpu libdeepmd=2.0.0=*cpu lammps-dp=2.0.0 horovod -c https://conda.deepmodeling.org
```
创建虚拟环境后，每次使用前需要激活环境：
```bash
conda activate deepmd
``` 

## 利用docker安装

DeePMD-kit同样可以通过docker安装。

获取CPU版本：
```bash
docker pull ghcr.io/deepmodeling/deepmd-kit:2.0.0_cpu
```
获取GPU版本：
```bash
docker pull ghcr.io/deepmodeling/deepmd-kit:2.0.0_cuda10.1_gpu
```    
获取ROCm版本：
```bash
docker pull deepmodeling/dpmdkit-rocm:dp2.0.3-rocm4.5.2-tf2.6-lmp29Sep2021
```
如果你想从源码开始安装DeePMD-kit，请点[此处](https://docs.deepmodeling.org/projects/deepmd/en/master/install/install-from-source.html)。

**翻译：区展鹏 校对：方满娣**
