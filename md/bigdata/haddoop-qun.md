## 一、软件说明

软件名称 | 版本号
-|:-:
JDK | 1.8 
hadoop | 2.9.2
zookeeper | 3.4.13

## 二、集群规划

主机名 | IP | 安装的软件 | 运行的进程
-|:-|:-|:-
alphasta01 | 192.168.16.90 | JDK、hadoop| NameNode、DFSZKFailoverController
alphasta02 | 192.168.16.91 | JDK、hadoop| NameNode、DFSZKFailoverController
alphasta03 | 192.168.16.92 | JDK、hadoop| ResourceManager
alphasta04 | 192.168.16.93 | JDK、hadoop、zookeeper| DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta05 | 192.168.16.94 | JDK、hadoop、zookeeper| DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta06 | 192.168.16.95 | JDK、hadoop、zookeeper| DataNode、NodeManager、JournalNode、QuorumPeerMain
