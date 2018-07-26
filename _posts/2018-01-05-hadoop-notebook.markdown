---
layout:    post
title:      "hadoop课程学习"
subtitle:   "Learning Hadoop"
date:       2018-01-05 12:00:00
author:     "scyhssm"
header-img: "img/hadoop.png"
header-mask: 0.3
catalog:    true
tags:
    - hadoop
---

> 这是对老贝hadoop课件的整理学习

## 初识hadoop
hadoop解决的问题1.海量数据的存储HDFS 2.海量数据的分析MapReduce 3.资源调度YARN。

HDFS分布式海量数据存储，YARN mapreduce解决海量数据的分布式运算，Hbase分布式nosql数据库，Hive数据仓库工具，zookeeper分布式应用协调服务。

Pig类似于hive语法和sql差别大，Mahout机器学习工具，Flume自动化日志采集工具，Sqoop Hadoop和关系数据库之间的数据交换工具，Oozie串联所有分析流程的工作流工具。

HDFS，在一个操作系统存不下需要多台机器上存储的系统。通透性，通过网络访问文件在程序和用户看来就像访问本地磁盘。容错，即使有节点脱机省体还可以持续运作而不会有数据损失。

HDFS是分布式文件管理系统的一种，适用于一次写入多次查询，不支持并发写，小文件不适合存储，对于频繁写入的小文件，更适合用Hbase。

MapReduce主要用于搜索领域，MR由两个阶段组成，Map和Reduce可以实现分布式运算，形参是key、value对，表示函数的输入信息。

zookeeper是Hadoop的分布式协调服务，它包含一个简单的原语集，分布式应用程序可以基于它实现同步服务，配置维护和命名服务。

Zookeeper提供通用的分布式锁，协调分布式应用，Hadoop2.0确保整个集群只有一个活跃的Namenode，存储配置信息。

HBase使用Zookeeper确保整个集群只有一个Hmaster，察觉HRegionServer联机和宕机，存储访问控制列表。

HBase是一个高可靠，高性能，面向列，可伸缩的分布式存储系统，利用HBase技术可在廉价的PC Server上搭建大规模结构化存储集群。

HBase利用HDFS作为文件存储系统，利用MapReduce处理HBase中海量数据，利用zookeeper作为协调工具。

Hive是基于Hadoop的数据仓库工具，可以将结构化的数据文件映射为一张数据库表，提供简单的sql查询功能，转换sql语句为MapReducer任务。

Flume是分布式的日志收集系统，它将服务器中数据收集起来送到指定地方。

Sqoop主要用来在Hadoop和关系数据库中传递数据，Sqoop架构很简单，整合了Hive、Hbase和Oozie，通过mapreduce任务传递数据，提供并发特性和容错。

YARN是全新的资源管理器，将资源管理器和处理组件分开。YARN架构不受MapReduce约束，在YARN上可以使用其他计算框架如Spark，减少耦合。

Hadoop有单机模式，伪分布式（一台机器模拟分布式集群），集群模式。另外最好选择奇数台机器，根据任务在每台机器上开启进程，SSH免密登陆保证机器通讯。

Zookeeper选择奇数台机器的原因有，投票需要超过半数，2N+1最高容忍N台机器投反对，同理2N也是最高N台机器投反对，容忍度相同。节约资源的情况下最好用奇数台投票。

另外一个原因Zookeeper要求集群中过半机器可用才可以对外可用，奇数台机器可以令系统更稳定。

## MR介绍
mapreduce数据源，读入为键值对的格式，key为偏移量，例如（106，003240230402304023040...939489N9+00032+9999...）。

mapreduce数据流input->map->shuffle->reduce->output.

map获得数据内容来自本地数据、本地机架数据和跨机架数据几种。

一个map任务获得数据，将部分数据放到缓存中，再将缓存数据partition，将partition过的内容存到磁盘，map中不断进行缓存->磁盘的操作，最后形成一个shuffle过的内容。

reduce获得多个map任务中经过初步shuffle的部分指定内容，即多个map的同一部分内容传给一个reduce由它再对数据操作。注意一般是几个reduce几个part-0000X结果。

## HDFS
主从结构，两台namenode，只有一个投入管理，另一台提供高可用性，多台datanode。

