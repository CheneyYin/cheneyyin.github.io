---
layout: post
title: "《分布式计算--原理、算法与系统》第二章 分布式计算模型 "
subtitle: "本文概况了分布式计算的基本抽象，包括分布式事件、通信信道、全局状态的数学模型。"
date: 2022-07-11
author: "Cheney.Yin"
header-img: "img/bg-material.jpg"
tags: 
 - 《分布式计算--原理、算法与系统》
 - 分布式计算
 - 分布式事件
 - 通信信道
 - 全局状态
 - 数学模型
---

> 本文为《分布式计算--原理、算法与系统》第二章“分布式计算模型”的读书笔记。本文概况了分布式计算的基本抽象，包括分布式事件、通信信道、全局状态的数学模型。

# 1 基本假设

假设，一个分布式程序由$$n$$个**异步**进程$$p_1$$，$$p_2$$，$$...$$，$$p_i$$，$$...$$，$$p_n$$组成，进程之间使用通信网络进行消息传递。

> 为了保证一般性，各个进程之间没有共享存储，只能互发消息联系。

令$$C_{ij}$$表示进程$$p_i$$到$$p_j$$的通信信道，$$m_{ij}$$表示进程$$p_i$$发往$$p_j$$的消息。进程可以自发的执行某个动作，异步发送消息。

> 异步发送消息，即进程发送消息后不用等待该消息是否发送完成，转而执行其它动作。

# 2 分布式事件类型

进程的执行的动作可简单抽象为原子动作，这些动作可以分为三类：**内部事件**、**消息发送事件**、**消息接收事件**。

令$$e^{x}_i$$表示进程$$p_i$$上的第$$x$$个事件；对于一个消息$$m$$，$$send(m)$$表示发送消息，$$rec(m)$$表示接收消息。

各种事件的发生，将会导致对应的进程和通信信道的状态改变。例如，一个内部事件改变其所处的进程的状态，一个发送事件或者接收事件则改变事件收发双方的状态。

进程$$p_i$$运行过程会产生一个事件序列：$$e^1_i$$，$$e^2_i$$，$$...e^x_i$$，$$e^{x+1}_i$$，$$...$$，该序列记为$$\mathcal{H_i}$$：

$$
\mathcal{H_i}=(h_i,\to_i)
$$

其中，$$h_i$$是进程$$p_i$$产生的事件集合；二元关系$$\to_i$$则定义了事件之间顺序，即**因果依赖**（Causal Dependency）。

进程间发送、接收消息使得信息得以流动，同时也在发送进程和接收进程间建立了因果依赖关系。这里，使用$$\to_{msg}$$表示进程间消息交换导致的因果依赖，对两个进程间交换消息$$m$$有：
$$
send(m)\to_{msg}rec(m)
$$
关系$$\to_{msg}$$定义了在发送和接收事件的因果依赖。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distrcmp_pas_1-1.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图1 某分布式运行时空图
  	</div>
</center>

图1是一个包含三个进程的分布式运行时空图，横轴表示进程运行过程（即时间方向），每一个点表示一个事件，点与点之间的箭头线表示一次消息传递。

> 通常，事件的执行要花费一定的时间，这里简化细节，假设事件的执行是原子的（具有不可分割性、瞬时性），所以用点表示事件执行。

图1中，对于进程$$p_1$$，$$e^1_1$$、$$e^3_1$$是内部事件，$$e^2_1$$、$$e^5_1$$是消息发送事件，$$e^4_1$$是消息接收事件。

# 3 分布式事件关系

分布式事件间存在两类关系：因果优先关系（Causal Precedence Relation）、逻辑并发和物理并发关系。

## 3.1 因果优先关系

一个分布式应用的运行导致不同进程产生一组分布式事件，令$$H=\cup_ih_i$$表示分布式计算过程中执行事件的集合。在集合$$H$$上定义一个二元关系$$\to$$，来表示分布式运行中事件间的因果依赖关系。
$$
\forall e^x_i,\forall e^y_j \in H, e^x_j \to e^y_j \Leftrightarrow \begin {cases}
e^x_i \to_i e^y_j i.e.,(i=j) \land (x < y) &or //进程内顺序\\\
e^x_i \to_{msg} e^y_j &or //进程间的消息传递\\\
\exists e^z_k \in H: e^x_i \to e^z_k \land e^z_k \to e^y_j &//间接依赖
\end{cases}
$$


在分布式计算事件间的因果优先关系引发了一个**反自反偏序关系（严格偏序关系）**[^1]，记为$$\mathcal{H}=(H,\to)$$。

