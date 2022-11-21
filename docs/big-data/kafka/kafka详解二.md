# kafka详解二

## 1、 offset

### 1.1 offset介绍

老版本 Consumer 的位移管理是依托于 Apache ZooKeeper 的，它会自动或手动地将位移数据提交到 ZooKeeper 中保存。当 Consumer 重启后，它能自动从 ZooKeeper 中读取位移数据，从而在上次消费截止的地方继续消费。这种设计使得 Kafka Broker 不需要保存位移数据，减少了 Broker 端需要持有的状态空间，因而有利于实现高伸缩性。

新版本 Consumer 的位移管理机制其实也很简单，就是**将 Consumer 的位移数据作为一条条普通的 Kafka 消息，提交到 __consumer_offsets 中。可以这么说，__consumer_offsets 的主要作用是保存 Kafka 消费者的位移信息。**它要求这个提交过程不仅要实现高持久性，还要支持高频的写操作。显然，Kafka 的主题设计天然就满足这两个条件。

```
ps: __consumer_offsets的消息格式却是 Kafka 自己定义的，用户不能修改，也就是说你不能随意地向这个主题写消息，因为一旦你写入的消息不满足 Kafka 规定的格式，那么 Kafka 内部无法成功解析，就会造成 Broker 的崩溃。事实上，Kafka Consumer 有 API 帮你提交位移，也就是向位移主题写消息。
```

### 1.2 __consumer_offsets kv

```
这个topic下的消息格式为k-v结构，k-v分别代表消息的键值和消息体。
k-(Group ID，主题名，分区号)
v-(位移值，ts，metadata)
```

__consumer_offset何时被创建？**当 Kafka 集群中的第一个 Consumer 程序启动时，Kafka 会自动创建位移主题**.
**如果位移主题是 Kafka 自动创建的，那么该主题的分区数是 50，副本数是 3**

### 1.3 提交 offset

创建位移主题当然是为了用的，那么什么地方会用到位移主题呢？前面一直在说 Kafka Consumer 提交位移时会写入该主题，那 Consumer 是怎么提交位移的呢？目前 Kafka Consumer 提交位移的方式有两种：**自动提交位移和手动提交位移。**

#### 1.3.1 自动提交

```
Consumer 端有个参数叫 enable.auto.commit，如果值是 true，则 Consumer 在后台默默地为你定期提交位移，提交间隔由一个专属的参数 auto.commit.interval.ms 来控制。自动提交位移有一个显著的优点，就是省事，你不用操心位移提交的事情，就能保证消息消费不会丢失。但这一点同时也是缺点。因为它太省事了，以至于丧失了很大的灵活性和可控性，你完全没法把控 Consumer 端的位移管理,同时也会导致数据丢失。
```

#### 1.3.2 手动提交

```
事实上，很多与 Kafka 集成的大数据框架都是禁用自动提交位移的，如 Spark、Flink 等。这就引出了另一种位移提交方式：手动提交位移，即设置 enable.auto.commit = false。一旦设置了 false，作为 Consumer 应用开发的你就要承担起位移提交的责任。Kafka Consumer API 为你提供了位移提交的方法，如 consumer.commitSync 等。当调用这些方法时，Kafka 会向位移主题写入相应的消息。
```

#### 1.3.3 删除过期消息

```
Kafka 使用Compact 策略来删除位移主题中的过期消息，避免该主题无限期膨胀。
对于同一个 Key 的两条消息 M1 和 M2，如果 M1 的发送时间早于 M2，那么 M1 就是过期消息。Compact 的过程就是扫描日志的所有消息，剔除那些过期的消息，然后把剩下的消息整理在一起。
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181140002.jpeg)

```
Kafka 提供了专门的后台线程定期地巡检待 Compact 的主题，看看是否存在满足条件的可删除数据。这个后台线程叫 Log Cleaner。
很多实际生产环境中都出现过位移主题无限膨胀占用过多磁盘空间的问题，如果你的环境中也有这个问题，我建议你去检查一下 Log Cleaner 线程的状态，通常都是这个线程挂掉了导致的。
```



## 2、Consumer offset

Consumer 端有个位移的概念，它和消息在分区中的位移不是一回事儿，虽然它们的英文都是 Offset。Consumer 的消费位移，它记录了 Consumer 要消费的下一条消息的位移。切记是下一条消息的位移，而不是目前最新消费消息的位移。

```
假设一个分区中有 10 条消息，位移分别是 0 到 9。某个 Consumer 应用已消费了 5 条消息，这就说明该 Consumer 消费了位移为 0 到 4 的 5 条消息，此时 Consumer 的位移是 5，指向了下一条消息的位移。
```

**Consumer 需要向 Kafka 汇报自己的位移数据，这个汇报过程被称为提交位移**（Committing Offsets）。因为 Consumer 能够同时消费多个分区的数据，所以位移的提交实际上是在分区粒度上进行的，即**Consumer 需要为分配给它的每个分区提交各自的位移数据**。

```
PS:提交位移主要是为了表征 Consumer 的消费进度，这样当 Consumer 发生故障重启之后，就能够从 Kafka 中读取之前提交的位移值，然后从相应的位移处继续消费，从而避免整个消费过程重来一遍。换句话说，位移提交是 Kafka 提供给你的一个工具或语义保障，你负责维持这个语义保障，即如果你提交了位移 X，那么 Kafka 会认为所有位移值小于 X 的消息你都已经成功消费了。
```

**位移提交的语义保障是由你来负责的，Kafka 只会“无脑”地接受你提交的位移**。

**从用户的角度来说，位移提交分为自动提交和手动提交；从 Consumer 端的角度来说，位移提交分为同步提交和异步提交**。

```
所谓自动提交，就是指 Kafka Consumer 在后台默默地为你提交位移，作为用户的你完全不必操心这些事；而手动提交，则是指你要自己提交位移，Kafka Consumer 压根不管.

```

### 2.1 自动提交

```
开启自动提交位移的方法很简单。Consumer 端有个参数 enable.auto.commit，把它设置为 true 或者压根不设置它就可以了。因为它的默认值就是 true，即 Java Consumer 默认就是自动提交位移的。如果启用了自动提交，Consumer 端还有个参数就派上用场了：auto.commit.interval.ms。它的默认值是 5 秒，表明 Kafka 每 5 秒会为你自动提交一次位移。
```

```java
Properties props = new Properties();
     props.put("bootstrap.servers", "localhost:9092");
     props.put("group.id", "test");
     props.put("enable.auto.commit", "true");
     props.put("auto.commit.interval.ms", "2000");
     props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
     KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
     consumer.subscribe(Arrays.asList("foo", "bar"));
     while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
         for (ConsumerRecord<String, String> record : records)
             System.out.printf("offset = %d, key = %s, value = %s%n", record.offset(), record.key(), record.value());
     }
```



### 2.2 手动提交

```
和自动提交相反的，就是手动提交了。开启手动提交位移的方法就是设置 enable.auto.commit 为 false。但是，仅仅设置它为 false 还不够，因为只是告诉 Kafka Consumer 不要自动提交位移而已，你还需要调用相应的 API 手动提交位移。
最简单的 API 就是KafkaConsumer#commitSync()。该方法会提交 KafkaConsumer#poll() 返回的最新位移。从名字上来看，它是一个同步操作，即该方法会一直等待，直到位移被成功提交才会返回。如果提交过程中出现异常，该方法会将异常信息抛出。下面这段代码展示了 commitSync() 的使用方法：
```

```java
while (true) {
            ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofSeconds(1));
            process(records); // 处理消息
            try {
                        consumer.commitSync();
            } catch (CommitFailedException e) {
                        handle(e); // 处理提交失败异常
            }
}
```

### 2.3 自动提交 手动提交 区别

```
一旦设置了 enable.auto.commit 为 true，Kafka 会保证在开始调用 poll 方法时，提交上次 poll 返回的所有消息。从顺序上来说，poll 方法的逻辑是先提交上一批消息的位移，再处理下一批消息，因此它能保证不出现消费丢失的情况。但自动提交位移的一个问题在于，它可能会出现重复消费。

在默认情况下，Consumer 每 5 秒自动提交一次位移。现在，我们假设提交位移之后的 3 秒发生了 Rebalance 操作。在 Rebalance 之后，所有 Consumer 从上一次提交的位移处继续消费，但该位移已经是 3 秒前的位移数据了，故在 Rebalance 发生前 3 秒消费的所有数据都要重新再消费一次。虽然你能够通过减少 auto.commit.interval.ms 的值来提高提交频率，但这么做只能缩小重复消费的时间窗口，不可能完全消除它。这是自动提交机制的一个缺陷。

反观手动提交位移，它的好处就在于更加灵活，你完全能够把控位移提交的时机和频率。但是，它也有一个缺陷，就是在调用 commitSync() 时，Consumer 程序会处于阻塞状态，直到远端的 Broker 返回提交结果，这个状态才会结束。在任何系统中，因为程序而非资源限制而导致的阻塞都可能是系统的瓶颈，会影响整个应用程序的 TPS。当然，你可以选择拉长提交间隔，但这样做的后果是 Consumer 的提交频率下降，在下次 Consumer 重启回来后，会有更多的消息被重新消费。

鉴于这个问题，Kafka 社区为手动提交位移提供了另一个 API 方法：KafkaConsumer#commitAsync()。从名字上来看它就不是同步的，而是一个异步操作。调用 commitAsync() 之后，它会立即返回，不会阻塞，因此不会影响 Consumer 应用的 TPS。由于它是异步的，Kafka 提供了回调函数（callback），供你实现提交之后的逻辑，比如记录日志或处理异常等。下面这段代码展示了调用 commitAsync() 的方法：
```

```java
while (true) {
            ConsumerRecords<String, String> records = 
	consumer.poll(Duration.ofSeconds(1));
            process(records); // 处理消息
            consumer.commitAsync((offsets, exception) -> {
	if (exception != null)
	handle(exception);
	});
}
```

commitAsync 是否能够替代 commitSync 呢？答案是不能。commitAsync 的问题在于，出现问题时它不会自动重试。因为它是异步操作，倘若提交失败后自动重试，那么它重试时提交的位移值可能早已经“过期”或不是最新值了。因此，异步提交的重试其实没有意义，所以 commitAsync 是不会重试的。

显然，如果是手动提交，我们需要将 commitSync 和 commitAsync 组合使用才能到达最理想的效果，原因有两个：

1. 我们可以利用 commitSync 的自动重试来规避那些瞬时错误，比如网络的瞬时抖动，Broker 端 GC 等。因为这些问题都是短暂的，自动重试通常都会成功，因此，我们不想自己重试，而是希望 Kafka Consumer 帮我们做这件事。
2. 我们不希望程序总处于阻塞状态，影响 TPS。

来看一下下面这段代码，它展示的是如何将两个 API 方法结合使用进行手动提交。

```java
try {
            while (true) {
                        ConsumerRecords<String, String> records = 
                                    consumer.poll(Duration.ofSeconds(1));
                        process(records); // 处理消息
                        commitAysnc(); // 使用异步提交规避阻塞
            }
} catch (Exception e) {
            handle(e); // 处理异常
} finally {
            try {
                        consumer.commitSync(); // 最后一次提交使用同步阻塞式提交
	} finally {
	     consumer.close();
}
}
```

这段代码同时使用了 commitSync() 和 commitAsync()。对于常规性、阶段性的手动提交，我们调用 commitAsync() 避免程序阻塞，而在 Consumer 要关闭前，我们调用 commitSync() 方法执行同步阻塞式的位移提交，以确保 Consumer 关闭前能够保存正确的位移数据。将两者结合后，我们既实现了异步无阻塞式的位移管理，也确保了 Consumer 位移的正确性，所以，如果你需要自行编写代码开发一套 Kafka Consumer 应用，那么我推荐你使用上面的代码范例来实现手动的位移提交。

### 2.4 更精细的提交offset

Kafka Consumer API 为手动提交提供了这样的方法：commitSync(Map<TopicPartition, OffsetAndMetadata>) 和 commitAsync(Map<TopicPartition, OffsetAndMetadata>)。它们的参数是一个 Map 对象，键就是 TopicPartition，即消费的分区，而值是一个 OffsetAndMetadata 对象，保存的主要是位移数据。

比如，如何每处理 100 条消息就提交一次位移呢？在这里，我以 commitAsync 为例，展示一段代码，实际上，commitSync 的调用方法和它是一模一样的。

```java
private Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
int count = 0;
……
while (true) {
            ConsumerRecords<String, String> records = 
	consumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> record: records) {
                        process(record);  // 处理消息
                        offsets.put(new TopicPartition(record.topic(), record.partition()),
                                    new OffsetAndMetadata(record.offset() + 1)；
                        if（count % 100 == 0）
                                    consumer.commitAsync(offsets, null); // 回调处理逻辑是 null
                        count++;
	}
}
```





![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181143507.jpeg)

## 3. kafka的文件存储机制

### 3.1 概述

```
	同一个topic下有多个不同的partition，每个partition为一个目录，partition命名的规则是topic的名称加上一个序号，序号从0开始。
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181143098.png)

