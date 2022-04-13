# 如何在五分钟之内Setup一次DeepMD-kit训练
DeepMD-kit 是一款实现深度势能(Deep Potential)的软件。虽然网上已经有很多关于DeepMD-kit的资料和信息以及官方指导文档，但是对于小白用户仍不是很友好。今天，这篇教程将在五分钟内带你入门DeepMD-kit。

首先让我们先关注一下DeepMD-kit的训练流程：

数据准备->训练->冻结/压缩模型

是不是觉得很简单？让我们对这几个步骤作更深的阐述：

1. ***数据准备*** : 将DFT的计算结果转化成DeepMD-kit能够识别的数据格式。
2. ***训练*** : 使用前一步准备好的数据通过DeepMD-kit来训练一个深度势能模型(Deep Potential Model)
3. ***冻结/压缩模型*** :最后我们需要做的就是冻结/压缩一个训练过程中输出的重启动文件来生成模型。相信你已经迫不及待想要上手试试啦！让我们开始吧！

## 下载数据
第一步，让我们下载和解压教程中提供给我们的数据:
```bash
$ wget https://dp-public.oss-cn-beijing.aliyuncs.com/community/DeePMD-kit-FastLearn.tar
$ tar xvf DeePMD-kit-FastLearn.tar
```

然后我们可以进入到下载好的数据目录检查一下 :
为不同的目的设置了三个目录：
  * 00.data：包含 VASP 结果 OUTCAR 的示例
  * 01.train：包含DeePMD-kit配置示例input.json
  * data：包含 DeePMD-kit 训练/验证数据的示例
```   
$ cd DeePMD-kit-FastLearn
$ ls
00.data 01.train data
``` 
## 数据准备
现在让我们进入到00.data目录 :
```bash
$ cd 00.data
$ ls
OUTCAR
```
目录当中有一个VASP计算结果输出`OUTCAR`文件，我们需要将他转换成DeepMD-kit数据格式。DeepMD-kit数据格式在[官方文档](https://deepmd.readthedocs.io/)中有相应的介绍，但是看起来很复杂。别怕，这里将介绍一款数据转换神器dpdata，只用一行python命令就能处理好数据，是不是超级方便！
```bash
import dpdata
dpdata.LabeledSystem('OUTCAR').to('deepmd/npy', 'data', set_size=200)
```
在上面的命令中，我们将VASP计算结果输出文件`OUTCAR`转换成DeepMD-kit数据格式并保存在`data`目录当中，其中`npy`是指numpy压缩格式，也就是DeepMD-kit训练所用的数据格式。

假设你已经有了分子动力学输出“OUTCAR”文件，其中包含了1000帧。`set_size=200`将会将1000帧分成五个子集，每份200帧，分别命名为`data/set.000`\~`data/set.004`。这五个子集中，`data/set.000`\~`data/set.003`将会被DeepMD-kit用作训练集，`data/set.004`将会被用作测试集。如果设置`set_size=1000`，那么只有一个集合，这个集合既是训练集又是测试集(当然这个测试的参考意义就不大了)。

教程当中提供的`OUTCAR`只包含了一帧数据，所以在`data`目录(与`OUTCAR`在同一目录)当中将只会出现一个集合data/set.000。如果你想用对数据做一些其他的处理，更详细的信息请参考下一章。

我们将跳过更详细的处理步骤，接下来我们进入到根目录当中使用教程已经为你提供好的数据。
```bash
$ cd ..
$ ls
00.data 01.train data
```
## 训练
开始DeepMD-kit训练需要准备一个输入脚本，大家是不是还有没从被INCAR脚本支配的恐惧中走出来？别怕，配置DeepMD-kit比配置VASP简单多了。我们已经为你准备好了`input.json`，你可以在"01.train"目录当中找到它
```bash
$ cd 01.train
$ ls
input.json
```
DeepMD-kit的强大之处在于一样的训练参数可以适配不同的系统，所以我们只需要微调`input.json`来开始训练。首先需要修改的参数是 :
```bash
"type_map":     ["O", "H"],
```
DeepMD-kit中对每个原子类型是按从0开始的整数编号的。这个参数给了这样的编号系统中每个原子类型一个元素名。这里我们照写`data/type_map.raw`的内容就好。比如我们改成 :
```bash
"type_map":    ["A", "B","C"],
```
其次，我们修改一下近邻搜索参数 :
```bash
"sel":       [46, 92],
```
这个list中每个数给出了某原子的近邻原子中，各个类型的原子的最大数目。比如`46`就是类型为0,元素为`O`的近邻数最多有46个。这里我们换成了ABC，这个参数我们要相应的修改。不知道最大近邻数怎么办？可以用体系的密度粗略估计一个。或者可以盲试一个数，如果不够大DeepMD-kit会在WARNINGS中告诉你的。下面我们改成了 :
```bash
"sel":       [64, 64, 64]
```
此外我们需要修改的参数是`"training_data"`中的`"systems"`
```bash
"training_data":{
     "systems":     ["../data/data_0/", "../data/data_1/", "../data/data_2/"],
 ```
以及`"validation_data"`
```bash
"validation_data":{
     "systems":     ["../data/data_3"],
```
这里我将稍微介绍一下data system的定义。DeepMD-kit认为，具有同样原子数，且原子类型相同的数据能够形成一个system。我们的数据是从一个分子动力学模拟生成的，当然满足这个条件，因此可以放到一个system中。dpdata也是这么帮我们做的。当数据无法放到一个system中时，就需要设multiple systems，写成一个list。
```bash
"systems": ["system1", "system2"]
```
最后我们还需要改另外一个参数 :
```bash
"numb_steps":   1000,
```
`numb_steps`表示的是深度学习的SGD方法训练多少步(这只是一个例子，你可以在实际使用当中设置更大的步数)。

配置脚本大功告成！训练开始。我们在当前目录下运行 :
```bash
dp train input.json
```
在训练的过程当中，我们可以通过观察输出文件`lcurve.out`来观察训练阶段误差的情况。其中第四列和第五列是能量的训练和测试误差，第六列和第七列是力的训练和测试误差。
##1.4.4. 冻结/压缩模型
训练完成之后，我们可以通过如下指令来冻结模型 :
```bash
dp freeze -o graph.pb
```
其中`-o`可以给模型命名，默认的模型输出文件名是`graph.pb`。同样，我们可以通过以下指令来压缩模型。
```bash
dp compress -i graph.pb -o graph-compress.pb
```
以上，我们就获得了一个或好或坏的DP模型。对于模型的可靠性以及如何使用它，我将在下一章中详细介绍。

**翻译：欧阳冠宇  校对：史国勇**
