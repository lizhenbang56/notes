- 小样本参数高效微调比上下文学习（ICL）更好、更便宜
- ICL 的主要优点是它使单个模型能够立即执行许多任务，而无需进行微调。
- ICL 会产生大量的计算、内存和存储成本，因为它涉及**每次进行预测时处理所有训练示例**
- 本文提出了 T-Few
- 在 T0 任务上，T-Few 的精度比 GPT-3 175B 的小样本 ICL 高出 6%

# 摘要

少样本上下文学习 (ICL) 通过提供少量训练示例作为输入的一部分，使预先训练的语言模型能够执行以前未见过的任务，而无需任何基于梯度的训练。 ICL 会产生大量的计算、内存和存储成本，因为它涉及**每次进行预测时处理所有训练示例**。 参数高效微调（PEFT）（例如适配器模块、提示调整、稀疏更新方法等）是一种替代范例，其中训练一小组参数以使模型能够执行新任务。 在本文中，我们严格比较了少样本 ICL 和 PEFT，并证明后者提供了更好的精度以及显着降低的计算成本。 在此过程中，我们引入了一种名为 (IA)3 的新 PEFT 方法，该方法通过学习向量来缩放激活，在仅引入相对少量的新参数的情况下获得更强的性能。 我们还提出了一个基于 T0 模型 [1] 的简单方案，称为 T-Few，它可以应用于新任务，而无需针对特定任务进行调整或修改。 我们通过将 T-Few 应用于 RAFT 基准 [2]，验证了 T-Few 在完全未见过的任务上的有效性，首次获得了超人类的性能，并且绝对优于最先进的 6%。 我们实验中使用的所有代码都是公开的。 

# 1 简介 

预训练的语言模型已成为自然语言处理的基石，因为它们可以显着提高感兴趣任务的数据效率，即使用预训练的语言模型进行初始化通常可以使用更少的标记数据产生更好的结果。 历史上常见的方法是在对感兴趣的下游任务执行基于梯度的微调之前，使用预训练模型的参数进行初始化。 虽然微调已经产生了许多最先进的结果 [1]，但它产生的模型专门用于具有一组全新参数值的单个任务，这在微调模型时可能变得不切实际 在许多下游任务上。 [3, 4] 推广的另一种方法是上下文学习 (ICL)，它通过输入提示示例来诱导模型执行下游任务。 Few-shot 提示将一小部分输入目标对转换为（通常）人类可理解的指令和示例 [3, 4]，以及需要预测的单个未标记示例。 值得注意的是，ICL 不需要基于梯度的训练，因此允许单个模型立即执行各种任务。 因此，执行 ICL 仅依赖于模型在预训练期间学习的功能。 这些特征最近引起了人们对 ICL 方法的极大兴趣 [5-10]。

VKQ softmax 密集非线性密集 T0 Susie 喜欢她奶奶的香蕉面包。苏西打电话给她的奶奶，让她送一些。奶奶住得很远。一周过去了，奶奶来看望苏西，这让苏西大吃一惊。这个故事可能的延续是什么？苏西很高兴。苏西很不高兴。 (IA)3 T-Few 中使用的损失 图 1：(IA)3 的图表以及 T-Few 配方中使用的损失项。左图：(IA)3 引入了学习向量 lk、lv 和 lff，它们分别重新缩放（通过逐元素乘法，可视化为 ）注意机制中的键和值以及位置前馈网络中的内部激活。右图：除了标准交叉熵损失 LLM 之外，我们还引入了可能性损失 LUL 和长度归一化损失 LLN，前者可降低错误输出的概率，后者将标准 softmax 交叉熵损失应用于长度归一化对数概率所有输出选择。尽管 ICL 具有实际优势，但它也有几个主要缺点。首先，每次模型进行预测时处理所有提示的输入目标对会产生大量的计算成本。其次，与微调相比，ICL 通常会产生较差的性能 [4]。最后，提示的精确格式（包括措辞 [11] 和示例排序 [12]）可能会对模型的性能产生重大且不可预测的影响，远远超出微调的运行间变化。