> 之所以为偏序关系而非全序关系，是因为该关系并非对所有事件成立，例如图1中$$e^3_1$$和$$e^4_2$$不存在因果关系。

> 之所以为严格偏序关系，是因为不存在某个事件依赖自身的事实，即$$\forall e^x \in H, e^x \nrightarrow e^x$$。

其中关系$$\to$$是Lamport的**happens before**关系，对于任意两个事件$$e_i$$和$$e_j$$，如果$$e_i \to e_j$$则事件$$e_j$$直接或间接依赖$$e_i$$。例如图1中存在关系$$e^2_1 \to e^2_2$$。

对于任意两个事件$$e_i$$和$$e_j$$，如果$$e_i \nrightarrow e_j$$则事件$$e_j$$不直接或间接依赖$$e_i$$。在图1中存在关系$$e^3_1 \nrightarrow e^3_3$$。

> - 对于任意两个事件$$e_i$$和$$e_j$$，$$e_i \nrightarrow e_j \nRightarrow e_j \nrightarrow e_i$$；
>
> - 对于任意两个事件$$e_i$$和$$e_j$$，$$e_i \to e_j \Rightarrow e_j \nrightarrow e_i$$。
>
>   例如图1中，进程$$p_1$$在$$e^2_1$$处向进程$$p_2$$发送消息，进程$$p_2$$在$$e^2_2$$处接收消息，根据常理，事件$$e_j$$依赖$$e_i$$，即先发送后接收；$$e^2_2 \nrightarrow e^2_1$$必然不成立，不可能在消息发送前接收到消息。

对于任意两个事件$$e_i$$和$$e_j$$，如果$$e_i \nrightarrow e_j$$且$$e_j \nrightarrow e_i$$，则事件$$e_i$$和$$e_j$$是并发的，这种关系记为$$e_i \parallel e_j$$。例如，在图1中存在$$e^3_1 \parallel e^3_3$$。

> 并发关系没有**传递性**，即$$(e_i \parallel e_j) \land (e_j \parallel e_k) \nRightarrow e_i \parallel e_k$$，例如图1中$$e^3_3 \parallel e^4_2$$ 且$$e^4_2 \parallel e^5_1$$，但是$$e^3_3 \nparallel e^5_1$$。

**注意对任意两个事件$$e_i$$和$$e_j$$，在分布式运行中，要么$$e_i \to e_j$$，要么$$e_j \to e_i$$，或者$$e_i \parallel e_j$$。**

## 3.2 逻辑并发和物理并发

- 逻辑并发：当两个事件间无因果影响时，两个事件是逻辑并发的。

- 物理并发：不同事件在物理时间的同一时刻发生。

  > 两个或者更多个事件不在同一物理时间发生，但它们可能是逻辑并发的。例如图1中，$$\{e^3_1,e^4_2,e^3_3\}$$是逻辑并发的，但是它们显然不在同一物理时刻发生。

# 4 通信网络模型

通信网络提供了几种模型，分别是：先进先出（First-In First-Out，FIFO）、非先进先出、因果序（Causal Ordering）。

在FIFO模型中，每个信道都运行一个FIFO消息**队列**，消息顺序是由信道维持的。

在非FIFO模型中，每个信道都运行一个**集合**，其中发送进程向集合加入消息，接收进程从中移除消息，**被加入或移除的消息顺序是随机的**。

因果序模型是基于**happens before**关系，一个支持因果序模型的系统满足因果依赖（CO）：
$$
\begin{align*}
\forall m_{ij},m_{kj},\qquad &if\qquad send(m_{ij}) \to send(m_{kj})\\\
&then\quad rec(m_{ij}) \to rec(m_{kj})
\end{align*}
$$


该特性确保那些发往同一目标的因果依赖的消息，以符合它们之间因果依赖关系的顺序进行发送。因果依赖消息的发送暗含了FIFO消息发送的特性。进一步，$$CO \subset FIFO \subset 非FIFO$$。

# 5 分布式系统的全局状态

分布式系统的全局状态是其各个组成部件本地状态的集合，包括各个处理器状态和所有通信信道的状态。处理器在任何时刻的状态由处理器寄存器状态、堆栈状态以及内存状态等来定义，而且依赖于分布式应用的本地语义；信道的状态则由信道中传输的消息集合给出。

**事件发生会改变相关处理器和信道的状态，进而导致全局状态变化。**

令$$LS^x_i$$表示处理器（或者进程）$$p_i$$在事件$$e^x_i$$发生后，在事件$$e^{x+1}_i$$发生前的状态。$$LS^0_i$$则表示$$p_i$$的初始状态，$$LS^x_i$$是$$p_i$$从初始状态直到事件$$e^x_i$$之间所有事件执行后的结果。

