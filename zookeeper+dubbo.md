Zookeeper与Dubbo

## zookeeper

### 1. 概述

+ 分布式协调服务、提供分布式数据一致性解决方案
  + 数据一致性指的是集群中数据保持一致，数据同步
  + 分成强一致性和弱一致性
  + 强一致性指的是，只要数据不一致就不提供服务，通过加锁机制实现
  + 弱一致性指的是，只要最终结果保持一致就行，不需要实时同步
+ 可以实现数据发布订阅、负载均衡、命名服务、集群管理分布式锁、分布式队列

### 2. CAP原则

+ 分布式系统中的一致性(consistency)、可用性(availability)、分区容错性(partition tolerance)
  + 一致性指的是强一致性
  + 可用性指的是，系统提供的服务需要一直处于可用状态，也就是说用户的操作请求在指定时间内得到响应
  + 分区容错性指的是，在遇到网络分区故障时，还能够保证对外提供一致性和可用性服务，除非整体瘫痪
+ 其中一致性和可用性是相互矛盾的，当维护一致性时需要加锁，直到同步完成，无法实现可用性
+ 在一个分布式系统中不可能同时满足CAP三个原则，最多可以满足两个，但**分区容错性**必须满足

### 3.  一致性协议

+ 当事务需要跨过多个分布式节点时，每个节点都需要保证事务的ACID特性，此时我们需要选举出一个协调者来协调各个节点的调度。

#### 3.1 2PC，二阶段提交

+ 将事务提交过程分成两个阶段，客户端将事务执行请求发送给协调者
+ 阶段一，事务执行
  + 协调者执行事务但不提交
  + 并向其他所有参与者节点发送事务内容，询问是否可以执行实务操作
  + 个参与者执行实务操作，但不提交，将执行结果反馈给协调者
+ 阶段二，事务提交
  + 如果协调者收到了所有参与者的反馈的ACK，也就是都能执行事务，则执行事务提交，否则中断事务
  + 事务提交
    + 协调者向各个参与者节点发送commit请求
    + 参与者接受到commit请求后，执行事务的提交操作
    + 各参与者完成事务提交后，向协调者发送提交成功反馈
    + 当协调者接收到各个参与者的提交成功反馈后，自己提交事务，完成事务提交
  + 中断事务
    + 协调者发送回滚请求
    + 各个参与者节点回滚事务
    + 反馈给协调者事务回滚结果
    + 协调者接受各参与者回滚结果后，回滚事务
+ 缺点
  + 同步阻塞，协调者需要一直等待参与者回馈，无法执行其他操作
  + 单点问题，协调者一旦故障，无法保障事务一致性
  + 部分参与者出现网络问题，无法执行提交事务，而其他参与者执行提交了事务，使整个分布式系统出现数据不一致现象

#### 3.2 3PC，三阶段提交

+ 二阶段的改进版，将事务分成 CanCommit、PreCommit、DoCommit三个阶段
+ 阶段一、CanCommit
  + 协调者向各个参与节点询问是否能进行事务操作
  + 各个参与者向协调者反馈响应
+ 阶段二、PreCommit
  + 根据阶段一的反馈分成两种情况
    + 发送预提交请求
    + 执行事务中断
  + 当所有参与都回应可以进行实务操作，协调者就像各参与者发送预提交请求，各参与者反馈自身执行情况
  + 若有一个参与者无法进行实务操作，则中断
+ 阶段三、DoCommit
  + 如果各参与者都正常执行事务，则协调者发送提交事务请求，并等待各协调者回应
  + 若有一参与者执行失败，则回滚
+ 三阶段优点
  + 三阶段提交中，加入了超时机制，也就是说协调者一定时间未收到某个节点的反馈，则默认其失败，协调者不用一直等待
  + 三阶段提交中，当参与者无法及时收到来自协调者的消息后，会默认执行提交
  + 三阶段解决了二阶段单点故障和同步阻塞的问题，但不能解决数据不一致

#### 3.3 paxos算法

##### 概念

+ 基于消息传递且具有高度容错性的一种算法，是目前**解决分布式一致性**问题最为有效的方法
+ 解决了3PC中的数据不一致问题，可以在节点失效、网络分区、网络延时等各种异常情况下，保证所有节点处于同一状态。
+ 而且还引入了**少数服从多**数原则

##### 四种角色

+ client，系统外部角色，发起请求，不参与决策
+ proposer，提案提议者
+ acceptor，提案表决者，是否接受该提案，超过半数的acceptor接受该提案才通过
+ learners，提案学习者，不参与决策，提案决定后同步执行提案

##### 两个阶段

+ prepare阶段
  + 
+ accept阶段