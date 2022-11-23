# kafka详解一



## 1、消息引擎背景

```
根据维基百科的定义，消息引擎系统是一组规范。企业利用这组规范在不同系统之间传递语义准确的消息，实现松耦合的异步式数据传递.
即：系统 A 发送消息给消息引擎系统，系统 B 从消息引擎系统中读取 A 发送的消息。
```

```
消息引擎的分类：

点对点模型：也叫消息队列模型。如果拿上面那个“民间版”的定义来说，那么系统 A 发送的消息只能被系统 B 接收，其他任何系统都不能读取 A 发送的消息。日常生活的例子比如电话客服就属于这种模型：同一个客户呼入电话只能被一位客服人员处理，第二个客服人员不能为该客户服务。

发布 / 订阅模型：与上面不同的是，它有一个主题（Topic）的概念，你可以理解成逻辑语义相近的消息容器。该模型也有发送方和接收方，只不过提法不同。发送方也称为发布者（Publisher），接收方称为订阅者（Subscriber）。和点对点模型不同的是，这个模型可能存在多个发布者向相同的主题发送消息，而订阅者也可能存在多个，它们都能接收到相同主题的消息。生活中的报纸订阅就是一种典型的发布 / 订阅模型。
```

```
消息引擎和JMS的关系：
JMS 是 Java Message Service，它也是支持上面这两种消息引擎模型的。严格来说它并非传输协议而仅仅是一组 API 罢了。不过可能是 JMS 太有名气以至于很多主流消息引擎系统都支持 JMS 规范，比如 ActiveMQ、RabbitMQ、IBM 的 WebSphere MQ 和 Apache Kafka。当然 Kafka 并未完全遵照 JMS 规范，相反，它另辟蹊径，探索出了一条特有的道路。
```

## 2、 Kafka概述

### 2.1、kafka的定义：

kafka是一个分布式的、基于发布订阅模式的消息队列，主要应用于大数据实时处理领域。

```
PUBLISH & SUBSCRIBE
Read and write streams of data like a messaging system.

PROCESS
Write scalable stream processing applications that react to events in real-time.

STORE
Store streams of data safely in a distributed, replicated, fault-tolerant cluster.
```



### 2.2 为什么有消息系统

#### 2.1.1 异步处理

```
异步处理：
场景说明：用户注册后，需要发注册邮件和注册短信。传统的做法有两种：串行的方式和并行方式。

串行方式：将注册信息写入数据库成功后，发送注册邮件，再发送注册短信。以上三个任务全部完成后，返回给客户。
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181131933.png)

```
并行方式：将注册信息写入数据库成功后，发送注册邮件的同时，发送注册短信。以上三个任务完成后，返回给客户端。与串行的差别是，并行的方式可以提高处理的时间。
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181132311.png)

```
假设三个业务节点每个使用50毫秒钟，不考虑网络等其他开销，则串行方式的时间是150毫秒，并行的时间可能是100毫秒。
如以上案例描述，传统的方式系统的性能（并发量，吞吐量，响应时间）会有瓶颈。如何解决这个问题呢？

引入消息队列：
用户的响应时间相当于是注册信息写入数据库的时间，也就是50毫秒。注册邮件，发送短信写入消息队列后，直接返回，因为写入消息队列的速度很快，基本可以忽略，因此用户的响应时间可能是50毫秒。因此架构改变后，系统的吞吐量提高到每秒20QPS。比串行提高了3倍，比并行提高了
```



![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181132148.png)

#### 2.1.2 解耦

```
场景说明：用户下单后，订单系统需要通知库存系统。传统的做法是，订单系统调用库存系统的接口。如下图：
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181132065.png)

```
传统模式的缺点:

假如库存系统无法访问，则订单减库存将失败，从而导致订单失败，订单系统与库存系统耦合。

如何解决以上问题呢？引入应用消息队列后的方案，如下图：
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181133262.png)

```
订单系统：用户下单后，订单系统完成持久化处理，将消息写入消息队列，返回用户订单下单成功

库存系统：订阅下单的消息，采用拉/推的方式，获取下单信息，库存系统根据下单信息，进行库存操作

假如：在下单时库存系统不能正常使用。也不影响正常下单，因为下单后，订单系统写入消息队列就不再关心其他的后续操作了。实现订单系统与库存系统的应用解耦。
```



#### 2.1.3 流量削峰