最近的研究还表明，即使提供了不正确的标签，ICL 也能表现良好，这引发了关于到底进行了多少学习的问题 [9]。使模型能够以最少的更新执行新任务的另一个范例是参数高效微调（PEFT），其中仅通过更新少量添加或选择的参数来微调预训练模型。最近的方法已经与微调完整模型的性能相匹配，同时仅更新或添加完整模型参数的一小部分（例如 0.01%）[13, 14]。此外，某些 PEFT 方法允许混合任务批次，其中批次中的不同示例进行不同的处理 [14]，使得 PEFT 和 ICL 都适用于多任务模型。虽然 PEFT 的优点解决了微调的一些缺点（与 ICL 相比），但人们相对较少关注 PEFT 方法在可用标记数据很少时是否能正常工作。我们在本文中的主要目标是通过提出一种方法（即模型、PEFT 方法和一组固定的超参数）来缩小这一差距，该方法可以在新的、未见过的任务上获得强大的性能，同时仅更新模型的一小部分参数。具体来说，我们的方法基于 T0 模型 [1]，这是 T5 [15] 的变体，在提示数据集的多任务混合上进行了微调。为了提高分类和多项选择任务的性能，我们添加了似然性 [16, 17] 和基于长度归一化的 [4] 损失项。此外，我们还开发了 (IA)3 ，这是一种将中间激活乘以学习向量的 PEFT 方法。 (IA)3 获得了比全模型微调更强的性能，同时更新的参数数量减少了 10,000 倍。最后，我们展示了在微调之前预训练 (IA)3 参数的好处 [18, 19]。我们的整体方案，我们称之为“T-Few”，其性能明显优于 ICL（即使针对 16 倍大的模型），并且在现实世界的小样本学习基准 RAFT [2] 上首次超越人类，同时要求显着提高更少的计算并允许在推理过程中进行混合任务批次。为了方便使用 T-Few 解决新问题和 PEFT 的未来研究，我们发布了我们的代码。

在以下部分提供 ICL 和 PEFT 的背景之后，我们在第 3 节中讨论 T-Few 的设计。在第 4 节中，我们提出了比较 T-Few 与强 ICL 基线的实验。最后，我们在附录 B 中讨论相关工作，并在第 5 节中进行总结。 

# 2 背景 

在本节中，我们提供 ICL 和 PEFT 的概述，重点是描述进行预测的计算、内存和磁盘存储成本。现实世界的成本取决于实现和硬件，因此我们分别以计算的 FLOP 数和内存和存储的字节数来报告成本。其他相关工作在附录 B 中讨论。 

## 2.1 少样本上下文学习 (ICL) 

ICL [3, 4] 旨在通过输入串联和提示的输入目标示例（称为“镜头”）来诱导模型执行任务）以及未标记的查询示例。接受 Brown 等人的循环 2 个字母任务。 [4] 作为一个例子，4-shot 输入或上下文将是“请将字母打乱成一个单词，并写下该单词：asinoc =赌场，yfrogg = froggy，plesim =简单，iggestb =最大，astedro =”，为此，所需的输出将被“烘焙”。 ICL 通过输入上下文并从模型中采样来引入自回归语言模型来执行此任务。对于分类任务，每个标签都与一个字符串相关联（例如，用于情感分析的“正面”和“负面”），并且通过选择模型分配最高概率的标签字符串来分配标签。对于多项选择任务（例如，在问题的 N 个可能答案之间进行选择），模型的预测类似地是通过确定哪个选项被分配最高概率来确定的。 ICL 的主要优点是它使单个模型能够立即执行许多任务，而无需进行微调。这还支持混合任务批次，其中一批数据中的不同示例通过在输入中使用不同的上下文来对应不同的任务。 ICL 通常也仅使用有限数量的标记示例（称为少样本学习）来执行，从而使其数据高效。

尽管有这些优点，ICL 也存在明显的实际缺点：首先，进行预测的成本要高得多，因为模型需要处理所有上下文中标记的示例。具体来说，忽略 Transformer 语言模型中自注意力操作的二次复杂度（与模型其余部分的成本相比通常很小 [20]），处理 k-shot ICL 的 k 个训练示例会增加计算成本与单独处理未标记的示例相比，大约 k + 1 倍。内存成本类似地与 k 呈近似线性关系，但在推理过程中，内存成本通常主要由存储模型参数决定。另外，存储给定任务的上下文示例需要少量的磁盘存储空间。例如，如果任务的提示输入和目标长度为 512 个令牌，则存储 32 个示例将需要大约 66 KB 的磁盘存储空间（32 个示例 × 512 个令牌 × 32 位）。除了上述成本之外，ICL 还表现出不直观的行为。赵等人。 [12]表明上下文中示例的排序严重影响模型的预测。敏等人。 [9]表明，即使上下文示例的标签被交换（即变得不正确），ICL 仍然可以表现良好，这引发了关于 ICL 是否真的从标记示例中“学习”的问题。已经提出了各种方法来缓解这些问题。降低计算成本的一种方法是缓存上下文示例的键和值向量。这是可能的，因为仅解码器的 Transformer 语言模型具有因果掩码模式，因此模型对上下文的激活并不不依赖于未标记的示例。在极端情况下，每个上下文示例包含 512 个令牌的 32 个镜头 ICL 将为 GPT-3 模型生成超过 144 GB 的缓存键和值向量（32 个示例 × 512 个令牌 × 96 层 × 12288 dmodel × 32 位）每个用于键和值向量）。另外，Min 等人。 [21]提出了集成 ICL，其中不是使用连接 k 个训练示例的输出概率，而是将每个训练示例上的模型的输出概率（即 k 个示例中每个示例的 1-shot ICL）相乘。这将非参数内存成本降低了 k/2 倍，但将计算成本增加了 2 倍。在任务性能方面，Min 等人。 [21] 发现集成 ICL 优于标准串联变体。 

