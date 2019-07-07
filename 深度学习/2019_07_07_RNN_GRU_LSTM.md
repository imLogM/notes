# RNN、GRU、LSTM

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles) ，不允许转载~

文章中的数学公式若无法正确显示，请参见：[正确显示数学公式的小技巧](https://segmentfault.com/a/1190000019359797)


## 1. RNN

$$H_t = \phi(W_{xh}X_t + W_{hh}H_{t-1} + b_h)$$

$$O_t = W_{hq}H_t + b_q$$

> 其中，$\phi$ 为激活函数。

> RNN 容易出现"梯度爆炸"和"梯度衰减"，应对梯度衰减的方式是使用 GRU 或 LSTM，应对梯度爆炸的方式是"裁剪梯度"：
> $$min(\frac{\theta}{||g||})g$$
> 其中，$g$ 为求得的梯度，限制梯度的 $L_2$ 范数的值不超过 $\theta$。

## 2. GRU

- 重置门：

$$R_t = \sigma(W_{xr}X_t + W_{hr}H_{t-1} + b_r)$$

- 更新门：

$$Z_t = \sigma(W_{xz}X_t + W_{hz}H_{t-1} + b_z)$$

- 候选隐状态：

$$\tilde{H_t} = tanh[W_{xh}X_t + W_{hh}(R_t \odot H_{t-1}) + b_h]$$

- 隐状态：

$$H_t = Z_t \odot H_{t-1} + (1-Z_t) \odot \tilde{H_t}$$

> 其中，$\sigma$ 为 sigmoid 函数，$\odot$ 表示 element-wise 乘法。

## 3. LSTM

- 输入门:

$$I_t = \sigma(W_{xi}X_t + W_{hi}H_{t-1} + b_i)$$

- 遗忘门：

$$F_t = \sigma(W_{xf}X_t + W_{hf}H_{t-1} + b_f)$$

- 输出门：

$$O_t = \sigma(W_{xo}X_t + W_{ho}H_{t-1} + b_o)$$

- 候选记忆细胞：

$$\tilde{C_t} = tanh(W_{xc}X_t + W_{hc}H_{t-1} + b_c)$$

- 记忆细胞：

$$C_t = F_t \odot C_{t-1} + I_t \odot \tilde{C_t}$$

- 隐状态：

$$H_t = O_t \odot tanh(C_t)$$

## 4. 参考资料

- 动手学深度学习：[https://github.com/d2l-ai/d2l-zh](https://github.com/d2l-ai/d2l-zh)

