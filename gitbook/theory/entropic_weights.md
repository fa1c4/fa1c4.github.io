# Entropic Weights

不同特征之间的权重分配可以通过不同特征的熵差异进行优化, 使得熵更大 (更离散) 的特征获得更高的权重占比, 直观上来说差异更大的特征维度可以更好的区分不同类别. 下面是整理的基于熵的权重分配公式.



Supposing there are m samples and n features in the dataset, $$x_{ij}$$ is the j-th feature value in the i-th sample. In order to eliminate the influence of features dimension on incommensurability, it is necessary to standardize features using the equations of relative optimum membership degree. There are two ways to standardize features value.


$$
r_{i j}^{\prime}=\frac{x_{i j}}{\max _j x_{i j}},(i=1, \ldots, m ; j=1, \ldots, n) \qquad (1)
$$
 
$$
r_{i j}^{\prime}=\frac{\min _j x_{i j}}{x_{i j}}, \min _j x_{i j} \neq 0,(i=1, \ldots, m ; j=1, \ldots, n) \qquad (2)
$$


Calculation of the features' entropy according to the definition of entropy. The entropy of the j-th feature is determined by (3)


$$
H_j=-\frac{\sum_{i=1}^m f_{i j} \ln f_{i j}}{\ln m},(i=1, \ldots, m ; j=1, \ldots, n) \qquad (3)
$$


wherein


$$
f_{i j}=\frac{r_{i j}^{\prime}}{\sum_{i=1}^m r_{i j}},(i=1, \ldots, m ; j=1, \ldots, n) \qquad (4)
$$


Calculation of the feature's entropy weight. Entropy weight of the j-th feature is determined by (5)


$$
w_j=\frac{1-H_j}{n-\sum_{j=1}^n H_j}, \sum_{j=1}^n w_j=1,(j=1, \ldots, n) \qquad (5)
$$




## References

[1] Yufeng WANG, Ming DAI. Safety assessment based on information entropy and unascertained theory. Shanxi Coal,
2009(5):8-10.

[2] Li, Xiangxin, et al. "Application of the entropy weight and TOPSIS method in safety evaluation of coal mines." *Procedia engineering* 26 (2011): 2085-2091.

