---
layout:    post
title:      "入门hadoop集群搭建"
subtitle:   "To build hadoop cluster"
date:       2017-09-25 12:00:00
author:     "scyhssm"
header-img: "img/hadoop.jpg"
catalog:    true
tags:
    - hadoop
---

>对于入门者来说，搭建hadoop集群不是一件轻松的事，希望我的文档能有所帮助


# 简介

首先了解下HADOOP集群：HDFS集群和YARN集群，逻辑上分离但是物理上在一起。

HDFS集群：负责海量数据的存储，集群角色主要有NameNode／DataNode。  
YARN集群：负责海量数据运算时的资源调度，角色主要有ResourceManager／NodeManager。  
mapreduce：一个应用程序开发包。  

以3节点搭建集群系统,准备：  
1.VMware  
2.centos6.5  
3.jdk1.8  
4.hadoop2.8  

为了化简搭建，我们先将一台虚拟机配置完成后，再用克隆的办法配置另两台虚拟机。


# 步骤

1.创建用户HADOOP  
2.配置网络环境  
3.安装java  
4.hadoop下载安装  
5.复制虚拟机  
6.ssh免密登陆  
7.检查运行环境  


# 1.创建用户HADOOP

假设我们已经会使用虚拟机软件和虚拟机，这里我们推荐使用VMware，VirtualBox这款工具相比VMware体验上还是差了不少。下文我们将会说到为什么用VMware。

创建HADOOP用户
```
groupadd HADOOP    #添加一个HADOOP组
useradd HADOOP -g HADOOP    #把HADOOP用户加到HADOOP组中
vi /etc/sudoers    #编辑sudoers文件
```
HADOOP ALL=(ALL) ALL，在root ALL=(ALL) ALL下加入这行

# 2.配置网络环境

hosts文件中需要固定的IP地址，我们不能用DHCP的方式每次获取不同的IP地址，因此采用静态的方式。

另外配置网络时，建议了解下虚拟机网络模式，NAT，Host-Only，桥接，我们为了能够访问外网及让内网间的虚拟机能够相互访问，采用NAT。

之前配置网络环境用的是VirtualBox，很奇怪的一点是明明用的是NAT，但是虚拟机之间不能ping通，改用VMware，注册码很容易找到。

```
dhclient    #利用DHCP获取IP地址，子网掩码，网关
vi /etc/sysconfig/network-scripts/ifcfg-eth0    #配置网络
```
这里由于dhclient，因此不需要配置/etc/resolv.conf。

```
ONBOOT=yes    #系统启动激活网卡
NM_CONTROLLED=no    #禁用network manager
BOOTPROTO=static    #采用静态方法
IPADDR=192.168.158.101    #IP地址
NETMASK=255.255.255.0    #子网掩码
GATEWAY=192.168.158.2    #网关，不要用1
```
这是网络配置

```
vi /etc/hosts
```
在hosts下加入这三行，分别代表3台机子的IP地址，主机名，另两台等会克隆配置。
```
192.168.158.101    hdp-01
192.168.158.102    hdp-02
192.168.158.103    hdp-03
```

配置完后记得重启服务
```
service network restart
```
ping回环，ping外网，ping主机，ping本机IP测试网络，ifconfig看看网络配置是否生效。

# 3.安装java
centos6.5默认安装openjdk，我的机子上版本为1.7，网上找了下，openjdk不影响hadoop使用，官方推荐sunjdk，有需要的建议安装sunjdk。

oracle官网下载jdk1.8，笔者用解压文件配置的方式，可以在centos中下载，也可以在主机上下载后通过ssh传输到虚拟机。

下载位置为/home/HADOOP/Downloads。

```
mkdir /usr/java
tar -zxvf /home/HADOOP/Downloads/XXXXXXXXX.tar.gz /usr/java
```

解压后就是写配置文件了

```
vi /etc/profile    #打开配置文件并启用编辑
```

在最后加入如下内容

```
export JAVA_HOME=/usr/java/jdkXXXXXXX
export CLASSPATH=.:$JAVA_HOME/jre/lib/rt.jar:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$PATH:$JAVA_HOME/bin
```

使配置生效

```
source /etc/profile
```

这里不采用暴力卸载openjdk方式使sunjdk的配置生效，因此需要在版本管理工具alternatives切换sunjdk的版本。

```
alternatives --config java
```

选择sunjdk的版本数字，应用后查看jdk是否已经切换。

```
java -version
```

# 4.hadoop下载安装

下载hadoop前注意版本所需的jdk，官网可以查到，最新的3.0-alpha要求最低jdk版本从7升级到了8，这里安装选择hadoop2.8。

这里依旧下载到HADOOP的Downloads目录下，解压文件。

```
tar -zxvf /home/HADOOP/Downloads/hadoop-XXXXX.tar.gz /usr
cd /usr
mv hadoop-XXXXX hadoop    #重命名文件夹
```