namenode负责，接收用户操作请求，维护文件系统的目录结构，管理文件与block之间的关系，block与datanode的关系。

datanode负责，存储文件，文件被分成block存储在磁盘上，保证数据安全，文件要有多个副本。

Namenode，保管元数据，响应client发来的读写请求，将响应的元数据信息发给client。

Datanode，实际保存数据，一个datanode包括数个block，每个block保存一份数据，2.0中默认block大小为64MB。

Rack，每个rack包含数个datanode，每份数据要在datanodes保存多次，在选择冗余的datanode节点时，尽量选择不在一个rack上的datanode。

Replication，HDFS默认将每份数据保存为3份，将每个replication放在不同的datanode上。

Client，客户端向namenode发起读写请求，接收到元数据后和datanode交互实际读写数据。

namenode中的元数据作用，可以在datanode宕机时及时进行容错处理，元数据中保存了文件名，文件被切分成几个block，每个block保存在哪些机器上，同时也告诉了client文件保存位置。

Namenode是整个文件系统的管理节点，维护整个文件系统的文件目录树，文件／目录的元信息和每个文件对应的数据块列表。

Namenode文件包括：fsimage，元数据镜像文件存储某一时段Namenode内存元数据信息，edits，操作日志文件，fstime，保存最近一次checkpoint时间以上这些文件保存在Linux文件系统中。

Namenode始终在内存中保存metadata，用于处理“读请求”，到有“写请求”到来时，namenode会首先editlog到磁盘，即向edits文件中写日志，成功返回才修改内存，并向客户端返回。

Hadoop会维护一个fsimage文件，也就是namenode中metadata镜像，但是fsimage不会随时与namenode内存中的metadata保持一致，而是每隔一段时间通过合并edits来更新内容。

fsimage包含文件系统中所有目录和文件inode序列化形式。

Secondary namenode用来合并fsimage和edits文件来更新Namenode的fsimage的，因此如果namenode宕机，secondary还保存了一份最近的fsimage用来恢复。

HDFS的shell操作有，-moveFromLocal，从本地文件移动到hdfs。-get[-ignoreCrc]<src><localdst>，复制文件到本地，可以忽略crc校验。

-getmerge <src><localdst>，将源目录中所有文件排序合并到一个文件中。-cat <src>，在终端显示文件内容。-text <src>在终端显示文件内容。

-copyToLocal [-ignoreCrc] <src> <localdst>,复制到本地。-moveToLocal <src> <localdst>。-mkdir <path>,创建文件夹。-touchz <path>,创建一个空文件。

-help [cmd],显示命令的帮助信息。-ls <path>,显示当前目录下所有文件。-du <path>,显示目录中所有文件大小。-count[-q] <path>,显示目录中文件数量。

-mv <src> <dst>，移动多个文件到目标目录。-cp <src> <dst>,复制多个文件到目标目录。-rm，删除文件or文件夹.-put <localsrc><dst>,本地文件复制到hdfs。-copyFromLocal，同put。

可以编写代码写或者读文件内容，文件系统获得要用FileSystem fs = FileSystem.get(URI.create(dst),conf).

RPC（remote procedure call）,远程过程调用协议。一种通过网络从远程计算机程序上请求服务，不需要了解底层网络技术的协议。RPC协议假定某些传输协议的存在，如TCP和UDP。

在OSI网络通信模型中，RPC跨越了传输层和应用层，RPC使得开发网络分布式多程序在哪的应用程序更加容易。Hadoop整个体系结构构建在RPC之上。

RPC采用客户机/服务器模式，请求程序是一台客户机，服务提供程序是服务器。客户机调用进程发送一个有进程参数的调用信息到服务进程，等待应答信息。服务端，进程保持睡眠状态直到调用信息到达。

调用信息到达，服务器获得参数，计算结果发送答复信息，等待下一个调用信息，最后，客户端调用进程接收答复信息，获得进程结果，然后调用执行继续进行。

Hadoop中的block计算距离公式，如果在同一个节点中两个block距离为0，如果在同组机架的不同节点距离为2，如果在不同机架相同数据中心上距离为4，如果在不同数据中心距离为6.

HA（high availability），Hadoop用nameservice概念代替namenode概念，一个nameservice包含两个namenode，一个为active状态一个为standby。

