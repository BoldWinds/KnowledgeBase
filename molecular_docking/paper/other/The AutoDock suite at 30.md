---
DOI: [https://doi.org/10.1002/pro.3934]
Description: 综述介绍AD工具
Publication Year: 2020
Rating: ⭐
Status: Done
Tag: AutoDock4, Vina
---
介绍30年以来的AutoDock软件套件，包括经验自由能力场、对接引擎、位点预测方法以及用于可视化和分析的交互工具；除此以外还有一些专门化的工具

### AutoDock Suite

有效的分子对接方法需要解决下面两个问题：
- 用一种能反映生物分子间相互作用潜在能量的方式来对复合物试验构象进行评分
- 用一种能充分搜索空间的方法来探索可能的构象

AutoDock开发早期是使用物理方法构建立场来预测构象，但显然这种方法在构象空间较大时很难搜索，所以要想办法简化这些力场。

### AutoDock v1

用体积评估结合能，用模拟退火(annealing)算法作为搜索算法。并施加一些优化：
- 为每种配体原子类型预先计算体积图
- 配体的构象自由度也仅限于扭转旋转，键长和键角被约束为起始构象的几何形状
- 减少原子类型的数量，从而减少要计算和存储的图的数量和大小

### AD4

相比初始版本，做出了以下改进：
- 结合能评估方法：改进了氢键的几何结构；对力场参数进行经验加权
- 搜索方法：混合遗传算法/局部搜索方法
- 在力场描述中添加梯度
- 将AD4移植到GPU

### AutoDock Vina

详情见《AutoDock Vina: Improving the speed and accuracy of docking with a new scoring function, efficient optimization, and multithreading》

### AutoDockFlexible Receptor（ADFR）

- 创建了一种更通用的受体表示形式，不仅可以定义侧链，还能定义在构象搜索过程中会发生运动的环和结构域。也就是说，会考虑==受体的柔性==。PS：AD4和ADVina假设刚性
- 支持对接共价配体

### AutoDock CrankPep（ADCP）

专门用于==肽==对接

### Map Method

Interaction Maps(相互作用图) 是一种用于描述配体与受体相互作用能量分布的工具，一般通过体积预计算配体原类型在受体空间中的能量信息。

### Enhancements

### Solent Effctes 溶剂效应

ADVina采用经验性的方法来估算去除溶剂所带来的影响，这种方法使用的函数对距离的依赖程度非常小，而且没有考虑到桥连水的局部效应。文章提出了更明确的方法：将水分子附着在配体所有可能的相互作用位置上，然后允许这些水分子在对接模拟过程中与蛋白质相互作用或者消失

### 总结

没什么好讲解的内容，这篇文章就是讲解了AutoDock是什么、AutoDock有哪些套件以及AutoDock的诸多套件可以干什么。没有将任何理论相关的东西