```
流量削锋也是消息队列中的常用场景，一般在秒杀或团抢活动中使用广泛！

应用场景：秒杀活动，一般会因为流量过大，导致流量暴增，应用挂掉。为解决这个问题，一般需要在应用前端加入消息队列。

可以控制活动的人数，可以缓解短时间内高流量压垮应用。

用户的请求，服务器接收后，首先写入消息队列。假如消息队列长度超过最大数量，则直接抛弃用户请求或跳转到错误页面。

秒杀业务根据消息队列中的请求信息，再做后续处理。
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181133401.png)

#### 2.1.4 消息队列其它用处

**解耦**
允许你独立的扩展或修改两边的处理过程，只要确保它们遵守同样的接口约束。

**冗余**
消息队列把数据进行持久化直到它们已经被完全处理，通过这一方式规避了数据丢失风险。许多消息队列所采用的"插入-获取-删除"范式中，在把一个消息从队列中删除之前，需要你的处理系统明确的指出该消息已经被处理完毕，从而确保你的数据被安全的保存直到你使用完毕。

**扩展性**
因为消息队列解耦了你的处理过程，所以增大消息入队和处理的频率是很容易的，只要另外增加处理过程即可。

**灵活性 & 峰值处理能力**
在访问量剧增的情况下，应用仍然需要继续发挥作用，但是这样的突发流量并不常见。如果为以能处理这类峰值访问为标准来投入资源随时待命无疑是巨大的浪费。使用消息队列能够使关键组件顶住突发的访问压力，而不会因为突发的超负荷的请求而完全崩溃。

**可恢复性**
系统的一部分组件失效时，不会影响到整个系统。消息队列降低了进程间的耦合度，所以即使一个处理消息的进程挂掉，加入队列中的消息仍然可以在系统恢复后被处理。

**顺序保证**
在大多使用场景下，数据处理的顺序都很重要。大部分消息队列本来就是排序的，并且能保证数据会按照特定的顺序来处理。（Kafka 保证一个 Partition 内的消息的有序性）

**缓冲**
有助于控制和优化数据流经过系统的速度，解决生产消息和消费消息的处理速度不一致的情况。

**异步通信**
很多时候，用户不想也不需要立即处理消息。消息队列提供了异步处理机制，允许用户把一个消息放入队列，但并不立即处理它。想向队列中放入多少消息就放多少，然后在需要的时候再去处理它们。

### 2.3、 Kafka核心概念

~~~
	Kafka是最初由Linkedin公司开发，是一个分布式、分区的、多副本的、多订阅者，基于zookeeper协调的分布式日志系统（也可以当做MQ系统），常见可以用于web/nginx日志、访问日志，消息服务等等，Linkedin于2010年贡献给了Apache基金会并成为顶级开源项目。
	
	kafka是一个分布式消息队列。具有高性能、持久化、多副本备份、横向扩展能力。生产者往队列里写消息，消费者从队列里取消息进行业务逻辑。Kafka就是一种发布-订阅模式。将消息保存在磁盘中，以顺序读写方式访问磁盘，避免随机读写导致性能瓶颈。

~~~



### 2.4、 kafka特性

- `高吞吐、低延迟`

  ~~~
  kakfa 最大的特点就是收发消息非常快，kafka 每秒可以处理几十万条消息，它的最低延迟只有几毫秒。
  ~~~

- `高伸缩性`

  ~~~
   每个主题(topic) 包含多个分区(partition)，主题中的分区可以分布在不同的主机(broker)中。
  ~~~

- `持久性、可靠性`

  ~~~
  Kafka 能够允许数据的持久化存储，消息被持久化到磁盘，并支持数据备份防止数据丢失。
  ~~~

- `容错性`

  ~~~
   允许集群中的节点失败，某个节点宕机，Kafka 集群能够正常工作。
  ~~~

- `高并发`

  ~~~
  支持数千个客户端同时读写。
  ~~~



### 2.5、kafka核心模块解析

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181133366.png)

#### 1、生产者API

允许应用程序发布记录流至一个或者多个kafka的主题（topics）。

#### 2、消费者API

允许应用程序订阅一个或者多个主题，并处理这些主题接收到的记录流。

#### 3、StreamsAPI

允许应用程序充当流处理器（stream processor），从一个或者多个主题获取输入流，并生产一个输出流到一个或 者多个主题，能够有效的变化输入流为输出流。

#### 4、ConnectAPI

允许构建和运行可重用的生产者或者消费者



### 2.6、 Kafka集群架构

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181133045.png)



* producer

  ~~~
   消息生产者，发布消息到Kafka集群的终端或服务
  ~~~

* broker

  ~~~
  Kafka集群中包含的服务器，一个borker就表示kafka集群中的一个节点
  ~~~

* topic

  ~~~
  每条发布到Kafka集群的消息属于的类别，即Kafka是面向 topic 的。
  更通俗的说Topic就像一个消息队列，生产者可以向其写入消息，消费者可以从中读取消息，一个Topic支持多个生产者或消费者同时订阅它，所以其扩展性很好。
  ~~~

*  partition

  ~~~
  每个 topic 包含一个或多个partition。Kafka分配的单位是partition
  ~~~

* replication

  ~~~
  partition的副本，保障 partition 的高可用。
  ~~~

* consumer

  ~~~
  从Kafka集群中消费消息的终端或服务
  ~~~

* consumer group

  ~~~
  每个 consumer 都属于一个 consumer group，每条消息只能被 consumer group 中的一个 Consumer 消费，但可以被多个 consumer group 消费。
  ~~~

* leader

  ~~~
  每个partition有多个副本，其中有且仅有一个作为Leader，Leader是当前负责数据的读写的partition。 producer 和 consumer 只跟 leader 交互
  ~~~

* follower

  ~~~
  Follower跟随Leader，所有写请求都通过Leader路由，数据变更会广播给所有Follower，Follower与Leader保持数据同步。如果Leader失效，则从Follower中选举出一个新的Leader。
  ~~~



* controller

     ~~~
     	知道大家有没有思考过一个问题，就是Kafka集群中某个broker宕机之后，是谁负责感知到他的宕机，以及负责进行Leader Partition的选举？如果你在Kafka集群里新加入了一些机器，此时谁来负责把集群里的数据进行负载均衡的迁移？包括你的Kafka集群的各种元数据，比如说每台机器上有哪些partition，谁是leader，谁是follower，是谁来管理的？如果你要删除一个topic，那么背后的各种partition如何删除，是谁来控制？还有就是比如Kafka集群扩容加入一个新的broker，是谁负责监听这个broker的加入？如果某个broker崩溃了，是谁负责监听这个broker崩溃？这里就需要一个Kafka集群的总控组件，Controller。他负责管理整个Kafka集群范围内的各种东西。

     ~~~

* zookeeper

     ~~~
     (1)	Kafka 通过 zookeeper 来存储集群的meta元数据信息
     (2)一旦controller所在broker宕机了，此时临时节点消失，集群里其他broker会一直监听这个临时节点，发现临时节点消失了，就争抢再次创建临时节点，保证有一台新的broker会成为controller角色。
     ~~~

* offset

  * 偏移量

  ~~~
  消费者在对应分区上已经消费的消息数（位置），offset保存的地方跟kafka版本有一定的关系。
  kafka0.8 版本之前offset保存在zookeeper上。
  kafka0.8 版本之后offset保存在kafka集群上。
  	它是把消费者消费topic的位置通过kafka集群内部有一个默认的topic，
  	名称叫 __consumer_offsets，它默认有50个分区。
  ~~~

* ISR机制

  ~~~
  	光是依靠多副本机制能保证Kafka的高可用性，但是能保证数据不丢失吗？不行，因为如果leader宕机，但是leader的数据还没同步到follower上去，此时即使选举了follower作为新的leader，当时刚才的数据已经丢失了。

  	ISR是：in-sync replica，就是跟leader partition保持同步的follower partition的数量，只有处于ISR列表中的follower才可以在leader宕机之后被选举为新的leader，因为在这个ISR列表里代表他的数据跟leader是同步的。
  ~~~

## 3. kafka集群安装部署

- 1、下载安装包（http://kafka.apache.org）

  ~~~
  kafka_2.11-1.1.0.tgz
  ~~~

- 2、规划安装目录

  ~~~
  /opt/install
  ~~~

- 3、上传安装包到node01服务器，并解压

  ~~~
  通过FTP工具上传安装包到node01服务器的/opt/soft路径下，然后进行解压
  cd /opt/soft/
  tar -zxf kafka_2.11-1.1.0.tgz -C /opt/install/

  ~~~

- 4、修改配置文件

  - 在node01上修改

    - 进入到kafka安装目录下有一个config目录，进行修改配置文件

      - node01执行以下命令进行修改配置文件

      ```properties
      cd  /opt/install/kafka_2.11-1.1.0/config
      vim server.properties

      #指定kafka对应的broker id ，唯一
      broker.id=0
      #指定数据存放的目录
      log.dirs=/opt/install/kafka_2.11-1.1.0/logs
      #指定zk地址
      zookeeper.connect=node01:2181,node02:2181,node03:2181
      #指定是否可以删除topic ,默认是false 表示不可以删除
      delete.topic.enable=true
      #指定broker主机名
      host.name=node01

      ```

- 5、node01执行以下命令分发kafka安装目录到其他节点

  ```shell
  cd /opt/install/
  scp -r kafka_2.11-1.1.0/ node02:$PWD
  scp -r kafka_2.11-1.1.0/ node03:$PWD
  ```

- 6、修改node02和node03上的配置

  - node02执行以下命令进行修改配置

    ```properties
    cd /opt/install/kafka_2.11-1.1.0/config/
    vi server.properties

    #指定kafka对应的broker id ，唯一
    broker.id=1
    #指定数据存放的目录
    log.dirs=/opt/install/kafka_2.11-1.1.0/logs
    #指定zk地址
    zookeeper.connect=node01:2181,node02:2181,node03:2181
    #指定是否可以删除topic ,默认是false 表示不可以删除
    delete.topic.enable=true
    #指定broker主机名
    host.name=node02
    ```
  ```

  ```

- node03执行以下命令进行修改配置

  ```properties
  cd /opt/install/kafka_2.11-1.1.0/config/
    vi server.properties
    
    #指定kafka对应的broker id ，唯一
    broker.id=2
    #指定数据存放的目录
    log.dirs=/opt/install/kafka_2.11-1.1.0/logs
    #指定zk地址
    zookeeper.connect=node01:2181,node02:2181,node03:2181
    #指定是否可以删除topic ,默认是false 表示不可以删除
    delete.topic.enable=true
    #指定broker主机名
    host.name=node03
  ```


### 3.1、 kafka集群启动和停止

#### 3.1.1、 启动

- 先启动zk集群

- 然后在所有节点执行脚本

  ```shell
  cd /opt/install/kafka_2.11-1.1.0/
  nohup bin/kafka-server-start.sh config/server.properties 2>&1 & 

  ```

- 一键启动kafka

  - start_kafka.sh

    ```shell
    #!/bin/sh
    for host in node01 node02 node03
    do
            ssh $host "source /etc/profile;nohup /opt/install/kafka_2.11-1.1.0/bin/kafka-server-start.sh /opt/install/kafka_2.11-1.1.0/config/server.properties >/dev/null 2>&1 &"
            echo "$host kafka is running"

    done
    ```


#### 3.2.1、  停止

- 所有节点执行关闭kafka脚本

  ```
  cd /opt/install/kafka_2.11-1.1.0/
  bin/kafka-server-stop.sh 
  ```

- 一键停止kafka

  - stop_kafka.sh

    ```shell
    #!/bin/sh
    for host in node01 node02 node03
    do
      ssh $host "source /etc/profile;nohup /opt/install/kafka_2.11-1.1.0/bin/kafka-server-stop.sh &" 
      echo "$host kafka is stopping"
    done
    ```

#### 3.3.1、 一键启动和停止脚本

* kafkaCluster.sh 

  ~~~shell
  #!/bin/sh
  case $1 in 
  "start"){
  for host in node01 node02 node03 
  do
    ssh $host "source /etc/profile; nohup /opt/install/kafka_2.11-1.1.0/bin/kafka-server-start.sh /opt/install/kafka_2.11-1.1.0/config/server.properties > /dev/null 2>&1 &"   
    echo "$host kafka is running..."  
  done  
  };;

  "stop"){
  for host in node01 node02 node03 
  do
    ssh $host "source /etc/profile; nohup /opt/install/kafka_2.11-1.1.0/bin/kafka-server-stop.sh >/dev/null  2>&1 &"   
    echo "$host kafka is stopping..."  
  done
  };;
  esac
  ~~~

* 启动

  ~~~shell
  sh kafkaCluster.sh start
  ~~~

* 停止

  ~~~shell
  sh kafkaCluster.sh stop
  ~~~



##  4、 kafka的命令行的管理使用

- 1、创建topic

  - kafka-topics.sh

  - node01执行以下命令创建topic

    ```shell
    cd /opt/install/kafka_2.11-1.1.0/
    bin/kafka-topics.sh --create --partitions 3 --replication-factor 2 --topic test --zookeeper node01:2181,node02:2181,node03:2181
    ```

- 2、查询所有的topic

  - kafka-topics.sh

    ```shell
    cd /opt/install/kafka_2.11-1.1.0/
    bin/kafka-topics.sh --list --zookeeper node01:2181,node02:2181,node03:2181 
    ```



* 3、查看topic的描述信息

  * kafka-topics.sh

    ~~~shell
    cd /opt/install/kafka_2.11-1.1.0/
    bin/kafka-topics.sh --describe --topic test --zookeeper node01:2181,node02:2181,node03:2181  
    ~~~

* 4、删除topic

  * kafka-topics.sh

    ~~~shell
    cd /opt/install/kafka_2.11-1.1.0/
    bin/kafka-topics.sh --delete --topic test --zookeeper node01:2181,node02:2181,node03:2181 
    ~~~


- 5、node01模拟生产者写入数据到topic中

  ​		node01执行以下命令，模拟生产者写入数据到kafka当中去

  ```shell
  cd /opt/install/kafka_2.11-1.1.0/
  bin/kafka-console-producer.sh --broker-list node01:9092,node02:9092,node03:9092 --topic test 
  ```



- 6、node01模拟消费者拉取topic中的数据

  ​	node02执行以下命令，模拟消费者消费kafka当中的数据

  ```shell
  cd /opt/install/kafka_2.11-1.1.0/
  bin/kafka-console-consumer.sh --zookeeper node01:2181,node02:2181,node03:2181 --topic test --from-beginning
  它会把消息的偏移量保存在zk上

  或者

  cd /opt/install/kafka_2.11-1.1.0/
  bin/kafka-console-consumer.sh --bootstrap-server node01:9092,node02:9092,node03:9092 --topic test --from-beginning
  它会把消息的偏移量保存在kafka集群内置的topic中
  ```



7、任意kafka服务器执行以下命令可以增加topic分区数

```
cd /opt/install/kafka_2.11-1.1.0

