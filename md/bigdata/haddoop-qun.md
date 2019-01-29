# hadoop 高可用分布式集群搭建

## 基础环境

系统/软件 | 版本号
-|:-:
JDK | 1.8 
hadoop | 2.9.2
zookeeper | 3.4.13
Linux系统 | CentOS 7

## 集群规划

主机名|IP |安装的软件|运行的进程
-|:-|:-|:-
alphasta01|192.168.23.181|JDK、hadoop|NameNode、DFSZKFailoverController
alphasta02|192.168.23.182|JDK、hadoop|NameNode、DFSZKFailoverController
alphasta03|192.168.23.183|JDK、hadoop|ResourceManager
alphasta04|192.168.23.184|JDK、hadoop|ResourceManager
alphasta05|192.168.23.185|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta06|192.168.23.186|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta07|192.168.23.187|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain

## 设置系统环境

1. #### 设置机器名

> 用户root

> 根据集群规划设置机器名

> 重启后生效

```
hostname test                              临时生效
hostnamectl set-hostname centos7           永久生效，需重启
```
2. #### 设置IP地址

> 用户 root

> 配置文件 /etc/sysconfig/network-scripts/ifcfg-enp0s3

> 重启网络 service network restart

> 查看IP是否生效 ifconfig 如果需要设置后远程连接，建议重启机器

```
TYPE="Ethernet"                                 以太网类型
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="static"                              使用静态IP，而不是用DHCP分配IP
DEFROUTE="yes"
NAME="enp0s3"                                   网络连接名称
UUID="b6591044-1086-4cc4-af48-876b26c0e739"
DEVICE="enp0s3"                                 对应第一张网卡
ONBOOT="yes"                                    是否启动运行
IPADDR=192.168.23.181                           指定本机IP地址
GATEWAY=192.168.16.1                            指定网关
NETMASK=255.255.255.0                            
DNS1=222.222.222.222                            DNS地址，配置后同步到 /etc/resolv.conf
```

3. #### 设置HOST映射文件

> 用户root

> 配置文件 /etc/hosts

> 根据集群规划添加如下内容，并使用 #ping alphasta01检测

```   
192.168.23.181     alphasta01
192.168.23.182     alphasta02
192.168.23.183     alphasta03
192.168.23.184     alphasta04
192.168.23.185     alphasta05
192.168.23.186     alphasta06
192.168.23.187     alphasta07
```
4. #### 关闭防火墙和SELinux 需重启机器生效
 
> CentOS 7.0默认使用的是firewall作为防火墙，使用iptables必须重新设置一下

- 直接关闭防火墙
```
systemctl  stop     firewalld.service        停止firewall
systemctl  disable  firewalld.service        禁止firewall开机启动

systemctl  restart   iptables.service        重启防火墙使配置生效
systemctl  enable    iptables.service        设置防火墙开机启动
```

- ~~如果使用service iptables 需要设置一下~~

```
yum -y install iptables-services                           切换到root用户 安装

service iptables [start|stop|restart|save|status]          之后就可以使用了

vi /etc/sysconfig/iptables                                 修改防火墙配置，如添加端口9908
```    

- 关闭SELinux

> 用户root

> 配置文件 /etc/selinux/config

> 设置 SELINUX=disable

5. #### SSH免密码登录
    
- 分别在所有节点上执行 ssh-keygen 命令，一路回车，生成秘钥即可 
- 分别在所有结点上执行以下命令，把本机的公钥追到其他节点的 .ssh/authorized_keys 里，期间可能需要输入一次yes

```
ssh-copy-id alphasta01
ssh-copy-id alphasta02
ssh-copy-id alphasta03
ssh-copy-id alphasta04
ssh-copy-id alphasta05
ssh-copy-id alphasta06
ssh-copy-id alphasta07
```
- 验证是否免密成功 ssh alphasta01 

***
##### 集群搭建方式采用先安装配置好一台样板机，然后再拷贝到其它设备的方式进行，故需按以下顺序安装分发
*** 

