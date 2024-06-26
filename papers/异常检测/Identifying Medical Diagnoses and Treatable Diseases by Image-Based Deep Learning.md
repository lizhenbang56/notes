---
文档链接: https://www.cell.com/action/showPdf?pii=S0092-8674%2818%2930154-5
---
## SUMMARY

我们建立了一个基于深度学习框架的诊断工具，用于筛查患有常见可治疗的致盲性**视网膜疾病**的患者。我们的框架利用迁移学习，通过使用少量数据训练神经网络，与传统方法相比，提高了可靠性和可解释性。将这种方法应用于光学相干断层扫描图像数据集，我们展示了与人类专家在分类年龄相关性黄斑变性和糖尿病黄斑水肿方面性能相当的结果。我们还通过突出神经网络识别的区域，提供了更加透明和可解释的诊断。我们进一步展示了我们的人工智能系统在使用胸部X射线图像诊断儿童肺炎方面的普适性。这个工具最终可能有助于加速这些可治疗疾病的诊断和转诊，从而促进早期治疗，提高临床结果。

## INTRODUCTION

人工智能（AI）有潜力通过执行人类专家难以完成的分类任务和快速审查大量图像来彻底改变疾病诊断和管理。尽管具有潜力，但临床可解释性和AI的可行准备仍然具有挑战性。图像分类的传统算法方法先前依赖于（1）手工制作的对象分割，然后是（2）使用针对每个对象类别专门设计的统计分类器或浅层神经计算机学习分类器来识别每个分割对象，并最终（3）对图像进行分类（Goldbaum等，1996年）。创建和改进多个分类器需要许多熟练的人员和大量时间，并且计算成本高昂（Chaudhuri等，1989年；Hoover和Goldbaum，2003年；Hoover等，2000年）。

卷积神经网络层的发展使得在图像分类和检测图像中物体方面取得了显著的进展（Krizhevsky等，2017年；Zeiler和Fergus，2014年）。这些是多个处理层，对其中的图像分析滤波器或卷积进行应用。通过在图像上系统地对多个滤波器进行卷积，构建了每个层内的图像的抽象表示，产生了作为输入传递到下一层的特征映射。这种架构使得能够以像素形式输入图像并输出所需的分类。一个分类器中的图像到分类的方法替代了以前图像分析方法的多个步骤。

解决某一领域数据不足的一种方法是利用来自类似领域的数据，这种技术称为迁移学习。迁移学习已被证明是一种非常有效的技术，特别是当面对数据有限的领域时（Donahue等，2013年；Razavian等，2014年；Yosinski等，2014年）。与训练完全空白的网络不同，通过使用前馈方法固定下层已经优化以识别普遍图像中发现的结构的权重，并使用反向传播重新训练上层的权重，模型可以更快地识别特定类别图像的区别特征，并且需要的训练示例和计算资源明显减少（见图1）。

在这项研究中，我们旨在开发一种有效的迁移学习算法来处理医学图像，以在每个图像中提供准确及时的关键病理诊断。这种技术的主要示例涉及视网膜的光学相干断层扫描（Optical Coherence Tomography, OCT）图像，但也在一组儿科胸部X射线照片中进行了测试，以验证该技术跨多种成像模式的普适性。