## 2.2 参数高效的微调

虽然标准微调更新了预训练模型的所有参数，但已经证明可以更新或添加相对少量的参数。早期的方法建议添加适配器[22-24]，它们是插入固定预训练模型各层之间的小型可训练前馈网络。从那时起，人们提出了各种复杂的 PEFT 方法，包括选择参数的稀疏子集来训练的方法 [25, 26]、产生低秩更新 [13]、在较低维子空间中执行优化 [27]、添加使用超复杂乘法的低阶适配器[28]等等。相关地，提示调整[14]和前缀调整[29]将学习到的连续嵌入连接到模型的输入或激活以诱导其执行任务；这可以看作是 PEFT 方法[30]。最先进的 PEFT 方法可以与微调所有模型参数的性能相匹配，同时仅更新模型参数的一小部分（例如 0.01%）。 PEFT 大大降低了训练和保存模型的内存和存储要求。此外，某些 PEFT 方法直接允许混合任务批次 - 例如，提示 3 调整只需将不同的提示嵌入连接到批次中的每个示例即可使单个模型执行许多任务 [14]。另一方面，重新参数化模型的 PEFT 方法（例如 [27, 13]）对于混合任务批次而言成本高昂或繁重。另外，不同的 PEFT 方法会不同程度地增加执行推理所需的计算和内存。例如，适配器有效地向模型添加了额外的（小）层，导致计算成本和内存的小幅但不可忽略的增加。 PEFT 产生的额外成本是微调本身的成本，该成本必须执行一次，然后在模型用于推理时摊销。然而，我们将证明，在考虑微调和推理时，PEFT 的计算效率可以显着提高，同时实现比 ICL 更好的精度。 

# 3 设计 T-Few 配方 

鉴于 PEFT 允许模型以相对较小的存储要求和计算成本适应新任务，我们认为 PEFT 是 ICL 的一个有前途的替代方案。因此，我们的目标是开发一种方法，使模型能够在具有有限标记示例的新任务上获得高精度，同时允许在推理过程中进行混合任务批次，并产生最小的计算和存储成本。通过配方，我们指的是特定的模型和超参数设置，可以在任何新任务上提供强大的性能，而无需手动调整或按任务调整。通过这种方式，我们可以确保我们的方法在少数镜头设置中是一个现实的选择，其中有限的标记数据可用于评估 [31, 32]。 

## 3.1 模型和数据集 