bin/kafka-topics.sh --zookeeper zkhost:port --alter --topic topicName --partitions 8
```







##  5、kafka的生产者和消费者api代码开发

### 5.1 生产者代码开发

* 创建maven工程引入依赖

  ~~~xml
   <dependencies>
          <dependency>
              <groupId>org.apache.kafka</groupId>
              <artifactId>kafka-clients</artifactId>
              <version>1.0.1</version>
          </dependency>
          <!-- https://mvnrepository.com/artifact/org.apache.kafka/kafka -->
          <dependency>
              <groupId>org.apache.kafka</groupId>
              <artifactId>kafka_2.11</artifactId>
              <version>1.1.0</version>
          </dependency>
      </dependencies>
      <build>
          <plugins>
              <plugin>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-compiler-plugin</artifactId>
                  <version>3.0</version>
                  <configuration>
                      <source>1.8</source>
                      <target>1.8</target>
                      <encoding>UTF-8</encoding>
                      <!--    <verbal>true</verbal>-->
                  </configuration>
              </plugin>
              <plugin>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-shade-plugin</artifactId>
                  <version>2.4.3</version>
                  <executions>
                      <execution>
                          <phase>package</phase>
                          <goals>
                              <goal>shade</goal>
                          </goals>
                          <configuration>
                              <filters>
                                  <filter>
                                      <artifact>*:*</artifact>
                                      <excludes>
                                          <exclude>META-INF/*.SF</exclude>
                                          <exclude>META-INF/*.DSA</exclude>
                                          <exclude>META-INF/*.RSA</exclude>
                                      </excludes>
                                  </filter>
                              </filters>
                              <transformers>
                                  <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                      <mainClass></mainClass>
                                  </transformer>
                              </transformers>
                          </configuration>
                      </execution>
                  </executions>
              </plugin>
          </plugins>
      </build>
  ~~~

* 代码开发

  ~~~java
  import org.apache.kafka.clients.producer.KafkaProducer;
  import org.apache.kafka.clients.producer.ProducerRecord;

  import java.util.Properties;

  public class KafkaProducerStudy {
      /**
       * 通过javaAPI实现向kafka当中生产数据
       * @param args
       */
      public static void main(String[] args) {
          Properties props = new Properties();
          props.put("bootstrap.servers", "node01:9092,node02:9092,node03:9092");
          //消息的确认机制
          props.put("acks", "all");
          props.put("retries", 0);
          //缓冲区的大小  //默认32M
          props.put("buffer.memory", 33554432);
          //批处理数据的大小，每次写入多少数据到topic   //默认16KB
          props.put("batch.size", 16384);
          //可以延长多久发送数据   //默认为0 表示不等待 ，立即发送
          props.put("linger.ms", 1);
          props.put("buffer.memory", 33554432);
          //指定数据序列化和反序列化
          props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
          props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
          KafkaProducer<String, String> producer = new KafkaProducer<String,String>(props);
          for(int i =0;i<100;i++){
              //既没有指定分区号，也没有数据的key，直接使用轮序的方式将数据发送到各个分区里面去
              ProducerRecord record = new ProducerRecord("test", "helloworld" + i);
              producer.send(record);
          }
          //关闭消息发送客户端
          producer.close();
      }
  }
  ~~~



### 5.2 消费者代码开发

#### 5.2.1、自动提交偏移量代码开发

~~~java
package com.xichuan.consumer;

import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.Arrays;
import java.util.Properties;

//todo:需求：开发kafka消费者代码（自动提交偏移量）
public class KafkaConsumerStudy {
    public static void main(String[] args) {
        //准备配置属性
        Properties props = new Properties();
        //kafka集群地址
        props.put("bootstrap.servers", "node01:9092,node02:9092,node03:9092");
        //消费者组id
        props.put("group.id", "consumer-test");
        //允许自动提交偏移量
        props.put("enable.auto.commit", "true");
        //自动提交偏移量的时间间隔
        props.put("auto.commit.interval.ms", "1000");
        //默认是latest
        //earliest: 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，从头开始消费
        //latest: 当各分区下有已提交的offset时，从提交的offset开始消费；无提交的offset时，消费新产生的该分区下的数据
        //none : topic各分区都存在已提交的offset时，从offset后开始消费；只要有一个分区不存在已提交的offset，则抛出异常
        props.put("auto.offset.reset","earliest");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        //指定消费哪些topic
        consumer.subscribe(Arrays.asList("test"));
        while (true) {
            //不断的拉取数据
            ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records) {
                //该消息所在的分区号
                int partition = record.partition();
                //该消息对应的key
                String key = record.key();
                //该消息对应的偏移量
                long offset = record.offset();
                //该消息内容本身
                String value = record.value();
                System.out.println("partition:"+partition+"\t key:"+key+"\toffset:"+offset+"\tvalue:"+value);
            }
        }
    }
}

~~~

#### 5.2.2、手动提交偏移量代码开发

~~~java
package com.xichuan.consumer;


import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Properties;

//todo:需求：开发kafka消费者代码（手动提交偏移量）
public class KafkaConsumerControllerOffset {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "node01:9092,node02:9092,node03:9092");
        props.put("group.id", "controllerOffset");
        //关闭自动提交，改为手动提交偏移量
        props.put("enable.auto.commit", "false");
        props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        //指定消费者要消费的topic
        consumer.subscribe(Arrays.asList("test"));

        //定义一个数字，表示消息达到多少后手动提交偏移量
        final int minBatchSize = 20; //条  20*16k

        //定义一个数组，缓冲一批数据
        List<ConsumerRecord<String, String>> buffer = new ArrayList<ConsumerRecord<String, String>>();
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(100);//100毫秒超时时间
            for (ConsumerRecord<String, String> record : records) {
                buffer.add(record);
            }
            if (buffer.size() >= minBatchSize) {
                //insertIntoDb(buffer);  拿到数据之后，进行消费
                System.out.println("缓冲区的数据条数："+buffer.size());
                System.out.println("我已经处理完这一批数据了...");
                consumer.commitSync();//手动提交
                buffer.clear();
            }
        }
    }
}

~~~

#### 5.2.3、指定分区数据进行消费

因为每个topic都可能有多个分区，所以我们也可以针对指定的分区进行消费

```scala
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.TopicPartition;

