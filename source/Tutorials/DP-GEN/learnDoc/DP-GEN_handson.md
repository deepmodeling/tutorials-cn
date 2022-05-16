# DP-GEN上机教程(v0.10.3)

## DP-GEN简介
Deep Potential GENerator (DP-GEN)是一个通过实施“同步学习”方案生成可靠的DP模型的软件包。DP-GEN的工作流程通常包括三个过程：run，init和auto-test。
- init：通过第一性原理计算产生初始训练数据集。
- run：DP-GEN的主要流程，在这个阶段可以实现训练数据集的自动扩充和DP模型的质量提高。
- auto-test：计算一组简单的性质和/或执行测试以与 DFT 和/或经验原子间势函数进行比较。

本上机教程旨在帮助您快速掌握DP-GEN的run过程。

## 输入文件
在本教程中，我们以气相的甲烷分子作为例子。我们准备的DP-GEN run过程的输入文件位于dpgen_example/run。首先下载并解压dpgen_example：
```sh
wget https://dp-public.oss-cn-beijing.aliyuncs.com/community/dpgen_example.tar.xz
tar xvf dpgen_example.tar.xz
```
进入并查看dpgen_example/run。
```sh
$ cd dpgen_example/run
$ ls
INCAR_methane  machine.json  param.json  POTCAR_C  POTCAR_H
```
- param.json是当前DP-GEN run过程的设置。
- machine.json是设置机器环境和资源要求的任务调度器。machine.json会在别处单独介绍。
- INCAR*和POTCAR*是VASP软件的INCAR和POTCAR输入文件。

## Run过程
DP-GEN的run过程包含一系列连续的迭代。每一次迭代均包含三步：探索，标记和训练。相对应地，每次迭代会产生三个子文件夹：00.train, 01.model_devi 和 02.fp。
param.json
我们提供了一个param.json的示例。
```json
{
     "type_map": ["H","C"],
     "mass_map": [1,12],
     "init_data_prefix": "../",
     "init_data_sys": ["init/CH4.POSCAR.01x01x01/02.md/sys-0004-0001/deepmd"],
     "sys_configs_prefix": "../",
     "sys_configs": [
         ["init/CH4.POSCAR.01x01x01/01.scale_pert/sys-0004-0001/scale-1.000/00000*/POSCAR"],
         ["init/CH4.POSCAR.01x01x01/01.scale_pert/sys-0004-0001/scale-1.000/00001*/POSCAR"]
     ],
     "_comment": " that's all ",
     "numb_models": 4,
     "default_training_param": {
         "model": {
             "type_map": ["H","C"],
             "descriptor": {
                 "type": "se_a",
                 "sel": [16,4],
                 "rcut_smth": 0.5,
                 "rcut": 5.0,
                 "neuron": [120,120,120],
                 "resnet_dt": true,
                 "axis_neuron": 12,
                 "seed": 1
             },
             "fitting_net": {
                 "neuron": [25,50,100],
                 "resnet_dt": false,
                 "seed": 1
             }
         },
         "learning_rate": {
             "type": "exp",
             "start_lr": 0.001,
             "decay_steps": 5000
         },
         "loss": {
             "start_pref_e": 0.02,
             "limit_pref_e": 2,
             "start_pref_f": 1000,
             "limit_pref_f": 1,
             "start_pref_v": 0.0,
             "limit_pref_v": 0.0
         },
         "training": {
             "stop_batch": 400000,
             "disp_file": "lcurve.out",
             "disp_freq": 1000,
             "numb_test": 4,
             "save_freq": 1000,
             "save_ckpt": "model.ckpt",
             "disp_training": true,
             "time_training": true,
             "profiling": false,
             "profiling_file": "timeline.json",
             "_comment": "that's all"
         }
     },
     "model_devi_dt": 0.002,
     "model_devi_skip": 0,
     "model_devi_f_trust_lo": 0.05,
     "model_devi_f_trust_hi": 0.15,
     "model_devi_e_trust_lo": 10000000000.0,
     "model_devi_e_trust_hi": 10000000000.0,
     "model_devi_clean_traj": true,
     "model_devi_jobs": [
         {"sys_idx": [0],"temps": [100],"press": [1.0],"trj_freq": 10,"nsteps": 300,"ensemble": "nvt","_idx": "00"},
         {"sys_idx": [1],"temps": [100],"press": [1.0],"trj_freq": 10,"nsteps": 3000,"ensemble": "nvt","_idx": "01"}
     ],
     "fp_style": "vasp",
     "shuffle_poscar": false,
     "fp_task_max": 20,
     "fp_task_min": 5,
     "fp_pp_path": "./",
     "fp_pp_files": ["POTCAR_H","POTCAR_C"],
     "fp_incar": "./INCAR_methane"
}
```
以下是关键词的详细说明。
基础关键词 (第 2-3 行):
| 关键词         | 类型            | 说明             |
|-------------|-----------------|-------------------------|
| "type_map"  | 字符串列表   | 原子类型              |
| "mass_map"  | 浮点数列表   | 相对原子质量           |

