---
layout:     post
title:      "Hadoop分布式集群安装（一）"
subtitle:   " \"前期准备\""
date:       2017-06-23 00:00:00
author:     "hxwhou"
header-img: "img/post-bg-hadoop-1.jpg"
catalog: true
tags:
    - hadoop
    - 大数据
    - 分布式
---

>Hadoop2.0已经发布了稳定版本了，增加了很多特性，比如HDFS HA、YARN等。本文将面向初学者从零开始介绍如何在CentOS中安装带有HA（High Availability）特性的Hadoop分布式集群。

## 1. 安装前的准备
首先，进入root用户，在root账号下配置系统环境，并安装全局可用的（即所有用户可用的）jdk。\
#### 关闭Linux系统防火墙 \
＃service iptables stop 或者  /etc/init.d/iptables stop \
#查看防火墙状态 \
service iptables status \
#查看防火墙开机启动状态 \
chkconfig iptables --list \
#关闭防火墙开机启动 \
chkconfig iptables off \
![img](img/in-post/20170623/001.png)

#### 禁用SELINUX
# vi /etc/sysconfig/selinux
![img](img/in-post/20170623/002.png)