作为第一步，我们必须选择一个预训练的模型。理想情况下，模型在对有限数量的标记示例进行微调后，应该在新任务上获得高性能。在将 PEFT 方法应用于不同预训练模型的初步实验中，我们在 T0 [1] 下获得了最佳性能。 T0 基于 T5 [15]，这是一种编码器-解码器 Transformer 模型 [33]，它是通过掩码语言建模目标 [34] 在大量未标记文本数据的语料库上进行预训练的。 T0 是通过在数据集的多任务混合上微调 T5 来创建的，以实现零样本泛化，即无需任何额外的基于梯度的训练即可执行任务的能力。用于训练 T0 的数据集中的示例是通过应用公共提示池 (P3 [35]) 中的提示模板来提示的，该模板将每个数据集中的每个示例转换为提示的文本到文本格式，其中每个标签对应于不同的字符串。为了简洁起见，我们省略了对T0和T5的详细描述；有兴趣的读者可以参考 Sanh 等人。 [1] 和拉斐尔等人。 [15]。 T0 发布了 30 亿和 110 亿参数变体，分别称为“T0-3B”和简称“T0”。在本节中（我们的目标是通过大量实验设计 T-Few 配方），我们使用 T0-3B 来降低计算成本。对于所有模型和实验，我们使用 Hugging Face Transformers [36]。虽然 T0 是为零样本泛化而设计的，但我们将证明它在仅使用几个标记示例进行微调后也能获得强大的性能。为了测试 T0 的泛化能力，Sanh 等人。 [1] 选择了一组任务（和相应的数据集）来支持多任务训练混合物 - 具体来说，句子完成（COPA [37]、H-SWAG [38] 和 Story Cloze [39] 数据集）、自然语言推理（ANLI [40]、CB [41] 和 RTE [42]）、共指消解（WSC [43] 和 Winogrande [44]）和词义消歧（WiC [45]）。然后可以通过测量这些保留数据集的性能来直接评估泛化能力。稍后我们还将在第 4.3 节的 RAFT 基准 [2] 中测试 T-Few 的能力，这是一组未见过的“现实世界”小样本任务，没有验证集和保留测试集。 ANLI、WiC、WSC 根据知识共享许可证获得许可。 Winogrande 根据 Apache 许可证获得许可。 COPA 受 BSD-2 条款许可。我们找不到 RTE 和 CB 的许可证，但它们是 SuperGLUE 的一部分，其中提到数据集允许在研究环境中使用。为了便于比较，我们对每个数据集使用与 Brown 等人相同数量的小样本训练示例。 [4]，从 20 到 70 不等。不幸的是，Brown 等人使用的少镜头数据集子集。 [4]尚未公开披露。为了进行更稳健的比较，我们通过对具有不同种子的子集进行采样并报告中位数和四分位数范围来构建五个小样本数据集。我们使用 P3 Bach 等人的提示模板提示每个数据集中的示例。 [35]，在每个步骤的每个示例中使用随机采样的提示模板。除非另有说明，我们将模型训练 1K 步骤，批量大小为 8，并在训练结束时报告性能。为了进行评估，我们使用“排名分类”，其中模型对所有可能的标签字符串的对数概率进行排名，如果排名最高的选择是第 4 个正确答案，则模型的预测被认为是正确的。排名分类评估与分类任务和多项选择任务兼容。由于模型性能可能会根据所使用的提示模板而显着变化，因此我们报告了 P3 中所有提示模板以及每个数据集的少数样本数据子集的准确度中值。对于所有数据集，当测试标签不公开时（例如 SuperGLUE 数据集），我们会报告测试集或验证集的准确性。在正文中，我们报告了上述九个数据集的准确率中值。附录中提供了每个数据集的详细结果。 

## 3.2 似然性训练和长度归一化 

在研究 PEFT 方法之前，我们首先探索两个额外的损失项来提高语言模型的小样本微调的性能。语言模型通常使用交叉熵损失 LLM = − 1 TP t log p(yt|x, y<t) 进行训练，其中模型经过训练以增加正确目标序列 y = (y1, y2, . . . , yT ) 给定输入序列 x。为了进行评估，我们使用排名分类（第 3.1 节中描述），它取决于模型分配给正确选择的概率以及模型分配给错误选择的概率。为了在训练期间考虑到这一点，我们考虑添加一个似然性损失 [16, 17]： LUL = − PN n=1 PT (n) t=1 log(1 − p( ˆ y (n) i |x, y ˆ (n) <t )) PN n=1 T(n) (1) 这会阻止模型从不正确的目标序列中预测标记，其中 ˆy (n) = (ˆy1, yˆ2, . . , yˆT(n) )是 N 个错误目标序列中的第 n 个。我们假设添加 LUL 将改善排名分类的结果，因为模型将被训练为错误选择分配较低的概率，从而提高正确选择排名最高的机会。给定训练示例的可能目标序列可能具有显着不同的长度，尤其是在多项选择任务中。因此，基于概率对每个选择进行排名可以“支持”较短的选择，因为模型为每个标记分配的概率 ≤ 1。为了纠正这个问题，我们考虑在执行排名分类时使用长度归一化，它将模型在每个可能的答案选择上的得分除以选择中的标记数量（如 GPT-3 [4] 中使用的）。当在评估期间使用长度归一化时，我们在训练期间引入一个额外的损失项，它更接近地反映长度归一化的评估。首先，我们计算给定输出序列 β(x, y) = 1 T PT t=1 log p(yt|x, y<t) 的长度归一化对数概率。然后，我们通过最小化 softmax 交叉熵损失来最大化正确答案选择的长度归一化对数概率： LLN = − log exp( β (x, y)) exp( β (x, y)) + PN n= 1 exp( β (x, ˆ y(n))) (2) 当使用 LLM、LUL 和 LLN 训练模型时，我们只需将它们相加即可。这避免了引入任何超参数，这些超参数在小样本设置中调整时会出现问题（其中实际大小的验证集必然很小[31, 32]）。我们在附录 C 中报告了在所有数据集上使用和不使用长度归一化的所有 T0-3B 参数的微调结果。我们发现添加 LLN 将准确性从 60.7% 提高到 62.71%，并且包含 LUL 和 LLN 提供了进一步的改进至 63.3%。由于这些损失项无需引入任何额外的超参数即可提高性能，因此我们将它们包含在我们的配方中，并在以下所有实验中使用它们。 

