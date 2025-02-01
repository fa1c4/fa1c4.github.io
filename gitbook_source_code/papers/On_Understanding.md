# On Understanding and Forecasting Fuzzers Performance with Static Analysis [CCS 2024]

![image-20250127222112881](assets/image-20250127222112881.png)



对于模糊测试的实践者来说，必须理解不同技术对测试结果的影响，并根据要测试的程序选择理想的配置。现有的研究比较组装好的fuzzer，却没有评估单独一种技术对测试结果的贡献，并且组合后的模糊测试器很难被分解为相互独立的组件。

本文引入一种新方法，在编译时提取静态分析特征与各种模糊测试技术的性能结果相关联。该方法使用不同的度量来揭示程序静态属性与模糊器动态运行时性能之间的关系。展示了**机器学习**模型如何使用通过**静态分析**收集的信息，为特定程序提出定制的模糊器配置。

在11个测试程序中，与 AFLplusplus、LibFuzzer 和 Honggfuzz 相比，使用建议配置的模糊测试程序取得了最好的性能提升。



## Method











## Conclusions







