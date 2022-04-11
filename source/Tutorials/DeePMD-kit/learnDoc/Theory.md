# 理论

在介绍DP具体方法之前, 我们首先定义一个 <img src="https://latex.codecogs.com/svg.image?N"> 原子系统的坐标矩阵 <img src="https://latex.codecogs.com/svg.image?\mathcal{R}&space;\in&space;\mathbb{R}^{N&space;\times&space;3}">，

<img src="https://latex.codecogs.com/svg.image?\mathcal{R}=\left\{{r}_{1}^{T},&space;\cdots,&space;{r}_{i}^{T},&space;\cdots,&space;{r}_{N}^{T}\right\}^{T},&space;{r}_{i}=\left(x_{i},&space;y_{i},&space;z_{i}\right),(1)">

<img src="https://latex.codecogs.com/svg.image?{r}_{i}"> 表示原子 <img src="https://latex.codecogs.com/svg.image?i"> 的三维笛卡尔坐标。此外，我们将坐标矩阵 <img src="https://latex.codecogs.com/svg.image?\mathcal{R}"> 转换成局域坐标矩阵 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\left\{{\mathcal{R}}^{i}\right\}_{i=1}^{N}">,

<img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{R}}^{i}=\left\{{r}_{1&space;i}^{T},&space;\cdots,&space;{r}_{j&space;i}^{T},&space;\cdots,&space;{r}_{N_{i},&space;i}^{T}\right\}^{T},&space;{r}_{j&space;i}=\left(x_{j&space;i},&space;y_{j&space;i},&space;z_{j&space;i}\right),(2)">


其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}j">和<img src="https://latex.codecogs.com/svg.image?\dpi{110}N_{i}"> 是原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 在截断半径<img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{c}">内近邻原子的编号， <img src="https://latex.codecogs.com/svg.image?j&space;\left(1&space;\leq&space;j&space;\leq&space;N_{i}\right)"> 表示原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 的近邻原子编号, <img src="https://latex.codecogs.com/svg.image?\dpi{110}{r}_{j&space;i}&space;\equiv&space;{r}_{j}-{r}_{i}"> 表示的是原子<img src="https://latex.codecogs.com/svg.image?\dpi{110}j"> 和原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 之间的相对距离。

在DP方法中, 一个系统的总能量 <img src="https://latex.codecogs.com/svg.image?\dpi{110}E"> 等于各个原子的局域能量的总和

<img src="https://latex.codecogs.com/svg.image?\dpi{110}&space;E=\sum_{i}&space;E_{i},(3)">

其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}E_{i}"> 是原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 的局域能量. 此外，<img src="https://latex.codecogs.com/svg.image?\dpi{110}E_{i}"> 取决于原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 的局域环境:

<img src="https://latex.codecogs.com/svg.image?\dpi{110}&space;E=\sum_{i}&space;E_{i}=\sum_{i}&space;E\left(\mathcal{R}^{i}\right),(4)">

