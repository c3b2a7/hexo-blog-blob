---
title: 学点RocketMQ
categories: 正常的文章
date: 2022-05-02 11:42:17
tags: MessageQueue
---



对于 MQ 来说，其实不管是 `RocketMQ`、`Kafka` 还是其他消息队列，它们的本质都是：**一发一存一消费**。下面我们先以这个本质作为根，一起由浅入深地聊聊 MQ。

## 从MQ的本质说起

### 原始模型

将 MQ 掰开了揉碎了来看，都是一发一存一消费，再通俗点就是一个转发器。

生产者先将消息投递到一个叫做队列的容器中，消费者不断从这个容器中取出消息[^1]进行消费，仅此而已。

![image-20220108204854445](https://s2.loli.net/2022/01/08/dPEVqTYy9m8Lxjw.png)

上图便是消息队列最**原始的模型**，它包含了两个关键词：消息和队列。

> 1、消息：就是要传输的数据，可以是最简单的字符串，也可以是任何自定义的复杂格式（只要能按预定格式解析（序列化/反序列化[^2]）出来即可）。
>  2、队列：一种先进先出数据结构。它是存放消息的容器，消息从队尾入队，从队头出队，入队即发消息的过程，出队即收消息的过程。

### 原始模型的演进

再看今天我们最常用的消息队列产品（RocketMQ、Kafka 等等），你会发现：它们都在最原始的消息模型上做了扩展，同时提出了一些新名词，比如：主题（topic）、分区[^3]（partition）、生产/消费组（producer/consumer group）等等。

要彻底理解这些新概念，先从消息模型的演进说起（**道理好比：架构从来不是设计出来的，而是演进来的**）。

#### 队列模型

最初的消息队列就是上一节讲的原始模型，它是一个严格意义上的队列（Queue）。消息按照什么顺序写进去，就按照什么顺序读出来。不过，队列没有 “读” 这个操作，读就是出队，从队头中 “删除” 这个消息。

![image-20220108205116346](https://s2.loli.net/2022/01/08/cDBuRJMgZtWOTCz.png)

这便是队列模型：它允许多个生产者往同一个队列发送消息。但是，如果有多个消费者，实际上是竞争的关系，也就是一条消息只能被其中一个消费者接收到，消费完就不能再次被消费。

#### 发布-订阅模型

如果需要将一份消息数据分发给多个消费者，并且每个消费者都要求收到全量的消息。很显然，队列模型无法满足这个需求。

一个可行的方案是：为每个消费者创建一个单独的队列，让生产者发送多份。这种做法比较笨，而且同一份数据会被复制多份，也很浪费空间。

为了解决这个问题，就演化出了另外一种消息模型：**发布-订阅模型**。

![image-20220108205149218](https://s2.loli.net/2022/01/08/1W84MyeRzusVBrC.png)

在发布-订阅模型中，存放消息的容器变成了 “主题”，消费者在接收消息之前需要先 “订阅主题”。最终，每个消费者都可以收到同一个主题的全量消息。

仔细对比下它和 ”队列模型” 的异同：发布消息的是生产者，主题即消息的容器，消费者即订阅者，无本质区别。唯一的不同点在于：**一份消息数据是否可以被多次消费**。

### 小结

上面两种模型说白了就是：单播和广播[^4]。并且当 **发布-订阅模型** 中只有一个订阅者时，它和队列模型就一样了。

这也解释了为什么现在主流的 `RocketMQ`、`Kafka` 都是基于发布-订阅模型实现的？至于各类产品中的生产/消费组[^5]、分区、消费模式、消息回溯、延迟消费等等衍生概念都是在上述原始模型的基础上，为了解决某些问题而产生的。



## 通过MQ本质看应用场景

回顾一下上面消息队列的原始模型，不难发现消息队列的这两个特性：

- 在没有消息队列时，通常生产者作为调用方、消费者作为被调方，消息通过一次RPC[^6]直接进行传达并被“消费”。引入消息队列后，一次RPC变成了两次RPC，生产者和消费者之间无需知道对方的存在。

- 多了一个中间节点（消息队列）转储消息，原本的同步操作变成了异步。

再回想起倒背如流的消息队列应用场景：`系统解耦`、`异步通信`、`流量削峰`、`内容分发 `等等，无非是利用了上面两个特性。



那么引入消息队列后就没有坏处吗？



当然不。鱼和熊掌不可兼得，在某些方面获得便利的同时也引入了一些问题：

1、消息的及时性问题，通常而言业务对消息的及时性要求不高。

2、引入额外的组件，系统复杂度上升，稳定性下降。

3、因为网络波动和异步，会存在消息的顺序和重复问题。

4、生产者发送失败和消费者消费失败的处理带来的一致性的问题。



纵观目前主流的消息队列，也并没有完全解决这些问题，根据侧重不同平衡利弊，所以才有了现在诸多的消息队列。那目前主流的消息队列是如何平衡和解决这些问题，内部实现又是如何呢？下面我们就以`RocketMQ`为例，探究一下内部设计和实现细节。



## RocketMQ

原始队列模型毕竟简单，无法满足真实的应用场景，所以目前主流的消息队列都引入了一些额外的概念来解决这些泛问题，描述的名词可能不同但作用总体上相差无几。所以先来了解一下RocketMQ中的概念[^7]，以便更好的理解后文内容。

### RocketMQ中的概念

![img](https://s2.loli.net/2022/01/08/VFlgEfhbtrz3DIj.png)

RocketMQ核心由四部分组成，分别是**NameServer**、**Broker**、**Producer**以及**Consumer**。

> 通过上图可以看到，这四部分都可集群部署，这是RocketMQ吞吐量大、可用性高的原因之一。Broker的集群部署模式也有很多种：支持多Master模式、多Master多Slave异步复制模式、多Master多Slave同步复制模式。

#### NameServer

NameServer是一个功能齐全的服务，其角色类似Dubbo中的Zookeeper，但NameServer与Zookeeper相比更轻量。主要是因为NameServer被设计成几乎无状态的，节点之间相互无通信，通过部署多个NameServer并在客户端做可用性轮训，实现了一个伪集群的高可用。

NameServer压力不会太大，平时主要开销是在维持心跳和提供Topic-Broker的关系数据。

#### Broker

Broker是具体提供业务的服务，负责存储消息，转发消息。每个Broker启动时都会向NameServer注册自己的信息，并与NameServer保持长链接及心跳，定时将Topic信息更新到NameServer。

有一点需要注意，Broker向NameServer发送心跳时， 会带上当前自己所负责的所有Topic信息，如果Topic个数太多（万级别），会导致一次心跳中Topic的数据就几十M，网络情况差的话， 网络传输失败，心跳失败，导致NameServer误认为Broker心跳失败。

#### Producer

Producer是指具体生产消息的服务，即生产者。生产者在启动时会。从支持三种消息发送方式：

- `单向发送`：消息发出去后，可以继续发送下一条消息或执行业务代码，不需要等待服务器回应，且**没有回调函数**[^8]。
- `异步发送`：消息发出去后，可以继续发送下一条消息或执行业务代码，不等待服务器回应，**有回调函数**。
- `同步发送`：消息发出去后，等待服务器响应成功或失败，才能继续后面的操作。

#### Consumer

Consumer是指具体消费消息的服务，即消费者。消费者在启动时会从NameServer拉取Topic及Queue信息，根据负载均衡策略选择消费的Queue，支持集群消费和广播消费，可提供近实时的消息订阅机制。RocketMQ中提供了两张消费API，即Pull和Push两种：

- Pull，拉取型消费。消费者主动从Broker中拉去消息进行消费，只要拉取到消息，用户应用就会启动消费过程，所以 Pull 也称主动消费类型。

- Push，推送式消费。RocketMQ Client封装了消息的拉取、消费进度和其他的内部维护工作，将消息到达后执行的回调接口留给用户应用程序来实现。所以 Push 称为被动消费类型，但从实现上看还是基于 Pull 模式，不同于 Pull 的是 Push 首先要注册回调函数，当拉取到消息后触发回调函数开始消费消息。



### RocketMQ关键特性及实现原理

分布式消息系统作为实现分布式系统可扩展、可伸缩性的关键组件，需要具有高吞吐量、高可用等特点。而谈到消息系统的设计，就回避不了两个问题：

1. 消息的顺序问题
2. 消息的重复问题

RocketMQ作为阿里开源的一款高性能、高吞吐量的消息中间件，它是怎样来解决这两个问题的？

#### 顺序消息

消息有序指的是一类消息消费时，能按照发送的顺序来消费。例如：一个订单产生了 3 条消息，分别是订单创建、订单付款、订单完成。消费时，要按照这个顺序消费才有意义。但同时多个订单之间同类的消息又是可以并行消费的。

假如生产者产生了2条消息：M1、M2，要保证这两条消息的顺序，应该怎样做？你脑中想到的可能是这样：

![image-20220108235348191](https://s2.loli.net/2022/01/08/7v8x2fJsbDZQc9r.png)

M1发送到S1后，M2发送到S2，如果要保证M1先于M2被消费，那么需要M1到达消费端后，通知S2，然后S2再将M2发送到消费端。

这个模型存在的问题是，如果M1和M2分别发送到两台Broker上，就不能保证M1一定先达到，也就不能保证M1被先消费，那么就需要在MQ Broker集群维护消息的顺序。那么如何解决？一种简单的方式就是将M1、M2发送到同一个Broker上：

![image-20220109012300101](https://s2.loli.net/2022/01/09/VrdigoQ6DwNnOk7.png)

这样可以保证M1先于M2到达MQ Broker（客户端等待M1成功后再发送M2），根据队列先进先出的原则，M1会先于M2被消费，这样就保证了消息的顺序。

这个模型，理论上可以保证消息的顺序，但在实际运用中你应该会遇到下面的问题：

![image-20220109014152500](https://s2.loli.net/2022/01/09/mJQnxjZAk2GHUiT.png)

网络延迟问题。只要将消息从一台服务器发往另一台服务器，就会存在网络延迟问题。如上图所示，如果发送M1耗时大于发送M2的耗时，那么M2就先被消费，仍然不能保证消息的顺序。即使M1和M2同时到达消费端，由于不清楚消费端1和消费端2的负载情况，仍然有可能出现M2先于M1被消费。如何解决这个问题？将M1和M2发往同一个消费者即可，且发送M1后，需要消费端响应成功后才能发送M2。

但又会引入另外一个问题，如果发送M1后，消费端1没有响应，那是继续发送M2呢，还是重新发送M1？一般为了保证消息至少被消费一次[^9]，肯定会选择重发M1到另外一个消费端2，就如下图所示。

![image-20220109013847952](https://s2.loli.net/2022/01/09/FhmzuITX4pZWv3q.png)

这样的模型就严格保证消息的顺序，细心的你仍然会发现问题，消费端1没有响应Server时有两种情况，一种是M1确实没有到达，另外一种情况是消费端1已经响应，但是Server端没有收到。如果是第二种情况，重发M1，就会造成M1被重复消费。也就是我们后面要说的第二个问题，消息重复问题。

回过头来看消息顺序问题，严格的顺序消息非常容易理解，而且处理问题也比较容易，要实现严格的顺序消息，简单且可行的办法就是：

> 保证 **生产者 - MQBroker - 消费者** 是一对一对一的关系。

但是这样设计，消息系统的吞吐量就大打折扣，也会导致更多的异常处理，比如：只要消费端出现问题，就会导致当前流程阻塞，又不得不花费更多的精力来解决阻塞问题。但消息队列需要做到高吞吐量、高容错性。这似乎是一对不可调和的矛盾，那么RocketMQ是如何解决这个问题的呢？

首先可以明确的是RocketMQ并没有实现严格意义上的顺序消息。有些问题，看起来很重要，但实际上我们可以通过合理的设计或者将问题分解来规避。如果硬要把时间花在解决它们身上，实际上是浪费的，效率低下的。从这个角度来看消息的顺序问题，我们可以得出两个结论：

> 1、不关注乱序的应用实际大量存在
> 2、队列无序并不意味着消息无序

最后我们从源码角度分析RocketMQ怎么实现发送顺序消息。

```java
// RocketMQ默认提供了两种MessageQueueSelector实现：随机/Hash
SendResult sendResult = producer.send(msg, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
    }
}, orderId);
```

在获取到路由信息以后，会根据`MessageQueueSelector`实现的算法来选择一个队列，同一个OrderId获取到的队列是同一个队列。

```java
private SendResult send()  {
    // 获取topic路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    if (topicPublishInfo != null && topicPublishInfo.ok()) {
        MessageQueue mq = null;
        // 根据我们的算法，选择一个发送队列
        // 这里的arg = orderId
        mq = selector.select(topicPublishInfo.getMessageQueueList(), msg, arg);
        if (mq != null) {
            return this.sendKernelImpl(msg, mq, communicationMode, sendCallback, timeout);
        }
    }
}
```

#### 消息重复

上面在解决消息顺序问题时，引入了一个新的问题，就是消息重复。那么RocketMQ是怎样解决消息重复的问题呢？还是“恰好”不解决。

造成消息的重复的根本原因是：网络不可达。只要通过网络交换数据，就无法避免这个问题。所以解决这个问题的办法就是不解决，转而绕过这个问题。那么问题就变成了：如果消费端收到两条一样的消息，应该怎样处理？

> 1、消费端处理消息的业务逻辑保持幂等性
> 2、保证每条消息都有唯一编号且保证消息处理成功与去重表的日志同时出现

第1条很好理解，只要保持幂等性，不管来多少条重复消息，最后处理的结果都一样。第2条原理就是利用一张日志表来记录已经处理成功的消息的ID，如果新到的消息ID已经在日志表中，那么就不再处理这条消息。

我们可以看到第1条的解决方式，很明显应该在消费端实现，不属于消息系统要实现的功能。第2条可以消息系统实现，也可以业务端实现。正常情况下出现重复消息的概率不一定大，且由消息系统实现的话，肯定会对消息系统的吞吐量和高可用有影响，所以最好还是由业务端自己处理消息重复的问题，这也是RocketMQ不解决消息重复的问题的原因。

**RocketMQ不保证消息不重复，如果你的业务需要保证严格的不重复消息，需要你自己在业务端去重。**

#### 发送消息

`Producer`轮询某topic下的所有队列的方式来实现发送方的负载均衡，如下图所示：

![img](https://s2.loli.net/2022/04/11/6G4AXyCJj9fFlto.png)

首先分析一下RocketMQ的客户端发送消息的源码：

```java
// 构造Producer
DefaultMQProducer producer = new DefaultMQProducer("ProducerGroupName");
// 初始化Producer，整个应用生命周期内，只需要初始化1次
producer.start();
// 构造Message
Message msg = new Message("TopicTest1",// topic
                        "TagA",// tag：给消息打标签,用于区分一类消息，可为null
                        "OrderID188",// key：自定义Key，可以用于去重，可为null
                        ("Hello MetaQ").getBytes());// body：消息内容
// 发送消息并返回结果
SendResult sendResult = producer.send(msg);
// 清理资源，关闭网络连接，注销自己
producer.shutdown();
```

在整个应用生命周期内，生产者需要调用一次start方法来初始化，初始化主要完成的任务有：

> 1. 如果没有指定`namesrv`地址，将会自动寻址
> 2. 启动定时任务：更新namesrv地址、从namsrv更新topic路由信息、清理已经挂掉的broker、向所有broker发送心跳...
> 3. 启动负载均衡的服务

初始化完成后，开始发送消息，发送消息的主要代码如下：

```java
private SendResult sendDefaultImpl(Message msg,......) {
    // 检查Producer的状态是否是RUNNING
    this.makeSureStateOK();
    // 检查msg是否合法：是否为null、topic,body是否为空、body是否超长
    Validators.checkMessage(msg, this.defaultMQProducer);
    // 获取topic路由信息
    TopicPublishInfo topicPublishInfo = this.tryToFindTopicPublishInfo(msg.getTopic());
    // 从路由信息中选择一个消息队列
    MessageQueue mq = topicPublishInfo.selectOneMessageQueue(lastBrokerName);
    // 将消息发送到该队列上去
    sendResult = this.sendKernelImpl(msg, mq, communicationMode, sendCallback, timeout);
}
```

代码中需要关注的两个方法`tryToFindTopicPublishInfo`和`selectOneMessageQueue`。前面说过在producer初始化时，会启动定时任务获取路由信息并更新到本地缓存，所以`tryToFindTopicPublishInfo`会首先从缓存中获取topic路由信息，如果没有获取到，则会自己去`namesrv`获取路由信息。`selectOneMessageQueue`方法通过轮询的方式，返回一个队列，以达到负载均衡的目的。

如果Producer发送消息失败，会自动重试，重试的策略：

1. 重试次数 < retryTimesWhenSendFailed（可配置）
2. 总的耗时（包含重试n次的耗时） < sendMsgTimeout（发送消息时传入的参数）
3. 同时满足上面两个条件后，Producer会选择另外一个队列发送消息

#### 消息存储

RocketMQ的消息存储是由`consume queue`和`commit log`配合完成的。

##### Consume Queue

consume queue是消息的逻辑队列，相当于字典的目录，用来指定消息在物理文件`commit log`上的位置。

我们可以在配置中指定consume queue与commitlog存储的目录
每个`topic`下的每个`queue`都有一个对应的`consumequeue`文件，比如：

```
${rocketmq.home}/store/consumequeue/${topicName}/${queueId}/${fileName}
```

Consume Queue文件组织，如图所示：

![img](https://s2.loli.net/2022/04/11/vyoNJKg9CZmdfSw.png)

1. 根据`topic`和`queueId`来组织文件，图中TopicA有两个队列0,1，那么TopicA和QueueId=0组成一个ConsumeQueue，TopicA和QueueId=1组成另一个ConsumeQueue。
2. 按照消费端的`GroupName`来分组重试队列，如果消费端消费失败，消息将被发往重试队列中，比如图中的`%RETRY%ConsumerGroupA`。
3. 按照消费端的`GroupName`来分组死信队列，如果消费端消费失败，并重试指定次数后，仍然失败，则发往死信队列，比如图中的`%DLQ%ConsumerGroupA`。

> 死信队列（Dead Letter Queue）一般用于存放由于某种原因无法传递的消息，比如处理失败或者已经过期的消息。

Consume Queue中存储单元是一个20字节定长的二进制数据，顺序写顺序读，如下图所示：

![img](https://s2.loli.net/2022/04/11/1layLKp3utAqST6.png)

1. CommitLog Offset是指这条消息在Commit Log文件中的实际偏移量
2. Size存储中消息的大小
3. Message Tag HashCode存储消息的Tag的哈希值：主要用于订阅时消息过滤（订阅时如果指定了Tag，会根据HashCode来快速查找到订阅的消息）

##### Commit Log

CommitLog：消息存放的物理文件，每台`broker`上的`commitlog`被本机所有的`queue`共享，不做任何区分。
文件的默认位置如下，仍然可通过配置文件修改：

```bash
${user.home}\store\${commitlog}\${fileName}
```

CommitLog的消息存储单元长度不固定，文件顺序写，随机读。消息的存储结构如下表所示，按照编号顺序以及编号对应的内容依次存储。

![img](https://s2.loli.net/2022/04/11/eOFKHkidxRDUwN5.png)

##### 消息存储实现

消息存储实现，比较复杂，也值得大家深入了解，这小节只以代码说明一下具体的流程。

```java
// Set the storage time
msg.setStoreTimestamp(System.currentTimeMillis());
// Set the message body BODY CRC (consider the most appropriate setting
msg.setBodyCRC(UtilAll.crc32(msg.getBody()));
StoreStatsService storeStatsService = this.defaultMessageStore.getStoreStatsService();
synchronized (this) {
    long beginLockTimestamp = this.defaultMessageStore.getSystemClock().now();
    // Here settings are stored timestamp, in order to ensure an orderly global
    msg.setStoreTimestamp(beginLockTimestamp);
    // MapedFile：操作物理文件在内存中的映射以及将内存数据持久化到物理文件中
    MapedFile mapedFile = this.mapedFileQueue.getLastMapedFile();
    // 将Message追加到文件commitlog
    result = mapedFile.appendMessage(msg, this.appendMessageCallback);
    switch (result.getStatus()) {
    case PUT_OK:break;
    case END_OF_FILE:
         // Create a new file, re-write the message
         mapedFile = this.mapedFileQueue.getLastMapedFile();
         result = mapedFile.appendMessage(msg, this.appendMessageCallback);
     break;
     DispatchRequest dispatchRequest = new DispatchRequest(
                topic,// 1
                queueId,// 2
                result.getWroteOffset(),// 3
                result.getWroteBytes(),// 4
                tagsCode,// 5
                msg.getStoreTimestamp(),// 6
                result.getLogicsOffset(),// 7
                msg.getKeys(),// 8
                /**
                 * Transaction
                 */
                msg.getSysFlag(),// 9
                msg.getPreparedTransactionOffset());// 10
    // 1.分发消息位置到ConsumeQueue
    // 2.分发到IndexService建立索引
    this.defaultMessageStore.putDispatchRequest(dispatchRequest);
}
```

##### 消息的索引文件

如果一个消息包含key值的话，会使用IndexFile存储消息索引，文件的内容结构如图：

![img](https://s2.loli.net/2022/04/11/fRUIDHaF14gtkoc.png)

索引文件主要用于根据key来查询消息的，流程主要是：

1. 根据查询的 key 的 hashcode%slotNum 得到具体的槽的位置(slotNum 是一个索引文件里面包含的最大槽的数目，例如图中所示 slotNum=5000000)
2. 根据 slotValue(slot 位置对应的值)查找到索引项列表的最后一项(倒序排列,slotValue 总是指向最新的一个索引项)
3. 遍历索引项列表返回查询时间范围内的结果集(默认一次最大返回的 32 条记录)

#### 消息订阅

RocketMQ消息订阅有两种模式，一种是Push模式，即MQServer主动向消费端推送；另外一种是Pull模式，即消费端在需要时，主动到MQServer拉取。但在具体实现时，Push和Pull模式都是采用消费端主动拉取的方式。

首先看下消费端的负载均衡：

![img](https://s2.loli.net/2022/04/11/XDkhIBlJypvx7u9.png)

消费端会通过RebalanceService线程，10秒钟做一次基于topic下的所有队列负载：

1. 遍历Consumer下的所有topic，然后根据topic订阅所有的消息
2. 获取同一topic和Consumer Group下的所有Consumer
3. 然后根据具体的分配策略来分配消费队列，分配的策略包含：平均分配、消费端配置等

如同上图所示：如果有 5 个队列，2 个 consumer，那么第一个 Consumer 消费 3 个队列，第二 consumer 消费 2 个队列。这里采用的就是平均分配策略，它类似于我们的分页，TOPIC下面的所有queue就是记录，Consumer的个数就相当于总的页数，那么每页有多少条记录，就类似于某个Consumer会消费哪些队列。

通过这样的策略来达到大体上的平均消费，这样的设计也可以很方面的水平扩展Consumer来提高消费能力。

消费端的Push模式是通过长轮询的模式来实现的，就如同下图：

![img](https://s2.loli.net/2022/04/11/AsmClOr6nFLS3Jg.png)

Consumer端每隔一段时间主动向broker发送拉消息请求，broker在收到Pull请求后，如果有消息就立即返回数据，Consumer端收到返回的消息后，再回调消费者设置的Listener方法。如果broker在收到Pull请求时，消息队列里没有数据，broker端会阻塞请求直到有数据传递或超时才返回。

当然，Consumer端是通过一个线程将阻塞队列`LinkedBlockingQueue<PullRequest>`中的`PullRequest`发送到broker拉取消息，以防止Consumer一致被阻塞。而Broker端，在接收到Consumer的`PullRequest`时，如果发现没有消息，就会把`PullRequest`扔到ConcurrentHashMap中缓存起来。broker在启动时，会启动一个线程不停的从ConcurrentHashMap取出`PullRequest`检查，直到有数据返回。



*未完待续...*





[^1]: 这里是笼统的概括，并不代表消息一定是消费者从队列中拉取，也存在主动push的消息队列产品，对应到消息队列的Push/Pull模型。
[^2]: 序列化和反序列化代表一种数据编解码的协议。通常用于将一种结构的数据转换成另一种通用结构进行传输或存储，也可以反序列化还原。部分序列化协议具有语言无关、平台无关等特性，比如protobuf。
[^3]: 不同产品中描述分区的概念不同，RocketMQ中用Queue描述某一分区，Kafka用Partition描述某一分区。一个Topic下所有分区内的消息即为该Topic全量的消息。
[^4]: 在MQ中通常用不同消费模式来描述单播和广播实现的效果，这里因为产品无关性，所以用笼统概念描述。
[^5]: 代表一类生产或消费逻辑相同的服务，也叫生产者组和消费者组。
[^6]: RPC(Remote Procedure Call)，远程过程调用。可以通俗的理解为不同系统间的接口调用。
[^7]: 此处的概念并不是指RocketMQ独有，部分名词属于消息领域模型中通用的概念。
[^8]: 回调函数多用于异步操作完成后的通知或处理，在这里用于确认消息是否发送成功。
[^9]: 几乎所有的MQ都做到了至少一次（At least once），三种消息交付方式之间的区别详见[此处](https://stackoverflow.com/questions/44204973/difference-between-exactly-once-and-at-least-once-guarantees)。