## 3.3 使用 $\texttt{(IA)}^3$ 进行参数高效微调 

为了与少样本 ICL 相比，我们需要一种具有以下属性的 PEFT 方法：
- 首先，它必须添加或更新尽可能少的参数，以避免产生存储空间和内存成本。
- 其次，在新任务的几次训练后，它应该达到很高的准确性。
- 最后，它必须允许混合任务批次，因为这是 ICL 的功能。

为了轻松启用混合任务批次，PEFT 方法理想情况下不应修改模型本身。否则，批次中的每个示例实际上都需要由不同的模型或计算图进行处理。直接修改模型激活的方法提供了更方便的替代方案，因为这可以根据示例对应的任务对批次中的每个示例独立且廉价地完成。提示调整和前缀调整方法 [14, 29] 通过将学习的向量连接到激活或嵌入序列来工作，因此是允许混合任务批次的激活修改 PEFT 方法的示例。然而，正如我们稍后将讨论的 5，我们无法通过即时调整获得合理的精度，并且发现性能更高的 PEFT 方法不允许混合任务批次。因此，我们开发了一种新的 PEFT 方法来满足我们的需求。作为替代方案，我们探索了模型激活与学习向量的逐元素相乘（即重新缩放）。具体来说，我们考虑 l x 形式的适应，其中 l ∈ R d 是学习的特定于任务的向量，表示逐元素乘法，x ∈ RT ×d 是长度为 T 的激活序列。我们使用“广播符号”[46]，使得 l x 的第 (i, j) 项  为 ljxi,j 。在初步实验中，我们发现没有必要为 Transformer 模型中的每组激活引入学习的缩放向量。相反，我们发现在自注意力和编码器-解码器注意力机制中的键和值以及位置前馈网络的中间激活上引入重新缩放向量就足够了。具体来说，使用 Vaswani 等人的符号。 [33]，我们引入了三个学习向量 lk ∈ R dk 、 lv ∈ R dv 和 lff ∈ R dff ，它们被引入到注意力机制中：softmax Q(lk  KT ) √ dk (lv  V ) 并且在位置-wise 前馈网络为 (lff  γ(W1x))W2，其中 γ 是前馈网络非线性。我们在每个 Transformer 层块中引入一组单独的 lk、lv 和 lff 向量。这为 L 层块 Transformer 编码器和 L(2dk + 2dv + dff) 总共添加了 L(dk + dv + dff) 个新参数（因子 2 说明了自注意力和编码器的存在） L 层块解码器的解码器注意力）。 lk、lv 和 lff 都用 1 初始化，这样模型计算的整体函数在它们相加时不会改变。我们将我们的方法称为 (IA)3 ，它代表“通过抑制和放大内部激活注入适配器”。 (IA)3 使混合任务批次成为可能，因为批次中的每个激活序列都可以单独且廉价地乘以其关联的学习任务向量。我们还注意到，如果模型仅用于单个任务，则 (IA)3 引入的修改也可以永久应用于权重矩阵，因此不需要元素乘法并且模型的架构保持不变。这是可能的，因为 (IA)3 中执行的逐元素乘法总是与矩阵乘法同时发生，并且 l  
W x = (l  W)x。在这种情况下，与原始模型相比，我们的方法不会产生额外的计算成本。为了验证 (IA)3 ，我们将其与在保留任务的少镜头数据集上微调 T0-3B 的设置中的各种现有适应方法进行比较。具体来说，我们与 9 种强大的 PEFT 方法进行比较：BitFit [47]，它仅更新偏差参数；适配器[23]在自注意力和位置前馈网络之后引入特定于任务的层； Compacter 和 Compacter++ [28] 通过使用低秩矩阵和超复数乘法改进了适配器；提示调整 [14]，它学习与模型输入连接的特定于任务的提示嵌入； FISH Mask [26]，根据近似 Fisher 信息选择要更新的参数子集； Intrinsic SAID [27]，在低维子空间中执行优化；前缀调整[29]，它学习与模型的激活连接的特定于任务的向量； LoRA [13] 为参数矩阵分配低秩更新。此外，我们还包括全模型微调和仅更新层归一化参数的基线。对于允许更改参数效率的某些方法，我们报告了不同预算的结果：FISH Mask 的稀疏度为 0.2% 和 0.02%，用于提示调整的 10 和 100 个学习提示向量，以及 Intrinsic SAID 的 20,000 或 500,000 维子空间。结果如图所示。 2，每个数据集的详细结果见附录 D。我们发现 (IA)3 是唯一比全模型微调基线获得更高准确度的方法。虽然其他 PEFT 方法（例如 Intrinsic SAID 和提示调整）更新或引入的参数较少，但 (IA)3 的性能要好得多。我们的结果和设置与我们所比较的 PEFT 方法的一些过去的工作不同。马哈巴迪等人。 [28] 报告指出，Compacter 和 Compacter++ 的性能优于全模型微调，包括在少样本设置中。莱斯特等人。 [14]发现即时调优可以匹配全模型微调，并且在后续的工作中Wei等人。 [48]发现，当应用于少样本设置中的多任务微调模型时，即时调整表现良好。在这两种情况下，我们都尝试了各种超参数选择，以尝试匹配过去的结果。我们假设分歧来自于我们使用不同的模型和不同的数据集。特别是对于即时调整，我们注意到验证集性能在训练过程中可能会大幅波动，暗示可能存在优化问题。 6 0.001% 0.01% 0.1% 更新的参数百分比 50 55 60 65 准确度 所有参数 (IA)³ LoRA BitFit Layer Norm Compacter Compacter++ Prompt Tuning Prefix Tuning Adapter FISH Mask Intrinsic SAID 图 2：应用 LUL 和 LLN 时 PEFT 方法的准确度到T0-3B。具有可变参数预算的方法用更大和更小的标记来表示更多或更少的参数。 10 12 10 13 10 14 10 15 每个示例的 FLOP 50 55 60 65 70 准确度 T-Few T0 T5+LM GPT-3 6.7B GPT-3 13B GPT-3 175B 图 3：不同少样本学习方法的准确度。 T-Few 将 (IA)3 用于 T0 的 PEFT 方法，T0 使用零样本学习，T5+LM 和 GPT-3 变体使用少样本 ICL。 x 轴对应于推理成本；详细信息请参见第 4.2 节。 3.4 预训练 (IA)3 在最近的工作中，Gu 等人。 [18]，Vu 等人。 [19]表明，在对下游小样本任务进行微调时，在提示调整中预训练提示嵌入可以提高性能。对于预训练，Gu 等人。 [18] 使用一套应用于未标记文本数据的自我监督任务，Vu 等人。 [19]考虑使用来自单独任务或多任务混合的嵌入。我们关注 Vu 等人。 [19] 并简单地在用于训练 T0 的相同多任务混合物上预训练 (IA)3 引入的新参数。在对每个下游数据集上的 (IA)3 参数进行微调之前，我们预训练了 100,000 个步骤，批量大小为16。附录 E 详细介绍了有和没有预训练 (IA)3 的准确度的完整比较。我们发现预训练将微调准确度从 64.6 提高到 65.8，因此将其添加到我们的配方中。 3.5 组合成分 总之，T-Few 配方定义如下：我们使用 T0 模型作为骨干。我们添加 (IA)3 用于下游任务适应，并使用从 T0 的相同多任务混合上的预训练 (IA)3 初始化的参数。作为目标，我们使用标准语言建模损失 LLM、错误选择的可能性损失 LUL 和长度归一化损失 LLN 的总和。我们使用 Adafactor 优化器 [49] 训练 1,000 个步骤，批量大小为 8 个序列，学习率为 3e − 3，线性衰减计划具有 60 步预热。我们在训练和推理过程中将提示模板应用于下游数据集，以将每个示例转换为指导性的文本到文本格式。重要的是，我们以完全相同的方式将此配方应用于每个下游数据集，而无需对每个数据集的超参数进行调整或修改。这使得该配方成为小样本学习设置的现实选择，其中验证集根据定义很小[31, 32]。 

