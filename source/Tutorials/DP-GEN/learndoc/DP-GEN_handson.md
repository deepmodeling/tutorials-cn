# DP-GEN (v0.10.6) 使用手册

## DP-GEN 工作流

深度势能生成器 (Deep Potential GENerator, DP-GEN) 是一个**实现并发学习方案以生成可靠的深度势能 (DP) 模型**的软件包。通常，DP-GEN 工作流包含三个过程：初始化、运行和自动测试。

1. <span style="color:purple; font-weight:bold">初始化 *(init)* </span>：通过第一性原理计算生成初始训练数据集
2. <span style="color:purple; font-weight:bold">运行 *(run)* </span>：DP-GEN 的主要过程，其中不断丰富训练集并自动提升 DP 模型的质量。
3. <span style="color:purple; font-weight:bold">自动测试 *(autotest)* </span>：计算一组简单的性质 和/或 执行测试以与DFT和/或经验原子间势进行比较。

这篇教程旨在帮助您快速掌握<span style="font-weight:bold">运行</span>过程，因此对于<span style="font-weight:bold">初始化</span>与<span style="font-weight:bold">自动测试</span>过程仅提供了简单的介绍。

## 案例：气相甲烷分子

这里使用气相甲烷分子的案例，介绍 DP-GEN 的基本使用方法。

### 初始化 *init*

初始数据集用于训练多个（默认为 4 个）初始 DP 模型，可以通过 <span style="color:purple; font-weight:bold">1.自定义方式</span> 或 <span style="color:purple; font-weight:bold">2.DP-GEN 提供的标准方式</span> 生成。

#### **自定义方法**

直接执行从头算分子动力学 (AIMD) 模拟是生成初始数据的常见自定义方法。对于通过 AIMD 模拟生成初始数据的用户，我们给出以下建议：

- 在较高温度下执行 AIMD 模拟。
- 从多个（尽可能多的）不相关的初始配置开始 AIMD 模拟。
- 以一定的时间间隔保存 AIMD 轨迹中的快照，以避免对高度相关的配置进行采样。

#### **DP-GEN的标准方式**

对于块体材料，可以使用 DP-GEN 的`init_bulk`方法生成初始数据。在`init_bulk`方法中，给定的配置最初通过从头算 *ab initio* 弛豫，<span style="background-color:pink">随后缩放或扰动。接下来，使用这些缩放或扰动配置开始小规模 AIMD 仿真</span>，并将 AIMD 格式数据最终转换为 DeePMD-kit 所需的数据格式。基本上，`init_bulk` 可以分为四个部分：

1. 在文件 `00.place_ele`中弛豫
2. 扰动和缩放文件夹`01.scale_pert`
3. 在文件夹`02.md`中运行一个简短的 AIMD
4. 收集文件夹`02.md`中的数据。

对于表面系统，可以使用DP-GEN的`init_surf`方法生成初始数据。`init_surf`一般分为两部分：

1. 在文件夹`00.place_ele`中构建特定表面
2. 扰动和缩放文件夹`01.scale_pert`

以上步骤在以 DP-GEN 的标准方式生成初始数据时自动执行。用户只需要准备进行**第一性原理计算**和 **DP-GEN** 的输入文件（`param.json` 和 `machine.json`）。

以标准方式生成块体材料的初始数据时，执行以下命令：

```sh
dpgen init_bulk param.json machine.json
```

对于表面系统，执行：

```sh
dpgen init_surf param.json machine.json
```