```
	每一个partition目录下的文件被平均切割成大小相等（默认一个文件是1G，可以手动去设置）的数据文件，每一个数据文件都被称为一个段（segment file），但每个段消息数量不一定相等，这种特性能够使得老的segment可以被快速清除。默认保留7天的数据。
	每次满1G后，在写入到一个新的文件中。
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181144937.png)

```
	另外每个partition只需要支持顺序读写就可以。如上图所示：
首先00000000000000000000.log是最早产生的文件，该文件达到1G后又产生了新的00000000000002025849.log文件，新的数据会写入到这个新的文件里面。
	这个文件到达1G后，数据又会写入到下一个文件中。也就是说它只会往文件的末尾追加数据，这就是顺序写的过程，生产者只会对每一个partition做数据的追加（写操作）。
```

### 3.2  数据消费问题讨论

```
问题：如何保证消息消费的有序性呢？比如说生产者生产了0到100个商品，那么消费者在消费的时候按照0到100这个从小到大的顺序消费？

*** 那么kafka如何保证这种有序性呢？***
难度就在于，生产者生产出0到100这100条数据之后，通过一定的分组策略存储到broker的partition中的时候，
比如0到10这10条消息被存到了这个partition中，10到20这10条消息被存到了那个partition中，这样的话，消息在分组存到partition中的时候就已经被分组策略搞得无序了。

那么能否做到消费者在消费消息的时候全局有序呢？
遇到这个问题，我们可以回答，在大多数情况下是做不到全局有序的。但在某些情况下是可以做到的。比如我的partition只有一个，这种情况下是可以全局有序的。

那么可能有人又要问了，只有一个partition的话，哪里来的分布式呢？哪里来的负载均衡呢？
所以说，全局有序是一个伪命题！全局有序根本没有办法在kafka要实现的大数据的场景来做到。但是我们只能保证当前这个partition内部消息消费的有序性。

结论：一个partition中的数据是有序的吗？回答：间隔有序，不连续。

针对一个topic里面的数据，只能做到partition内部有序，不能做到全局有序。特别是加入消费者的场景后，如何保证消费者的消费的消息的全局有序性，
这是一个伪命题，只有在一种情况下才能保证消费的消息的全局有序性，那就是只有一个partition。
```

### 3.3 Segment文件

- Segment file是什么

```
	生产者生产的消息按照一定的分区策略被发送到topic中partition中，partition在磁盘上就是一个目录，该目录名是topic的名称加上一个序号，在这个partition目录下，有两类文件，一类是以log为后缀的文件，一类是以index为后缀的文件，每一个log文件和一个index文件相对应，这一对文件就是一个segment file，也就是一个段。
	其中的log文件就是数据文件，里面存放的就是消息，而index文件是索引文件，索引文件记录了元数据信息。log文件达到1个G后滚动重新生成新的log文件
```

- Segment文件特点

```
    segment文件命名的规则：partition全局的第一个segment从0（20个0）开始，后续的每一个segment文件名是上一个segment文件中最后一条消息的offset值。

那么这样命令有什么好处呢？
假如我们有一个消费者已经消费到了368776（offset值为368776），那么现在我们要继续消费的话，怎么做呢？


	第1步是从所有log文件的文件名中找到对应的log文件，第368776条数据位于上图中的“00000000000000368769.log”这个文件中，
这一步涉及到一个常用的算法叫做“二分查找法”（假如我现在给你一个offset值让你去找，你首先是将所有的log的文件名进行排序，然后通过二分查找法进行查找，
很快就能定位到某一个文件，紧接着拿着这个offset值到其索引文件中找这条数据究竟存在哪里）；
第2步是到index文件中去找第368776条数据所在的位置。

索引文件（index文件）中存储这大量的元数据，而数据文件（log文件）中存储这大量的消息。

索引文件（index文件）中的元数据指向对应的数据文件（log文件）中消息的物理偏移地址。
```



### 3.4  kafka如何快速查询数据

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181144894.png)



```
	上图的左半部分是索引文件，里面存储的是一对一对的key-value，其中key是消息在数据文件（对应的log文件）中的编号，比如“1,3,6,8……”，
	分别表示在log文件中的第1条消息、第3条消息、第6条消息、第8条消息……，那么为什么在index文件中这些编号不是连续的呢？
	这是因为index文件中并没有为数据文件中的每条消息都建立索引，而是采用了稀疏存储的方式，每隔一定字节的数据建立一条索引。
	这样避免了索引文件占用过多的空间，从而可以将索引文件保留在内存中。
但缺点是没有建立索引的Message也不能一次定位到其在数据文件的位置，从而需要做一次顺序扫描，但是这次顺序扫描的范围就很小了。

	其中以索引文件中元数据8,1686为例，其中8代表在右边log数据文件中从上到下第8个消息(在全局partiton表示第368777个消息)，其中1686表示该消息的物理偏移地址（位置）为1686。
	
	要是读取offset=368777的消息，从00000000000000368769.log文件中的1686的位置进行读取，那么怎么知道何时读完本条消息，否则就读到下一条消息的内容了？
	
```



![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181144569.png)

参数说明：

| 关键字                 | 解释说明                                     |
| ------------------- | ---------------------------------------- |
| 8 byte offset       | 在parition(分区)内的每条消息都有一个有序的id号，这个id号被称为偏移(offset),它可以唯一确定每条消息在parition(分区)内的位置。即offset表示partiion的第多少message |
| 4 byte message size | message大小                                |
| 4 byte CRC32        | 用crc32校验message                          |
| 1 byte “magic"      | 表示本次发布Kafka服务程序协议版本号                     |
| 1 byte “attributes" | 表示为独立版本、或标识压缩类型、或编码类型。                   |
| 4 byte key length   | 表示key的长度,当key为-1时，K byte key字段不填         |
| K byte key          | 可选                                       |
| value bytes payload | 表示实际消息数据。                                |

```
	这个就需要涉及到消息的物理结构了，消息都具有固定的物理结构，包括：offset（8 Bytes）、消息体的大小（4 Bytes）、crc32（4 Bytes）、magic（1 Byte）、attributes（1 Byte）、key length（4 Bytes）、key（K Bytes）、payload(N Bytes)等等字段，可以确定一条消息的大小，即读取到哪里截止。
```



### 3.5 kafka高效文件存储设计特点

- Kafka把topic中一个parition大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。
- 通过索引信息可以快速定位message
- 通过index元数据全部映射到memory，可以避免segment file的IO磁盘操作。
- 通过索引文件稀疏存储，可以大幅降低index文件元数据占用空间大小。





## 4. 为什么Kafka速度那么快

```
	Kafka是大数据领域无处不在的消息中间件，目前广泛使用在企业内部的实时数据管道，并帮助企业构建自己的流计算应用程序。
	Kafka虽然是基于磁盘做的数据存储，但却具有高性能、高吞吐、低延时的特点，其吞吐量动辄几万、几十上百万，这其中的原由值得我们一探究竟。
```

### 4.1 顺序读写

- 磁盘顺序读写性能要远高于内存的随机读写

```
	众所周知Kafka是将消息记录持久化到本地磁盘中的，一般人会认为磁盘读写性能差，可能会对Kafka性能如何保证提出质疑。实际上不管是内存还是磁盘，快或慢关键在于寻址的方式，磁盘分为顺序读写与随机读写，内存也一样分为顺序读写与随机读写。基于磁盘的随机读写确实很慢，但磁盘的顺序读写性能却很高，一般而言要高出磁盘随机读写三个数量级，一些情况下磁盘顺序读写性能甚至要高于内存随机读写。
	磁盘的顺序读写是磁盘使用模式中最有规律的，并且操作系统也对这种模式做了大量优化，Kafka就是使用了磁盘顺序读写来提升的性能。Kafka的message是不断追加到本地磁盘文件末尾的，而不是随机的写入，这使得Kafka写入吞吐量得到了显著提升
```



### 4.2 Page Cache(页缓存)

```
	为了优化读写性能，Kafka利用了操作系统本身的Page Cache，就是利用操作系统自身的内存而不是JVM空间内存。这样做的好处有：

（1）避免Object消耗：如果是使用Java堆，Java对象的内存消耗比较大，通常是所存储数据的两倍甚至更多。
（2）避免GC问题：随着JVM中数据不断增多，垃圾回收将会变得复杂与缓慢，使用系统缓存就不会存在GC问题。
```



### 4.3 零拷贝(sendfile)

- 零拷贝并不是不需要拷贝，而是减少不必要的拷贝次数。通常是说在IO读写过程中。

  ```
   Kafka利用linux操作系统的 "零拷贝（zero-copy）" 机制在消费端做的优化。
  ```

- ​

```
	此时我们会发现用户态“空空如也”。数据没有来到用户态，而是直接在核心态就进行了传输，但这样依然还是有多次复制。首先数据被读取到read buffer中，然后发到socket buffer，最后才发到网卡。虽然减少了用户态和核心态的切换，但依然存在多次数据复制。

如果可以进一步减少数据复制的次数，甚至没有数据复制是不是就会做到最快呢？
```



## 5. kafka整合flume

- 1、安装flume

- 2、添加flume的配置

  - vi flume-kafka.conf

  ```shell
  #为我们的source channel  sink起名
  a1.sources = r1
  a1.channels = c1
  a1.sinks = k1

  #指定我们的source数据收集策略
  a1.sources.r1.type = spooldir
  a1.sources.r1.spoolDir = /opt/install/flumeData/files
  a1.sources.r1.inputCharset = utf-8

  #指定我们的source收集到的数据发送到哪个管道
  a1.sources.r1.channels = c1

  #指定我们的channel为memory,即表示所有的数据都装进memory当中
  a1.channels.c1.type = memory
  a1.channels.c1.capacity = 1000
  a1.channels.c1.transactionCapacity = 100
  ```


  #指定我们的sink为kafka sink，并指定我们的sink从哪个channel当中读取数据
  a1.sinks.k1.channel = c1
  a1.sinks.k1.type = org.apache.flume.sink.kafka.KafkaSink
  a1.sinks.k1.kafka.topic = xichuan
  a1.sinks.k1.kafka.bootstrap.servers = node01:9092,node02:9092,node03:9092
  a1.sinks.k1.kafka.flumeBatchSize = 20
  a1.sinks.k1.kafka.producer.acks = 1
  ```

