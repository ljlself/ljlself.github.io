---
layout: article
title: Zookeeper Atomic Broadcast Protocol Introduction
---

# Zookeeper Atomic Broadcast Protocol Introduction

## ------Discovery, Synchronisation and Broadcast

### 概念
-----

#### Epoch:
对于一个Zookeeper集群，一个Leader从开始建立领导到领导结束，称为一个Epoch，也可称为一个周期。在一个周期之中，有且仅有一个Leader。记做(e,e’,e’’)等。  
在Zookeeper中，每次进入一个新的Epoch，Epoch的值便会进行递增，存在严格的次序关系。如果一个Epoch e早于另一个Epoch e’，记做（e⧼e’）。
  

#### Primary Process:
用于产生并分发Transaction。通常的实现是将Primary Process和Leader用同一个进程表示，因此在一定的情况下可以把二者看做是同一样事物。记做（ ρ ）。
  

#### Zxid:
即transaction identifier。用于实现transaction的有序特性。记做（ z ）。  
zxid用epoch和counter对表示，即 z=<e,c>  
z.epoch代表当前Primary Process或者说是Leader的epoch，z.counter代表当前Primary   Process分发的transaction的序号。每次产生新的transaction的时候，z.counter就会递增。  
当开始一个新的epoch时，即新的Leader进行领导时，e依据之前的epoch进行递增，而c则归零。  
例如有两个zxid，<e,z>和<e’,c’> 。当e<e’或者e=e’,c<c’我们成<e,c>在<e’,c’>之前。记做（<e,c> ⧼ <e’,c’>）
  

#### Transaction
表示一个事务。记做<v,z>。
v 表示事务的值即事务的具体内容，而 z 则是事务的zxid。事务由zxid进行区分。

#### Peer:
Peer是Follower和Leader的统称，集群中每一个节点都称为一个Peer。每个Peer都由以下结构体定义。

	struct state{
     	history  h.f;
     	acceptedEpoch  f.p;
     	currentEpoch  f.a;
     	lastZxid  f.zxid ;
	}peerState;	

* history：表示当前Peer已经接受过的transaction的集合。
* acceptedEpoch：表示已经接受过的最大的Epoch。
* currentEpoch:表示接受的新Leader的Epoch。
* lastZxid：表示最近处理的transaction的zxid。

#### Quorum:
[2]中对Quorum做出了如下定义：

> A system comprises a set of processes Π = {p , p2 , ...... , pn }    
> We assume that a quorum system Q is deﬁned a priori, and that Q satisﬁes the following:
> 
>  ∏ = {p1, p2 , ……, pn}    
>  ⋀   ∀Q ∈ 𝒬 ， Q ∈ ∏   
>  ⋀   ∀Q1，Q2∈ 𝒬  ，Q1∩Q2 ≠ ∅  

用通俗的话来说Quorum是指集群中的大多数节点的集合。大多数即指包含的节点数超过集群总节点数的一半。
得到来自Quorum的消息，即认为得到了集群中超过半数Follower的消息。

#### 三种状态：

#### ELECTION：
- 当进程启动后，进入ELECTION阶段，试图成为一个Leader或者选举出一个Leader。此时选举出的Leader只是称为Perspective Leader。并不是真正建立领导的Leader，Leader真正建立领导是在Synchronisation过后。
- 当Follower发现当前Leader挂了或者放弃了领导，那么进入ELECTION阶段。
- 当Leader发现接受领导的Follower不能够组成一个合适的集群时，便进入ELECTION阶段。

#### LEADING：
- 进程被选举成为Leader，那么它就进入了LEADING阶段。

#### FOLLOWING：
- 进程找到了一个选出来的Leader，那么它就进入FOLLOWING阶段，并开始追随该Leader。在该阶段的进程称为Follower。

### 过程
---

- （Phase 0：Election）
- Phase 1：Discovery
- Phase 2：Synchronisation
- Phase 3：Broadcast

Phase 0 : Election阶段

- Peer初始化后进入该阶段。在这个阶段中，Peer进入到Election状态，并运行Leader选举算法，算法执行结束后，Peer成为Leader或者Follower。
- 在这个过程中，有投票过程。结果会出现Follower和Leader（Perspective）进入下一阶段。

Phase 1 : Discovery阶段

- 该阶段Follower与Leader进行通信
- 在该阶段开始的时候，一个Follower会对Leader发起一个Leader-Follower连接，一个Follower一次只能连接一个Leader。
- 如果发生连接被拒绝或者发生任何错误，都会使得对应的Follower重新进入phase0。

![zab discovery](/image/zab_protocal_introduction/zab_discovery.jpg)

1. Follower将自己的acceptedEpoch通过一个称为FOLLOWERINFO的消息发送给Election阶段选出的Leader。
2. 当Leader获取到来自Quorum的acceptedEpoch(记做e)后，生成一个新的Epoch（记做e’），e’满足以下 条件，e’ 大于任何一个由Follower发来的e。
3. Leader将e’通过一个称为NEWEPOCH的消息发送给Quorum。
4. Follower收到来自Leader的NEWEPOCH消息之后， 进行判断
    - 若e’>acceptedEpoch，则将e’作为acceptedEpoch的值。
    - 若e’<acceptedEpoch，进入重新选举状态。
