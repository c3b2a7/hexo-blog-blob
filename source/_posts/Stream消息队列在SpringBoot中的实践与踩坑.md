---
title: Stream消息队列在SpringBoot中的实践与踩坑
tags:
  - Spring
  - Redis
  - MessageQueue
  - SpringBoot
categories: 正常的文章
date: 2020-06-28 21:54:57
---

## 前言

`Redis5`新增了一个`Stream`的数据类型，这个类型作为消息队列来使用时弥补了`List`和`Pub/Sub`的不足并且提供了更强大的功能，比如`ack`机制以及消费者组等概念，在有轻量消息队列使用需求时，使用这个新类型那是再好不过了。对于这个类型，在这里就不赘述了，想了解的话可以看一下这篇[文章](http://www.hellokang.net/redis/stream.html#_1-%E6%A6%82%E8%BF%B0)，在这里，我们就具体来讲一下在`SpringBoot`中的实践与踩坑。

注意，`SpringBoot`版本需要大于2.2（即`spring-data-redis`需要大于2.2）。

## 新API和类

在开始正文之前，我们先简单了解一下在2.2引入的和Stream操作相关的方法和类。

### 消息和消息ID的对象

消息（或者称为记录）和消息ID在Spring-Data-Redis中使用`Record`和`RecordId`来表示。

一个`Record`包含三部分内容：

- `stream`表示这个消息要发往那个Stream，也就是Stream的key
- `recordId`表示这个消息的ID，一般Redis服务器自动生成，也可以指定
- `value`表示消息内容

SpringBoot为我们提供了五种消息类型的抽象：`MapRecord`、`ObjectRecord`、`ByteRecord`、`ByteBufferRecord`、`StringRecord`，以及一个消息ID类型：`RecordId`。

> 这里另外说一下：其实除开`ObjectRecord`，其他几个`Record`都是通过继承`MapRecord`扩展而来的。`StringRecord`中的消息内容也并非仅仅是一个字符串，而是一个键值都为字符串类型的`Map`（`ByteRecord`、`ByteBufferRecord`同理）。而`ObjectRecord`最后也会使用`HashMapper`转换成`MapRecord`。为什么最后都是操作`Map`类型？这是因为Stream中的内容是以多个`key-value`这种键值对的形式存储的。

那么我们怎样去创建一个消息对象呢？

一般来说我们使用前两个消息类型比较多，所以Spring-Data-Redis很贴心的在`Record`这个顶级接口中提供了两个静态方法用于直接构造`MapRecord`和`ObjectRecord`：

```java
static <S, K, V> MapRecord<S, K, V> of(Map<K, V> map)
    Assert.notNull(map, "Map must not be null!");
    return StreamRecords.mapBacked(map);
}
static <S, V> ObjectRecord<S, V> of(V value) {
    Assert.notNull(value, "Value must not be null!");
    return StreamRecords.objectBacked(value);
}
```

我们可以看到，这两个方法实际上是调用了`StreamRecords`中提供的静态方法来创建，`StreamRecords`这个类提供了下面这些方法用于创建五种`Record`：

```java
ByteRecord rawBytes(Map<byte[], byte[]> raw) 
ByteBufferRecord rawBuffer(Map<ByteBuffer, ByteBuffer> raw) 
StringRecord string(Map<String, String> raw)
<S, K, V> MapRecord<S, K, V> mapBacked(Map<K, V> map)
<S, V> ObjectRecord<S, V> objectBacked(V value)
RecordBuilder<?> newRecord()  // 通过builder方式创建
```

当然，我们还可以通过使用某个具体的`Record`类型的`create`静态方法来创建，下面是几个示例：

```java
String streamKey = "channel:stream:key1";//stream key
MailInfo mailInfo = new MailInfo("554205726@qq.com", "sendmail");//定义一个Object类型的消息内容
Map<String, String> map = new HashMap<String, String>() {{
    put("receiver", "534619360@qq.com");
}}; //定义一个Map类型的消息内容
Record.of(mailInfo).withStreamKey(streamKey);
Record.of(map).withStreamKey(streamKey).withId(RecordId.of("123"));//指定id
StreamRecords.objectBacked(mailInfo).withStreamKey(streamKey);
StreamRecords.mapBacked(map).withStreamKey(streamKey).withId(RecordId.autoGenerate());//指定id
ObjectRecord.create(streamKey, mailInfo); //使用ObjectRecord的create静态方法创建
```

如果我们不通过`withId`方法显示调用去指定`id`，那么默认的情况下就是使用`RecordId.autoGenerate()`自动生成。还有一个需要注意的地方就是在使用`StreamRecords`的方法来构建`Record`时一定要记住用`withStreamKey`方法来指定`Stream Key`。

不管是消息或是消息ID，这些类基本都提供了扁平化的api来构造，使用起来还是很简单的。那么在构造了一个`Record`后怎么将其持久化到Redis的Stream类型中呢？

### 向Stream添加消息（Record)

使用`RedisTemplate`操作`Stream`：

```java
String streamKey = "channel:stream:key1";//stream key
MailInfo mailInfo = new MailInfo("554205726@qq.com", "sendmail");

ObjectRecord<String, MailInfo> record = ObjectRecord.create(streamKey, mailInfo);

RecordId id = record.getId(); //构造Record时使用的RecordId
RecordId recordId = redisTemplate.opsForStream().add(record); //返回的RecordId

id.getTimestamp(); //null
id.getSequence(); //null
id.shouldBeAutoGenerated(); //true

recordId.getTimestamp(); //not null
recordId.getSequence(); //not null
recordId.shouldBeAutoGenerated(); //false
```

使用`add`方法来添加记录，该方法执行成功后返回添加的记录的`id`信息。注意下面的结果，这里有两个问题需要注意：

1. 为什么我们构造`Record`时使用的`RecordId`和添加记录返回的`RecordId`不同？

    这个问题很好理解，因为构造`Record`时不指定`id`时虽然是自动生成，但是这个自动生成**并不是**在构造时就自动生成好了的，而是在执行Redis命令持久化时Redis服务器来自动生成的，所以前者在获取时间戳和序号的时返回`null`。

2. 为什么添加记录返回的`RecordId`调用`shouldBeAutoGenerated`方法返回`false`呢，不是自动生成了吗？

    其实也很好理解，因为在持久化一条Stream的记录时，我们可以指定`id`，也可以选择让Redis来自动生成，那么这也就导致`add`方法执行成功获取到Redis返回的`id`信息后在构造`RecordId`时**并不知道**返回的这个`id`是我们之前指定的还是Redis自动生成的，所以说前者返回`true`，后者返回`false`并不难理解。

> 说到这里，其实你去看一下`Record`构造时默认自动生成`id`是如何做到的就很好理解了。在这里稍微提一下：在构造`Record`时默认使用`RecordId.autoGenerate()`作为`RecordId`，而这个方法返回了一个匿名对象，这个匿名对象重写了上面那三个方法，前两个方法重写直接返回`null`，后者也就是`shouldBeAutoGenerated`方法返回`true`。


## 实现消息队列

在基本了解了`SpringBoot2.2`新增的几个Stream操作api和相关类之后，也就到了我们Stream实现消息队列的实践部分了。为了方便，下面我会以发送邮件为例来讲一下如何使用Strean来实现消息队列。

为了方便后续的讲解，先构造一个简单的邮件信息类作为我们的消息内容：

```java MailInfo.java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class MailInfo {

    private String receiver;

    private String description;
}
```

### 构造StreamMessageListenerContainer

在使用`Pub/Sub`模式时，我们都是先创建一个`RedisMessageListenerContainer`容器，向这个容器注册监听器然后在`onMessage`方法中处理业务逻辑即可。那么使用`Stream`类型的话有没有提供一个类似的容器呢？答案是肯定的。在`SpringBoot2.2`提供了`StreamMessageListenerContainer`这个Stream类型专有的消息监听容器，而唯一的实现也就是`DefaultStreamMessageListenerContainer`。

`StreamMessageListenerContainer`的构造函数相比`RedisMessageListenerContainer`多了一个`StreamMessageListenerContainerOptions`，这个对象是使用`builder`方式来创建的：

```java
StreamMessageListenerContainerOptions<String, ObjectRecord<String, MailInfo>> options =
        StreamMessageListenerContainerOptions.builder()
                .batchSize(100) //一批次拉取的最大count数
                .executor(executor)  //线程池
                .pollTimeout(Duration.ZERO) //阻塞式轮询
                .targetType(MailInfo.class) //目标类型（消息内容的类型）
                .build();
StreamMessageListenerContainer<String, ObjectRecord<String, MailInfo>> container =
    StreamMessageListenerContainer.create(redisConnectionFactory, options);
```

在构造`StreamMessageListenerContainerOptions`时最关键的就是`targetType`、`objectMapper`以及设置序列化器这几个方法，这些参数的设置会直接影响到后续接收到消息后能否反序列化为java对象！由于这部分内容涉及源码过多，在后面一部分我们再针对这几个方法进行详细的探查。

在构造完`StreamMessageListenerContainer`之后，现在该怎么注册消息监听器呢？我们接着往下看。

### 注册StreamListener

在Pub/Sub模式中我们使用`addMessageListener(MessageListener, Topic)`方法添加一个`MessageListener`到指定的`Topic`，那么在使用Stream消息的监听容器时，我们是使用`receive`方法。

`Spring-data-redis`提供了三个方法用于注册`StreamListener`：

```java
default Subscription receive(StreamOffset<K> streamOffset, StreamListener<K, V> listener) {
    return register(StreamReadRequest.builder(streamOffset).build(), listener);
}
default Subscription receive(Consumer consumer, StreamOffset<K> streamOffset, StreamListener<K, V> listener) {
    return register(StreamReadRequest.builder(streamOffset).consumer(consumer).autoAcknowledge(false).build(), listener);
}
default Subscription receiveAutoAck(Consumer consumer, StreamOffset<K> streamOffset, StreamListener<K, V> listener) {
    return register(StreamReadRequest.builder(streamOffset).consumer(consumer).autoAcknowledge(true).build(), listener);
}
```

如果你发现你哪里`receive`方法和我的不太一样？那么你使用的版本应该是2.2，这个版本的问题比较多，比如上面这个地方，使用第二个方法注册`StreamListener`，在消息被消费之后会自动ack，因为`ConsumerStreamReadRequestBuilder`的`autoAck`属性默认就是`true`（除非使用第一个方法指定`StreamReadRequest`），这个问题在2.3修复了，感兴趣可以去看看这部分源码，修补提交[在这里](https://github.com/spring-projects/spring-data-redis/commit/fccaeb23e99423b6ecca5dd023975a78c15c62ed#diff-879e58efc888ebee209566792c347c00)。

> 我个人最初就是使用的SpringBoot2.2.2，使用过程中发现[问题真的有点很多](https://jira.spring.io/projects/DATAREDIS/issues/DATAREDIS-1043?filter=allopenissues)。比如还有一处序列化器泛型类型错误导致`StreamMessageListenerContainerOptions`构造混乱的问题【[修补提交](https://github.com/spring-projects/spring-data-redis/commit/df720bde8fa3ccd811d010471440e07ce10b796c#diff-879e58efc888ebee209566792c347c00)】，所以说如果你想尝鲜，那么强烈建议使用`SpringBoot`最新发布的版本。

#### Consumer和StreamOffset

可以看到`receive`方法另外还需要`Consumer`和`StreamOffset`两个参数，

**Consumer**

```java
Consumer.from("group name", "consumer name")
```

`Consumer`表示消费者组中的某个消费者，这个东西只会在消费者组模式中用到。我们一般通过上面这种方式来创建，第一个参数表示消费者组，第二个参数表示消费者。

**StreamOffset**

`StreamOffset`用于表示在某个Stream上的偏移量，它包含两部分内容，一个是stream的`key`，另一个是`ReadOffset`用于表示读取偏移量。前者应该不需要过多的解释，那么`ReadOffset`这个读取偏移量是干嘛用的呢？

要搞清楚`ReadOffset`，我们首先要知道Stream中偏移量的含义，在Stream中偏移量既可以表示消费记录时的偏移量，又可以表示消费者组在Stream上的偏移量。还记得Redis中我们怎么读取Stream中的记录吗？

通过`xread`命令也就是非消费者组模式直接读取，或者使用`xreadgroup`命令在消费者组中命令一个消费者去消费一条记录，这个时候，我们可以通过`0`、`>`、`$`分别表示第一条记录、最后一次未被消费的记录和最新一条记录，这也就是`ReadOffset`的用途之一：**用于表示直接读取或消费者组中消费者读取记录时的偏移量**。

那么还有另外的用途吗？

当然了，还记得怎样创建消费者组吗？一般我们使用`xgroup create`命令创建一个消费者组时可以选择从Stream的第一条消息开始，或者Stream的中间某个记录开始，又或者从Stream的最新一条记录开始。也就分别代表了`0`、`$`。这也就是`ReadOffset`的用途之二：**用于表示创建消费者组时该消费者组在Stream上的偏移量**。

理解`ReadOffset`最快最简单的方法就是在Redis-cli中用Redis命令操作一番。这其中还有一些值得注意的问题，比如创建消费者组时不能使用`>`表示最后一次未被消费的记录；比如`0`表示从第一条开始并且包括第一条；`$`表示从最新一条开始但并不是指当前Stream的最后一条记录，所以使用`$`时最新一条也就是表示下一个`xadd`添加的那一条记录，所以说`$`在非消费者组模式的阻塞读取下才有意义！

### 实现StreamListener

同样的用Pub/Sub来类比，在Pub/Sub模式下我们实现的消息监听器是一般是`MessageListener`或者使用`MessageListenerAdapter`反射调用处理方法，在Strean消息队列的实现中必然也需要一个监听器用于处理真正的业务逻辑，这个类目前只有一个，也就是`StreamListener`：

```java
public class StreamMessageListener implements StreamListener<String, ObjectRecord<String, MailInfo>> {
    
    private final Logger logger = LoggerFactory.getLogger(StreamMessageListener.class);
    private final StringRedisTemplate stringRedisTemplate;

    public StreamMessageListener(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    @Override
    public void onMessage(ObjectRecord<String, MailInfo> message) {
        RecordId id = message.getId();
        MailInfo messageValue = message.getValue();
        logger.info("消费stream:{}中的信息:{}, 消息id:{}", message.getStream(), messageValue, id);
        // 发邮件...
        stringRedisTemplate.opsForStream().acknowledge(MAIL_GROUP, message); //手动ack
    }
}
```

`StreamListener`和`MessageListener`差不多，只需要实现`onMessage`方法，只不过多了个泛型参数罢了。在实现消息监听器后也就可以使用`receive`方法进行注册了：

```java
container.receive(Consumer.from(MAIL_GROUP, "consumer-1"),
        StreamOffset.create(MAIL_CHANNEL, ReadOffset.lastConsumed()),
        new StreamMessageListener(stringRedisTemplate));
```

注册完成之后启动`StreamMessageListenerContainer`容器：

```java
container.start();
```

完整代码：

```java
@Component
public class StreamConsumerRunner implements ApplicationRunner, DisposableBean {

    public static final String MAIL_CHANNEL = "channel:stream:mail";
    public static final String MAIL_GROUP = "group:mail";

    private final ThreadPoolTaskExecutor executor;
    private final RedisConnectionFactory redisConnectionFactory;
    private final StringRedisTemplate stringRedisTemplate;

    private StreamMessageListenerContainer<String, ObjectRecord<String, MailInfo>> container;

    public StreamConsumerRunner(ThreadPoolTaskExecutor executor, RedisConnectionFactory redisConnectionFactory, StringRedisTemplate stringRedisTemplate) {
        this.executor = executor;
        this.redisConnectionFactory = redisConnectionFactory;
        this.stringRedisTemplate = stringRedisTemplate;
    }


    @Override
    public void run(ApplicationArguments args) {
        StreamMessageListenerContainerOptions<String, ObjectRecord<String, MailInfo>> options =
                StreamMessageListenerContainerOptions.builder()
                        .batchSize(10)
                        .executor(executor)
                        .pollTimeout(Duration.ZERO)
                        .targetType(MailInfo.class)
                        .build();

        StreamMessageListenerContainer<String, ObjectRecord<String, MailInfo>> container =
                StreamMessageListenerContainer.create(redisConnectionFactory, options);

        prepareChannelAndGroup(stringRedisTemplate.opsForStream(), MAIL_CHANNEL, MAIL_GROUP);

        container.receive(Consumer.from(MAIL_GROUP, "consumer-1"),
                StreamOffset.create(MAIL_CHANNEL, ReadOffset.lastConsumed()),
                new StreamMessageListener(stringRedisTemplate));

        this.container = container;
        this.container.start();
    }

    private void prepareChannelAndGroup(StreamOperations<String, ?, ?> ops, String channel, String group) {
        String status = "OK";
        try {
            StreamInfo.XInfoGroups groups = ops.groups(channel);
            if (groups.stream().noneMatch(xInfoGroup -> group.equals(xInfoGroup.groupName()))) {
                status = ops.createGroup(channel, group);
            }
        } catch (Exception exception) {
            RecordId initialRecord = ops.add(ObjectRecord.create(channel, "Initial Record"));
            Assert.notNull(initialRecord, "Cannot initialize stream with key '" + channel + "'");
            status = ops.createGroup(channel, ReadOffset.from(initialRecord), group);
        } finally {
            Assert.isTrue("OK".equals(status), "Cannot create group with name '" + group + "'");
        }
    }

    @Override
    public void destroy() {
        this.container.stop();
    }

    public static class StreamMessageListener implements StreamListener<String, ObjectRecord<String, MailInfo>> {

        private final Logger logger = LoggerFactory.getLogger(StreamMessageListener.class);
        private final StringRedisTemplate stringRedisTemplate;

        public StreamMessageListener(StringRedisTemplate stringRedisTemplate) {
            this.stringRedisTemplate = stringRedisTemplate;
        }

        @Override
        public void onMessage(ObjectRecord<String, MailInfo> message) {
            RecordId id = message.getId();
            MailInfo messageValue = message.getValue();

            logger.info("消费stream:{}中的信息:{}, 消息id:{}", message.getStream(), messageValue, id);

            stringRedisTemplate.opsForStream().acknowledge(MAIL_GROUP, message);
        }
    }
}
```

注意`prepareChannelAndGroup`方法，在初始化容器时，如果key对应的stream或者group不存在时会抛出异常，所以我们需要提前检查并且初始化。


## 测试

添加一个测试接口：

```java
@GetMapping("/sendMail")
public ResponseEntity<RecordId> sendMail(String receiver, String description) {
    MailInfo mailInfo = new MailInfo(receiver, description);
    ObjectRecord<String, MailInfo> record = Record.of(mailInfo).withStreamKey(channel);
    RecordId recordId = redisTemplate.opsForStream().add(record);
    return ResponseEntity.ok(recordId);
}
```

访问进行测试：

```log
2020-06-28 19:26:17.870  INFO 21900 --- [         task-1] reamConsumerRunner$StreamMessageListener : 消费stream:channel:stream:mail中的信息:MailInfo(receiver=534619360@qq.com, description=发送邮件), 消息id:1593343576237-0
```

控制台输出日志，如果在redis-cli中使用`xpending`命令检查ack信息会发现也是0，因为我们虽然使用`receive`方法注册，但是在`onMessage`中手动提交了确认，当然，你也可以使用`receiveAutoAck`方法添加。

## 实践中踩到的坑

### 自动ack和泛型类型错误

这两个问题在前面已经提到了并且在2.3已经修复，这里不多说。还是那句话，如果想尝鲜，那么强烈推荐使用SpringBoot最新发布的版本。

### RedisTemplate序列化器使用错误导致容器无法反序列化

`RedisTemplate`的hashvalue的序列化器最初使用的json序列化器，导致容器监听到新消息反序列化时抛出异常：

```console
java.lang.IllegalArgumentException: Value must not be null!
    at org.springframework.util.Assert.notNull(Assert.java:198)
    at org.springframework.data.redis.connection.stream.Record.of(Record.java:81)
    at org.springframework.data.redis.connection.stream.MapRecord.toObjectRecord(MapRecord.java:147)
    at org.springframework.data.redis.core.StreamObjectMapper.toObjectRecord(StreamObjectMapper.java:132)
    at org.springframework.data.redis.core.StreamObjectMapper.map(StreamObjectMapper.java:158)
    at org.springframework.data.redis.core.StreamOperations.read(StreamOperations.java:458)
    at org.springframework.data.redis.stream.DefaultStreamMessageListenerContainer.lambda$getReadFunction$2(DefaultStreamMessageListenerContainer.java:232)
    at org.springframework.data.redis.stream.StreamPollTask.doLoop(StreamPollTask.java:138)
    at org.springframework.data.redis.stream.StreamPollTask.run(StreamPollTask.java:123)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
    at java.lang.Thread.run(Thread.java:748)
```

这是为什么呢？我们知道Redis中Stream中存的是键值对并且`DefaultStreamOperations`中操作的都是`byte[]`，也就是说我们虽然添加的是`ObjectRecord`，但是会先转换成`MapRecord`，然后再被转换成`ByteRecord`，最后进行序列化。来看一下`add`方法：

```java
public RecordId add(Record<K, ?> record) {
    Assert.notNull(record, "Record must not be null");
    MapRecord<K, HK, HV> input = StreamObjectMapper.toMapRecord(this, record); //转换成MapRecord
    ByteRecord binaryRecord = input.serialize(keySerializer(), hashKeySerializer(), hashValueSerializer()); //再使用序列化器转换成ByteRecord
    return execute(connection -> connection.xAdd(binaryRecord), true);
}
```

通过上面这个方法我们可以发现stream序列化时和其他类型不一样，我们在使用json序列化一个对象时都是直接进行的，而这里分了两步并且序列化器是用于第二部转换，那么`ObjectRecord`是怎么转换成`MapRecord`的呢？点进`StreamObjectMapper.toMapRecord`方法可以看到其实是通过`ObjectRecord#toMapRecord`方法完成的，这个方法需要一个`HashMapper`用于将对象的属性/属性值映射构造成Map类型，你会发现`opsForStream`方法重载了一个默认无参的方法，而这个方法默认使用的是`ObjectHashMapper`，在我们构造`StreamMessageListenerContainerOptionsBuilder`时调用`targetType`时默认使用的也是`ObjectHashMapper`。而这个`ObjectHashMapper`会将对象中的属性和属性值转换成`byte[]`形式，所以在第一步之后这个`MapRecord`中的值的类型已经是`byte[]`了，那么也就导致第二步在使用json序列化器转换为`ByteRecord`时出现这种情况：`objectMapper.writeValueAsBytes(byte[])`，这是一个测试实例：

```java
@Test
void test() throws JsonProcessingException {
    ObjectMapper mapper = new ObjectMapper();
    byte[] value = "534619360@qq.com".getBytes();
    byte[] bytes = mapper.writeValueAsBytes(value);
    System.out.println(new String(bytes));
    //输出"NTM0NjE5MzYwQHFxLmNvbQ=="
}
```

反应到stream中值就变成了\\"NTM0NjE5MzYwQHFxLmNvbQ==\\"（引号需要转义）。为了便于理解，我们可以使用设置使用json序列化器的`RedisTemplate`进行`add`断点debug测试看一下转换后的两个`Record`中的内容：

```java
@Test
void test() {
    stringRedisTemplate.setHashValueSerializer(RedisSerializer.json());
    MailInfo mailInfo = new MailInfo("534619360@qq.com", "send mail");
    stringRedisTemplate.opsForStream().add(Record.of(mailInfo).withStreamKey(channel));
}
```

运行测试并断点查看`MapRecord`和`ByteRecord`：

![](https://raw-1257226137.cos.ap-guangzhou.myqcloud.com/images/20200628204658.png)

使用Redis Desktop Manager查看值：

![](https://raw-1257226137.cos.ap-guangzhou.myqcloud.com/images/20200628204930.png)

测试结束终端抛出上面提到的异常。这个问题解决办法就是使用`String`序列化器也就是使用`StringRedisTemplate`，因为这个序列化器不能序列化`byte[]`类型的对象，使用这个序列化器在序列化时如果已经是`byte[]`，那么就会直接返回原`byte[]`：

![](https://raw-1257226137.cos.ap-guangzhou.myqcloud.com/images/20200628210421.png)

更具体的细节可以跟着`add`方法debug一遍。

### ReadOffset使用错误导致group中消费者消费失败

异常：

```log
RedisCommandExecutionException: ERR The $ ID is meaningless in the context of XREADGROUP: you want to read the history of this consumer by specifying a proper ID, or use the > ID to get new messages. The $ ID would just return an empty result set.
```

上面`StreamOffset`中也提到了，这涉及到`0`、`>`、`$`的使用场景和范围，如果出现这个异常，很有可能你在消费者组模式下设置消费者读取的offset时使用了`ReadOffset.latest()`，而这个对应着`$`，也就是最新一条记录。如果不明白那么你可能对这三个标识符的使用还不是很理解，最好的解决办法就是使用redis命令先完整的操作一遍。

### stream或者group不存在导致启动抛出异常

同样在上面提到了，在构造`StreamMessageListenerContainer`时需要stream和group存在才可以。解决方法就是提前检查并初始化，上面已给出代码。


## 总结

其实在实践之初，我在网上也搜了很多相关的资料，但是发现这些资料基本都是使用redis-cli进行命令上的操作，并没有`SpringBoot`中实现。这次实践可谓是艰辛，由于目前该支持的迭代次数比较少，不乏一些bug或者小问题（2.3已经比较稳定），并且只有`lettuce`提供了stream类型的操作实现，而`lettuce`本身又有些小毛病，这些因素结合在一起也就导致这个过程花费了我整整两天时间，而这篇博客又花了整整一下午的时间才算完成，其中可能有些内容因为涉及东西比较多只能粗略提一下，并且语言组织上不太好可能不好去理解。话说回来，这篇文章也算是我自己实践后的一个个人总结吧，这个过程其实学到的东西还是很多的，也不枉费花了这么多时间。如果你发现文章有什么地方有问题或者有什么地方不理解，也欢迎在评论区留言一起交流~