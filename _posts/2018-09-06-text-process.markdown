---
layout:    post
title:      "Linux文本处理"
subtitle:   "Linux text process"
date:       2018-09-06 12:00:00
author:     "scyhssm"
header-img: "img/linux.jpg"
header-mask: 0.3
catalog:    true
tags:
    - Linux
---

> 面试中的文本处理指令被高频提问，整理下

1. find
最常见和强大的查找命令，用它查找任何想要的文件

2. locate
find -name的另一种写法，比find速度快很多，搜索的是一个数据库(/var/lib/locatedb)。该数据库每天更新，locate查不到最新变动的文件，可以先用updatedb命令

3. whereis
用于程序名的搜索，只搜索二进制文件、man说明文件和源代码文件

4. which
在PATH变量指定路径中搜索某个系统命令的位置

5. type
区分某个命令是shell自带的还是由外部的独立二进制文件提供的

6. grep
文本搜索，可以查找文本中的一部分内容，可以查找多个文本

7. xargs
xargs能够将输入的数据转换为特定命令的命令行参数，把多行变单行，单行变多行

8. sort
可以定义排序规则,对文本进行排序

9. uniq
消除重复行

10. tr
对标准输入的字符进行替换、压缩和删除

11. cut
按列切分文本

12. parse
按列拼接文本

13. wc
统计行和字符的工具

14. sed
用于文本替换

15. awk
数据流处理工具