# 4 使用 T-Few 超越 ICL 

在 T0-3B 上设计并建立了 T-Few 配方后，我们现在将其应用于 T0（具有 110 亿个参数），并将性能与强大的少样本 ICL 基线进行比较。从现在开始，我们在所有任务中使用完全相同的配方和超参数。 

## 4.1 T0 任务的性能 

首先，我们在 T0 训练混合物中保留的数据集上评估 T-Few。我们将零样本学习与 T0 [1] 进行比较（因为我们发现少样本 ICL 的性能比 T0 的零样本学习差，请参阅附录 F）；带有 T5+LM 的少样本 ICL [14]（T0 所基于的下一步预测语言模型）；以及具有 6.7、13 和 1750 亿参数变体的 GPT-3 的少样本 ICL。有关这些基线的更多详细信息，请参阅附录 F。

保留的 T0 数据集（第 3.1 节中描述）的准确性如表 1 和图 2 所示。 3，附录 F 中报告了每个数据集的结果。我们发现 T-Few 大大优于所有其他方法。值得注意的是，T-Few 的精度比 GPT-3 175B 的小样本 ICL 高出 6%，尽管其尺寸缩小了约 16 倍，并且其性能比更小的 GPT-3 变体还要大。T-Few 还获得了比 T0 的零样本学习和 T5+LM 的少样本 ICL 更高的准确度。方法推理 FLOP 训练 FLOP 磁盘空间 Acc。 T-Few 1.1e12 2.7e16 4.2 MB 72.4% T0 [1] 1.1e12 0 0 B 66.9% T5+LM [14] 4.5e13 0 16 kB 49.6% GPT-3 6.7B [4] 5.4e13 0 16 kB 57.2% GPT-3 13B [4] 1.0e14 0 16 kB 60.3% GPT-3 175B [4] 1.4e15 0 16 kB 66.6% 表 1：不同少样本学习方法和模型的保留 T0 任务的准确性和计算成本。 T-Few 获得了最高精度，且计算成本比使用 GPT-3 175B 的 ICL 低 1,000 倍。使用 T-Few 进行微调的成本大约与使用 GPT-3 175B 的 20 个示例中的 ICL 一样多。方法附件T-Few 75.8% 人类基线 [2] 73.5% PET [50] 69.6% SetFit [51] 66.9% GPT-3 [4] 62.7% 表 2：截至撰写本文时 RAFT 的 Top-5 最佳方法。 T-Few 是第一个超越人类基线的方法，其准确度比次佳方法高出 6% 以上。 

