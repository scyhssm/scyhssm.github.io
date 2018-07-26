---
layout:    post
title:      "kafka的了解使用"
subtitle:   "Use of kafka"
date:       2018-01-18 12:00:00
author:     "scyhssm"
header-img: "img/kafka.png"
header-mask: 0.3
catalog:    true
tags:
    - kafka
---

>kafka是hadoop生态内不可或缺的一环，是学习高并发分布式必不可少的一项技术，这篇文章通过实例揭开kafka神秘的面纱，部分内容摘取自英文官方文档。

## Kafka的介绍
Kafka是一个分布式流处理平台，可以这么理解：
* Kafka能够让你发布和订阅流式的记录
* Kafka让你能够容错地存储流式记录
* Kafka让你能够处理流式的数据
Kafka擅长在系统和应用间构建实时的可依赖的数据管道，构造实时的流应用转换或者应对数据流。

Kafka运行在集群状态下，kafka集群存储记录的流称为topics，每个记录由key，value和时间戳组成。

Kafka在client和server之间的交流用的是TCP协议。

Kafka的一个topic分多个partition，每个partition对应一个集群，由一个leader管理，其余follower复制leader数据。

partition的好处可以方便的扩展topic容量，也可以作为并行单位，用于高并发。

一个集群中的leader也可以成为其他集群的follower。

生产者将数据发送到topics上，同时也有责任选择将记录放到哪个partition中。

在搭建zookeeper集群时遇到坑，没有关闭集群防火墙。

kafka集群搭建不赘述，网上博客都有，只需要以搭建好的zookeeper为基础修改配置文件即可。

下面来看一个状态:
```
kafka-topics.sh --create --zookeeper hdp-01:2181,hdp-02:2181,hdp-03:2181 --partitions 3 --replication-factor 3 --topic test
kafka-topics.sh --describe --zookeeper hdp-01:2181,hdp-02:2181,hdp-03:2181 --topic test #查看topic test状态
#状态信息
Topic:test	PartitionCount:3	ReplicationFactor:3	Configs:
	Topic: test	Partition: 0	Leader: 0	Replicas: 0,2,1	Isr: 0,2,1
	Topic: test	Partition: 1	Leader: 1	Replicas: 1,0,2	Isr: 1,0,2
	Topic: test	Partition: 2	Leader: 2	Replicas: 2,1,0	Isr: 2,1,0
```
第一列是表明是哪个topic，第二个是分区标示，第三个是该分区的leader是谁，第四个是该分区的几个副本分别存储在集群上节点的标号，第五个是能够选举成为isr的节点标号。

删除命令，注意需要在server.proerties中设置delete.topic.enable=true，并重启kafka
```
kafka-topics.sh --delete --zookeeper hdp-01:2181,hdp-02:2181,hdp-03:2181 --topic test
# 如果不设置为true会报如下错误，由于是测试，不赘述
Topic test is marked for deletion.
Note: This will have no impact if delete.topic.enable is not set to true.
```

发送消息测试，生产者使用端口9092
```
kafka-console-producer.sh --broker-list hdp-01:9092,hdp-02:9092,hdp-03:9092 --topic test  #前面有设置过监听端口为9092
#输入信息
This is a message
This is another message
```

接收消息测试，消费者使用端口2181，至于为什么占用zookeeper的2181的内部机制，我猜是接收消息直接向zookeeper请求
```
kafka-console-consumer.sh --zookeeper hdp-01:2181,hdp-02:2181,hdp-03:2181 --topic test --from-beginning
```
--from-beginning是从第一个消息开始接收，注意2181是zookeeper用于客户端使用，2888是zookeeper相互间通信，3888是领导者选举。

另外创建流时，createDirectStream方法用的是9092端口，而createStream用的是2181端口。

kafka日志清理有按照时间来定期清理，有按照内容大小来清理。

生产者同步异步，如果设置为异步async，生产者可以以batch形式push数据，这样会提高broker性能，但会增加丢失数据的风险。

acks参数设置为0表示producer不会等待broker的响应，acks设置为1表示producer会在leader partition收到消息时得到broker的一个确认，这样会有更好的可靠性。

除了sync和async还有oneway模式，就是只顾消息发送出去不管死活，消息可靠性低，低延迟、高吞吐，即request.required.acks为0.

在spark-streaming接收kafka数据上，0.8.2.1中有两种方法，老的方法是使用Receiver和Kafka的high-level（高级）API实现。新方法不需要用Receiver（spark1.3引入）。新方法有不同的编程模型，性能特性，语义保证。

在本实验中，我们弃用已经过时的Receiver方法，使用新的稳定API。

传输实验中遇到问题java.lang.ClassNotFoundException: org.apache.spark.streaming.kafka.KafkaRDDPartition，还未解决。

