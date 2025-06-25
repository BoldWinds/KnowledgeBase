---
DOI: [https://doi.org/10.1109/TCBB.2024.3467127]
Description: Vina GPU版本2.1
Publication Year: 2024
Rating: ⭐⭐⭐⭐
Status: Done
Tag: GPU, Vina
---

主要做了以下三件事的优化：
- 进一步优化BFGS算法，提出RILC-BFGS
- 优化Gird Cache的共享
- 预测结合口袋，优化配体结构

# Method

大体分为4步：
1. 从受体蛋白质中选取最好的结构
2. 用COACH-D方法预测对接口袋
3. 用Gypsum-DL优化配体结构
4. 分子对接：又可以分为三个步骤，分别是预处理、计算grid cache和实际对接
![arch of vina gpu 2.1](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506251722648.jpeg)

## RILC-BFGS

解决了原先BFGS的三大问题：

1. long iteration：循环非常耗时
2. inaccurate line search：采用的Armijo-Goldstein线搜索方法简单，在收敛性和精度上存在局限性
3. high computational complexity：矩阵更新导致计算量非常大，达到 O(n^2)

RILC-BFGS给出了对应的解决办法：

1. 采用提前停止标准
2. 用不精确的 Lewis-Overton 线搜索方法替换 Armijo-Goldstein 线搜索
3. 采用具有两循环递归和谨慎更新特性的低复杂度的 LBFGS 算法，

代码逻辑如下：
![code](https://lbw-img-lbw.oss-cn-beijing.aliyuncs.com/img/202506251721770.png)

最终，RILC-BFGS实现了 $O(mn)$ 的算法复杂度，其中 $m$ 是内存矩阵的容量， $n$ 是原子数量？

## Grid Cache Sharing

还以为有啥新奇的，结果和2.0版本中的Grid Cache相同

## Structure Optimization

### Receptor Preparation

传统方法会选用分辨率最低的结构，因为认为低分辨率可以带来更高的结构精度。但是根据观察，这样做并不能一直产生最佳的筛选精度

在Vina GPU 2.1中，为了解决这个问题，采取了一种交叉对接的方法：

1. 获取与目标受体对应的所有晶体结构
2. 采用交叉对接方法，将每个受体结构分别与从其他结构中提取的配体对接
3. 基于RMSD选择最佳结构：对接后选择平均RMSD最低的受体结构
4. 使用 OpenBabel 对选定的受体结构添加氢原子，已解决氢原子缺失或结构不完整的问题

### Pocket Prediction

把经过前面步骤得到的受体结构上传到 COACH-D 的服务器上，该服务器会返回口袋的五个预测以及相应的残留指数。根据该残留指数，可以计算出口袋的中心和大小

### Ligand Optimization

使用 Gypsum-DL 生成配体的 3D 结构，再将其转换为PDBQT的格式。目的是为了更精细的建模配体

# Eval

### 评估指标

- Enhancement Factor(EF): %EF_{x\%}=\frac{Hits_s}{N_s}\times \frac{N_t}{Hits_t}%
- Number of hits(Hit): $Hit_{x\%}=\frac{Hits_s}{Hits_t} \times 100\%$

其中 $Hits_s$ 是$top x\%$ 化合物($N_s$)中活跃的化合物数量； $Hits_t$ 是所有化合物($N_t$)中活跃的化合物数量

文章在以下数据库上分别进行了准确率和运行速度的对比：
- AutoDock-GPU 140复合物数据集
- DrugBank上的虚拟药物筛选
- Selleck上的虚拟药物筛选
- 消融实验：分别对比准确率和运行速度

具体实验结果看图