5. 根据4的判断条件
    - 符合e’>acceptedEpoch，则将e’通过消息ACKEPOCH发送给Leader。ACKEPOCH消息中包含Follower的currentEpoch，history和lastZxid。
    - 符合e’<acceptedEpoch，则返回election状态，重新进行leader选举。
6. Leader收到来自Quorum的ACKEPOCH消息，根据ACKEPOCH的内容找出一个Follower。该Follower满足以下条件
    - currentEpoch最大
    - 若currentEpoch存在相同的，则按照lastZxid的先后次序，取大者。
7. Leader将找出的Follower的History作为自己的History. 进入Phase 2。


Phase 2：Synchronisation

- 同步阶段是Leader根据在Phase 1中获得的History，与Quorum进行同步的过程。
-  Leader将自己History中的Transaction发送给Follower，Follower收到之后若发现该Transaction自己没做过，则接受该Transaction。接受之后发送消息给leader。
- 若Leader发现来自Quorum的确认消息，则发送Commit消息给Quorum。
- Quorum接受到Commit消息之后执行Transaction。
- 该阶段结束我们认为该leader已经成为集群的真正leader了。

![zab synchronisation](/image/zab_protocal_introduction/zab_synchronisation.jpg)

1. Leader经过Phase 1后，已经获得了符合条件的Follower的History。将Epoch e’和History封装在NEWLEADER消息中发送给Follower。
2. Follower收到Leader发送过来的NEWLEADER消息
    - 若acceptedEpoch = e’，则将History中包含的未完成过的Transaction按照zxid的次序一件一件全部接受，即与Leader的History进行同步，将History写入自身的History。
    - 若acceptedEpoch != e’，则返回election状态，重新进行leader选举。
3. 根据2的判断条件，
    - 若acceptedEpoch = e’，Follower完成之后进行发送一个ACKNEWLEDER消息给Leader。
    - 若acceptedEpoch != e’，进入election状态，重新进行leader选举。
4. Leader接受到来自Quorum发来的ACKNEWLEADER消息之后，发送COMMIT消息给所有Follower，进入Phase 3。
5. Follower收到COMMIT消息之后，依据zxid的次序执行记录在案单尚未执行的Transaction。该步骤与之前记录的步骤类似于两阶段的提交的过程。完成之后进入Phase 3。

Phase 3 : Broadcast

- 如果不发生任何错误，那么整个集群就会一直处于该阶段。
- 该阶段集群接受Zookeeper客户端的请求，并把请求作为广播发出去。
- 该阶段刚开始的时候，quorum会保持一致性，即集群中只会存在一个leader。与此同时，leader会允许新的follower加入集群。新节点会更新该epoch的transaction来与其他节点保持一致。
- 进入Phase 3阶段才能够真正为应用提供服务，进入Phase 3后会调用ready(e)来表明已经准备好了。

![zab broadcast](/image/zab_protocal_introduction/zab_broadcast.jpg)

1. Leader调用ready(e’)表明已经准备完毕。e’为Leader的Epoch。
2. Client向Leader发送请求v。
3. Leader收到Client的请求，生成Transaction <e’,<v,z>>，其中z=<e’,c>。其中隐含Epoch e’周期中z之前的Transaction已经成功完成。c是所有已经完成的Transaction的counter的增值。
4. Follower接收来自Leader发来的Transaction。将该Transaction记录到自己的History中。
5. Follower返回ACK消息给Leader。
6. Leader收到Quorum返回的ACK消息，发送COMMIT消息给Follower。
7. Follower接受到COMMIT消息之后，等待还没有记录完毕但在z之前的Transaction<e’,z’>记录完毕，再执行Transaction<e’,<v,z>>。
8. 新Follower此时想加入集群（与Discovery和Synchronisation阶段加入集群相同）。首先发送FOLLOWERINFO给Leader。
9. Leader收到来自新Follower的FOLLOWERINFO。发送NEWEPOCH(e’)和NEWLEADER(e’，History)消息给新Follower。
10. 新Follower收到来自Leader的消息，返回ACKNEWLEADER消息。
11. Leader收到新Follower的ACKNEWLEADER消息，返回COMMIT消息确认把新Follower加入集群。

### 其他
---

- 以上算法都是异步的并且没有考虑节点奔溃等错误情况。
- 针对这种情况，Zab采用的是心跳机制。Follower和Leader之间进行心跳消息的传递。
- 如果Leader在timeout时间之内没有收到Quorum发来的心跳消息，则放弃自己的Leading状态，回到phase0进入重新选举状态。
- 如果Follower在timeout时间之内没有收到Leader发来的心跳消息，则直接进入phase0阶段进行Election。

### 参考文献
---
[1] Andre Medeiros. ZooKeeper’s atomic broadcast protocol: Theory and practice. 2012.  
[2] Flavio P. Junqueira, Benjamin C. Reed, and Marco Seraﬁni. Zab: High-performance broadcast for primary-backup systems.  
[3] Benjamin Reed. Apache’s Wiki page of a Zab documentation, January 2012. https://cwiki.apache.org/confluence/display/ZOOKEEPER/Zab1.0.