import java.util.Arrays;
import java.util.Properties;

public class ConsumPartition {
    public static void main(String[] args) {
        Properties props= new Properties();
        //指定kafka的broker的通信地址
        props.put("bootstrap.servers","localhost:9092"); props.put("group.id", "test");
        props.put("enable.auto.commit","true");
        props.put("auto.commit.interval.ms","1000");
        props.put("key.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
        props.put("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer");
        KafkaConsumer<String,String> consumer = new KafkaConsumer<>(props);
        String topic ="foo";
        TopicPartition partition0 = new TopicPartition(topic, 0);
        TopicPartition partition1 = new TopicPartition(topic, 1);
        //通过assign来注册仅仅消费某些分区的数据
        consumer.assign(Arrays.asList(partition0, partition1));
//手动指定消费指定分区的数据---end
        while (true) {
            ConsumerRecords<String,String> records = consumer.poll(100);
            for(ConsumerRecord<String, String> record : records)
                System.out.printf("offset= %d, key = %s, value = %s%n", record.offset(), record.key(),record.value());
        }
    }
}
```



## 6、kafka的分区策略

### 6.1、 分区的概念

Kafka 有主题（Topic）的概念，它是承载真实数据的逻辑容器，而在主题之下还分为若干个分区，也就是说 Kafka 的消息组织方式实际上是三级结构：主题 - 分区 - 消息。主题下的每条消息只会保存在某一个分区中，而不会在多个分区中被保存多份.

​	![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181134652.png)

```
对数据进行分区的主要原因，就是为了实现系统的高伸缩性（Scalability）。不同的分区能够被放置到不同节点的机器上，而数据的读写操作也都是针对分区这个粒度而进行的，这样每个节点的机器都能独立地执行各自分区的读写请求处理。并且，我们还可以通过添加新的节点机器来增加整体系统的吞吐量。

比如在 Kafka 中叫分区，在 MongoDB 和 Elasticsearch 中就叫分片 Shard，而在 HBase 中则叫 Region，在 Cassandra 中又被称作 vnode。从表面看起来它们实现原理可能不尽相同，但对底层分区（Partitioning）的整体思想却从未改变。

除了提供负载均衡这种最核心的功能之外，利用分区也可以实现其他一些业务级别的需求，比如实现业务级别的消息顺序的问题。
```



### 6.2、 分区策略

所谓分区策略是决定生产者将消息发送到哪个分区的算法。Kafka 为我们提供了默认的分区策略，同时它也支持你自定义分区策略.

```java
public interface Partitioner extends Configurable, Closeable {

