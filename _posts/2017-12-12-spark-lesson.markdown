---
layout:    post
title:      "spark课程学习"
subtitle:   "Learning spark"
date:       2017-12-12 12:00:00
author:     "scyhssm"
header-img: "img/ssm.jpg"
header-mask: 0.3
catalog:    true
tags:
    - spark
    - hadoop
---

>老贝的spark课程学习笔记，想了想如果光学不做点东西怕印象不深刻，遂记录。

## spark数据分析介绍
spark是集群计算平台，提供python，java，scala，sql和很多库的API。

运行在hadoop集群上能够获得所有的hadoop存储数据。

用内存运行相同任务相比hadoop mapreduce要快100倍，而用磁盘运行快10倍。

可以批处理，也可以流处理，对迭代有良好的支持，能够交互命令。

spark本质上是一个分布式计算框架，sql语句和机器学习库能够自动加速。

MLlib，GraphX是spark上的核心库。

spark可以依赖其他的文件系统，但是目前来看还是一个hadoop应用。

spark本身使用scala写的，运行在JVM上。
## 下载开始spark
py shell
```
pyspark
```
scala shell
```
spark-shell
```
scala读取文件
```
lines = sc.textFile("")
```
在shell环境下，SparkContext被自动创建为sc，SparkContext代表了指向集群的连接。

执行计算的节点被称为executor，执行者。

我们可以事先编辑python脚本并用spark-submit提交，在运行脚本前事先包含了spark依赖和spark Python API的运行环境。

用scala导入环境,设置启动配置
```
import org.apache.spark.SparkConf
import org.apache.spark.SparkContext
import org.apache.spark.SparkContext._
val conf = new SparkConf().setMaster("local").setAppName("My App")
val sc = new SparkContext(conf)
```
可以用sc.stop()停止SparkContext，或者仅仅退出应用System.exit()

另外maven中可以加入spark依赖，通过maven方式实现应用

我实现书中的例子都是用scala，要提前学习scala语法。引外强荐一款scala世界的maven，SBT。
## 使用RDD编程
RDD，分布式弹性数据集。工作是创建新的RDD，转换存在的RDD，在RDD上操作计算结果。

RDD会被分成多个部分，每部分分给不同的节点计算。

RDD可以包含java、python和scala对象，还有用户定义的类。

用户可以通过加载外部的数据集或者是在程序内形成RDD。

在调用cache()方法时如果不限定默认的存储级别，就和persist()方法是一样的，缓存策略为MEMORY_ONLY。

创建RDD的两种方法，一种是从外部加载数据集，另一种是转化程序里已有的RDD。

RDD支持transformations和actions。Transformation用于RDD返回新的RDD，像map()和filter()。Actions返回程序的结果，如count()和first()。

观察方法类型可以可以看返回的是RDD还是其他的数据类型。

过滤scala
```
val errorsRDD = inputRDD.filter(line=>line.contains("error"))
```
union 合并两个RDD，不去重
```
val t = sc.textFile("file:///home/HADOOP/Downloads/1.txt")   //如果不加file://是从hdfs文件系统中去寻找的
val hi = t.filter(line => line.contains("hi"))
val hello = t.filter(line => line.contains("hello"))
val zh = hi.union(hello)
zh.collect
```
take()用来返回指定数量的RDD，collect()用来返回整个RDD

Lazy Evaluation，当我们用transformation在RDD上时，操作并不会马上生效，加载数据也是Lazy Evaluation的。

当要使Lazy Evaluation的RDD生效时要用action，例如count。

Lazy Evaluation是spark的设计要点，实际是将表达式存储起来，不求值，在需要求值的时候再求值。

例如递归计算，需要进行大量的递归，如果求一棵树中的最小节点并将该树中所有节点替换成最小节点的值。

一种方法是第一次遍历找出最小值，第二次遍历替换。

另一种思想是可以把递归表达式保存下来，等到返回最小值的时候再执行表达式替换，这就是Lazy Evaluayion,减少不必要的计算。

在hadoop mapreduce中，我们需要考虑将计算放在一起来避免不必要的计算。

Spark在使用时才计算的Lazy Evaluation特性，用复杂的操作和很多简单操作的结合并没有实质上的优化。

Spark的算子依赖于传递函数，每一种语言都有不同的传递函数给Spark的方法。

在对函数进行操作时，要格外注意序列化，如果不能序列化不能传递到其他的节点中去，就会报错。

在函数中要用到算子需要序列化，传输流到每台节点上。这时如果用this.query传输的是整个this，造成资源大大浪费，或者this不支持序列化就会导致失败。

