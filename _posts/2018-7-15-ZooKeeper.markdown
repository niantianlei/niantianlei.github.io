---
layout:     post
title:      "ZooKeeper简介及ZAB协议原理解读"
subtitle:   " \"Interpretation of the principle of ZAB protocol and ZooKeeper\""
author:     "Nian Tianlei"
header-img: "img/post-bg-2016.jpg"
header-mask: 0.4
catalog:    true
tags:
    - 分布式
---

## ZooKeeper简介
ZooKeeper为分布式应用提供了高效且可靠的分布式协调服务，具有高可用、高性能、严格的顺序访问控制能力等特点。  

没有沿用传统的Master/Slave模型，而是引入Leader、Follower和Observer三种角色。ZooKeeper集群中的所有机器通过一个Leader选举过程来选定一台被称为“Leader”的机器，Leader服务器为客户端提供读和写服务。此外，Follower和Observer都能提供读服务，唯一的区别在于，Observer不参与Leader选举过程，也不参与写操作的“过半写成功”策略，因此Observer可以在不影响写性能的情况下提升集群的读性能。  

**特点：**
ZooKeeper将全量数据存储在内存内，来提高服务器吞吐、减少延迟。  
只要集群中存在超过一半的机器能够正常工作，那么整个集群就可以正常对外提供服务。  
对于来自客户端的每个请求，ZooKeeper都会分配一个全局唯一的递增编号，反映了所有事务操作的先后顺序。  

## ZAB协议
之前写了个文章介绍了Paxos协议，但ZooKeeper并没有完全使用Paxos，而是使用ZooKeeper Atomic Broadcast（ZAB，原子广播协议）作为保障数据一致性的核心算法。  

使用一个单一的主进程来接收兵处理客户端的所有事务请求，并采用ZAB的原子广播协议，将服务器数据的状态变更以事务Proposal的形式广播到所有的副本进程上去。  
考虑到执行顺序，前后可能存在一定的依赖关系，有这样一个要求：ZAB协议必须能够保证一个全局的变更序列被顺序应用，也就是说，ZAB协议需要保证如果一个状态变更已经被处理了，那么所有其依赖的状态变更都应该已经被提前处理掉了。  

**核心：**  
整个zookeeper集群中只有一个节点即Leader将客户端的写操作转化为事务(或提议proposal)。Leader节点再数据写完之后，将向所有的follower节点发送数据广播请求(或数据复制)，等待所有的follower节点反馈。在ZAB协议中，只要超过半数follower节点反馈OK，Leader节点就会向所有的follower服务器发送commit消息。即将leader节点上的数据同步到follower节点之上。  

ZAB协议包括两种基本的模式，分别是崩溃恢复和消息广播。  
Leader服务器出现异常（网络中断、崩溃退出、重启等），或不存在过半服务器与leader保持正常通信时，会进入恢复模式，并选举出新的Leader服务器。当集群中有过半机器与新Leader服务器完成了状态同步后，ZAB就会退出恢复模式。  

集群中有过半的Follower服务器完成了和Leader服务器的状态同步，会进入消息广播模式。  
当新加入一个服务器时，会找到Leader，进行同步数据，然后加入到消息广播中。  
当集群中的非leader收到事务请求后，会将事务请求转发给leader。  

#### 消息广播
消息广播过程使用原子广播协议，类似二阶段提交的过程。区别是移除了中断逻辑，当过半Follower反馈确认后，就开始提交事务。  
但这种处理方法无法处理leader崩溃导致的数据不一致的问题，因此需要崩溃恢复解决。  
此外，消息广播是基于具有FIFO特性的TCP协议，可以保证消息广播过程中消息接收和发送的顺序性。  

所有事物请求必须由一个称为Leader的服务器处理，分配一个全局递增的事务id将事务请求转换成一个事务提议，并将提议发送给所有Follower的队列中，follower取出消息进行处理，向leader发送ACK信号。得到超过一半数量的Follower正确反馈后，会向所有Follower发送commit请求，提交事务。  

#### 崩溃恢复
恢复需满足下面两个要求：  
1.能够保证提交已经被Leader提交的事务提议  
2.能够丢弃只在leader被提出的事务提议   
因此选举出来的leader服务器拥有集群中所有机器最高事务id的提议。   

完成leader选举后，开始数据同步。  
leader将没有同步的事务以提议的形式发给follower，并紧跟着一个commit请求，直至同步完成。  
对于需要被丢弃提议的处理。  
事务ZXID是一个长度64位的数字，其中低32位单调递增，即每当leader产生一个新的事务,低32位加1。每当选举出一个新leader时，新的leader就从本地事务日志中取出ZXID,然后解析出高32位（称为epoch，代表leader周期），进行加1，再将低32位的全部设置为0。这样就保证了每次新选举的leader后，保证了ZXID的唯一性而且是保证递增的。


## Paxos与ZAB
Paxos致力于一致性的分布式系统，而ZAB设计目标是构建高可用的分布式主备系统。  
两者都是超过半数的节点确认后，再提交事务。  
ZAB添加了一个同步阶段  