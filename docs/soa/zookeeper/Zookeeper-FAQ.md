## 谈下你对 Zookeeper 的认识？ 

ZooKeeper 是一个分布式的，开放源码的分布式应用程序协调服务。它是一个为分布式应用提供一致性服务的软件，提供的功能包括：配置维护、域名服务、分布式同步、组服务等。

ZooKeeper 的目标就是封装好复杂易出错的关键服务，将简单易用的接口和性能高效、功能稳定的系统提供给用户。

------



## Zookeeper 都有哪些功能？ 

1. 集群管理：监控节点存活状态、运行请求等；

2. 主节点选举：主节点挂掉了之后可以从备用的节点开始新一轮选主，主节点选举说的就是这个选举的过程，使用 Zookeeper 可以协助完成这个过程；

3. 分布式锁：Zookeeper 提供两种锁：独占锁、共享锁。独占锁即一次只能有一个线程使用资源，共享锁是读锁共享，读写互斥，即可以有多线线程同时读同一个资源，如果要使用写锁也只能有一个线程使用。Zookeeper 可以对分布式锁进行控制。

4. 命名服务：在分布式系统中，通过使用命名服务，客户端应用能够根据指定名字来获取资源或服务的地址，提供者等信息。
5. 统一配置管理：分布式环境下，配置文件管理和同步是一个常见问题，一个集群中，所有节点的配置信息是一致的，比如 Hadoop 集群、集群中的数据库配置信息等全局配置

------



## zookeeper负载均衡和nginx负载均衡区别

Nginx是著名的反向代理服务器，zk是分布式协调服务框架，都可以做负载均衡

zk的负载均衡是可以调控，nginx只是能调权重，其他需要可控的都需要自己写插件；

但是nginx的吞吐量比zk大很多，应该说按业务选择用哪种方式

------



## 一致性协议2PC、3PC？

### 2PC

**阶段一：提交事务请求（”投票阶段“）**

当要执行一个分布式事务的时候，事务发起者首先向协调者发起事务请求，然后协调者会给所有参与者发送 `prepare` 请求（其中包括事务内容）告诉参与者你们需要执行事务了，如果能执行我发的事务内容那么就先执行但不提交，执行后请给我回复。然后参与者收到 `prepare` 消息后，他们会开始执行事务（但不提交），并将 `Undo` 和 `Redo` 信息记入事务日志中，之后参与者就向协调者反馈是否准备好了

**阶段二：执行事务提交**

协调者根据各参与者的反馈情况决定最终是否可以提交事务，如果反馈都是Yes，发送提交`commit`请求，参与者提交成功后返回 `Ack` 消息，协调者接收后就完成了。如果反馈是No 或者超时未反馈，发送 `Rollback` 请求，利用阶段一记录表的 `Undo` 信息执行回滚，并反馈给协调者`Ack` ，中断消息