这是kafka生产者代码，从文件中读取数据，并整合数据发送给kafka，经测试，在Kafka 中可以获取并消费.
```
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.io.BufferedReader;
import java.io.FileReader;
import java.util.Properties;

public class ProducerExample {
    public static void main(String[] args) throws Exception{
        String topic = "newtest";   //指定发送topic
        String brokers = "hdp-01:9092,hdp-02:9092,hdp-03:9092";  //集群的broker

        Properties props = new Properties();  //设置属性
        props.put("bootstrap.servers",brokers);  
        props.put("key.serializer","org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer","org.apache.kafka.common.serialization.StringSerializer");
        props.put("producer.type", "async");
        props.put("request.required.acks", 1);

        KafkaProducer<String,String> producer = new KafkaProducer<String, String>(props);

        BufferedReader reader = new BufferedReader(new FileReader("/home/HADOOP/Downloads/2014-02_CitiBike_trip_data.csv"));  //从csv文件中读取数据
        String line;
        int i=0;
        while((line=reader.readLine())!=null) {
            String item[]=line.split(",");
            try {
                //sync local cache is not existed,async is existed and has a batch to remain
                producer.send(new ProducerRecord<String, String>(topic, Integer.toString(i), item[0]+","+item[13]));  //异步发送信息
            } catch (Exception e) {
                e.printStackTrace();
            }
            i++;
        }
        System.out.println(i);
        producer.close();
    }
}
```

发送者pom.xml结构
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.kafka</groupId>
    <artifactId>Producer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <spark.version>2.2.0</spark.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.11</artifactId>
            <version>${spark.version}</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka_2.11</artifactId>
            <version>1.6.2</version>
        </dependency>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>2.12.4</version>
        </dependency>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>0.8.2.1</version>
        </dependency>
        <!--<dependency>-->
            <!--<groupId>org.apache.kafka</groupId>-->
            <!--<artifactId>kafka-streams</artifactId>-->
            <!--<version>1.0.0</version>-->
        <!--</dependency>-->
        <!--<dependency>-->
            <!--<groupId>org.apache.kafka</groupId>-->
            <!--<artifactId>kafka-clients</artifactId>-->
            <!--<version>1.0.0</version>-->
        <!--</dependency>-->
    </dependencies>
</project
```

这是spark-streaming整合kafka的消费者代码
```
import kafka.serializer.StringDecoder
import org.apache.spark.streaming.kafka._
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.SparkConf

object Consumer {
  def main(args: Array[String]){
    val sparkConf = new SparkConf().setMaster("spark://192.168.158.101:7077").setAppName("Consumer")
    val ssc = new StreamingContext(sparkConf,Seconds(2))
    ssc.checkpoint("checkpoint")

    val kafkaParams = Map[String,String]("bootstrap.servers" -> "hdp-01:9092,hdp-02:9092,hdp-03:9092")
    val topics = Set("newtest")
    val directkafkastreaming = KafkaUtils.createDirectStream[String,String,StringDecoder,StringDecoder](ssc,kafkaParams,topics)  //最关键的直接读取数据
    val lines = directkafkastreaming.map(_._2)  //获得value值
    val dataAnalize = lines.XXXXXX  //这里写对获得的数据的处理逻辑，最后将结果赋予dataAnalize
    dataAnalize.print()
    ssc.start()
    ssc.awaitTermination()
  }
}
```

消费者pom.xml结构
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.test</groupId>
    <artifactId>Consumer</artifactId>
    <version>1.0-SNAPSHOT</version>

    <!-- https://mvnrepository.com/artifact/org.apache.spark/spark-streaming-kafka-0-8 -->
    <dependencies>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming-kafka-0-8_2.11</artifactId>
            <version>2.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.scala-lang</groupId>
            <artifactId>scala-library</artifactId>
            <version>2.11.8</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-core_2.11</artifactId>
            <version>2.2.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.spark</groupId>
            <artifactId>spark-streaming_2.11</artifactId>
            <version>2.2.0</version>
        </dependency>
    </dependencies>
</project>
```

根据数据源2014-02_CitiBike_trip_data.csv文件，我们分析2014年2月份数据：
1. 不同年龄段的人乘骑时间分别是多少
2. 不同性别的人乘骑时间分别是多少
3. 哪个自行车站点途径最多
4. 哪些自行车被乘骑最多
5. 哪个站点最少人光顾

通过对生产者发送字段和消费者处理字段的5次修改，我们可以获得以下结果
1. 1990-2000这10年出生的人乘骑时间最长，乘骑时间(Seconds)363690
2. 男性乘骑总时长大于女性乘骑总时长
3. Broad St & Bridge St的途径次数是最多的
4. 取前3位自行车，20135，14905，20039
5. 最少光顾的站点W Broadway & Spring St
