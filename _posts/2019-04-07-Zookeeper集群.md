---

layout: post
title: ZooKeeper集群
categories: JAVA进阶
description: Zookeeper集群相关概念。
keywords: Zookeeper

---

## Zookeeper集群

### 1.集群管理

集群管理主要是干两件事：是否有机器加入和推出，选举leader。

- 所有机器约定在父目录GroupMenbers下创建临时目录节点，然后监听父目录节点的子节点变化消息（watcher）。一旦有机器挂掉，该机器与ZK的连接断开，临时节点也自动删除，此时其他机器都会收到通知；
- 选举leader，所有机器创建临时顺序节点，根据选举算法选出leader。

### 2.工作原理

#### 2.1 集群中的角色和状态

在ZK的集群中，每个节点共有下面3种角色和4种状态：

- 角色：leader，follower，observer（follower和observer统称为learner）

  - leader：负责所有事务写操作（会话状态变更及数据节点变更操作），保证集群事务处理的顺序性，默认设置下，也处理读请求。

    恢复数据，维持与learner的心跳，接收learner请求并判断客户端请求类型；

  - follower：处理客户端非事务请求，转发事务请求到leader服务器；参与leader选举投票，参与事务操作的“过半通过”投票策略。

    向leader发送请求，接收leader消息并进行处理，接收客户端请求（如果为写请求，发送给leader进行投票）；

  - observer：只提供读取服务。在不影响写性能的情况下提升集群读取性能。不参与任何形式的投票。

- 状态：looking，leading，following，observing

  - looking：当前server不知道leader是谁，正在搜寻；

  - leading：当前server即为选举出来的leader；
  - following：leader已经选举出来，当前server与之同步；
  - observing：不参与选举与投票，仅仅接受（observing）选举和投票的结果。

#### 2.2 Zookeeper原子消息广播协议（Zookeeper Atomic Broadcast protocol，ZAB协议）

ZK主要根据ZAB协议实现分布式集群系统数据一致性。

ZAB协议的核心是在整个ZK集群中只有一个节点即leader将客户端的写操作转化为事务（或提议proposal）。leader节点在数据写完之后，将向所有的follower节点发送数据广播请求（数据复制），等待所有的follower节点反馈，只要超过半数follower节点反馈ok，leader节点就会向所有的follower服务器发送commit消息，也就是将leader节点上的数据同步到follower节点上。

Zab协议**原子广播**有两种模式：恢复模式（Recovery选主）和广播模式（Broadcase同步）。

- **恢复模式：**

  当服务启动或者leader奔溃后，ZAB进入恢复模式，选举出leader，且大多数server完成了和leader的状态同步之后，恢复模式结束，状态同步保证了leader和server具有相同的系统状态。

  ZAB协议奔溃恢复需要满足两个要求：

  - 确保已经被leader提交的proposal必须最终被所有的follower提交；
  - 确保已经被leader提出但是没有提交的proposal被丢弃；

  也就是说，新选举出的leader不能包含未提交的proposal。

- **广播模式：**

  ZK中数据副本的传递策略就是广播模式。

  - 客户端发起一个写操作；

  - leader服务器将客户端的request请求转化为事务proposal提议，同时为每个提议分配一个全局唯一的id，即ZXID；
  - leader服务器与每个follower之间都有一个队列，FIFO，leader将消息发送到该队列；
  - follower服务器从队列中取出消息处理完（写入本地事务日志中）之后，向leader服务器发送ACK确认；
  - leader服务器收到半数以上的follower的ACK之后，即认为可以发送commit（通过quorums法定人数确定）；
  - leader向所有的follower服务器发送commit消息。

#### 2.3 ZXID

**全局唯一，保证顺序。**

leader通过ZAB中的zxid决定事务的编号，从而保证**事务处理的顺序性**。每个事务请求，ZK都会为其分配一个全局唯一的事务id，即ZXID，是一个64位的数字，高32位表示该事务发生的集群选举周期（集群每发生一次leader选举，值加1），低32位表示该事务在当前选择周期内的递增次序（leader每处理一个事务请求，值加1，发生一次leader选举，低32位清零）。

### 3.leader选举