## 配置样板机

1. #### 添加hadoop用户/用户组
```
groupadd -g 1000 hadoop                    指定用户组ID 1000 （系统用户组ID不低于500）如果ID已存在，就改为其它或去掉 -g 1000
useradd -u 2000 -g hadoop hadoop           指定用户UID 2000 及所属的主组
passwd hadoop                              指定用户密码
```

2. #### 如果用到sudo命令，需把hadoop用户加入到sudoers中

```
chmod u+w /etc/sudoers                     先修改该配置文件的权限
vi /etc/sudoers                            打开该配置文件，找到 root ALL=(ALL) ALL 在其下边添加 hadoop ALL=（ALL）
```

3. #### 统一目录结构（所属组和用户均为hadoop）

```
mkdir /alphastaApp                          上传目录
mkdir /opt/modules                          软件安装目录
chown -R hadoop:hadoop /opt/modules         将该目录的用户组设定为 hadoop
```

4. #### 安装和配置JDK

- JDK安装包（jdk-8u191-linux-x64.tar.gz）               上传 到 /alphastaApp目录下
- cd /alphastaApp                                      验证是否上传成功
- tar -zxvf jdk-8u191-linux-x64.tar.gz -C /usr/lib     将JDK解压到 /usr/lib目录下
- cd /usr/lib      ls                                  验证是否解压成功
- vi /etc/profile                                      打开配置文件，按下 shift + G 快速到文件末尾 按o键新增一行
- 在文件末尾添加如下内容，:wq保存退出（shift +ZZ）
```
export JAVA_HOME=/usr/lib/jdk-8u191-linux-x64
export PATH=$PATH:$JAVA_HOME/bin
```
- source /etc/profile                                   使配置文件生效
- Java -version                                         在任意目录输入此命令验证是否安装成功

- 分发配置好的JDK
    - 如果profile文件已经配置，各个节点不一样，就选择各自配置profile即可

```
scp -r /usr/lib/jdk1.8.0_191 root@alphasta07:/usr/lib/
scp -r /etc/profile root@alphasta07:/etc/
```

- 分发到新节点后，需要生效 source /etc/profile 配置文件

5. #### 安装和配置 hadoop

- 上传hadoop 安装包到 /alphastaApp
- tar -zxvf hadoop-2.9.2.tar.gz -C /opt/modules/      解压到/opt/modules目录下
- cd /opt/modules   ls                                切换到/opt/modules目录下查看是否解压成功
 
> 安装完hadoop之后再配置HDFS，（hadoop2.0所有的配置文件都在$HADOOP_HOME/etc/hadoop目录下）

- 将hadoop 添加到环境变量
```
vi /etc/profile                                        shift + G 跳转到文件末尾，添加如下内容
export HADOOP_HOME=/opt/modules/hadoop-2.9.2
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/bin
```

- 配置hadoop，共7个文件(core-site.xml、hdfs-site.xml、mapreduce-site.xml、yarn-site.xml、hadoop-env.sh、mapred-env.sh、yarn-env.sh)


> cd $HADOOP_HOME/etc/hadoop  切换到hadoop 配置文件目录

> 第一个文件：core-site.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://alphasta</value>
        </property>

        <!-- 指定hadoop临时目录 需创建-->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/opt/modules/hadoop-2.9.2/tmp</value>
        </property>

        <!-- 指定zookeeper地址 -->
        <property>
                <name>ha.zookeeper.quorum</name>
                <value>alphasta05:2181,alphasta06:2181,alphasta07:2181</value>
        </property>

        <property>
                <name>ha.zookeeper.session-timeout.ms</name>
                <value>3000</value>
        </property>

        <property>
                <name>ipc.client.connect.max.retries</name>
                <value>100</value>
        </property>

        <property>
                <name>ipc.client.connect.retry.interval</name>
                <value>10000</value>
        </property>