- 3、创建topic

  ```shell
  kafka-topics.sh --create --topic xichuan --partitions 3 --replication-factor 2  --zookeeper node01:2181,node02:2181,node03:2181
  ```

- 4、启动flume

  ```shell
  bin/flume-ng agent -n a1 -c conf -f conf/flume-kafka.conf -Dflume.root.logger=info,console
  ```

- 5、启动kafka控制台消费者，验证数据写入成功

  ```shell
  kafka-console-consumer.sh --topic xichuan --bootstrap-server node01:9092,node02:9092,node03:9092  --from-beginning
  ```



## 6. kafka监控工具安装和使用

### 6.1. Kafka Manager

```
kafkaManager它是由雅虎开源的可以监控整个kafka集群相关信息的一个工具。
（1）可以管理几个不同的集群
（2）监控集群的状态(topics, brokers, 副本分布, 分区分布)
（3）创建topic、修改topic相关配置
```

- 1、上传安装包

  ```
  kafka-manager-1.3.0.4.zip
  ```

- 2、解压安装包

  - unzip kafka-manager-1.3.0.4.zip  -d /opt/install

- 3、修改配置文件

  - 进入到conf

    - vim application.conf

      ```shell
      #修改kafka-manager.zkhosts的值，指定kafka集群地址
      kafka-manager.zkhosts="node01:2181,node02:2181,node03:2181"
      ```

- 4、启动kafka-manager

  - 启动zk集群，kafka集群，再使用root用户启动kafka-manager服务。
  - bin/kafka-manager 默认的端口是9000，可通过 -Dhttp.port，指定端口
  - -Dconfig.file=conf/application.conf指定配置文件

  ```
  nohup bin/kafka-manager -Dconfig.file=conf/application.conf -Dhttp.port=8080 &
  ```

- 5、访问地址

  - kafka-manager所在的主机名:8080

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181145034.png)



![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181147493.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181147366.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181148742.png)



### 6.2. KafkaOffsetMonitor

```
该监控是基于一个jar包的形式运行，部署较为方便。只有监控功能，使用起来也较为安全

(1)消费者组列表
(2)查看topic的历史消费信息.
(3)每个topic的所有parition列表(topic,pid,offset,logSize,lag,owner)
(4)对consumer消费情况进行监控,并能列出每个consumer offset,滞后数据。
```

- 1、下载安装包

  ```
  KafkaOffsetMonitor-assembly-0.2.0.jar
  ```

- 2、在服务器上新建一个目录kafka_moitor，把jar包上传到该目录中

- 3、在kafka_moitor目录下新建一个脚本

  - vim start_kafka_web.sh

  ```shell
  #!/bin/sh
  java -cp KafkaOffsetMonitor-assembly-0.2.0.jar com.quantifind.kafka.offsetapp.OffsetGetterWeb --zk node01:2181,node02:2181,node03:2181 --port 8089 --refresh 10.seconds --retain 1.days
  ```

- 4、启动脚本

  ```shell
  nohup sh start_kafka_web.sh &
  ```

- 5、访问地址

  ```
  在浏览器中即可使用ip:8089访问kafka的监控页面。
  ```



![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181148833.png)



![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181148321.png)



### 6.3. Kafka Eagle

- 1、下载Kafka Eagle安装包

  - http://download.smartloli.org/
    - kafka-eagle-bin-1.2.3.tar.gz

- 2、解压 

  - tar -zxvf kafka-eagle-bin-1.2.3.tar.gz -C /opt/install
  - 解压之后进入到kafka-eagle-bin-1.2.3目录中
    - 得到kafka-eagle-web-1.2.3-bin.tar.gz
    - 然后解压  tar -zxvf kafka-eagle-web-1.2.3-bin.tar.gz
    - 重命名  mv kafka-eagle-web-1.2.3  kafka-eagle-web

- 3、修改配置文件

  - 进入到conf目录

    - 修改system-config.properties

      ```shell
      # 填上你的kafka集群信息
      kafka.eagle.zk.cluster.alias=cluster1
      cluster1.zk.list=node01:2181,node02:2181,node03:2181

      # kafka eagle页面访问端口
      kafka.eagle.webui.port=8048

      # kafka sasl authenticate
      kafka.eagle.sasl.enable=false
      kafka.eagle.sasl.protocol=SASL_PLAINTEXT
      kafka.eagle.sasl.mechanism=PLAIN
      kafka.eagle.sasl.client=/opt/install/kafka-eagle-bin-1.2.3/kafka-eagle-web/conf/kafka_client_jaas.conf

      #  添加刚刚导入的ke数据库配置，我这里使用的是mysql
      kafka.eagle.driver=com.mysql.jdbc.Driver
      kafka.eagle.url=jdbc:mysql://node03:3306/ke?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
      kafka.eagle.username=root
      kafka.eagle.password=123456
      ```

- 4、配置环境变量

  - vi /etc/profile

    ```shell
    export KE_HOME=/opt/install/kafka-eagle-bin-1.2.3/kafka-eagle-web
    export PATH=$PATH:$KE_HOME/bin
    ```

- 5、启动kafka-eagle

  - 进入到$KE_HOME/bin目录
    - 执行脚本sh ke.sh start

- 6、访问地址

  - 启动成功后在浏览器中输入`http://node01:8048/ke`就可以访问kafka eagle 了。 

    1.用户名：admin

    2.password：123456

    3.登录首页

    ![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181149625.png)

    4.仪表盘信息

    ![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181149179.png)

    5.kafka集群信息


![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181149352.png)

​	 6.zookeeper集群




![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181151001.png)

​	7.topic信息


![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181151935.png)

​	8.consumer消费者信息


![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181153103.png)

​	9.zk客户端命令


![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181153090.png)



## 7、kafka controller

**控制器组件（Controller），是 Apache Kafka 的核心组件。它的主要作用是在 Apache ZooKeeper 的帮助下管理和协调整个 Kafka 集群**。

集群中任意一台 Broker 都能充当控制器的角色，但是，在运行过程中，只能有一个 Broker 成为控制器，行使其管理和协调的职责。