令$$send(m) \leqslant LS^x_i$$，表示$$\exists y:\ 1 \leq y \leq x \ :: e^y_i=send(m)$$，即截止至事件$$e^x_i$$发生，$$p_i$$上发送过消息$$m$$。

令$$rec(m) \nleqslant LS^x_i$$，表示$$\forall y:\ 1 \leq y \leq x \ :: \ e^y_i \neq rec(m)$$，即截止至事件$$e^x_i$$发生，$$p_i$$从未接收过消息$$m$$。

一个信道的状态难以形式化地描述，其状态依赖于所有连接的处理器的状态。令$$SC^{x,y}_{ij}$$表示信道$$C_{ij}$$的状态，其定义如下：
$$
SC^{x,y}_{ij}=\{m_{ij}\ \lvert\ send(m_{ij}) \leqslant LS^x_i \land rec(m_{ij}) \nleqslant LS^y_j\}
$$
因此，信道状态$$SC^{x,y}_{ij}$$表示直至事件$$e^x_i$$，$$p_i$$发送的所有信息中，$$p_j$$直至事件$$e^y_j$$还未接收的消息。也就是信道$$C_{ij}$$截止在事件$$e^y_j$$时，内积压的$$p_i$$在截止在事件$$e^x_i$$时发送的所有信息。

分布式系统的全局状态是所有处理器本地状态和信道状态的集合，全局状态可定义为
$$
GS=\{\cup_iLS^{x_i}_i,\ \cup_{j,k}SC^{y_j,z_k}_{jk}\}
$$
全局状态的快照(Snapshot)是有意义的，分布式系统的所有组成部件的状态都必须在同一时刻进行记录。但是，只有在所有进程的本地时钟都能够完美地和其它进程的时钟同步，或者有一个所有处理器都能够随时访问的全局系统时钟才可能实现。但是，二者都不能。

然而，分布式系统的所有部件的状态无法在同一瞬间被记录，记录每个消息作为发送消息或接收消息的情况也是有意义的。

万事有因才有果，一个消息没有被发送，也就无法被接收，这种状态被称为一致性全局状态。在分布式系统下，非一致性全局状态是不存在的，没有意义的。

一个全局状态$$GS=\{\cup_iLS^{x_i}_i,\ \cup_{j,k}SC^{y_j,z_k}_{jk}\}$$是一个一致性全局状态，假如满足如下条件：
$$
\forall m_{ij}:\ send(m_{ij}) \nleqslant LS^{x_i}_i \Rightarrow m_{ij} \notin SC^{x_i,y_j}_{ij} \land rec(m_{ij}) \nleqslant LS^{y_j}_j
$$
也就是没有发送消息，就没有接收消息。

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distrcmp_pas_1-2.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图2 某分布式运行的时空图
  	</div>
</center>

在图2中，包含本地状态$$\{LS^1_1,LS^3_2,LS^3_3,LS^2_4\}$$的全局状态$$GS_1$$是不一致的，因为$$p_2$$的状态$$LS^3_2$$已经记录了消息$$m_{12}$$的接收，然而$$p_1$$的状态$$LS^1_1$$还未记录该消息的发送，所有不满足一致性条件，即没有发送消息，就没有接收消息。

在图2中，包含本地状态$$\{LS^2_1,LS^4_2,LS^4_3,LS^2_4\}$$的全局状态$$GS_2$$是一致状态，因为除了包含消息$$m_{21}$$的$$C_{21}$$外的其余信道都是空的（空信道意味着发送的消息都已被接收）。

全局状态$$GS=\{\cup_i LS^{x_i}_i, \cup_{j,k}SC^{y_j,z_k}_{jk}\}$$是非中转的，假如：
$$
\forall i, \forall j:\ 1 \leq i,j \leq n\ ::\ SC^{y_i,z_j}_{ij}\ = \varnothing
$$
所有信道在非中转全局状态中都被记录为空。

如果一个全局状态是非中转的，并且是一致的，则该全局状态是**强一致**的。例如图2中，包含本地状态$$\{LS^2_1,LS^3_2,LS^4_3,LS^2_4\}$$的全局状态是强一致状态。

# 6 分布式计算的运行分割

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distrcmp_pas_1-3.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图3 分布式运行中的分割线
  	</div>
</center>

分割线与每条进程线的某个点相交，可以把整个计算程分为两个部分，分布式事件也被分为两个集合，分别为过去集合（PAST）和未来集合（FUTURE）。PAST集合为分割线左侧所有事件组成，FUTURE集合则为分割线右侧所有事件组成。对于一条分割线$$C$$，$$PAST(C)$$、$$FUTURE(C)$$分别表示$$C$$的PAST事件集合和FUTURE事件集合。