## 4.2 比较计算成本 

确定 T-Few 明显优于基于 ICL 的模型后，我们现在比较每种小样本学习方法的相对成本。为简单起见，我们使用 Kaplan 等人引入的基于 Transformer 的语言模型的 FLOPs-per-token 估计。 [20]。具体来说，我们估计具有 N 个参数的仅解码器 Transformer（例如 GPT 系列）使用每个令牌 2N FLOP 进行推理，每个令牌使用 6N FLOP 进行训练。像 T0 和 T5 这样的编码器-解码器模型（其中编码器和解码器具有相同的层数和层大小）仅使用编码器或解码器处理每个令牌（每个令牌大约具有完整模型的一半参数），因此 FLOPs每个令牌的估计值减半为每个令牌的 N 和 3N FLOP，用于推理和训练。我们注意到，FLOP 并不是对现实世界计算成本的直接衡量，因为延迟、功耗和其他成本可能会因硬件和其他因素而有很大差异[52]。然而，我们关注的是 FLOP，因为它是一个独立于硬件的指标，与现实世界的成本密切相关，用于运行不同方法的硬件设置我们认为不同方法之间可能会有很大差异。我们在表 1 中总结了成本并在下面进行讨论。对于所有估计，我们使用我们考虑的数据集中的中位数镜头数 (41)。排名评估和我们的可能性损失都需要处理每个可能的输出选择，以获得对未标记示例的预测。对于我们考虑的数据集，输入和所有可能目标的中值组合标记化序列长度为 103。对于为少镜头 ICL 处理的上下文中的示例，只需要正确的目标，产生的中值序列长度为 98。假设键和值向量被缓存，因此使用 ICL 处理单个示例涉及处理 41 × 98 + 103代币。表 1 中提供了我们的成本估算摘要。推理成本。除了提高准确性之外，避免几次 ICL 的主要优点是大大降低推理成本。使用 T-Few 处理单个输入和所有目标选择需要 11e9×103 = 1.1e12 FLOPs，而使用 GPT-3 175B 的少样本 ICL 需要 2×175e9× (41 × 98 + 103) = 1.4e15 FLOPs – 超过多了3个数量级。使用较小的 GPT-3 变体的 ICL 的推理成本也显着高于 T-Few 的推理成本。正如 2.1 节中所讨论的，当要重用同一组上下文中的示例时缓存键和值向量可以减少 ICL 的计算成本。然而，这只会导致大约 41 倍的减少，这不足以使任何 GPT-3 ICL 成本低至 T-Few。培训费用。由于 T-Few 是唯一涉及更新参数的方法，因此它也是唯一会产生训练成本的方法。以 8 个长度为 103 的序列批量大小训练 1,000 个步骤的 110 亿参数编码器-解码器模型需要大约 3 × 11e9 × 1, 000 × 8 8 × 103 = 2.7e16 FLOP。虽然并不无关紧要，但这仅比使用 GPT-3 175B 处理带有少量 ICL 的单个示例所需的 FLOP 大约大 20 倍。换句话说，训练 T-Few 的成本与使用 GPT-3 175B 用少样本 ICL 处理 20 个样本的成本一样多。我们还发现，在单个 NVIDIA A100 GPU 上，使用 T-Few 在单个数据集上微调 T0 仅需要大约半小时。截至撰写本文时，如果使用 Microsoft Azure，这将花费约 2 美元。2 存储成本。 T-Few 的存储成本也最大。当存储为单精度浮点数时，(IA)3 添加的参数会占用 4.2 MB 的磁盘空间。相比之下，ICL 方法仅需要存储标记化的上下文示例（通常存储为 32 位整数），从而导致较小的 41 × 98 × 32 位 = 16 kB 磁盘空间需求。然而，我们注意到，4.2 MB 与模型检查点本身的磁盘大小相比显得相形见绌——存储 10,000 个任务的 (IA)3 适应向量将占用与 T0 检查点 (41.5 GB) 一样多的空间。内存使用情况。在推理过程中，主要内存成本是由模型参数产生的。唯一小于T0（T-Few使用）的型号是GPT-3 6.7B；否则，T-Few 在推理过程中会产生较低的内存成本。由于需要缓存反向传播的中间激活和 Adafactor 中的梯度累加器变量，因此在训练 T-Few 时会产生额外的内存成本。然而，如上所述，可以在单个 80GB A100 GPU 上使用 T-Few 配方。 

