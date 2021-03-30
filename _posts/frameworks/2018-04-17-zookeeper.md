---
layout: post
title: Zookeeper 架构原理
categories: [frameworks,内功心法]
excerpt: Zookeeper 作为一个分布式的服务框架，主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储，但是 Zookeeper 并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化。通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。
keywords: Zookeeper
---


## 1.Zookeeper是什么
Zookeeper 作为一个分布式的服务框架，主要用来解决分布式集群中应用系统的一致性问题，它能提供基于类似于文件系统的目录节点树方式的数据存储，但是 Zookeeper 并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化。通过监控这些数据状态的变化，从而可以达到基于数据的集群管理。

简单的说，zookeeper=文件系统+通知机制。

## 2.zookeeper的数据模型
![zookeeper文件系统](/images/posts/frameworks/zookeeper1.jpg)

Zookeeper 会维护一个具有层次关系的数据结构，它非常类似于一个标准的文件系统，每个节点都叫数据节点（znode），节点上可以存储数据。

数据结构的特点：

1. 每个子目录项如 NameService 都被称作为 znode，这个 znode 是被它所在的路径唯一标识，如 Server1 这个 znode 的标识为 /NameService/Server1
2. znode 可以有子节点目录，并且每个 znode 可以存储数据，注意 EPHEMERAL 类型的目录节点不能有子节点目录
3. znode 是有版本的，每个 znode 中存储的数据可以有多个版本，也就是一个访问路径中可以存储多份数据
4. znode 可以是临时节点，一旦创建这个 znode 的客户端与服务器失去联系，这个 znode 也将自动删除，Zookeeper 的客户端和服务器通信采用长连接方式，每个客户端和服务器通过心跳来保持连接，这个连接状态称为 session，如果 znode 是临时节点，这个 session 失效，znode 也就删除了
5. znode 的目录名可以自动编号，如 App1 已经存在，再创建的话，将会自动命名为 App2
6. znode 可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等，一旦变化可以通知设置监控的客户端，这个是 Zookeeper 的核心特性，Zookeeper 的很多功能都是基于这个特性实现的。

节点的类型：

1. PERSISTENT-持久化目录节点
客户端与zookeeper断开连接后，该节点依旧存在
2. PERSISTENT_SEQUENTIAL-持久化顺序编号目录节点
客户端与zookeeper断开连接后，该节点依旧存在，只是Zookeeper给该节点名称进行顺序编号
3. EPHEMERAL-临时目录节点
客户端与zookeeper断开连接后，该节点被删除
4. EPHEMERAL_SEQUENTIAL-临时顺序编号目录节点
客户端与zookeeper断开连接后，该节点被删除，只是Zookeeper给该节点名称进行顺序编号


## 3.zookeeper的架构
![zookeeper集群](/images/posts/frameworks/zookeeper2.png)

zookeeper本身可以是单机模式，也可以是集群模式，为了zookeeper本身不出现单点故障，通常情况使用集群模式，而且是master/slave模式的集群。

### 3.1 Zookeeper集群中的角色介绍

1. Leader
Leader不直接接受client的请求，但接受由其他Follower和Observer转发过来的Client请求，此外，Leader还负责投票的发起和决议，即时更新状态和数据。
2. Follower
Follower角色接受客户端请求并返回结果，参与Leader发起的投票和选举，但不具有写操作的权限。
3. Observer
Observer角色接受客户端连接，将写操作转给Leader，但Observer不参与投票（即不参加一致性协议的达成），只同步Leader节点的状态，Observer角色是为集群系统扩展而生的。



### 3.2 Zookeeper 的读写机制

* Zookeeper是一个由多个server组成的集群
* 一个leader，多个follower
* 每个server保存一份数据副本
* 全局数据一致
* 分布式读写
* 更新请求转发，由leader实施


### 3.3 Zookeeper 的保证　

* 更新请求顺序进行，来自同一个client的更新请求按其发送顺序依次执行
* 数据更新原子性，一次数据更新要么成功，要么失败
* 全局唯一数据视图，client无论连接到哪个server，数据视图都是一致的
* 实时性，在一定时间范围内，client能读到最新数据

