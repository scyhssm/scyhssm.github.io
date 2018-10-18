---
layout:    post
title:      "MapReduce笔记"
subtitle:   "MapReduce NoteBook"
date:       2018-09-04 12:00:00
author:     "scyhssm"
header-img: "img/mapreduce.png"
header-mask: 0.3
catalog:    true
tags:
    - MapReduce
    - Hadoop
---

> MapReduce的笔记

## MapReduce中的Partition、Sort、Combine
1. MapReduce中的map任务会输出key/value对，保存在缓冲区，缓冲区到一定的溢写比时取出
2. 将从缓冲区中获得的key/value对Partition分区
3. 对各个Partition的key/value进行排序
4. 若可以聚合，则建议写到磁盘前Combine减小大小
5. 每次溢写都会产生溢写文件，如果小文件过多，可以对溢写文件再进行一次combine

## MapReduce1.0之JobTracker
JobTracker负责资源监控和作业调度，JobTracker监控所有TaskTracker和Job的健康状况，一旦失败就转移到其他节点，为作业分配资源。JobTracker中包含任务调度器，其负责选择合适的任务使用资源，任务调度器可插拔。

## MapReduce1.0弊端
1. JobTracker存在单点故障问题，是MapReduce任务的入口点，如果出现问题整个服务会瘫痪。
2. JobTracker负担太重，如果Job数过多，会消耗大量内存。
3. 把资源简单划分为map task slot和reduce task slot，资源利用率低

## MapReduce2.0
笔记更详细
1. ResourceManager
* 负责接收客户端提交的Job，分配和调度资源
* 启动ApplicationMaster，分配ApplcationMaster资源
* 监控ApplicationMaster在其失败时重启
* 监控NodeManager
2. ApplcationMaster
* 为MapReduce类型的程序申请资源，分配任务
* 负责相关数据的切分
* 监控任务的执行及容错
3. NodeManager
* 管理单个节点资源，向ResourceManager汇报
* 接收并处理来自ResourceManager的分配Container命令
* 接收并处理来自ApplicationMaster执行任务的命令
