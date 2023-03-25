---
layout: post
title: "《分布式计算--原理、算法与系统》2 --- 逻辑时间"
subtitle: "介绍与逻辑时间相关的原理、算法"
date: 2022-07-12
author: "Cheney.Yin"
header-img: "//imgloc.com/i/iADI63"
tags: 
 - 《分布式计算--原理、算法与系统》
 - 逻辑时间
 - 分布式计算
 - 因果关系
---

> 本文为《分布式计算--原理，算法与系统》第三章“逻辑时间”笔记。

# 背景

一般情况下，可以使用物理时间跟踪因果关系。但是，在分布式系统中，不可能存在**全局物理时间**。在分布式系统中，可以使用逻辑时间（Logic Time）来捕获分布式计算时间之间的因果关系（Causality）。

和日常生活中使用钟表推断因果关系相比，在分布式计算中，事件发生的频率要高数个数量级，而事件的执行时间则小数个数量级。如果物理时钟不能精确地同步，在事件高频发生、瞬时执行的场景下，是难以准确捕获事件因果关系的。例如，广泛使用的网络时间协议（Network Time Protocol，NTP）只能确保事件准确度为几十毫秒，并不适合捕获分布式系统中的因果关系。

> 如果事件的执行耗时远远大于NTP偏移的几十毫秒，NTP提供时间戳仍然能够用于推断分布式事件的因果关系。

逻辑时钟系统则解决了这些问题。在一个逻辑时钟系统中，每个进程都有一个按照规则运行的逻辑时钟，每个事件都被赋予一个时间戳，事件间的因果关系可以从时间戳推导出来。这些时间戳满足单调性，如事件$$a$$影响事件$$b$$，那么事件$$a$$的事件戳小于事件$$b$$。

# 逻辑时钟系统框架

## 定义

逻辑时钟系统由一个时间域$$T$$和一个逻辑时钟$$C$$组成。时间域$$T$$上存在偏序关系$$<$$，这种关系表示事件发生优先或因果优先。逻辑时钟$$C$$是一个函数，该函数将分布式系统中的事件$$e$$映射到时间域$$T$$，得到的时间戳记为$$C(e)$$，其定义如下：

$$
C:\ H\mapsto T
$$

并满足如下性质：

- **时钟一致性条件**：对任意两个事件$$e_i$$和$$e_j$$，$$e_i \to e_j \Rightarrow C(e_i) < C(e_j)$$。

如果对于对任意两个事件$$e_i$$和$$e_j$$，满足$$e_i \to e_j \Leftrightarrow C(e_i) < C(e_j)$$，则逻辑时钟系统为**强一致**的。

## 实现逻辑时钟

实现逻辑时钟要解决如下两个问题：

1. 每个进程表示逻辑时间的数据结构。

   对于任意进程$$p_i$$，

   - 拥有本地逻辑时钟$$lc_i$$，可以测量进程$$p_i$$的进度。
   - 拥有全局时钟$$gc_i$$，表示进程$$p_i$$从本地角度所见的全局逻辑时间。

2. 能够保证时钟一致性条件，用于更新数据结构的协议。

   - **R1规则**：管理当进程执行一个事件（发送、接收或内部事件）时，如何更新本地逻辑时钟。
   - **R2规则**：管理进程如何更新它的全局逻辑时钟，使得全局进度和进程所见全局时间得以更新。规定了在发送消息时附加何种有关逻辑时间的信息，规定了信息如何被接收、如何用于更新进程所见的全局时间。

# 标量时间

## 定义

标量时间（Scalar Time）表示法是由Lamport于1978年提出，用于分布式系统中的事件全排序。该表示法中的时间域是非负整数集合，进程$$p_i$$的本地逻辑时钟和本地所见全局时间都被归为一个整数变量$$C_i$$。

**R1**和**R2**更新时钟规则如下：

1. **R1规则**。

   在执行一个事件之前，进程$$p_i$$执行如下动作：
   
   $$
   C_i\ :=\ C_i + d,\qquad d>0
   $$

2. **R2规则**。

   每个消息附加消息发送方的时钟值，当进程$$p_i$$接收到带有时间戳$$C_{msg}$$的消息时，它执行如下动作：

   （1）$$C_i\ :=\ max(C_i,C_{msg})$$；

   （2）执行R1规则；

   （3）传递（Deliver）该消息。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distcmp_pas_2_1.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图1 标量时间变化
  	</div>
</center>

图1显示了$$d=1$$的标量时间演化过程。

## 基本性质

### 一致性（Consistency）

对于标量时钟满足单调性，即：

$$
\forall e_i,e_j \qquad e_i \to e_j \Rightarrow C(e_i) < C(e_j)
$$

### 全序（Total Ordering）

