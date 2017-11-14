---
layout:     post
title:      "学习使用scrapy做爬虫"
subtitle:   "Learn scrapy to make web crawler"
date:       2017-11-12 12:00:00
author:     "scyhssm"
header-img: "img/web-crawler.jpg"
header-mask: 0.3
catalog:    true
tags:
    - scrapy
    - web crawler
---

>学习爬虫，先从一个简单的例子敲起

## 创建项目
在创建项目前需要安装好scrapy，如果没有，那下面这条指令会报错。创建项目文件夹tutorial。
```
scrapy startproject tutorial
```
该命令会创建如下文件夹
```
tutorial/
    scrapy.cfg
    tutorial/
        __init__.py
        items.py
        pipelines.py
        settings.py
        middlewares.py
        spiders/
            __init__.py
            ···
```
文件分别是：
scrapy.cfg：
* scrapy.cfg:项目的配置文件
* tutorial/:项目的python模块。在这里添加代码。
* tutorial/items.py:项目中的item文件
* tutorial/pipelines.py:项目中的pipelines文件
* tutorial/settings.py:项目的设置文件
* tutorial/spiders/:放置spider代码的目录

## 定义Item
Item 是保存爬去到的数据的容器；使用方法类似于python字典。
根据需要从dmoz.org获取到的数据对Item进行建模。我们需要从dmoz中获取名字，url，以及网站的描述。
```
import scrapy

class TutorialItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    title = scrapy.Field()
    link = scrapy.Field()
    desc = scrapy.Field()
    pass
```

## 编写第一个爬虫(Spider)
Spider是用户编写用于从单个网站爬取数据的类。
包含类一个用于下载的初始URL，如何跟进网页中的链接以及如何分析页面中的内容，提取生成item的方法。
为了创建Spider，需要继承scrapy.Spider类，定义三个属性：
* name:用于唯一区别Spider。
* start_urls:包含了spider在启动时进行爬取的url列表。第一个页面应该在其中。
* parse():spider的一个方法。被调用时，每个初始URL完成下载后生成的Response对象将会作为唯一的参数传递给该参数。该方法负责解析返回的数据（response data），提取数据（生成Item）以及生成需要进一步处理的URL的Request对象。
```
import scrapy

class TutorialSpider(scrapy.Spider):
    name = "seek114"
    allowed_domains = ["http://www.seek114.com/"]
    start_urls = [
        "http://www.seek114.com/sort/2/21/"
        "http://www.seek114.com/sort/2/22/"
    ]

    def parse(self,response):
    	filename = response.url.split("/")[-2]
    	with open(filename,'wb') as f:
    		f.write(response.body)
```

## 爬取
启动spider:
```
scrapy crawl seek114
```
crawl seek114 启动用于爬取 http://www.seek114.com/ 的spider,注意
```
2017-11-12 20:37:23 [scrapy.core.engine] INFO: Spider opened
2017-11-12 20:37:23 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://www.seek114.com/robots.txt> (referer: None)
2017-11-12 20:37:23 [scrapy.core.engine] DEBUG: Crawled (200) <GET http://www.seek114.com/sort/2/21/http://www.seek114.com/sort/2/22/> (referer: None)
2017-11-12 20:37:23 [scrapy.core.engine] INFO: Closing spider (finished)
```
robots.txt，告诉网络蜘蛛，哪些内容是不应该被获取的。看到输出的log中含有初始URL，log中没有指向其他页面。这里执行失败，如果执行顺利，应该有两个包含url对应内容的文件被创建。

## 刚才的执行过程
scrapy为start_urls属性中每个URL创建了scrapy.Request对象，并将parse方法作为回调函数赋值给Request。
Request对象经过调度，执行生成scrapy.http.Response对象并送回给spider的parse()方法。
在这里要了解回调函数的概念，类似于先将地址给request，request成功执行后生成scrapy.http.Response对象，这时候request再调用parse方法。

## 提取Item
从网页中提取数据有很多方法。scrapy选用了一直基于XPath和CSS表达式机制：Scrapy Selectors。
XPath是XML路径语言，是一种用来确定XML文档中某部分位置的语言。
XPath部分例子及对应的含义：
* /html/head/title：选择HTML文档中<head>标签内的<title>元素  
* /html/head/title/text():选择上面提到的<title>元素的文字  
* //td:选择所有的<td>元素  
* //div[@class='mine']:选择所有具有class="mine"属性的div元素  

为了配合XPath，Scrapy除了提供selector之外，还提供了方法来避免每次从response中提取数据时生成selector的麻烦。
selector有四个基本的方法:
* xpath():传入xpath表达式，返回该表达式所对应的所有节点的selector list列表。
* css():传入css表达式，返回该表达式所对应的所有节点的selector list列表。
* extract():序列化该节点为unicode字符串并返回list。
* re():根据传入的正则表达式对数据进行提取，返回unicode字符串list列表。

## 改写spider
使用XPath解析
```
import scrapy

class TutorialSpider(scrapy.Spider):
    name = "seek114"
    allowed_domains = ["http://www.seek114.com/"]
    start_urls = [
        "http://www.seek114.com/sort/2/21/"
        "http://www.seek114.com/sort/2/22/"
    ]

    def parse(self,response):
    	for sel in response.xpath('//ul/li'):
    		title = sel.xpath('a/text()').extract()
    		link = sel.xpath('a/@href').extract()
    		desc = sel.xpath('text()').extract()
    		print title,link,desc
```

## 使用item
为了将之前爬取到到数据以Item对象返回，修改代码：
```
import scrapy

class TutorialSpider(scrapy.Spider):
    name = "seek114"
    allowed_domains = ["http://www.seek114.com/"]
    start_urls = [
        "http://www.seek114.com/sort/2/21/"
        "http://www.seek114.com/sort/2/22/"
    ]

    def parse(self,response):
    	for sel in response.xpath('//ul/li'):
        item = TutorialItem()
    		item['title'] = sel.xpath('a/text()').extract()
    		item['link'] = sel.xpath('a/@href').extract()
    		item['desc'] = sel.xpath('text()').extract()
    		print title,link,desc
        yield item
```

## 保存爬取到到数据
最简单的方法
```
scrapy crawl seek114 -o items.json
```

## 参考
[scrapy文档](http://scrapy-chs.readthedocs.io/zh_CN/0.24/)
