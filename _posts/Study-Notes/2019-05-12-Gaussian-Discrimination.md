---
layout:     post
title:      "Gaussian Discrimination"
subtitle:   "Discriminant Funtions for the Normal Density"
date:       2019-05-12 14:06:19
author:     "StephenNG"
header-img: "img/in-post/post-motion-deblur/bg.png"
tags:
    - ML
---

<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

# 1. Intro

The minimum error‐rate classification can be achieved by the discriminant function:

$$
g_{i}(x)=\ln P\left(x | \omega_{i}\right)+\ln P\left(\omega_{i}\right)
$$

对于多维的高斯分布有：

$$
P\left(x | \omega_{i}\right)=\frac{1}{(2 \pi)^{\frac{d}{2}}\left|\Sigma_{i}\right|^{\frac{1}{2}}} \exp \left[-\frac{1}{2}\left(x-\mu_{i}\right)^{T} \Sigma_{i}^{-1}\left(x-\mu_{i}\right)\right]
$$

由此得到：

$$
g_{i}(x)=-\frac{1}{2}\left(x-\mu_{i}\right)^{T} \Sigma_{i}^{-1}\left(x-\mu_{i}\right)-\frac{d}{2} \ln 2 \pi-\frac{1}{2} \ln \left|\Sigma_{i}\right|+\ln P\left(\omega_{i}\right) \tag{0}
$$

其中$$\mu$$是均值，$$\Sigma$$是协方差。

下面分析一下几种特殊情况。

## 1.1 协方差为对角矩阵

去掉常数和与i无关的项得到：

$$
g_{i}(\mathbf{x})=-\frac{\left\|\mathbf{x}-\boldsymbol{\mu}_{i}\right\|^{2}}{2 \sigma^{2}}+\ln P\left(\omega_{i}\right)
$$

其中$$\|\cdot\|$$是欧几里得范数，即

$$
\left\|\mathbf{x}-\boldsymbol{\mu}_{i}\right\|^{2}=\left(\mathbf{x}-\boldsymbol{\mu}_{i}\right)^{t}\left(\mathbf{x}-\boldsymbol{\mu}_{i}\right)
$$

展开得到：

$$
g_{i}(\mathbf{x})=-\frac{1}{2 \sigma^{2}}\left[\mathbf{x}^{t} \mathbf{x}-2 \boldsymbol{\mu}_{i}^{t} \mathbf{x}+\boldsymbol{\mu}_{i}^{t} \boldsymbol{\mu}_{i}\right]+\ln P\left(\omega_{i}\right)
$$

其中二次项对于每一个i是一样的，所以可以忽略，于是得到一个线性判别方程：

$$
g_{i}(\mathbf{x})=\mathbf{w}_{i}^{t} \mathbf{x}+w_{i 0} \tag{1}
$$

其中

$$
\mathbf{w}_{i}=\frac{1}{\sigma^{2}} \boldsymbol{\mu}_{i}
$$

以及

$$
w_{i 0}=\frac{-1}{2 \sigma^{2}} \boldsymbol{\mu}_{i}^{t} \boldsymbol{\mu}_{i}+\ln P\left(\omega_{i}\right)
$$

## 1.2. 协方差相等



## 1.3. 协方差为任意值

$$
g_{i}(\mathbf{x})=\mathbf{x}^{t} \mathbf{W}_{i} \mathbf{x}+\mathbf{w}_{i}^{t} \mathbf{x}+w_{i 0} \tag{3}
$$

其中

$$
\mathbf{W}_{i}=-\frac{1}{2} \boldsymbol{\Sigma}_{i}^{-1}
$$

$$
\mathbf{w}_{i}=\mathbf{\Sigma}_{i}^{-1} \boldsymbol{\mu}_{i}
$$

以及

$$
w_{i 0}=-\frac{1}{2} \boldsymbol{\mu}_{i}^{t} \boldsymbol{\Sigma}_{i}^{-1} \boldsymbol{\mu}_{i}-\frac{1}{2} \ln \left|\boldsymbol{\Sigma}_{i}\right|+\ln P\left(\omega_{i}\right)
$$

* （一个经验式的总结）$$\Sigma$$一般为正定的对角矩阵。以二维空间的两个高斯分布的决策边界为例，假设

    $$
    \Sigma_0^{-1} = 
    \begin{bmatrix} 
        a & 0 \\ 
        0 & b \\ 
    \end{bmatrix} 
    $$

    $$
    \Sigma_1^{-1} = 
    \begin{bmatrix} 
        c & 0 \\ 
        0 & d \\ 
    \end{bmatrix} 
    $$
    
    由于决策面方程为

    $$
    g_{0}(\mathbf{x}) - g_{1}(\mathbf{x}) = 0
    $$

    代入$$(3)$$式后得到形如：

    $$
    (a-c){x}_1^2 + (b-d){x}_2^2 + (\cdots)x_1 + (\cdots)x_2 + (\cdots) = 0
    $$

    的决策面方程。

    由此可以通过改变$$\Sigma$$来控制决策面变成直线、抛物线、双曲线、椭圆等形状。

![](../../../../img/in-post/post-gaussian-discrimination/1.png)
<small class="img-hint">图源《Pattern Classification》p26</small>