### 3.4 Zookeeper节点数据操作流程
![zookeeper节点数据操作](/images/posts/frameworks/zookeeper3.png)

1. 在Client向Follwer发出一个写的请求
2. Follwer把请求发送给Leader
3. Leader接收到以后开始发起投票并通知Follwer进行投票
4. Follwer把投票结果发送给Leader
5. Leader将结果汇总后如果需要写入，则开始写入同时把写入操作通知给Follwer，然后commit;
6. Follwer把请求结果返回给Client

**Follower主要有四个功能：**
1. 向Leader发送请求（PING消息、REQUEST消息、ACK消息、REVALIDATE消息）；
2. 接收Leader消息并进行处理；
3. 接收Client的请求，如果为写请求，发送给Leader进行投票；
4. 返回Client结果。

**Follower的消息循环处理如下几种来自Leader的消息：**
1. PING消息： 心跳消息；
2. PROPOSAL消息：Leader发起的提案，要求Follower投票；
3. COMMIT消息：服务器端最新一次提案的信息；
4. UPTODATE消息：表明同步完成；
5. REVALIDATE消息：根据Leader的REVALIDATE结果，关闭待revalidate的session还是允许其接受消息；
6. SYNC消息：返回SYNC结果到客户端，这个消息最初由客户端发起，用来强制得到最新的更新。


### 3.5 leader 选举

leader选举是zk中最重要的技术之一，也是保证分布式数据一致性的关键所在。当集群中的一台服务器处于如下两种情况之一时，就会进入leader选举阶段——服务器初始化启动、服务器运行期间无法与leader保持连接。

选举阶段，集群间互传的消息称为投票，投票Vote主要包括二个维度的信息：ID、ZXID

* ID 被推举的leader的服务器ID，集群中的每个zk节点启动前就要配置好这个全局唯一的ID。
* ZXID 被推举的leader的事务ID ，该值是从机器DataTree内存中取的，即事务已经在机器上被commit过了。

节点进入选举阶段后的大体执行逻辑如下：
* （1）设置状态为LOOKING，初始化内部投票Vote (id,zxid) 数据至内存，并将其广播到集群其它节点。节点首次投票都是选举自己作为leader，将自身的服务ID、处理的最近一个事务请求的ZXID（ZXID是从内存数据库里取的，即该节点最近一个完成commit的事务id）及当前状态广播出去。然后进入循环等待及处理其它节点的投票信息的流程中。
* （2）循环等待流程中，节点每收到一个外部的Vote信息，都需要将其与自己内存Vote数据进行PK，规则为取ZXID大的，若ZXID相等，则取ID大的那个投票。若外部投票胜选，节点需要将该选票覆盖之前的内存Vote数据，并再次广播出去；同时还要统计是否有过半的赞同者与新的内存投票数据一致，无则继续循环等待新的投票，有则需要判断leader是否在赞同者之中，在则退出循环，选举结束，根据选举结果及各自角色切换状态，leader切换成LEADING、follower切换到FOLLOWING、observer切换到OBSERVING状态。


## 4.zookeeper 应用场景

### 4.1 统一命名服务（Name Service）

分布式应用中，通常需要有一套完整的命名规则，既能够产生唯一的名称又便于人识别和记住，通常情况下用树形的名称结构是一个理想的选择，
树形的名称结构是一个有层次的目录结构，既对人友好又不会重复。说到这里你可能想到了 JNDI，没错 Zookeeper 的 Name Service 与 JNDI 能够完成的功能是差不多的，
它们都是将有层次的目录结构关联到一定资源上，但是 Zookeeper 的 Name Service 更加是广泛意义上的关联，也许你并不需要将名称关联到特定资源上，
你可能只需要一个不会重复名称，就像数据库中产生一个唯一的数字主键一样。
Name Service 已经是 Zookeeper 内置的功能，你只要调用 Zookeeper 的 API 就能实现。如调用 create 接口就可以很容易创建一个目录节点。

