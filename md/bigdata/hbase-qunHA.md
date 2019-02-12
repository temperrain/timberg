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

## HBase基本操作

1. 在任意一个节点，进入shell环境,bin/hbase shell
- 创建表
```
create 'student','Sname','Ssex','Sage','Sdept','course'
create 'teacher',{NAME=>'username',VERSIONS=>5} // 创建表示指定保存的版本数
```

- 查看表详情

> describe 'student' 

- 显示所有表

> list 

- 插入数据

```
put 'student','95001','Sname','LiYing'
put 'student','95001','Ssex','Male'
put 'student','95001','course:math','80'
put 'student','95001','course:english','90'

put 'student','95002','Sname','ZhangYiDa'
put 'student','95002','Ssex','Femal'
put 'student','95002','course:math','90'
put 'student','95002','course:english','70'
```

- 查询数据（有两个命令）
    - get命令，用于查看表的某一行数据；
    - scan命令用于查看某个表的全部数据

```
get 'student','95001'
get 'student','95001','course'
get 'student','95001','course:math'

scan 'student'
```

- 删除数据（delete和deleteall）
    - delete用于删除一个数据，是put的反向操作
    - deleteall操作用于删除一行数据

- 修改数据

_在添加数据时，HBase会自动为添加的数据添加一个时间戳，故在需要修改数据时，只需直接添加数据，HBase即会生成一个新的版本，从而完成“改”操作，旧的版本依旧保留，系统会定时回收垃圾数据，只留下最新的几个版本，保存的版本数可以在创建表的时候指定。下面是一个操作的例子：_

```
hbase(main):034:0> get 'student','95001'
COLUMN                             CELL                                                                                               
 Sname:                            timestamp=1537497681798, value=LiYing                                                              
 Ssex:                             timestamp=1537497682400, value=Male                                                                
 course:english                    timestamp=1537497872225, value=90                                                                  
 course:math                       timestamp=1537497681859, value=80                                                                  
4 row(s) in 0.0310 seconds

hbase(main):035:0> put 'student','95001','course:english','100'
0 row(s) in 0.0130 seconds

hbase(main):036:0> get 'student','95001'
COLUMN                             CELL                                                                                               
 Sname:                            timestamp=1537497681798, value=LiYing                                                              
 Ssex:                             timestamp=1537497682400, value=Male                                                                
 course:english                    timestamp=1537498062541, value=100                                                                 
 course:math                       timestamp=1537497681859, value=80                                                                  
4 row(s) in 0.0130 seconds
```

- 删除表（有两步）
    - 第一步先让该表不可用
    - 第二步删除表
```
disable 'student'
drop 'student'
```

- 查看历史的表

```
create 'teacher',{NAME=>'username',VERSIONS=>5}

put 'teacher','91001','username','Mary'
put 'teacher','91001','username','Mary1'
put 'teacher','91001','username','Mary2'
put 'teacher','91001','username','Mary3'
put 'teacher','91001','username','Mary4'  
put 'teacher','91001','username','Mary5'

get 'teacher','91001',{COLUMN=>'username',VERSIONS=>5}

hbase(main):064:0> get 'teacher','91001',{COLUMN=>'username',VERSIONS=>5}
COLUMN                             CELL                                                                                               
 username:                         timestamp=1537498459746, value=Mary5                                                               
 username:                         timestamp=1537498455244, value=Mary4                                                               
 username:                         timestamp=1537498455193, value=Mary3                                                               
 username:                         timestamp=1537498455174, value=Mary2                                                               
 username:                         timestamp=1537498455149, value=Mary1                                                               
5 row(s) in 0.0110 seconds
```

- 退出HBase

> exit


