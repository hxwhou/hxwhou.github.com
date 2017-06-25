---
layout:     post
title:      "Hadoop分布式集群安装"
subtitle:   " \"进入正题\""
date:       2017-06-24 00:00:00
author:     "hxwhou"
header-img: "img/post-bg-hadoop-2.jpg"
header-mask: 0.1
wechat-qrcode: "img/wechat-qrcode.jpg"
catalog: true
tags:
    - hadoop
    - 大数据
    - 分布式
---

>Hadoop2.0已经发布了稳定版本了，增加了很多特性，比如HDFS HA、YARN等。本文将面向初学者从零开始介绍如何在CentOS中安装带有HA（High Availability）特性的Hadoop分布式集群。

在上一篇文章中，我们主要介绍了搭建Hadoop集群的相关准备工作，在此基础上对整个集群进行了简单的规划，并完成了Zookeeper集群的安装，以便实现Hadoop集群的HA特性。

废话不多说了，下面接着[上一篇](http://www.hxwhou.com/2017/06/23/hadoop-distributed-cluster-installation-1/)没写完的继续👉

## 1.配置免密码登录

**不是都准备完了吗？还准备啥来着！！！**

别急，且听我娓娓道来。有了上一篇所做的准备，这里的准备其实九牛一毛了，主要是配置集群间的ssh免密码登录，本来也可以放到前一篇进行介绍，但是考虑到关联性问题，我还是决定在此展开。毕竟安装Zookeeper集群是不需要配置免密码登录的。

有关ssh的介绍这里就不赘述了，不了解的童鞋可以自行Google解决。

参考上一篇中对整个集群的规划，因为hdp131065和hdp131066都将作为NameNode节点来启动HDFS集群，所以需要配置这两个节点到其余DataNode的免密码登录。
#### 生成密钥对
```
ssh-keygen -t rsa
```
![img](/img/in-post/20170624/001.png)
#### 拷贝公钥
* 将hdp131065的公钥拷贝到其他节点，包括自己：

```
ssh-copy-id hdp131065
ssh-copy-id hdp131067
ssh-copy-id hdp036184
ssh-copy-id hdp036185
ssh-copy-id hdp036186
```
* 在hdp131067上生成密钥对，将公钥拷贝到如下节点

```
ssh-copy-id hdp131067
ssh-copy-id hdp131065
ssh-copy-id hdp036184
ssh-copy-id hdp036185
ssh-copy-id hdp036186
```
因为YARN的ResourceManager将被部署到hdp131107和hdp131108两个节点上，所以同样需要配置这两个节点到其余相关节点的免密码登录。
* 在hdp131107上生成密钥对，将公钥拷贝到如下节点

```
ssh-copy-id hdp131107
ssh-copy-id hdp131108
ssh-copy-id hdp036184
ssh-copy-id hdp036185
ssh-copy-id hdp036186
```
* 在hdp131108上生成密钥对，将公钥拷贝到如下节点

```
ssh-copy-id hdp131108
ssh-copy-id hdp131107
ssh-copy-id hdp036184
ssh-copy-id hdp036185
ssh-copy-id hdp036186
```
为了方便起见，也可以将所有节点的公钥都拷贝到.ssh/authorized_keys
中，然后分发给所有节点，这样就可以实现所有节点到其余所有节点彼此之间的免密码登录。
是不是觉得我有点啰哩啰嗦了，非常抱歉，这真的不是我想要的，我只是想让更多的人能够理解，所以才不惜一切的赘述。

***好了，闲话少扯，下面正式进入Hadoop集群搭建！***
## 2.安装配置Hadoop集群

下面的配置在hdp131065节点上完成，然后分发到其余节点上。
#### 下载并解压hadoop安装包
从官网下载Hadoop2.7.2发行版，解压到指定的目录。
![img](/img/in-post/20170624/002.png)
#### 配置HDFS
Hadoop相关的配置文件都在$HADOOP\_HOME/etc/hadoop目录下。

* 配置Hadoop系统环境变量（optional）
```
vim /etc/profile
export HADOOP_HOME=/home/hadoop/softwares/hadoop-2.7.2
export PATH=$PATH:$HADOOP_HOME/bin
```
* 修改hadoop-env.sh
![img](/img/in-post/20170624/003.png)
* 修改core-site.xml
![img](/img/in-post/20170624/004.png)
* 修改hdfs-site.xml

```
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
  <property>
    <name>dfs.permissions</name>
    <value>false</value>
  </property>  
  <property>
    <name>dfs.nameservices</name>
    <value>ns1</value>
  </property>	
  <property>
    <name>dfs.ha.namenodes.ns1</name>
    <value>nn1,nn2</value>
  </property>
  <!--namenode1 conf-->
  <property>
    <name>dfs.namenode.rpc-address.ns1.nn1</name>
    <value>hdp131065:9000</value>
  </property>	
  <property>
    <name>dfs.namenode.http-address.ns1.nn1</name>
    <value>hdp131065:50070</value>
  </property>
  <!--namenode2 conf-->
  <property>
    <name>dfs.namenode.rpc-address.ns1.nn2</name>
    <value>hdp131067:9000</value>
  </property>	
  <property>
    <name>dfs.namenode.http-address.ns1.nn2</name>
    <value>hdp131067:50070</value>
  </property>
  <!--namenode edits metadata location-->
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
 <value>qjournal://hdp036184:8485;hdp036185:8485;hdp036186:8485/ns1</value>
  </property>
  <property>
    <name>dfs.journalnode.edits.dir</name>
    <value>/home/hadoop/softwares/hadoop-2.7.2/journaldata</value>
  </property>
  <property>
    <name>dfs.ha.automatic-failover.enabled</name>
    <value>true</value>
  </property> 
  <property>
    <name>dfs.client.failover.proxy.provider.ns1</name>
 <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
  </property>	
  <property>
    <name>dfs.ha.fencing.methods</name>
    <value>
      sshfence
      shell(/bin/true)
    </value>
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.private-key-files</name>
    <value>/home/hadoop/.ssh/id_rsa</value>
  </property>
  <property>
    <name>dfs.ha.fencing.ssh.connect-timeout</name>
    <value>30000</value>
  </property>  
</configuration>
```

#### 配置MapReduce（mapred-site.xml）
![img](/img/in-post/20170624/005.png)
#### 配置YARN
修改yarn-site.xml文件

```
<configuration>
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.enabled</name>
    <value>true</value>
  </property>
  <property>
    <name>yarn.resourcemanager.cluster-id</name>
    <value>yrc</value>
  </property>
  <property>
    <name>yarn.resourcemanager.ha.rm-ids</name>
    <value>rm1,rm2</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm1</name>
    <value>hdp131107</value>
  </property>
  <property>
    <name>yarn.resourcemanager.hostname.rm2</name>
    <value>hdp131108</value>
  </property>
  <property>
    <name>yarn.resourcemanager.zk-address</name>
    <value>hdp036184:2181,hdp036185:2181,hdp036186:2181</value>
  </property>
  <property>
    <description>The address for the web proxy as HOST:PORT, if this is not
    given then the proxy will run as part of the RM</description>
    <name>yarn.web-proxy.address</name>
    <value>hdp131107:8889</value>
  </property>
  <property>
    <name>yarn.log-aggregation-enable</name>
    <value>true</value>
  </property>  
  <property>
    <name>yarn.log-aggregation.retain-seconds</name>
    <value>604800</value>
  </property>   
  <property>
    <description>Amount of physical memory, in MB, that can be allocated 
    for containers.</description>
    <name>yarn.nodemanager.resource.memory-mb</name>
    <value>8192</value>
  </property> 
  <property>
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>8</value>
  </property>
</configuration>
```
#### 修改slaves
slaves是指定子节点的位置，因为要在hdp131065上启动HDFS、在hdp131107启动YARN，所以hdp131065上的slaves文件指定的是DataNode的位置，hdp131107上的slaves文件指定的是Nodemanager的位置。因为我们的DataNode和Nodemanager是在相同的节点上，故slaves中的内容如下：
![img](/img/in-post/20170624/006.png)

#### 拷贝到其他节点
![img](/img/in-post/20170624/007.png)
至此，hadoop集群已经配置完成，接下来启动集群。

## 3.启动集群

#### 启动zookeeper集群
![img](/img/in-post/20170624/008.png)
#### 启动journalnode
分别在hdp036184，hdp036185，hdp036186节点上启动journalnode
![img](/img/in-post/20170624/009.png)
#### 格式化HDFS
在hdp131065上执行命令: `hdfs namenode -format`
![img](/img/in-post/20170624/010.png)
格式化后会在core-site.xml中配置的hadoop.tmp.dir路径下生成NameNode元数据文件，这里我配置的是/home/hadoop/softwares/hadoop-2.7.2/data/tmp。为了实现NameNode的HA就需要在hdp131067上也启动NameNode，此时就需要将此目录下的metadata拷贝到hdp131067的对应目录下。这里有两种方式：1）使用scp直接拷贝 2）通过hdfs指令拷贝，建议使用第二种方式。

第二种方式需要在未格式化的NameNode节点上执行`hdfs namenode -bootstrapStandby`指令来拷贝元数据, 但执行此命令前需要先在已格式化的NameNode上启动NameNode进程，因为此命令需要从active NameNode上fetch元数据。
![img](/img/in-post/20170624/011.png)
然后在hdp131067上运行`hdfs namenode -bootstrapStandby`来同步元数据。
![img](/img/in-post/20170624/012.png)
紧接着在此节点上启动NameNode进程。
#### 格式化ZKFC
在hdp131065上执行命令`hdfs zkfc -formatZK`
![img](/img/in-post/20170624/013.png)
我们会发现，在格式化成功后会在ZK上创建一个hadoop-ha/ns1的znode，其用来保存故障转移系统的数据。
#### 启动zkfc进程
在两个NameNode节点上分别执行以下命令启动DFSZKFailoverController进程。
![img](/img/in-post/20170624/014.png)
#### 启动DataNode
在三个DataNode节点上分别执行以下命令启动DataNode进程
![img](/img/in-post/20170624/015.png)
#### 启动YARN
注意：是在hdp131107上执行start-yarn.sh，把namenode和resourcemanager分开是因为性能问题，因为他们都要占用大量资源，所以把他们分开了，他们分开了就要分别在不同的机器上启动。
`sbin/start-yarn.sh`
#### 启动JobHistoryServer
在hdp131108上启动JobHistoryServer
![img](/img/in-post/20170624/016.png)
#### 启动WebAppProxyServer
![img](/img/in-post/20170624/017.png)
到此，hadoop-2.7.2配置完毕，可以通过浏览器访问hdfs:
![img](/img/in-post/20170624/018.png)

## 4.验证HDFS HA

* 首先向HDFS上上传一些文件。
```
hadoop fs -mkdir /user/hadoop/etc
hadoop fs -put etc/hadoop/* /user/hadoop/etc
hadoop fs -ls /user/hadoop
```
* 然后再kill掉active的NameNode
```
kill -9 <pid of NN>
```
* 通过浏览器访问：http://hdp131067:50070

  * NameNode 'hdp131067:9000' (active)
  * 这个时候hdp131067上的NameNode变成了active.

* 再执行命令：`hadoop fs -ls /user/hadoop`
![img](/img/in-post/20170624/019.png)
刚才上传的文件依然存在！！！
* 手动启动那个挂掉的NameNode.
```
sbin/hadoop-daemon.sh start namenode
```
* 通过浏览器访问：http://hdp131065:50070
  * NameNode 'hdp131065:9000' (standby)

## 5.验证YARN

运行一下hadoop提供的例子中的QuasiMonteCarlo程序：
![img](/img/in-post/20170624/020.png)
打开浏览器查看YARN集群的Web UI，可以看到Job运行成功。
![img](/img/in-post/20170624/021.png)
Over，大功告成～


**附：测试集群工作状态的一些指令 ：**

```
bin/hdfs dfsadmin -report	查看hdfs的各节点状态信息
bin/hdfs haadmin -getServiceState nn1	获取一个namenode节点的HA状态
sbin/hadoop-daemon.sh start namenode  单独启动一个namenode进程
sbin/hadoop-daemon.sh start zkfc   单独启动一个zkfc进程
bin/yarn rmadmin -getServiceState rm2 获取一个resourcemanager节点的HA状态
```

