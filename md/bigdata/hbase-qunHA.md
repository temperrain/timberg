# HBase完全分布式集群搭建

## 基础环境

系统/软件 | 版本号
-|:-:
HBase | 2.1.2
Linux系统 | CentOS 7

## 集群规划

主机名|IP |安装的软件|运行的进程
-|:-|:-|:-
alphasta01|192.168.23.191|hbase|HMaster
alphasta02|192.168.23.192|hbase|HRegionServer
alphasta03|192.168.23.193|hbase|HRegionServer


## 搭建hadoop完全分布式集群
基于：[hadoop 3节点完全分布式集群](https://github.com/temperrain/timberg/blob/master/md/bigdata/hadoop-qun3HA.md)

## 部署HBase   
1. hbase-env.sh 文件
> tar -zxvf hbase-2.1.2-bin.tar.gz                          解压缩HBase压缩包      

> JAVA_HOME=export JAVA_HOME=/usr/lib/jdk1.8.0_191          在hbase-env.sh文件中，添加java环境变量

> export  HBASE_MANAGES_ZK=false                            关闭HBase自带的Zookeeper,使用Zookeeper集群

2. hbase-site.xml 文件

```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
　　<property> 
　　　　<name>hbase.rootdir</name> 
　　　　<value>hdfs://alphasta01:8020/hbase</value> 
　　</property> 
　　<property> 
　　　　<name>hbase.cluster.distributed</name> 
　　　　<value>true</value> 
　　</property> 
　　<property> 
　　　　<name>hbase.zookeeper.quorum</name> 
　　　　<value>alphasta01,alphasta02,alphasta03</value> 
　　</property> 
　　<property> 
　　　　<name>hbase.zookeeper.property.dataDir</name> 
　　　　<value>/opt/modules/hbase-2.1.2/tmp/zk/data</value> 
　　</property>
</configuration>
```

3. regionservers 文件

> vi   regionservers           编辑文件,加入 alphasta02 alphasta03 一个一行

4. 将HBase复制分发到 alphasta02、alphasta03的机器上

> [root@alphasta01 modules]$ scp -r hbase-2.1.2 root@alphasta02:/opt/modules

> [root@alphasta01 modules]$ scp -r hbase-2.1.2 root@alphasta03:/opt/modules 


## 启动HBase

> 节点 alphasta01

> bin/start-hbase.sh 

> 在alphasta01、alphasta02、alphasta03任意一台机器使用bin/hbase shell 进入shell环境，使用命令version等，查看hbase信息

## 查看各节点进程

于集群规划中的一致
