---
DOI: [https://doi.org/10.1021/acs.jcim.2c01504]
Description: Vina GPU版本2.0
Publication Year: 2023
Rating: ⭐⭐⭐⭐
Status: Done
Tag: GPU, Vina
---

对原本AutoDock Vina的衍生工具进行了GPU移植，具体的，移植了Quick Vina 2和Quick Vina-W

QuickVina 2 依靠新颖的一阶一致性检查启发式方法来提高对接速度，该方法可以省去一些不必要的局部搜索并保持原始 AutoDock Vina 的精度。QuickVina-W 可以通过添加进程间通信来提高分子对接的准确性和速度，并且 QuickVina-W 支持盲对接，从而无需多次运行对接工具。

# Method

## Vina GPU+

![arch of vina gpu +](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506251720754.jpeg)

- block 1：构象准备
- block 2：grid cache准备，是Vina GPU+新增的功能
- block 3：执行所有受体与配体的对接

### Grid Cache Preparation

在Vina-GPU中，grid cache直接作为输入数据；而在Vina GPU+中，其成为了需要计算得到的数据，并且由GPU加速计算得到

为了减少重复计算，Vina-GPU+会为网格点处可能出现的所有共17种不同原子类型的分子间能量，进行预计算。最后grid cache存放在全局内存种，所有GPU线程共享

### Molecular Docking

过程与Vina-GPU一致

## QuickVina 2-GPU

![arch of quickvina 2-gpu](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506251719707.jpeg)

大体的工作流程与Vina-GPU非常相像，但是具体的每个线程执行的MC+BFGS流程不太一样

QuickVina 2-GPU采用了”first-order-necessary-conditon”的启发式方法。当一个构象C_i与它的2N个最近的构象中，存在一个构象C_j，当它和 C_i满足一定条件时， C_i 就是重要的

确定出“重要”的构象 C_i 后，通过对其求偏导，可以得知驻点在 C_i 的左侧还是右侧，这样可以确定BFGS的初始搜索方向，加快搜索

## Quick Vina-W-GPU

![arch of quickvina-w-gpu](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506251718660.jpeg)

每个work item会给对应的线程添加一个全局缓冲区和单独缓冲区。所有线程中的最佳构象会根据三维位置存储在全局缓冲区中，全局缓冲区以八叉树的形式实现

八叉树会把整个对接盒子（docking box）划分为多个小的子空间，可以减少搜索的计算量。每个搜索空间都可以视为八叉树的根节点，8个小盒子为子节点，以此递归。又由于GPU并行计算无法用递归算法，所以在实现中八叉树是动态分配的，存放在全局内存中。具体的，GPU根据线程数量和搜索步长分配内存，并且根据分子构象的三维位置为每个构象确定一个二进制编码，将其存储在全局内存中。与二进制编码相对应的全局缓冲区中的分子构象即为最近构象。

对于具体的每个线程所执行的事情，相比Quick Vina，它多了一次重要性检查。在Quick Vina中，它只会做一次本地检查(I-check)；而在Quick Vina-W中，还增加了一个全局检查(G-check)：

- G-check会检查线程中正在处理的分子构象和全局内存中的构象的空间距离，决定其是否重要。注意，在这里，全局内存中的构象是其他线程在其搜索空间内搜索到的最佳构象
- 如果G-Check不通过，则再进行I-Check

# Eval

在RIPK1和RIPK3上进行虚拟筛选，以及在AutoDock-GPU的数据集上测试对接准确度

结果就是相对于未加速的AutoDock Vina、Quick Vina 2和Quick Vina-W平均实现了65.6, 1.4和3.6倍的加速，并且确保了对接精度相当