# CNN 用 ReLU，RNN 用 tanh ?

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles) ，不允许转载~

## 1. CNN 用 ReLU ？

sigmoid 的导数在 [0, 0.25] 范围里；tanh 的导数在 [0, 1] 范围里；ReLU 的导数为 {0, 1}。

如果 CNN 的每一层都使用 sigmoid，小于1的导数的多次连乘引发"梯度消失"。

那为什么不使用 tanh？ tanh 的导数虽然可以达到1，但在边缘仍有"梯度消失"的问题出现。只能说 tanh 在一定程度上缓解了"梯度消失"。

ReLU 的正半区没有"梯度消失"，但负半区存在"梯度消失"，（但也有人认为负半区的存在可以使参数矩阵稀疏，有一定正则化的效果）。

最最重要的是，ReLU 的计算量是远小于 sigmoid 和 tanh 的。

> A four-layer convolutional neural network with ReLUs (solid line) reaches a 25% training error rate on CIFAR-10 six times faster than an equivalent network with tanh neurons (dashed line).

------

ReLU 最先不是针对深度网络发明出来的，所以我们很难从发明者的角度探寻 ReLU 的诞生解决了深度网络的什么问题。事实情况是，当学者们把 ReLU 用在深度网络上发现效果很好，就陆陆续续提出了一些理论来解释为什么 ReLU 的效果好。所以，这些支持 ReLU 的理论是有一些生硬的。

因为 ReLU 不是专门为深度网络研发的，ReLU 移植到深度网络中后，仍有许多问题存在，还有很大的改进空间。

参考：[Krizhevsky等人是怎么想到在CNN里用Dropout和ReLU的？
](https://www.zhihu.com/question/28720729)

## 2. RNN 用 tanh ？

"RNN 用 tanh" 主要是说 GRU 和 LSTM 的隐状态的激活函数一般用的 tanh；它们内部的各种门，因为是 0-1 的输出，所以一般用 sigmoid 函数。

RNN 每个时间步的参数矩阵 W 是相同的，不加激活函数的话相当于 W 的连乘，那么 W 中绝对值小于1的元素会在连乘中快速变为0，绝对值大于1的元素会在连乘中快速变为无穷。所以说，RNN 比 CNN 更容易出现"梯度消失"和"梯度爆炸"。

明白了 RNN "梯度爆炸"的来源，就应该能明白为什么不推荐 ReLU 了。

> At first sight, ReLUs seem inappropriate for RNNs because they can have very large outputs, so they might be expected to be far more likely to explode than units that have bounded values.

明白了 RNN "梯度消失"的来源，就应该能明白为什么 tanh 也不是非常适合。（tanh 会有梯度消失的问题，虽然相比 sigmoid 已经小很多了。）

所以，ReLu 和 tanh 都不是完美的，会有 `RNN + ReLU + 梯度裁剪` 和 `GRU/LSTM + tanh` 两套方案哪个更好的争论。

参考：[RNN中为什么要采用tanh而不是ReLu作为激活函数？](https://www.zhihu.com/question/61265076)
