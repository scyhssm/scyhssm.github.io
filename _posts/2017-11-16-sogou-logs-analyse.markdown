---
layout:     post
title:      "搜狗搜索日志分析"
subtitle:   "Analyse the log of sogou"
date:       2017-11-16 12:00:00
author:     "scyhssm"
header-img: "img/sogou.jpg"
header-mask: 0.3
catalog:    true
tags:
    - hadoop
    - hive
    - 数据分析
    - 搜狗搜索日志
---

>在大数据的情况下，hadoop生态圈中的hive是一款处理分析数据的绝佳利器。

## 数据分析
首先我们来了解下数据分析是什么。  
数据分析是指用适当的统计分析方法对收集来的大量数据进行分析。  
提取有用信息和形成结论而对数据加以详细研究和概括总结的过程。  
这一过程也是质量管理体系的支持过程。  
在实用中，数据分析可帮助人们作出判断，以便采取适当行动。  
在数据量较小的情况下，Excel就是一款常用的分析工具。  
如果在数据量增长变得很大的情况下，Excel的缺点就暴露出来了。  
一般的关系型数据库则能够支持更大量的数据，对数据分析。  
但是在大数据时代，存储内容增加以及关系型数据库的劣势显现出来。  
很多数据并不一定有固定的关系，能够以数据表来规范。  
数据分析要能够适应大的数据量，这样结论才更深刻，更具有普遍性。  
因此该实验基于HDFS和HIVE做数据分析，来挖掘搜索词条下的潜藏价值。
## 数据过滤
虽然数据已经被过滤过，但是如有需要过滤的，可以使用如下代码进行数据过滤。
```
import java.io.*;

public class filter{
  public static void main(String[] args){
    try {
      FileReader fr = new FileReader("home/HADOOP/sogou-data/sogou.full.utf8");
      FileWriter fw = new FileWriter("home/HADOOP/sogou-data/sogou.full.utf8.flt");
      BufferedReader br = new BufferedReader(fr);
      BufferedWriter bw = new BufferedWriter(fw);

      String str = null;

      while((str = br.readLine()) != null){
        String[] fields = str.split("\t");
        if(fields[1]!=""&&fields[2]!=""){
          System.out.println(fields[1]+fields[2]);
          bw.write(str+"\n");
          System.out.println(str);
        }
      }
      bw.close();
      fw.close();
      br.close();
      fr.close();
    }
    catch(FileNotFoundException e){
      e.printStackTrace();
    }
    catch(IOException e){
      e.printStackTrace();
    }
  }
}
```