最好用query=this.query先获取，然后再使用query执行要分到多个节点的算子。

flatMap(),从flat上可以看出是扁平化操作。如果是map，多条记录为行，每行进行map，形成2维的数据。flatMap()则全部打平，不分行，是一维的数据。

distinct()去重，intersection()交集，subtract()去掉a中b也包含的相同部分,cartesian()笛卡尔积。

reduce()操作两个元素通过计算返回新元素。

aggregate(),首先第一步写的是单个节点中的操作，第二步写的是将不同节点中内容整合在一起的操作。如求平均值的操作。
```
val result = rdd.par.aggregate((0,0))((acc,number) => (acc._1+number,acc._2+1),(par1,par2)=>(par1._1+par2._1,par1._2+par2._2))
val avg = result._1/result._2.toDouble
```

在spark中还有很多基础算子，不一一列举。

避免一个RDD多次计算，我们可以持久化数据。每个节点将自己计算的数据持久化，如果持久化失败，当需要数据时spark会重新计算丢失的部分。

可以复制数据到多个节点上增加高可靠性。

持久化有多个选择，MEMORY_ONLY,MEMORY_AND_DISK等，要依据不同的需求选择合适的持久化策略。

cache太多不能存在memory中，spark将会用Least Recently Used机制会挤出久的数据，也可以手动unpersist()将RDD从cache中移除。

pipe()传送内容给script。

## Key/Value RDD操作
Spark提供特殊操作针对key/values，这些RDD叫pair RDD。

join方法能够将两个RDD中拥有相同key的元素整合在一起。

可以创建pair通过如下方法，其含义是x为一个textFile输入，通过map的split分割分出2维列表，取第一个元素作为key，整行内容作为value。
```
val pairs = lines.map(x=>(x.split(" ")(0),x))
```

对pair简单的过滤操作
```
pairs.filter{case(key,value)=>value.length<20}
```

注意花括号的使用。如果要调用的参数有两个或两个以上的参数，只能使用小括号。如果单一参数，小括号和花括号可以互换。

从实际上说，一般小括号用于单行，花括号用于多行。参数是case实现的偏函数，只能使用花括号，case一般是多行。

repartition通过节点之间的互联重组数据创建新的partitions。

spark可以用coalesce这个和repartition类似的函数避免数据迁移重组partition，但一般都是减少partition才可以使用。

为了检查partition的数量，scala中可以用rdd.partitions.size()检查大小。

groupByKey不适合运用在大数据集上，尽量不要用，groupByKey会造成不必要的传输。

reduceByKey首先在节点中计算后结果合并，而groupByKey是传送数据后再计算。combineByKey组合数据，foldByKey都优于gourpByKey。

cogroup将多个RDD中同一个Key对应的Value组合到一起。

join有inner，left，right，cross。spark还有sort自定义规则，countByKey,collectAsMap,lookup(key)等方法。

提前分割数据，好的分割数据策略能够减少网络IO的使用。在某些情况下不必优化，如被给予的RDD只被扫描一次，就不必提前分区。

当数据集被多次使用，如join操作，提前partition是很有用的，spark内部提供了HashPartitioner和RangePartitioner两种分区策略，也可以自定义分区策略。

试想用join时，如果不partition，两张表结合无规则，该操作会hash两个数据集，通过网络发送相同key的数据。

如果先将一个表partition，有规则后spark只需要shuffle另一张表，减少了网络传输。

注意要将partition的表持久化，如果不持久化会造成使用被partition的RDD重新估算，就和没有使用partitionBy是一样的。

原因是partitionBy并没有引发rdd计算，只有持久化才会引发存储在内存或磁盘中。如果partitionBy但是没有持久化，在每次action时都会重新partitionBy。

spark中一些操作会利用已知的策略自动partition，在另外一些操作就会使用这些partition信息。

如sortByKey()和groupByKey()会导致range partition和hash partition，map会造成新的RDD遗忘之前的partition信息。

从partition受益的操作，cogroup(),groupWith(),join(),leftOuterJoin(),rightOuterJoin(),groupByKey(),reduceByKey(),combineByKey(),lookup().

如果操作仅在一个RDD上完成，提前partition会让计算在一台机器上完成，最后每台机器传送结果给maser。

cogroup或者join使用提前partition可以让至少一个RDD不必shuffled。

两个RDD都有相同的partitionr，在网络传输时就不用shuffle。

spark计算模型，为了避免像Hadoop将计算结果存在磁盘在迭代计算下多次存取结果，设计两个特性。

数据集分布式存储在节点内存，减少迭代反复IO操作。数据集不可变，只记录转换过程实现无共享数据读写同步问题、出错的可重算性。