    /**
     * Compute the partition for the given record.
     *
     * @param topic The topic name
     * @param key The key to partition on (or null if no key)
     * @param keyBytes The serialized key to partition on( or null if no key)
     * @param value The value to partition on or null
     * @param valueBytes The serialized value to partition on or null
     * @param cluster The current cluster metadata
     */
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);

    /**
     * This is called when partitioner is closed.
     */
    public void close();

}
```

##### 1.2.1 轮训策略

也称 Round-robin 策略，即顺序分配。比如一个主题下有 3 个分区，那么第一条消息被发送到分区 0，第二条被发送到分区 1，第三条被发送到分区 2，以此类推。当生产第 4 条消息时又会重新开始，即将其分配到分区 0，就像下面这张图展示的那样。

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181134765.png)

这就是所谓的轮询策略。轮询策略是 Kafka Java 生产者 API 默认提供的分区策略。如果你未指定`partitioner.class`参数，那么你的生产者程序会按照轮询的方式在主题的所有分区间均匀地“码放”消息。

**轮询策略有非常优秀的负载均衡表现，它总是能保证消息最大限度地被平均分配到所有分区上，故默认情况下它是最合理的分区策略，也是我们最常用的分区策略之一。**

##### 1.2.2 随机策略

也称 Randomness 策略。所谓随机就是我们随意地将消息放置到任意一个分区上，如下面这张图所示。



![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181135231.png)

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return ThreadLocalRandom.current().nextInt(partitions.size());
```