## 数据扩展
所有的数据已经被过滤过，因此直接扩展，这步要格外小心，因为硬盘空间可能不够。  
使用java对数据进行扩展,拆分时间字段，添加年、月、日、小时。  
如果在linux中对数据进行操作，shell脚本分割更方便，扩展凭喜好。  
不过在面临4.6G+这样的大文本文件时，尽量选择速度快的，底层一些的语言扩展。  
```
import java.io.*;

public class fenge{
  public static void main(String[] args){
    try{
      FileReader reader = new FileReader("/home/HADOOP/sogou-data/sogou.full.utf8");
      FileWriter writer = new FileWriter("/home/HADOOP/sogou-data/sogou.full.utf8.ext");
      BufferedWriter bw = new BufferedWriter(writer);
      BufferedReader br = new BufferedReader(reader);

      String str = null;
      String y = null;
      String m = null;
      String d = null;
      String h = null;
      String mi = null;
      String se = null;

      while((str = br.readLine())!= null){
        System.out.println(str);
        y = "\t"+str.substring(0,4);
        m = "\t"+str.substring(4,6);
        d = "\t"+str.substring(6,8);
        h = "\t"+str.substring(8,10);
        mi = "\t"+str.substring(10,12);
        y = "\t"+str.substring(12,14);
        str = str+y+m+d+h+mi+se;
        System.out.println(str);
        bw.write(str+"\n");
      }
      bw.close();
      writer.close();
      br.close();
      reader.close();
    }
    catch(FileNotFoundException e){
      e.printStackTrace();
    }
    catch(IOException e){
      e.printStackTrace();
    }
  }
}
```
再将该文件内容放到hdfs中,并查看是否成功放置。  
```
hadoop fs -put /home/HADOOP/sogou-data/sogou.full.utf8.ext /sogou_full
hadoop fs -ls /sogou_full
```
由于后面对数据操作4.6G+对数据量过大，因此选用容量较小的数据
```
hadoop dfs -mkdir -p /sogou/20111230
hadoop dfs -put /home/zkpk/resources/sogou-data/500w/sogou.500w.utf8 /sogou/20111230/
hadoop dfs -mkdir -p /sogou_ext/20111230
hadoop dfs -put /home/zkpk/resources/sogou-data/500w/sogou.500w.utf8.flt /sogou_ext/20111230
```
## 利用Hive构建日志数据的数据仓库
启动hadoop集群
```
cd $HADOOP/bin
./start-all.sh
```
启动hive  
```
/usr/hive/bin/hive
```
查看数据库
```
show databases;
```
创建数据库
```
create database sogou;
```
使用数据库
```
use sogou;
```
查看所有表名
```
show tables;
```
创建外部表
```
CREATE EXTERNAL TABLE sogou.sogou_20111230(
ts STRING,
uid STRING,
keyword STRING,
rank INT,
orderx INT,
url STRING)
COMMENT 'This is the sogou search data of one day'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/sogou/20111230';
```
如果已经在sogou数据库内,表名前的sogou.可以去掉  
查看新创建表结构
```
show create table sogou.sogou_20111230;
describe sogou.sogou_20111230;
```
删除表
```
drop table sogou.sogou_20111230;
```
## 创建分区表
按照年、月、天、小时分区，外部表使用hdfs中的数据，不需要拷贝数据到hive中去。
```
CREATE EXTERNAL TABLE sogou.sogou_ext_20111230(
ts STRING,
uid STRING,
keyword STRING,
rank INT,
xorder INT,  #注意不可以用order，否则会报错，因为order已经被hive定义
url STRING,
year INT,
month INT,
day INT,
hour INT
)
COMMENT 'This is the sogou search data of extend data'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/sogou_ext/20111230';
```
创建带分区的表
```
CREATE EXTERNAL TABLE sogou.sogou_partition(
ts STRING,
uid STRING,
keyword STRING,
rank INT,
xorder INT,
url STRING
)
COMMENT 'This is the sogou search data by partition'
partitioned by (
year INT,
month INT,
day INT,
hour INT
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;
```
灌入数据
```
set hive.exec.dynamic.partition.mode=nonstrict;
INSERT OVERWRITE TABLE sogou.sogou_partition
PARTITION(year,month,day,hour)
select * from sogou.sogou_ext_20111230;
```
注意这里灌入数据需要选用一张已经创建好的正常表  
查询结果
```
select * from sogou_ext_20111230 limit 10;
select url from sogou_ext_20111230 limit 10;
select * from sogou_ext_20111230
where uid='96994a0480e7e1edcaef67b20d8816b7';
```
## 条数统计
由于统计分析使用大容量的数据和相对较大的数据，相对较大的数据查询更快。  
且主要目的是分析数据，达到实验的效果。  
因此选用500W条数据记录，也能起到良好统计分析效果。  
### 数据总条数
```
select count(*) from sogou_partition;
```
结果为500W
### 非空查询条数
```
select count(*) from sogou_partition where keyword is not null and keyword != '';
```
由于我们进行过数据过滤，因此结果也为500W
### 无重复总条数
```
select count(*) from (select ts from sogou_partition
group by ts,uid,keyword,url having count(*) = 1) a;
```
按照ts，uid，keyword，url无重复来计算无重数数量。  
这里select后不能跟*,因为group过的一些值会变化，用*会报错。  
该条语句相对于前2条执行时间较长，因为计算复杂。  
可想而知在4.6G数据量下执行的时间，执行结果为4999272。  
### 独立UID总数
```
select count(distinct(uid)) from sogou_ext_20111230;
```
结果为1352664
### 某个时间总数
```
select count(*)  from sogou_partition where hour=2;
```
sogou_partition是分区表，寻找某个时间总条数速度非常快。结果为45881。
## 关键词分析
### 查询关键词长度
```
select avg(a.cnt) from (select size(split(keyword,'\\s+')) as cnt from sogou_ext_20111230) a;
```
在这里'\\s+'的含义解释：\\s代表空格、回车、换行等字符，+表示一个或者多个。  
即将keyword分割开来。  
这条查询语句长度为1.0869984。  
### 查询频度排名
```
select keyword,count(*) as cnt from sogou_ext_20111230
group by keyword order by cnt desc limit 50;
```
通过对相同的查询键值进行group操作,查询结果：
```
百度	38441
baidu	18312
人体艺术	14475
4399小游戏	11438
qq空间	10317
优酷	10158
新亮剑	9654
馆陶县县长闫宁的父亲	9127
公安卖萌	8192
百度一下 你就知道	7505
百度一下	7104
4399	7041
魏特琳	6665
qq网名	6149
7k7k小游戏	5985
黑狐	5610
儿子与母亲不正当关系	5496
新浪微博	5369
李宇春体	5310
新疆暴徒被击毙图片	4997
hao123	4834
123	4829
4399洛克王国	4112
qq头像	4085
nba	4027
龙门飞甲	3917
qq个性签名	3880
张去死	3848
cf官网	3729
凰图腾	3632
快播	3423
金陵十三钗	3349
吞噬星空	3330
dnf官网	3303
武动乾坤	3232
新亮剑全集	3210
电影	3155
优酷网	3115
两次才处决美女罪犯	3106
电影天堂	3028
土豆网	2969
qq分组	2940
全国各省最低工资标准	2872
清代姚明	2784
youku	2783
争产案	2755
dnf	2686
12306	2682
身份证号码大全	2680
火影忍者	2604
```
从热搜词上可以看出搜狗用户的喜好，以及当时的热点。
### 查询用户点击该url位置排名
```
select rank,count(*) as cnt from sogou_ext_20111230
group by rank order by cnt desc ;
```
返回结果
```
1	2071720
2	905769
3	554258
4	375813
5	283848
6	218351
7	179380
8	151384
10	131002
9	128344
12	21
13	19
11	19
14	10
16	9
15	6
19	4
17	4
```
可以看到结果基本呈下降趋势，但是会有异常点出现。  
比如第9个会小于第10个条目的点击量。  
搜狗的搜索结果一般呈现为一页10条。  
可能的情况为有很多人在选择url时选着选着就拉到了最下面。  
第10条被点一次点击的概率较第九条就会更大。  
![点击位置总数](/img/点击位置总数.png)
而过了10条后，后面的内容被点击的次数锐减，因为这些内容在第二页。  
显然更多人会选择第一页的内容作为选择条目。  
大多数人习惯于点击第一页呈现的条目。  
从算法的角度上来说，第一页的内容的确和用户搜索的关联度最大。  
### 查询用户点击的顺序号排名
```
select xorder,count(*) as cnt from sogou_ext_20111230
group by xorder order by cnt desc;
```
返回结果
```
1	3465833
2	912017
3	355093
4	150291
5	65530
6	29253
7	13068
8	5731
9	2372
10	812
```
用户点击一定是呈递减的状态，递减的幅度不会太大，一般会减少1-3倍。
![点击顺序号总数](/img/点击顺序号总数.png)
## UID分析
### UID的查询次数分布
```
select SUM(IF(uids.cnt=1,1,0)),SUM(IF(uids.cnt=2,1,0)),
SUM(IF(uids.cnt=3,1,0)),SUM(IF(uids.cnt>3,1,0)) from
(select uid,count(*) as cnt from sogou_ext_20111230 group by uid) uids;
```
返回结果549148	257163	149562	396791，查询次数呈递减的状态。  
递减的幅度还不够大，说明sogou的结果还不尽人意。  
否则不会一个ID搜索多次，建议再优化算法。  
### UID平均查询次数
```
select sum(a.cnt)/count(a.uid) from (select uid,count(*) as cnt from
sogou_ext_20111230 group by uid) a;
```
返回结果3.6964094557111005。  
由于没有同一时段百度，谷歌的用户平均查询次数统计，不能妄下结论。  
如果相比较而言平均查询次数较高的话。  
可能是搜狗的查询内容还没有满足用户的需求,用户需要多次查询才能得到满意结果。  
### 查询次数大于2次的用户总数
```
select count(a.uid) from (
select uid,count(*) as cnt from sogou_ext_20111230
group by uid having cnt > 2) a;
```
### 查询次数大于2次的用户占比
#### UID总数
```
select count(distinct (uid)) from sogou.sogou_ext_20111230;
```
结果为1352664
#### UID2次以上的数量
```
select count(a.uid) from (
select uid,count(*) as cnt from sogou.sogou_ext_20111230
group by uid having cnt > 2) a;
```
结果为546353
第二次除以第一次 546353/1352664 = 0.4，说明查询次数在2次以上的用户只占了小部分。
### 查询次数大于2次的数据展示
```
select b.* from
(select uid,count(*) as cnt from sogou_ext_20111230
group by uid having cnt > 2) a
join sogou_ext_20111230 b on a.uid=b.uid
limit 50;
```
该条hql限制展示了50条数据，否则所有大于2次的用户都会被记录下来。  
很多用户查询的是其他的搜索引擎，所以只在sogou中查询一次，该用户就不继续使用sogou。  
结果如下
```
20111230222802	000080fd3eaf6b381e33868ec6459c49	福彩3d单选号码走势图	11	http://zst.cjcp.com.cn/cjw3d/view/3d_danxuan.php	2011	12	30	22
20111230222603	000080fd3eaf6b381e33868ec6459c49	福彩3d单选一注法	10	5	http://www.18888.com/read-htm-tid-6069520.html	2011	12	30	22
20111230222158	000080fd3eaf6b381e33868ec6459c49	福彩3d单选一注法	63	http://bbs.17500.cn/thread-2453170-1-1.html	2011	12	30	22
20111230220953	000080fd3eaf6b381e33868ec6459c49	福彩3d单选一注法	41	http://www.55125.cn/3djq/20111103_352210.htm	2011	12	30	22
20111230222417	000080fd3eaf6b381e33868ec6459c49	福彩3d单选一注法	74	http://bbs.18888.com/read-htm-tid-4017348.html	2011	12	30	22
20111230222128	000080fd3eaf6b381e33868ec6459c49	福彩3d单选一注法	52	http://www.zibocn.com/Infor/i8513.html	2011	12	30	22
20111230211504	0000c2d1c4375c8a827bff5dab0cc0a6	穿越小说txt	3	2http://www.booktxt.com/chuanyue/	2011	12	30	21
20111230213029	0000c2d1c4375c8a827bff5dab0cc0a6	浮生若梦txt	1	1http://ishare.iask.sina.com.cn/f/15694326.html?from=like	2011	12	30	21
20111230211319	0000c2d1c4375c8a827bff5dab0cc0a6	穿越小说txt	2	1http://www.zlsy.net.cn/	2011	12	30	21
20111230213047	0000c2d1c4375c8a827bff5dab0cc0a6	浮生若梦txt	2	2http://www.txtinfo.com/txtshow/txt6105.html	2011	12	30	21
20111230205803	0000c2d1c4375c8a827bff5dab0cc0a6	步步惊心歌曲	4	1http://www.tingge123.com/zhuanji/1606.shtml	2011	12	30	20
20111230205643	0000c2d1c4375c8a827bff5dab0cc0a6	步步惊心主题曲	4	1http://bubujingxin.net/music.shtml	2011	12	30	20
20111230212531	0000c2d1c4375c8a827bff5dab0cc0a6	乱世公主txt	1	1http://ishare.iask.sina.com.cn/f/20689380.html	2011	12	30	21
20111230210041	0000c2d1c4375c8a827bff5dab0cc0a6	步步惊心歌曲	5	2http://www.yue365.com/mlist/10981.shtml	2011	12	30	21
20111230213911	0000c2d1c4375c8a827bff5dab0cc0a6	浮生若梦小说在线阅读	21	http://www.readnovel.com/partlist/22004/	2011	12	30	21
20111230213835	0000c2d1c4375c8a827bff5dab0cc0a6	浮生若梦小说txt下载	21	http://www.2yanqing.com/f_699993/244670/download.html	2011	12	30	21
20111230195652	0000d08ab20f78881a2ada2528671c58	棉花价格行情走势图	11	http://www.yz88.org.cn/jg/	2011	12	30	19
20111230200124	0000d08ab20f78881a2ada2528671c58	棉花价格最新	1	1http://www.cnjidan.com/mianhua.asp	2011	12	30	20
20111230195312	0000d08ab20f78881a2ada2528671c58	棉花价格	3	3http://www.yz88.org.cn/jg/	2011	12	30	19
20111230200339	0000d08ab20f78881a2ada2528671c58	棉花价格最新	2	2http://www.yz88.org.cn/jg/	2011	12	30	20
20111230200016	0000d08ab20f78881a2ada2528671c58	棉花价格最新	1	1http://www.cnjidan.com/mianhua.asp	2011	12	30	20
20111230194925	0000d08ab20f78881a2ada2528671c58	棉花价格	1	1http://www.socotton.com/News/NewsList.asp?CateID=Industry	2011	12	30	19
20111230202227	0000d08ab20f78881a2ada2528671c58	weifangmianbu pifashichang	1	1	http://wenwen.soso.com/z/q248759586.htm	2011	12	30	20
20111230200844	0000d08ab20f78881a2ada2528671c58	weifangmianbu pifashichang	1	1	http://wenwen.soso.com/z/q248759586.htm	2011	12	30	20
20111230195114	0000d08ab20f78881a2ada2528671c58	棉花价格	2	2http://www.cnjidan.com/mianhua.asp	2011	12	30	19
20111230222857	0000e7482034da216ce878a9f16feb49	ppt	1	1	http://www.51ppt.com.cn/	2011	12	30	22
20111230224913	0000e7482034da216ce878a9f16feb49	powerpoint	1	1http://www.skycn.com/soft/14233.html	2011	12	30	22
20111230223155	0000e7482034da216ce878a9f16feb49	ppt	6	2	http://www.pptbz.com/	2011	12	30	22
20111230224641	0000e7482034da216ce878a9f16feb49	格式转换工厂	2	1http://dl.pconline.com.cn/download/51244.html	2011	12	30	22
20111230224424	0000e7482034da216ce878a9f16feb49	小说下载	1	1http://www.txtbook.com.cn/	2011	12	30	22
20111230230330	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	72	http://blog.sina.com.cn/s/blog_5da99b370100lq09.html	2011	12	30	23
20111230222838	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发公司房地产开发费用最高扣除额	8	3	http://www.chinaacc.com/new/287_292_201103/18wa1215406044.shtml	2011	12	30	22
20111230224552	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	54	http://www.duk.cn/d-287538.html	2011	12	30	22
20111230223024	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	41	http://zhidao.baidu.com/question/71864587	2011	12	30	22
20111230223729	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	62	http://www.ck100.com/shiwu/200903/51884.html	2011	12	30	22
20111230223806	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	53	http://zhidao.baidu.com/question/168690048	2011	12	30	22
20111230223852	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	51	http://www.360doc.com/content/11/0304/16/3189259_98087575.shtml	2011	12	30	22
20111230223928	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	42	http://www.360doc.com/content/10/1206/16/2901133_75548781.shtml	2011	12	30	22
20111230224752	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	76	http://www.cnnsr.com.cn/jtym/swk/20100902/2010090208464558635.shtml	2011	12	30	22
20111230224710	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	65	http://www.123cha.com/classinfo/250791.html	2011	12	30	22
20111230225956	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	10	7	http://www.cnnsr.com.cn/jtym/swk/20101022/2010102209561859976.shtml	2011	12	30	22
20111230230048	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	10	1	http://www.cnnsr.com.cn/jtym/swk/20101022/2010102209561859976.shtml	2011	12	30	23
20111230224538	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	43	http://hangzhou.fangtoo.com/wd/inqd162703a.html	2011	12	30	22
20111230230140	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	91	http://www.redlib.cn/html/9160/2010/71212862.htm	2011	12	30	23
20111230222544	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发公司房地产开发费用最高扣除额	2	2	http://www.chinaacc.com/new/635_796/2010_6_11_su73151552431116010229315.shtml	2011	12	30	22
20111230222029	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发公司房地产开发费用最高扣除额	1	1	http://china.findlaw.cn/fangdichan/tudi/tdzcs/tdzcsjs/694.html	2011	12	30	22
20111230224313	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	93	http://www.lawtime.cn/info/shuifa/fcs/2011090957849.html	2011	12	30	22
20111230171823	000312ca0eaa91c30e5bafbcf2981bfd	厦门工商红盾信息网	11	http://www.xm.fjaic.gov.cn/	2011	12	30	17
20111230224455	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	21	http://www.lawtime.cn/info/shuifa/fcs/2011121974290.html	2011	12	30	22
20111230230259	000312ca0eaa91c30e5bafbcf2981bfd	房地产开发费用扣除	51	http://bbs.chinaacc.com/forum-3-7/topic-1138731.html	2011	12	30	23
```
这里筛选出的是大于2次用户搜索的内容时间点击的url等一系列记录。  
## 用户行为分析
### 点击次数与Rank之间的关系分析
rank在10以内的点击次数  
```
select count(*) from sogou_ext_20111230 where rank < 11;
```
结果是4999869  
总数
```
select count(*) from sogou_ext_20111230;
```
结果是5000000
计算占比，4999869／5000000 = 99.99%，说明绝大部分人只看搜索引擎的前10个结果。  
该结果和我平时的习惯也对应，同时和上面 *查询用户点击该url位置排名* 结果相同。  
所以传统的基于整个结果集合查准率和查全率的评价方式不再适用于网络信息检索的评价。  
我们需要着重强调在评价指标中有关最靠前结果文档与用户查询需求的相关度的部分。  
说明广告放在第一页这个做法非常流氓，相对减少了我们获取到的内容。  
### 直接输入URL查询
直接输入URL的查询数
```
select count(*) from sogou.sogou_ext_20111230
where keyword like '%www%';
```
‘%www%’是hql的正则匹配，搜索内容中凡是出现了www的都会被统计在内。  
该结果为73979。  
总数为
```
select count(*) from sogou.sogou_ext_20111230;
```
结果为5000000  
占比 73979/5000000=1.48%,占比非常小。  
按照个人的经验，一般这样子输入的情况是:  
1.不会使用浏览器，以为在搜索栏输入地址就可以直接进入  
2.想进入一个网站但是输错地址，导致其成为一次搜索条目  
3.不确定网址，想通过这种方法找到精确网址  
直接输入URL的查询中，点击数点击的结果就是用户输入的URL的网址所占的比例  
```
select SUM(IF(instr(url,keyword)>0,1,0)) from
(select * from sogou.sogou_ext_20111230 where keyword like '%www%') a;
```
结果为27561，所占比例27561 / 73979 = 37.25%，比重不大，只是小部分。  
由此印证了上述观点并得出结论：  
很大一部分用户提交含有URL的查询是由于没有记全网址想借助引擎找到自己想浏览的网页。  
搜索引擎在处理这部分查询的时候，可能的比较理想的方式是首先把相关的完整URL返回给用户。  
这样比较容易符合用户查询需求。  
### 统计不同的时间段活跃用户数和搜索数量
该数据最小分到小时，因此按照不同的小时来分。  
我们之前已经创建了分区表，细分到小时，因此可以直接使用。  
分区表是一种粗粒度，简易的索引策略。  
该数据量还不够大，其真正适合是大数据的场景。  
即先按照一种规则将数据分区，方便对数据管理，而不是为了提高性能。  
在某些场合下可能会快于不分区的表。  
先统计不同时间段的搜索数量  
在sogou_partition表下操作
```
select hour,count(*) from sogou_partition group by hour order by hour desc;
```
在sogou_ext_20111230表下操作
```
select hour,count(*) from sogou_ext_20111230 group by hour order by hour desc;
```
结果相同，第一个操作用时197秒，第二个操作用时158秒。
```
23	194554
22	270842
21	328949
20	353101
19	340115
18	295207
17	289648
16	317122
15	318645
14	306242
13	295936
12	274234
11	276103
10	315973
9	279105
8	165616
7	52832
6	32991
5	28213
4	27924
3	34244
2	45881
1	65707
0	90816
```
在凌晨搜索条目最少,高峰时段是从早上9点起到晚上22点，通过和自己比较，还是非常符合的。  
![搜索总次数柱状图](/img/24小时1.png)
这是某一天的情况，可能有特殊性，比如说周末是否午夜搜索数量会相较平时多更多，都是待发掘的。  
统计不同时间段的用户数量
```
select hour,count(*) from (select hour,uid from sogou_ext_20111230 group by hour,uid) a
group by hour order by hour desc;
```
由于hive有别名检查，因此在  
(select hour,uid from sogou_ext_20111230 group by hour,uid)语句  
后必须加入a否则会报错。
统计结果
```
23	76443
22	108551
21	132394
20	142621
19	139642
18	125869
17	125386
16	131140
15	129013
14	125464
13	123115
12	120189
11	118303
10	132183
9	120027
8	77109
7	25867
6	14339
5	11144
4	11281
3	13499
2	18128
1	25935
0	38459
```
用户数和搜索条目一样，呈相同的分布。  
![搜索用户数柱状图](/img/24小时2.png)
这是当天的搜索数据，没有离群点。  
可以推测，如果某个时段发生重大时间，搜索记录数是不是会升高。  
经过计算，比例相近，没有特殊的离群点，但在早上会有一个低谷。  
![搜索条目/用户数](/img/24小时3.png)
## 独立用户行为分析
### 通过搜狗查询其他搜索引擎的搜索条目
```
select count(*) from sogou_ext_20111230 where
(keyword like '%百度%' or keyword like '%baidu%'
or keyword like '%谷歌%' or keyword like '%google%');
```
主要的搜索引擎为百度谷歌，其他部分占比很小，因此主要统计百度和谷歌的搜索量。  
结果为92846
```
select count(*) from sogou.sogou_ext_20111230;
```
结果为5000000，占比为92846/5000000=1.85%。  
说明有一小部分用户使用sogou的原因是要用其查找别的搜索引擎。  
这部分用户并不是sogou的忠实用户。  
我们可以进一步分析这些用户的行为状态来修改sogou搜索引擎的体验。
### 查询搜索过“仙剑奇侠传”的用户
查询搜索过“仙剑奇侠传”的uid，并且次数大于三
```
select uid,count(*) as cnt from sogou.sogou_ext_20111230 where keyword='仙剑奇侠传'
group by uid having cnt > 3;
```
获得结果
```
653d48aa356d5111ac0e59f9fe736429	6
e11c6273e337c1d1032229f1b2321a75	5
```
这两位同志搜索仙剑奇侠传较多，但是据我所知，仙剑奇侠传有影视作品也有游戏。  
所以需要查找用户搜索记录来得知其需要的内容。  
查找uid是653d48aa356d5111ac0e59f9fe736429的搜索记录
```
select * from sogou.sogou_ext_20111230 where uid='653d48aa356d5111ac0e59f9fe736429';
```
结果为
```
20111230132620	653d48aa356d5111ac0e59f9fe736429	放羊的星星	2	1http://tv.sohu.com/s2010/fydxx/	2011	12	30	13
20111230132714	653d48aa356d5111ac0e59f9fe736429	放羊的星星	1	1http://tv.sogou.com/series/wxt4vu5644qlpror6k24jugh2ddq.html?p=40230600	2011	12	30	13
20111230132832	653d48aa356d5111ac0e59f9fe736429	放羊的星星	2	1http://tv.sohu.com/s2010/fydxx/	2011	12	30	13
20111230151148	653d48aa356d5111ac0e59f9fe736429	我可能不会爱你	2	1http://tv.sohu.com/s2011/wknbhan/	2011	12	30	15
20111230151459	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传	5	1http://www.163dyy.com/detail/500.html	2011	12	30	15
20111230151537	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传	1	1http://www.tvmao.com/drama/WVgxbA==/episode	2011	12	30	15
20111230161956	653d48aa356d5111ac0e59f9fe736429	7聊	1	1	http://www.7liaos.com/	2011	12	30	16
20111230170003	653d48aa356d5111ac0e59f9fe736429	7聊	1	1	http://www.7liaos.com/	2011	12	30	17
20111230205230	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传	5	1http://www.163dyy.com/detail/500.html	2011	12	30	20
20111230205453	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传第一部全集	11	http://tv.sogou.com/series/wxt4vu5644qm7sn5updont6awsv3lwwsxozl6.html?p=40230600	2011	12	30	20
20111230205645	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传第一部	61	http://www.youku.com/playlist_show/id_16700878.html	2011	12	30	20
20111230205830	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传1	1	1http://tv.sogou.com/series/wxt4vu5644qm7sn5updont6awsv3lwwsxozl6.html?p=402306002011	12	30	20
20111230210113	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠转1	2	1http://www.youku.com/playlist_show/id_3549043.html	2011	12	30	21
20111230210302	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传	5	1http://www.163dyy.com/detail/500.html	2011	12	30	21
20111230210423	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传	3	1http://www.114dyw.com/teleplay1/xianjianqixiachuan/	2011	12	30	21
20111230210532	653d48aa356d5111ac0e59f9fe736429	仙剑奇侠传	5	1http://www.163dyy.com/detail/500.html	2011	12	30	21
```
查找uid是e11c6273e337c1d1032229f1b2321a75的搜索记录
```
select * from sogou.sogou_ext_20111230 where uid='e11c6273e337c1d1032229f1b2321a75';
```
结果为
```
20111230054505	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传4官网	1	1http://pal4.52pk.com/	2011	12	30	5
20111230054532	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传4结局	1	1http://zhidao.baidu.com/question/196334214	2011	12	30	5
20111230055017	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传4	5	1http://baike.baidu.com/view/10142.htm	2011	12	30	5
20111230055036	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传3	7	1http://baike.baidu.com/view/33571.htm	2011	12	30	5
20111230055048	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传2	6	1http://baike.baidu.com/view/246644.htm	2011	12	30	5
20111230055056	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传	4	1http://baike.baidu.com/view/2188.htm	2011	12	30	5
20111230055118	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传三外传	11	http://baike.baidu.com/view/246650.htm	2011	12	30	5
20111230063411	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传四动画	41	http://www.56.com/w77/play_album-aid-1824744_vid-MTY3MjkwOTc.html	2011	12	30	6
20111230064818	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传3结局动画	21	http://v.youku.com/v_show/id_XNDczMTU3Ng==.html	2011	12	30	6
20111230065134	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传3结局	8	1http://zhidao.baidu.com/question/143395514	2011	12	30	6
20111230065159	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传三	5	1http://baike.baidu.com/view/4219.htm	2011	12	30	6
20111230065251	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传三游戏剧情	21	http://zhidao.baidu.com/question/106721096	2011	12	30	6
20111230065336	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传三	5	1http://baike.baidu.com/view/4219.htm	2011	12	30	6
20111230072523	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传四	2	1http://baike.baidu.com/view/31425.htm	2011	12	30	7
20111230072543	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传	4	1http://baike.baidu.com/view/2188.htm	2011	12	30	7
20111230074026	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传二	2	1http://baike.baidu.com/view/246644.htm	2011	12	30	7
20111230080950	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传	4	1http://baike.baidu.com/view/2188.htm	2011	12	30	8
20111230081857	e11c6273e337c1d1032229f1b2321a75	阿奴	2	1	http://baike.baidu.com/view/47446.htm	2011	12	30	8
20111230092353	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传	4	1http://baike.baidu.com/view/2188.htm	2011	12	30	9
20111230093148	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传二	2	1http://baike.baidu.com/view/246644.htm	2011	12	30	9
20111230101020	e11c6273e337c1d1032229f1b2321a75	仙剑奇侠传	4	1http://baike.baidu.com/view/2188.htm	2011	12	30	10
20111230110154	e11c6273e337c1d1032229f1b2321a75	Grenade	2	1	http://baike.baidu.com/view/2086505.htm	2011	12	30	11
20111230140524	e11c6273e337c1d1032229f1b2321a75	北京庐舍宾馆	2	1http://baike.baidu.com/view/4916228.htm	2011	12	30	14
20111230140619	e11c6273e337c1d1032229f1b2321a75	北京庐舍宾馆	3	2http://www.zhuna.cn/hotel-23516.html	2011	12	30	14
20111230140727	e11c6273e337c1d1032229f1b2321a75	北京庐舍宾馆	4	3http://www.17u.cn/HotelInfo-27993.html	2011	12	30	14
20111230140751	e11c6273e337c1d1032229f1b2321a75	北京庐舍宾馆	1	4http://www.sunnychina.com/hotel/hotel_15894.html	2011	12	30	14
20111230140759	e11c6273e337c1d1032229f1b2321a75	北京庐舍宾馆	8	5http://www.yoostrip.com/hotel/hotel_17602.html	2011	12	30	14
20111230140827	e11c6273e337c1d1032229f1b2321a75	北京庐舍宾馆	9	6http://hotel.elong.com/detail360_cn_00101382.html	2011	12	30	14
20111230143033	e11c6273e337c1d1032229f1b2321a75	如家	1	1	http://www.homeinns.com/	2011	12	30	14
20111230181611	e11c6273e337c1d1032229f1b2321a75	东洛杉矶学院	1	1http://baike.baidu.com/view/4932647.htm	2011	12	30	18
20111230181707	e11c6273e337c1d1032229f1b2321a75	东洛杉矶学院	2	2http://www.elac.edu/	2011	12	30	18
```
第一位用户搜索仙剑奇侠传是搜索其影视作品，第二位用户搜索仙剑奇侠传是搜索其游戏作品。  
为了精准推荐，给用户的搜索结果可以参考用户其他的搜索记录。  
确定用户对哪方面内容感兴趣，投放更精准的推荐。  
但是不能缺失推荐的多样性，有可能该用户虽然喜欢影视，但是有可能会对游戏也有产生兴趣。  
因此该方法对算法的要求会很高。  
### 其他的查询
除了上述一些内容以外，还有很多查询内容值得考虑。  
用户搜索条目的平均长度，如果要更加细化，可以归类一种用户，专门分析该类用户的搜索长度。  
如果可以将搜索内容分类，还可以添加例如对某类用户搜索的活跃时间计算。  
## 实时数据
每个UID在当天的查询点击次数
### 创建临时表
```
create table sogou.uid_cnt(uid STRING, cnt INT)
COMMENT 'This is the sogou search data of one day'
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE;
```
### 查询并插入
```
INSERT OVERWRITE TABLE sogou.uid_cnt select uid,count(*)
as cnt from sogou.sogou_ext_20111230 group by uid;
```
获取前10条记录,记录显示正常。
```
00005c113b97c0977c768c13a6ffbb95	2
000080fd3eaf6b381e33868ec6459c49	6
0000c2d1c4375c8a827bff5dab0cc0a6	10
0000d08ab20f78881a2ada2528671c58	9
0000e7482034da216ce878a9f16feb49	5
0001520a31ed091fa857050a5df35554	1
0001824d091de069b4e5611aad47463d	1
0001894c9f9de37ef9c90b6e5a456767	2
0001b04bf9473458af40acb4c13f1476	1
0001f5bacf60b0ff8c1c9e66e4905c1f	2
```
