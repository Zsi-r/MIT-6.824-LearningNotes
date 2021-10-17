# MIT-6.824-LearningNotes
Paper and notes about Distributed System.

![duq7ofg2ra](img/duq7ofg2ra.jpeg)



## 1. Raft算法

> 大部分笔记都记录在Raft论文里了。以下只是raft的部分补充笔记。

### 1.1 Raft的4个主要内容（leader选举、日志复制、安全性、日志压缩）：

- **Leader Election领带人选举**：Leader宕机后选取新的Leader

- **Log Replication日志复制**

- **Safety安全性**：

  - 日志只能从领导人流向Follower
  - 只有**当前Leader任期**的日志条目才能通过计算数目来进行提交。为了提交之前任期日志条目，只能通过在当前任期新commit了一个当前任期的日志条目，才能把在此之前的日志条目一并提交

- **成员变更**：join consensus共同一致

- **日志压缩**：snapshot快照。

  - **何时发送InstallSnapshotRPC？**

    =>当Leader发送AppendEntryRPC是，发现目标Follower的nextIndex比自己第一个log的index还小时，就把自己的Snapshot发送给该Follower

### 1.2 如何解决脑裂？



## 2. Multi-Raft算法

- 多个Raft集群，每个集群一个数据库分片，以达到负载均衡。多个Raft集群可以协同以减少资源开销
- Multi-Raft --> **TiDB、CockroachDB、PolarDB**
- Parallel-Raft是PolarDB中的Multi-Raft实现，通过支持乱序日志复制（乱序确认、乱序提交、乱序应用）等手段来提升性能。



## 3. Paxos协议

- 实际上Paxos 其实是一类协议，Paxos 中包含 Basic Paxos、Multi-Paxos、Cheap Paxos 、Egalitarian Paxos (EPaxos) 和其他的变种
- Raft 是 Multi-Paxos的简化 

### 3.1 Basic-Paxos算法

- 就**一个**提案达成一致
- 与Basic-Paxos相比，Raft是强leader机制，必须要有leader

### 3.2 Multi-Paxos算法

- 就**一批**提案成一致
- 若用Basic-Paxos一次提交一个提案的效率不高：
  - 因为每个提案都需要至少3次网络IO：产生logID、prepare、accept
- **所以Multi-Paxos选出唯一的Leader**，提案只能由于leader发起
- 所以省掉：产生logID阶段和prepare阶段，直接执行accept，得到多数派确认即表示提案达成一致

#### 3.2.1 Raft和Multi-Paxos的区别

- 两点区别：
  - Raft的append 操作必须是连续的， 而Paxos可以并发 (这里并发只是append log的并发, 应用到状态机还是有序的)。
  - Raft必须有leader，paxos可以没有leader
  - Raft选主有限制，必须包含最新、最全日志的节点才能被选为leader。而Multi-Paxos没有这个限制，日志不完备的节点也能成为leader。
  - Raft相当于是Paxos的悲观版本，直接选择了Leader的方式，利用用Term和成员管理去消除很多冲突。Paxos愿意去乐观解决冲突
- Raft可以看成是简化版的Multi-Paxos。
- Multi-Paxos允许并发的写log，所以当leader节点故障后，剩余节点有可能都有日志空洞。所以选出新leader后, 需要将新leader里没有的log补全,在依次应用到状态机里。
- 总的来说Raft只是Paxos实现的一种，但是很明显，Raft身上的限制比较多，而Paxos在算法层面上更通用，所以相比于Raft，在Paxos的基础上进行优化，上限应该会更高

### 3.3 总结

#### 3.3.1 租约

- 解决如何选主如何换届的问题
- 使用心跳？----> 容易因网络拥塞产生脑裂
- 使用etcd/zookeeper作为租约颁发节点
- 每次租约时长内只有一个节点获得租约，到期后必须重新颁发租约
- leader宕机后，也只有租约到期后才能重新发起选举

## 4. Zab协议

> [lilomin's blog](https://juejin.cn/post/6844904118117466119)

## 5. NWR协议

- NWR含义：

  - N: 在分布式存储系统中，有多少份备份数据；
  - W: 代表一次成功的更新操作要求至少有W份数据写入成功；
  - R: 代表一次成功的读数据操作要求至少有R份数据成功读取；

- **当 W+R > N的时候，对客户端来讲系统是强一致性的**

- 如当W+R以常见的N=3、W=2、R=2为例：

  - N=3表示任何一个对象都必须有三个副本(Replica)
  - W=2表示对数据的修改操作(Write/Modify)只需要在3个Replica中的2个上面完成就返回；
  - R=2表示从三个对象中要读取到2个数据对象才能返回；
- 在分布式系统中，数据的单点是不允许存在的，即数据的replica不能为1

## 6. CAP模型

#### 6.1 CA模型（牺牲分区容错性）

- 例子：
  - 单站点数据库、集群数据库、LDAP、xFS文件系统
- 实现方式：
  - 两阶段提交、缓存验证协议

#### 6.2 CP模型（牺牲可用性）

- 例子：
  - 分布式数据库、分布式锁、绝大部分协议
- 实现方式：
  - 悲观锁、少数分区不可用

#### 6.3 AP模型（牺牲一致性）

- 例子：
  - Coda、Web缓存、DNS
- 实现方式：
  - 到期/租赁、解决冲突、乐观锁

#### 6.4 最新发展

- 2012年Eric Brewer的文章指出CAP三选二是错误的
- 理由：
  - 分区很少发生，没什么理由在不发生分区的时候牺牲C或A
  - C和A之间的取舍可以以非常细小的粒度反复发生，如每一次具体的操作、或特定的数据和用户
  - C、A、P是一定程度上衡量，而不是非黑即白：C分成很多级别，A是在0%~100%之间变化的，P有不同含义

### 7. BASE模型

- Base Available：基本可用
- Soft state：软状态、柔性事务，即允许系统中的数据存在中间状态
- Eventual consistency：最终一致性
- 源于电子商务，是反ACID的

## 8. 一致性模型

> [cwen's blog](https://int64.me/2020/%E4%B8%80%E8%87%B4%E6%80%A7%E6%A8%A1%E5%9E%8B%E7%AC%94%E8%AE%B0.html)
>
> [Irita's blog](https://lrita.github.io/2019/10/12/consistency-models/)

- 从强到弱分为：
  - 线性一致性Linearizability consistency ，也叫原子性
  - 顺序一致性 Sequential consistency
  - 因果一致性 Causal consistency
  - 最终一致性 Eventual consistency
- <img src="img/jepsen.png" alt="image.png" style="zoom: 80%;" />
- 其中线性...和顺序...是强一致性，其它的是弱一致性
- 强一致性：保证看见时一定是一致的状态

## 9. 分布式事务

### 9.1 两阶段提交 2PC

### 9.2 三阶段提交 3PC

### 9.3 基于消息的分布式事务

#### 9.3.1 本地消息表 ebay

#### 9.3.2 事务消息 RocketMQ