nameservice是对外访问对逻辑名，这样用户程序访问时，不需要知道两个namenode具体运作细节。failover控制器监控namenode状态，通过心跳机制向zk集群汇报情况。

如果active宕机，failover会立即向zookeeper集群汇报，集群会通知当前处于standby状态对namenode变为active状态。

另外namenode通过fsimage和edits文件协调存储元信息，集群模式下，edits单独存储在一个journalNode分布式日志系统。

hadoop2.0引入了journalnode，即采用多个journalnodes存储edits，namenode在写edits时也会并行向每个journalnode写相同的edits，这是通过QJM（QuerumJournalManager）管理的。

用来存储edits文件的journalnode节点由zookeeper负责协调，处于active的节点的namenode向journalnode写文件时，会同时向所有节点写数据，由于journalnode多，只要多数写入成功，就成功。

因此fsimage较大只有standby保存，而edits较小可以用多台journalnode保存，这样在处于active的namenode宕机时，保证edits和fsimage都存在。

### HDFS联邦

该特性允许一个HDFS集群中存在多个namenode同时提供对外服务，这些namenode分管一部分目录，彼此相互隔离。即HDFS联邦中的namenodes共享块池功能。

由于数据量增加，业务量增大，一个namenodes显然会遇到瓶颈，因此需要有多个namenodes集合在一起，扩展性能。

Namespace逻辑上包括两层，Namespace由目录，文件和块组成，支持所有文件系统操作包括增加、删除、修改和列出文件和目录（从文件上）。

Block Storage Service有两个部分：1.block管理，提供datanode集群的注册和定期的心跳检查，处理block的报告并掌握block位置，支持block相关操作，增删改查和得到block位置，管理副本位置及副本的复制和删除，存储由提供datanodes的本地系统提供，允许读写。

2.为了水平扩展，使用多个独立的namenodes／namespaces，这些namenode之间相连，即namenodes是独立的，不需要相互协调。datanode被所有的namenode使用用来作为通用的数据块存储设备，每一个datanode中注册集群中所有的namenode。datanodes发送心跳和block报告并且处理namenode发送的命令。

## Hadoop IO
Hadoop数据完整性，写入数据校验和由datanode负责，读取数据校验和由client负责，后台校验和由datanode后台自动运行，数据副本机制由Hadoop平台提供。

Hadoop有多种压缩格式，如deflate，gzip，bzip2，LZO，snappy。可以在mapreduce中显式设置压缩格式，FileOutputFormat.setOutputCompressorClass（job，Gzip.class）。

序列化是把结构化对象转化为字节流，反序列化是序列化的逆过程，把字节流转回结构化对象。Hadoop中不使用java的序列化。

Hadoop序列化格式特点：紧凑，高效使用存储空间；快速，读写数据的额外开销小；可扩展，可透明地读取老格式的数据；互操作，支持多语言的交互。

序列化在分布式系统中的作用：进程间通信，永久存储；Hadoop节点间通信。

Writable接口，是根据DataInput和DataOutput实现的简单、有效的序列化对象。MR的任意key和value必须实现Writable接口。writable接口中有write和readFiles接口。

Hadoop中序列化的java对象成为writable的有：boolean->BooleanWritable,byte->ByteWritable,short->ShortWritable,int->IntWritable(VIntWritable),float->FloatWritable,long->LongWritable(VLongWritable),double->DoubleWritable.

除了java常用类型外，还可以自定义Writable实现我们想要的对象的序列化，以便于更好存储。

SequenceFile使用键值对持久化保存数据，非常适合日志文件，可以作为小文件的容器。存储在SequenceFile中的键和值不一定需要是Writable类型，只要能被Serialization序列化和反序列化，任何类型都可以。

SequenceFile格式的普通压缩，一个Record中有Record length，Key length，Key，Value。存储时压缩value成为Compressed value。

如果是SequenceFile格式的块压缩，是一次性压缩多条记录，利用记录间的相似性压缩，效率更高。一个压缩block中存储Number of records,Compressed key lengths，Compressed keys，Compressed value length，Compressed values。可以这么想，一段相同记录重复多次都可以用一个小标记表示，这样重复次数越多压缩效率越高，显然block一次存储多个record就是利用这个思想。

