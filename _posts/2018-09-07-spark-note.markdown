---
layout:    post
title:      "Spark笔记"
subtitle:   "Spark NoteBook"
date:       2018-09-07 12:00:00
author:     "scyhssm"
header-img: "img/apache-spark.png"
header-mask: 0.3
catalog:    true
tags:
    - Spark
---

> Spark的笔记

## Spark运行流程
1. 构建Spark Application的运行环境，启动SparkContext
2. SparkContext向资源管理器(YARN、Mesos)申请运行Executor资源，启动Executorbackend
3. Executor向SparkContext申请Task
4. SparkContext将应用程序分发给Executor
5. SparkContext根据RDD构建DAG图，将DAG图分解成Stage、将TaskSet发送给Task Scheduler，最后由Task Scheduler将Task发送给Executor运行
6. Task在Executor运行，运行完释放所有资源

## Spark术语
* Application：Application指的是用户编写的Spark应用程序，包括一个Driver功能的代码和分布在集群中多个节点运行的Executor代码
* Driver：Spark中的Driver运行Application的main函数并创建SparkContext，创建SparkContext目的是准备Spark应用程序运行环境，在Spark中由SparkContext负责与ClusterManager通信，通常用SparkContext代表Driver
* Executor：每个Application都有各自的Executor，在Yarn模式下其进程名为CoarseGrainedExecutor Backend。一个CoarseGrainedExecutor Backend有且仅有一个Executor对象，负责将Task包装成TaskRunner，从线程池抽取空闲线程运行Task

## Standalone架构
主要节点有Client、Master、Worker。Master可以用ZooKeeper实现HA，Master是资源调度节点。Driver既可以运行在Master节点也可以运行在本地Client端。

## Yarn Client
* Spark Yarn Client向Yarn的ResourceManager申请启动Application Master，在SparkContent初始化中创建DAGScheduler和TaskScheduler。
* ResourceManager会在集群中选取NodeManager为应用分配Container，在Container中启动ApplicationMaster，和Cluster区别是该ApplicationMaster不运行SparkContext，只和SparkContext联系进行资源分派。
* SparkContext初始化完毕与ApplicationMaster建立通讯，向ResourceManager注册，根据任务信息向ResourceManager申请资源。
* ApplicationMaster申请到资源会和NodeManager通信，要求其在获得的Container中启动CoarseGrainedExecutorBackend
* CoarseGrainedExecutorBackend向SparkContext注册并申请Task，Client分配Task让其运行，CoarseGrainedExecutorBackend会向Driver汇报运行的状态和进度
* 完成后Client的SparkContext向ResourceManager申请注销并关闭自己

## Yarn Cluster
* 把Spark的Driver作为一个ApplicationMaster在集群中启动
* 由ApplicationMaster创建应用程序申请资源，启动Executor运行Task，监控整个运行过程

## Cluster和Client对比
* 理解YARN-Client和YARN-Cluster深层次的区别之前先清楚一个概念：Application Master。在YARN中，每个Application实例都有一个ApplicationMaster进程，它是Application启动的第一个容器。它负责和ResourceManager打交道并请求资源，获取资源之后告诉NodeManager为其启动Container。从深层次的含义讲YARN-Cluster和YARN-Client模式的区别其实就是ApplicationMaster进程的区别
* YARN-Cluster模式下，Driver运行在AM(Application Master)中，它负责向YARN申请资源，并监督作业的运行状况。当用户提交了作业之后，就可以关掉Client，作业会继续在YARN上运行，因而YARN-Cluster模式不适合运行交互类型的作业
* YARN-Client模式下，Application Master仅仅向YARN请求Executor，Client会和请求的Container通信来调度他们工作，也就是说Client不能离开