</configuration>
```

> 第二个文件：hdfs-site.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

	<!--指定hdfs的nameservice为alphasta，需要和core-site.xml中的保持一致 -->
	<property>
		<name>dfs.nameservices</name>
		<value>alphasta</value>
	</property>

	<!-- alphasta下面有两个NameNode，分别是nn1，nn2 -->
	<property>
		<name>dfs.ha.namenodes.alphasta</name>
		<value>nn1,nn2</value>
	</property>

	<!-- nn1的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.alphasta.nn1</name>
		<value>alphasta01:8020</value>
	</property>

	<!-- nn2的RPC通信地址 -->
	<property>
		<name>dfs.namenode.rpc-address.alphasta.nn2</name>
		<value>alphasta02:8020</value>
	</property>

	<!-- nn1的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.alphasta.nn1</name>
		<value>alphasta01:50070</value>
	</property>

	<!-- nn2的http通信地址 -->
	<property>
		<name>dfs.namenode.http-address.alphasta.nn2</name>
		<value>alphasta02:50070</value>
	</property>

	<!-- 指定NameNode的元数据在JournalNode上的存放位置 -->
	<property>
		<name>dfs.namenode.shared.edits.dir</name>
		<value>qjournal://alphasta05:8485;alphasta06:8485;alphasta07:8485/alphasta</value>
	</property>

	<!-- 指定JournalNode在本地磁盘存放数据的位置 -->
	<property>
		<name>dfs.journalnode.edits.dir</name>
		<value>/opt/modules/hadoop-2.9.2/tmp/journal</value>
	</property>

	<!-- 开启NameNode失败自动切换 -->
	<property>
		<name>dfs.ha.automatic-failover.enabled</name>
		<value>true</value>
	</property>

	<!-- 配置失败自动切换实现方式 -->
	<property>
		<name>dfs.client.failover.proxy.provider.alphasta</name>
		<value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>

	<!-- 配置隔离机制，多个机制用换行分割，即每个机制暂用一行 -->
	<property>
		<name>dfs.ha.fencing.methods</name>
		<value>
		sshfence
		shell(/bin/true)
		</value>
	</property>

	<!-- 使用sshfence隔离机制时需要ssh免密码登陆,需要配置私钥所在的路径，注意现在配置的是默认路径，如果路径有变化这里也要变化-->
	<property>
		<name>dfs.ha.fencing.ssh.private-key-files</name>
		<value>/root/.ssh/id_rsa</value>
	</property>

	<!-- 配置sshfence隔离机制超时时间 -->
	<property>
		<name>dfs.ha.fencing.ssh.connect-timeout</name>
		<value>30000</value>
	</property>

	<!--指定namenode名称空间的存储地址 -->
	<property>
		<name>dfs.namenode.name.dir</name>
		<value>file:///opt/modules/hadoop-2.9.2/hdfs/name</value>
	</property>

	<!--指定datanode数据存储地址 -->
	<property>
		<name>dfs.datanode.data.dir</name>
		<value>file:///opt/modules/hadoop-2.9.2/hdfs/data</value>
	</property>

	<!--指定数据冗余份数 -->
	<property>
		<name>dfs.replication</name>
		<value>3</value>
	</property>

</configuration>
```
> 第三个文件：mapred-site.xml

```
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

	<!-- 指定mr框架为yarn方式 -->
	<property>
		<name>mapreduce.framework.name</name>
		<value>yarn</value>
	</property>

	<!-- 配置 MapReduce JobHistory Server 地址 ，默认端口10020 -->
	<property>
		<name>mapreduce.jobhistory.address</name>
		<value>0.0.0.0:10020</value>
	</property>

	<!-- 配置 MapReduce JobHistory Server web ui 地址， 默认端口19888 -->
	<property>
		<name>mapreduce.jobhistory.webapp.address</name>
		<value>0.0.0.0:19888</value>
	</property>

</configuration>
```

> 第四个文件：yarn-site.xml（配置ResourceManager HA）