lineage,就是RDD间的相互依赖关系，不宜过长，如果太长可以截中间一部分保存下来。

用scala实现PageRank，思想：1.一个网页被其他网页链接到PageRank值会很高。2.一个PageRank很高的网页链接到其他网页，被链接到的网页PageRank会很高。

另外PageRank的细节，多个出度需要评分PageRank值，没有出度则默认对所有网页都有出度，没有外向链接的页面会吞噬掉用户向下浏览的概率。

设置d=用户继续向后浏览概率为0.85，最小值(1-d)／N，N为所有页面。最小值即无出度的情况下，各个页面的排名相等。运行付给每个页面初始值，迭代到收敛。

scala实现
```
val links = sc.objectFile[(String,Seq[String])]("links").partitionBy(new HashPartitioner(100)).persist()
var ranks = links.mapValues(v=>1.0)
for(i <- 0 until 10){
    val contributions = links.join(ranks).flatMap{
        case(pageId,(linkList,rank))=>linkList.map(dest=>(dest,rank/linkList.size))
    }
    ranks = contributions.reduceByKey((x,y)=>x+y).mapValues(v=>0.15+0.85*v)
}
ranks.saveAsTextFile("ranks")
```
## Advanced Spark Programming
在spark中传递一些方法，如map或者是filter，它们可以使用在它们外部定义的变量，但是每个任务获得的是变量的副本，将副本更新不会将记录同步到driver中去。

Spark的可共享变量，如accumulators和broadcast变量，不像map有束缚，它们能够做到相互传递变化达到累加结果或者是广播的作用。

accumulators最常用的用法是计算job执行时events发生的数量。

spark的一种结构，可参照yarn架构：

1.Driver作为应用的入口，app master管理应用

2.Master规划调度资源，cluster manager，管理资源

3.Worker回应节点状态运行excutor

4.Excutor分配job管理job中的task，Worker为应用启动的进程，执行多个task

5.Job由多个stage组成，由action触发

6.Stage，DAGScheduler根据shuffle将job划分多个stage，一个stage包含多个task，stage的分割方法是按照宽依赖和窄依赖，如果tasks都是窄依赖，构成一个stage，因为不用shuffle。如果要shuffle的操作，例如groupBy就会构成宽依赖，好处是可以将需要shuffle的发往不用shuffle的节点，利用shuffle分stage，如果是窄依赖可以一个partition一个流水线，在一个流水线中对一个partition执行整个流水线操作，不需要等待其他partition的运行结果。宽依赖的groupBy由于需要打乱partition重新shuffle，所以不能一个partition分配一个流水线。要等待所有partition处理完毕才能执行下一个task。

7.Task被送到excutor的工作单元，是在一个partition上的单个数据处理流程，线程，如一个partition的一个map操作

在python上的累加器计算空行数，worker节点上的task不能获取accumlator，只能写。
```
file = sc.textFile(inputFile)
blankLines = sc.accumulator(0)
def extractCallSigns(line):
    global blankLines
    if(line == ""):
        blankLines += 1
    return line.split(" ")
callSigns = file.flatMap(extractCallSigns)
callSigns.saveAsTextFile(outputDir + "/callsigns")
print "Blank lines: %d" % blankLines.value
```

累加错误数量
```
validSignCount = sc.accumulator(0)
invalidSignCount = sc.accumulator(0)
def validateSign(sign):
    global validSignCount,invalidSignCount
        if re.match(r"\A\d?[a-zA-Z]{1,2}\d{1,4}[a-zA-Z]{1,3}\Z",sign):
            validSignCount += 1
            return True
        else:
            invalidSignCount += 1
            return False
validSigns = callSigns.filter(validSigns)
contactCount = validSigns.map(lambda sign:(sign,1)).reduceByKey(lambda (x,y):x + y)
contactCount.count()
if invalidSignCount.value<0.1*validSignCount.value:
    contactCount.saveAsTextFile(outputDir + "/contactCount")
else:
    print "Too many errors:%d in %d" % (invalidSignCount.value, validSignCount.value)
```

Spark会通过自动重新执行错误或者过慢的机器加快效率和容错率，如果节点运行map()中的一个partition过慢或者错误，spark将会在另一个节点rerun该task。

注意累加器陷阱，如果累加器action多次程序不当可能会累加多次，另外在spark中我们希望有一些满足交换律和结合律的符号，如*和+。

broadcast变量，这个变量和accumulator相反。通过传送数据的reference到worker中，worker只读，修改仅能够在driver。

