# Practical Statistics for Fuzzing

Fuzzing 实验中会涉及一些统计学的理论知识, 本文从基本的概念到实用理论公式等进行总结.



## Hypothesis Testing

假设检验 (Hypothesis Testing) 是统计学中用于判断样本数据是否支持某一假设的统计方法. 它的目的是通过样本数据来评估某个假设是否成立, 比如样本分布是否符合正态分布. 

假设检验包括两个对立的假设

+ 零假设 (Null Hypothesis, $$H_0$$): 通常表示没有效应或没有差异, 或者现象是偶然发生的, 期望通过检验来证伪
+ 备择假设 (Alternative Hypothesis, $$H_1$$): 通常表示有某种效应或差异, 或期望验证某种新的关系

要检验假设, 需要选择合适的检验方法, 例如: t检验、卡方检验、Mann–Whitney U 检验等. 并选择显著性水平 $$\alpha$$, 比如 0.05, 当 p 值小于 $$\alpha$$ 时拒绝零假设, 否则接受零假设. 



## P-Value

p 值 (p-value) 是统计学中用于衡量样本数据与零假设 (null hypothesis) 相符程度的指标.

p 值表示在零假设成立的情况下, 观察到当前样本数据或比其更极端的结果的概率. p 值越小, 意味着数据与零假设越不一致. (注意: p 值并不是零假设成立的概率: 它表示在零假设成立的情况下, 观测到某个统计量值或更极端结果的概率). 

**形式地**, 进行一次假设检验, 其中零假设为 $$H_0$$: 样本符合 t 分布, 备择假设为 $$H_1$$: 样本不符合 t 分布. 设有一个统计量 t, 该统计量基于样本数据计算得出, 并假设其分布已知. 设基于样本数据计算的检验统计量为 $$T_{obs}$$, 假设零假设 $$H_0$$ 成立, 并根据零假设的分布推断出检验统计量的分布 T, 计算 p 值的公式 (右尾检验, 如果是双尾检验则是比较绝对值 $$|T_{obs}|$$), 如下


$$
p = P(T \geq T_{obs} | H_0)
$$




## Vargha-Delaney effect size

**Vargha-Delaney effect size** 是一种非参数效应量度量, 用于比较两个独立样本的效应大小. 公式如下


$$
\hat{A}_{12}=\frac{W}{n_1 \cdot n_2}
$$
其中, $$W$$ 是 Mann–Whitney U 检验的秩和 (rank sum), $$n_1$$ 和 $$n_2$$ 分别是两组样本的大小 (组 1 和组 2 的样本数). 

结果的解释

+ $$\hat{A}_{12} = 0.5$$: 两组数据没有显著差异
+ $$\hat{A}_{12} < 0.5$$: 组 1 优于组 2 
+ $$\hat{A}_{12} > 0.5$$: 组 1 劣于组 2 



科学家们普遍推荐在推断性检验中报告效应量 (effect size), 主要目的有

+ 提供效应的实际意义: **p 值**只能体现结果的显著性, **响应量**则反映结果的重要性
+ 跨研究的比较: 不同的研究可能使用不同的样本量和方法, 报告效应量能够**标准化**结果
+ 提高研究的可重复性: 其他研究者可以通过效应量来验证和比较自己的研究结果, 从而判断一个发现是否具有**可靠性**和**一致性**



## Mann–Whitney U Test

to expand ...





## Reference 

[1] David S. Moore, George P. McCabe, Bruce Craig-Introduction to the Practice of Statistics (6th Edition) - W. H. Freeman

[2] https://en.wikipedia.org/wiki/Mann%E2%80%93Whitney_U_test

[3] Nonparametric Statistical Methods (3rd Edition) - Myles Hollander & Douglas A. Wolfe