```
在开始之前，先简单介绍一下 Apache ZooKeeper 框架。要知道，控制器是重度依赖 ZooKeeper 的，因此，我们有必要花一些时间学习下 ZooKeeper 是做什么的。

Apache ZooKeeper 是一个提供高可靠性的分布式协调服务框架。它使用的数据模型类似于文件系统的树形结构，根目录也是以“/”开始。该结构上的每个节点被称为 znode，用来保存一些元数据协调信息。

如果以 znode 持久性来划分，znode 可分为持久性 znode 和临时 znode。持久性 znode 不会因为 ZooKeeper 集群重启而消失，而临时 znode 则与创建该 znode 的 ZooKeeper 会话绑定，一旦会话结束，该节点会被自动删除。

ZooKeeper 赋予客户端监控 znode 变更的能力，即所谓的 Watch 通知功能。一旦 znode 节点被创建、删除，子节点数量发生变化，抑或是 znode 所存的数据本身变更，ZooKeeper 会通过节点变更监听器 (ChangeHandler) 的方式显式通知客户端。

依托于这些功能，ZooKeeper 常被用来实现集群成员管理、分布式锁、领导者选举等功能。Kafka 控制器大量使用 Watch 功能实现对集群的协调管理。在下图中展示的是 Kafka 在 ZooKeeper 中创建的 znode 分布。你不用了解每个 znode 的作用，但你可以大致体会下 Kafka 对 ZooKeeper 的依赖。
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181158972.png)

### 7.1 控制器是如何被选出来的

Broker 在启动时，会尝试去 ZooKeeper 中创建 /controller 节点。Kafka 当前选举控制器的规则是：**第一个成功创建 /controller 节点的 Broker 会被指定为控制器**。



### 7.2 控制器的作用

控制器是起协调作用的组件

#### 7.2.1 主题管理（创建、删除、增加分区）

这里的主题管理，就是指控制器帮助我们完成对 Kafka 主题的创建、删除以及分区增加的操作。换句话说，当我们执行**kafka-topics 脚本**时，大部分的后台工作都是控制器来完成的。



#### 7.2.2 分区重分配

分区重分配主要是指，**kafka-reassign-partitions 脚本**提供的对已有主题分区进行细粒度的分配功能。这部分功能也是控制器实现的。

#### 7.2.3 Preferred 领导者选举

Preferred 领导者选举主要是 Kafka 为了避免部分 Broker 负载过重而提供的一种换 Leader 的方案。

#### 7.2.4 集群成员管理（新增 Broker、Broker 主动关闭、Broker 宕机）

包括自动检测新增 Broker、Broker 主动关闭及被动宕机。这种自动检测是依赖于前面提到的 Watch 功能和 ZooKeeper 临时节点组合实现的。

比如，控制器组件会利用**Watch 机制**检查 ZooKeeper 的 /brokers/ids 节点下的子节点数量变更。目前，当有新 Broker 启动后，它会在 /brokers 下创建专属的 znode 节点。一旦创建完毕，ZooKeeper 会通过 Watch 机制将消息通知推送给控制器，这样，控制器就能自动地感知到这个变化，进而开启后续的新增 Broker 作业。

侦测 Broker 存活性则是依赖于刚刚提到的另一个机制：**临时节点**。每个 Broker 启动后，会在 /brokers/ids 下创建一个临时 znode。当 Broker 宕机或主动关闭后，该 Broker 与 ZooKeeper 的会话结束，这个 znode 会被自动删除。同理，ZooKeeper 的 Watch 机制将这一变更推送给控制器，这样控制器就能知道有 Broker 关闭或宕机了，从而进行“善后”。

#### 7.2.5 数据服务

控制器的最后一大类工作，就是向其他 Broker 提供数据服务。控制器上保存了最全的集群元数据信息，其他所有 Broker 会定期接收控制器发来的元数据更新请求，从而更新其内存中的缓存数据。

#### 7.2.6  控制器保存了什么数据

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181338851.png)



- 所有主题信息。包括具体的分区信息，比如领导者副本是谁，ISR 集合中有哪些副本等。
- 所有 Broker 信息。包括当前都有哪些运行中的 Broker，哪些正在关闭中的 Broker 等。
- 所有涉及运维任务的分区。包括当前正在进行 Preferred 领导者选举以及分区重分配的分区列表。

#### 7.2.7 控制器故障转移

在 Kafka 集群运行过程中，只能有一台 Broker 充当控制器的角色，那么这就存在**单点失效**（Single Point of Failure）的风险，Kafka 是如何应对单点失效的呢？答案就是，为控制器提供故障转移功能，也就是说所谓的 Failover。

**故障转移指的是，当运行中的控制器突然宕机或意外终止时，Kafka 能够快速地感知到，并立即启用备用控制器来代替之前失败的控制器**。这个过程就被称为 Failover，该过程是自动完成的，无需你手动干预。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181339026.png)

最开始时，Broker 0 是控制器。当 Broker 0 宕机后，ZooKeeper 通过 Watch 机制感知到并删除了 /controller 临时节点。之后，所有存活的 Broker 开始竞选新的控制器身份。Broker 3 最终赢得了选举，成功地在 ZooKeeper 上重建了 /controller 节点。之后，Broker 3 会从 ZooKeeper 中读取集群元数据信息，并初始化到自己的缓存中。至此，控制器的 Failover 完成，可以行使正常的工作职责了。

### 7.3 控制器内部设计原理

在0.11版本之前，控制器是多线程的设计，会在内部创建很多个线程。比如，控制器需要为每个 Broker 都创建一个对应的 Socket 连接，然后再创建一个专属的线程，用于向这些 Broker 发送特定请求。如果集群中的 Broker 数量很多，那么控制器端需要创建的线程就会很多。另外，控制器连接 ZooKeeper 的会话，也会创建单独的线程来处理 Watch 机制的通知回调。除了以上这些线程，控制器还会为主题删除创建额外的 I/O 线程。

比起多线程的设计，更糟糕的是，这些线程还会访问共享的控制器缓存数据。我们都知道，多线程访问共享可变数据是维持线程安全最大的难题。为了保护数据安全性，控制器不得不在代码中大量使用**ReentrantLock 同步机制**，这就进一步拖慢了整个控制器的处理速度。

鉴于这些原因，社区于 0.11 版本重构了控制器的底层设计，最大的改进就是，**把多线程的方案改成了单线程加事件队列的方案**。我直接使用社区的一张图来说明。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181339456.png)

社区引入了一个**事件处理线程**，统一处理各种控制器事件，然后控制器将原来执行的操作全部建模成一个个独立的事件，发送到专属的事件队列中，供此线程消费。这就是所谓的单线程 + 队列的实现方式。

值得注意的是，这里的单线程不代表之前提到的所有线程都被“干掉”了，控制器只是把缓存状态变更方面的工作委托给了这个线程而已。

这个方案的最大好处在于，控制器缓存中保存的状态只被一个线程处理，因此不再需要重量级的线程同步机制来维护线程安全，Kafka 不用再担心多线程并发访问的问题，非常利于社区定位和诊断控制器的各种问题。事实上，自 0.11 版本重构控制器代码后，社区关于控制器方面的 Bug 明显少多了，这也说明了这种方案是有效的。

针对控制器的第二个改进就是，**将之前同步操作 ZooKeeper 全部改为异步操作**。ZooKeeper 本身的 API 提供了同步写和异步写两种方式。之前控制器操作 ZooKeeper 使用的是同步的 API，性能很差，集中表现为，**当有大量主题分区发生变更时，ZooKeeper 容易成为系统的瓶颈**。新版本 Kafka 修改了这部分设计，完全摒弃了之前的同步 API 调用，转而采用异步 API 写入 ZooKeeper，性能有了很大的提升。根据社区的测试，改成异步之后，ZooKeeper 写入提升了 10 倍！



## 8. kafka内核原理

### 8.1  ISR机制

当ack=all时，leader 收到数据，所有 follower 都开始同步数据， 但有一个 follower，因为某种故障，迟迟不能与 leader 进行同步，那 leader 就要一直等下去， 直到它完成同步，才能发送 ack。这个问题怎么解决呢?

Leader 维护了一个动态的 in-sync replica set (ISR)，意为和 leader 保持同步的 follower 集 合。当 ISR 中的 follower 完成数据的同步之后，leader 就会给 follower 发送 ack。如果 follower 长时间未向 leader 同步数据，则该 follower 将被踢出 ISR，该时间阈值由replica.lag.time.max.ms 参数设定。Leader 发生故障之后，就会从 ISR 中选举新的 leader。

	光是依靠多副本机制能保证Kafka的高可用性，但是能保证数据不丢失吗？
	不行，因为如果leader宕机，但是leader的数据还没同步到follower上去，此时即使选举了follower作为新的leader，当时刚才的数据已经丢失了。
	
	ISR是：in-sync replica，就是跟leader partition保持同步的follower partition的数量，只有处于ISR列表中的follower才可以在leader宕机之后被选举为新的leader，因为在这个ISR列表里代表他的数据跟leader是同步的。
	
	如果要保证写入kafka的数据不丢失，首先需要保证ISR中至少有一个follower，其次就是在一条数据写入了leader partition之后，要求必须复制给ISR中所有的follower partition，才能说代表这条数据已提交，绝对不会丢失，这是Kafka给出的承诺。
### 8.2 高水位 high Watermark(hw)

#### 8.2.1 什么是高水位

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181340835.png)

图中标注“Completed”的蓝色部分代表已完成的工作，标注“In-Flight”的红色部分代表正在进行中的工作，两者的边界就是水位线。

在 Kafka 的世界中，水位的概念有一点不同。Kafka 的水位不是时间戳，更与时间无关。它是和位置信息绑定的，具体来说，它是用消息位移来表征的。

#### 8.2.2 高水位的作用

1. 定义消息可见性，即用来标识分区下的哪些消息是可以被消费者消费的。
2. 帮助 Kafka 完成副本同步。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181340078.png)

假设这是某个分区 Leader 副本的高水位图。首先，请你注意图中的“已提交消息”和“未提交消息”。在分区高水位以下的消息被认为是已提交消息，反之就是未提交消息。消费者只能消费已提交消息，即图中位移小于 8 的所有消息。注意，这里我们不讨论 Kafka 事务，因为事务机制会影响消费者所能看到的消息的范围，它不只是简单依赖高水位来判断。它依靠一个名为 LSO（Log Stable Offset）的位移值来判断事务型消费者的可见性。

另外，需要关注的是，**位移值等于高水位的消息也属于未提交消息。也就是说，高水位上的消息是不能被消费者消费的**。

图中还有一个日志末端位移的概念，即 Log End Offset，简写是 LEO。它表示副本写入下一条消息的位移值。注意，数字 15 所在的方框是虚线，这就说明，这个副本当前只有 15 条消息，位移值是从 0 到 14，下一条新消息的位移是 15。显然，介于高水位和 LEO 之间的消息就属于未提交消息。这也从侧面告诉了我们一个重要的事实，那就是：**同一个副本对象，其高水位值不会大于 LEO 值**。

**高水位和 LEO 是副本对象的两个重要属性**。Kafka 所有副本都有对应的高水位和 LEO 值，而不仅仅是 Leader 副本。只不过 Leader 副本比较特殊，Kafka 使用 Leader 副本的高水位来定义所在分区的高水位。换句话说，**分区的高水位就是其 Leader 副本的高水位。**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181341901.jpg)

**LEO:指的是每个副本最大的 offset;** 

**HW:指的是消费者能见到的最大的 offset，ISR 队列中最小的 LEO。**

 **(1)follower 故障**

follower 发生故障后会被临时踢出 ISR，待该 follower 恢复后，follower 会读取本地磁盘 记录的上次的 HW，并将 log 文件高于 HW 的部分截取掉，从 HW 开始向 leader 进行同步。 等该 follower 的 LEO 大于等于该 Partition 的 HW，即 follower 追上 leader 之后，就可以重 新加入 ISR 了。

  **(2)leader 故障**

leader 发生故障之后，会从 ISR 中选出一个新的 leader，之后，为保证多个副本之间的数据一致性，其余的 follower 会先将各自的 log 文件高于 HW 的部分截掉，然后从新的 leader 同步数据。

```
  注意:这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复。
```

### 8.3  HW更新机制

每个副本对象都保存了一组高水位值和 LEO 值，但实际上，在 Leader 副本所在的 Broker 上，还保存了其他 Follower 副本的 LEO 值。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181342824.png)

Broker 0 上保存了某分区的 Leader 副本和所有 Follower 副本的 LEO 值，而 Broker 1 上仅仅保存了该分区的某个 Follower 副本。Kafka 把 Broker 0 上保存的这些 Follower 副本又称为**远程副本**（Remote Replica）。Kafka 副本机制在运行过程中，会更新 Broker 1 上 Follower 副本的高水位和 LEO 值，同时也会更新 Broker 0 上 Leader 副本的高水位和 LEO 以及所有远程副本的 LEO，但它不会更新远程副本的高水位值，也就是我在图中标记为灰色的部分。

为什么要在 Broker 0 上保存这些远程副本呢？其实，它们的主要作用是，**帮助 Leader 副本确定其高水位，也就是分区高水位**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181342625.jpeg)

Leader 副本保持同步。判断的条件有两个。

1. 该远程 Follower 副本在 ISR 中。
2. 该远程 Follower 副本 LEO 值落后于 Leader 副本 LEO 值的时间，不超过 Broker 端参数 replica.lag.time.max.ms 的值。如果使用默认值的话，就是不超过 10 秒。



#### 8.3.1 Leader 副本

一、处理生产者请求的逻辑如下：

1. 写入消息到本地磁盘。
2. 更新分区高水位值。 i. 获取 Leader 副本所在 Broker 端保存的所有远程副本 LEO 值{LEO-1，LEO-2，……，LEO-n}。 ii. 获取 Leader 副本高水位值：currentHW。 iii. 更新 currentHW = min(currentHW, LEO-1，LEO-2，……，LEO-n)。

二、处理 Follower 副本拉取消息的逻辑如下：

1. 读取磁盘（或页缓存）中的消息数据。
2. 使用 Follower 副本发送请求中的位移值更新远程副本 LEO 值。
3. 更新分区高水位值（具体步骤与处理生产者请求的步骤相同）。

#### 8.3.2 Follower 副本

从 Leader 拉取消息的处理逻辑如下：

1. 写入消息到本地磁盘。
2. 更新 LEO 值。
3. 更新高水位值。 i. 获取 Leader 发送的高水位值：currentHW。 ii. 获取步骤 2 中更新过的 LEO 值：currentLEO。 iii. 更新高水位为 min(currentHW, currentLEO)。

#### 8.4 Leader Epoch 

Follower 副本的高水位更新需要一轮额外的拉取请求才能实现。如果把上面那个例子扩展到多个 Follower 副本，情况可能更糟，也许需要多轮拉取请求。也就是说，Leader 副本高水位更新和 Follower 副本高水位更新在时间上是存在错配的。这种错配是很多“数据丢失”或“数据不一致”问题的根源。基于此，社区在 0.11 版本正式引入了 Leader Epoch 概念，来规避因高水位更新错配导致的各种不一致问题。

所谓 Leader Epoch，大致可以认为是 Leader 版本。它由两部分数据组成。

1. Epoch。一个单调增加的版本号。每当副本领导权发生变更时，都会增加该版本号。小版本号的 Leader 被认为是过期 Leader，不能再行使 Leader 权力。
2. 起始位移（Start Offset）。Leader 副本在该 Epoch 值上写入的首条消息的位移。

```
假设现在有两个 Leader Epoch<0, 0> 和 <1, 120>，那么，第一个 Leader Epoch 表示版本号是 0，这个版本的 Leader 从位移 0 开始保存消息，一共保存了 120 条消息。之后，Leader 发生了变更，版本号增加到 1，新版本的起始位移是 120。

Kafka Broker 会在内存中为每个分区都缓存 Leader Epoch 数据，同时它还会定期地将这些信息持久化到一个 checkpoint 文件中。当 Leader 副本写入消息到磁盘时，Broker 会尝试更新这部分缓存。如果该 Leader 是首次写入消息，那么 Broker 会向缓存中增加一个 Leader Epoch 条目，否则就不做更新。这样，每次有 Leader 变更时，新的 Leader 副本会查询这部分缓存，取出对应的 Leader Epoch 的起始位移，以避免数据丢失和不一致的情况。
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181344571.png)

如果不使用leader epoch，会导致数据丢失。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181344414.png)

通过leader epoch，不会导致数据丢失

## 9. producer消息发送原理

### 9.1  producer核心流程概览

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181344159.png)

- 1、ProducerInterceptors是一个拦截器，对发送的数据进行拦截

  ```
  ps：说实话这个功能其实没啥用，我们即使真的要过滤，拦截一些消息，也不考虑使用它，我们直接发送数据之前自己用代码过滤即可
  ```

- 2、Serializer 对消息的key和value进行序列化

- 3、通过使用分区器作用在每一条消息上，实现数据分发进行入到topic不同的分区中

- 4、RecordAccumulator收集消息，实现批量发送

  ```
  它是一个缓冲区，可以缓存一批数据，把topic的每一个分区数据存在一个队列中，然后封装消息成一个一个的batch批次，最后实现数据分批次批量发送。
  ```