MapFile是经过排序的SequenceFile，MapFile有索引，MapFile可以视为是Java.util.Map的持久化形式。

## MapReduce开发
集群上运行mapreduce一般流程：1.作业打包 2.启动作业，观察控制台输出 3.Web界面 4.查看日志文件 5.MapReduce工作流.

mapreduce web界面可以看集群最多容纳的map和reduce数量以及现在的map和reduce数量，一个job上map和reduce的完成度，完成的jobs和失败的jobs列举，job在某台机器上的map和reduce情况等。

MapReduce作业调优有调优mapper数量，reducer数量，是否用combiners，intermediate compression，custom serialization，shuffle tweaks等。

一般如果不调整，使用默认的，hadoop自有其map和reduce的运行规则，如果调整可以手动调整map和reduce个数，还能调整另外的参数。

## MapReduce工作机制
[k1,v1]->[k2,v2]->[k2,list(v2)]->[k3,v3].

第一代MapReduce存在的问题：1.JobTracker是集群事务的集中处理点，存在单点故障；2.JobTracker需要完成的任务太多，既要维护job状态又要维护job的task状态，造成过多资源消耗；3.在taskTracker端，用map／reduce task作为资源的表示过于简单，没有考虑到cpu、内存等资源情况；4.把资源强制划分为map／reduce slot，当只有map task时，reduce slot不能用，反过来也一样。

Yarn/MRv2最基本的想法是将原JobTracker主要的资源管理和job调度／监视功能分开作为两个单独的守护进程。分为一个全局的ResourceManager和ApplicationMaster。

Application相当于map-reduce job或者DAG jobs，ResourceManager和NodeManager组成基本的数据计算框架。

MRv2运行流程：1.MR JobClient向resourceManager（AsM）提交一个job；2.AsM向Scheduler请求一个供MR AM运行的container，然后启动；3.MR AM启动后向AsM注册；4.MR JobClient向AsM获取MR AM相关的信息，直接和MR AM通信；5.MR AM计算splits并为所有的map构造资源请求；6.MR AM做一些必要的MR OutputCommitter的准备工作；7.MR AM向RM发起资源请求，得到一组供map／reduce task运行的container，然后和NM一起对每个container执行一些必要的任务，包括资源本地化；8.MR AM监视运行的task直到完成，当task失败，申请新的container运行失败的task；9.每个map／reduce task完成后，MR AM运行MR outputCommitter的cleanup代码，进行收尾工作；10.当所有的map／reduce完成后，MR AM运行outputCommitter的必要的job commit或者abort APIs；11.MR AM退出。（注意ApplicationManager负责job生命周期内协调的所有工作，类似1.0的jobtracker）

YARN大大减少Job Tracker资源消耗，并让检测每个Job子任务状态的程序分布式化（NodeManager）了。YARN中Application Master是一个可变更部分，用户可以对不同编程模型编写AM，让更多类型的编程模型跑在Hadoop集群中（例如spark on yarn）。Jobtracker监控job下任务运行负担大，现在AM去做，RM检测AM的运行状况，如果出问题，将在其他机器上重启。

shuffle是MR处理流程中的一个过程，它的每个步骤是分散在各个map task和reduce task节点上完成的，分为3个操作：1.分区partition。2.sort根据key排序。3.combiner进行局部value的合并。

map任务：1.每个map都有一个环形内存缓冲区，存储任务输出。默认大小100MB（io.sort.mb），一旦达到阈值0.8（io.sort.spill.percent），一个后台线程把内容写到磁盘指定目录（mapred.local.dir）下的新建的一个溢出文件。2.写磁盘前，要partition，sort。如果有combiner，combine排序后数据。3.等最后记录写完，合并全部溢出写文件作为一个分区且排序的文件。

reduce任务：1.reducer通过http方式得到输出文件的分区。2.TaskTracker为分区文件运行reduce任务。复制阶段把map输出复制到reducer的内存或磁盘。一个map任务完成，reduce就开始复制输出。3.排序阶段合并map输出，然后走reduce阶段。

map端调优，可以调整io.sort.mb,io.sort.record.percent，io.sort.spill.percent，io.sort.factor，min.num.sills.for.combine，mapred.compress.map.output，mapred.map.output.compression.codec，tasktracker.http.threads.

