---
方法类别: 基于重建
创新点: 把异常图片的特征重建成正常图片的特征。使用 Transformer 作为解码器，解决重建过程中的“同一性”捷径问题
Publish: NIPS2022
Year: "2022"
标题: A Unified Model for Multi-class Anomaly Detection
文档链接: https://arxiv.org/pdf/2206.03687.pdf
类别数: 多类
监督方式: 无监督
Month: "06"
合成异常的方式: 不合成异常样本
I-AUROC-MVTecAD单类无监督: "96.6"
P-AUROC-MVTecAD单类无监督: "96.6"
I-AUROC-MVTecAD多类无监督: "96.5"
P-AUROC-MVTecAD多类无监督: "96.8"
---
- 基于重建的方法：重建由预训练的主干网络提取的特征
- 难点是防止模型学习重建的捷径
- 通过比较重建前后的特征确定异常区域
## Abstract

尽管无监督异常检测技术发展迅速，但现有方法需要为不同对象训练单独的模型。在这项工作中，我们提出了UniAD，它通过统一的框架实现了对多类别的异常检测。在这种具有挑战性的情况下，流行的重建网络可能会陷入“相同的捷径”，即正常和异常样本都能被很好地恢复，因此无法发现异常值。为了克服这一障碍，我们进行了三项改进。首先，我们重新审视全连接层、卷积层以及注意力层的公式，并确认了查询嵌入（即在注意力层内）在防止网络学习捷径中的重要作用。因此，我们提出了一种逐层查询解码器，以帮助模型建模多类别分布。其次，我们采用了一个邻域屏蔽的注意力模块，进一步避免了输入特征到重建输出特征的信息泄漏。第三，我们提出了一种特征抖动策略，促使模型即使在有噪声输入的情况下也能恢复正确的信息。我们在MVTec-AD和CIFAR-10数据集上评估了我们的算法，在这些数据集上，我们的算法均以较大的优势超越了最先进的替代方法。例如，在MVTec-AD中为15个类别学习一个统一模型时，我们在异常检测任务（从88.1%提升至96.5%）和异常定位任务（从89.5%提升至96.8%）上都超过了第二名的竞争者。代码可在[https://github.com/zhiyuanyou/UniAD](https://github.com/zhiyuanyou/UniAD) 上获取。

## 1 Introduction

异常检测在制造缺陷检测 [4]、医学图像分析 [17] 和视频监控 [47] 等领域的应用越来越广泛。考虑到异常类型的多样性，一种常见的解决方案是对正常样本的分布进行建模，然后通过寻找异常值来识别异常样本。因此，学习正常数据的紧凑边界至关重要，如图 1a 所示。为此，现有方法 [6, 11, 26, 28, 49, 50, 53] 提出了为不同类别的对象训练单独模型的方法，如图 1c 所示。然而，这种一类一模型的方案可能会消耗大量内存，特别是随着类别数量的增加，而且在正常样本表现出大量类内多样性的场景下也不适用（即，一个对象由各种类型组成）。

在本工作中，我们致力于解决一个更实际的任务，即使用统一框架从不同对象类别中检测异常。任务设置如图 1d 所示，训练数据涵盖了各种类别的正常样本，并且要求学习的模型在不进行任何微调的情况下对所有这些类别进行异常检测。值得注意的是，分类信息（即类标签）在训练和推断阶段都无法获得，这在很大程度上简化了数据准备的难度。然而，解决这样的任务是非常具有挑战性的。回想一下，无监督异常检测的原理是对正常数据的分布进行建模，并找到一个紧凑的决策边界，如图 1a 所示。当涉及到多类别情况时，我们期望模型能够同时捕捉所有类别的分布，使它们可以共享相同的边界，如图 1b 所示。但是，如果我们专注于特定类别，比如图 1b 中的绿色类别，那么来自其他类别的所有样本都应被视为异常，无论它们本身是正常的（即蓝色圆圈）还是异常的（即蓝色三角形）。从这个角度来看，如何准确地建模多类别分布变得至关重要。

学习正常数据分布的一种广泛使用的方法是**基于图像（或特征）重建** [2, 5, 27, 40, 52]，它假设一个训练良好的模型始终产生正常样本，而不考虑输入中的缺陷。通过这种方式，异常样本的重建误差将会很大，使它们与正常样本区分开来。然而，我们发现流行的重建网络在本文中研究的具有挑战性的任务上表现不佳。它们通常会陷入“身份捷径”，表现为**直接返回输入的副本而不考虑其内容**。结果，即使是异常样本也可以用学习的模型很好地恢复，因此变得难以检测。此外，在统一情况下，正常数据的分布更加复杂，“身份捷径”问题被放大。直观地说，要学习一个能够重建所有种类对象的统一模型，就需要模型非常努力地学习联合分布。从这个角度来看，学习一个“身份捷径”似乎是一个更容易的解决方案。

为了解决这个问题，我们精心设计了一个特征重建框架，**防止模型学习捷径**。首先，我们重新审视了神经网络中使用的全连接层、卷积层以及注意力层的公式，并观察到全连接层和卷积层都面临着学习捷径的风险。在多类别设置下，这个缺点会进一步放大，因为正常数据的分布变得更加复杂。相比之下，注意力层免受这种风险的影响，得益于可学习的查询嵌入（见第 3.1 节）。因此，我们提出了一种逐层查询解码器来加强查询嵌入的使用。其次，我们认为全注意力（即每个特征点都与彼此相关）也会导致捷径问题，因为它提供了直接将输入复制到输出的机会。为了避免信息泄漏，我们采用了一个邻居屏蔽的注意力模块，其中一个特征点既与自身又与其邻居无关。第三，受 Bengio 等人的启发 [3]，我们提出了一种**特征抖动**策略，要求模型即使在有噪声的输入下也能恢复源消息。所有这些设计有助于模型摆脱“身份捷径”，如图 2b 所示。

在 MVTec-AD [4] 和 CIFAR-10 [24] 上进行的大量实验表明，我们的方法 UniAD 在统一任务设置下相比现有的替代方案具有足够的优势。例如，在 MVTec-AD 的 15 个类别中学习单个模型时，我们在异常检测和异常定位任务上实现了最先进的性能，将 AUROC 分别从 88.1% 提升到 96.5% 和从 89.5% 提升到 96.8%。

## 相关工作

### 异常检测

1）传统方法扩展了传统的机器学习方法用于单类分类，例如单类支持向量机（OC-SVM）[39]和支持向量数据描述（SVDD）[36, 42]。补丁级嵌入[49]、几何变换[18]和弹性权重整合[34]被用于改进。

2）伪异常将异常检测转化为监督学习，包括分类[26, 33, 46]、图像去噪[53]和超球体分割[28]。然而，这些方法在一定程度上依赖于代理异常与未知的真实异常匹配的程度[13]。3）建模然后比较假设预训练网络能够提取用于异常检测的判别特征[11, 35]。PaDiM [11]和MDND [35]提取预训练特征来建模正态分布，然后利用距离度量来衡量异常。然而，这些方法需要记忆和建模所有正常特征，因此计算成本较高。4）知识蒸馏提出由正常样本上的教师蒸馏的学生只能提取正常特征[6, 13, 38, 45, 46]。最近的工作主要集中在模型集成[6]、特征金字塔[38, 45]和反向蒸馏[13]。基于重建的异常检测。这些方法依赖于一个假设，即仅在正常区域上训练的重建模型在正常区域成功，但在异常区域失败[5, 8, 27, 37, 50]。早期尝试包括自动编码器（AE）[5, 9]、变分自动编码器（VAE）[23, 27]和生成对抗网络（GAN）[2, 31, 37, 52]。然而，这些方法面临一个问题，即模型可能学会将异常也恢复得很好的技巧。因此，研究人员采用不同的策略来解决这个问题，例如添加指导信息（结构化[54]或语义[40, 47]）、记忆机制[19, 21, 30]、迭代机制[12]、图像掩码策略[48]和伪异常[9, 33]。最近，DRAEM [53]首先恢复受伪异常干扰的正常图像以进行表示，然后利用判别网络区分异常，取得了出色的性能。然而，DRAEM [53]在统一情况下已经不再有效。此外，仍然有一个重要的方面尚未得到很好的研究，即什么样的架构是最好的重建模型？在本文中，我们首先比较和分析了三种流行的架构，包括MLP、CNN和Transformer。然后，我们根据Transformer进一步设计了三项改进，构成了我们的UniAD。在异常检测中的Transformer。带有注意机制的Transformer [43]，最初在自然语言处理中提出，已成功应用于计算机视觉[7, 16]。一些尝试试图利用Transformer进行异常检测。InTra [32]采用Transformer逐个恢复所有蒙版补丁来恢复图像。VT-ADL [29]和AnoVit [51]都将Transformer编码器应用于重建图像。然而，这些方法直接使用原始Transformer，并没有弄清楚Transformer为什么会带来改进。相反，我们确认了查询嵌入的有效性以防止捷径，并相应设计了逐层查询解码器。此外，为了避免完全关注的信息泄漏，我们采用了邻居蒙版关注模块。
## 3 Method

### 3.1 Revisiting feature reconstruction for anomaly detection

在图2中，遵循特征重建范式[40, 50]，我们构建了一个MLP、一个CNN和一个带有查询嵌入的Transformer来重建由预训练的主干网络提取的特征。重建误差表示异常可能性。三个网络的架构见附录。指标每10个时期评估一次。需要注意的是，周期性评估是不切实际的，因为在训练期间无法获得异常。如图2a所示，在一段时间的训练后，三个网络的性能严重下降，损失变得极小。我们将这归因于“相同快捷方式”问题，即**正常区域和异常区域都能被很好地恢复，因此无法发现异常**。这种推测通过图2b中的可视化结果得到了验证（附录中有更多结果）。然而，与MLP和CNN相比，Transformer的性能下降要小得多，表明了更轻微的快捷方式问题。这鼓励我们进行如下分析。我们将正常图像中的特征表示为 $x^{+} \in \mathbb{R}^{K\times C}$，其中K是特征数，C是通道维度。为简单起见，批处理维度被省略。类似地，异常图像中的特征被表示为 $x^{-} \in \mathbb{R}^{K\times C}$。重建损失选择为均方误差损失。

### 3.3 实现细节 

【特征提取】我们采用在ImageNet上预训练的固定EfficientNet-b4 [41]作为特征提取器。从阶段1到阶段4选择特征。这里的阶段指的是具有相同大小特征图的块的组合。然后，这些特征被调整到相同的大小，并沿着通道维度连接以形成一个特征图，$\boldsymbol{f}org ∈ R Corg×H×W$。 

【特征重建】首先，特征图forg被分解为H×W个特征标记，然后通过线性投影将Corg减小到较小的通道C。然后这些标记被NME和LQD处理。在注意模块中添加了可学习的位置嵌入以提供空间信息。然后，另一个线性投影被用来从C恢复到Corg。重塑之后，重建的特征图frec ∈ R Corg×H×W最终被获得。 目标函数。我们的模型使用MSE损失进行训练，如下所示： L = 1/(H × W) ||forg − frec||2^2 (4) 用于异常定位的推断。异常定位的结果是一个异常分数图，为每个像素分配一个异常分数。具体来说，异常分数图s被计算为重建差异的L2范数： s = ||forg − frec||2 ∈ R H×W (5) 然后s被上采样到图像大小，使用双线性插值获得定位结果。 用于异常检测的推断。异常检测旨在检测图像是否包含异常区域。我们将异常分数图s转换为图像的异常分数，方法是取s的平均池化的最大值。

### 4.2 Anomaly detection on MVTec-AD

【设置】异常检测旨在检测图像是否包含异常区域。异常检测性能在MVTec-AD [4]上进行评估。图像大小选为224 × 224，并将用于调整特征图的大小设置为14 × 14。EfficientNet-b4 [41]的从第1阶段到第4阶段的特征图被调整大小并连接在一起形成一个272通道的特征图。降低的通道维度设置为256。使用带有权重衰减1 × 10^-4的AdamW优化器[22]。

我们的模型在8个GPU（NVIDIA Tesla V100 16GB）上进行了1000个epoch的训练，批量大小为64。学习率最初为1 × 10^-4，并在800个epoch后降低了0.1。编码器和解码器的层数均为4。邻域大小、抖动尺度和抖动概率分别设置为7×7、20和1。评估使用了5个随机种子。在单独情况和统一情况下，重构模型都是从头开始训练的。

【基线】我们的方法与包括US [6]、PSVDD [49]、PaDiM [11]、CutPaste [26]、MKD [38]和DRAEM [53]在内的基准线进行了比较。在单独情况下，基准线的度量在它们的论文中报告，除了从[53]借来的US的度量。在统一情况下，US、PSVDD、PaDiM、CutPaste、MKD和DRAEM都使用公开可用的实现进行运行。MVTec-AD [4]上的异常检测的定量结果显示在表1中。尽管所有基准线在**单独情况**下都取得了出色的性能，但它们在**统一情况**下的性能急剧下降。之前的SOTA，DRAEM，一种通过伪异常训练的基于重构的方法，遭受了近10%的下降。对于另一个强基线CutPaste，一个伪异常方法，下降幅度高达18.6%。然而，我们的UniAD从单独情况（96.6%）到统一情况（96.5%）几乎没有性能下降。此外，我们击败了最强的竞争者DRAEM，大幅领先（8.4%），展示了我们的优势。

### 4.3 Anomaly localization on MVTec-AD

【设置和基线】异常定位旨在定位异常图像中的异常区域。我们选择MVTec-AD[4]作为基准数据集。设置与第4.2节中的相同。除了第4.2节中的竞争对手外，还包括FCDD[28]，其在单独情况下的度量在其论文中报告。在统一情况下，我们使用FCDD的实现运行FCDD。MVTec-AD[4]上的异常定位的定量结果报告在表2中。与第4.2节类似，从单独情况切换到统一情况，所有竞争对手的性能都显著下降。例如，一个重要的基于蒸馏的基线US的性能下降了12.1%。FCDD，一个伪异常方法，遭受了28.7%的剧烈下降，反映了伪异常不适用于统一情况。然而，我们的UniAD甚至从**单独情况96.6到统一情况96.8**略有改善，证明了我们的UniAD适用于统一情况。此外，我们明显超越了最强基线PaDiM，提高了7.3%。这一显著改进反映了我们模型的有效性。MVTec-AD[4]上的异常定位的定性结果如图6所示。对于全局（图6a）和局部（图6b）结构异常，以及分散的纹理扰动（图6c）和多个纹理划痕（图6d），我们的方法都能成功地将异常重构为其相应的正常样本，然后通过重构差异准确地定位异常区域。附录中提供更多定性结果。