为了保证数据一致性，传送到node而不是task，task是线程，可以用进程中的内容。

python country lookup,注意如果传送一张巨大的table表，创建副本就会让内存消耗过多
```
signPrefixes = loadCallSignTable()
def processSignCount(sign_count,signPrefixes):
    country = lookupCountry(sign_count[0],signPrefixes)
    count = sign_count[1]
    return (country,count)
countryContactCounts = contactCounts.map(processSignCount).reduceByKey(lambda x,y:x+y)
```

通过broadcast优化
```
signPrefixes = sc.broadcast(loadCallSignTable())
def processSignCount(sign_count,signPrefixes):
    country = lookupCountry(sign_count[0],signPrefixes.value)
    count = sign_count[1]
    return (country,count)
countryContactCounts = (contactCounts.map(processSignCount).reduceByKey((lambda x,y:x+y)))
countryContactCounts.saveAsTextFile(outputDir+"/countries.txt")
```

当要broadcast大量内容时，选择一种合理的序列化方式可以起到加快效率的作用，选择不同的序列化library。

per-partition basis可以避免重复建立工作，比如数据库连接。

python共享连接池，map每一个partition执行processCallSigns方法，urls将signs中的内容map出来加上http://73s.com/qsos/%s.json。

为urls制造request请求，制造result结果，相当于在一个partition中创建了一个公共连接池，所有的请求都只用一个connection，而不用每次都创建connection。
```
def processCallSigns(signs):
    http = urllib3.PoolManager()
    urls = map(lambda x:"http://73s.com/qsos/%s.json" % x,signs)
    requests = map(lambda x:(x,http.request('GET',x)),urls)
    result = map(lambda x:(x[0],json.loads(x[1].data)),request)
    return filter(lambda x:x[1] is not None,result)
def fetchCallSigns(input):
    return input.mapPartitions(lambda callSigns:processCallSigns(callSigns))
contactsContactList = fetchCallSigns(validSigns)
```
## Running on a cluster
好处是单机版代码可以直接放到集群上跑没有影响。

spark也可以工作在yarn和mesos架构上。

该节和前面内容重复，用脚本spark-submit提交任务，spark方式提交。
```
./bin/spark-submit \
—master spark://hostname:7077 \
—deploy-mode cluster \
—class com.databricks.examples.SparkExample \
—name “Example Program” \
—jars dep1.jar,dep2.jar,dep3.jar \ —total-executor-cores 300 \
—executor-memory 10g \
myApp.jar
```

用yarn方式提交
```
export HADOP_CONF_DIR=/opt/hadoop/conf $ ./bin/spark-submit \
./bin/spark-submit \
—master yarn \
—py-files somelib-1.2.egg,otherlib-4.4.zip,other-file.py \ —deploy-mode client \
—name “Example Program” \
—queue exampleQueue \  #通过队列划分优先级，用队列方式定义不同的权限，有些queue中的cpu要多，有些内存要多等，queue中有被提交的不同job
—num-executors 40 \
—executor-memory 10g \
my_script.py
```

cluster manager唤醒executor，executor将自己注册到driver。

Standalone是简单模式或称独立模式，可以单独部署到一个集群中，无依赖任何其他资源管理系统。不使用其他调度工具时会存在单点故障，使用Zookeeper等可以解决.

spark使用可插拔的cluster manager，这让它能够使用yarn，mesos，standalone等多种cluster manager。

spark可以在yarn工作节点上同时运行driver和executor，类似于yarn架构上的master和worker。

spark-submit开始运行driver，找到main，driver找到cluster manager请求executor需要的资源，cluster manager运行executor。

driver开始运行用户的application，根据RDD action和transformation，driver发送由tasks组成的工作给executor，计算结果。

如果调用SparkContext.stop()则executor会马上终止释放资源。

spark-submit可选多种参数，具体可查。

实际情况很多族群由很多个用户共享。

如果一个master不能保证高可用性，可以使用zookeeper管理多个master。
## spark steaming
DStream流相当于一个有序的sequence RDD，用各种各样的输入流构建Flume，Kafka，HDFS。

DStream有transformation产生一个新的DStream，还有输出操作，流数据理论上要7*24小时服务。

流数据一直在流动，不知道起始位置，设置checkpoint是很有必要的，可以恢复数据。

非守护线程还在执行，进程就不会终止。没有非守护线程，该进程中的守护线程就会被杀死，进程结束。