可以通过以下两个步骤得到 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{R}}^{i}"> 到 <img src="https://latex.codecogs.com/svg.image?\dpi{110}E_{i}"> 的映射：  
第一步，如图[figure](https://gitee.com/liangwenshuo1118/myblog/blob/master/img/descriptor.png) 所示,通过将 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{R}}^{i}">要映射到特征矩阵，或者说描述子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{D}}^{i}">，这里的 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{D}}^{i}"> 保留了体系的平移、旋转和置换不变性。具体来说， <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{R}}^{i}&space;\in&space;\mathbb{R}^{N_{i}&space;\times&space;3}"> 首先被映射到一个扩展矩阵 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\tilde{{\mathcal{R}}}^{i}&space;\in&space;\mathbb{R}^{N_{i}&space;\times&space;4}">，

<img src="https://latex.codecogs.com/svg.image?\dpi{110}&space;\left\{x_{j&space;i},&space;y_{j&space;i},&space;z_{j&space;i}\right\}&space;\mapsto\left\{s\left(r_{j&space;i}\right),&space;\hat{x}_{j&space;i},&space;\hat{y}_{j&space;i},&space;\hat{z}_{j&space;i}\right\},(5)">

其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\hat{x}_{j&space;i}=\frac{s\left(r_{j&space;i}\right)&space;x_{j&space;i}}{r_{j&space;i}}">, <img src="https://latex.codecogs.com/svg.image?\dpi{110}\hat{y}_{j&space;i}=\frac{s\left(r_{j&space;i}\right)&space;y_{j&space;i}}{r_{j&space;i}}">,  <img src="https://latex.codecogs.com/svg.image?\dpi{110}\hat{z}_{j&space;i}=\frac{s\left(r_{j&space;i}\right)&space;z_{j&space;i}}{r_{j&space;i}}">. <img src="https://latex.codecogs.com/svg.image?\dpi{110}s\left(r_{j&space;i}\right)"> 是一个权重函数，用来减少离原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 比较远的原子的权重, 定义如下:

<img src="https://latex.codecogs.com/svg.image?\dpi{110}s\left(r_{j&space;i}\right)=&space;\begin{cases}\frac{1}{r_{j&space;i}},&space;&&space;r_{j&space;i}<r_{c&space;s}&space;\\&space;\frac{1}{r_{j&space;i}}&space;\{&space;{(\frac{r_{j&space;i}&space;-&space;r_{c&space;s}}{&space;r_c&space;-&space;r_{c&space;s}})}^3&space;(-6&space;{(\frac{r_{j&space;i}&space;-&space;r_{c&space;s}}{&space;r_c&space;-&space;r_{c&space;s}})}^2&space;&plus;15&space;\frac{r_{j&space;i}&space;-&space;r_{c&space;s}}{&space;r_c&space;-&space;r_{c&space;s}}&space;-10)&space;&plus;1&space;\},&space;&&space;r_{c&space;s}<r_{j&space;i}<r_{c}&space;\\&space;0,&space;&&space;r_{j&space;i}>r_{c}\end{cases},(6)">

其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{j&space;i}"> 是原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 和原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}j"> 之间的欧式距离, <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{cs}"> 是“平滑截断半径”。引入 <img src="https://latex.codecogs.com/svg.image?\dpi{110}s\left(r_{j&space;i}\right)"> 之后， <img src="https://latex.codecogs.com/svg.image?\dpi{110}\tilde{{\mathcal{R}}}^{i}"> 里的各个参数会从 <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{cs}"> 到 <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{c}"> 平滑地趋于零。 接着 <img src="https://latex.codecogs.com/svg.image?\dpi{110} \{s\left(r_{j&space;i}\right)\}_{j=1}^{N_i}">, 也就是 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\tilde{{\mathcal{R}}}^{i}"> 的第一列通过一个嵌入神经网络得到一个嵌入矩阵 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\mathcal{G}^{i&space;1}&space;\in&space;\mathbb{R}^{N_{i}&space;\times&space;M_{1}}">. 选取 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{G}}^{i&space;1}&space;\in&space;\mathbb{R}^{N_{i}&space;\times&space;M_{1}}"> 的前 <img src="https://latex.codecogs.com/svg.image?\dpi{110}M_{2}(<M_{1})"> 列，我们就得到了另外一个嵌入矩阵 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\mathcal{G}^{i&space;2}&space;\in&space;\mathbb{R}^{N_{i}&space;\times&space;M_{2}}">. 最后，我们就可以得到原子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}i"> 的描述子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{D}}^{i}">：

<img src="https://latex.codecogs.com/svg.image?\dpi{110}\mathcal{D}^{i}=\left(\mathcal{G}^{i&space;1}\right)^{T}&space;\tilde{\mathcal{R}}^{i}\left(\tilde{\mathcal{R}}^{i}\right)^{T}&space;\mathcal{G}^{i&space;2},(7)">

在描述子中, 平移和旋转不变性是由矩阵乘积 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\tilde{\mathcal{R}}^{i}\left(\tilde{\mathcal{R}}^{i}\right)^{T}"> 来保证的, 置换不变性是由矩阵乘积 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\left(\mathcal{G}^{i}\right)^{T}&space;\tilde{\mathcal{R}}^{i}"> 来保证的。  

第二步, 每一个描述子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}{\mathcal{D}}^{i}"> 都将通过一个拟合神经网络被映射到一个局域能量 <img src="https://latex.codecogs.com/svg.image?\dpi{110}E_{i}"> 上面。

