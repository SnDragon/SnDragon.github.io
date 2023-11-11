---
title: Paxos算法
date: 2023-05-06 11:10:44
tags: ["分布式协议", "Paxos"]
---
Paxos算法是Leslie Lamport于1990年提出的一种基于消息传递且具有高度容错特性的**共识（consensus）**算法。
包含 2 个部分：
* 一个是 Basic Paxos 算法，描述的是多节点之间如何就某个值（提案 Value）达成共识；
* 另一个是 Multi-Paxos 思想，描述的是执行多个 Basic Paxos 实例，就一系列值达成共识

## Basic Paxos
### 问题
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/e3753a4a-9f88-4a7e-9d02-7cb66bf6a10c.jpg" />

有多个客户端（比如客户端 1、2）访问系统，试图创建同一个只读变量（比如 X），客户端 1 试图创建值为 3 的 X，客户端 2 试图创建值为 7 的 X，要如何达成共识，实现各节点上 X 值的一致？

<!-- more -->
### 三种角色
在 Basic Paxos 中，有提议者（Proposer）、接受者（Acceptor）、学习者（Learner）三种角色，他们之间的关系如下：
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/3a1f1d35-9e42-4d03-9ca4-a8ded0b752a6.jpg" />


* 提议者（Proposer）：提议一个值，用于投票表决。一般是集群中收到客户端请求的节点，发起**二阶段提交**，进行共识协商；
* 接受者（Acceptor）：对每个提议的值进行投票，并存储接受的值，比如 A、B、C 三个节点。 一般来说，集群中的所有节点都在扮演接受者的角色，参与共识协商，并接受和存储数据。
* 学习者（Learner）：被告知投票的结果，接受达成共识的值，存储保存，不参与投票的过程。一般来说，学习者是数据备份节点，比如“Master-Slave”模型中的 Slave，被动地接受数据，容灾备份。

### 重点
1. Basic Paxos 是通过**二阶段提交**的方式来达成共识的。
2. 除了共识，Basic Paxos 还实现了**容错**，在少于一半的节点出现故障时，集群也能工作。它不像分布式事务算法那样，必须要所有节点都同意后才提交操作，因为“所有节点都同意”这个原则，在出现节点故障的时候会导致整个集群不可用。也就是说，“大多数节点都同意”的原则，赋予了 Basic Paxos 容错的能力，让它能够容忍少于一半的节点的故障。
3. 本质上而言，提案编号的大小代表着优先级，根据提案编号的大小，接受者保证三个承诺，具体来说：如果准备请求的提案编号，小于等于接受者已经响应的准备请求的提案编号，那么接受者将承诺不响应这个准备请求；如果接受请求中的提案的提案编号，小于接受者已经响应的准备请求的提案编号，那么接受者将承诺不通过这个提案；如果接受者之前有通过提案，那么接受者将承诺，会在准备请求的响应中，包含已经通过的最大编号的提案信息。

### 例子
#### 准备（Prepare）阶段
在准备请求中不需要指定提议的值，只需要携带提案编号即可。
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/d5835844-a9e8-42d3-a473-916c77eda179.jpg" />
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/14f8c3b2-92c2-411e-aade-5c9400a30964.jpg" />
#### 接受（Accept）阶段
客户端 1、2 在收到大多数节点的准备响应之后，会分别发送接受请求：
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/b28ad373-fc7e-4e46-93f7-431b3936caee.jpg" />

<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/fd39aa35-48cc-4a80-a8b4-8d3747985200.jpg" />

> 如果集群中有学习者，当接受者通过了一个提案时，就通知给所有的学习者。当学习者发现大多数的接受者都通过了某个提案，那么它也通过该提案，接受该提案的值

## Multi Paxos
兰伯特提到的 Multi-Paxos 是一种思想，不是算法。而 Multi-Paxos 算法是一个统称，它是指基于 Multi-Paxos 思想，通过多个 Basic Paxos 实例实现一系列值的共识的算法（比如 Chubby 的 Multi-Paxos 实现、Raft 算法等）

### 问题
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/45fa7666-1a73-4f21-a739-bccc41348353.jpg" />

如果我们直接通过多次执行 Basic Paxos 实例，来实现一系列值的共识，就会存在这样几个问题：
* 如果多个提议者同时提交提案，可能出现因为提案编号冲突，在准备阶段没有提议者接收到大多数准备响应，**协商失败**，需要重新协商。如，一个 5 节点的集群，如果 3 个节点作为提议者同时提案，就可能发生因为没有提议者接收大多数响应（比如 1 个提议者接收到 1 个准备响应，另外 2 个提议者分别接收到 2 个准备响应）而准备失败，需要重新协商。
* 2 轮 RPC 通讯（准备阶段和接受阶段）往返消息多、耗**性能**、延迟大。

### 解决方案
#### 领导者
通过引入领导者节点，领导者节点作为唯一提议者，这样就不存在多个提议者同时提交提案的情况，也就不存在提案冲突的情况了：
<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/36c8cb5a-2eaa-42b4-9fba-6bd99b5c452f.jpg" />

> 在论文中，兰伯特没有说如何选举领导者，需要我们在实现 Multi-Paxos 算法的时候自己实现。 比如在 Chubby 中，主节点（也就是领导者节点）是通过执行 Basic Paxos 算法，进行投票选举产生的。

#### 优化 Basic Paxos 执行
当领导者处于稳定状态时，省掉准备阶段，直接进入接受阶段”这个优化机制，优化 Basic Paxos 执行。也就是说，领导者节点上，序列中的命令是最新的，不再需要通过准备请求来发现之前被大多数节点通过的提案，领导者可以独立指定提案中的值。这时，领导者在提交命令时，可以省掉准备阶段，直接进入到接受阶段：

<img src="https://longerwu-1252728875.cos.ap-guangzhou.myqcloud.com/blogs/fe28d489-62ef-483c-b40c-3bd2af98dcca.jpg" />

### 注意点
Basic Paxos 是经过证明的，而 Multi-Paxos 是一种思想，缺失实现算法的必须编程细节，这就导致，Multi-Paxos 的最终算法实现，是建立在一个未经证明的基础之上的，正确性是个问号。

在实际使用时，不推荐你设计和实现新的 Multi-Paxos 算法，而是建议优先考虑 Raft 算法，因为 Raft 的正确性是经过证明的。