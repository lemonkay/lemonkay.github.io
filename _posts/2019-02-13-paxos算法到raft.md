---
    layout: post
    title: Paxos算法到Raft
---

## paxos算法
- Paxos算法描述：Paxos算法分为两个阶段。具体如下：

    1. 阶段一：

        * a. Proposer选择一个提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。

        * b. 如果一个Acceptor收到一个编号为N的Prepare请求，且N大于该Acceptor已经响应过的所有Prepare请求的编号，那么它就会将它已经 接受过的编号最大的提案（如果有的话）作为响应反馈给Proposer，同时该Acceptor承诺不再接受任何编号小于N的提案。

    2. 阶段二：

        * a.如果Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，那么它就会发送一个针对[N,V]提案的 Accept请求给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的value，如果响应中 不包含任何提案，那么V就由Proposer自己决定。

        * b.如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor没有对编号大于N的Prepare请求做出过 响应，它就接受该提案。

-  Lamport觉得同行无法接受他的幽默感，于是用容易接受的方法重新表述了一遍。

 > * Phase 1.   
        (a) A proposer selects a proposal number n and sends a prepare request with number n to a majority of acceptors.  
        (b) If an acceptor receives a prepare request with number n greater than that of any prepare request to which it has already responded, then it responds to the request with a promise not to accept any more proposals numbered less than n and with the highest-numbered pro- posal (if any) that it has accepted.

 > * Phase 2.   
       (a) If the proposer receives a response to its prepare requests (numbered n) from a majority of acceptors, then it sends an accept request to each of those acceptors for a proposal numbered n with a value v, where v is the value of the highest-numbered proposal among the responses, or is any value if the responses reported no proposals  
       (b) If an acceptor receives an accept request for a proposal numbered n, it accepts the proposal unless it has already responded to a prepare request having a number greater than n.


- [wiki的描述](https://zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95)


## raft算法
- 论文地址：[raft](../file/raft.pdf)

- raft的一些应用：
    * redis sentinel和cluster都是基于raft的leader election。

    * nacos-naming，多服务也有raft的实现，**@see RaftConsistencyServiceImpl**

    * neo4j的因果集群模式，core server

- [raft协议](https://raft.github.io/)



## 其它衍生
- zookeeper 的zab
- kafka isr的多数选举