标量时钟能够用于排列分布式系统的全序事件。在标量时钟系统下，存在两个事件的时间戳相同的情况（如图1中，进程$$p_1$$的第三个事件与进程$$p_2$$的第二个事件。）。为了保证排列为全序，需要对相同标量时间戳事件进行比较。由于事件的时间戳相同，意味着两个事件为并发关系，所以按照进程标识排序并发事件，并且这也**不会破坏任何因果关系**。

> $$\forall e_i,e_j \qquad C(e_i)=C(e_j) \Rightarrow e_i \parallel e_j$$，
>
> 证明： 假设$$C(e_i)=C(e_j) \nRightarrow e_i \parallel e_j$$，那么$$e_i \to e_j$$或者$$e_j \to e_i$$。
>
> 根据标量时钟系统R1规则、R2规则可知，
>
> 当$$e_i \to e_j$$时，$$C(e_j) - C(e_i) \geqslant d$$，当$$e_j \to e_i$$时，$$C(e_i) - C(e_j) \geqslant d$$。
>
> 由于$$d > 0$$，所以假设不成立。故此，可证$$C(e_i)=C(e_j) \Rightarrow e_i \parallel e_j$$。

一个事件的时间戳用$$(t,i)$$表示，其中$$t$$表示发生时间，$$i$$为事件在进程上的标识。若事件$$x$$的时间戳为$$(h,i)$$，事件$$y$$的时间戳为$$(k,j)$$，事件$$x$$和$$y$$的全序关系$$\prec$$定义如下：

$$
x \prec y \Leftrightarrow (h < k) \quad \lor \quad (h = k \ \land \ i < j)
$$

另外，$$x \prec y \Rightarrow x \to y \lor x \parallel y$$。

> **为什么$$h < k \Rightarrow x \prec y$$？**
>
> 标量时钟系统不是强一致的，所以$$h < k$$蕴含两种可能的情况：
>
> （1）$$x \to y$$
>
> 自然，全序关系$$x \prec y$$可以成立。
>
> （2）$$x \parallel y$$
>
> 因为$$x$$和$$y$$不存在依赖关系，则$$x \prec y$$或者$$y \prec y$$均不会破坏因果关系，为了同情况（1）在形式上对齐，取$$x \prec y$$成立。

### 事件计数

如果$$d$$总为$$1$$，则标量时间有如下性质：

如果事件$$e$$有一个时间戳$$h$$，那么$$h-1$$表示在产生事件$$e$$之前，所需要的最小逻辑持续时间（以事件为计数单位），即事件$$e$$的高度。

也就是说，在事件$$e$$产生之前，已经顺序产生了$$h-1$$个事件，不管这些事件是由哪些进程产生的。如图1中，以事件$$b$$的为终点的因果路径上，有5个事件优先于事件$$b$$。

### 非强一致性

标量时钟系统不是强一致的，因为$$\forall e_i,e_j \quad C(e_i) < C(e_j) \nRightarrow e_i \to e_j$$。

# 向量时间

由于标量时钟系统中，每个进程的本地逻辑时钟和本地全局时钟被压缩成了一个变量，造成不同进程的事件之间因果依赖信息缺失，导致标量时钟系统不是强一致的。向量时钟（Vector Clock）系统解决这些问题。

## 定义

在向量时钟系统中，时间域$$T$$是由$$n$$维非负整数向量集合表示。每个进程$$p_i$$维护一个向量$$vt_i[1 \cdots n]$$，其中分量$$vt_i[i]$$是进程$$p_i$$的本地逻辑时钟，其记录了进程$$p_i$$上的逻辑时间的进度。$$vt_i[j]$$表示进程$$p_j$$的本地时间的最近信息。例如，若$$vt_i[j]=x$$，那么进程$$p_i$$就知道进程$$p_j$$的逻辑时间已经进展到了$$x$$。向量$$vt_i$$为进程$$p_i$$所见的全局时间。

**R1**和**R2**更新时钟规则如下：

- **R1规则**。

  在执行一个事件之前，进程$$p_i$$更新其本地逻辑时间规则如下：
  
  $$
  vt_i[i]\ :=\ vt_i[i] + d, \qquad d > 0
  $$

- **R2规则**。

  消息发送方进程，在发送消息时将其向量时钟附加在消息$$m$$上。进程接收到消息$$(m,vt)$$时，进程$$p_i$$执行如下动作：

  （1）更新它的全局逻辑时间，如下：
  
  $$
  1 \leqslant k \leqslant n:\quad vt_i[k] \ := \ max(vt_i[k], vt[k])
  $$
  
  （2）执行**R1规则**；

  （3）传递消息$$m$$。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distcmp_pas_2_2.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图2 向量时间的演进
  	</div>
</center>

图2为增量$$d=1$$的向量时钟演进的例子。