- 5、Sender线程从RecordAccumulator获取消息

- 6、构建ClientRequest对象

- 7、将ClientRequest交给 NetWorkClient准备发送

- 8、NetWorkClient 将请求放入到KafkaChannel的缓存

- 9、发送请求到kafka集群

- 10、调用回调函数，接受到响应

##  10. producer核心参数

- 回顾之前的producer生产者代码

~~~scala
package com.xichuan.producer;

import org.apache.kafka.clients.producer.*;

import java.util.Properties;
import java.util.concurrent.ExecutionException;

/**

- 需求：开发kafka生产者代码
  */
  public class KafkaProducerStudyDemo {
  public static void main(String[] args) throws ExecutionException, InterruptedException {
      //准备配置属性
      Properties props = new Properties();
      //kafka集群地址
      props.put("bootstrap.servers", "node01:9092,node02:9092,node03:9092");
      //acks它代表消息确认机制   // 1 0 -1 all
      props.put("acks", "all");
      //重试的次数
      props.put("retries", 0);
      //批处理数据的大小，每次写入多少数据到topic
      props.put("batch.size", 16384);
      //可以延长多久发送数据
      props.put("linger.ms", 1);
      //缓冲区的大小
      props.put("buffer.memory", 33554432);
      props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
      props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

  ```
  //添加自定义分区函数
  props.put("partitioner.class","com.xichuan.partitioner.MyPartitioner");
  
  Producer<String, String> producer = new KafkaProducer<String, String>(props);
  for (int i = 0; i < 100; i++) {
  
      // 这是异步发送的模式
      producer.send(new ProducerRecord<String, String>("test", Integer.toString(i), "hello-kafka-"+i), new Callback() {
          public void onCompletion(RecordMetadata metadata, Exception exception) {
              if(exception == null) {
                  // 消息发送成功
                  System.out.println("消息发送成功");
              } else {
                  // 消息发送失败，需要重新发送
              }
          }
  
      });
  
      // 这是同步发送的模式
      //producer.send(record).get();
      // 你要一直等待人家后续一系列的步骤都做完，发送消息之后
      // 有了消息的回应返回给你，你这个方法才会退出来
  }
  producer.close();

  }

}
~~~



### 10.1 常见异常处理

- 不管是异步还是同步，都可能让你处理异常，常见的异常如下：

  - 不管是异步还是同步，都可能让你处理异常，常见的异常如下：

    ```
    1）LeaderNotAvailableException：这个就是如果某台机器挂了，此时leader副本不可用，会导致你写入失败，要等待其他follower副本切换为leader副本之后，才能继续写入，此时可以重试发送即可。如果说你平时重启kafka的broker进程，肯定会导致leader切换，一定会导致你写入报错，是LeaderNotAvailableException

    2）NotControllerException：这个也是同理，如果说Controller所在Broker挂了，那么此时会有问题，需要等待Controller重新选举，此时也是一样就是重试即可

    3）NetworkException：网络异常，重试即可
    我们之前配置了一个参数，retries，他会自动重试的，但是如果重试几次之后还是不行，就会提供Exception给我们来处理了。
    ```

  - `retries`

    - 重新发送数据的次数
      - 默认为0，表示不重试


  - `retry.backoff.ms`
    - 两次重试之间的时间间隔
      - 默认为100ms



### 10.2 提升消息吞吐量

- `buffer.memory`

  - 设置发送消息的缓冲区，默认值是33554432，就是32MB

  ```
  如果发送消息出去的速度小于写入消息进去的速度，就会导致缓冲区写满，此时生产消息就会阻塞住，所以说这里就应该多做一些压测，尽可能保证说这块缓冲区不会被写满导致生产行为被阻塞住
  ```

- `compression.type`

  - producer用于压缩数据的压缩类型。默认是none表示无压缩。可以指定gzip、snappy
  - 压缩最好用于批量处理，批量处理消息越多，压缩性能越好。

- `batch.size`

  - producer将试图批处理消息记录，以减少请求次数。这将改善client与server之间的性能。
  - 默认是16384Bytes，即16kB，也就是一个batch满了16kB就发送出去

  ```
  如果batch太小，会导致频繁网络请求，吞吐量下降；如果batch太大，会导致一条消息需要等待很久才能被发送出去，而且会让内存缓冲区有很大压力，过多数据缓冲在内存里。
  ```

- `linger.ms`

  - 这个值默认是0，就是消息必须立即被发送

  ```
  	一般设置一个100毫秒之类的，这样的话就是说，这个消息被发送出去后进入一个batch，如果100毫秒内，这个batch满了16kB，自然就会发送出去。
  	但是如果100毫秒内，batch没满，那么也必须把消息发送出去了，不能让消息的发送延迟时间太长，也避免给内存造成过大的一个压力。
  ```

### 10.3 请求超时

- `max.request.size`	
  - 这个参数用来控制发送出去的消息的大小，默认是1048576字节，也就1mb
  - 这个一般太小了，很多消息可能都会超过1mb的大小，所以需要自己优化调整，把他设置更大一些（企业一般设置成10M）
- `request.timeout.ms`
  - 这个就是说发送一个请求出去之后，他有一个超时的时间限制，默认是30秒
  - 如果30秒都收不到响应，那么就会认为异常，会抛出一个TimeoutException来让我们进行处理

### 10.4 ACK参数

- acks参数，其实是控制发送出去的消息的持久化机制的。

  - `acks=0`

    - 生产者只管发数据，不管消息是否写入成功到broker中，数据丢失的风险最高

    ```
    	producer根本不管写入broker的消息到底成功没有，发送一条消息出去，立马就可以发送下一条消息，这是吞吐量最高的方式，但是可能消息都丢失了。
    你也不知道的，但是说实话，你如果真是那种实时数据流分析的业务和场景，就是仅仅分析一些数据报表，丢几条数据影响不大的。会让你的发送吞吐量会提升很多，你发送弄一个batch出去，不需要等待人家leader写成功，直接就可以发送下一个batch了，吞吐量很大的，哪怕是偶尔丢一点点数据，实时报表，折线图，饼图。
    ```

  - `acks=1`

    - 只要leader写入成功，就认为消息成功了.

    ```
    	默认给这个其实就比较合适的，还是可能会导致数据丢失的，如果刚写入leader，leader就挂了，此时数据必然丢了，其他的follower没收到数据副本，变成leader.
    ```

  - `acks=all 或者 acks=-1`

    - 这个leader写入成功以后，必须等待其他ISR中的副本都写入成功，才可以返回响应说这条消息写入成功了，此时你会收到一个回调通知.

    ```
    这种方式数据最安全，但是性能最差。
    ```

  - `如果要想保证数据不丢失，得如下设置`

    ```
    （1）min.insync.replicas = 2
    	ISR里必须有2个副本，一个leader和一个follower，最最起码的一个，不能只有一个leader存活，连一个follower都没有了。

    （2）acks = -1
    	每次写成功一定是leader和follower都成功才可以算做成功，这样leader挂了，follower上是一定有这条数据，不会丢失。
    	
    （3）retries = Integer.MAX_VALUE
    	无限重试，如果上述两个条件不满足，写入一直失败，就会无限次重试，保证说数据必须成功的发送给两个副本，如果做不到，就不停的重试。
    	除非是面向金融级的场景，面向企业大客户，或者是广告计费，跟钱的计算相关的场景下，才会通过严格配置保证数据绝对不丢失
    ```


### 10.5 重试乱序

- max.in.flight.requests.per.connection
  - 每个网络连接已经发送但还没有收到服务端响应的请求个数最大值

```
消息重试是可能导致消息的乱序的，因为可能排在你后面的消息都发送出去了，你现在收到回调失败了才在重试，此时消息就会乱序，所以可以使用“max.in.flight.requests.per.connection”参数设置为1，这样可以保证producer必须把一个请求发送的数据发送成功了再发送后面的请求。避免数据出现乱序
```



## 11. broker核心参数

- server.properties配置文件核心参数

  ```
  【broker.id】
  每个broker都必须自己设置的一个唯一id

  【log.dirs】
  这个极为重要，kafka的所有数据就是写入这个目录下的磁盘文件中的，如果说机器上有多块物理硬盘，那么可以把多个目录挂载到不同的物理硬盘上，然后这里可以设置多个目录，这样kafka可以数据分散到多块物理硬盘，多个硬盘的磁头可以并行写，这样可以提升吞吐量。

  【zookeeper.connect】
  连接kafka底层的zookeeper集群的

  【Listeners】
  broker监听客户端发起请求的端口号，默认是9092

  【unclean.leader.election.enable】
  默认是false，意思就是只能选举ISR列表里的follower成为新的leader，1.0版本后才设为false，之前都是true，允许非ISR列表的follower选举为新的leader

  【delete.topic.enable】
  默认true，允许删除topic

  【log.retention.hours】
  可以设置一下，要保留数据多少个小时(默认168小时)，这个就是底层的磁盘文件，默认保留7天的数据，根据自己的需求来就行了
  ```

## 12. consumer消费原理

### 12.1 Offset管理

​	    每个consumer内存里数据结构保存对每个topic的每个分区的消费offset，定期会提交offset，老版本是写入zk，但是那样高并发请求zk是不合理的架构设计，zk是做分布式系统的协调的，轻量级的元数据存储，不能负责高并发读写，作为数据存储。所以后来就是提交offset发送给内部topic：__consumer_offsets，提交过去的时候，key是group.id+topic+分区号，value就是当前offset的值，每隔一段时间，kafka内部会对这个topic进行compact。也就是每个group.id+topic+分区号就保留最新的那条数据即可。而且因为这个 __consumer_offsets可能会接收高并发的请求，所以默认分区50个，这样如果你的kafka部署了一个大的集群，比如有50台机器，就可以用50台机器来抗offset提交的请求压力，就好很多。

### 12.2 Coordinator

- Coordinator的作用

  ```
  	每个consumer group都会选择一个broker作为自己的coordinator，他是负责监控这个消费组里的各个消费者的心跳，以及判断是否宕机，然后开启rebalance.
  	根据内部的一个选择机制，会挑选一个对应的Broker，Kafka总会把你的各个消费组均匀分配给各个Broker作为coordinator来进行管理的.
  	consumer group中的每个consumer刚刚启动就会跟选举出来的这个consumer group对应的coordinator所在的broker进行通信，然后由coordinator分配分区给你的这个consumer来进行消费。coordinator会尽可能均匀的分配分区给各个consumer来消费。
  ```

- 如何选择哪台是coordinator

  ```
  	首先对消费组的groupId进行hash，接着对consumer_offsets的分区数量取模，默认是50，可以通过offsets.topic.num.partitions来设置，找到你的这个consumer group的offset要提交到consumer_offsets的哪个分区。
  	比如说：groupId，"membership-consumer-group" -> hash值（数字）-> 对50取模 -> 就知道这个consumer group下的所有的消费者提交offset的时候是往哪个分区去提交offset，找到consumer_offsets的一个分区，consumer_offset的分区的副本数量默认来说1，只有一个leader，然后对这个分区找到对应的leader所在的broker，这个broker就是这个consumer group的coordinator了，consumer接着就会维护一个Socket连接跟这个Broker进行通信。
  ```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181345078.png)



## 13. Rebalance 高级

### 13.1. Consumer Group Coordinator

Rebalance 就是让一个 Consumer Group 下所有的 Consumer 实例就如何消费订阅主题的所有分区达成共识的过程。在 Rebalance 过程中，所有 Consumer 实例共同参与，在**协调者组件**的帮助下，完成订阅主题分区的分配。但是，在整个过程中，所有实例都不能消费任何消息，因此它对 Consumer 的 TPS 影响很大。

```
所谓协调者，在 Kafka 中对应的术语是 Coordinator，它专门为 Consumer Group 服务，负责为 Group 执行 Rebalance 以及提供位移管理和组成员管理等。

Consumer 端应用程序在提交位移时，其实是向 Coordinator 所在的 Broker 提交位移。同样地，当 Consumer 应用启动时，也是向 Coordinator 所在的 Broker 发送各种请求，然后由 Coordinator 负责执行消费者组的注册、成员管理记录等元数据管理操作。

所有 Broker 在启动时，都会创建和开启相应的 Coordinator 组件。也就是说，所有 Broker 都有各自的 Coordinator 组件。
```

### 13.2. kafka 如何为 Consumer Group确定Coordinator

```
1、确定由位移主题的哪个分区来保存该 Group 数据：partitionId=Math.abs(groupId.hashCode() % offsetsTopicPartitionCount)。

2、找出该分区 Leader 副本所在的 Broker，该 Broker 即为对应的 Coordinator。

ps：首先，Kafka 会计算该 Group 的 group.id 参数的哈希值。比如你有个 Group 的 group.id 设置成了“test-group”，那么它的 hashCode 值就应该是 627841412。其次，Kafka 会计算 __consumer_offsets 的分区数，通常是 50 个分区，之后将刚才那个哈希值对分区数进行取模加求绝对值计算，即 abs(627841412 % 50) = 12。此时，我们就知道了位移主题的分区 12 负责保存这个 Group 的数据。有了分区号，算法的第 2 步就变得很简单了，我们只需要找出位移主题分区 12 的 Leader 副本在哪个 Broker 上就可以了。这个 Broker，就是我们要找的 Coordinator。
```

### 13.3. rebalance的弊端

```
1、Rebalance 影响 Consumer 端 TPS。这个之前也反复提到了，这里就不再具体讲了。总之就是，在 Rebalance 期间，Consumer 会停下手头的事情，什么也干不了。

2、Rebalance 很慢。如果你的 Group 下成员很多，就一定会有这样的痛点。

3、Rebalance 效率不高。当前 Kafka 的设计机制决定了每次 Rebalance 时，Group 下的所有成员都要参与进来，而且通常不会考虑局部性原理，但局部性原理对提升系统性能是特别重要的。
```

那么针对这些问题，kafka是否可以解决？---- 无解。

所以需要尽量的避免rebalance。

```
发生rebalance的情况如下：
1、组成员数量发生变化
2、订阅主题数量发生变化
3、订阅主题的分区数发生变化

后面两个通常都是运维的主动操作，所以它们引发的 Rebalance 大都是不可避免的。那么如何避免组成员数量发生变化？
```

### 13.4. 如何避免组成员数量发生变化

如果 Consumer Group 下的 Consumer 实例数量发生变化，就一定会引发 Rebalance。这是 Rebalance 发生的最常见的原因。

#### 13.4.1 增加实例

通常来说，增加 Consumer 实例的操作都是计划内的，可能是出于增加 TPS 或提高伸缩性的需要。总之，它不属于我们要规避的那类“不必要 Rebalance”。

#### 13.4.2 减少实例

如果就是要停掉某些 Consumer 实例，那自不必说，关键是在某些情况下，Consumer 实例会被 Coordinator 错误地认为“已停止”从而被“踢出”Group。如果是这个原因导致的 Rebalance，就需要处理了。

```
Coordinator 会在什么情况下认为某个 Consumer 实例已挂从而要退组呢？

当 Consumer Group 完成 Rebalance 之后，每个 Consumer 实例都会定期地向 Coordinator 发送心跳请求，表明它还存活着。如果某个 Consumer 实例不能及时地发送这些心跳请求，Coordinator 就会认为该 Consumer 已经“死”了，从而将其从 Group 中移除，然后开启新一轮 Rebalance。Consumer 端有个参数，叫 session.timeout.ms，就是被用来表征此事的。该参数的默认值是 10 秒，即如果 Coordinator 在 10 秒之内没有收到 Group 下某 Consumer 实例的心跳，它就会认为这个 Consumer 实例已经挂了。可以这么说，session.timout.ms 决定了 Consumer 存活性的时间间隔。

除了这个参数，Consumer 还提供了一个允许你控制发送心跳请求频率的参数，就是 heartbeat.interval.ms。这个值设置得越小，Consumer 实例发送心跳请求的频率就越高。频繁地发送心跳请求会额外消耗带宽资源，但好处是能够更加快速地知晓当前是否开启 Rebalance，因为，目前 Coordinator 通知各个 Consumer 实例开启 Rebalance 的方法，就是将 REBALANCE_NEEDED 标志封装进心跳请求的响应体中。

除了以上两个参数，Consumer 端还有一个参数，用于控制 Consumer 实际消费能力对 Rebalance 的影响，即 max.poll.interval.ms 参数。它限定了 Consumer 端应用程序两次调用 poll 方法的最大时间间隔。它的默认值是 5 分钟，表示你的 Consumer 程序如果在 5 分钟之内无法消费完 poll 方法返回的消息，那么 Consumer 会主动发起“离开组”的请求，Coordinator 也会开启新一轮 Rebalance。

```



### 13.5 解决非必要的rebalance

**第一类非必要 Rebalance 是因为未能及时发送心跳，导致 Consumer 被“踢出”Group 而引发的**。

```
需要仔细地设置session.timeout.ms 和 heartbeat.interval.ms的值。

设置 session.timeout.ms = 6s。
设置 heartbeat.interval.ms = 2s。
要保证 Consumer 实例在被判定为“dead”之前，能够发送至少 3 轮的心跳请求，即 session.timeout.ms >= 3 * heartbeat.interval.ms。

将 session.timeout.ms 设置成 6s 主要是为了让 Coordinator 能够更快地定位已经挂掉的 Consumer。毕竟，我们还是希望能尽快揪出那些“尸位素餐”的 Consumer，早日把它们踢出 Group。
```

**第二类非必要 Rebalance 是 Consumer 消费时间过长导致的**。

```
合理的设置max.poll.interval.ms，时间尽量设置为比下游处理完成时间略长一点
```

**如果上述两种配置完成后，还会进行rebalance，那么就需要去设置GC了。**



## 14. consumer核心参数

```
【heartbeat.interval.ms】
默认值：3000
consumer心跳时间，必须得保持心跳才能知道consumer是否故障了，然后如果故障之后，就会通过心跳下发rebalance的指令给其他的consumer通知他们进行rebalance的操作

【session.timeout.ms】
默认值：10000	
kafka多长时间感知不到一个consumer就认为他故障了，默认是10秒

【max.poll.interval.ms】
默认值：300000  
如果在两次poll操作之间，超过了这个时间，那么就会认为这个consume处理能力太弱了，会被踢出消费组，分区分配给别人去消费，一遍来说结合你自己的业务处理的性能来设置就可以了

【fetch.max.bytes】
默认值：1048576
获取一条消息最大的字节数，一般建议设置大一些

【max.poll.records】
默认值：500条
一次poll返回消息的最大条数，

【connections.max.idle.ms】
默认值：540000  
consumer跟broker的socket连接如果空闲超过了一定的时间，此时就会自动回收连接，但是下次消费就要重新建立socket连接，这个建议设置为-1，不要去回收

【auto.offset.reset】
    earliest
		当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费		  
	latest
		当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从当前位置开始消费
	none
		topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常
		
注：我们生产里面一般设置的是latest

【enable.auto.commit】
默认值：true
设置为自动提交offset

【auto.commit.interval.ms】
默认值：60 * 1000
每隔多久更新一下偏移量


如果消费者这端要保证数据被处理且只被处理一次：
   屏蔽掉了下面这2种情况：
	（1）数据的重复处理
	（2）数据的丢失
	
一般来说：需要手动提交偏移量，需要保证数据处理成功与保存偏移量的操作在同一事务中就可以了	
```



官网查看kafka参数<http://kafka.apache.org/10/documentation.html>

## 15、kafka 副本机制

所谓的副本机制（Replication），也可以称之为备份机制，通常是指分布式系统在多台网络互联的机器上保存有相同的数据拷贝。副本机制有什么好处呢？

1. **提供数据冗余**。即使系统部分组件失效，系统依然能够继续运转，因而增加了整体可用性以及数据持久性。
2. **提供高伸缩性**。支持横向扩展，能够通过增加机器的方式来提升读性能，进而提高读操作吞吐量。
3. **改善数据局部性**。允许将数据放入与用户地理位置相近的地方，从而降低系统延时。

遗憾的是，对于 Apache Kafka 而言，目前只能享受到副本机制带来的第 1 个好处，也就是提供数据冗余实现高可用性和高持久性。

### 15.1 kafka副本定义

Kafka 是有主题概念的，而每个主题又进一步划分成若干个分区。副本的概念实际上是在分区层级下定义的，每个分区配置有若干个副本。

```
所谓副本（Replica），本质就是一个只能追加写消息的提交日志。根据 Kafka 副本机制的定义，同一个分区下的所有副本保存有相同的消息序列，这些副本分散保存在不同的 Broker 上，从而能够对抗部分 Broker 宕机带来的数据不可用。
```

接下来我们来看一张图，它展示的是一个有 3 台 Broker 的 Kafka 集群上的副本分布情况。从这张图中，我们可以看到，主题 1 分区 0 的 3 个副本分散在 3 台 Broker 上，其他主题分区的副本也都散落在不同的 Broker 上，从而实现数据冗余。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181346876.png)

### 15.2 副本角色

该如何确保副本中所有的数据都是一致的呢？特别是对 Kafka 而言，当生产者发送消息到某个主题后，消息是如何同步到对应的所有副本中的呢？针对这个问题，最常见的解决方案就是采用**基于领导者（Leader-based）的副本机制**。Apache Kafka 就是这样的设计。

基于领导者的副本机制的工作原理如下图所示：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181347792.png)



```
1、在 Kafka 中，副本分成两类：领导者副本（Leader Replica）和追随者副本（Follower Replica）。每个分区在创建时都要选举一个副本，称为领导者副本，其余的副本自动称为追随者副本。

2、Kafka 的副本机制比其他分布式系统要更严格一些。在 Kafka 中，追随者副本是不对外提供服务的。这就是说，任何一个追随者副本都不能响应消费者和生产者的读写请求。所有的请求都必须由领导者副本来处理，或者说，所有的读写请求都必须发往领导者副本所在的 Broker，由该 Broker 负责处理。追随者副本不处理客户端请求，它唯一的任务就是从领导者副本异步拉取消息，并写入到自己的提交日志中，从而实现与领导者副本的同步。

3、当领导者副本挂掉了，或者说领导者副本所在的 Broker 宕机时，Kafka 依托于 ZooKeeper 提供的监控功能能够实时感知到，并立即开启新一轮的领导者选举，从追随者副本中选一个作为新的领导者。老 Leader 副本重启回来后，只能作为追随者副本加入到集群中。
```

### 15.3. 为何如此设计？

为什么追随者副本是不对外提供服务？

```
1.方便实现“Read-your-writes”。

所谓 Read-your-writes，顾名思义就是，当你使用生产者 API 向 Kafka 成功写入消息后，马上使用消费者 API 去读取刚才生产的消息。

举个例子，比如你平时发微博时，你发完一条微博，肯定是希望能立即看到的，这就是典型的 Read-your-writes 场景。如果允许追随者副本对外提供服务，由于副本同步是异步的，因此有可能出现追随者副本还没有从领导者副本那里拉取到最新的消息，从而使得客户端看不到最新写入的消息。

2.方便实现单调读（Monotonic Reads）。

什么是单调读呢？就是对于一个消费者用户而言，在多次消费消息时，它不会看到某条消息一会儿存在一会儿不存在。

如果允许追随者副本提供读服务，那么假设当前有 2 个追随者副本 F1 和 F2，它们异步地拉取领导者副本数据。倘若 F1 拉取了 Leader 的最新消息而 F2 还未及时拉取，那么，此时如果有一个消费者先从 F1 读取消息之后又从 F2 拉取消息，它可能会看到这样的现象：第一次消费时看到的最新消息在第二次消费时不见了，这就不是单调读一致性。但是，如果所有的读请求都是由 Leader 来处理，那么 Kafka 就很容易实现单调读一致性。
```

### 15.4. In-sync Replicas（ISR）

追随者副本不提供服务，只是定期地异步拉取领导者副本中的数据而已。既然是异步的，就存在着不可能与 Leader 实时同步的风险。在探讨如何正确应对这种风险之前，必须要精确地知道同步的含义是什么。或者说，Kafka 要明确地告诉我们，追随者副本到底在什么条件下才算与 Leader 同步。

基于这个想法，Kafka 引入了 In-sync Replicas，也就是所谓的 ISR 副本集合。ISR 中的副本都是与 Leader 同步的副本，相反，不在 ISR 中的追随者副本就被认为是与 Leader 不同步的。那么，到底什么副本能够进入到 ISR 中呢？

我们首先要明确的是，Leader 副本天然就在 ISR 中。也就是说，**ISR 不只是追随者副本集合，它必然包括 Leader 副本。甚至在某些情况下，ISR 只有 Leader 这一个副本**。

能够进入ISR的追随者副本需要满足一定的条件：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181347951.png)



图中有 3 个副本：1 个领导者副本和 2 个追随者副本。Leader 副本当前写入了 10 条消息，Follower1 副本同步了其中的 6 条消息，而 Follower2 副本只同步了其中的 3 条消息。现在，请你思考一下，对于这 2 个追随者副本，你觉得哪个追随者副本与 Leader 不同步？

答案是，要根据具体情况来定。看上去好像 Follower2 的消息数比 Leader 少了很多，它是最有可能与 Leader 不同步的。的确是这样的，但仅仅是可能。

事实上，这张图中的 2 个 Follower 副本都有可能与 Leader 不同步，但也都有可能与 Leader 同步。也就是说，Kafka 判断 Follower 是否与 Leader 同步的标准，不是看相差的消息数，而是另有“玄机”。

**这个标准就是 Broker 端参数 replica.lag.time.max.ms 参数值**。

```
这个参数的含义是 Follower 副本能够落后 Leader 副本的最长时间间隔，当前默认值是 10 秒。这就是说，只要一个 Follower 副本落后 Leader 副本的时间不连续超过 10 秒，那么 Kafka 就认为该 Follower 副本与 Leader 是同步的，即使此时 Follower 副本中保存的消息明显少于 Leader 副本中的消息。

Follower 副本唯一的工作就是不断地从 Leader 副本拉取消息，然后写入到自己的提交日志中。如果这个同步过程的速度持续慢于 Leader 副本的消息写入速度，那么在 replica.lag.time.max.ms 时间后，此 Follower 副本就会被认为是与 Leader 副本不同步的，因此不能再放入 ISR 中。此时，Kafka 会自动收缩 ISR 集合，将该副本“踢出”ISR。

值得注意的是，倘若该副本后面慢慢地追上了 Leader 的进度，那么它是能够重新被加回 ISR 的。这也表明，ISR 是一个动态调整的集合，而非静态不变的。
```

### 15.4.1 Unclean 领导者选举（Unclean Leader Election）

既然 ISR 是可以动态调整的，那么自然就可以出现这样的情形：ISR 为空。因为 Leader 副本天然就在 ISR 中，如果 ISR 为空了，就说明 Leader 副本也“挂掉”了，Kafka 需要重新选举一个新的 Leader。可是 ISR 是空，此时该怎么选举新 Leader 呢？

**Kafka 把所有不在 ISR 中的存活副本都称为非同步副本**。通常来说，非同步副本落后 Leader 太多，因此，如果选择这些副本作为新 Leader，就可能出现数据的丢失。毕竟，这些副本中保存的消息远远落后于老 Leader 中的消息。在 Kafka 中，选举这种副本的过程称为 Unclean 领导者选举。**Broker 端参数 unclean.leader.election.enable 控制是否允许 Unclean 领导者选举**。

开启 Unclean 领导者选举可能会造成数据丢失，但好处是，它使得分区 Leader 副本一直存在，不至于停止对外提供服务，因此提升了高可用性。反之，禁止 Unclean 领导者选举的好处在于维护了数据的一致性，避免了消息丢失，但牺牲了高可用性。

如果你听说过 CAP 理论的话，你一定知道，一个分布式系统通常只能同时满足一致性（Consistency）、可用性（Availability）、分区容错性（Partition tolerance）中的两个。显然，在这个问题上，Kafka 赋予你选择 C 或 A 的权利。

你可以根据你的实际业务场景决定是否开启 Unclean 领导者选举。不过，强烈建议你**不要**开启它，毕竟我们还可以通过其他的方式来提升高可用性。如果为了这点儿高可用性的改善，牺牲了数据一致性，那就非常不值当了。

## 16、kafka 重放

kafka重放即重设消费者组位移。

### 16.1、为什么要重设消费者组位移

Kafka 和传统的消息引擎在设计上是有很大区别的，其中一个比较显著的区别就是，Kafka 的消费者读取消息是可以重演的（replayable）。

像 RabbitMQ 或 ActiveMQ 这样的传统消息中间件，它们处理和响应消息的方式是破坏性的（destructive），即一旦消息被成功处理，就会被从 Broker 上删除。

反观 Kafka，由于它是基于日志结构（log-based）的消息引擎，消费者在消费消息时，仅仅是从磁盘文件上读取数据而已，是只读的操作，因此消费者不会删除消息数据。同时，由于位移数据是由消费者控制的，因此它能够很容易地修改位移的值，实现重复消费历史数据的功能。

### 16.2 重设位移策略

不论是哪种设置方式，重设位移大致可以从两个维度来进行。

1. 位移维度。这是指根据位移值来重设。也就是说，直接把消费者的位移值重设成我们给定的位移值。
2. 时间维度。我们可以给定一个时间，让消费者把位移调整成大于该时间的最小位移；也可以给出一段时间间隔，比如 30 分钟前，然后让消费者直接将位移调回 30 分钟之前的位移值。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181348713.jpeg)

```
1、Earliest 策略表示将位移调整到主题当前最早位移处。这个最早位移不一定就是 0，因为在生产环境中，很久远的消息会被 Kafka 自动删除，所以当前最早位移很可能是一个大于 0 的值。如果你想要重新消费主题的所有消息，那么可以使用 Earliest 策略。

2、Latest 策略表示把位移重设成最新末端位移。如果你总共向某个主题发送了 15 条消息，那么最新末端位移就是 15。如果你想跳过所有历史消息，打算从最新的消息处开始消费的话，可以使用 Latest 策略。

3、Current 策略表示将位移调整成消费者当前提交的最新位移。有时候你可能会碰到这样的场景：你修改了消费者程序代码，并重启了消费者，结果发现代码有问题，你需要回滚之前的代码变更，同时也要把位移重设到消费者重启时的位置，那么，Current 策略就可以帮你实现这个功能。

4、Specified-Offset 策略则是比较通用的策略，表示消费者把位移值调整到你指定的位移处。这个策略的典型使用场景是，消费者程序在处理某条错误消息时，你可以手动地“跳过”此消息的处理。在实际使用过程中，可能会出现 corrupted 消息无法被消费的情形，此时消费者程序会抛出异常，无法继续工作。一旦碰到这个问题，你就可以尝试使用 Specified-Offset 策略来规避。

5、如果说 Specified-Offset 策略要求你指定位移的绝对数值的话，那么 Shift-By-N 策略指定的就是位移的相对数值，即你给出要跳过的一段消息的距离即可。这里的“跳”是双向的，你既可以向前“跳”，也可以向后“跳”。比如，你想把位移重设成当前位移的前 100 条位移处，此时你需要指定 N 为 -100

6、DateTime 允许你指定一个时间，然后将位移重置到该时间之后的最早位移处。常见的使用场景是，你想重新消费昨天的数据，那么你可以使用该策略重设位移到昨天 0 点。

7、Duration 策略则是指给定相对的时间间隔，然后将位移调整到距离当前给定时间间隔的位移处，具体格式是 PnDTnHnMnS。如果你熟悉 Java 8 引入的 Duration 类的话，你应该不会对这个格式感到陌生。它就是一个符合 ISO-8601 规范的 Duration 格式，以字母 P 开头，后面由 4 部分组成，即 D、H、M 和 S，分别表示天、小时、分钟和秒。举个例子，如果你想将位移调回到 15 分钟前，那么你就可以指定 PT0H15M0S。
```



### 16.3 如何重设消费者位移

1、通过消费者 API 来实现。

2、通过 kafka-consumer-groups 命令行脚本来实现。

#### 16.3.1 通过消费者 API 来实现

通过 Java API 的方式来重设位移，你需要调用 KafkaConsumer 的 seek 方法，或者是它的变种方法 seekToBeginning 和 seekToEnd。我们来看下它们的方法签名。

```java
/**
     * Overrides the fetch offsets that the consumer will use on the next {@link #poll(long) poll(timeout)}. If this API
     * is invoked for the same partition more than once, the latest offset will be used on the next poll(). Note that
     * you may lose data if this API is arbitrarily used in the middle of consumption, to reset the fetch offsets
     *
     * @throws IllegalArgumentException if the provided TopicPartition is not assigned to this consumer
     *                                  or if provided offset is negative
     */
    @Override
    public void seek(TopicPartition partition, long offset) {
        acquireAndEnsureOpen();
        try {
            if (offset < 0)
                throw new IllegalArgumentException("seek offset must not be a negative number");

            log.debug("Seeking to offset {} for partition {}", offset, partition);
            this.subscriptions.seek(partition, offset);
        } finally {
            release();
        }
    }

    /**
     * Seek to the first offset for each of the given partitions. This function evaluates lazily, seeking to the
     * first offset in all partitions only when {@link #poll(long)} or {@link #position(TopicPartition)} are called.
     * If no partitions are provided, seek to the first offset for all of the currently assigned partitions.
     *
     * @throws IllegalArgumentException if {@code partitions} is {@code null} or the provided TopicPartition is not assigned to this consumer
     */
    public void seekToBeginning(Collection<TopicPartition> partitions) {
        acquireAndEnsureOpen();
        try {
            if (partitions == null) {
                throw new IllegalArgumentException("Partitions collection cannot be null");
            }
            Collection<TopicPartition> parts = partitions.size() == 0 ? this.subscriptions.assignedPartitions() : partitions;
            for (TopicPartition tp : parts) {
                log.debug("Seeking to beginning of partition {}", tp);
                subscriptions.needOffsetReset(tp, OffsetResetStrategy.EARLIEST);
            }
        } finally {
            release();
        }
    }

    /**
     * Seek to the last offset for each of the given partitions. This function evaluates lazily, seeking to the
     * final offset in all partitions only when {@link #poll(long)} or {@link #position(TopicPartition)} are called.
     * If no partitions are provided, seek to the final offset for all of the currently assigned partitions.
     * <p>
     * If {@code isolation.level=read_committed}, the end offset will be the Last Stable Offset, i.e., the offset
     * of the first message with an open transaction.
     *
     * @throws IllegalArgumentException if {@code partitions} is {@code null} or the provided TopicPartition is not assigned to this consumer
     */
    public void seekToEnd(Collection<TopicPartition> partitions) {
        acquireAndEnsureOpen();
        try {
            if (partitions == null) {
                throw new IllegalArgumentException("Partitions collection cannot be null");
            }
            Collection<TopicPartition> parts = partitions.size() == 0 ? this.subscriptions.assignedPartitions() : partitions;
            for (TopicPartition tp : parts) {
                log.debug("Seeking to end of partition {}", tp);
                subscriptions.needOffsetReset(tp, OffsetResetStrategy.LATEST);
            }
        } finally {
            release();
        }
    }

```

根据方法的定义，可以知道，每次调用 seek 方法只能重设一个分区的位移。seek 的变种方法 seekToBeginning 和 seekToEnd 则拥有一次重设多个分区的能力。在调用它们时，可以一次性传入多个主题分区。



 **Earliest实现**

```java
Properties consumerProperties = new Properties();
consumerProperties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);
consumerProperties.put(ConsumerConfig.GROUP_ID_CONFIG, groupID);
consumerProperties.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
consumerProperties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
consumerProperties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
consumerProperties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, brokerList);
 
