---
DOI: [https://doi.org/10.1002/jcc.26779]
Description: 经验系数改进
Publication Year: 2022
Rating: ⭐
Status: Done
Tag: Vina
---

旨在通过调整 AutoDock Vina（Vina）的经验参数，提高其配体亲和力排序的准确性，解决 Vina 对接能量与实验相关性低的问题（在 [[Autodock Vina Adopts More Accurate Binding Poses but Autodock4 Forms Better Binding Affinity]] 中有提到）

# 实验

## 实验设置

使用了两组复合物：
1. 从Protein Data Bank (PDB)下载的800个复合
2. 2019 版 PDBbind中的1036个复合物

使用AutoDockTools生成PDBQT文件，使用Gasteiger-Marsili方法计算原子电荷

Vina的设置就是默认设置：8、7kcal/mol、20_20_20

## 实验方法

为了测试经验参数的改变对结果的影响，每次迭代会改变6种参数中其中一个10%，其他参数不变。对于gauss1, repulsion, hydrophobic 和 hydrogen，它们会在默认值的 $-50\%$ 到 $+50\%$ 波动；而gauss 2在默认值的 $-50\%$ 到 $+150\%$ 波动；rotation在默认值的$ -90\%$ 到 $+50\%$ 波动

由于每次变化10%，所以一共测得78组数据（只改变一个参数），分别是包括RMSD和成功率

在1000个样本上测得

## 相关系数

guass2和rotation对相关系数R的影响很大

实验测得，当gauss2增加时和rotation减小时，R会增加；分别在 $+140\%$ 和 $-60\%$ 取得最高的相关系数

## 结合能

主要对gauss2比较敏感

RMSE对gauss1、gauss2和rotation比较敏感

## 对接成功率

评估了在2埃、1.5埃、1埃和0.5埃下的对接成功率

repulsion在任何截断下都对成功率有比较明显的影响；而对接成功率对疏水项和rotation的变化不敏感

# 结果

对于不同的目的，分出了三个参数组
- Set1：相关系数优先
- Set2：平衡相关系数和成功率
- Set3：成功率优先