配置文件

```
cd /usr/hadoop/etc/hadoop
vi hadoop-env.sh
```
hadoop-env.sh
```
export JAVA_HOME=/usr/java/jdk1.8.XXXX
```

core-site.xml
```
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://hdp-01:9000</value>
    </property>
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/home/HADOOP/apps/hadoop-2.6.1/tmp</value>
    </property>
</configuration>
```

hdfs-site.xml
```
<configuration>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/usr/hadoop/data/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/usr/hadoop/data/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>3</value>
    </property>
    <property>
        <name>dfs.secondary.http.address</name>
        <value>hdp-01:50090</value>
    </property>
</configuration>
```

mapred-site.xml
```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
</configuration>
```

yarn-site.xml
```
<configuration>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>hdp-01</value>
    </property>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
</configuration>
```

slaves
```
hdp-01
hdp-02
hdp-03
```
记得删除slaves文件下的localhost这行。

# 5.复制虚拟机

关闭虚拟机->创建完整克隆->修改网络配置。

在前4个步骤中，我们已经将需要用到的工具基本搭建配置完成，这时候复制虚拟机修改一些配置即可快速制作两台虚拟机,这里以修改一台为例。

1.修改主机名

```
hostname hdp-02
vi /etc/sysconfig/network
HOSTNAME=hdp-02
```

2.修改网络配置

我们的网络完全复制过来，因mac地址变化，VMware为我们多加一个网卡eth1，eth0被禁用。

先用dhclient获取一个地址。
```
dhclient
vi /etc/udev/rules.d/70-persistent-net.rules
```
将eth1改成eth0，之前eth0那条记录删除。

```
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```
```
HWADDR=XXXXXXXXXXXXX    #这里是新获得的eth1的mac地址
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.158.102
NETMASK=255.255.255.0
GATEWAY=192.168.158.2
```
网络配置完成，重启系统应用配置，这里仅仅重启服务是不够的，涉及到硬件mac地址修改。

重启，并将余下一台虚拟机配置完，相互之间ping一下。
```
ping 127.0.0.1
ping 192.168.158.101
ping 192.168.158.102
ping 192.168.158.103
ping 192.168.158.2
ping baidu.com
ping hdp-01
ping hdp-02
ping hdp-03
```
测试无误，说明网络能够连通。

# 6.ssh免密登陆

我们创建HADOOP用户是要用它来运行集群的，因此要切换到HADOOP用户下，让HADOOP用户免密登陆。

hdp-01操作
```
$ cd ~		#进入hadoop用户目录
$ ssh-keygen -t rsa -P ""   
#这是生成ssh密码的命令，-t 参数表示生成算法，有rsa和dsa两种；-P表示使用的密码，这里使用""空字符串表示无密码。
#回车后，会提示三次输入信息，我们直接回车即可。这样就在~/.ssh目录下生成了几个东西
$ cd ~/.ssh
$ cat id_rsa.pub >> authorized_keys #这个命令将id_rsa.pub的内容追加到了authorized_keys的内容后面
```

hdp-02,hdp-03也做相同的操作，获得公钥后将其添加到第一台机子的authorized_keys下。
```
scp id_rsa.pub hdp-01:/home/hadoop/.ssh/id_rsa.pub.node2    #第三台相同
```

回到hdp-01的～/.ssh下
```
cat id_rsa.pub.node2 >> authorized_keys
cat id_rsa.pub.node3 >> authorized_keys
```

将authorized_keys复制到hdp-02，hdp-03下
```
scp authorized_keys hdp-02:/home/hadoop/.ssh/
scp authorized_keys hdp-03:/home/hadoop/.ssh/
```

这里有一个要点，即设置权限，权限设置也有讲究，万一设置太高可能会被怀疑修改，依旧会验证。
```
sudo chmod 644 ~/.ssh/authorized_keys
sudo chmod 700 ~/.ssh
```

现在就可以无密码访问了，第一次还是要输入yes，添加到白名单。

# 7.检查运行环境

防火墙需要关闭，临时关闭下次打开防火墙还开着。要么永久关闭防火墙，要么添加一些新的防火墙规则，要么每次开集群都关次防火墙。

```
cd /usr/hadoop/bin/
hadoop namenode -format    #格式化namenode
start-all.sh    #启动集群，稍微等一会集群打开
start-yarn.sh    #启动yarn
jps    #观察进程
hadoop dfsadmin -report    #查看集群状态，是否3个节点
```
也可以火狐进到hdp-01:50090查看集群状态。

需要注意一个问题，因此刚刚搭建集群可能出现各种各样的问题，经常格式化namenode，clusterID会不同，导致集群启动时clusterID不同的节点无法被启用。

```
cd /usr/hadoop/data/data/current
vi VERSION    #观察这里的clusterID三台机子是否相同，不同则改成hdp-01的
```

群集搭建完成，如有遗漏或者错误，欢迎联系我指出。