流式过滤，每秒的数据构造成一个小的离线数据RDD，RDD操作都可用。
```
//Create a StreamingContext with a 1-second batch size from a SparkConf
val ssc = new StreamingContext(conf, Seconds(1))  
//Create a DStream using data received after connecting to port 7777 on the local machine
val lines = ssc.socketTextStream(“localhost”, 7777)
//Filter our DStream for lines with “error”
val errorLines = lines.filter(_.contains(“error”))
//Print out the lines with errors
errorLines.print()
```
启动流式过滤器
```
//Start our streaming context and wait for it to “finish”
ssc.start()
//Wait for the job to finish
ssc.awaitTermination()
```
设置batch internal，决定收集一段数据的间隔，形成一个RDD的所需时间，收到一个RDD后执行filter。

上面的代码由print这个action触发transformation过程。

Receiver有一个long task，该task不能挂掉，用于接收数据，接收的数据拷贝到另一个executor。

检查点一般是5-10个batch，默认情况下，接收到的数据分发到2个节点中去，如果数据还在，spark steaming允许重新计算RDD。

stateless，是一个DStream中的记录无状态，和前一个无关。如果是有状态，之前的流都会被计算，需要记录之前流的记录。

在DStream中使用map()和reduceByKey()
```
val accessLogDStream = logData.map(line =>
ApacheAccessLog.parseFromLogLine(line))
val ipDStream = accessLogsDStream.map(entry => (entry.getIpAddress(), 1)) val ipCountsDStream = ipDStream.reduceByKey((x, y) => x + y)
```

操作join两个DStream，同一个时间段两个DStream做操作。因为时间无边无际，一直在产生同名的DStream，这就无法确定执行join的是哪对流，需要同一个时间段的内容执行操作
```
val ipBytesDStream =
accessLogsDStream.map(entry => (entry.getIpAddress(), entry.getContentSize())) val ipBytesSumDStream = ipBytesDStream.reduceByKey((x, y) => x + y)
val ipBytesRequestCountDStream = ipCountsDStream.join(ipBytesSumDStream)
```

transformation就是transform封装
```
val outlierDStream = accessLogsDStream.transform { rdd => extractOutliers(rdd) }
```

两种方法算stateful的内容，一种是windows，只能计算一个滑动窗口中的值，另一个是updateStateByKey()，stateful必须设置检查点使用。

window使用，每20秒统计前30秒的数据
```
val accessLogsWindow = accessLogsDStream.window(Seconds(30), Seconds(20))
val windowCounts = accessLogsWindow.count()
```

window简便方法，可以用上一个获得的结果加上新加入的再减去过旧的。
```
val ipDStream = accessLogsDStream.map(logEntry => (logEntry.getIpAddress(), 1))
val ipCountDStream = ipDStream.reduceByKeyAndWindow(
// Adding elements in the new batches entering the window
    {(x, y) => x + y},
// Removing elements from the oldest batches exiting the window
    {(x, y) => x - y},
    Seconds(30), // Window duration
    Seconds(10)) // Slide duration
```

countByvalue，统计同一个value出现的次数。

updateStateByKey用法，values是key的多个events（每个rdd都可能有key的状态），state是以前的状态信息，最后返回的是一个新的包含多个(key,state)对的RDD
```
def updateRunningSum(values: Seq[Long], state: Option[Long]) = {
    Some(state.getOrElse(0L) + values.size)
}
val responseCodeDStream = accessLogsDStream.map(log =>
    (log.getResponseCode(), 1L))
val responseCodeCountDStream =
    responseCodeDStream.updateStateByKey(updateRunningSum _)
```

output是lazy evaluation。

注意foreachRDD用法，不要产生多余的创建链接或者关闭链接代码，否则就会产生性能损耗。

input source有HDFS，kafka等。

spark创建文件，在临时目录创建写文件，再将它放到spark监控目录下，叫原子地创建文件。

spark支持和消息中间件打交道，现有的消息中间件Twitter, Apache Kafka, Amazon Kinesis, Apache Flume, and ZeroMQ.

消息中间件，解耦客户端和服务端，如果服务端挂机，客户端可以和消息中间件交流。消息中间件可以作为客户端和服务端的缓冲区使用。

Maven artifact spark-streaming-XXX_2.10的方法加入spark消息中间件支持。

spark-streaming中driver宕机处理方法依赖checkpoint和集群，worker宕机处理方法和不是streaming时一样。

receiver宕机，从hdfs来就可以再读，如果是unreliable的数据就会丢失数据，无法解决，最好的办法是从可靠的输入数据源获得数据。

batch size理论上500ms是最好的最小大小，调整window大小调优spark性能。

receiver有时候会成为瓶颈，创建多个reveiver，每个reveiver对应一个DStream，union多个DStream成为一个DStream。