数据相关的关键词 (第 4-10 行):
| 关键词                   | 类型            | 说明                                                                                                 |
|-----------------------|-----------------|-------------------------------------------------------------------------------------------------------------|
| "init_data_prefix"    | 字符串          | 初始数据目录的前缀                                                                         |
| "init_data_sys"       | 字符串列表       | 初始数据的目录。您可以在此处使用绝对路径或相对路径。                             |
| "sys_configs_prefix"  | 字符串           | sys_configs的前缀                                                                                       |
| "sys_configs"         | 字符串列表        | 包含应在迭代中探索的结构的目录。此处支持通配符。  |

训练相关的关键词 (第 12-58 行):
| 关键词                       | 类型     | 说明                                  |
|---------------------------|----------|----------------------------------------------|
| "numb_models"             | 整数     | 在 00.train 阶段训练的目标模型个数。  |
| "default_training_param"  | 字典     | deepmd-kit的训练参数。          |

探索相关的关键词 (第 59-69 行):
| 关键词                      | 类型                    | 说明                                                                                                                                                                                                                                   |
|--------------------------|-------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| "model_devi_dt"          | 浮点数                   | MD的时间步长                                                                                                                                                                                                                               |
| "model_devi_skip"        | 整数                  | 从每次 MD 中跳过指定数量的结构后进行 fp 计算                                                                                                                                                                                                |
| "model_devi_f_trust_lo"  | 浮点数                   | 力筛选标准的下限。如果是列表，应该为sys_configs中的每个索引分别设置。                                                                                                                                  |
| "model_devi_f_trust_hi"  | 整数                  | 力筛选标准的上限。如果是列表，应该为sys_configs中的每个索引分别设置。                                                                                                                                  |
| "model_devi_e_trust_lo"  | 浮点数或浮点数列表      | 维里力筛选标准的下限。如果是列表，应该为sys_configs中的每个索引分别设置。应与 DeePMD-kit v2.x 一起使用。                                                                                            |
| "model_devi_e_trust_hi"  | 浮点数或浮点数列表      | 维里力筛选标准的上限。如果是列表，应该为sys_configs中的每个索引分别设置。应与 DeePMD-kit v2.x 一起使用。                                                                                            |
| "model_devi_clean_traj"  | 布尔数或整数            | 如果是布尔数，代表是否清除MD轨迹文件。如果是整数，将保留最近 n 次迭代的轨迹文件并删除其他文件。  |
| "model_devi_jobs"        | 字典列表              | 为 01.model_devi 的探索阶段设置。列表中每个字典对应一次迭代。model_devi_jobs的索引与迭代的索引一致。                                                                           |

标记相关的关键词 (第 70-76 行):
| 关键词               | 类型            | 说明                                                                                                              |
|-------------------|-----------------|--------------------------------------------------------------------------------------------------------------------------|
| "fp_style"        | 字符串       | 第一性原例计算的软件。目前可选择的软件包括 “vasp”, “pwscf”, “siesta” 和 “gaussian”。                      |
| "shuffle_poscar"  | 布尔数       | 是否对poscar进行混洗                                                                                                                      |
| "fp_task_max"     | 整数         | 02.fp 中每次迭代计算的最大结构数量。                                                       |
| "fp_task_min"     | 整数         | 02.fp 中每次迭代计算的最小结构数量。                                                           |
| "fp_pp_path"      | 字符串       | 用于 02.fp 的赝势文件的目录路径。                                                          |
| "fp_pp_files"     | 字符串列表    | 02.fp 中使用的赝势文件。注意元素顺序应与 type.map 一致。  |
| "fp_incar"        | 字符串       | VASP 的输入文件。INCAR中必须指定 KSPACING 和 KGAMMA。                                                             |

