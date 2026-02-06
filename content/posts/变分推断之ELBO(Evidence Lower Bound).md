---
title: 变分推断之ELBO(Evidence Lower Bound)
date: 2025-03-29T13:50:55
slug: blog-post-slug
tags:
  - Math
categories:
  - 数学基础
description: 从后验近似推导 ELBO 的一份笔记
summary: 用 KL(q||p) 把后验推断改写为优化问题，并推导出 ELBO 及其常用等价形式，顺便解释每一项在训练中的含义。
cover:
  image:
draft: false
share: true
---

## 问题定义
很多潜变量模型会写成联合分布 $p(x,z)=p(z)p(x|z)$。做贝叶斯推断时，我们真正想要的是后验 $p(z|x)$：在观测到 $x$ 之后，潜变量 $z$ 可能是什么。

困难点在于：后验里有一个归一化常数（也叫 evidence / marginal likelihood）：

$$
p(x)=\int p(x,z)\,dz
$$

当 $z$ 维度高、模型复杂时，这个积分几乎不可能精确计算，于是 $p(z|x)$ 也就“算不出来”。

变分推断（Variational Inference, VI）的思路很朴素：我们挑一个容易计算的分布族 $Q$，用 $q(z)\in Q$ 去近似真实后验，并通过优化找到最合适的那个 $q$。ELBO（Evidence Lower Bound）就是把这个优化目标写成一个可计算的下界。

## 推导
### 概率分布的表示
![|490](https://raw.githubusercontent.com/powerli2002/project-img/main/myblog/20250329140836140.png)

图里的大圆可以理解为“所有可能的分布”，小圆 $Q$ 是我们允许的近似后验空间（例如均值场、对角高斯等）。我们希望在 $Q$ 里找到一个 $q^*(z)$，让它尽量贴近真实后验 $p(z|x)$。

常用的距离度量是 KL 散度，于是目标可以写成：

$$
q^*(z)=\mathop{\mathrm{arg\,min}}\limits_{q(z)\in Q}\;\mathrm{KL}\left(q(z)\,\|\,p(z|x)\right)
$$

之所以要限制在 $Q$ 里，是为了让优化与计算变得可行：我们希望能（1）采样 $q$，（2）算出或估计所需的期望与 KL。

### ELBO 推导
从 KL 的定义出发：

$$
\mathrm{KL}\left(q(z)\,\|\,p(z|x)\right)=\mathbb{E}_q[\log q(z)]-\mathbb{E}_q[\log p(z|x)]
$$

用贝叶斯公式把后验展开：

$$
\log p(z|x)=\log p(x,z)-\log p(x)
$$

把它代回 KL：

$$
\mathrm{KL}\left(q(z)\,\|\,p(z|x)\right)=\mathbb{E}_q[\log q(z)]-\mathbb{E}_q[\log p(x,z)]+\log p(x)
$$

把不依赖 $q$ 的 $\log p(x)$ 移到左边，就得到一个非常关键的分解：

$$
\log p(x)=\underbrace{\mathbb{E}_q[\log p(x,z)]-\mathbb{E}_q[\log q(z)]}_{\mathrm{ELBO}(q)}+\mathrm{KL}\left(q(z)\,\|\,p(z|x)\right)
$$

由于 KL 散度恒非负，立刻得到下界：

$$
\log p(x)\ge \mathrm{ELBO}(q)
$$

这就是 “Evidence Lower Bound” 这个名字的来源。并且因为 $\log p(x)$ 不依赖 $q$，所以在同一个 $Q$ 上：

$$
q^*(z)=\mathop{\mathrm{arg\,max}}\limits_{q(z)\in Q}\;\mathrm{ELBO}(q)
$$
### ELBO计算
把联合分布拆成 $p(x,z)=p(x|z)p(z)$，ELBO 还能写成更常用、也更易解释的形式：

$$
\mathrm{ELBO}(q)=\mathbb{E}_q[\log p(x|z)]-\mathrm{KL}\left(q(z)\,\|\,p(z)\right)
$$

这两项的含义可以简单理解为：

- $\mathbb{E}_q[\log p(x|z)]$：数据项，让模型在给定潜变量时更“解释得通”观测 $x$（在 VAE 语境下常被叫作 reconstruction term）。
- $\mathrm{KL}(q(z)\|p(z))$：正则项，把近似后验拉向先验，避免 $q$ 偏离得太离谱。

在实现上，通常会遇到两类计算：

1. **KL 可解析**：比如 $q(z|x)$ 与先验 $p(z)$ 都是高斯、且协方差形式简单时，KL 有闭式表达，可以直接算。
2. **期望用采样近似**：从 $q(z)$ 采样 $z^{(s)}$，用 Monte Carlo 估计 $\mathbb{E}_q[\log p(x|z)]$。如果要对参数求梯度，很多模型会结合重参数化技巧来降低方差（例如 VAE）。


## Reference

- Blei, Kucukelbir, McAuliffe. *Variational Inference: A Review for Statisticians*. JASA 2017.
- Jordan et al. *An Introduction to Variational Methods for Graphical Models*. Machine Learning 1999.
- Kingma, Welling. *Auto-Encoding Variational Bayes*. ICLR 2014.
