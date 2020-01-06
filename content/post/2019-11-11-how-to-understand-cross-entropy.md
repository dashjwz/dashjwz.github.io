---
title: How To Understand Cross Entropy
author: dashjay
date: '2019-11-11'
slug: how-to-understand-cross-entropy
categories:
  - math
  - data science
tags: []
---


## 机缘巧合看到交叉熵之后觉得非常神奇

反复看代码没搞懂在干什么，因此找了本机器学习的书希望能够理解这个玩意儿

> 除了均方误差之外，交叉熵误差也被用于损失函数，公式如下

$$
E = - \sum_k{t_k log{y_k}}
$$

$$这里面，log表示以e为底的自然对数log_e,y_k是神经网络的输出,t_k是正确标签。并且，t_k是使用如下one-hot的方式表示。$$

`[0,0,0,1,0,0,0,0]`

如果计算机判断这个是第四个的，入上方数组表示，并且又输出了一个概率如下
`[0.1,0.1,0.4,0.6,0.1,0.1....]`

$$那么计算机就会使用log_{0.6}的到对应的交叉熵是0.51, 如果对应概率是0.1，则对应的交叉熵是2.3左右$$

```

def crosss_entropy_error(y, t):

    delta = 1e-7
    return -np.sum(t*np.log(y + delta))


```
用以上的方式即可完成交叉熵的编写，用sum函数的因为输出的是向量，但是其他列都是0.只有一列有数据。
