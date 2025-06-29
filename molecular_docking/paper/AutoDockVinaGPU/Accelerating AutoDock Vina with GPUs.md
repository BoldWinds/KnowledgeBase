---
DOI: [https://doi.org/10.3390/molecules27093041]
Description: Vina GPU版本 
Publication Year: 2022
Rating: ⭐⭐⭐⭐
Status: Done
Tag: GPU, Vina
---
- modified Monte Carlo using simulating annealing AI algorithm.
- BFGS优化算法
- OpenCL实现
- 平均21倍对接加速

![Arch of ADvina GPU](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506251714906.jpeg)

## Implementation

- host：准备工作和构象的后优化
- device：MC算法加速

### Host

数据准备部分：
1. 读配体受体文件
2. 设置OpenCL
3. 准备数据
4. GPU内存分配

在数据准备的过程中，Vina原本将每个构象视为异构树，后续对其进行深度优先遍历即可。但是GPU是没有办法上千个线程同时递归的，所以要把异构树改为深度优先顺序的异构节点表

> 异构树(Heterogeneous Tree)
> 
> 一个分子由许多原子构成，这里把分子的构象抽象为树，分子内部的原子相关信息抽象为节点。每个节点会存储原子的位置、方向等基本信息。
> 
> 通过异构树可以从根节点出发，逐步细化到分子中各个原子或原子团的信息，最终表示完整的构象

后处理部分：
1. 集群
2. 排序筛选出最好的20个构象
3. 对最好的20个构象进行增强
4. 输出最终的配体文件

### Device

### 数据存储

- 准备好的输入数据（构象）被存储在GPU的Constant Memory中。Constant Memory用于存储在对接期间不会发生变化的数据，例如grid map, 初始构象等
- 迭代局部搜索的MC算法和最终的构象都是存储在Global Memory中的

### 计算过程

用如下的公式来表示构象：

$$  
C=\left\{x, y, z, a, b, c, d, \psi_{1}, \psi_{2}, ..., \psi_{N_{r o t}}\right\}  
$$
其中x,y,z代表原子的位置；a,b,c,d代表原子的方向，以及\psi_{1}, \psi_{2}, ..., \psi_{N_{rot }}代表N_{rot}个可旋转键的扭转

对于每个这样的构象，都分配一个OpenCL中的`work item`，之后，它会在随机的位置、方向或扭转中的一个方面进行随机突变，模拟分子在实际环境中的动态变化，生成新的可能构象。与此同时，用一个评分函数对突变后的构象进行持续评估。这样不断重复随机突变、评估和筛选的步骤，逐步找到势能更低、更稳定的构象

评估之后用BFGS算法调整下次搜索，通过Metropolis 准则确定是否接受新的构象

## 实验

分为两个方面：
1. 更改为OpenCL实现后的精确度是否由变化
2. 更改为OpenCL实现后的速度是否有提升

### 实验设置

### 复合物

- Astex Diversity Set中的85个复合物
- CASF - 2013中的35个复合物
- Protein Data Bank中的20个复合物

对于以上140个复合物，按照原子数量分为小型、中型和大型复合物

### 环境

Vina：
- 搜索穷举度参数设置为128
- CPU参数设置为最大值20

Vina-GPU:
- OpenCL 3.0
- 分别在1080ti, 2080ti和3090上以FP32精度运行
- 搜索深度由启发式公式（经验性）确定

### 参数

- 线程数：对于小型复合物，大约1000线程时收敛；而对于中型或大型复合物，则需要8000
- 搜索深度：随着搜索深度增加，对小型复合物几乎无影响；中型复合物会快速收敛，运行时间缓慢增加；大型复合物缓慢收敛，运行时间和搜索深度呈线性增加

### 对接准确率

图五的a和b分别在对接分数和RMSD上比较了Vina-GPU与Vina的对接正确率
- 在对接分数上，Vina和Vina-GPU的平均分数为-8.9和-8.7，非常接近
- 在构象的均方根偏差上，取2埃为为截断值。Vina-GPU有107个成功对接；Vina有114个成功对接。RMSD分别为1.7和1.5

### 运行时间

Vina-GPU能实现1.03-50.8不等的加速比（根据GPU和配体复杂程度）

对最耗时的MC算法进行单独分析，也达到了几十倍的加速比

### 构象空间分析（可解释性）

Vina-GPU会有数千个线程同时运行，它们会把整个搜索空间划分成数千个子空间，每个线程在子空间中搜索，最后对所有子空间搜索出来的构象进行排序筛选找到最优构象

图10展示了Vina与Vina-GPU的对比

### 消融实验

由于Vina-GPU的算法本身就可以串行化，作者还把Vina-GPU串行放到CPU上执行，结果几乎与在GPU上并行执行相同

### 实际虚拟筛选

文中展示了Vina和Vina-GPU在真实情况下的虚拟筛选下的结果，实现了14.8的加速比，并且对接结果有高度相似性