当服务刚启动，leader奔溃，follower不足半数时，会进行leader选举，此时不对外提供服务。选举策略基于basic paxos算法，有LeaderElection和FashLeaderElection两种，默认使用FastLeaderElection算法完成。

basic paxos算法流程如下：

- 发起选举的线程由当前server的线程担任，主要功能是对投票结果进行统计，并选出推荐的server；
- 选举线程首先向所有server发起一次询问（包括自己）；
- 选举线程收到回复后，验证是否是自己发起的询问（验证ZXID是否一致），然后获取对方id，并存储到当前询问对象列表中，最后获取对方提议的leader相关信息（id，zxid），并将这些信息存储到当此选举的投票记录表中；
- 收到所有server回复以后，取出ZXID最大的server，并将这个server相关信息设置成下一次要投票的server；
- 线程将当前最大的server设置为当前server要推荐的leader，如果此时获胜的server获得n/2+1的server票数，设置当前推荐的leader为获胜的server，将根据获胜的server相关信息设置自己的状态，否则继续这个过程，直到leader被选举出来。

fast paxos流程是在选举过程中，某server首先向所有server提议自己要成为leader，当其他server收到提议后，解决epoch和zxid冲突，并接受对方的提议，然后向对方发送接收提议完成的消息，重复这个流程，选举出leader。

在恢复模式下，如果是刚从奔溃状态恢复的或者刚启动的server还会从磁盘快照中恢复数据和会话信息，ZK会记录事务日志并定期进行快照，方便在恢复时进行状态恢复。

### 4.集群中可能出现的问题

#### 4.1 假死，脑裂

Zookeeper通过心跳判断客户端是否还活着。ZK集群中，每个节点都尝试注册一个象征的临时leader节点，其他没有注册成功的成为follower，并且通过watch机制监控master所创建的临时节点，ZK通过内部心跳机制确定leader的状态。一旦leader出现意外，ZK能很快获悉并且通知其他follower，其他follower在之后做出相关反应，这样就完成了leader的切换。但是心跳出现超时可能是leader挂了，也可能是leader和ZK之间网络通信出现问题，这种情况就是**假死**，leader并未死掉，但是与ZK之间的网络出现问题，导致ZK认为其挂掉了，然后通知其他节点进行切换，这样follower中就有一个成为leader，但是原本的leader并未死掉，这就是**脑裂**。

- 假死：由于网络原因导致心跳超时，认为leader死了，但其实leader还存活着；
- 脑裂：由于假死会发起新的leader选举，选举出一个新的leader，但旧的leader网络又通了，导致出现两个leader。

**主要原因：**

- ZK集群和ZK client判断超时并不能做到完全同步，可能一前一后，如果是级权先于client发现，那就会出现这种问题；
- 在发现并切换后通知各个客户端也可能有先后。

**解决办法：**

- quorums：法定人数，指定一个数目，leader下线后，询问集群中的其他节点，leader是否也已经下线，当达到这个数目，则确认leader已经下线。
- Redundant communications：冗余通信的方式，集群中采用多种通信方式，防止一种通信方式失效导致集群中的节点无法通信；
- Fencing：共享资源的方式，比如能看到共享资源就表示在集群中，能够获得共享资源的锁的就是Leader，看不到共享资源的，就不在集群中。

ZK默认采用Quorums方式，即只有集群中超过半数节点投票才能选举出Leader。这种方式可以确保Leader的唯一性，要么选出唯一的一个Leader，要么选举失败。在ZK中Quorums有两个作用：

- 集群中最少的节点数用来选举Leader保证集群可用；
- 通知客户端数据已经安全保存前，集群中最少数量的节点数已经保存了该数据。一旦这些节点保存了该数据，客户端将通知已经安全保存了，可以继续其他任务。

### 参考：

[zk使用原理](https://www.cnblogs.com/wade-luffy/p/5767811.html)

[Zookeeper的功能以及工作原理](https://www.cnblogs.com/felixzh/p/5869212.html)

[zookeeper（二）常见问题汇总](https://blog.csdn.net/yjp198713/article/details/79400927)

[zookeeper中的ZAB协议理解](https://blog.csdn.net/junchenbb0430/article/details/77583955)