---
layout:     post
title:      "关于j2ee项目启动问题的解决"
subtitle:   "resolved problems of init j2ee projects"
date:       2017-07-03 12:00:00
author:     "scyhssm"
header-img: "img/sunrise.jpg"
header-mask: 0.3
catalog:    true
tags:
    - j2ee
    - maven
    - 阿里云中央仓库
---

>前几天老师给一个项目，要求我能够让这个项目run起来。

>于是乎按照这个项目的要求改jdk，加maven依赖，下载能够运行项目的tomcat版本，结果运行不起来。


## 中文路径

关于中文路径这个问题，我的建议是，无论是啥项目，最好都别涉及到中文。内容涉及到中文无所谓，比较悲催的是，作为一名程序员，在重装windows的时候，初始化用户名写了自己的大名，不用缩写，耿直得可怕。这就导致一系列问题，User这个文件夹是以我的大名为文件夹名的，把用户名改了也不顶用。

eclipse默认maven仓库位置是在User下.m2这个文件夹，本着不要让路径出现中文的原则，我把eclipse默认maven仓库位置修改到D盘下。再在D盘建了一个settings.xml文件，项目报的问题又翻新了。


---


## 新的问题

新的问题报得奇怪，org.apache.catalina.LifecycleException 大概是这个问题。Google到的解决方案简直是五花八门，事实证明没有一个适合我的。果然光上网寻求解决方案不结合实际问题的解决问题方式就是在耗时间，送命。

直觉告诉我，是maven依赖有问题，项目的运行环境和别人一样了，为啥就是跑不起来。于是和叶侃对了下依赖包，果然，我的包是比他少的。还没有百分之一百确定原因是不是这个，先解决这个问题试试。

要解决这个问题有两个方法，一个方法是拷贝别人现成的可用的jar包到maven repository里，缺啥放啥。刚开始是这么试的，把缺的包拷进来，结果还有问题，说实话，整个人是有点崩溃的，maven仓库都一样，还报这个问题跑不起来确实很打击人。为了解决这个问题，把这个项目删了推倒重来，重新搭环境好几次，结果还是没有改变。

带着绝望的心情试到一种方法时，项目竟然神奇地跑起来了。相信各位都有这个体验，即被一个陈年老BUG在绝望中被解决的兴奋感冲击。


---


## 解决方法

由于settings.xml配置默认到国外镜像站下载maven依赖包，增加了下载风险，有些jar包虽然表面上躺在那里，仿佛已经被下载的样子，但其实这个包可能是有问题的。由于网络的原因，这个包下载相比国内镜像站出错率更高。所以解决方法就是，把maven repository里的依赖包统统删掉，再在settings.xml里添加国内镜像站地址，这样的依赖包更靠谱。推荐阿里云中央仓库，完美替代oschina。
```
<mirror>  
    <id>alimaven</id>  
    <name>aliyun maven</name>  
    <url>http://maven.aliyun.com/nexus/content/groups/public/</url>  
    <mirrorOf>central</mirrorOf>          
</mirror>
```
想了想为什么第一种解决方法不成功，因为只把缺少的包拷过来了。部分jar包可能看着是下载下来，其实是不完整的。假如说当时把所有的依赖包都删了，相当于把别人仓库复制过来，估计也可行，但是没想到这一方面，所以花了许久的时间解决maven repository的问题。
