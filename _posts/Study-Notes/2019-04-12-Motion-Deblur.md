---
layout:     post
title:      "Motion Deblur"
subtitle:   "A Note of 《High-quality Motion Deblurring from a Single Image》"
date:       2019-04-11 09:45:48
author:     "StephenNG"
header-img: "img/in-post/post-motion-deblur/bg.png"
tags:
    - CG
    - OpenGL
---

- [1. Intro](#1-intro)
- [2. 振铃效应](#2-%E6%8C%AF%E9%93%83%E6%95%88%E5%BA%94)
  - [振铃效应的进一步分析](#%E6%8C%AF%E9%93%83%E6%95%88%E5%BA%94%E7%9A%84%E8%BF%9B%E4%B8%80%E6%AD%A5%E5%88%86%E6%9E%90)
- [3. 作者提出的模型](#3-%E4%BD%9C%E8%80%85%E6%8F%90%E5%87%BA%E7%9A%84%E6%A8%A1%E5%9E%8B)
  - [似然$$p(\mathbf{I}$$\|$$\mathbf{L}, \mathbf{f})$$](#%E4%BC%BC%E7%84%B6pmathbfimathbfl-mathbff)
  - [先验$$p(\mathbf{f})$$](#%E5%85%88%E9%AA%8Cpmathbff)
  - [先验$$p(\mathbf{L})$$](#%E5%85%88%E9%AA%8Cpmathbfl)
    - [全局先验$$p_g(\mathbf{L})$$](#%E5%85%A8%E5%B1%80%E5%85%88%E9%AA%8Cpgmathbfl)
    - [局部先验$$p_l(\mathbf{L})$$](#%E5%B1%80%E9%83%A8%E5%85%88%E9%AA%8Cplmathbfl)
- [4. 优化](#4-%E4%BC%98%E5%8C%96)
  - [4.1. 优化$$\mathbf{L}$$](#41-%E4%BC%98%E5%8C%96mathbfl)
    - [4.1.1. 更新$$\mathbf{\Psi}$$](#411-%E6%9B%B4%E6%96%B0mathbfpsi)
    - [4.1.3. 更新$$\mathbf{L}$$](#413-%E6%9B%B4%E6%96%B0mathbfl)
  - [4.2. 优化$$\mathbf{f}$$](#42-%E4%BC%98%E5%8C%96mathbff)
- [附录](#%E9%99%84%E5%BD%95)
  - [**MAP和MLE**](#map%E5%92%8Cmle)


<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

# 1. Intro

**动态模糊**是数字摄影中常见的现象。如何把单张模糊图片去模糊化，是数字图像处理的一个基本问题。

如果假设模糊核(blur kernel)，即点扩散函数(point spread function, PSF)，是时间平移不变(shift-invariant)的，那么去模糊化的问题就简化成为**图像的反卷积**(deconvolution)问题了。

图像反卷积可以进一步分成盲(blind)和非盲两种情况。

1. 在非盲反卷积中，我们已经知道了模糊核，只需计算出去模糊的图像。传统方法（如Wiener filtering和RL deconvolution）仍被广泛使用，不过它们的缺点是，在图像的强边缘处，可能出现[振铃效应](https://zh.wikipedia.org/wiki/%E6%8C%AF%E9%88%B4%E6%95%88%E6%87%89)(ringing artifacts)；
2. 在盲反卷积中，模糊核和潜藏图像都未知。自然图像的复杂性以及模糊核的多样性容易造成*先验概率的过拟合或欠拟合*（这个我还看不懂）。

# 2. 振铃效应

普遍认为振铃效应是由于[吉布斯现象](https://zh.wikipedia.org/wiki/%E5%90%89%E5%B8%83%E6%96%AF%E7%8E%B0%E8%B1%A1)而产生的。

<center><img src="../../../../img/in-post/post-motion-deblur/2.png" width="500" ></center>

我们知道自然图像中，会有许多明显的边缘，这些边缘即突变信号；吉布斯现象揭示了很难用有限的傅里叶基函数来很好地表示突变信号(step signals)，因为在突变点的附近会出现过冲，即使随着傅利叶级数的项数N增大，过冲会不断靠近不连续点，但是它的峰值不会下降（约为原函数在不连续点处跳变值的9%）。

<center><img src="../../../../img/in-post/post-motion-deblur/SquareWave.gif" width="500" ></center>

然而，论文作者发现有限的傅里叶基函数其实可以很好地重建自然图像，他用数百个傅里叶基函数很好地重建了一些图像（并且，*"not computationally expensive at all"*），下图是用512个函数重建一维方波信号的展示：

<center><img src="../../../../img/in-post/post-motion-deblur/3.png" width="500" ></center>

经过分析，论文作者认为反卷积过程中出现的类似振铃效应的现象，并不是由吉布斯现象产生。他们认为主要的原因是噪声模型的不准确，以及PSF估计的误差。

## 振铃效应的进一步分析

对于时间平移不变的动态模糊来说，通常用一个卷积过程来表示：

$$I = L \otimes f + n \tag{1}$$

其中$$I$$是退化图像(degraded image)，$$L$$是潜藏图像，$$f$$是一个线性的PSF，$$n$$是附加的噪声。

为什么噪声和核误差会造成振铃效应呢？假设核函数$$f$$以及潜藏图像$$L$$分别由当前的估计$$f ^\prime$$、$$L ^\prime$$加上它们的误差$$\Delta f$$、$$\Delta L$$组成，那么有：

$$
\begin{aligned}
    I &= (L ^\prime + \Delta L) \otimes (f ^\prime + \Delta f) + n\\
    &= L ^\prime \otimes f ^\prime + \Delta L \otimes f ^\prime + L ^\prime \otimes \Delta f + \Delta L \otimes \Delta f + n \\
\end{aligned} \tag{2}
$$

如果噪音$$n$$不准确，那么容易把$$\Delta L$$和$$\Delta f$$当作噪音$$n$$的一部分，造成评估过程的不稳定。（个人理解，这里作者解释的角度主要是，**噪声**的不准确对评估的影响，而**核误差**的方面，只是在$$\Delta f$$中体现）

过去的对于噪声模拟的模型通常是使$$n$$或者它的梯度$$\partial n$$满足均值为0的高斯分布，但这种方式不能捕捉到图像噪声的一个重要特征：空间随机性。下图中(d)展示了基于这种简单模型估算出的噪声有着明显的结构性，不满足空间随机。

<center><img src="../../../../img/in-post/post-motion-deblur/4.png" width="600" ></center>

总结一下，反卷积中出现的振铃现象不是由吉布斯现象引起，而主要是由图像噪声以及核函数估计误差造成的。在估算过程中，它们会混合在一起（见式(2)），造成估算出的图像噪声表现出了空间的结构性。

# 3. 作者提出的模型

作者提出的概率模型把盲类型和非盲类型的反卷积统一成为一个**最大后验概率估计**([maximum a posteriori, MAP](https://en.wikipedia.org/wiki/Maximum_a_posteriori_estimation))形式([MAP的一个例子说明](https://medium.com/@chih.sheng.huang821/%E8%B2%9D%E6%B0%8F%E6%B1%BA%E7%AD%96%E6%B3%95%E5%89%87-bayesian-decision-rule-%E6%9C%80%E5%A4%A7%E5%BE%8C%E9%A9%97%E6%A9%9F%E7%8E%87%E6%B3%95-maximum-a-posterior-map-96625afa19e7)/[MAP的通俗解释](https://medium.com/@amatsukawa/maximum-likelihood-maximum-a-priori-and-bayesian-parameter-estimation-d99a23a0519f)/文章末尾有关于[MAP和MLE的通俗理解](#MAP和MLE))。

论文作者的算法是通过不断迭代，在迭代中更新对于模糊核函数和潜藏图像的估计，直到收敛。

<center><img src="../../../../img/in-post/post-motion-deblur/7.png" width="500" ></center>
<small class="img-hint">论文作者的算法：通过不断迭代，更新模糊核和潜藏图像的估计，直至收敛</small>

根据贝叶斯定理，

$$
p(\mathbf{L}, \mathbf{f} | \mathbf{I}) \propto p(\mathbf{I} | \mathbf{L}, \mathbf{f}) p(\mathbf{L}) p(\mathbf{f}) \tag{3}
$$

其中
$$p(\mathbf{L}, \mathbf{f} | \mathbf{I})$$是**似然**(likelihood)，$$p(\mathbf{L})$$和$$p(\mathbf{f})$$
代表了先验的概率。以下逐一解释。

> 似然：给定样本X = x下参数θ为某一个值的可能性；
>
> 概率：给定参数θ下样本随机向量为x的可能性；
>

## 似然$$p(\mathbf{I}$$\|$$\mathbf{L}, \mathbf{f})$$

在已知潜藏图像和模糊核的情况下，观察到的图像出现的概率如何求解？基于卷积模型$$\mathbf{n}=\mathbf{L} \otimes \mathbf{f}-\mathbf{I}$$，这个似然就是【噪音$$\mathbf{n}$$的取值为当前向量值】的概率。

噪音$$\mathbf{n}$$的模型为：每个像素相当于一个**独立同分布**(independent and identically distributed, **i.i.d.**)的随机变量，符合**高斯分布**。

在以前，似然的模型通常表示为，使噪音绝对值符合高斯分布：
$$\prod_{i} N\left(n_{i} | 0, \zeta_{0}\right)$$，或者使噪音的一阶梯度值符合高斯分布：
$$\prod_{i} N\left(\nabla n_{i} | 0, \zeta_{1}\right)$$，其中$$i$$表示第i个像素，$$\zeta_{0}$$和$$\zeta_{1}$$是标准差(standard deviations, SD, 又称均方差)。但正如之前提到的，这些模型不能捕捉到所有的空间随机性（即，会出现空间的结构特征）。

所以，这里作者采用了更激进的方法：**约束噪音的二阶及以下的所有导数**（直接看后面的<a href="#Likelihood">似然公式</a>）。

这里作者利用到了符合高斯分布的i.i.d.的一些性质：如果变量本身符合标准差为$$\zeta_{0}$$的高斯分布，那么它的q阶导数符合高斯分布$$N\left(0, \zeta_{q}\right)$$，其中标准差$$\zeta_{q} = \sqrt{2^{q}} \times \zeta_{0}$$。简而言之，导数每增加一阶，标准差乘一次$$\sqrt{2}$$。

在这里，一阶导数包括$$\partial_{x} n_{i}$$，$$\partial_{y} n_{i}$$，即在x方向和y方向的偏导；二阶导数包括$$\partial_{x x}$$、$$\partial_{x y}$$和$$\partial_{y y}$$。

导数计算方法采用相邻像素之间的**前向差分**(forward difference)。

一阶导数：

$$\partial_{x} n(x, y)=n(x+1, y)-n(x, y)$$

$$\partial_{y} n(x, y)=n(x, y+1)-n(x, y)$$

二阶导数：

$$\begin{aligned}
    \partial_{x x} n(x, y) &= \partial_{x} n(x+1, y)-\partial_{x} n(x, y) \\ 
    &=(n(x+2, y)-n(x+1, y))-(n(x+1, y)-n(x, y)) \\
\end{aligned}$$

$$\begin{aligned}
    \partial_{y y} n(x, y) &= \partial_{y} n(x, y+1)-\partial_{y} n(x, y) \\ 
    &=(n(x, y+2)-n(x, y+1))-(n(x, y+1)-n(x, y)) \\
\end{aligned}$$

$$\begin{aligned}
    \partial_{x y} n(x, y) &= \partial_{y} n(x+1, y)-\partial_{y} n(x, y) \\ 
    &=(n(x+1, y+1) - n(x+1, y)) - (n(x, y+1) - n(x, y)) \\
\end{aligned}$$

<span id="Likelihood"></span>

然后，作者把**似然**最终定义为：

$$
\begin{aligned} 
    p(\mathbf{I} | \mathbf{L}, \mathbf{f}) &=\prod_{\partial^{*} \in \Theta} \prod_{i} N\left(\partial^{*} n_{i} | 0, \zeta_{\kappa\left(\partial^{*}\right)}\right) \\ 
    &=\prod_{\partial^{*} \in \Theta} \prod_{i} N\left(\partial^{*} I_{i} | \partial^{*} I_{i}^{c}, \zeta_{\kappa\left(\partial^{*}\right)}\right) 
\end{aligned}
$$

其中$$\Theta=\left\{\partial^{0}, \partial_{x}, \partial_{y}, \partial_{x x}, \partial_{x y}, \partial_{y y}\right\}$$，并定义$$\partial^{0} n_i=n_i$$。简而言之，就是把所有二阶及以下导数在高斯分布下的、所有像素单独的概率乘起来。

## 先验$$p(\mathbf{f})$$

因为模糊核表示了相机的路径，它应该是稀疏的，大多数值接近于0。所以作者在模糊核上应用了指数分布模型：

$$p(\mathbf{f})=\prod_{j} e^{-\tau f_{j}}, \quad f_{j} \geq 0$$

其中$$\tau$$是率参数(rate parameter)，$$j$$是模糊核中像素的索引值。

## 先验$$p(\mathbf{L})$$

当设计$$\mathbf{L}$$的先验分布时，考虑到它需要有两个作用：

1. 作为一个正则项避免反卷积过程的过拟合(? ill-posedness)；
2. 在恢复潜藏图像的时候减小振铃效应。

因此，作者把$$p(\mathbf{L})$$分为两个部分：全局先验和局部先验，即：

$$p(\mathbf{L})=p_{g}(\mathbf{L}) p_{l}(\mathbf{L})$$

### 全局先验$$p_g(\mathbf{L})$$

根据已有的研究，自然图像中的梯度分布满足重尾分布(heavy-tailed distribution)。下图展示了梯度分布的对数密度（按我的理解应该是对密度取对数吧），数据采集自10幅自然图像。

<center><img src="../../../../img/in-post/post-motion-deblur/8.png" width="500" ></center>

作者把分布函数模拟为两个连续函数的串联：

$$
\Phi(x)=\left\{
    \begin{array}
        {ll}{-k|x|} & {x \leq l_{t}} \\ 
        {-\left(a x^{2}+b\right)} & {x>l_{t}}
    \end{array}
\right.
$$

其中$$x$$是图像的梯度级别(gradient level)。在这里，作者把参数设置为$$k=2.7, a=6.1 \times 10^{-4}, b=5.0$$。现在我们为图像的对数梯度密度选好了模型，最终的全局先验$$p_{g}(\mathbf{L})$$定义为：

$$
p_{g}(\mathbf{L}) \propto \prod_{i} e^{\Phi\left(\partial L_{i}\right)}
$$

### 局部先验$$p_l(\mathbf{L})$$

退化图像中光滑的部分，去模糊化的图像也应该是光滑的，也就是不应该出现明显的边缘，而振铃效应通常则会有很多的违反这一限制的pattern。作者通过限制退化图像和潜藏图像的梯度差距，使其满足先验的高斯分布，来减小振铃效应的影响。

定义光滑部分的方法是：用一个和模糊核一样大小的窗口，计算窗口内所有像素的标准差，如果小于一个阈值，则认为该窗口的中心点像素是光滑的；把所有光滑的像素合起来，就是光滑的区域。

<center><img src="../../../../img/in-post/post-motion-deblur/9.png" width="500" ></center>

虽然这部分的后验概率只对图像中光滑的部分进行了限制，不过由于模糊核的面积覆盖，实际上对非光滑部分会有明显的影响，也能有效减弱非光滑部分的振铃效应现象。

# 4. 优化

到这里，MAP中的所有项都得到了定义（见(3)式）；再定义其取对数之后取负数的值为能量，得到能量方程的表达式：

$$
\begin{aligned}
    E(\mathbf{L}, \mathbf{f}) \propto & \left(\sum_{\partial^{*} \in \Theta} w_{\kappa\left(\partial^{*}\right)}\left\|\partial^{*} \mathbf{L} \otimes \mathbf{f}-\partial^{*} \mathbf{I}\right\|_{2}^{2}\right)+ \\ 
    & \lambda_{1}\left\|\Phi\left(\partial_{x} \mathbf{L}\right)+\Phi\left(\partial_{y} \mathbf{L}\right)\right\|_{1}+ \\ 
    & \lambda_{2}\left(\left\|\partial_{x} \mathbf{L}-\partial_{x} \mathbf{I}\right\|_{2}^{2} \circ \mathbf{M}+\left\|\partial_{y} \mathbf{L}-\partial_{y} \mathbf{I}\right\|_{2}^{2} \circ \mathbf{M}\right)+\|\mathbf{f}\|_{1}
\end{aligned} \tag{4}
$$

## 4.1. 优化$$\mathbf{L}$$

在这一步，我们固定$$\mathbf{f}$$，优化$$\mathbf{L}$$。能量$$E$$可以简化为$$E_{\mathbf{L}}$$：

$$
\begin{aligned} 
    E_{\mathrm{L}}=&\left(\sum_{\partial^{*} \in \Theta} w_{\kappa\left(\partial^{*}\right)}\left\|\partial^{*} \mathbf{L} \otimes \mathbf{f}-\partial^{*} \mathbf{I}\right\|_{2}^{2}\right)+\lambda_{1}\left\|\Phi\left(\partial_{x} \mathbf{L}\right)+\Phi\left(\partial_{y} \mathbf{L}\right)\right\|_{1} \\ 
    &+\lambda_{2}\left(\left\|\partial_{x} \mathbf{L}-\partial_{x} \mathbf{I}\right\|_{2}^{2} \circ \mathbf{M}+\left\|\partial_{y} \mathbf{L}-\partial_{y} \mathbf{I}\right\|_{2}^{2} \circ \mathbf{M}\right) 
\end{aligned} \tag{5}
$$

但$$E_{\mathbf{L}}$$此时仍然是一个包含着数十万个未知数的非凸函数。为了更有效地优化它，作者使用了**变量替换**以及**在迭代过程中更新权重**的技术（使权重随迭代次数增大，为了使得替换的变量与原变量更加接近最后趋于一致）。

这个方法基于的idea是把(5)式中复杂的卷积项（即第一项）与其他项分离开来，这样**卷积项就可以通过傅里叶变换快速地进行计算**。

引入的替代变量是$$\Psi=\left(\Psi_{x}, \Psi_{y}\right)$$，对应$$\partial \mathbf{L}=\left(\partial_{x} \mathbf{L}, \partial_{y} \mathbf{L}\right)$$，然后另外加入约束条件$$\Psi \approx \partial \mathbf{L}$$。

方程(5)由此转化为
$$
\begin{aligned} 
    E_{\mathbf{L}}=&\left(\sum_{\partial * \in \Theta} w_{\kappa\left(\partial^{*}\right)}\left\|\partial^{*} \mathbf{L} \otimes \mathbf{f}-\partial^{*} \mathbf{I}\right\|_{2}^{2}\right)+\lambda_{1}\left\|\Phi\left(\Psi_{x}\right)+\Phi\left(\Psi_{y}\right)\right\|_{1} \\ 
    &+\lambda_{2}\left(\left\|\Psi_{x}-\partial_{x} \mathbf{I}\right\|_{2}^{2} \circ \mathbf{M}+\left\|\Psi_{y}-\partial_{y} \mathbf{I}\right\|_{2}^{2} \circ \mathbf{M}\right) \\ 
    &+\gamma\left(\left\|\Psi_{x}-\partial_{x} \mathbf{L}\right\|_{2}^{2}+\left\|\Psi_{y}-\partial_{y} \mathbf{L}\right\|_{2}^{2}\right) 
\end{aligned} \tag{6}
$$

其中$$\gamma$$是随着迭代次数而增大的权重，目的是最终能够满足$$\Psi=\partial \mathbf{L}$$，从而使得，最小化$$E_{\mathbf{L}}$$就相当于最小化$$E$$。

基于此，我们可以通过固定$$\Psi$$与$$\mathbf{L}$$的其中一个，更新另一个，然后迭代进行此过程来收敛到最优解。$$\Psi$$的全局最优可以通过简单的分支方法(branching approach)来获得，$$\mathbf{L}$$则可以通过**快速傅里叶变换**来优化。

### 4.1.1. 更新$$\mathbf{\Psi}$$

把$$\mathbf{\Psi}$$拆分成每个像素，对每个像素的能量单独优化即可。

### 4.1.3. 更新$$\mathbf{L}$$

因为涉及到卷积，所以把$$\mathbf{L}$$通过傅里叶变换转换到频域中进行处理。

## 4.2. 优化$$\mathbf{f}$$

略。

# 5. 具体实现




# 附录

**MAP和MLE**
---
**MLE：Maximum Likelihood Estimate - 极大似然估计**
 
**MAP：Maximum a Posteriori - 最大后验概率估计**

假设我们投掷一枚硬币三次，结果都是正面朝上，那么请问投掷一次硬币结果是正面朝上的几率$$\theta$$是多大呢？

现在我们获得了数据$$D$$={1，1，1}，要求解$$\theta$$。一种解决方法是**找出$$\theta$$，使得在$$\theta$$已知的情况下，已获得的数据$$D$$出现的概率$$P(D;\theta)$$最大**：

$$h^*_{MLE}=\mathop {argmax}_{\theta}P(D;\theta) \tag{MLE}$$

我们使用$$\;;\;$$来说明$$\theta$$是分布$$P$$的一个参数，$$\theta$$定义了$$P$$。在这个例子里，服从二项分布的$$P(D;\theta) = \theta * \theta * \theta$$，所以当$$\theta=1.0$$时，$$P$$最大。这就是<font color="CornflowerBlue"><strong>极大似然估计</strong></font>。

但是，生活常识告诉我们，$$\theta$$应该是一个接近0.5左右的数才对。这就是我们的**先验知识(priors)**，那么，我们怎样把先验知识加入到$$\theta$$的大小估计中，使它更符合我们的直觉呢？

这就要用到另一种解决方法：<strong>在$$D$$已知的情况下，找出出现概率最大的一个$$h$$，亦即需要求出后验分布(posteriori distribution)$$P(\theta | D)$$，然后找到极值点</strong>：

$$h^*_{MAP}=\mathop {argmax}_{\theta}P(\theta|D) \tag{MAP}$$

为什么这种方法可以让我们把先验知识加入到估计中？因为，贝叶斯定理告诉我们：

$$P(\theta|D)=\frac{P(D|\theta)P(\theta)}{P(D)}$$

<!-- ![Bayes](../../../../img/in-post/post-motion-deblur/Bayes'_Theorem_MMB_01.jpg) -->

而由于我们有了先验知识，实际上对于任意一个特定的$$\theta$$，我们已经知道了$$P(\theta)$$，而$$P(D$$\|$$\theta)$$这一项也可以算出来。所以我们解决了分子，但是分母呢？

$$P(D)=\int P(D,\theta) \, {\rm d}\theta$$

分母实际上是在$$\theta$$满足特定分布的时候，$$D$$出现的概率，而这只和$$\theta$$的分布相关（which我们已经通过先验知识假定好了），和$$\theta$$的具体取值无关。所以$$(MAP)$$式可以改写为：

$$h^*_{MAP}=\mathop {argmax}_{\theta}P(D|\theta)P(\theta) \tag{MAP*}$$

然后我们就可以通过各种方法找出一个$$\theta$$值使得上式取值最大了。一个常用的方法是[共轭梯度下降法](https://en.wikipedia.org/wiki/Conjugate_gradient_method)。

在这个抛硬币的例子中，如果我们假设$$\theta$$应该满足一个$$\mu=0.5$$，$$\sigma^2$$很小（即聚拢在0.5附近）的高斯分布，那么最后求出来的$$\theta$$应该比0.5略大，比1小很多。这就是<font color="CornflowerBlue"><strong>最大后验概率估计</strong></font>。

MAP的优势和劣势如下：

![MAP的优势和劣势](../../../../img/in-post/post-motion-deblur/5.png)

---

* **正则化 Regularization**（L1正则化）：见《正则化.ppt》

* **内点法 Interior Point Method**：[百度百科](https://baike.baidu.com/item/%E5%86%85%E7%82%B9%E6%B3%95/115627?fr=aladdin)

* **共轭梯度法**：[wiki](https://zh.wikipedia.org/wiki/%E5%85%B1%E8%BD%AD%E6%A2%AF%E5%BA%A6%E6%B3%95)

* <span style="border-bottom:1px solid skyblue">下划线示例</span>

---

$$
f(n) = 
\begin{cases}
\end{cases}
$$