String topic = "test";  // 要重设位移的 Kafka 主题 
try (final KafkaConsumer<String, String> consumer = 
	new KafkaConsumer<>(consumerProperties)) {
         consumer.subscribe(Collections.singleton(topic));
         consumer.poll(0);
         consumer.seekToBeginning(
	consumer.partitionsFor(topic).stream().map(partitionInfo ->          
	new TopicPartition(topic, partitionInfo.partition()))
	.collect(Collectors.toList()));
} 
```

1. 你要创建的消费者程序，要禁止自动提交位移。
2. 组 ID 要设置成你要重设的消费者组的组 ID。
3. 调用 seekToBeginning 方法时，需要一次性构造主题的所有分区对象。
4. 最重要的是，一定要调用带长整型的 poll 方法，而不要调用 consumer.poll(Duration.ofSecond(0))。

**Latest实现**

```java
consumer.seekToEnd(
	consumer.partitionsFor(topic).stream().map(partitionInfo ->          
	new TopicPartition(topic, partitionInfo.partition()))
	.collect(Collectors.toList()));
```

**Current实现**

实现 Current 策略的方法很简单，需要借助 KafkaConsumer 的 committed 方法来获取当前提交的最新位移

```java
consumer.partitionsFor(topic).stream().map(info -> 
	new TopicPartition(topic, info.partition()))
	.forEach(tp -> {
	long committedOffset = consumer.committed(tp).offset();
	consumer.seek(tp, committedOffset);
});
```

**Specified-Offset实现**

```java
long targetOffset = 1234L;
for (PartitionInfo info : consumer.partitionsFor(topic)) {
	TopicPartition tp = new TopicPartition(topic, info.partition());
	consumer.seek(tp, targetOffset);
}
```

**Shift-By-N 实现**

```java
for (PartitionInfo info : consumer.partitionsFor(topic)) {
         TopicPartition tp = new TopicPartition(topic, info.partition());
	// 假设向前跳 123 条消息
         long targetOffset = consumer.committed(tp).offset() + 123L; 
         consumer.seek(tp, targetOffset);
}
```

**datatime 实现**

如果要实现 DateTime 策略，需要借助另一个方法：**KafkaConsumer.** **offsetsForTimes 方法**。假设要重设位移到 2019 年 6 月 20 日晚上 8 点，那么具体代码如下：

```java
long ts = LocalDateTime.of(
	2019, 6, 20, 20, 0).toInstant(ZoneOffset.ofHours(8)).toEpochMilli();