```
<?xml version="1.0"?>

<configuration>

	<!--开启resourcemanagerHA,默认为false -->
	<property>
		<name>yarn.resourcemanager.ha.enabled</name>
		<value>true</value>
	</property>

	<!--开启自动恢复功能 -->
	<property>
		<name>yarn.resourcemanager.recovery.enabled</name>
		<value>true</value>
	</property>

	<!-- 指定RM的cluster id -->
	<property>
		<name>yarn.resourcemanager.cluster-id</name>
		<value>yrc</value>
	</property>

	<!--配置resourcemanager -->
	<property>
		<name>yarn.resourcemanager.ha.rm-ids</name>
		<value>rm1,rm2</value>
	</property>

	<!-- 分别指定RM的地址 -->
	<property>
		<name>yarn.resourcemanager.hostname.rm1</name>
		<value>alphasta03</value>
	</property>

	<property>
		<name>yarn.resourcemanager.hostname.rm2</name>
		<value>alphasta04</value>
	</property>

 <!-- <property> <name>yarn.resourcemanager.ha.id</name> <value>rm1</value> 
  <description>If we want to launch more than one RM in single node,we need 
  this configuration</description> </property> -->

	<!-- 指定zk集群地址 -->
	<property>
		<name>ha.zookeeper.quorum</name>
		<value>alphasta05:2181,alphasta06:2181,alphasta07:2181</value>
	</property>

	<!--配置与zookeeper的连接地址-->
	<property>
		<name>yarn.resourcemanager.zk-state-store.address</name>
		<value>alphasta05:2181,alphasta06:2181,alphasta07:2181</value>
	</property>

	<property>
		<name>yarn.resourcemanager.store.class</name>
		<value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
	</property>

	<property>
		<name>yarn.resourcemanager.zk-address</name>
		<value>alphasta05:2181,alphasta06:2181,alphasta07:2181</value>
	</property>

	<property>
		<name>yarn.resourcemanager.ha.automatic-failover.zk-base-path</name>
		<value>/yarn-leader-election</value>
		<description>Optionalsetting.Thedefaultvalueis/yarn-leader-election</description>
	</property>

	<property>
		<name>yarn.nodemanager.aux-services</name>
		<value>mapreduce_shuffle</value>
	</property>

</configuration>
```
> yarn-site.xml（单点 ResourceManager）

```
<configuration>
            <!-- 指定resourcemanager地址 -->
            <property>
                    <name>yarn.resourcemanager.hostname</name>
                    <value>alphasta03</value>
            </property>
            <!-- 指定nodemanager启动时加载server的方式为shuffle server -->
            <property>
                    <name>yarn.nodemanager.aux-services</name>
                    <value>mapreduce_shuffle</value>
            </property>
</configuration>
```

> hadoop-env.sh & mapred-env.sh & yarn-env.sh 配置hadoop守护进程的运行环境

```
export JAVA_HOME=/usr/lib/jdk1.8.0_191
export CLASS_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib

export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
 
export HADOOP_HOME=/opt/modules/hadoop-2.9.2
export HADOOP_PID_DIR=/opt/modules/hadoop-2.9.2/pids 
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native 
export HADOOP_OPTS="$HADOOP_OPTS-Djava.library.path=$HADOOP_HOME/lib/native" 
export HADOOP_PREFIX=$HADOOP_HOME
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop

export HDFS_CONF_DIR=$HADOOP_HOME/etc/hadoop

export YARN_HOME=$HADOOP_HOME
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
```

> 配置slaves（slaves文件中存放的被管理者的主机名，规划中为alphasta05，alphasta06，alphasta07）

```
alphasta05
alphasta06
alphasta07
```
- 分发已配置的应用
    - 分别将配置好的hadoop文件分发到 alphasta01 ~ alphasta07

```
scp -r /opt/modules/hadoop-2.9.2 root@alphasta01:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@alphasta02:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@alphasta03:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@alphasta04:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@alphasta05:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@alphasta06:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@alphasta07:/opt/modules/
```
> 传输完后到对应的目录查看是否有相应的hadoop-2.9.2目录,并将配置文件也分发到所有节点

