---
linkcolor: red
urlcolor: purple
citecolor: blue
...

# 深度学习

## 卷积神经网络

### [ImageNet Classification with Deep Convolutional Neural Networks](http://papers.nips.cc/paper/4824-imagenet-classification-with-deep-convolutional-neural-networks.pdf)

提出了AlexNet的论文，引用14K。

### [Deep Sparse Rectifier Neural Networks](http://proceedings.mlr.press/v15/glorot11a/glorot11a.pdf)

提出了`ReLU`的论文。

### [Improving neural networks by preventing co-adaptation of feature detectors](https://arxiv.org/pdf/1207.0580.pdf)

提出了`Dropout`的论文。

### [Batch Normalization: Accelerating Deep Network Training by Reducing Internal Covariate Shift](https://arxiv.org/pdf/1502.03167.pdf)

提出了`Batch Normalization`的论文。

### [Densely Connected Convolutional Networks](https://arxiv.org/pdf/1608.06993.pdf)

提出了`DenseNet`的论文。

### [MobileNets: Efficient Convolutional Neural Networks for Mobile Vision Applications](https://arxiv.org/pdf/1704.04861.pdf)

提出了`MobileNet`的论文。

## 神经网络加速

### [Dynamic Network Surgery for Efficient DNNs](https://arxiv.org/pdf/1608.04493.pdf)

我在微软的主要工作就是验证这篇论文以及给出工程上的实现。

提醒：这篇文章中提到的GitHub仓库中的代码实现的有一些问题。

### [Learning both Weights and Connections for Efficient Neural Networks](http://papers.nips.cc/paper/5784-learning-both-weights-and-connections-for-efficient-neural-network.pdf)

是`Dynamic Network Surgery for Efficient DNNs`的主要参考文献。

提出了一种去掉某种阈值以下的参数，后用更小的learning rate来重新训练（训练耗时可能是原来训练的数倍）来补偿去掉参数的做法；可以将网络大小缩小10x左右（对于VGG或LeNet)。

### [EIE: Efficient Inference Engine on Compressed Deep Neural Network](https://arxiv.org/pdf/1602.01528.pdf)

给出了另一种ASIC加速CNN的方案。
其中有目前主流硬件的对比，包括Core i7, Titan X, Terga K1, DaDianNao等等。

### [Dadiannao: A machine-learning supercomputer](https://www.google.co.jp/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&ved=0ahUKEwjGg-339JLVAhUK8RQKHSXtBg4QFggpMAE&url=http%3A%2F%2Fdl.acm.org%2Fcitation.cfm%3Fid%3D2742217&usg=AFQjCNGFRlZOM4tQ9nSVIP_oEwMRKs8Fdw)

提出了一种ASIC来加速CNN网络的方案，给出了部分实现细节。同时是ASPLOS'14和MICRO'14的Best Paper。

这篇论文的前驱是`DianNao`，后继包括`PuDianNao`和`ShiDianNao`。前四篇的[介绍](http://novel.ict.ac.cn/diannao/)。再往后有`Cambricon`和`Cambricon-X`，前者给出了一个指令集，后者着重优化了稀疏矩阵下的运算效率。同时这两篇文章也是现在`寒武纪`的学术基础，它是一个做硬件加速神经网络的创业公司。

以及，这几年（2014-2016）中，ISCA上有不少关于硬件加速NN的文章。

### [Learning Structured Sparsity in Deep Neural Networks](http://papers.nips.cc/paper/6504-learning-structured-sparsity-in-deep-neural-networks.pdf)

这篇文章的[GitHub地址](https://github.com/wenwei202/caffe/tree/scnn)。

这篇文章是结构化稀疏的工作中非常重要的一篇，在它的slide中提到的关于结构化和非结构化的比较值得一看。

核心观点是去掉卷积核等来减少矩阵运算规模，而不是修改矩阵的density。

### [Channel Pruning for Accelerating Very Deep Neural Networks](https://arxiv.org/pdf/1707.06168.pdf)

这篇文章的[GitHub地址](https://github.com/yihui-he/channel-pruning)。

这篇文章我还没有验证。