## 输出文件
在配置好machine.json之后，我们可以通过以下指令简单运行DP-GEN的run过程:
```sh
$ dpgen run param.json machine.json
```
在dpgen_example/run中，可以发现自动生成了一个文件夹和两个文件。 
```sh
$ ls 
dpgen.log  INCAR_methane  iter.000000  machine.json  param.json  record.dpgen 
```
- iter.000000包含DP-GEN run过程中第一次迭代的主要结果。
- record.dpgen记录run过程的当前阶段。
- dpgen.log包括时间和迭代信息。

第一次迭代完成后，iter.000000 的文件夹结构如下:
```sh
$ tree iter.000000/ -L 1
./iter.000000/
├── 00.train
├── 01.model_devi
└── 02.fp
```
### 00.train

首先，我们查看iter.000000/00.train目录。
```sh
$ tree iter.000000/00.train -L 1
./iter.000000/00.train/
├── 000
├── 001
├── 002
├── 003
├── data.init -> /root/dpgen_example
├── data.iters
├── graph.000.pb -> 000/frozen_model.pb
├── graph.001.pb -> 001/frozen_model.pb
├── graph.002.pb -> 002/frozen_model.pb
└── graph.003.pb -> 003/frozen_model.pb
```
- 文件夹00x包含DeePMD-kit的输入输出文件，在这个文件夹中训练了一个完整的模型。
- graph.00x.pb是00x/frozen.pb的链接，是DeePMD-kit生成的模型。这些模型之间的唯一区别是神经网络初始化的随机种子。
我们可以随机选择其中之一，如000。
```sh
$ tree iter.000000/00.train/000 -L 1
./iter.000000/00.train/000
├── checkpoint
├── frozen_model.pb
├── input.json
├── lcurve.out
├── model.ckpt-400000.data-00000-of-00001
├── model.ckpt-400000.index
├── model.ckpt-400000.meta
├── model.ckpt.data-00000-of-00001
├── model.ckpt.index
├── model.ckpt.meta
└── train.log
```
- input.json 是当前DeePMD-kit训练任务的设置。
- checkpoint用于重启训练。
- model.ckpt*是模型相关的文件。
- frozen_model.pb是冻结的模型。 
- lcurve.out记录能量和力的训练精度。
- train.log包含包括版本、数据、硬件信息、时间等。

### 01.model_devi
接下来，我们查看 iter.000000/01.model_devi目录。
```sh
$ tree iter.000000/01.model_devi -L 1
./iter.000000/01.model_devi/
├── confs
├── graph.000.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.000.pb
├── graph.001.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.001.pb
├── graph.002.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.002.pb
├── graph.003.pb -> /root/dpgen_example/run/iter.000000/00.train/graph.003.pb
├── task.000.000000
├── task.000.000001
├── task.000.000002
├── task.000.000003
├── task.000.000004
├── task.000.000005
├── task.000.000006
├── task.000.000007
├── task.000.000008
└── task.000.000009
```
- 文件夹confs包含LAMMPS MD的初始构型，该构型由param.json中“sys_configs”指定的POSCAR转换而来。

- 文件夹 task.000.00000x 包含LAMMPS的输入输出文件。我们可以随机查看其中之一，如 task.000.000001。
```sh
$ tree iter.000000/01.model_devi/task.000.000001
./iter.000000/01.model_devi/task.000.000001
├── conf.lmp -> ../confs/000.0001.lmp
├── input.lammps
├── log.lammps
├── model_devi.log
└── model_devi.out
```
- conf.lmp是confs文件夹中000.0001.lmp的链接，作为 MD 的初始构型。
- input.lammps是LAMMPS的输入文件。
- model_devi.out记录了MD中相关标签、能量和力的模型偏差。它可以作为挑选结构和进行第一性原理计算的标准。

