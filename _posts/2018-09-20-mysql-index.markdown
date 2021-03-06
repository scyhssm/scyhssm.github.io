---
layout:    post
title:      "MySQL索引详解"
subtitle:   "MySQL index"
date:       2018-09-20 12:00:00
author:     "scyhssm"
header-img: "img/MySQL-img.png"
header-mask: 0.3
catalog:    true
tags:
    - MySQL
---

> 面试的时候遇到MySQL建立索引的问题，发现其学问很深

## 索引前言
索引类似书的目录，有助于快速命中符合条件的记录。不同的存储引擎编排数据的方式不同，因此不同的存储引擎所能支持的索引类型可能也不同。

## 索引类型
### B-Tree索引
B-Tree是MyISAM和InnoDB的默认索引类型，B-Tree索引应用场景。
* 等值匹配
可用于=、!=、<>、IN、NOT IN、<=>查询语句的优化
* 范围匹配
可用于>、>=、<、<=、between and查询语句的优化
* 匹配最左前缀
对于like bai%后缀模糊的查询可以在name上建立索引优化，对%bai前缀模糊的查询无法使用索引
* 覆盖索引
B-Tree的索引key可能包含多列，如果B-Tree索引可以包含所有要查询的值，称其为覆盖索引，否则可能还要根据主键找要查询的值
* 排序
由于B-Tree排好序，所以可以用来优化order by和group by等操作

### 哈希索引
哈希索引基于哈希表实现，哈希索引建得好理论上可以使用哈希索引一次定位，查询效率高于B-Tree索引，但是有限制。
1. 只有精确匹配索引所有列的查询才有效，哈希索引是利用索引的部分字段计算哈希值，因此字段必须完全相同，而不能是部分
2. 不能支持范围查询
3. 哈希索引只包含索引字段的哈希值和指向数据的指针，不能用索引中的值来避免读取行，不可和B-Tree构造的辅助索引一样进行覆盖索引
4. 哈希索引不能进行排序

### 全文索引
一般使用倒排索引实现，记录结构是一个词对应一个链表，链表中元素的结构是DocumentID+Frequency，即词在哪些文档中出现+在文档中出现的频率

### 空间索引
MyISAM支持空间索引，用作地理数据的存储。例如空间索引用一个矩形框住某个区域，当要找某个地理位置时判断精度和纬度范围能够覆盖该地理位置，能够覆盖就找到了矩形框，再找到该地理位置。

## 聚簇索引和非聚簇索引
### 聚簇索引
一张表只有一个聚簇索引，是一种数据存储方式，将主键和数据行存放在同一个文件，用主键去索引，如果没有主键系统会自动生成一个不为空的索引列作为主键。  
二级索引（辅助索引），叶子节点存放主键值，用于到聚簇索引中找到数据行。可以指定多列或者单列构成二级索引，可以构造多个辅助索引加快查询速度。

### 非聚簇索引
非聚簇索引的索引和数据存放在不同的文件，对MyISAM的一张表会有三种文件：FRM（表结构）、MYD（数据行）、MYI（索引）。

## 高效索引方法及注意点
索引会占用空间，加重插入、删除和修改记录的负担，要正确使用索引
### 独立的列
索引列不能是表达式的一部分，也不能是函数的参数
### 索引的选择性
选择能够过滤掉更多数据行的列，选择选择性高的列
### 前缀索引
由于TEXT、VARCHAR内容很多，对TEXT、VARCHAR使用前缀索引，减少索引空间
### 联合索引
可以对多列进行索引，对多列都进行排序，条件更精确，如果找最左边的列但是不找右边的列，找到后其他列也是经过排序的
### 覆盖索引
索引包含所有要查询字段的信息，可以见上
### 冗余和重复索引
重复索引指在相同的列按照相同顺序创建了相同类型的索引
* 如主键索引、唯一索引重复创建，都是根据主键实现的
* 如创建联合索引(A,B)再创建索引A就是冗余索引

### 索引最左匹配原则
当使用联合索引(A,B)时，对A单独查找可以用到联合索引(A,B),但是对B不行，实际上建立了两个索引(A),(A,B)。  
对A单独查找绝对有序，而B是第二条件，直接使用B是用不到索引的，可以参照B+Tree的结构。