Map<TopicPartition, Long> timeToSearch = 
         consumer.partitionsFor(topic).stream().map(info -> 
	new TopicPartition(topic, info.partition()))
	.collect(Collectors.toMap(Function.identity(), tp -> ts));
 
for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : 
	consumer.offsetsForTimes(timeToSearch).entrySet()) {
consumer.seek(entry.getKey(), entry.getValue().offset());
}
```

**Duration 实现**

位移回调30分钟前

```java
Map<TopicPartition, Long> timeToSearch = consumer.partitionsFor(topic).stream()
         .map(info -> new TopicPartition(topic, info.partition()))
         .collect(Collectors.toMap(Function.identity(), tp -> System.currentTimeMillis() - 30 * 1000  * 60));
 
for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : 
         consumer.offsetsForTimes(timeToSearch).entrySet()) {
         consumer.seek(entry.getKey(), entry.getValue().offset());
}
```

**总之，使用 Java API 的方式来实现重设策略的主要入口方法，就是 seek 方法**。

#### 16.3.2 通过命令行设置

位移重设还有另一个重要的途径：**通过 kafka-consumer-groups 脚本**。需要注意的是，这个功能是在 Kafka 0.11 版本中新引入的。这就是说，如果你使用的 Kafka 是 0.11 版本之前的，那么你只能使用 API 的方式来重设位移。

**Earliest**

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-earliest –execute
```

