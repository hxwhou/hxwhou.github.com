---
layout:     post
title:      "Hadoop分布式集群安装（一）"
subtitle:   " \"前期准备\""
date:       2017-06-23 00:00:00
author:     "hxwhou"
header-img: "img/post-bg-hadoop-1.jpg"
wechat-qrcode: "img/wechat-qrcode.jpg"
catalog: true
tags:
    - hadoop
    - 大数据
    - 分布式
---

>Hadoop2.0已经发布了稳定版本了，增加了很多特性，比如HDFS HA、YARN等。本文将面向初学者从零开始介绍如何在CentOS中安装带有HA（High Availability）特性的Hadoop分布式集群。

## 1. 安装前的准备
首先，进入root用户，在root账号下配置系统环境，并安装全局可用的（即所有用户可用的）jdk。
#### 关闭Linux系统防火墙 
```
＃service iptables stop 或者  /etc/init.d/iptables stop 
* 查看防火墙状态 
service iptables status 
* 查看防火墙开机启动状态 
chkconfig iptables --list 
* 关闭防火墙开机启动 
chkconfig iptables off 
```
![img](/img/in-post/20170623/001.png)

#### 禁用SELINUX
```
＃vi /etc/sysconfig/selinux
```
![img](/img/in-post/20170623/002.png)

#### 设置静态IP地址（optional）
```
＃vi /etc/sysconfig/network-scripts/ifcfg-eth0
```
修改完配置文件后不会立即生效，这时需要重启server（reboot）
或者重启network服务（sudo  service network restart）
#### 修改hostname
```
＃vi /etc/sysconfig/network
```
注意，在hadoop中最好使用全限定名作为主机名，如:hadoop.cruise.com
由于一些特殊的原因，本文并没有遵循这个原则。
#### 绑定IP地址与hostname
```
＃vi /etc/hosts
```
![img](/img/in-post/20170623/003.png)
然后ping主机名验证是否配置成功。
注意，在hadoop集群环境下，必须将所有节点的host都配置到此文件中，这样才能根据hostname与IP的对应关系找到相应的节点。
#### 安装JDK
因为JDK基本是Linux下所有用户的标配，故这里我们将安装全局可用的JDK。
从Oracle官网下载JDK安装包，解压到指定的目录，配置系统环境变量。
```
mkdir /usr/local/java
tar -zxvf jdk-7u80-linux-x64.tar.gz -C /usr/local/java/
```
![img](/img/in-post/20170623/004.png)
以上就是root用户下需要的基础配置，所有hadoop集群的Node都要进行相同的配置。

## 2. 创建hadoop用户
通常我们在安装Hadoop集群的时候都不是在root用户下直接安装，所以接下来我们要在Linux下创建一个新用户（hadoop）。
![img](/img/in-post/20170623/005.png)
切换到hadoop用户下，注意接下来关于hadoop集群的安装都将在此用户下进行。

#### 创建两个目录，分别用于存放压缩包和软件的安装
在这里我创建了一个installPkg目录用来存放hadoop，zookeeper等压缩包，创建一个softwares目录用来安装软件。
![img](/img/in-post/20170623/006.png)
#### 下载安装包并解压
从官网下载相应的安装包，因为我们要使用zookeeper实现Hadoop的HA的故障转移（后面会详细介绍），所以可以看到我们下载了两个安装包（hadoop-2.7.2和zookeeper-3.4.9）.
![img](/img/in-post/20170623/007.png)
将它们解压到指定的目录下，这里是解压到softwares目录下。

## 3. Hadoop集群安装规划
Hadoop2.0中的HDFS HA通常由两个NameNode组成，一个处于active状态，另一个处于standby状态。Active NameNode对外提供服务，而Standby NameNode则不对外提供服务，仅同步active namenode的状态，以便能够在它失败时快速进行切换。

hadoop2.0官方提供了两种HDFS HA的解决方案，一种是NFS，另一种是QJM。这里我们使用简单的QJM。在该方案中，主备NameNode之间通过一组JournalNode同步元数据信息，一条数据只要成功写入多数JournalNode即认为写入成功，通常需要配置奇数个JournalNode。这里还需要配置一个Zookeeper集群，用于ZKFC（DFSZKFailoverController）故障转移，当Active NameNode挂掉了，会自动切换Standby NameNode为Active状态。

在之前的Hadoop2.0发布版中YARN ResourceManager只有一个，存在单点故障，hadoop-2.4.1解决了这个问题，有两个ResourceManager，一个是Active，一个是Standby，状态由zookeeper进行协调。

下面给出一个简单的Hadoop分布式集群安装规划，仅供参考，具体的实践中还是要根据实际情况进行相应的调整。
![img](/img/in-post/20170623/008.png)

**下面我们按照上面的集群安装规划一步步完成集群的搭建。**

## 4. 安装Zookeeper集群
#### 解压zookeeper安装包
从官网下载zookeeper的安装包，解压到指定路径
```
tar -zxvf zookeeper-3.4.9.tar.gz -C ../softwares/
```
注意：这里是在hdp036184节点上进行配置，其余节点上zookeeper安装会经由此节点拷贝并做相应的修改即可。
#### 修改配置文件
![img](/img/in-post/20170623/009.png)
修改dataDir到zookeeper安装路径下，并在最后添加zookeeper集群信息。
![img](/img/in-post/20170623/010.png)
在上面配置的dataDir所对应的目录下新建一个myid文件，并写入值1.
![img](/img/in-post/20170623/011.png)
#### 拷贝到其他节点
将配置好的zookeeper依次拷贝到其他节点上
```
scp -r zookeeper-3.4.9 hdp036185:/home/hadoop/softwares/
scp -r zookeeper-3.4.9 hdp036186:/home/hadoop/softwares/
```
#### 修改其余节点上的myid
分别将hdp036185和hdp036186上的myid修改为2和3.
![img](/img/in-post/20170623/012.png)
至此，zookeeper的配置就完成了，可以启动zookeeper进行验证。
#### 验证zookeeper集群
分别在三个节点上启动zookeeper，查看其状态，其中一个节点是leader，其余两个是follower。
![img](/img/in-post/20170623/013.png)
注意，在启动zookeeper集群之前必须保证相互之间是能够用hostname ping通的，如果ping ［hostname］不通，就需要查看/etc/hosts是否配置正确。
![img](/img/in-post/20170623/014.png)
本文中正确的配置应该如上图所示，到目前为止每个节点的hosts文件中应该都是这样的。

over～

本想着一篇文章搞定，没想到写了这么长还没正式开始Hadoop的安装，实在是觉得有必要分成两篇来进行阐述了。
So，[下一篇](http://www.hxwhou.com/2017/06/24/hadoop-distributed-cluster-installation-2/)我们继续👉👉👉
					
