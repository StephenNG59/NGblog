---
layout:     post
title:      "Motion Deblur"
subtitle:   "A Note of 《High-quality Motion Deblurring from a Single Image》"
date:       2019-04-11 09:45:48
author:     "StephenNG"
header-img: "img/post-bg-2015.jpg"
tags:
    - CG
    - OpenGL
---

<script type="text/javascript" src="http://cdn.mathjax.org/mathjax/latest/MathJax.js?config=default"></script>

# 1. Intro

**动态模糊**是数字摄影中常见的现象。如何把单张模糊图片去模糊化，是数字图像处理的一个基本问题。

如果假设模糊核(blur kernel)，即点扩散函数(point spread function, PSF)，是时间平移不变(shift-invariant)的，那么去模糊化的问题就简化成为**图像的反卷积**(deconvolution)问题了。

图像反卷积可以进一步分成盲(blind)和非盲两种情况。

1. 在非盲反卷积中，我们已经知道了模糊核，只需计算出去模糊的图像。传统方法（如Wiener filtering和RL deconvolution）仍被广泛使用，不过它们的缺点是，在图像的强边缘处，可能出现[振铃效应](https://zh.wikipedia.org/wiki/%E6%8C%AF%E9%88%B4%E6%95%88%E6%87%89)(ringing artifacts)；
2. 在盲反卷积中，模糊核和原图像都未知。自然图像的复杂性以及模糊核的多样性容易造成*先验概率的过拟合或欠拟合*（这个我还看不懂）。

# 2. 振铃效应

普遍认为振铃效应是由于[吉布斯现象](https://zh.wikipedia.org/wiki/%E5%90%89%E5%B8%83%E6%96%AF%E7%8E%B0%E8%B1%A1)而产生的。

![](../../../../img/in-post/post-motion-deblur/2.png)

我们知道自然图像中，会有许多明显的边缘，这些边缘即突变信号；吉布斯现象揭示了很难用有限的傅里叶基函数来很好地表示突变信号(step signals)，因为在突变点的附近会出现过冲，即使随着傅利叶级数的项数N增大，过冲会不断靠近不连续点，但是它的峰值不会下降（约为原函数在不连续点处跳变值的9%）。

![吉布斯现象](../../../../img/in-post/post-motion-deblur/SquareWave.gif)

然而，论文作者发现有限的傅里叶基函数其实可以很好地重建自然图像，他用数百个傅里叶基函数很好地重建了一些图像（并且，*"not computationally expensive at all"*），下图是用512个函数重建一维方波信号的展示：

![image](../../../../img/in-post/post-motion-deblur/3.png)

经过分析，论文作者认为反卷积过程中出现的类似振铃效应的现象，并不是由吉布斯现象产生。他们认为主要的原因是噪声模型的不准确，以及PSF估计的误差。

### 振铃效应的进一步分析

对于时间平移不变的动态模糊来说，通常用一个卷积过程来表示：

$$I = L \bigotimes f + n \tag{1}$$

其中$$I$$是模糊化的图像(degraded image)，$$L$$是原图像，$$f$$是一个线性的PSF，$$n$$是附加的噪声。

为什么噪声和核误差会造成振铃效应呢？假设核函数$$f$$以及原图像$$L$$分别由当前的估计$$f ^\prime$$、$$L ^\prime$$加上它们的误差$$\Delta f$$、$$\Delta L$$组成，那么有：

$$
\begin{aligned}
    I &= (L ^\prime + \Delta L) \bigotimes (f ^\prime + \Delta f) + n\\
    &= L ^\prime \bigotimes f ^\prime + \Delta L \bigotimes f ^\prime + L ^\prime \bigotimes \Delta f + \Delta L \bigotimes \Delta f + n \\
\end{aligned} \tag{2}
$$

如果噪音$$n$$不准确，那么容易把$$\Delta L$$和$$\Delta f$$当作噪音$$n$$的一部分，造成评估过程的不稳定。（个人理解，这里作者解释的角度主要是，**噪声**的不准确对评估的影响，而**核误差**的方面，只是在$$\Delta f$$中体现）

过去的对于噪声模拟的模型通常是使$$n$$或者它的梯度$$\partial n$$满足均值为0的高斯分布，但这种方式不能捕捉到图像噪声的一个重要特征：空间随机性。下图中(d)展示了基于这种简单模型估算出的噪声有着明显的结构性，不满足空间随机。

![image](../../../../img/in-post/post-motion-deblur/4.png)

总结一下，反卷积中出现的振铃现象不是由吉布斯现象引起，而主要是由图像噪声以及核函数估计误差造成的。在估算过程中，它们会混合在一起（见式(2)），造成估算出的图像噪声表现出了空间的结构性。

# 作者提出的模型

作者提出的概率模型把盲类型和非盲类型的反卷积统一称为一个**最大后验概率估计**([maximum a posteriori, MAP](https://en.wikipedia.org/wiki/Maximum_a_posteriori_estimation))形式([MAP的一个例子说明](https://medium.com/@chih.sheng.huang821/%E8%B2%9D%E6%B0%8F%E6%B1%BA%E7%AD%96%E6%B3%95%E5%89%87-bayesian-decision-rule-%E6%9C%80%E5%A4%A7%E5%BE%8C%E9%A9%97%E6%A9%9F%E7%8E%87%E6%B3%95-maximum-a-posterior-map-96625afa19e7)/[MAP的通俗解释](https://medium.com/@amatsukawa/maximum-likelihood-maximum-a-priori-and-bayesian-parameter-estimation-d99a23a0519f)/文章末尾有关于[MAP和MLE的通俗理解](#MAP和MLE))。他们的算法不断迭代，更新模糊核函数，以及估计的原图像，直到收敛。

![image](../../../../img/in-post/post-motion-deblur/7.png)

---

**MAP和MLE**
---
**MLE：Maximum Likelihood Estimate - 极大似然估计**
 
**MAP：Maximum a Posteriori - 最大后验概率估计**

假设我们投掷一枚硬币三次，结果都是正面朝上，那么请问投掷一次硬币结果是正面朝上的几率$$\theta$$是多大呢？

现在我们获得了数据$$D$$={1，1，1}，要求解$$\theta$$。一种解决方法是**找出$$\theta$$，使得在$$\theta$$已知的情况下，已获得的数据$$D$$出现的概率$$P(D;\theta)$$最大**：

$$h^*_{MLE}=\mathop {argmax}_{\theta}P(D;\theta) \tag{MLE}$$

我们使用$$\;;\;$$来说明$$\theta$$是分布$$P$$的一个参数，$$\theta$$定义了$$P$$。在这个例子里，$$P(D;\theta) = \theta * \theta * \theta$$，所以当$$\theta=1.0$$时，$$P$$最大。这就是<font color="CornflowerBlue"><strong>极大似然估计</strong></font>。

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