用于比较向量时间戳$$vh$$和$$vk$$，定义如下关系：

$$vh = vk \Leftrightarrow \forall x:\ vh[x] = vk[x]$$

$$vh \leqslant vk \Leftrightarrow \forall x: vh[x] \leqslant vk[x]$$

$$vh < vk \Leftrightarrow vh \leqslant vk\ \land\ \exists x:\ vh[x] < vk[x]$$

$$vh \parallel vk \Leftrightarrow \neg(vh < vk)\ \land\ \neg(vk < vh)$$

## 基本性质

### 同构（Isomorphism）

如果两个事件$$x$$和$$y$$，分别有时间戳$$vh$$和$$vk$$，那么

$$
\begin{align*}
& x \to y \Leftrightarrow vh < vk \\
& x \parallel y \Leftrightarrow vh \parallel vk
\end{align*}
$$

这样，由分布式计算产生的偏序事件集合（由因果关系$$\to$$归纳出的事件集合）与它们的向量时间戳是**同构关系**。

如果发生一个事件的过程是已知的，那么比较两个时间戳可以被简化为：若事件$$x$$和$$y$$分别发生在进程$$p_i$$和$$p_j$$，并且事件的时间戳分别为$$vh$$和$$vk$$，那么

$$
\begin{align*}
& x \to y \Leftrightarrow vh[i] \leqslant vk[i] \\
& x \parallel y \Leftrightarrow vh[i] > vk[i]\quad \land \quad vh[j] < vk[j]
\end{align*}
$$

> $$x \to y \Leftrightarrow vh[i] \leqslant vk[i]$$ ?
>
> 证明：
>
> （1）当$$x \to y$$，根据**R2规则**可知，
> 
> $$
> vk[z]\ := max(vk[z], vh[z]),\qquad z \in [1,n]\ \land z \neq j
> $$
> 
> 那么，$$vk[i]\ := max(vk[i],vh[i])$$，所以$$x \to y \Rightarrow vh[i] \leqslant vk[i]$$。
>
> （2）当$$vh[i] \leqslant vk[i]$$时，
>
> 假设A. 假设在进程$$p_i$$上存在事件$$e$$，事件$$e$$直接影响事件$$y$$，事件$$e$$的向量时间戳为$$ve$$，那么$$vk[i] = ve[i]$$。
>
> 如果$$vh[i] = vk[i]$$，那么$$vh[i]=ve[i]$$，根据**R1规则**的单调性可知，事件$$x$$和$$e$$为同一事件，自然$$x \to y$$。
>
> 如果$$vh[i]<vk[i]$$，那么$$vh[i]<ve[i]$$，同样由**R1规则**的单调性可知，$$x \to e$$，又因为$$e \to y$$，根据传递性可知$$x \to y$$。
>
> 所以在假设A下，$$vh[i] \leqslant vk[i] \Rightarrow x \to y$$。
>
> 假设B. 假设在进程$$p_j$$上存在事件$$e$$，事件$$x$$直接影响事件$$e$$，事件$$e$$的向量时间戳为$$ve$$，那么$$vh[i]=ve[i]$$。
>
> 由于$$vh[i] \leqslant vk[i]$$，所以$$ve[i] \leqslant vk[i]$$。
>
> 由于事件$$e$$和$$y$$都发生在进程$$p_j$$上，那么$$e \to y$$或者$$y \to e$$成立。
>
> 又因为$$ve[i] \leqslant vk[i]$$，必然$$e \to y$$。
>
> 所以在假设B下，$$vh[i] \leqslant vk[i] \Rightarrow x \to y$$。
>
> 所以$$vh[i] \leqslant vk[i] \Rightarrow x \to y$$成立。
>
> 综上，$$x \to y \Leftrightarrow vh[i] \leqslant vk[i]$$。

### 强一致性

向量时钟系统具有**强一致性**，即分布式系统中的两个事件$$x$$、$$y$$，$$x \to y \Leftrightarrow C(x) < C(y)$$

> 在向量时钟系统中，$$\forall x, y\ x \to y \Leftrightarrow C(x) < C(y)$$ ?
>
> 证：
>
> （1）当$$x$$和$$y$$发生在同一进程上，
>
> 根据**R1**规则可知，$$x \to y \Leftrightarrow C(x)<C(y)$$。
>
> （2）当$$x$$和$$y$$发生在不同进程上，
>
> 进程分别为$$p_i$$、$$p_j$$，那么
> 
> $$
> x \to y \Leftrightarrow C(x)[i] \leqslant C(y)[i]
> $$
> 
> 若$$C(x)<C(y)$$，则$$C(x) \leqslant C(y)\ \land \exists k: C(x)[k]<C(y)[k]$$。由$$C(x) \leqslant C(y)$$可知$$C(x)[i] \leqslant C(y)[i]$$，所以$$C(x)<C(y) \Rightarrow x \to y$$。
>
> 若$$x \to y$$，则根据**R1、R2规则**可知，
> 
> $$
> \begin{align*}
> & C(y)[z] := max(C(y)[z], C(x)[z]), \qquad &z \in [1,n]\ \land\ z \neq j \\
> & C(y)[j] := C(y)[j] + d, \qquad &d>0
> \end{align*}
> $$
> 
> 故而，$$C(x) \leqslant C(y)\ \land\ C(x)[j] < C(y)[j]$$，即$$C(x)<C(y)$$。
>
> 所以$$x \to y \Rightarrow C(x) <C(y)$$。
>
> 综上$$x \to y \Leftrightarrow C(x)<C(y)$$。

