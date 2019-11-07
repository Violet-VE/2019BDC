![Image Name](https://cdn.kesci.com/upload/image/pz5m73c0tf.png)
# 2019高校大数据挑战赛

- 比赛地址：[2019高校大数据挑战赛](https://www.kesci.com/home/competition/5cb80fd312c371002b12355f)

## 赛题分析
1. 赛题任务
	- 比赛以字节跳动旗下的某些 APP（如今日头条）作为实际场景，根据用户输入的关键词（query）和系统推荐的文章标题（title），来**预测某个 query 下 title 的点击率**
	
	>比赛召回部分（候选集）已经确定，因此影响比赛成绩的因素（如文本预处理等）少了很多，所以我们需要做的就是排序部分（文本匹配）的工作
	
2. 比赛数据
	- 脱敏后的 query 和网页文本数据，已经分词为 term 并脱敏，term 之间使用空格分割。
	- 初赛 1 亿数据量，复赛 10 亿数据量（数据量是比较大的，所以需要合理安排线上平台资源的分配）
	- 下面是训练集的数据格式，而测试集不含 label
	![Image Name](https://cdn.kesci.com/upload/image/pz5m26tphi.png?imageView2/0/w/1280/h/1280)

3. 评估指标
	- 评估指标是 qAUC，即每组 query 下的平均 AUC（Area Under Curve）值，其中 AUC_i 为同一个 query_id 下的 AUC，计算得到的 qAUC 越大越好
	![Image Name](https://cdn.kesci.com/upload/image/pz5sfwra5a.png?imageView2/0/w/960/h/960)
  
## 解题思路之机器学习
1. 机器学习
	- 如果将这个题目可以看作：(文本)**点击率预估**的问题，则可以直接使用机器学习或者数据挖掘的方法来求解，即进行 **特征工程+模型优化** 的方法
	- 另外，由于这是一个中文文本相关的赛题，因此特征工程方面有很多是提取文本相关的特征，因此需要了解一些**自然语言处理**和**中文文本处理**的前置知识（比如对句子进行分词的统计、TF-IDF 技术、词的表征方法等，都可以在网上找到相关的资料来学习）
	
2. 特征工程
	- **统计特征**
		query/title 的长度/重复出现的数量，公共词的数量及在 query/title 中的占比和多项式特征，每组 title 长度的最大值最小值和平均值、不同粒度下的 CTR 特征 …
	- **距离特征**
		基于词向量：canberra 距离，曼哈顿距离，欧几里得距离，braycurtis 距离，相关系数 …
		基于离散的词：曼哈顿距离，欧几里得距离，jaccard 距离，levenshtein 距离，levenshtein_jaro 距离，汉明距离 …
	- **相似度特征**
		基于词向量：余弦相似度、levenshtein 相似度 …
		基于离散的词：余弦相似度、共现词的数量及占比 …
	- **语义特征**
		N-Gram 特征、TF-IDF 特征、首词/末词/词的顺序以及出现位置 …

3. 特征选择
	- 在这里我们主要使用了过滤法（filter）和包裹法（wrapper），以及结合 LightGBM 的特征重要度（feature_importance）来进行筛选
	
4. 模型建立
	- 考虑到本赛题的样本量比较大，因此使用 LightGBM 来建立模型，它计算速度比 XGBoost 更快，而且科赛平台的技术人员已经为我们安装了 GPU 版本的 LightGBM，因此可以使用 GPU 进行加速计算（为科赛的小哥哥小姐姐点赞👍）

5. 方法小结
	- 机器学习模型的上限不高，因为它无法完全利用文本的信息，也许做更加精细的关于文本的特征可能会有更好的效果，但是这种特征是比较难以想到或者不存在的

后面在决赛答辩的时候，了解到其他队伍利用了 BM25 相关度特征和一些自定义的 CTR 特征来提升模型结果

推荐一些文本处理工具库：Levenshtein、Textdistance、Difflib、Fuzzywuzzy 等

下图是部分重要特征分析：
	![Image Name](https://cdn.kesci.com/upload/image/pz5y68rdb2.png?imageView2/0/w/960/h/960)
  
## 解题思路之深度学习
1. 深度学习
	- 将这个题目可以看作：**短文本匹配+点击率预估** 的问题。使用深度学习的方法可以更好的捕捉到词/句子/段落/文本之间的关系，关键的一点就是找到对每个 term（本赛题是将每个词划分为term）的表示（比如词向量），再搭建深度神经网络来训练样本，从而得到不同文本之间（query和title）的相关程度
	- 而且本次比赛的数据量比较大，达到了上亿级别，因此也特别适合使用深度学习方法来进行建模
	
	![Image Name](https://cdn.kesci.com/upload/image/pz5x8qckoy.png?imageView2/0/w/960/h/960)

2. 词的表征方法
	- 词的表示方法有很多，主要是要考虑到不同词向量的适用场景和训练效率。fasttext 的训练速度很快、效果也不错，但由于这是一个脱敏的文本，因此其 subword 机制会受到一定影响；而目前很火的 BERT 预训练模型也无法使用，因为需要重新对这些脱敏后的 term 进行训练，而 BERT 的参数量太大，GPU 资源又有限，因此重新训练 BERT 是不可行的 … 
	- 于是我们队伍最终使用了经典的 **word2vec** 来训练词向量（有一些文本是重复的，因此可以考虑去重后加速训练速度），这是一种静态的词向量表示方法，但对于这个比赛来说已经足够了。训练使用是 gensim 库的 word2vec。其中，数据量是训练集的全部数据（使用迭代方式来训练，否则可能会因为内存不够而导致训练失败），词向量维度：300，滑动窗口大小：5，最小词频数：3，训练轮数：5，训练方式：skip-gram
	- 其实，对词的表征方法还可以使用不同词向量来组合表示，比如将 word2vec 和 fasttext 训练出的词向量进行加权求和，或者直接将这两种词向量进行横向拼接，来形成一个维度更大的词向量。这种操作方式，在某些任务上可能会取得更好的效果

3. 深度神经网络
	- 在深度学习方面，我们最先直接使用深度学习模型来进行训练，但发现模型训练时 loss 比较高，无论是调节学习率、更换优化器还是增加数据量等，loss 都无法明显降低。直到后面在讨论区受到一些大佬的指点，发现在**深度神经网络中融合其他特征**，可以使得模型训练的 loss 进一步降低，从而学习到更多的信息
	- 主要尝试了 **伪孪生网络** 和 **ESIM** 模型。（伪）孪生网络的训练速度比较快，效果也还不错。而 ESIM 由于采用了 attention 机制将句子对齐，可以更好的学习到句子之间的语义关系，从而可以很好地进行匹配，效果相比于孪生网络的效果提升不少，但训练的时间开销也更大。然后，我们团队也对 ESIM 做了一些定制化的改进，在提升模型效果的同时减少时间开销
	- 模型训练的一些细节：batch size 为 512（考虑到显存大小），学习率随着 epoch 增加递减，优化器为 Adam，以及对句子使用了动态padding的操作等
	
	![Image Name](https://cdn.kesci.com/upload/image/pz5xneb5tc.png?imageView2/0/w/960/h/960)

4. 模型集成
	- 为了尽可能地利用数据量，我们以每 1 亿或 2 亿数据来训练一个模型，挑选出每1亿或2亿中训练效果最好的模型来进行模型融合，最终使用了约 8 亿数据，使用效果最好的几个模型，并根据线上的结果来进行线性加权融合，以尽可能来降低模型在最终测试集上的泛化误差
	- 下图是深度学习模型效果与数据利用率一种可能的关系
	![Image Name](https://cdn.kesci.com/upload/image/pz5xw9wso4.png?imageView2/0/w/960/h/960)


## 比赛总结
- 需要**注重细节**：对比赛赛题有清晰的认识，将会直接关系到最后的成绩。像这种数据集比较大的比赛，更需要合理安排资源（比如内存、显存资源）和时间（可以使用一些多进程等其他技术来提高设备的利用率），如果有需要可以列一个详细的*计划安排表*，还可以另外建立一个文档来记录团队已完成和待完成的事项
- 比赛过后，发现**机器学习、深度学习的基础知识**以及对一些**框架地熟练使用**对于做比赛是至关重要的，还有对一些**最新前沿技术**的了解与应用很多时候就是提升比赛成绩的关键
- 在这个比赛学习到了很多关于深度学习和自然语言处理的知识，也结交了一些朋友，一个**团队良好的沟通**对于比赛成绩的提高是至关重要的

> *最后感谢我的队友（llhthinker、vhdsih）、实验室的挚友和科赛平台的工作人员* ❣️