## 4.3 现实世界小样本任务（RAFT）的性能 

到目前为止，我们已经评估了一组数据集的性能，这些数据集并未明确设计用于对小样本学习进行基准测试。为了更好地评估 T-Few 在现实世界中的性能，我们在 RAFT 基准 [2] 上评估了我们的方法。 RAFT 由 11 个“具有经济价值”的任务组成，旨在反映现实世界的应用程序。重要的是，每个 RAFT 数据集只有 50 个没有验证集的训练样本和一个没有公共标签的（更大的）测试集，因此不可能通过调整不切实际的大验证集或查看测试集来“作弊” [32, 31]。我们使用与数据集一起发布的标准提示将 T-Few 应用于 RAFT。当前 top-5 方法的准确度如表 2 所示，附录 H 中提供了更多详细信息。T-Few 达到了 75.8% 的最先进准确度，并且优于人类基线（73.5% 准确度）第一次。次优模型（来自 Schick 和 Schütze [50]）的准确率降低了 6%，而 GPT-3 175B 仅达到 62.7%。这些结果验证了 T-Few 可以很容易地按原样应用于新颖的现实任务，以获得强大的性能。 

## 4.4 消融实验

鉴于我们的T-Few 设计实验是在T0-3B 上进行的，我们在T0 上对T-Few 的一些成分进行了消融。详细结果显示在附录 G 中。虽然添加每种成分的收益并不总是显着提高每个单独数据集的准确性，但每种成分始终如一地提高整个数据集的平均性能：删除预训练会使准确性降低 1.6%，消除不太可能的训练长度归一化使准确度降低了 4.1%，而删除预训练和额外损失项则使准确度降低了 2.5%。 

# 5 结论 

我们引入了 T-Few，一种参数高效的少样本学习方法，它以更低的计算成本获得比少样本 ICL 更高的准确度。 T-Few 使用 (IA)3 ，这是一种新的 PEFT 方法，可通过学习向量重新调整内部激活。使用 (IA)3 比微调整个模型同时仅引入少量附加参数能产生更好的性能。 T-Few 还使用了两个额外的损失项，鼓励模型输出错误选择的较低概率，并考虑不同答案选择的长度。当将 T-Few 按原样（没有特定于任务的超参数调整或其他更改）应用于 RAFT 基准时，我们第一次获得了超人的性能，并且大大超过了之前提交的性能。通过计算成本的详细表征，我们发现 T-Few 在推理过程中使用的 FLOP 比使用 GPT-3 的少样本 ICL 少了 1,000 倍以上，并且在单个 NVIDIA A100 GPU 上训练只需要 30 分钟。由于我们所有的实验都是关于分类任务的，因此我们有兴趣在未来的工作中将 T-Few 应用于生成任务，例如摘要和问答。我们希望我们的结果能够为如何最好地使用大型语言模型执行小样本学习提供新的视角。