```
scp -r /etc/profile root@alphasta01:/etc/profile
scp -r /etc/profile root@alphasta02:/etc/profile
scp -r /etc/profile root@alphasta03:/etc/profile
scp -r /etc/profile root@alphasta04:/etc/profile
scp -r /etc/profile root@alphasta05:/etc/profile
scp -r /etc/profile root@alphasta06:/etc/profile
scp -r /etc/profile root@alphasta07:/etc/profile
```
> 传输完成使生效   source /etc/profile

> 验证是否传输成功及生效 echo $HADOOP_HOME验证

> 至此，hadoop的配置文件已经全部配置完毕

6. #### 安装和配置 zookeeper集群

- 1.解压zookeeper
    - 将下载好的zookeeper-3.4.13.tar.gz压缩包，上传到 /alphastaApp目录,并解压

> [root@alphasta01 alphastaApp]$ tar -zxvf zookeeper-3.4.13.tar.gz -C /opt/modules

- 验证成功，并授权

> [root@alphasta05 zookeeper-3.4.13]$cd /opt/modules/    ls  

> [root@alphasta05 zookeeper-3.4.13]$ chmod +wxr zookeeper-3.4.13

- 2.修改zookeeper的配置文件，并建立数据目录和日志目录


> [root@alphasta05 modules]$ cd zookeeper-3.4.13

> [root@alphasta05 zookeeper-3.4.13]$ mkdir data

> [root@alphasta05 zookeeper-3.4.13]$ mkdir logs

> [root@alphasta05 zookeeper-3.4.13]$ vi conf/zoo.cfg
```
# The number of milliseconds of each tick
tickTime=2000
# The number of ticks that the initial 
# synchronization phase can take
initLimit=10
# The number of ticks that can pass between 
# sending a request and getting an acknowledgement
syncLimit=5
# the directory where the snapshot is stored.
# do not use /tmp for storage, /tmp here is just 
# example sakes.
dataDir=/opt/modules/zookeeper-3.4.13/data
dataLogDir=/opt/modules/zookeeper-3.4.13/logs
server.1=alphasta05:2888:3888
server.2=alphasta06:2888:3888
server.3=alphasta07:2888:3888
# the port at which the clients will connect
clientPort=2181
# the maximum number of client connections.
# increase this if you need to handle more clients
#maxClientCnxns=60
#
# Be sure to read the maintenance section of the 
# administrator guide before turning on autopurge.
#
# http://zookeeper.apache.org/doc/current/zookeeperAdmin.html#sc_maintenance
#
# The number of snapshots to retain in dataDir
#autopurge.snapRetainCount=3
# Purge task interval in hours
# Set to "0" to disable auto purge feature
#autopurge.purgeInterval=1
```

[root@alphasta05 zookeeper-3.4.13]$ cd data 
[root@alphasta05 data]$ vi myid     添加内容    1

- 3.复制alphasta05 的zookeeper-3.4.13到alphasta06 和alphasta07 上

> [root@alphasta05 opt]$ scp -r zookeeper-3.4.13 root@alphasta06:/opt/modules/

> [root@alphasta05 opt]$ scp -r zookeeper-3.4.13 root@alphasta07:/opt/modules/

- 4.分别修改alphasta06 和alphasta07 上myid的值为2和3
> [root@alphasta06 zookeeper-3.4.13]$ vi data/myid        2

> [root@alphasta07 zookeeper-3.4.13]$ vi data/myid        3

- 5.分别启动alphasta05 、alphasta06 、alphasta07 上的zookeeper

[root@alphasta05 zookeeper-3.4.13]$ bin/sh zkServer.sh start
[root@alphasta06 zookeeper-3.4.13]$ bin/sh zkServer.sh start
[root@alphasta07 zookeeper-3.4.13]$ bin/sh zkServer.sh start

