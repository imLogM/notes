# 使github和segmentfault的markdown支持数学公式

作者：LogM

本文原载于 [https://segmentfault.com/u/logm/articles](https://segmentfault.com/u/logm/articles) ，不允许转载~

## 1. 由来
  
最近在写博客的时候，发现一个问题：

  1. segmentfault不支持markdown行内公式渲染；
  2. github不支持markdown数学公式渲染。

因此，需要想办法正常渲染markdown。否则又要回归繁琐的Github Page了。

## 2. 解决方法

chrome浏览器可以安装MathJax渲染插件解决，比如：

  1. [MathJax Plugin for Github](https://github.com/orsharir/github-mathjax)
  2. [TeX All the Things](https://github.com/emichael/texthings)

这两个我都用过，可以正常渲染。

第一个插件仅支持github，不需要配置。

第二个插件支持所有的网站，我自己测试在segmentfault上会经常抽风，但多刷新几次页面总有一次能刷出来。右键"Tex All the Things"的图标，选择"选项"，可以进行配置。

所以，对于我的博客中带有数学公式的文章，可以有如下几种方式确保数学公式正常渲染：
  
  1. 使用[插件2](https://github.com/emichael/texthings)在segmentfault上看博客，虽然抽风情况比较严重；
  2. 在[我的github](https://github.com/imLogM/notes)上找到对应文章，使用[插件1](https://github.com/orsharir/github-mathjax)查看；
  3. 在[我的github](https://github.com/imLogM/notes)上找到对应文章，点击右上角的"Raw"按钮，把源码复制到markdown阅读器查看。

## 3. 测试

这里提供一组测试，确认是否完美解决了问题。

```
下面参与测试的数学公式的原代码如下：

这是一个行内公式：$P = \frac{C_a^k \cdot C_b^{n-k}}{C_{a+b}^n}$

这是两个单行公式：
$$P = \frac{C_a^k \cdot C_b^{n-k}}{C_{a+b}^n}$$

$$
P = \frac{C_a^k \cdot C_b^{n-k}}{C_{a+b}^n}
$$
```

下面几行是你的显示效果，如果都显示为数学公式，则说明正常渲染：

这是一个行内公式：$P = \frac{C_a^k \cdot C_b^{n-k}}{C_{a+b}^n}$

这是两个单行公式：
$$P = \frac{C_a^k \cdot C_b^{n-k}}{C_{a+b}^n}$$

$$
P = \frac{C_a^k \cdot C_b^{n-k}}{C_{a+b}^n}
$$