本质上看随机策略也是力求将数据均匀地打散到各个分区，但从实际表现来看，它要逊于轮询策略，所以**如果追求数据的均匀分布，还是使用轮询策略比较好**。事实上，随机策略是老版本生产者使用的分区策略，在新版本中已经改为轮询了。

##### 1.2.3 按照消息key保存

Kafka 允许为每条消息定义消息键，简称为 Key。这个 Key 的作用非常大，它可以是一个有着明确业务含义的字符串，比如客户代码、部门编号或是业务 ID 等；也可以用来表征消息元数据。特别是在 Kafka 不支持时间戳的年代，在一些场景中，工程师们都是直接将消息创建时间封装进 Key 里面的。一旦消息被定义了 Key，那么你就可以保证同一个 Key 的所有消息都进入到相同的分区里面，由于每个分区下的消息处理都是有顺序的，故这个策略被称为按消息键保序策略，如下图所示。

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181136804.png)

```
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return Math.abs(key.hashCode()) % partitions.size();
```



Kafka 默认分区策略实际上同时实现了两种策略：如果指定了 Key，那么默认实现按消息键保序策略；如果没有指定 Key，则使用轮询策略。

##### 1.2.4 基于地理位置的分区策略

这种策略一般只针对那些大规模的 Kafka 集群，特别是跨城市、跨国家甚至是跨大洲的集群。

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
return partitions.stream().filter(p -> isSouth(p.leader().host())).map(PartitionInfo::partition).findAny().get();
```

可以从所有分区中找出那些 Leader 副本在南方的所有分区，然后随机挑选一个进行消息发送。



##### 1.3 用户自定义分区

```
kafka的分区策略决定了producer生产者产生的一条消息最后会写入到topic的哪一个分区中
```

```java
/**
     * Creates a record with a specified timestamp to be sent to a specified topic and partition
     * 
     * @param topic The topic the record will be appended to
     * @param partition The partition to which the record should be sent
     * @param timestamp The timestamp of the record, in milliseconds since epoch. If null, the producer will assign
     *                  the timestamp using System.currentTimeMillis().
     * @param key The key that will be included in the record
     * @param value The record contents
     * @param headers the headers that will be included in the record
     */
    public ProducerRecord(String topic, Integer partition, Long timestamp, K key, V value, Iterable<Header> headers) {
        if (topic == null)
            throw new IllegalArgumentException("Topic cannot be null.");
        if (timestamp != null && timestamp < 0)
            throw new IllegalArgumentException(
                    String.format("Invalid timestamp: %d. Timestamp should always be non-negative or null.", timestamp));
        if (partition != null && partition < 0)
            throw new IllegalArgumentException(
                    String.format("Invalid partition: %d. Partition number should always be non-negative or null.", partition));
        this.topic = topic;
        this.partition = partition;
        this.key = key;
        this.value = value;
        this.timestamp = timestamp;
        this.headers = new RecordHeaders(headers);
    }
```





- 1、指定具体的分区号

```java
//1、给定具体的分区号，数据就会写入到指定的分区中
producer.send(new ProducerRecord<String, String>("test", 0,Integer.toString(i), "hello-kafka-"+i));