- 6.查看zookeeper的状态
```
[root@alphasta05 zookeeper-3.4.13]$ bin/ sh zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/modules/zookeeper-3.4.13/bin/../conf/zoo.cfg
Mode: follower
[root@alphasta06 zookeeper-3.4.13]$ bin/ sh zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/modules/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: leader
[root@alphasta07 zookeeper-3.4.13]$ bin/sh zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /opt/modules/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
```


## 启动hadoop集群

1. 启动zookeeper集群

分别在alphasta05、alphasta06、alphasta07上执行如下命令启动zookeeper集群
> [root@alphasta05 zookeeper-3.4.13/bin]$ sh zkServer.sh start

验证zookeeper集群是否启动成功，有两个follower节点跟一个leader节点
> [root@alphasta05 zookeeper-3.4.13/bin]$ sh zkServer.sh status

2. 启动journalnode集群

根据集群规划，只有在alphasta05，alphasta06，alphasta07上运行JournalNode；

注意：在sbin目录下，有hadoop-daemon.sh和hadoop-daemons.sh命令，第一条是单独启动某台设备的某个进程，第二条是启动所有设备的某个进程；

故命令如下：
[root@alphasta05 hadoop-2.9.2]$ sbin/hadoop-daemon.sh start journalnode

运行jps，查看JournalNode进程

3. 格式化zkfc,让在zookeeper中生成ha节点

在alphasta01上执行如下命令（命令行下执行即可），完成格式化，
hdfs zkfc -formatZK

格式成功后，查看zookeeper中可以看到
> [root@alphasta05 bin]# sh zkCli.sh -server alphasta05:2181

可以看到
```
[zk: alphasta05:2181(CONNECTED) 2] ls /
[zookeeper, hadoop-ha]
[zk: alphasta05:2181(CONNECTED) 3]
```

4. 格式化hdfs

在alphasta01上执行命令: hdfs namenode -format，看到类似tmp/dfs/name has been successfully formatted描述信息时说明格式化成功

5. 启动NameNode

首先在alphasta01上启动active节点
> [root@alphasta01 hadoop-2.9.2]$ sbin/hadoop-daemon.sh start namenode

在alphasta02上同步namenode的数据，同时启动standby的namenod,命令如下
#把NameNode的数据同步到c7002上
> [root@alphasta02 hadoop-2.9.2]$ bin/hdfs namenode -bootstrapStandby


#启动alphasta02 上的namenode作为standby
> [root@alphasta02 hadoop-2.9.2]$ sbin/hadoop-daemon.sh start namenode

6. 第六步：启动datanode

根据集群规划在alphasta05 alphasta06 alphasta07 上分别执行如下命令（这里选择单独启动，选择全部启动也只有一个节点启动，待研究）
[root@alphasta01 hadoop-2.9.2]$ sbin/hadoop-daemon.sh start datanode

7. 启动yarn

在作为资源管理器上的机器alphasta03上启动， ,执行如下命令完成yarn的启动（同样只有一个节点启动了，故需要在alphasta04上也执行以下命令）
[root@alphasta01 hadoop-2.9.2]$ sbin/start-yarn.sh

8. 启动zkfc

在alphasta01 上执行如下命令，完成ZKFC的启动
[root@alphasta01 hadoop-2.9.2]$ sbin/hadoop-daemons.sh start zkfc  


### 全部启动后，查看进程

- 每个节点的进程都跟规划中的一致，除了alphasta03（多了一个nodemanager，待研究）不影响使用

- 验证namenode HA 
    - 分别登录nn1 nn2 管理控制台 http://192.168.23.181:50070 http://192.168.23.182:50070 查看其状态，然后关掉active状态的节点，查看另一个节点是否有standby 改为active

- 验证ResourceManager HA 
    - 使用该命令 yarn rmadmin -getServiceState rm1  可查看rm1的状态 yarn rmadmin -getServiceState rm2 查看rm2的状态，同样 关掉一个看另一个的状态的变化
