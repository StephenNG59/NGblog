---
layout:     post
title:      "Mesh Denoising via Kernel Low-rank Recovery"
subtitle:   "《Mesh Denoising Guided by Patch Normal Co-filtering via Kernel Low-rank Recovery》"
date:       2019-05-13 22:01:40
author:     "StephenNG"
header-img: "img/in-post/post-motion-deblur/bg.png"
tags:
    - CG
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

# 1. Intro

目标：在去除噪音的同时保持mesh内在的特征。

# 2. Patch Normal Co-Filtering

## 2.1. Nonlocal Patch Matrix Construction

对于普通的三角形网格，一个顶点的离散平均曲率(discrete mean curvature)只和它的1-ring临近网格(一环)有关。

如果在一环内发生扭曲，那么可能是噪声或者微小的feature；如果在一环至二环的方向发生扭曲，那么和中等大小的feature有关。

我们需要在全局中找到和某个patch相似的其他k个patch，来进行denoise。传统方法是通过法向量的近似度来寻找相似的patch，但法向量是表面的一阶导数，容易受噪声影响，所以我们采用了**Normal Voting Tensor**：

$$
T_{f}=\sum_{f_{i} \in P(f)} \mu_{i} \mathbf{n}_{i} \mathbf{n}_{i}^{T}
$$

其中$$\mu_{i}$$是权重，通过某种方法计算出来（《Feature detection of triangular meshes based on tensor voting theory》）。

把它分解为特征向量的线性组合：

$$
T_{f}=\lambda_{1} \mathbf{e}_{1} \mathbf{e}_{1}^{T}+\lambda_{2} \mathbf{e}_{2} \mathbf{e}_{2}^{T}+\lambda_{3} \mathbf{e}_{3} \mathbf{e}_{3}^{T}
$$

其中$$\lambda_{1} \geq \lambda_{2} \geq \lambda_{3} \geq 0$$是它的特征值，通过

$$
\rho_{i, j}=\left\|\lambda_{1, i}-\lambda_{1, j}\right\|_{2}^{2}+\left\|\lambda_{2, i}-\lambda_{2, j}\right\|_{2}^{2}+\left\|\lambda_{3, i}-\lambda_{3, j}\right\|_{2}^{2}
$$

来计算两个patch之间的相似度。以此得到和每个网格相似的patch group。

把每个patch中各个法向量组合后得到$$\mathbf{P}_{v}=\left[\mathbf{n}_{1}, \mathbf{n}_{2}, \cdots, \mathbf{n}_{i}\right]^{T}$$，对于target patch$$\mathbf{P}_{t}$$和它的k个相似的patch$$\mathbf{P}_{i}(i=1,\dot,k)$$，可以构建出对于target patch的一个patch matrix：

$$
\mathbf{X}=\left[\mathbf{P}_{v_{t}}, \mathbf{P}_{v_{1}}, \mathbf{P}_{v_{2}}, \cdots, \mathbf{P}_{v_{k}}\right]
$$

因为这(k+1)个patch之间是很相似的，所以这个patch matrix会是低维的，后面将介绍我们采用的方法来解决low-rank recovery问题。

## 2.2. Kernel Low-rank Recovery

### 2.2.1 模型

得到patch matrix之后，通过下式来恢复出ground truth：

$$
\min _{\mathbf{A}} \operatorname{rank}(\mathbf{A})+\lambda\|\mathbf{X}-\mathbf{A}\|_{F}^{2}
$$

其中第一项限制$$\mathbf{A}$$的rank为较低，因为这些patch之间有相似性；而第二项则通过系数$$\lambda$$和F范式（[wiki][f-norm-wiki]）来限制噪声大小，避免过度去噪。

但因为rank函数是离散的，上式很难解决（NP-Hard），所以把rank用它的tightest convex approximation，即核范数（参考[知乎：为什么核范数能凸近似矩阵的秩？][nuclear-norm-zhihu]，代替，上式转换为：

$$
\min _{\mathbf{A}}\|A\|_{*}+\lambda\|\mathbf{X}-\mathbf{A}\|_{F}^{2} \tag{100}
$$ 

但上式仍有需要改进的地方（[blessing of dimensionality][bod]）。所以-->

### 2.2.2 Kernelization and Relaxation


[f-norm-wiki]:https://zh.wikipedia.org/wiki/%E7%9F%A9%E9%99%A3%E7%AF%84%E6%95%B8#%E5%BC%97%E7%BD%97%E8%B4%9D%E5%B0%BC%E4%B9%8C%E6%96%AF%E7%AF%84%E6%95%B0
[nuclear-norm-zhihu]:https://www.zhihu.com/question/26471536
[bod]:https://simplystatistics.org/2015/04/09/a-blessing-of-dimensionality-often-observed-in-high-dimensional-data-sets/