reduce端调优，mapred.reduce.paralled.copies，mapred.reduce.copy.backoff，io.sort.factor，mapred.job.shuffle.input.buffer.percent，mapred.job.shuffle.merge.percent，mapred.inmem.merge.threshold，mapred.job.reduce.input.buffer.percent，

在某些时候，需要让某个mapper的计算结果进入指定的reducer逻辑，在mapreduce中，使用partitioner完成。partitioner是partitioner机制的基类，如果需要定制partitioner机制也需要继承该类。HashPartitioner是mapreduce的默认partitioner。

每一个map可能会产生大量的输出，combiner的作用是在map端对输出先做一次合并，以减少传输到reducer的数据量。

combiner最基本是实现本地key的归并，combiner类似于本地reduce。如果不用combiner，所有任务由reduce完成，效率会相对低下，先完成的map会在本地聚合，提升速度。

实现由自定义的combiner继承reducer，重写reduce方法，再在job里设置：job.setCombinerClass(XXXXX.class),combiner使用原则是不影响业务逻辑。

## MR特性
计数器是用于用户想要充分了解数据，收集作业统计上。用来记录某一特定发生的时间。获取计数器比获取日志更方便，分析计数器比分析日志更方便。

内置的计数器有：MapReduce task counters，Filesystem counters，FileInputFormat counters，FileOutputFormat counters，Job counters等计数器。

任务计数器：Filesystem bytes read（BYTES_READ),Bytes read(BYTES_READ)等。

排序的输出结果是部分排序，key的排序是根据WritableComparable中的compareTo方法。全局排序使用partitioner，将结果分为从大到小的不同区，不同区内的数据进行排序。

## HBase
NoSQL菲关系型数据库，随着web2.0网站兴起，传统关系型数据库暴露出很多问题。

CAP原理，一致性，可用性，分区容忍性。CAP原理最多只能实现两点，不能三者兼顾。

传统关系型数据库具有ACID属性，对一致性要求高，因此降低了A和P。为了提高系统性能和可扩展性，必须牺牲C。

考虑CA，传统上的关系型数据库。CP，主要是key-value数据库，典型代表是谷歌的Big Table按列存储。AP，面向文档的适用用分布式系统的数据库，如Amazon的Dynamo。

NoSQL优势，易扩展，大数据量，高性能，灵活的数据模型，高可用。

HBase是Big Table的开源版本，是分布式的，列存储的、高可靠性、高性能的存储系统。

不同于一般的关系型数据库，它适合非结构化数据存储，基于列而不是基于行。

HBase可以直接使用本地文件系统或者HDFS作为数据存储方式。

特点，表非常大，可以有数十亿行，上百万列。无模式，每行都有一个可排序的主键和任意多的列，列可以动态增加，不同行可以有不同的列。

面向列的存储和权限控制，列独立检索。空列不占用存储空间，表可以设计得非常稀疏。

每个单元中的数据可以有多个版本，默认情况下版本号自动分配，是单元格插入时的时间戳。

面向列的存储系统可以单独查询某列，降低IO。同列数据有很高的相似性，会增加压缩效率。HBase很多特性都是由列存储决定的。

适合HBase的有半结构化或者非结构化数据，对于数据结构字段不确定很难按照概念进行抽取的数据。要添加字段，高并发读写，需要历史数据的，读数据>写数据。

HBase的client使用HBase的RPC机制和HMaster和HRegionServer进行通信，管理类操作是由Client和HMaster进行RPC，数据读写类操作由client和HRegionServer进行RPC。

ZooKeeper Quorum中存储类-ROOT-i表的地址和HMaster的地址，HRegionServer将自己以短暂方式注册到ZooKeeper，使得HMaster可以随时感知到HRegionServer，而且可以避免HMaster单点问题。

HMaster没有单点问题，HBase可以启动多个HMaster，通过zookeeper的Master Election机制保证总有master运行。Hmaster主要负责Table和Region的管理工作。

管理用户的CUID，管理HRegionServer的负载均衡，调整Region分布。在Region Split后，负责新的Region分配。在HRegionServer停机后，负责失效HRegionServer上的Regions迁移。