以标准方式准备初始数据的更详细描述请参阅 [DP-GEN 文档](https://docs.deepmodeling.com/projects/dpgen/en/latest/) 的初始化部分。

#### **本教程中的初始数据**

在本教程中，我们以气相甲烷分子为例。我们已经在`dpgen_example/init`中准备了初始数据。现在下载`dpgen_example`并解压它：

```sh
wget https://dp-public.oss-cn-beijing.aliyuncs.com/community/dpgen_example.tar.xz
tar xvf dpgen_example.tar.xz
```

然后，检查`dpgen_example`文件夹

```sh
$ cd dpgen_example  
$ ls  
init run
```

- `init` 文件夹包含通过 AIMD 模拟得到的初始数据
- `run` 文件夹包含运行过程所需的输入文件

首先，使用 `tree` 命令查看 `init` 文件夹。

```sh
$ tree init -L 2  
```

你应该可以在屏幕上看到输出：

```sh
init  
├── CH4.POSCAR  
├── CH4.POSCAR.01x01x01  
│ ├── 00.place_ele  
│ ├── 01.scale_pert
│ ├── 02.md  
│ └── param.json  
├── INCAR_methane.md  
├── INCAR_methane.rlx  
└── param.json
```

- `CH4.POSCAR.01x01x01`包含由 DP-GEN `init_bulk` 流程生成的文件。
- `INCAR_*` 和 `CH4.POSCAR` 是 VASP 的标准 INCAR 和 POSCAR 文件
- `param.json` 用于指定 DP-GEN `init_bulk` 流程的详细信息。

请注意，`POTCAR` 和 `machine.json` 对于初始化和运行过程是相同的，可以在`run`文件夹中找到。

### 运行 *Run*

运行过程包含一系列连续迭代，按顺序进行，例如将系统加热到特定温度。每次迭代由三个步骤组成：<span style="color:purple; font-weight:bold">探索 *Exploration*</span>、<span style="color:purple; font-weight:bold">标记 *Labeling*</span>和<span style="color:purple; font-weight:bold">训练 *Training*</span>。

#### 输入文件

首先，介绍 DP-GEN 运行过程所需的输入文件。

我们已经在`dpgen_example/run` 中准备了输入文件，现在进入`dpgen_example/run`使用 `tree` 命令查看文件

```sh
$ cd dpgen_example/run
$ tree -L 1
.
├── INCAR_methane
├── machine.json
├── param.json
├── POTCAR_C
└── POTCAR_H
```

- `param.json` 是运行当前任务的 DP-GEN 设置。
- `machine.json` 是一个任务调度程序，其中设置了计算机环境和资源要求。
- `INCAR*` 和 `POTCAR*` 是 VASP 软件包的输入文件。 所有第一性原理计算都与您在 param.json 中设置的参数相同。

我们可以通过在 `param.json` 和 `machine.json` 中指定关键字来按预期执行运行过程。下面给出了这些关键字的描述。

##### param.json

`param.json` 中的关键字可以分为 4 个部分：

- <span style="color:purple; font-weight:bold">系统和数据 *System and Data*</span>：用于指定原子类型、初始数据等。
- <span style="color:purple; font-weight:bold">训练 *Training*</span>：主要用于指定训练步骤中的任务;
- <span style="color:purple; font-weight:bold">探索 *Exploration*</span>：主要用于在探索步骤中指定任务;
- <span style="color:purple; font-weight:bold">标记 *Labeling*</span>：主要用于指定标记步骤中的任务。

这里我们以气相甲烷分子为例，介绍`param.json`中的主要关键词。

###### 系统和数据

系统和数据相关的关键字如下：

```json
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
```  

关键词描述：

| 键词         | 字段类型            | 描述             |
|-------------|-----------------|-------------------------|
| "type_map"  | list    | 元素              |
| "mass_map"  | list    | 标准原子质量  |
| "init_data_prefix"    | str          | initial data 的前置路径                                                                          |
| "init_data_sys"       | list         | 初始数据的目录。您可以在此处使用绝对路径或相对路径。                             |
| "sys_configs_prefix"  | str          | sys_configs 的前置路径                                                                                      |
| "sys_configs"         | list         | 需要在迭代中探索的结构的路径，此处支持通配符  |

案例说明：

系统相关的关键词指定有关系统的基本信息。“type_map”给出了原子类型，即“H”和“C”。“mass_map”给出了标准原子重量，即“1”和“12”。
数据相关的关键词指定训练用于 model_devi 计算的初始 DP 模型和结构的初始化数据。
“init_data_prefix”和“init_data_sys”指定初始化数据的位置。“sys_configs_prefix”和“sys_configs”指定结构的位置。
在这里，初始化数据在“...... /init/CH4.POSCAR.01x01x01/02.md/sys-0004-0001/deepmd”中提供。这些结构分为两组，在“....../init/CH4.POSCAR.01x01x01/01.scale_pert/sys-0004-0001/scale-1.000/00000*/POSCAR“和”....../init/CH4.POSCAR.01x01x01/01.scale_pert/sys-0004-0001/scale-1.000/00001*/POSCAR”。

###### 训练

与训练相关的关键字如下：

```json
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
```

关键词描述：

| 键词                       | 字段类型     | 描述                                  |
|---------------------------|----------|----------------------------------------------|
| "numb_models"             | int  | 在 00.train 中训练的模型数量。  |
| "default_training_param"  | dict     | DeepMD-kit 的训练参数          |

案例说明：

训练相关键指定训练任务的详细信息。`numb_models`指定要训练的模型数量。`default_training_param`指定了 DeepMD-kit 的训练参数。在这里，将训练 4 个 DP 模型。

DP-GEN 的训练部分由 DeepMD-kit 执行，因此此处的关键字与 DeepMD-kit 的关键字相同，此处不再赘述。有关这些关键字的详细说明，请访问[DeepMD-kit 文档](https://docs.deepmodeling.com/projects/deepmd/en/master/)。

###### 探索

与探索相关的关键字如下：

```json
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
```

关键词描述：

| 键词                      | 字段类型                    | 描述   |
|--------------------------|-------------------------|---------------|
| "model_devi_dt"          | float  | MD 的时间步长                                                                                                                                                                                                                                |
| "model_devi_skip"        | int    | 每个 MD 中为 fp 跳过的结构数                                                                                                                                                                                                |
| "model_devi_f_trust_lo"  | float  | 选择的力下限。如果为 List，则应分别为每个索引设置sys_configs。                                                                                                                                 |
| "model_devi_f_trust_hi"  | int    | 选择的力上限。如果为 List，则应分别为每个索引设置sys_configs。                                                                                                                                  |
| "model_devi_v_trust_hi"  | float or list  | 选择的维里的下限。如果为 List，则应分别为每个索引设置sys_configs。应与 DeePMD-kit v2.x 一起使用。                                                                                             |
| "model_devi_v_trust_hi"  | float or list  | 选择的维里的上限。如果为 List，则应分别为每个索引设置sys_configs。应与 DeePMD-kit v2.x 一起使用。                                                                                             |
| "model_devi_clean_traj"  | bool or int    | 如果model_devi_clean_traj的类型是布尔类型，则表示是否清理MD中的traj文件夹，因为它们太大。如果是 Int 类型，则将保留 traj 文件夹的最新 n 次迭代，其他迭代将被删除。  |
| "model_devi_jobs"        | list            | 01.model_devi 中的探索设置。列表中的每个字典对应于一次迭代。model_devi_jobs 的索引与迭代的索引完全一致                                               |
| &nbsp;&nbsp;&nbsp;&nbsp;"sys_idx"   | List of integer         | 选择系统作为MD的初始结构并进行探索。序列与“sys_configs”完全对应。 |
| &nbsp;&nbsp;&nbsp;&nbsp;"temps" | list  | 分子动力学模拟的温度 (K)
| &nbsp;&nbsp;&nbsp;&nbsp;"press" | list  | 分子动力学模拟的压力 (Bar) 
| &nbsp;&nbsp;&nbsp;&nbsp;"trj_freq"   | int          | MD中轨迹的保存频率。                  |
| &nbsp;&nbsp;&nbsp;&nbsp;"nsteps"     | int          | 分子动力学运行步数                                 |
| &nbsp;&nbsp;&nbsp;&nbsp;"ensembles"  | str          | 决定在 MD 中选择的集成算法，选项包括 “npt” 和 “nvt”. |

案例说明

与探索相关的关键词指定探索任务中的细节。在 100 K 的温度和 1.0 bar 的压力下进行 MD 仿真，在 NVT 集合下积分时间为 2 fs。在“model_devi_jobs”中设置了两次迭代。MD 仿真在第一组和第二组的‘sys_configs'的 00 和 01 迭代中分别运行 300 和 3000 个时间步长。我们选择保存 MD 模拟中生成的所有结构，并将“trj_freq”设置为 10，因此在 00 和 01 次迭代中保存 30 和 300 个结构。如果保存结构的“max_devi_f”介于 0.05 和 0.15 之间，DP-GEN 会将该结构视为候选结构。我们选择清理MD中的 traj 文件夹，因为它们太大了。如果要保存 traj 文件夹的最近n次迭代，可以将“model_devi_clean_traj”设置为整数。

###### 标注

与标注相关的关键字如下：

```json
"fp_style": "vasp",
"shuffle_poscar": false,
"fp_task_max": 20,
"fp_task_min": 5,
"fp_pp_path": "./",
"fp_pp_files": ["POTCAR_H","POTCAR_C"],
"fp_incar": "./INCAR_methane"
```

关键词描述：

| 键词               | 字段类型            | 描述                                                                                                              |
|-------------------|-----------------|--------------------------------------------------------------------------------------------------------------------------|
| "fp_style"        | String          | 第一性原理软件软件。到目前为止，选项包括“vasp”、“pwscf”、“siesta”和“gaussian”。                       |
| "shuffle_poscar"  | Boolean         |                                                                                                                          |
| "fp_task_max"     | Integer         | 每次迭代 在 02.fp 中要计算的最大结构。                                                       |
| "fp_task_min"     | Integer         | 每次迭代 在 02.fp 中要计算的最小结构。                                                           |
| "fp_pp_path"      | String          | 用于 02.fp 的赝势文件路径。                                                          |
| "fp_pp_files"     | List of string  | 用于 02.fp 的赝势文件。请注意，元素的顺序应与 type_map 中的顺序相对应。  |
| "fp_incar"        | String          | VASP 输入文件。INCAR必须指定 KSSPACE 和 KGAMMA。                             |

案例说明：

标记相关键词指定标记任务的详细信息。 在这里，最少 1 个和最多 20 个结构将使用 VASP 代码进行标记，在每次迭代中，INCAR 以“....../INCAR_methane”提供，POTCAR 以“....../methane/POTCAR”提供。请注意，POSCAR 和 POTCAR 中元素的顺序应与 `type_map` 中的顺序相对应。

##### machine.json

DP-GEN运行过程中的每次迭代都由三个步骤组成：探索、标注和训练。因此，machine.json 由三个步骤组成：**train**、**model_devi** 和 **fp**。每个步骤都是字典列表。每个字典都可以被视为一个独立的计算环境。

在本节中，我们将向您展示如何在本地工作站上执行`train`步骤，在本地 Slurm 集群上执行`model_devi`步骤，以及如何使用新的 DPDispatcher 在远程 PBS 集群上执行 `fp` 步骤（“api_version” >= 1.0）。 对于每个步骤，需要三种类型的关键词：

- <span style="color:purple; font-weight:bold">命令 *Command*</span>：提供用于执行每个步骤的命令。
- <span style="color:purple; font-weight:bold">机器 *Machine*</span>：指定机器环境（本地工作站、本地或远程集群或云服务器）。
- <span style="color:purple; font-weight:bold">资源 *Resources*</span>：指定组、节点、CPU 和 GPU 的数量;启用虚拟环境。

###### **在本地工作站执行`train`步骤**

在此示例中，我们在本地工作站上执行`train`步骤。

```json
"train": [
    {
      "command": "dp",
      "machine": {
        "batch_type": "Shell",
        "context_type": "local",
        "local_root": "./",
        "remote_root": "/home/user1234/work_path"
      },
      "resources": {
        "number_node": 1,
        "cpu_per_node": 4,
        "gpu_per_node": 1,
        "group_size": 1,
        "source_list": ["/home/user1234/deepmd.env"]
      }
    }
  ],
```

关键词描述：

| 键词| 字段类型| 描述|
|------|------|------|
| "command"| String| 要执行此任务的命令。|
| "machine"| dict| `machine`定义。|
| &nbsp;&nbsp;&nbsp;&nbsp;"batch_type"  | str| 批处理作业系统类型。|
| &nbsp;&nbsp;&nbsp;&nbsp;"context_type"| str| 用于远程计算机的连接。|
| &nbsp;&nbsp;&nbsp;&nbsp;"local_root"  | str| 任务和相关文件所在的目录。|
| &nbsp;&nbsp;&nbsp;&nbsp;"remote_root" | str| 在远程计算机上执行任务的目录。|
| "resource"| dict| `resources`定义|
| &nbsp;&nbsp;&nbsp;&nbsp;"number_node" | int| 每个作业所需的节点数。|
| &nbsp;&nbsp;&nbsp;&nbsp;"cpu_per_node"| int| 分配给每个作业的每个节点的 CPU 编号。|
| &nbsp;&nbsp;&nbsp;&nbsp;"gpu_per_node"| int| 分配给每个作业的每个节点的 GPU 编号。|
| &nbsp;&nbsp;&nbsp;&nbsp;"group_size"  |int | 作业中的任务数。|
| &nbsp;&nbsp;&nbsp;&nbsp;"source_list" | str| 在远程计算机上执行任务的目录。|

案例说明：

DeeOMD-kit 代码用于 `train` ，调用的“命令”是“dp”。

在机器参数中，“batch_type”指定作业调度系统的类型。如果没有作业调度系统，我们可以使用“Shell”来执行任务。“context_type”是指数据传输的方法，“local”是指通过本地文件存储系统复制和移动数据（例如CP，MV等）。在 DP-GEN 中，所有任务的路径都由软件自动定位和设置，因此“local_root”始终设置为“./".每个任务的输入文件将被发送到“remote_root”，任务将在那里执行，因此我们需要确保路径存在。

在 resources 参数中，“number_node”、“cpu_per_node”和“gpu_per_node”分别指定任务所需的节点数、CPU 数和 GPU 数。在这里要强调的是“group_size”，其指定了将打包到一个组中的任务数量。在训练任务中，我们需要训练 4 个模型。如果我们只有一个 GPU，我们可以将“group_size”设置为4。如果“group_size”设置为 1，则 4 个模型将同时在一个 GPU 上训练，因为没有作业调度系统。最后，可以通过“source_list”激活环境变量。在此示例中，“source /home/user1234/deepmd.env”在“dp”之前执行，以加载执行训练任务所需的环境变量。

<!--对于 remote_root 参数的初次使用说明可再更详细一些。-->

###### **在本地 Slurm 集群上执行 `model_devi` 步骤**

在此示例中，我们在本地 Slurm 工作站执行`model_devi`步骤。

```json
"model_devi": [
    {
      "command": "lmp",
      "machine": {
       "context_type": "local",
        "batch_type": "Slurm",
        "local_root": "./",
        "remote_root": "/home/user1234/work_path"
      },
      "resources": {
        "number_node": 1,
        "cpu_per_node": 4,
        "gpu_per_node": 1,
        "queue_name": "QueueGPU",
        "custom_flags" : ["#SBATCH --mem=32G"],
        "group_size": 10,
        "source_list": ["/home/user1234/lammps.env"]
      }
    }
],
```

关键词描述：

| 键词  | 字段类型 | 描述|
|------|------|------------|
| "queue_name"  | String| 批处理作业调度系统的队列名称。|
| "custom_flags"| String| 传递到作业提交脚本头的额外的行。|

案例说明：

LAMMPS 代码用于 `model_devi`，调用的“命令”是“lmp”。

在 machine 参数中，我们通过将“batch_type”更改为“Slurm”来指定作业调度系统的类型。

在 resources 参数中，我们通过添加 “queue_name” 来指定任务提交到的队列的名称。我们可以通过“custom_flags”向计算脚本添加其他行。在`model_devi`步骤中，经常有许多简短的任务，因此我们通常会将多个任务（例如 10 个）打包到一个组中以供提交。其他参数与本地工作站的参数类似。

###### **在远程 PBS 群集中执行 `fp` 步骤**

在此示例中，我们在可通过 SSH 访问的远程 PBS 集群上执行 `fp` 步骤。

```json
"fp": [
    {
      "command": "mpirun -n 32 vasp_std",
      "machine": {
       "context_type": "SSHContext",
        "batch_type": "PBS",
        "local_root": "./",
        "remote_root": "/home/user1234/work_path",
        "remote_profile": {
          "hostname": "39.xxx.xx.xx",
          "username": "user1234"
         }
      },
      "resources": {
        "number_node": 1,
        "cpu_per_node": 32,
        "gpu_per_node": 0,
        "queue_name": "QueueCPU",
        "group_size": 5,
        "source_list": ["/home/user1234/vasp.env"]
      }
    }
],
```

关键词描述：

| 键词| 字段类型| 描述|
|----|-----|------------|
| "remote_profile"| dict| 用于维持与远程计算机连接的信息。|
| "hostname"| str | SSH 连接的主机名或 IP。|
| "username"| str | 目标 Linux 系统的用户名。|

案例说明：

VASP 代码用于 `fp` 任务，mpi 用于并行计算，因此添加了“mpirun -n 32”来指定并行线程数。

在 `machine` 参数中，“context_type”被修改为“SSHContext”，“batch_type”被修改为“PBS”。值得注意的是，“remote_root”应设置为远程 PBS 群集上的可访问路径。添加“remote_profile”以指定用于连接远程集群的信息，包括主机名、用户名、密码、端口等。

在`source`参数中，我们将“gpu_per_node”设置为 0，因为使用 CPU 进行 VASP 计算具有成本效益。

#### 开始运行过程

准备好了param.json和machine.json，我们就可以通过以下方式轻松运行DP-GEN：

```sh
dpgen run param.json machine.json
```

#### 结果分析

##### **00.train**

首先，让我们来查看这个文件夹 `iter.000000`/ `00.train`。

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

- 文件夹 00x 包含 DeePMD 套件的输入和输出文件，其中训练了模型。
- graph.00x.pb ，链接到 00x/frozen.pb，是 DeePMD-kit 生成的模型。这些模型之间的唯一区别是神经网络初始化的随机种子。

让我们随机选择其中之一查看，例如 000。

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

- `input.json` 是当前任务的 DeePMD-kit 的设置。
- `checkpoint`用于重新开始训练。
- `model.ckpt*` 是与模型相关的文件。
- `frozen_model.pb`是冻结模型。
- `lcurve.out`记录能量和力的训练精度。
- `train.log`包括版本、数据、硬件信息、时间等。

##### **01.model_devi**

然后，我们检查这个文件，`iter.000000/ 01.model_devi`。

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

- 文件夹 confs 包含从您在 param.json 的“sys_configs”中设置的 POSCAR 转换而来的 LAMMPS MD 的初始配置。
- 文件夹 task.000.00000x 包含 LAMMPS 的输入和输出文件。我们可以随机选择其中之一，例如 task.000.000001。

```sh
$ tree iter.000000/01.model_devi/task.000.000001
./iter.000000/01.model_devi/task.000.000001
├── conf.lmp -> ../confs/000.0001.lmp
├── input.lammps
├── log.lammps
├── model_devi.log
└── model_devi.out
```

- `conf.lmp`，链接到文件夹 confs 中的“000.0001.lmp”，作为 MD 的初始配置。
- `input.lammps` 是 LAMMPS 的输入文件。
- `model_devi.out` 记录在 MD 训练中相关的标签，能量和力的模型偏差。它是选择结构和进行第一性原理计算的标准。

通过查看 `model_devi.out` 的前几行, 您会看到:

```sh
$ head -n 5 ./iter.000000/01.model_devi/task.000.000001/model_devi.out
 #  step max_devi_v     min_devi_v     avg_devi_v     max_devi_f     min_devi_f     avg_devi_f 
 0     1.438427e-04   5.689551e-05   1.083383e-04   8.835352e-04   5.806717e-04   7.098761e-04
10     3.887636e-03   9.377374e-04   2.577191e-03   2.880724e-02   1.329747e-02   1.895448e-02
20     7.723417e-04   2.276932e-04   4.340100e-04   3.151907e-03   2.430687e-03   2.727186e-03
30     4.962806e-03   4.943687e-04   2.925484e-03   5.866077e-02   1.719157e-02   3.011857e-02
```

让我们专注于`max_devi_f`。

回想一下，我们将“trj_freq”设置为10，因此每10个步骤保存结构。是否选择结构取决于其`“max_devi_f”`。如果它介于`“model_devi_f_trust_lo”`（0.05）和`“model_devi_f_trust_hi”`（0.15）之间，DP-GEN 会将该结构视为候选结构。在这里，仅选择第 30 个结构，其`“max_devi_f”`是 5.866077e e-02。

##### **02.fp**

最后，我们来查看 `iter.000000/ 02.fp` 文件夹。

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

- `POTCAR` 是根据 param.json 的`fp_pp_files`生成的 VASP 输入文件。
- `candidate.shuffle.000.out` 记录将从最后一步 01.model_devi 中选择哪些结构。 候选的数量总是远远超过您一次期望计算的最大值。在这种情况下，DP-GEN将随机选择最多`fp_task_max`结构并形成文件夹任务。
- `rest_accurate.shuffle.000.out` 记录了我们的模型准确的其他结构（`max_devi_f`小于`model_devi_f_trust_lo`，无需再计算），
- `rest_failed.shuffled.000.out` 记录了我们的模型太不准确的其他结构（大于 `model_devi_f_trust_hi`，可能存在一些错误）。
- `data.000`：经过第一性原理计算后，DP-GEN 将收集这些数据并将其更改为 DeePMD-kit 所需的格式。在下一次迭代的“00.train”中，这些数据将与初始数据一起训练。

通过 `cat candidate.shuffled.000.out | grep task.000.000001`， 你将会看到：

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

`task.000.000001` 30 正是我们刚刚在 `01.model_devi` 中找到的满足要再次计算的标准。
在第一次迭代之后，我们检查 dpgen.log 和 record.dpgen 的内容。

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

可以发现，在 iter.000000 中生成了 310 个结构，其中收集了 12 个结构进行第一性原理计算。

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

每行包含两个数字：第一个是迭代的索引，第二个是 0 到 9 之间的数字，记录每个迭代中的哪个阶段当前正在运行。

| Index of iterations  | Stage in each iteration     | Process          |
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

如果DP-GEN的进程由于某种原因停止，DP-GEN将通过record.dpgen自动恢复主进程。您也可以根据自己的目的手动更改它，例如删除上次迭代并从一个检查点恢复。
在所有迭代之后，我们再来看看 `dpgen_example/run` 的文件结构：

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

以及 `dpgen.log` 的内容：

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

可以发现，在 'iter.000001' 中生成了 3010 个结构，其中没有收集结构进行第一性原理计算。因此，最终模型不会在 iter.000002/00.train 中更新。

## 简化

当你的数据集中包含大量重复数据时，这个步骤可以帮助你简化你的数据集。由于 `dpgen simplify`是在大型数据集上作用的，因此本部分将仅提供一个简单的演示。

要了解有关简化的更多信息，您可以参考
[DPGEN 文档](https://docs.deepmodeling.com/projects/dpgen/en/latest/)
[dpgen simplify parameters 文档](https://docs.deepmodeling.com/projects/dpgen/en/latest/simplify/simplify-jdata.html)
[dpgen simplify machine parameters 文档](https://docs.deepmodeling.com/projects/dpgen/en/latest/simplify/simplify-mdata.html)

这个演示可以从 dpgen/examples/simplify-MAPbI3-scan-lebesgue 下载。您可以在 [dpgen.examples](https://github.com/deepmodeling/dpgen/tree/master/examples) 中找到更多示例。

在示例中，“data”包含基于 MAPbI3 扫描案例的简单数据集。其中的内容已大幅减少，请勿过于当真，这只是一个演示。

`simplify_example`是工作路径，其中包含`INCAR`以及`simplify.json`和`machine.json`的模板。您可以使用命令
```sh
nohup dpgen simplify simplify.json machine.json 1>log 2>err &
```
测试 `dpgen simplify` 是否可以正常运行。

温馨提示：

1. `dpdispatcher 0.4.15`支持`machine.json`，请检查 https://docs.deepmodeling.com/projects/dpdispatcher/en/latest/ 以根据您的`dpdispatcher`版本更新参数。
2. `POTCAR`应由用户准备。
3. 请检查路径和文件名，并确保它们正确无误。

简化可用于迁移学习，请参阅 [案例研究：迁移学习]（../../../CaseStudies/Transfer-learning/index.html）

## 自动测试

`auto-test` 功能仅用于合金材料验证其DP模型的准确性，用户可以计算一组简单的属性，并将结果与 DFT 或传统经验力场的结果进行比较。DPGEN 的自动测试模块支持各种属性的计算，例如:

- 00.equi：（默认任务）平衡状态;

- 01.eos：状态方程;

- 02.elastic：杨氏弹性模量;

- 03.vacancy：空位形成能量;

- 04.interstitial：间隙形成能量;

- 05.surf：表面形成能量；

在本部分中，Al-Mg-Cu DP 势用于说明如何自动测试合金材料的 DP 势。每个`auto-test`任务包括三个阶段：

- `make`自动准备所有必需的计算文件和输入脚本;
- `run`可以帮助将计算任务提交到远程计算平台，当计算任务完成时，将自动收集结果;
- `post`自动将计算结果返回到本地根路径。

### 结构弛豫

#### 步骤 1 —— `make`

在单独的文件夹中准备以下文件。

```sh
├── machine.json
├── relaxation.json
├── confs
│   ├── mp-3034
```

**重要!** 上述 ID, mp-3034, 是 Material Project 数据库中 Al-Mg-Cu 条目的 ID.

为了利用`pymatgen`与 Material Project (MP) 相结合的优势，通过 MP-ID 自动生成计算任务的文件，您应该在环境变量配置文件`.bashrc`中添加 Material Project 的API。

您可以通过运行此命令轻松完成此操作:

```bash
vim .bashrc
// add this line into this file, `export MAPI_KEY="your-api-key-for-material-projects"`
```

如果您对 Material Project 的 api-key 没有了解，请参考[这里](https://materialsproject.org/api#:~:text=API%20Key,-Your%20API%20Key&text=To%20make%20any%20request%20to,anyone%20you%20do%20not%20trust.)。

- `machine.json`与 “init” 和 “run” 中使用的相同。有关它的更多信息，请查看[这里](https://bohrium-doc.dp.tech/#/docs/DP-GEN?id=步骤3：准备计算文件)。
- relaxtion.json

```json
{
    "structures":         ["confs/mp-3034"],//in this folder, confs/mp-3034, required files and scripts will be generated automatically by `dpgen autotest make relaxation.json`
    "interaction": {
            "type":        "deepmd",
            "model":       "graph.pb",
            "in_lammps":   "lammps_input/in.lammps",
            "type_map":   {"Mg":0,"Al": 1,"Cu":2} //if you  calculate other materials, remember to modify element types here.
    },
    "relaxation": {
            "cal_setting":{"etol": 1e-12,
                           "ftol": 1e-6,
                           "maxiter": 5000,
                           "maximal": 500000,
                           "relax_shape":     true,
                           "relax_vol":       true}
    }
}
```

运行以下命令：

```bash
dpgen autotest make relaxation.json 
```

然后自动生成用于计算的相应文件和脚本。

#### 步骤 2 —— `run`

```bash
nohup dpgen autotest run relaxation.json machine.json &
```

运行此命令以执行结构弛豫
。
#### 步骤 3 —— `post`

```bash
dpgen autotest post relaxation.json 
```

### 性质计算

#### 步骤 1 —— `make`

用于属性计算的参数位于 `property.json` 中。

```json
{
    "structures":       ["confs/mp-3034"],
    "interaction": {
        "type":          "deepmd",
        "model":         "graph.pb",
        "deepmd_version":"2.1.0",
        "type_map":     {"Mg":0,"Al": 1,"Cu":2}
    },
    "properties": [
        {
         "type":         "eos",
         "vol_start":    0.9,
         "vol_end":      1.1,
         "vol_step":     0.01
        },
        {
         "type":         "elastic",
         "norm_deform":  2e-2,
         "shear_deform": 5e-2
        },
        {
         "type":             "vacancy",
         "supercell":        [3, 3, 3],
         "start_confs_path": "confs"
        },
        {
         "type":         "interstitial",
         "supercell":   [3, 3, 3],
         "insert_ele":  ["Mg","Al","Cu"],
         "conf_filters":{"min_dist": 1.5},
         "cal_setting": {"input_prop": "lammps_input/lammps_high"}
        },
        {
         "type":           "surface",
         "min_slab_size":  10,
         "min_vacuum_size":11,
         "max_miller":     2,
         "cal_type":       "static"
        }
        ]
}
```

运行以下命令：

```bash
dpgen autotest make property.json
```

#### 步骤 2 —— `run`

运行以下命令：

```bash
nohup dpgen autotest run property.json machine.json &
```

#### 步骤 3 —— `post`

运行以下命令：

```sh
dpgen autotest post property.json
```

在文件夹中，您可以使用命令 `tree . -L 1`，然后查看输出：

```sh
(base) ➜ mp-3034 tree . -L 1
.
├── dpdispatcher.log
├── dpgen.log
├── elastic_00
├── eos_00
├── eos_00.bk000
├── eos_00.bk001
├── eos_00.bk002
├── eos_00.bk003
├── eos_00.bk004
├── eos_00.bk005
├── graph_new.pb
├── interstitial_00
├── POSCAR
├── relaxation
├── surface_00
└── vacancy_00
```

- 01.eos：状态方程；

```bash
(base) ➜ mp-3034 tree eos_00 -L 1
eos_00
├── 99c07439f6f14399e7785dc783ca5a9047e768a8_flag_if_job_task_fail
├── 99c07439f6f14399e7785dc783ca5a9047e768a8_job_tag_finished
├── 99c07439f6f14399e7785dc783ca5a9047e768a8.sub
├── backup
├── graph.pb -> ../../../graph.pb
├── result.json
├── result.out
├── run_1660558797.sh
├── task.000000
├── task.000001
├── task.000002
├── task.000003
├── task.000004
├── task.000005
├── task.000006
├── task.000007
├── task.000008
├── task.000009
├── task.000010
├── task.000011
├── task.000012
├── task.000013
├── task.000014
├── task.000015
├── task.000016
├── task.000017
├── task.000018
├── task.000019
└── tmp_log
```

`EOS` 计算结果显示在 `eos_00/results.out` 文件中：

```sh
(base) ➜ eos_00 cat result.out 
conf_dir: /root/1/confs/mp-3034/eos_00
 VpA(A^3)  EpA(eV)
 15.075   -3.2727 
 15.242   -3.2838 
 15.410   -3.2935 
 15.577   -3.3019 
 15.745   -3.3090 
 15.912   -3.3148 
 16.080   -3.3195 
 16.247   -3.3230 
 16.415   -3.3254 
 16.582   -3.3268 
 16.750   -3.3273 
 16.917   -3.3268 
 17.085   -3.3256 
 17.252   -3.3236 
 17.420   -3.3208 
 17.587   -3.3174 
 17.755   -3.3134 
 17.922   -3.3087 
 18.090   -3.3034 
 18.257   -3.2977 
```

- 02.elastic: 杨氏弹性模量；
`elastic` 计算结果显示在 `elastic_00/results.out` 文件中：

```bash
(base) ➜ elastic_00 cat result.out 
/root/1/confs/mp-3034/elastic_00
 124.32   55.52   60.56    0.00    0.00    1.09 
  55.40  125.82   75.02    0.00    0.00   -0.17 
  60.41   75.04  132.07    0.00    0.00    7.51 
   0.00    0.00    0.00   53.17    8.44    0.00 
   0.00    0.00    0.00    8.34   37.17    0.00 
   1.06   -1.35    7.51    0.00    0.00   34.43 
# Bulk   Modulus BV = 84.91 GPa
# Shear  Modulus GV = 37.69 GPa
# Youngs Modulus EV = 98.51 GPa
# Poission Ratio uV = 0.31
```

- 03.vacancy：空位形成能量;
`vacancy` 计算结果显示在 `vacancy_00/results.out` 文件中：

```bash
(base) ➜ vacancy_00 cat result.out 
/root/1/confs/mp-3034/vacancy_00
Structure:      Vac_E(eV)  E(eV) equi_E(eV)
[3, 3, 3]-task.000000: -10.489  -715.867 -705.378 
[3, 3, 3]-task.000001:   4.791  -713.896 -718.687 
[3, 3, 3]-task.000002:   4.623  -714.064 -718.687 
```

- 04.interstitial：间隙形成能量;
`interstitial` 计算结果显示在 `interstitial_00/results.out` 文件中：

```bash
(base) ➜ vacancy_00 cat result.out 
/root/1/confs/mp-3034/vacancy_00
Structure:      Vac_E(eV)  E(eV) equi_E(eV)
[3, 3, 3]-task.000000: -10.489  -715.867 -705.378 
[3, 3, 3]-task.000001:   4.791  -713.896 -718.687 
[3, 3, 3]-task.000002:   4.623  -714.064 -718.687 
```

- 05.surf：表面形成能量；
`surface` 计算结果显示在 `surface_00/results.out` 文件中：

```bash
(base) ➜ surface_00 cat result.out  
/root/1/confs/mp-3034/surface_00
Miller_Indices:         Surf_E(J/m^2) EpA(eV) equi_EpA(eV)
[1, 1, 1]-task.000000:          1.230      -3.102   -3.327
[1, 1, 1]-task.000001:          1.148      -3.117   -3.327
[2, 2, 1]-task.000002:          1.160      -3.120   -3.327
[2, 2, 1]-task.000003:          1.118      -3.127   -3.327
[1, 1, 0]-task.000004:          1.066      -3.138   -3.327
[2, 1, 2]-task.000005:          1.223      -3.118   -3.327
[2, 1, 2]-task.000006:          1.146      -3.131   -3.327
[2, 1, 1]-task.000007:          1.204      -3.081   -3.327
[2, 1, 1]-task.000008:          1.152      -3.092   -3.327
[2, 1, 1]-task.000009:          1.144      -3.093   -3.327
[2, 1, 1]-task.000010:          1.147      -3.093   -3.327
[2, 1, 0]-task.000011:          1.114      -3.103   -3.327
[2, 1, 0]-task.000012:          1.165      -3.093   -3.327
[2, 1, 0]-task.000013:          1.137      -3.098   -3.327
[2, 1, 0]-task.000014:          1.129      -3.100   -3.327
[1, 0, 1]-task.000015:          1.262      -3.124   -3.327
[1, 0, 1]-task.000016:          1.135      -3.144   -3.327
[1, 0, 1]-task.000017:          1.113      -3.148   -3.327
[1, 0, 1]-task.000018:          1.119      -3.147   -3.327
[1, 0, 1]-task.000019:          1.193      -3.135   -3.327
[2, 0, 1]-task.000020:          1.201      -3.089   -3.327
[2, 0, 1]-task.000021:          1.189      -3.092   -3.327
[2, 0, 1]-task.000022:          1.175      -3.094   -3.327
[1, 0, 0]-task.000023:          1.180      -3.100   -3.327
[1, 0, 0]-task.000024:          1.139      -3.108   -3.327
[1, 0, 0]-task.000025:          1.278      -3.081   -3.327
[1, 0, 0]-task.000026:          1.195      -3.097   -3.327
[2, -1, 2]-task.000027:         1.201      -3.121   -3.327
[2, -1, 2]-task.000028:         1.121      -3.135   -3.327
[2, -1, 2]-task.000029:         1.048      -3.147   -3.327
[2, -1, 2]-task.000030:         1.220      -3.118   -3.327
[2, -1, 1]-task.000031:         1.047      -3.169   -3.327
[2, -1, 1]-task.000032:         1.308      -3.130   -3.327
[2, -1, 1]-task.000033:         1.042      -3.170   -3.327
[2, -1, 0]-task.000034:         1.212      -3.154   -3.327
[2, -1, 0]-task.000035:         1.137      -3.165   -3.327
[2, -1, 0]-task.000036:         0.943      -3.192   -3.327
[2, -1, 0]-task.000037:         1.278      -3.144   -3.327
[1, -1, 1]-task.000038:         1.180      -3.118   -3.327
[1, -1, 1]-task.000039:         1.252      -3.105   -3.327
[1, -1, 1]-task.000040:         1.111      -3.130   -3.327
[1, -1, 1]-task.000041:         1.032      -3.144   -3.327
[1, -1, 1]-task.000042:         1.177      -3.118   -3.327
[2, -2, 1]-task.000043:         1.130      -3.150   -3.327
[2, -2, 1]-task.000044:         1.221      -3.135   -3.327
[2, -2, 1]-task.000045:         1.001      -3.170   -3.327
[1, -1, 0]-task.000046:         0.911      -3.191   -3.327
[1, -1, 0]-task.000047:         1.062      -3.168   -3.327
[1, -1, 0]-task.000048:         1.435      -3.112   -3.327
[1, -1, 0]-task.000049:         1.233      -3.143   -3.327
[1, 1, 2]-task.000050:          1.296      -3.066   -3.327
[1, 1, 2]-task.000051:          1.146      -3.097   -3.327
[1, 0, 2]-task.000052:          1.192      -3.085   -3.327
[1, 0, 2]-task.000053:          1.363      -3.050   -3.327
[1, 0, 2]-task.000054:          0.962      -3.132   -3.327
[1, -1, 2]-task.000055:         1.288      -3.093   -3.327
[1, -1, 2]-task.000056:         1.238      -3.102   -3.327
[1, -1, 2]-task.000057:         1.129      -3.122   -3.327
[1, -1, 2]-task.000058:         1.170      -3.115   -3.327
[0, 0, 1]-task.000059:          1.205      -3.155   -3.327
[0, 0, 1]-task.000060:          1.188      -3.158   -3.327
```

## 总结

现在，用户已经了解了DP-GEN的基本用法。 有关更多信息，请参阅：

1. GitHub：https：//github.com/deepmodeling/dpgen
2. 论文：https：//deepmodeling.com/blog/papers/dpgen/

**翻译：阙浩辉**