每条分割线也对应了一个全局状态。例如在图3中，分割线$$C_1$$对应了一个本地状态为$$\{LS^1_1,LS^3_2,LS^3_3,LS^2_4\}$$全局状态。

**定义**：如果$$e^{Max\_PAST_i(C)}_i$$表示进程$$p_i$$上，分割线$$C$$的PAST集合中最新的事件，那么由分割线所表示的一致性全局状态是$$\{\cup_i LS^{Max\_PAST_i(C)}_i,\cup_{j,k}SC^{y_j,z_k}_{jk}\}$$，其中，
$$
SC^{y_j,z_k}_{jk} = \{m \lvert send(m) \in PAST(C) \land rec(m) \in FUTURE(C)\}
$$
$$SC^{y_j,z_k}_{jk}$$的定义说明，任何信道中仅暂存发送的消息，不存在已接收但未发送的消息。在图3中，分割线$$C_1$$对应的全局状态是非一致的，因为接收消息事件$$e^2_2 \in PAST(C_1)$$，使得信道状态$$SC^{e^1_1,e^3_2}_{1,2}$$违背$$SC^{y_j,z_k}_{jk}$$的定义，所以该全局状态是非一致的。同理，分割线$$C_2$$对应的全局状态是一致的。

# 7 事件的过去和未来锥面

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/resources/distrcmp_pas_1-4.svg" width = "75%" alt=""/>
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">
      图4 分布式计算中过去和未来锥面的示意图
  	</div>
</center>

## 7.1 过去锥面

在分布式计算中，事件$$e_j$$应该只收到与其有因果依赖关系的事件$$e_i$$影响，即$$e_i \to e_j$$，那么所有$$e_i$$可用的信息均可以被$$e_j$$访问到，所有这样的事件$$e_i$$都属于$$e_j$$的过去事件。

令$$Past(e_j)$$表示在计算$$(H,\to)$$中$$e_j$$的过去事件，即，
$$
Past(e_j) = \{e_i \lvert\ \forall e_i \in H,\ e_i \to e_j\}
$$
令$$Past_i(e_j)$$表示进程$$p_i$$上所有属于$$Past(e_j)$$事件的集合，$$max(Past_i(e_j))$$则是进程$$p_i$$上最新的影响$$e_j$$的事件。

> $$max(Past_i(e_j))$$总是一个消息发送事件。

令$$Max\_Past(e_j) = \cup_{\forall i} \{max(Past_i(e_j))\}$$，$$Max\_Past(e_j)$$包含每个进程上影响$$e_j$$的最新事件，其被称之为事件$$e_j$$的**过去锥面**。

> $$Max\_Past(e_j)$$对应了一条一致性分割线。因为$$Max\_Past(e_j)$$内所有事件都为发送消息事件，而且紧贴锥面前的所有信道状态都为$$\varnothing$$，所有不存在没有发送事件的接收事件，因此这是一条一致性分割线。

## 7.2 未来锥面

令$$Future（e_j）$$表示事件$$e_j$$的未来事件，它包含所有受到$$e_j$$影响的事件$$e_i$$。在计算$$(H,\to)$$中，$$Future(e_j)$$定义为
$$
Future(e_j) = \{e_i \lvert \ \forall e_i \in H,\ e_j \to e_i\}
$$
$$Future_i(e_j)$$则表示进程$$p_i$$上所有受$$e_j$$影响的事件的集合，$$min(Future_i(e_j))$$是进程$$p_i$$上受$$e_j$$影响的首个事件。

> $$min(Future_i(e_j))$$为消息接收事件。

令$$Min\_Past(e_j) = \cup_{\forall i}\{min(Future_i(e_j))\}$$，它包含了每个进程上受事件$$e_j$$影响的第一个事件，这样的集合被称为事件$$e_j$$的**未来锥面**。

>$$Min\_Past(e_j)$$对应了一条一致性分割线。

在图4中，如果进程$$p_i$$上某个事件发生在$$max(Past_i(e_j))$$和$$min(Future_i(e_j))$$之间，那么该事件与事件$$e_j$$是并发的。
> 因为该事件既不影响$$e_j$$，也不受$$e_j$$影响，所以同$$e_j$$是并发关系。
>
> 集合$$H - Past(e_j) - Future(e_j)$$内的所有事件都与事件$$e_j$$是并发的。


---
[^1]: https://zh.wikipedia.org/wiki/%E5%81%8F%E5%BA%8F%E5%85%B3%E7%B3%BB