### 事件计数

如果在**R1规则**中，$$d$$总为1，那么在进程$$p_i$$中向量的第$$i$$个元素$$vt_i[i]$$表示在该进程上知道的瞬间之前发生的事件数。

如果事件$$e$$有时间戳$$vh$$，那么$$vh[j]$$就表示进程$$p_j$$执行的因果关系先于事件$$e$$的事件数。$$\sum_{j=1}^{n}{vh[j]}\ - 1$$表示分布式计算中因果关系先于$$e$$的事件总数。

# 向量时钟的有效实现

当分布式系统中进程数量非常大时，为了散播时间进度、更新时钟，向量时钟需要在消息中附加大量信息，消息的开销会随着进程数增加而线性增长。

## 差量技术

为了优化消息开销，Singhal和Kshemkalyani提出来差量技术。**在进程接收消息后更新向量时，并非向量的全部项都会被更新，**所以一个进程$$p_i$$发送一个消息给进程$$p_j$$时，消息中只需要包含自上次向$$p_j$$发送消息以来，向量时钟中被更新的部分项即可，并不需要发送整个向量时钟。

从上次发送给$$p_j$$消息以来，$$p_i$$向量时钟的项$$i_1$$，$$i_2$$，$$\cdots$$，$$i_{n_i}$$已经更新为$$v_1$$，$$v_2$$，$$\cdots$$，$$v_{n_i}$$，则进程$$p_i$$可把$$\{(i_1,v_1),(i_2,v_2),\cdots,(i_{n_i},v_{n_i})\}$$这样的压缩时间戳附加在发往$$p_j$$的消息中。当$$p_j$$接收到消息时，更新它的向量时钟：

$$
vt_i[i_k]=max(vt_i[i_k],v_k),\quad k=1,2,\cdots,n_i
$$

该技术消减了消息大小、通信带宽的需求。在最坏情况下，整个向量时钟都要被发送，一般情况下消息中时间戳尺寸是小于$$n$$的。

因为发送消息前要筛选出向量时间戳的被更新向量，所以每个进程中要记住它上一次发送给其它所有进程的向量时间戳，这导致每个进程上要有$$\mathbb{O}(n^2)$$的存储空间。

差量技术使用了如下优化方法，将每个进程的存储开销消减至$$\mathbb{O}(n)$$。该方法需要进程$$p_i$$维护两个附加向量：

（1）$$LS_i[1\cdots n]$$：$$LS_i[j]$$表示进程$$p_i$$上一次向$$p_j$$发送消息时的本地逻辑时间$$vt_i[i]$$。

（2）$$LU_i[1\cdots n]$$：$$LS_i[j]$$表示进程$$p_i$$上次更新$$vt_i[j]$$时（捕获进程$$p_j$$新进展时）的本地逻辑时间$$vt_i[i]$$。

当进程$$p_i$$接收到消息后，捕获到进程$$p_j$$新进展后，更新$$vt_i[j]$$，$$LU_i[j]$$也将更新。当进程$$p_i$$向$$p_j$$发送消息时，$$LS_i[j]$$将更新。**如果从上次$$p_i$$到$$p_j$$通信以来，向量时钟的部分项被更新，如$$vt_i[k]$$被更新，那么$$LS_i[j]<LU_i[k]$$必然成立。**进一步来看，
$$
LS_i[j] < LU_i[k] \Leftrightarrow vt_i[k]被更新
$$
因此，当$$p_i$$发送一个消息给$$p_j$$时，它需要附加如下压缩的向量时间戳：

$$
\{(x,vt_i[x])\ \lvert \ LS_i[j] < LU_i[x]\}
$$

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distcmp_pas_2_3.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图3 使用差量技术的向量时钟进展
  	</div>
</center>

从图3中，可以看出每次消息发送附加的时间戳未使用整个向量时间戳，但是没有破坏了向量时钟的正确性。

使用这样技术能够实质性地减少大规模分布式系统中维护向量时钟的开销，尤其是进程相互作用表现出**瞬时性和空间局部性时**。