通过 head model_devi.out，您会看到:
```sh
$ head -n 5 ./iter.000000/01.model_devi/task.000.000001/model_devi.out
 #  step max_devi_v     min_devi_v     avg_devi_v     max_devi_f     min_devi_f     avg_devi_f 
 0     1.438427e-04   5.689551e-05   1.083383e-04   8.835352e-04   5.806717e-04   7.098761e-04
10     3.887636e-03   9.377374e-04   2.577191e-03   2.880724e-02   1.329747e-02   1.895448e-02
20     7.723417e-04   2.276932e-04   4.340100e-04   3.151907e-03   2.430687e-03   2.727186e-03
30     4.962806e-03   4.943687e-04   2.925484e-03   5.866077e-02   1.719157e-02   3.011857e-02
```
现在我们关注max_devi_f。回想一下，我们已经设置了"trj_freq"为 10，所以每10步结构将会被保存一次。是否挑选出该结构取决于它的 "max_devi_f"。如果它位于"model_devi_f_trust_lo"(0.05) 和"model_devi_f_trust_hi"(0.15)之间，DP-GEN 会将该结构视为候选结构。这里只挑选20个结构，其 "max_devi_f"为 5.866077 e-02。

### 02.fp
最后，让我们查看iter.000000/ 02.fp目录。
```sh
$ tree iter.000000/02.fp -L 1
./iter.000000/02.fp
├── data.000
├── task.000.000000
├── task.000.000001
├── task.000.000002
├── task.000.000003
├── task.000.000004
├── task.000.000005
├── task.000.000006
├── task.000.000007
├── task.000.000008
├── task.000.000009
├── task.000.000010
├── task.000.000011
├── candidate.shuffled.000.out
├── POTCAR.000
├── rest_accurate.shuffled.000.out
└── rest_failed.shuffled.000.out
```
- POTCAR.000 是根据param.json中"fp_pp_files"生成的VASP POTCAR文件。
- candidate.shuffle.000.out记录了从上一步01.model_devi中挑选出的候选结构。候选者的数量往往远多于预期单次计算的最大值。在这种情况下DP-GEN将随机选择至多"fp_task_max"个结构并生成 task.*文件夹。
- rest_accurate.shuffle.000.out记录准确预测的结构("max_devi_f" 小于"model_devi_f_trust_lo"，这些结构无需进行进一步计算)。
- rest_failed.shuffled.000.out 记录了预测失败的结构 ("max_devi_f" 大于"model_devi_f_trust_hi"，这些结构可能是非物理的)。
- data.000: 在第一性原理计算之后，DP-GEN将收集这些数据并转化为DeePMD-kitPMD-kit支持的格式。在下一次迭代的 00.train 中，这些数据将与初始数据一起被训练。

通过 cat candidate.shuffled.000.out | grep task.000.000001，可以看到:
```sh
$ cat ./iter.000000/02.fp/candidate.shuffled.000.out | grep task.000.000001
iter.000000/01.model_devi/task.000.000001 190
iter.000000/01.model_devi/task.000.000001 130
iter.000000/01.model_devi/task.000.000001 120
iter.000000/01.model_devi/task.000.000001 150
iter.000000/01.model_devi/task.000.000001 280
iter.000000/01.model_devi/task.000.000001 110
iter.000000/01.model_devi/task.000.000001 30
iter.000000/01.model_devi/task.000.000001 230
```
task.000.000001 30 是我们在01.model_devi中发现的满足再次计算条件的结构。