```

- 2、不给定具体的分区号，给定key的值（key不断变化）

```java
//2、不给定具体的分区号，给定一个key值, 这里使用key的 hashcode%分区数=分区号
producer.send(new ProducerRecord<String, String>("test", Integer.toString(i), "hello-kafka-"+i));
```

- 3、不给定具体的分区号，也不给对应的key

```java
//3、不给定具体的分区号，也不给定对应的key ,这个它会进行轮训的方式把数据写入到不同分区中
producer.send(new ProducerRecord<String, String>("test", "hello-kafka-"+i));
```

- 4、自定义分区

  - 定义一个类实现接口Partitioner

  ```java
  package com.xichuan.partitioner;

  import org.apache.kafka.clients.producer.Partitioner;
  import org.apache.kafka.common.Cluster;

  import java.util.Map;

  //todo:需求：自定义kafka的分区函数
  public class MyPartitioner implements Partitioner{
      /**
       * 通过这个方法来实现消息要去哪一个分区中
       * @param topic
       * @param key
       * @param bytes
       * @param value
       * @param bytes1
       * @param cluster
       * @return
       */
      public int partition(String topic, Object key, byte[] bytes, Object value, byte[] bytes1, Cluster cluster) {
          //获取topic分区数
          int partitions = cluster.partitionsForTopic(topic).size();
          
          //key.hashCode()可能会出现负数 -1 -2 0 1 2
          //Math.abs 取绝对值
          return Math.abs(key.hashCode()% partitions);

      }

      public void close() {
          
      }

      public void configure(Map<String, ?> map) {

      }
  }

  ```

  - 配置自定义分区类

  ```java
  //在Properties对象中添加自定义分区类
  props.put("partitioner.class","com.xichuan.partitioner.MyPartitioner");
  ```

​        分区是实现负载均衡以及高吞吐量的关键，故在生产者这一端就要仔细盘算合适的分区策略，避免造成消息数据的“倾斜”，使得某些分区成为性能瓶颈，这样极易引发下游数据消费的性能下降



## 7、kafka 消息压缩

压缩就是用时间去换空间的经典 trade-off 思想，具体来说就是用 CPU 时间去换磁盘空间或网络 I/O 传输量，希望以较小的 CPU 开销带来更少的磁盘占用或更少的网络 I/O 传输。在 Kafka 中，压缩也是用来做这件事的。

### 7.1 何时压缩

在 Kafka 中，压缩可能发生在两个地方：生产者端和 Broker 端。

生产者程序中配置 compression.type 参数即表示启用指定类型的压缩算法。比如下面这段程序代码展示了如何构建一个开启 GZIP 的 Producer 对象：

```java
Properties props = new Properties();
 props.put("bootstrap.servers", "localhost:9092");
 props.put("acks", "all");
 props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
 props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
 // 开启 GZIP 压缩
 props.put("compression.type", "gzip");
 
 Producer<String, String> producer = new KafkaProducer<>(props);
```

这里比较关键的代码行是 props.put(“compression.type”, “gzip”)，它表明该 Producer 的压缩算法使用的是 GZIP。这样 Producer 启动后生产的每个消息集合都是经 GZIP 压缩过的，故而能很好地节省网络传输带宽以及 Kafka Broker 端的磁盘占用。

```
 Broker 端也有一个参数叫 compression.type，和上面那个例子中的同名。但是这个参数的默认值是 producer，这表示 Broker 端会“尊重”Producer 端使用的压缩算法。可一旦你在 Broker 端设置了不同的 compression.type 值，就一定要小心了，因为可能会发生预料之外的压缩 / 解压缩操作，通常表现为 Broker 端 CPU 使用率飙升。
```

### 7.2 何时解压缩

通常来说解压缩发生在消费者程序中，也就是说 Producer 发送压缩消息到 Broker 后，Broker 照单全收并原样保存起来。当 Consumer 程序请求这部分消息时，Broker 依然原样发送出去，当消息到达 Consumer 端后，由 Consumer 自行解压缩还原成之前的消息。



那么现在问题来了，Consumer 怎么知道这些消息是用何种压缩算法压缩的呢？其实答案就在消息中。Kafka 会将启用了哪种压缩算法封装进消息集合中，这样当 Consumer 读取到消息集合时，它自然就知道了这些消息使用的是哪种压缩算法。如果用一句话总结一下压缩和解压缩，那么我希望你记住这句话：**Producer 端压缩、Broker 端保持、Consumer 端解压缩。**



除了在 Consumer 端解压缩，Broker 端也会进行解压缩。每个压缩过的消息集合在 Broker 端写入时都要发生解压缩操作，目的就是为了对消息执行各种验证。我们必须承认这种解压缩对 Broker 端性能是有一定影响的，特别是对 CPU 的使用率而言。

### 7.3 各种压缩算法对比

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181136609.png)

在实际使用中，GZIP、Snappy、LZ4 甚至是 zstd 的表现各有千秋。但对于 Kafka 而言，它们的性能测试结果却出奇得一致，即在吞吐量方面：LZ4 > Snappy > zstd 和 GZIP；而在压缩比方面，zstd > LZ4 > GZIP > Snappy。具体到物理资源，使用 Snappy 算法占用的网络带宽最多，zstd 最少，这是合理的，毕竟 zstd 就是要提供超高的压缩比；在 CPU 使用率方面，各个算法表现得差不多，只是在压缩时 Snappy 算法使用的 CPU 较多一些，而在解压缩时 GZIP 算法则可能使用更多的 CPU。



### 7.4 最佳实践

```
1、Producer 端完成的压缩，那么启用压缩的一个条件就是 Producer 程序运行机器上的 CPU 资源要很充足。如果 Producer 运行机器本身 CPU 已经消耗殆尽了，那么启用消息压缩无疑是雪上加霜，只会适得其反

2、除了 CPU 资源充足这一条件，如果你的环境中带宽资源有限，那么也建议你开启压缩。如果客户端机器 CPU 资源有很多富余，我强烈建议你开启 zstd 压缩，这样能极大地节省网络资源消耗。

