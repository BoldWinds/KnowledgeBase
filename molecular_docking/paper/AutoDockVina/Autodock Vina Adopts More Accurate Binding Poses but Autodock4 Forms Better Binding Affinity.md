---
DOI: [https://doi.org/10.1021/acs.jcim.9b00778]
Description: 纯实验，对比Vina和AD4
Publication Year: 2020
Rating: ⭐⭐
Status: Done
Tag: AutoDock4, Vina
---
# **实验方法**

## **数据**

从PDB中获取了800个复合物的三维结构，通过AutoDock Tools对其参数化

## **Vina设置**

全局搜索穷举度设置为8，56，400分别对应短、中、长选项；最大能量差设置为7lcal/mod；网格中心为配体质心，

Vina生成了371200个配体构象，但只记录了48000个用于评估

## **AD4设置**

网格大小设置为[20\AA,20\AA,20\AA]；遗传算法运行次数为50，种群大小和代数分别为300和27000。GA 评估次数选择为 250000、2500000 和 25000000，分别对应短、中、长选项。

AD4共生成了120000 个配体构象

# **结果**

## **配体结合自由能**

总的结果：

- Vina的相关性比AD4要低，但是Vina在短，中，长不同选项下的变换很小，也就是说搜索穷举度从短增加到长似乎并不会让Vina的准确性显著提高；
- Vina的均方根误差也大于AD4。也就是说在本次实验中，Vina的搜索效果是不如AD4的
- Vina对于结合自由能的估计值与实验值相比有比较大的差异

HB与HP：
- 在氢键(HB)的计算上，AD4的相关系数高于Vina，RMSE也更低；
- 疏水(HP)的计算上，Vina的相关系数更高，RMSE也更低
- 但是由于HB相互作用远比HP相互作用要强，所以这可能是AD4预测效果好于Vina的一个原因

## **配体结合姿势**

若配体结合构象中原子位置相对于相应实验结构的均方根偏差（RMSD）小于 2\AA，那么该配体结合构象就被视为成功对接的形态

在确定配体结合构象方面，Vina 方法要优于 AD4 方案。Vina实验结构与对接形态之间的均方根偏差为$RMSE_{Vina}=1.3\pm 0.03Å$，而$RMSE_{AD4}=1.42\pm0.03Å$

并且AD4的对接成功率也低于Vina方案。知道在$0.5Å$处截断时，两种工具的对接成功率才相等

# **讨论**

以上的结果是总体的数据，文章还对于具体的受体进行讨论。对比了哪些蛋白质适合用Vina；哪些蛋白质适合AD4

# **总结**

对比了AutoDock4 和AutoDock Vina的性能和准确度，本文在800个配体和47个受体上进行了实验。总的来说，Vina收敛速度快很多，但是AD4在21个受体的第一子集中表现更好；Vina在包括10个受体的第二子集中表现更好；而他们在包含16个受体的第三个子集中表现都很差

提出的一个可能的改进方向是Vina或许可以提高HB的权重