第一次迭代之后，我们可以查看 dpgen.log和record.log 的内容。
 ```sh
$ cat dpgen.log
2022-03-07 22:12:45,447 - INFO : start running
2022-03-07 22:12:45,447 - INFO : =============================iter.000000==============================
2022-03-07 22:12:45,447 - INFO : -------------------------iter.000000 task 00--------------------------
2022-03-07 22:12:45,451 - INFO : -------------------------iter.000000 task 01--------------------------
2022-03-08 00:53:00,179 - INFO : -------------------------iter.000000 task 02--------------------------
2022-03-08 00:53:00,179 - INFO : -------------------------iter.000000 task 03--------------------------
2022-03-08 00:53:00,187 - INFO : -------------------------iter.000000 task 04--------------------------
2022-03-08 00:57:04,113 - INFO : -------------------------iter.000000 task 05--------------------------
2022-03-08 00:57:04,113 - INFO : -------------------------iter.000000 task 06--------------------------
2022-03-08 00:57:04,123 - INFO : system 000 candidate :     12 in    310   3.87 %
2022-03-08 00:57:04,125 - INFO : system 000 failed    :      0 in    310   0.00 %
2022-03-08 00:57:04,125 - INFO : system 000 accurate  :    298 in    310  96.13 %
2022-03-08 00:57:04,126 - INFO : system 000 accurate_ratio:   0.9613    thresholds: 1.0000 and 1.0000   eff. task min and max   -1   20   number of fp tasks:     12
2022-03-08 00:57:04,154 - INFO : -------------------------iter.000000 task 07--------------------------
2022-03-08 01:02:07,925 - INFO : -------------------------iter.000000 task 08--------------------------
2022-03-08 01:02:07,926 - INFO : failed tasks:      0 in     12    0.00 % 
2022-03-08 01:02:07,949 - INFO : failed frame:      0 in     12    0.00 % 
```
可以看出在 iter.000000 中有 310 个结构生成，其中12个结构被收集并用于第一性原理计算。

```sh
$ cat record.dpgen
0 0
0 1
0 2
0 3
0 4
0 5
0 6
0 7
0 8
```
每行包含两个数字：第一个是迭代的索引，第二个的范围从0到9，记录了当前正在运行的是每次迭代的哪个阶段。

| 迭代索引  | "单次迭代的阶段 "   | 过程          |
|----------------------|-----------------------------|------------------|
| 0                    | 0                           | make_train       |
| 0                    | 1                           | run_train        |
| 0                    | 2                           | post_train       |
| 0                    | 3                           | make_model_devi  |
| 0                    | 4                           | run_model_devi   |
| 0                    | 5                           | post_model_devi  |
| 0                    | 6                           | make_fp          |
| 0                    | 7                           | run_fp           |
| 0                    | 8                           | post_fp          |

如果DP-GEN的run过程由于某种原因停止，DP-GEN可以根据record.dpgen恢复主进程。用户也可以根据自己的目的手动更改它，例如删除最后一次迭代并从某个checkpoint续算。

### 结果观察

在所有的迭代完成后，我们可以查看 dpgen_example/run 的架构 
```sh
$ tree ./ -L 2
./
├── dpgen.log
├── INCAR_methane
├── iter.000000
│   ├── 00.train
│   ├── 01.model_devi
│   └── 02.fp
├── iter.000001
│   ├── 00.train
│   ├── 01.model_devi
│   └── 02.fp
├── iter.000002
│   └── 00.train
├── machine.json
├── param.json
└── record.dpgen
```
和 dpgen.log 的内容。
```sh
$ cat cat dpgen.log | grep system
2022-03-08 00:57:04,123 - INFO : system 000 candidate :     12 in    310   3.87 %
2022-03-08 00:57:04,125 - INFO : system 000 failed    :      0 in    310   0.00 %
2022-03-08 00:57:04,125 - INFO : system 000 accurate  :    298 in    310  96.13 %
2022-03-08 00:57:04,126 - INFO : system 000 accurate_ratio:   0.9613    thresholds: 1.0000 and 1.0000   eff. task min and max   -1   20   number of fp tasks:     12
2022-03-08 03:47:00,718 - INFO : system 001 candidate :      0 in   3010   0.00 %
2022-03-08 03:47:00,718 - INFO : system 001 failed    :      0 in   3010   0.00 %
2022-03-08 03:47:00,719 - INFO : system 001 accurate  :   3010 in   3010 100.00 %
2022-03-08 03:47:00,722 - INFO : system 001 accurate_ratio:   1.0000    thresholds: 1.0000 and 1.0000   eff. task min and max   -1    0   number of fp tasks:      0
```
可以发现在 iter.000001 生成了3010个结构，但没有新的结构被收集并用于第一性原理计算。因此，iter.000002/00.train 的最终模型不再更新。

**翻译：方满娣 校对：梁文硕**