**Latest**

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-latest --execute
```

**Current**

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-current --execute
```

**Specified-Offset** 

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --all-topics --to-offset <offset> --execute
```

 **Shift-By-N**

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --shift-by <offset_N> --execute
```

 **DateTime**

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --to-datetime 2019-06-20T20:00:00.000 --execute
```

**Duration**

```shell
bin/kafka-consumer-groups.sh --bootstrap-server kafka-host:port --group test-group --reset-offsets --by-duration PT0H30M0S --execute
```



## 17、无消息丢失配置

**一句话概括，Kafka 只对“已提交”的消息（committed message）做有限度的持久化保证。**

1、**已提交**

什么是已提交的消息？当 Kafka 的若干个 Broker 成功地接收到一条消息并写入到日志文件后，它们会告诉生产者程序这条消息已成功提交。此时，这条消息在 Kafka 看来就正式变为“已提交”消息了。

那为什么是若干个 Broker 呢？这取决于你对“已提交”的定义。你可以选择只要有一个 Broker 成功保存该消息就算是已提交，也可以是令所有 Broker 都成功保存该消息才算是已提交。不论哪种情况，Kafka 只对已提交的消息做持久化保证这件事情是不变的。

2、**有限度的持久化保证**

Kafka 不可能保证在任何情况下都做到不丢失消息(举个极端点的例子，如果地球都不存在了，Kafka 还能保存任何消息吗？)

```
假如你的消息保存在 N 个 Kafka Broker 上，那么这个前提条件就是这 N 个 Broker 中至少有 1 个存活。只要这个条件成立，Kafka 就能保证你的这条消息永远不会丢失。
```

### 17.1 producer 数据丢失

目前 Kafka Producer 是异步发送消息的，也就是说如果你调用的是 producer.send(msg) 这个 API，那么它通常会立即返回，但此时你不能认为消息发送已成功完成。

实际上，解决此问题的方法非常简单：**Producer 永远要使用带有回调通知的发送 API，也就是说不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)**。

不要小瞧这里的 callback（回调），它能准确地告诉你消息是否真的提交成功了。一旦出现消息提交失败的情况，你就可以有针对性地进行处理。

### 17.2 consumer 数据丢失

Consumer 端丢失数据主要体现在 Consumer 端要消费的消息不见了。Consumer 程序有个“位移”的概念，表示的是这个 Consumer 当前消费到的 Topic 分区的位置。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211181348182.png)

对于 Consumer A 而言，它当前的位移值就是 9；Consumer B 的位移值是 11。

```
这里的“位移”类似于我们看书时使用的书签，它会标记我们当前阅读了多少页，下次翻书的时候我们能直接跳到书签页继续阅读。

正确使用书签有两个步骤：第一步是读书，第二步是更新书签页。如果这两步的顺序颠倒了，就可能出现这样的场景：当前的书签页是第 90 页，我先将书签放到第 100 页上，之后开始读书。当阅读到第 95 页时，我临时有事中止了阅读。那么问题来了，当我下次直接跳到书签页阅读时，我就丢失了第 96～99 页的内容，即这些消息就丢失了。
```

如何解决？

**维持先消费消息（阅读），再更新位移（书签）的顺序**。(这种处理方式可能带来的问题是消息的重复处理，类似于同一页书被读了很多遍，但这不属于消息丢失的情形。)



**consumer丢失数据的另外一种场景：**

我们依然以看书为例。假设你花钱从网上租借了一本共有 10 章内容的电子书，该电子书的有效阅读时间是 1 天，过期后该电子书就无法打开，但如果在 1 天之内你完成阅读就退还租金。

为了加快阅读速度，你把书中的 10 个章节分别委托给你的 10 个朋友，请他们帮你阅读，并拜托他们告诉你主旨大意。当电子书临近过期时，这 10 个人告诉你说他们读完了自己所负责的那个章节的内容，于是你放心地把该书还了回去。不料，在这 10 个人向你描述主旨大意时，你突然发现有一个人对你撒了谎，他并没有看完他负责的那个章节。那么很显然，你无法知道那一章的内容了。

对于 Kafka 而言，这就好比 Consumer 程序从 Kafka 获取到消息后开启了多个线程异步处理消息，而 Consumer 程序自动地向前更新位移。假如其中某个线程运行失败了，它负责的消息没有被成功处理，但位移已经被更新了，因此这条消息对于 Consumer 而言实际上是丢失了。

这里的关键在于 Consumer 自动提交位移，与你没有确认书籍内容被全部读完就将书归还类似，你没有真正地确认消息是否真的被消费就“盲目”地更新了位移。

这个问题的解决方案也很简单：**如果是多线程异步处理消费消息，Consumer 程序不要开启自动提交位移，而是要应用程序手动提交位移**。在这里我要提醒你一下，单个 Consumer 程序使用多线程来消费消息说起来容易，写成代码却异常困难，因为你很难正确地处理位移的更新，也就是说避免无消费消息丢失很简单，但极易出现消息被消费了多次的情况。

## 18. 最佳实践

```
1、不要使用 producer.send(msg)，而要使用 producer.send(msg, callback)。记住，一定要使用带有回调通知的 send 方法。

2、设置 acks = all。acks 是 Producer 的一个参数，代表了你对“已提交”消息的定义。如果设置成 all，则表明所有副本 Broker 都要接收到消息，该消息才算是“已提交”。这是最高等级的“已提交”定义。

3、设置 retries 为一个较大的值。这里的 retries 同样是 Producer 的参数，对应前面提到的 Producer 自动重试。当出现网络的瞬时抖动时，消息发送可能会失败，此时配置了 retries > 0 的 Producer 能够自动重试消息发送，避免消息丢失。

4、设置 unclean.leader.election.enable = false。这是 Broker 端的参数，它控制的是哪些 Broker 有资格竞选分区的 Leader。如果一个 Broker 落后原先的 Leader 太多，那么它一旦成为新的 Leader，必然会造成消息的丢失。故一般都要将该参数设置成 false，即不允许这种情况的发生。

5、设置 replication.factor >= 3。这也是 Broker 端的参数。其实这里想表述的是，最好将消息多保存几份，毕竟目前防止消息丢失的主要机制就是冗余。

6、设置 min.insync.replicas > 1。这依然是 Broker 端参数，控制的是消息至少要被写入到多少个副本才算是“已提交”。设置成大于 1 可以提升消息持久性。在实际环境中千万不要使用默认值 1。

7、确保 replication.factor > min.insync.replicas。如果两者相等，那么只要有一个副本挂机，整个分区就无法正常工作了。我们不仅要改善消息的持久性，防止数据丢失，还要在不降低可用性的基础上完成。推荐设置成 replication.factor = min.insync.replicas + 1。

8、确保消息消费完成再提交。Consumer 端有个参数 enable.auto.commit，最好把它设置成 false，并采用手动提交位移的方式。就像前面说的，这对于单 Consumer 多线程处理的场景而言是至关重要的。
```























































































