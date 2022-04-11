# 上机教程(v2.0.3)
本教程将向您介绍 DeePMD-kit 的基本用法，以气相甲烷分子为例。 通常，DeePMD-kit 工作流程包含三个部分：数据准备、训练/冻结/压缩/测试和分子动力学。

DP 模型是使用 DeePMD-kit  (v2.0.3) 生成的。 使用 dpdata (v0.2.5) 工具将训练数据转换为 DeePMD-kit 的格式。 需要注意，dpdata 仅适用于 Python 3.5 及更高版本。 MD 模拟使用与 DeePMD-kit 自带的 LAMMPS（29 Sep 2021）进行。 dpdata 和 DeePMD-kit 安装和执行的详细信息可以在[DeepModeling 官方 GitHub 站点](https://github.com/deepmodeling) 中找到。 OVITO 用于 MD 轨迹的可视化。

本教程所需的文件可在 [此处](https://github.com/likefallwind/DPExample/raw/main/CH4.zip) 获得。 本教程的文件夹结构是这样的：
```bash
$ ls
00.data 01.train 02.lmp
```
 00.data 文件夹包含训练数据，文件夹 01.train 包含使用 DeePMD-kit 训练模型的示例脚本，文件夹 02.lmp 包含用于分子动力学模拟的 LAMMPS 示例脚本。

## 准备数据
DeePMD-kit的训练数据包含原子类型、模拟盒子、原子坐标、原子受力、系统能量和维里。 具有这些信息的分子系统的快照称为一帧。 一个System包含有许多具有相同原子数量和原子类型的帧。 例如，分子动力学轨迹可以转换为一个System，每个时间步长对应于系统中的一帧。

DeepPMD-kit 采用压缩数据格式。 所有的训练数据都应该首先转换成这种格式，然后才能被 DeePMD-kit 使用。 数据格式在 DeePMD-kit 手册中有详细说明，可查看[DeePMD-kit 官方网站](http://www.github.com/deepmodeling/deepmd-kit) 。

我们提供了一个名为 dpdata 的便捷工具，用于将 VASP、Gaussian、Quantum-Espresso、ABACUS 和 LAMMPS 生成的数据转换DeePMD-kit 的压缩格式。

例如，进入数据文件夹:
```bash
 $ cd 00.data
 $ ls 
OUTCAR
```
这里的OUTCAR 是通过使用 VASP 对气相甲烷分子进行从头算分子动力学 (AIMD) 模拟产生的。 现在进入python环境，例如
```bash
$ python
```
然后执行如下命令:
```python
import dpdata 
import numpy as np
data = dpdata.LabeledSystem('OUTCAR', fmt = 'vasp/outcar') 
print('# the data contains %d frames' % len(data))
```
在屏幕上，我们可以看到 OUTCAR 文件包含 200 帧数据。 我们随机选取 40 帧作为验证数据，其余的作为训练数据。
```python
index_validation = np.random.choice(200,size=40,replace=False)
index_training = list(set(range(200))-set(index_validation))
data_training = data.sub_system(index_training)
data_validation = data.sub_system(index_validation)
data_training.to_deepmd_npy('training_data')
data_validation.to_deepmd_npy('validation_data')
print('# the training data contains %d frames' % len(data_training)) 
print('# the validation data contains %d frames' % len(data_validation)) 
```
上述命令将OUTCAR（格式为VASP/OUTCAR）导入数据系统，然后将其转换为压缩格式（numpy数组）。DeePMD-kit 格式的数据存放在00.data文件夹里
```bash
$ ls training_data
set.000 type.raw type_map.raw
$ cat training_data/type.raw 
0 0 0 0 1
```
由于系统中的所有帧都具有相同的原子类型和原子序号，因此我们只需为整个系统指定一次类型信息。
```bash
$ cat training_data/type_map.raw 
H C
```
其中原子 H 被赋予类型 0，原子 C 被赋予类型 1。

## 训练
### 准备输入脚本 
一旦数据准备完成，接下来就可以进行训练。进入训练目录：
```bash
$ cd ../01.train
$ ls 
input.json
```
其中 input.json 提供了一个示例训练脚本。 这些选项在DeePMD-kit手册中有详细的解释，所以这里不做详细介绍。

在model模块, 指定嵌入和你和网络的参数。
```
    "model":{
    "type_map":    ["H", "C"],                           # the name of each type of atom
    "descriptor":{
        "type":            "se_e2_a",                    # full relative coordinates are used
        "rcut":            6.00,                         # cut-off radius
        "rcut_smth":       0.50,                         # where the smoothing starts
        "sel":             [4, 1],                       # the maximum number of type i atoms in the cut-off radius
        "neuron":          [10, 20, 40],                 # size of the embedding neural network
        "resnet_dt":       false,
        "axis_neuron":     4,                            # the size of the submatrix of G (embedding matrix)
        "seed":            1,
        "_comment":        "that's all"
    },
    "fitting_net":{
        "neuron":          [100, 100, 100],              # size of the fitting neural network
        "resnet_dt":       true,
        "seed":            1,
        "_comment":        "that's all"
    },
    "_comment":    "that's all"'
},
```
描述符se\_e2\_a用于DP模型的训练。将嵌入和拟合神经网络的大小分别设置为 [10, 20, 40] 和 [100, 100, 100]。 <img src="https://latex.codecogs.com/svg.image?\tilde{\mathcal{R}}^{i}"> 里的成分会从0.5到6Å平滑地趋于0。

下面的参数指定学习效率和损失函数：
```
    "learning_rate" :{
        "type":                "exp",
        "decay_steps":         5000,
        "start_lr":            0.001,    
        "stop_lr":             3.51e-8,
        "_comment":            "that's all"
    },
    "loss" :{
        "type":                "ener",
        "start_pref_e":        0.02,
        "limit_pref_e":        1,
        "start_pref_f":        1000,
        "limit_pref_f":        1,
        "start_pref_v":        0,
        "limit_pref_v":        0,
        "_comment":            "that's all"
    },
```
在损失函数中, pref\_e从0.02 逐渐增加到1 <img src="https://latex.codecogs.com/png.image?\dpi{110}\mathrm{eV}^{-2}">, and pref\_f从1000逐渐减小到1 <img src="https://latex.codecogs.com/png.image?\dpi{110}\AA^{2}&space;\mathrm{eV}^{-2}">，这意味着力项在开始时占主导地位，而能量项和维里项在结束时变得重要。 这种策略非常有效，并且减少了总训练时间。 pref_v 设为0 <img src="https://latex.codecogs.com/png.image?\dpi{110}\mathrm{eV}^{-2}">, 这表明训练过程中不包含任何维里数据。将起始学习率、停止学习率和衰减步长分别设置为0.001，3.51e-8，和5000。模型训练步数为<img src="https://latex.codecogs.com/png.image?\dpi{110}10^6"> 。

训练参数如下：
```
    "training" : {
        "training_data": {
            "systems":            ["../00.data/training_data"],		
            "batch_size":         "auto",                       
            "_comment":           "that's all"
        },
        "validation_data":{
            "systems":            ["../00.data/validation_data/"],
            "batch_size":         "auto",				
            "numb_btch":          1,
            "_comment":           "that's all"
        },
        "numb_steps":             100000,				            
        "seed":                   10,
        "disp_file":              "lcurve.out",
        "disp_freq":              1000,
        "save_freq":              10000,
    },
```
### 模型训练
准备好训练脚本后，我们可以用DeePMD-kit开始训练，只需运行
```bash
$ dp train input.json
```
在屏幕上，可以看到数据系统的信息
```bash
DEEPMD INFO      ----------------------------------------------------------------------------------------------------
DEEPMD INFO      ---Summary of DataSystem: training     -------------------------------------------------------------
DEEPMD INFO      found 1 system(s):
DEEPMD INFO                              system        natoms        bch_sz        n_bch          prob        pbc
DEEPMD INFO           ../00.data/training_data/             5             7           22         1.000          T
DEEPMD INFO      -----------------------------------------------------------------------------------------------------
DEEPMD INFO      ---Summary of DataSystem: validation   --------------------------------------------------------------
DEEPMD INFO      found 1 system(s):
DEEPMD INFO                               system       natoms        bch_sz        n_bch          prob        pbc
DEEPMD INFO          ../00.data/validation_data/            5             7            5         1.000          T
```
以及本次训练的开始和最终学习率
```bash
DEEPMD INFO      start training at lr 1.00e-03 (== 1.00e-03), decay_step 5000, decay_rate 0.950006, final lr will be 3.51e-08
```
如果一切正常，将在屏幕上看到每 1000 步打印一次的信息，例如
```bash
DEEPMD INFO    batch    1000 training time 7.61 s, testing time 0.01 s
DEEPMD INFO    batch    2000 training time 6.46 s, testing time 0.01 s
DEEPMD INFO    batch    3000 training time 6.50 s, testing time 0.01 s
DEEPMD INFO    batch    4000 training time 6.44 s, testing time 0.01 s
DEEPMD INFO    batch    5000 training time 6.49 s, testing time 0.01 s
DEEPMD INFO    batch    6000 training time 6.46 s, testing time 0.01 s
DEEPMD INFO    batch    7000 training time 6.24 s, testing time 0.01 s
DEEPMD INFO    batch    8000 training time 6.39 s, testing time 0.01 s
DEEPMD INFO    batch    9000 training time 6.72 s, testing time 0.01 s
DEEPMD INFO    batch   10000 training time 6.41 s, testing time 0.01 s
DEEPMD INFO    saved checkpoint model.ckpt
```
 在第 10000 步结束时，模型保存在 TensorFlow 的检查点文件 model.ckpt 中。 同时，训练和测试错误显示在文件 lcurve.out 中。
```bash
$ head -n 2 lcurve.out
#step       rmse_val       rmse_trn       rmse_e_val       rmse_e_trn       rmse_f_val       rmse_f_trn           lr
0           1.34e+01       1.47e+01         7.05e-01         7.05e-01         4.22e-01         4.65e-01     1.00e-03
```
和
```bash
$ tail -n 2 lcurve.out
999000      1.24e-01       1.12e-01         5.93e-04         8.15e-04         1.22e-01         1.10e-01      3.7e-08
1000000     1.31e-01       1.04e-01         3.52e-04         7.74e-04         1.29e-01         1.02e-01      3.5e-08
```
第 4、5 和 6、7 卷分别介绍了能量和力量训练和测试错误。 证明经过 140,000 步训练，能量测试误差小于 1 meV，力测试误差在 120 meV/Å左右。还观察到，力测试误差系统地（稍微）大于训练误差，这意味着对相当小的数据集有轻微的过度拟合。

当训练过程异常停止时，我们可以从提供的检查点重新开始训练，只需运行
```bash
$ dp train  --restart model.ckpt  input.json
```
在 lcurve.out 中，可以看到训练和测试错误，例如
```	bash
538000      3.12e-01       2.16e-01         6.84e-04         7.52e-04         1.38e-01         9.52e-02      4.1e-06
538000      3.12e-01       2.16e-01         6.84e-04         7.52e-04         1.38e-01         9.52e-02      4.1e-06
539000      3.37e-01       2.61e-01         7.08e-04         3.38e-04         1.49e-01         1.15e-01      4.1e-06
 #step      rmse_val       rmse_trn       rmse_e_val       rmse_e_trn       rmse_f_val       rmse_f_trn           lr
530000      2.89e-01       2.15e-01         6.36e-04         5.18e-04         1.25e-01         9.31e-02      4.4e-06
531000      3.46e-01       3.26e-01         4.62e-04         6.73e-04         1.49e-01         1.41e-01      4.4e-06
```
需要注意的是 input.json 需要和上一个保持一致。

### 冻结和压缩模型
在训练结束时，保存在 TensorFlow 的 checkpoint 文件中的模型参数通常需要冻结为一个以扩展名 .pb 结尾的模型文件。 只需执行
```
$ dp freeze -o graph.pb
DEEPMD INFO    Restoring parameters from ./model.ckpt-1000000
DEEPMD INFO    1264 ops in the final graph
```
它将在当前目录中输出一个名为 graph.pb 的模型文件。 
压缩 DP 模型通常会将基于 DP 的计算速度提高一个数量级，并且消耗更少的内存。 
graph.pb 可以通过以下方式压缩：
```
$ dp compress -i graph.pb -o graph-compress.pb
DEEPMD INFO    stage 1: compress the model
DEEPMD INFO    built lr
DEEPMD INFO    built network
DEEPMD INFO    built training
DEEPMD INFO    initialize model from scratch
DEEPMD INFO    finished compressing
DEEPMD INFO    
DEEPMD INFO    stage 2: freeze the model
DEEPMD INFO    Restoring parameters from model-compression/model.ckpt
DEEPMD INFO    840 ops in the final graph
```
将输出一个名为 graph-compress.pb 的模型文件。

### 模型测试
我们可以通过运行如下命令检查训练模型的质量
```bash
$ dp test -m graph-compress.pb -s ../00.data/validation_data -n 40 -d results
```
在屏幕上，可以看到验证数据的预测误差信息 
```
DEEPMD INFO    # number of test data    : 40 
DEEPMD INFO    Energy RMSE              : 3.168050e-03 eV
DEEPMD INFO    Energy RMSE/Natoms       : 6.336099e-04 eV
DEEPMD INFO    Force  RMSE              : 1.267645e-01 eV/A
DEEPMD INFO    Virial RMSE              : 2.494163e-01 eV
DEEPMD INFO    Virial RMSE/Natoms       : 4.988326e-02 eV
DEEPMD INFO    # ----------------------------------------------- 
```
它将在当前目录中输出名为 results.e.out 和 results.f.out 的文件。

## 使用LAMMPS运行MD 

现在让我们切换到 lammps 目录，检查使用 LAMMPS 运行 DeePMD 所需的输入文件。
```bash
$ cd ../02.lmp
```
首先，我们将训练目录中的输出模型软链接到当前目录
```bash
$ ln -s ../01.train/graph-compress.pb
```
这里有三个文件
```bash
$ ls
conf.lmp  graph-compress.pb  in.lammps
```
其中 conf.lmp 给出了气相甲烷 MD 模拟的初始配置，文件 in.lammps 是 lammps 输入脚本。 可以检查 in.lammps 并发现它是一个用于 MD 模拟的相当标准的 LAMMPS 输入文件，只有两个不同行：
```bash
pair_style  graph-compress.pb
pair_coeff  * *
```
其中调用pair style deepmd并提供模型文件graph-compress.pb，这意味着原子交互将由存储在文件graph-compress.pb中的DP模型计算。

可以以标准方式执行
```bash
$ lmp  -i  in.lammps
```
稍等片刻，MD模拟结束，生成log.lammps和ch4.dump文件。 它们分别存储热力学信息和分子的轨迹，我们可以通过OVITO可视化轨迹检查分子构型的演变。
```bash
$ ovito ch4.dump
```
**翻译：史国勇  校对：欧阳冠宇**