### 4.2 配置管理（Configuration Management）

配置的管理在分布式应用环境中很常见，例如同一个应用系统需要多台 Server 运行，但是它们运行的应用系统的某些配置项是相同的，
如果要修改这些相同的配置项，那么就必须同时修改每台运行这个应用系统的 Server，这样非常麻烦而且容易出错。
像这样的配置信息完全可以交给 Zookeeper 来管理，将配置信息保存在 Zookeeper 的某个目录节点中，然后将所有需要修改的应用机器监控配置信息的状态，
一旦配置信息发生变化，每台应用机器就会收到 Zookeeper 的通知，然后从 Zookeeper 获取新的配置信息应用到系统中。


### 4.3 集群管理（Group Membership）

Zookeeper能够很容易的实现集群管理的功能，如有多台 Server 组成一个服务集群，那么必须要一个“总管”知道当前集群中每台机器的服务状态，
一旦有机器不能提供服务，集群中其它机器必须知道，从而做出调整重新分配服务策略。同样当增加集群的服务能力时，就会增加一台或多台 Server，
同样也必须让“总管”知道。

Zookeeper不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个“总管”，让这个总管来管理集群，这就是 Zookeeper 的另一个功能 Leader Election。

它们的实现方式都是在 Zookeeper 上创建一个 EPHEMERAL 类型的目录节点，然后每个 Server 在它们创建目录节点的父目录节点上调用
getChildren(String path, boolean watch) 方法并设置 watch 为 true，由于是 EPHEMERAL 目录节点，当创建它的 Server 死去，
这个目录节点也随之被删除，所以 Children 将会变化，这时 getChildren上的 Watch 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了。
新增 Server 也是同样的原理。

Zookeeper 如何实现 Leader Election，也就是选出一个 Master Server。和前面的一样每台 Server 创建一个 EPHEMERAL 目录节点，
不同的是它还是一个 SEQUENTIAL 目录节点，所以它是个 EPHEMERAL_SEQUENTIAL 目录节点。之所以它是 EPHEMERAL_SEQUENTIAL 目录节点，
是因为我们可以给每台 Server 编号，我们可以选择当前是最小编号的 Server 为 Master，假如这个最小编号的 Server 死去，由于是 EPHEMERAL 节点，
死去的 Server 对应的节点也被删除，所以当前的节点列表中又出现一个最小编号的节点，我们就选择这个节点为当前 Master。
这样就实现了动态选择 Master，避免了传统意义上单 Master 容易出现单点故障的问题。


### 4.4 共享锁（Locks）

共享锁在同一个进程中很容易实现，但是在跨进程或者在不同 Server 之间就不好实现了。Zookeeper 却很容易实现这个功能，
实现方式也是需要获得锁的 Server 创建一个 EPHEMERAL_SEQUENTIAL 目录节点，然后调用 getChildren方法获取当前的目录节点列表中
最小的目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用
exists(String path, boolean watch) 方法并监控 Zookeeper 上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，
从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了。

### 4.5 队列管理
Zookeeper 可以处理两种类型的队列：

* 当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达，这种是同步队列。
* 队列按照 FIFO 方式进行入队和出队操作，例如实现生产者和消费者模型。

**同步队列用 Zookeeper 实现的实现思路如下：**
创建一个父目录 /synchronizing，每个成员都监控标志（Set Watch）位目录 /synchronizing/start 是否存在，然后每个成员都加入这个队列，
加入队列的方式就是创建 /synchronizing/member_i 的临时目录节点，然后每个成员获取 / synchronizing 目录的所有目录节点，
也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 /synchronizing/start 的出现，如果已经相等就创建 /synchronizing/start。

**FIFO 队列用 Zookeeper 实现思路如下：**
实现的思路也非常简单，就是在特定的目录下创建 SEQUENTIAL 类型的子目录 /queue_i，这样就能保证所有成员加入队列时都是有编号的，
出队列时通过 getChildren( ) 方法可以返回当前所有的队列中的元素，然后消费其中最小的一个，这样就能保证 FIFO。