3、解压缩端，尽量保证不要出现消息格式转换的情况
```



## 8、消费者组

### 8.1 消费者组的基本概念

**Consumer Group 是 Kafka 提供的可扩展且具有容错性的消费者机制**。

消费者组内必然可以有多个消费者或消费者实例（Consumer Instance），它们共享一个公共的 ID，这个 ID 被称为 Group ID。组内的所有消费者协调在一起来消费订阅主题（Subscribed Topics）的所有分区（Partition）。当然，每个分区只能由同一个消费者组内的一个 Consumer 实例来消费。

```
1、Consumer Group 下可以有一个或多个 Consumer 实例。这里的实例可以是一个单独的进程，也可以是同一进程下的线程。在实际场景中，使用进程更为常见一些。
2、Group ID 是一个字符串，在一个 Kafka 集群中，它标识唯一的一个 Consumer Group。
3、Consumer Group 下所有实例订阅的主题的单个分区，只能分配给组内的某个 Consumer 实例消费。这个分区当然也可以被其他的 Group 消费。
```

**Kafka 仅仅使用 Consumer Group 这一种机制，却同时实现了传统消息引擎系统的两大模型**（P2P/PubSub）：如果所有实例都属于同一个 Group，那么它实现的就是消息队列模型；如果所有实例分别属于不同的 Group，那么它实现的就是发布 / 订阅模型。



在实际使用场景中，怎么知道一个 Group 下该有多少个 Consumer 实例呢？理想情况下，Consumer 实例的数量应该等于该 Group 订阅主题的分区总数。

```
举个简单的例子，假设一个 Consumer Group 订阅了 3 个主题，分别是 A、B、C，它们的分区数依次是 1、2、3，那么通常情况下，为该 Group 设置 6 个 Consumer 实例是比较理想的情形，因为它能最大限度地实现高伸缩性。
```



针对 Consumer Group，Kafka 是怎么管理位移的呢？

```
老版本的 Consumer Group 把位移保存在 ZooKeeper 中。Apache ZooKeeper 是一个分布式的协调服务框架，Kafka 重度依赖它实现各种各样的协调管理。将位移保存在 ZooKeeper 外部系统的做法，最显而易见的好处就是减少了 Kafka Broker 端的状态保存开销。现在比较流行的提法是将服务器节点做成无状态的，这样可以自由地扩缩容，实现超强的伸缩性。Kafka 最开始也是基于这样的考虑，才将 Consumer Group 位移保存在独立于 Kafka 集群之外的框架中。

不过，慢慢地人们发现了一个问题，即 ZooKeeper 这类元框架其实并不适合进行频繁的写更新，而 Consumer Group 的位移更新却是一个非常频繁的操作。这种大吞吐量的写操作会极大地拖慢 ZooKeeper 集群的性能，因此 Kafka 社区渐渐有了这样的共识：将 Consumer 位移保存在 ZooKeeper 中是不合适的做法。

在新版本的 Consumer Group 中，Kafka 社区重新设计了 Consumer Group 的位移管理方式，采用了将位移保存在 Kafka 内部主题的方法。即：__consumer_offsets。
```

### 8.2 consumer Group rebalance

#### 8.2.1 rebalance介绍

**Rebalance 本质上是一种协议，规定了一个 Consumer Group 下的所有 Consumer 如何达成一致，来分配订阅 Topic 的每个分区**。比如某个 Group 下有 20 个 Consumer 实例，它订阅了一个具有 100 个分区的 Topic。正常情况下，Kafka 平均会为每个 Consumer 分配 5 个分区。这个分配的过程就叫 Rebalance。

```
那么 Consumer Group 何时进行 Rebalance 呢？Rebalance 的触发条件有 3 个。

1、组成员数发生变更。比如有新的 Consumer 实例加入组或者离开组，抑或是有 Consumer 实例崩溃被“踢出”组。
2、订阅主题数发生变更。Consumer Group 可以使用正则表达式的方式订阅主题，比如 consumer.subscribe(Pattern.compile(“t.*c”)) 就表明该 Group 订阅所有以字母 t 开头、字母 c 结尾的主题。在 Consumer Group 的运行过程中，你新创建了一个满足这样条件的主题，那么该 Group 就会发生 Rebalance。
3、订阅主题的分区数发生变更。Kafka 当前只能允许增加一个主题的分区数。当分区数增加时，就会触发订阅该主题的所有 Group 开启 Rebalance。
```

举个简单的例子来说明一下 Consumer Group 发生 Rebalance 的过程。假设目前某个 Consumer Group 下有两个 Consumer，比如 A 和 B，当第三个成员 C 加入时，Kafka 会触发 Rebalance，并根据默认的分配策略重新为 A、B 和 C 分配分区，如下图所示：

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211181136441.png)



Rebalance 之后的分配依然是公平的，即每个 Consumer 实例都获得了 3 个分区的消费权。这是我们希望出现的情形。

#### 8.2.2 rebalance的问题

1、Rebalance 过程对 Consumer Group 消费过程有极大的影响。如果你了解 JVM 的垃圾回收机制，你一定听过万物静止的收集方式，即著名的 stop the world，简称 STW。在 STW 期间，所有应用线程都会停止工作，表现为整个应用程序僵在那边一动不动。Rebalance 过程也和这个类似，在 Rebalance 过程中，所有 Consumer 实例都会停止消费，等待 Rebalance 完成。这是 Rebalance 为人诟病的一个方面。

2、目前 Rebalance 的设计是所有 Consumer 实例共同参与，全部重新分配所有分区。其实更高效的做法是尽量减少分配方案的变动。例如实例 A 之前负责消费分区 1、2、3，那么 Rebalance 之后，如果可能的话，最好还是让实例 A 继续消费分区 1、2、3，而不是被重新分配其他的分区。这样的话，实例 A 连接这些分区所在 Broker 的 TCP 连接就可以继续用，不用重新创建连接其他 Broker 的 Socket 资源。

3、Rebalance 实在是太慢了。曾经，有个国外用户的 Group 内有几百个 Consumer 实例，成功 Rebalance 一次要几个小时！这完全是不能忍受的。最悲剧的是，目前社区对此无能为力，至少现在还没有特别好的解决方案。所谓“本事大不如不摊上”，也许最好的解决方案就是避免 Rebalance 的发生吧。































