嵌入神经网络 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\mathcal{N}^e"> 和拟合神经网络 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\mathcal{N}^f"> 都是包含很多隐藏层的前馈神经网络。 前一层的输入数据 <img src="https://latex.codecogs.com/svg.image?\dpi{110}d_{l}^{\mathrm{in}}"> 通过一个线性运算和一个非线性的激活函数得到下一层的输入数据 <img src="https://latex.codecogs.com/svg.image?\dpi{110}d_{k}^{\mathrm{out}}">.

<img src="https://latex.codecogs.com/svg.image?\dpi{110}d_{k}^{o&space;u&space;t}=\varphi\left(\sum_{k&space;l}&space;w_{k&space;l}&space;d_{l}^{i&space;n}&plus;&space;b_{k}\right),(8)">

在公式（8）中, <img src="https://latex.codecogs.com/svg.image?\dpi{110}{w}_{k&space;l}"> 是权重参数, <img src="https://latex.codecogs.com/svg.image?\dpi{110}{b}_{k}"> 是偏置参数，<img src="https://latex.codecogs.com/svg.image?\dpi{110}\varphi"> 是一个非线性的激活函数。需要注意的是，在最后一层的输出节点是没有非线性激活函数的。在嵌入网络和拟合网络中的参数由最小化代价函数 <img src="https://latex.codecogs.com/svg.image?\dpi{110}L"> 得到:
<img src="https://latex.codecogs.com/svg.image?\dpi{110}L\left(p_{\epsilon},&space;p_{f},&space;p_{\xi}\right)=\frac{p_{\epsilon}}{N}&space;\Delta&space;\epsilon^{2}&plus;\frac{p_{f}}{3&space;N}&space;\sum_{i}\left|\Delta&space;{F}_{i}\right|^{2}&plus;\frac{p_{\xi}}{9N}\|\Delta&space;\xi\|^{2},(9)">

其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\Delta&space;\epsilon">, <img src="https://latex.codecogs.com/svg.image?\dpi{110}\Delta&space;{F}_{i}">, 和 <img src="https://latex.codecogs.com/svg.image?\dpi{110}\Delta&space;\xi"> 分别表示能量、力和维里的方均根误差 (RMSE) .
在训练的过程中, 前置因子 <img src="https://latex.codecogs.com/svg.image?\dpi{110}p_{\epsilon}">, <img src="https://latex.codecogs.com/svg.image?\dpi{110}p_{f}">, 和 <img src="https://latex.codecogs.com/svg.image?\dpi{110}p_{\xi}"> 由公式

<img src="https://latex.codecogs.com/svg.image?\dpi{110}p(t)=p^{\operatorname{limit}}\left[1-\frac{r_{l}(t)}{r_{l}^{0}}\right]&plus;p^{\operatorname{start}}\left[\frac{r_{l}(t)}{r_{l}^{0}}\right],(10)">

决定，其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{l}(t)"> 和 <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{l}^{0}"> 分别表示在训练步数为 <img src="https://latex.codecogs.com/svg.image?\dpi{110}t"> 和训练步数为0 时的学习率。 <img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{l}(t)"> 的定义为

<img src="https://latex.codecogs.com/svg.image?\dpi{110}r_{l}(t)=r_{l}^{0}&space;\times&space;d_{r}^{t&space;/&space;d_{s}},(11)">

其中 <img src="https://latex.codecogs.com/svg.image?\dpi{110}d_{r}"> 和 <img src="https://latex.codecogs.com/svg.image?\dpi{110}d_{s}"> 分别表示学习衰减率以及衰减步数。学习衰减率 <img src="https://latex.codecogs.com/svg.image?\dpi{110}d_{r}"> 要严格小于1。
如果读者想要了解更多细节，可以查看文章[DeepPot-SE](https://proceedings.neurips.cc/paper/2018/file/e2ad76f2326fbc6b56a45a56c59fafdb-Paper.pdf).

**翻译：范家豪  校对：杜云珍**