![](https://tva1.sinaimg.cn/large/00831rSTly1gclosfvncqj30hs09j0td.jpg)

优点：原理简单、实现方便。

缺点：

- **单点故障问题**，如果协调者挂了那么整个系统都处于不可用的状态了
- **阻塞问题**，即当协调者发送 `prepare` 请求，参与者收到之后如果能处理那么它将会进行事务的处理但并不提交，这个时候会一直占用着资源不释放，如果此时协调者挂了，那么这些资源都不会再释放了，这会极大影响性能
- **数据不一致问题**，比如当第二阶段，协调者只发送了一部分的 `commit` 请求就挂了，那么也就意味着，收到消息的参与者会进行事务的提交，而后面没收到的则不会进行事务提交，那么这时候就会产生数据不一致性问题



### 3PC

3PC，是 Three-Phase-Comimit 的缩写，即「**三阶段提交**」，是二阶段的改进版，将二阶段提交协议的“提交事务请求”过程一分为二。

**阶段一：CanCommit**

协调者向所有参与者发送 `CanCommit` 请求，参与者收到请求后会根据自身情况查看是否能执行事务，如果可以则返回 YES 响应并进入预备状态，否则返回 NO

**阶段二：PreCommit**

协调者根据参与者返回的响应来决定是否可以进行下面的 `PreCommit` 操作。如果上面参与者返回的都是 YES，那么协调者将向所有参与者发送 `PreCommit` 预提交请求，**参与者收到预提交请求后，会进行事务的执行操作，并将 Undo 和 Redo 信息写入事务日志中** ，最后如果参与者顺利执行了事务则给协调者返回成功的 `Ack` 响应。如果在第一阶段协调者收到了 **任何一个 NO** 的信息，或者 **在一定时间内** 并没有收到全部的参与者的响应，那么就会中断事务，它会向所有参与者发送中断请求 `abort`，参与者收到中断请求之后会立即中断事务，或者在一定时间内没有收到协调者的请求，它也会中断事务

**阶段三：DoCommit**

这个阶段其实和 `2PC` 的第二阶段差不多，如果协调者收到了所有参与者在 `PreCommit` 阶段的 YES 响应，那么协调者将会给所有参与者发送 `DoCommit` 请求，**参与者收到 DoCommit 请求后则会进行事务的提交工作**，完成后则会给协调者返回响应，协调者收到所有参与者返回的事务提交成功的响应之后则完成事务。若协调者在 `PreCommit` 阶段 **收到了任何一个 NO 或者在一定时间内没有收到所有参与者的响应** ，那么就会进行中断请求的发送，参与者收到中断请求后则会 **通过上面记录的回滚日志** 来进行事务的回滚操作，并向协调者反馈回滚状况，协调者收到参与者返回的消息后，中断事务。

![](https://tva1.sinaimg.cn/large/00831rSTly1gclot2rul3j30j60cpgmo.jpg)

降低了参与者的阻塞范围，且能在单点故障后继续达成一致。

但是最重要的一致性并没有得到根本的解决，比如在 `PreCommit` 阶段，当一个参与者收到了请求之后其他参与者和协调者挂了或者出现了网络分区，这个时候收到消息的参与者都会进行事务提交，这就会出现数据不一致性问题。

------



## 讲一讲 Paxos 算法？

`Paxos` 算法是基于**消息传递且具有高度容错特性的一致性算法**，是目前公认的解决分布式一致性问题最有效的算法之一，**其解决的问题就是在分布式系统中如何就某个值（决议）达成一致** 。

在 `Paxos` 中主要有三个角色，分别为 `Proposer提案者`、`Acceptor表决者`、`Learner学习者`。`Paxos` 算法和 `2PC` 一样，也有两个阶段，分别为 `Prepare` 和 `accept` 阶段。

在具体的实现中，一个进程可能同时充当多种角色。比如一个进程可能既是 Proposer 又是 Acceptor 又是Learner。Proposer 负责提出提案，Acceptor 负责对提案作出裁决（accept与否），learner 负责学习提案结果。

还有一个很重要的概念叫「**提案**」（Proposal）。最终要达成一致的 value 就在提案里。只要 Proposer 发的提案被半数以上的 Acceptor 接受，Proposer 就认为该提案里的 value 被选定了。Acceptor 告诉 Learner 哪个 value 被选定，Learner 就认为那个 value 被选定。

**阶段一：prepare 阶段**

1. `Proposer` 负责提出 `proposal`，每个提案者在提出提案时都会首先获取到一个 **具有全局唯一性的、递增的提案编号N**，即在整个集群中是唯一的编号 N，然后将该编号赋予其要提出的提案，在**第一阶段是只将提案编号发送给所有的表决者**。

2. 如果一个 Acceptor 收到一个编号为 N 的 Prepare 请求，如果小于它已经响应过的请求，则拒绝，不回应或回复error。若 N 大于该 Acceptor 已经响应过的所有 Prepare 请求的编号（maxN），那么它就会将它**已经批准过的编号最大的提案**（如果有的话，如果还没有的accept提案的话返回{pok，null，null}）作为响应反馈给 Proposer，同时该 Acceptor 承诺不再接受任何编号小于 N 的提案

   eg：假定一个 Acceptor 已经响应过的所有 Prepare 请求对应的提案编号分别是1、2、...5和7，那么该 Acceptor 在接收到一个编号为8的 Prepare 请求后，就会将 7 的提案作为响应反馈给 Proposer。

**阶段二：accept 阶段**

1. 如果一个 Proposer 收到半数以上 Acceptor 对其发出的编号为 N 的 Prepare 请求的响应，那么它就会发送一个针对 [N,V] 提案的 Accept 请求半数以上的 Acceptor。注意：V 就是收到的响应中编号最大的提案的 value，如果响应中不包含任何提案，那么 V 就由 Proposer 自己决定
2. 如果 Acceptor 收到一个针对编号为N的提案的Accept请求，只要该 Acceptor 没有对编号大于 N 的 Prepare 请求做出过响应，它就通过该提案。如果N小于 Acceptor 以及响应的 prepare 请求，则拒绝，不回应或回复error（当proposer没有收到过半的回应，那么他会重新进入第一阶段，递增提案号，重新提出prepare请求）
3. 最后是 Learner 获取通过的提案（有多种方式）

![](https://tva1.sinaimg.cn/large/00831rSTly1gcloyv70qsj30sg0lc0ve.jpg)

**`paxos` 算法的死循环问题**

其实就有点类似于两个人吵架，小明说我是对的，小红说我才是对的，两个人据理力争的谁也不让谁🤬🤬。

比如说，此时提案者 P1 提出一个方案 M1，完成了 `Prepare` 阶段的工作，这个时候 `acceptor` 则批准了 M1，但是此时提案者 P2 同时也提出了一个方案 M2，它也完成了 `Prepare` 阶段的工作。然后 P1 的方案已经不能在第二阶段被批准了（因为 `acceptor` 已经批准了比 M1 更大的 M2），所以 P1 自增方案变为 M3 重新进入 `Prepare` 阶段，然后 `acceptor` ，又批准了新的 M3 方案，它又不能批准 M2 了，这个时候 M2 又自增进入 `Prepare` 阶段。。。

就这样无休无止的永远提案下去，这就是 `paxos` 算法的死循环问题。



## 谈下你对 ZAB 协议的了解？

ZAB（Zookeeper Atomic Broadcast） 协议是为分布式协调服务 Zookeeper 专门设计的一种支持**崩溃恢复的原子广播协议**。

在 Zookeeper 中，主要依赖 ZAB 协议来实现分布式数据一致性，基于该协议，ZooKeeper 实现了一种主备模式的系统架构来保持集群中各副本之间数据的一致性。

尽管 ZAB 不是 Paxos 的实现，但是 ZAB 也参考了一些 Paxos 的一些设计思想，比如：

- leader 向 follows 提出提案(proposal)
- leader 需要在达到法定数量(半数以上)的 follows 确认之后才会进行 commit
- 每一个 proposal 都有一个纪元(epoch)号，类似于 Paxos 中的选票(ballot)

 `ZAB` 中有三个主要的角色，`Leader 领导者`、`Follower跟随者`、`Observer观察者` 。

- `Leader` ：集群中 **唯一的写请求处理者** ，能够发起投票（投票也是为了进行写请求）。
- `Follower`：能够接收客户端的请求，如果是读请求则可以自己处理，**如果是写请求则要转发给 Leader 。在选举过程中会参与投票，有选举权和被选举权 。**
- **Observer ：就是没有选举权和被选举权的 Follower 。**

在 ZAB 协议中对 zkServer(即上面我们说的三个角色的总称) 还有两种模式的定义，分别是消息广播和崩溃恢复

**消息广播模式**

![ZAB广播](http://file.sunwaiting.com/zab_broadcast.png)

1. Leader从客户端收到一个事务请求（如果是集群中其他机器接收到客户端的事务请求，会直接转发给 Leader 服务器）
2. Leader 服务器生成一个对应的事务 Proposal，并为这个事务生成一个全局递增的唯一的ZXID（通过其 ZXID 来进行排序保证顺序性）
3. Leader 将这个事务发送给所有的 Follows 节点
4. Follower 节点将收到的事务请求加入到历史队列(Leader 会为每个 Follower 分配一个单独的队列先进先出，顺序保证消息的因果关系)中，并发送 ack 给 Leader
5. 当 Leader 收到超过半数 Follower 的 ack 消息，Leader会广播一个 commit 消息
6. 当 Follower 收到 commit 请求时，会判断该事务的 ZXID 是不是比历史队列中的任何事务的 ZXID 都小，如果是则提交，如果不是则等待比它更小的事务的 commit

![zab commit流程](http://file.sunwaiting.com/zab_commit_1.png)

**崩溃恢复模式**

ZAB 的原子广播协议在正常情况下运行良好，但天有不测风云，一旦 Leader 服务器挂掉或者由于网络原因导致与半数的 Follower 的服务器失去联系，那么就会进入崩溃恢复模式。整个恢复过程结束后需要选举出一个新的 Leader 服务器。

恢复模式大致可以分为四个阶段：**选举、发现、同步、广播**

1. 当 leader 崩溃后，集群进入选举阶段，开始选举出潜在的新 leader(一般为集群中拥有最大 ZXID 的节点)
2. 进入发现阶段，follower 与潜在的新 leader 进行沟通，如果发现超过法定人数的 follower 同意，则潜在的新leader 将 epoc h加1，进入新的纪元。新的 leader 产生
3. 集群间进行数据同步，保证集群中各个节点的事务一致
4. 集群恢复到广播模式，开始接受客户端的写请求

------



## Zookeeper 怎么保证主从节点的状态同步？或者说同步流程是什么样的

Zookeeper 的核心是原子广播机制，这个机制保证了各个 server 之间的同步。实现这个机制的协议叫做 Zab 协议。Zab 协议有两种模式，它们分别是恢复模式和广播模式。同上

------



## 集群中为什么要有主节点？ 

在分布式环境中，有些业务逻辑只需要集群中的某一台机器进行执行，其他的机器可以共享这个结果，这样可以大大减少重复计算，提高性能，于是就需要进行 leader 选举。

------



## 集群中有 3 台服务器，其中一个节点宕机，这个时候 Zookeeper 还可以使用吗？ 

可以继续使用，单数服务器只要没超过一半的服务器宕机就可以继续使用。

集群规则为 2N+1 台，N >0，即最少需要 3 台。



## Zookeeper 宕机如何处理？ 

Zookeeper 本身也是集群，推荐配置不少于 3 个服务器。Zookeeper 自身也要保证当一个节点宕机时，其他节点会继续提供服务。如果是一个 Follower 宕机，还有 2 台服务器提供访问，因为 Zookeeper 上的数据是有多个副本的，数据并不会丢失；如果是一个 Leader 宕机，Zookeeper 会选举出新的 Leader。

Zookeeper 集群的机制是只要超过半数的节点正常，集群就能正常提供服务。只有在 Zookeeper 节点挂得太多，只剩一半或不到一半节点能工作，集群才失效。所以：

3 个节点的 cluster 可以挂掉 1 个节点(leader 可以得到 2 票 > 1.5)

2 个节点的 cluster 就不能挂掉任何1个节点了(leader 可以得到 1 票 <= 1)

------



## 说下四种类型的数据节点 Znode？

1. PERSISTENT：持久节点，除非手动删除，否则节点一直存在于 Zookeeper 上。

2. EPHEMERAL：临时节点，临时节点的生命周期与客户端会话绑定，一旦客户端会话失效（客户端与 Zookeeper连接断开不一定会话失效），那么这个客户端创建的所有临时节点都会被移除。

3. PERSISTENT_SEQUENTIAL：持久顺序节点，基本特性同持久节点，只是增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字。

4. EPHEMERAL_SEQUENTIAL：临时顺序节点，基本特性同临时节点，增加了顺序属性，节点名后边会追加一个由父节点维护的自增整型数字。

------



## Zookeeper选举机制

1. 首先对比zxid。zxid大的服务器优先作为Leader
2. 若zxid相同，比如初始化的时候，每个Server的zxid都为0，就会比较myid，myid大的选出来做Leader。

 **服务器初始化时选举** 

> 目前有3台服务器，每台服务器均没有数据，它们的编号分别是1,2,3按编号依次启动，它们的选择举过程如下：

1. Server1启动，给自己投票（1,0），然后发投票信息，由于其它机器还没有启动所以它收不到反馈信息，Server1的状态一直属于Looking。
2. Server2启动，给自己投票（2,0），同时与之前启动的Server1交换结果，由于Server2的编号大所以Server2胜出，**但此时投票数正好大于半数**，所以Server2成为领导者，Server1成为小弟。
3. Server3启动，给自己投票（3,0），同时与之前启动的Server1,Server2换信息，尽管Server3的编号大，但之前Server2已经胜出，所以Server3只能成为小弟。
4. 当确定了Leader之后，每个Server更新自己的状态，Leader将状态更新为Leading，Follower将状态更新为Following。

**服务器运行期间的选举**

> zookeeper运行期间，如果有新的Server加入，或者非Leader的Server宕机，那么Leader将会同步数据到新Server或者寻找其他备用Server替代宕机的Server。若Leader宕机，此时集群暂停对外服务，开始在内部选举新的Leader。假设当前集群中有Server1、Server2、Server3三台服务器，Server2为当前集群的Leader，由于意外情况，Server2宕机了，便开始进入选举状态。过程如下

1. 变更状态。其他的非Observer服务器将自己的状态改变为Looking，开始进入Leader选举。
2. 每个Server发出一个投票（myid，zxid），由于此集群已经运行过，所以每个Server上的zxid可能不同。假设Server1的zxid为100，Server3的为99，第一轮投票中，Server1和Server3都投自己，票分别为（1，100）,（3,99）,将自己的票发送给集群中所有机器。
3. 每个Server接收接收来自其他Server的投票，接下来的步骤与启动时步骤相同。








