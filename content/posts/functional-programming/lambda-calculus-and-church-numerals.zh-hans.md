---
title: 浅谈 Lambda 演算
date: 2020-04-02T20:43:10+08:00
utterances: 74
toc: true
math: true
draft: true
categories: Functional Programming
tags: 
    - Functional Programming
    - Lambda Calculus
    - Mathematics
    - Abstract Algebra
---

# 简介

**$\lambda$ 演算** ($\lambda - \text{calculus}$) 是由 [Alonzo Church](https://en.wikipedia.org/wiki/Alonzo_Church) 于 20 世纪 30 年代在研究数学基础时提出的。虽然其作为数学基础研究失败了，Lambda 演算是一种**通用计算模型**，正如图灵机一样，并且可以证明它实际上是与图灵机等价的。不过相比图灵机，$\lambda$ 演算只强调符号变换规则的使用，而并不关心硬件上的具体实现。

本文只会设计无类型的 Lambda 演算，并且作为自己学习 Lambda 演算的初步学习笔记（所以说写的不太好）。等到以后对整个体系有了进一步认知后再来修改吧 qwq。

# 无类型 Lambda 演算

对于无类型的 $\lambda$ 演算，唯一的数据类型就是函数。我们可以把它当作一种理论上的语言看待。我们不妨暂且忘记数字、布尔量、字符串、循环分支等等我们在其他编程语言中常见的概念，而只关注函数。

构成 $\lambda$ 表达式 ($\lambda$ expression) 的项 (term) 可以分为如下三种：

- 变量 (name): $x$
- 函数 (function)：$\lambda x . M$
  - 其含义为一个输入 $x$ 返回结果 $M$ 的函数。
  - 例如 
- 应用 (application)：$M \ N$

如果采用上下文无关文法递归地定义 $\lambda$ 表达式，则有：

$$
\begin{aligned}
<\text{expression}> & := <\text{name}> \mid <\text{function}> \mid <\text{application}> \\
<\text{function}> & := \lambda <\text{name}> . <\text{expression}> \\
<\text{application}> & := <\text{expression}> \ <\text{expression}>
\end{aligned}
$$



# Church 数



## Church Booleans



# Refs

- [A Tutorial Introduction to the Lambda Calculus](https://arxiv.org/pdf/1503.09060v1.pdf)
- [Comp 311 - Review 2](https://www.cs.rice.edu/~javaplt/311/Readings/supplemental.pdf)
- [Church encoding](https://blog.ploeh.dk/2018/05/22/church-encoding/)
- [Church numerals: a tutorial](https://karczmarczuk.users.greyc.fr/Essays/church.html)