Row Key行键，Table中的记录按照Row key排序。可以是任意字符串，最大长度64KB,row key保存为字节数组。

Timestamp时间戳，HBase通过row和column确定的为一个存储单元称为cell。每次数据操作对应时间戳，可以看作是数据的版本号，保存数据的最新n个版本或者某段时间的版本。

Column Family列簇，Table水平方向由一个或者多个列簇组成，必须在使用表前定义。一个列簇可以由任意多个column组成，列簇支持动态扩展。所有column二进制格式存储。

Cell，由row key，column，version唯一确定的单元。cell中数据没有类型，全部是字节码形式存储。Table增加会自动分裂多分splits，称为regions。一个region由[startkey,endkey）表示。

HBase中有两张特殊的表，-ROOT-和.META，ZooKeeper中记录了-ROOT-表的location。

-ROOT-记录了.META.表的Region信息，-ROOT-只有一个region。.META.，记录了用户表的Region信息，.META.可以有多个region。

Client访问用户数据前要首先访问zookeeper，再访问-ROOT-表，接着访问.META.表，最后找到用户数据的位置访问，需要多次IO操作。Client端会做cache缓存。

RegionServer是Region读写操纵的场所，Master管理Region的分配，基于zookeeper来保证高可用性HA。

HRegionServer由多个HRegion组成，每个HRegion对应Table中的一个Region，HRegion由多个HStore组成，每个HStore对应了Table中的一个column family。

每个column family其实就是一个集中的存储单元，最好将具备共同IO特性的column放在一个column family中，这样最高效。

HStore存储是HBase存储的核心，包含了Memstore，缓冲池，用户写入数据首先缓存到MenStore，当MemStore满了会flush到StoreFile。

当StoreFile文件数量增到阈值，会出发Compact操作，将多个StoreFiles合并成一个，合并过程中会进行版本合并和数据删除。

HBase只有增加数据，所有的更新和删除操作都是在compact操作中进行的，这样用户的写操作只要写入内存就可以马上返回，保证了HBase I/O的高性能。

当StoreFiles不断合并变得越来越大，大小超过一定阈值会出发split操作，把region split成两个region。

这时父region下线，分出的两个region会分配到相应的regionserver上，使得原先的1个region压力分到两个region上。

HLog，HRegionServer中都有一个HLog对象，HLog是一个实现Write Ahead Log的类，每次用户操作写入MemmStore同时也会写一份数据到HLog文件。

HLog文件定期滚动出新的，删除旧的文件，旧的文件会被持久化到StoreFile。

当HRegionServer意外终止，HMaster会通过ZooKeeper感知到。HMaster处理遗留的HLog文件，将其中不同region的Log进行拆分，放到相应的region目录下，将失效的region重新分配。

领取到这些region的HRegionServer在Load Region的过程中，会发现有历史HLog需要处理，重放HLog数据到MemStore中，然后flush到StoreFiles，完成数据恢复。

可以看下HFile的格式，HFile是HBase中KeyValue数据的存储格式，HFile是Hadoop的二进制文件，实际上StoreFile就是对HFile做了轻量级包装。

Master容错，zookeeper重新选择新的Master，无master过程中，读取还可以进行，但是无法进行负载均衡，region切分。

zookeeper容错，zookeeper是一个可靠的服务，一般配置3-5个zookeeper实例。

RegionServer定时发送心跳给ZooKeeper，一段时间内没有反应则将该RegionServer中的Region分配到其他的RegionServer上。日志由主服务器分割派送给新的RegionServer。

Data Block 是Hbase I/O的基本单元，每个Data块除了开头的Magic外就是一个个的key／value拼接而成。Magic内容是一些随机数字，目的是防止数据损坏。

HBase提供了配套的TableInputFormat和TableOutputFormat API，可以方便地将HBase Table作为Hadoop MapReduce的Source和Sink。

HBase特性有强一致性，同一行数据读写只在同一台regionserver上进行。水平伸缩性，region自动分裂。行事务性。三维有序，rowkey（ASC），columnlabel（ASC）和version（DESC）最后到值。

支持范围查询，高性能随机读写，和Hadoop无缝集成，支持多种压缩算法。

使用HBase不要创建过多到Column Family，合理利用三维有序存储。
