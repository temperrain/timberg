# hadoop 高可用完全分布式集群搭建

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

1. **设置机器名**

以root用户登录，命令行终端使用一下命令，根据集群规划分别设置机器名，重启后生效

```
hostname test                              临时生效
hostnamectl set-hostname centos7           永久生效，需重启
```

2. **设置IP地址**

以root用户登录，打开配置文件 # vi/etc/sysconfig/network-scripts/ifcfg-enp0s3，通过以上方法设置网络配置后，使用命令# service network restart 重启网络，并使用 # ifconfig 查看IP是否生效，如果需要设置后远程连接，建议重启机器。

```
 TYPE="Ethernet"                                 以太网类型
 PROXY_METHOD="none"
 BROWSER_ONLY="no"
 BOOTPROTO="static"                              使用静态IP，而不是用DHCP分配IP
 DEFROUTE="yes"
 NAME="enp0s3"                                    网络连接名称
 UUID="b6591044-1086-4cc4-af48-876b26c0e739"
 DEVICE="enp0s3"                                  对应第一张网卡
 ONBOOT="yes"                                     是否启动运行
 IPADDR=192.168.23.181                            指定本机IP地址
 GATEWAY=192.168.16.1                             指定网关
 NETMASK=255.255.255.0                            
 DNS1=222.222.222.222                             DNS地址，配置后同步到 /etc/resolv.conf
```

3. **设置HOST映射文件**

以root用户登录，在命令终端使用# vi /etc/hosts打开配置文件，根据集群规划添加如下内容，设置完成后，使用#ping alphasta01检测。

```   
192.168.23.181     alphasta01
192.168.23.182     alphasta02
192.168.23.183     alphasta03
192.168.23.184     alphasta04
192.168.23.185     alphasta05 
192.168.23.186     alphasta06
192.168.23.187     alphasta07
```
4. ***关闭防火墙和SELinux 需重启机器生效***
 
CentOS 7.0默认使用的是firewall作为防火墙，使用iptables必须重新设置一下

- 直接关闭防火墙
```
systemctl stop firewalld.service        停止firewall
systemctl disable firewalld.service     禁止firewall开机启动

systemctl restart iptables.service      重启防火墙使配置生效
systemctl enable iptables.service       设置防火墙开机启动
```

- ~~设置iptables service,如果使用service iptables 需要设置一下~~

```
yum -y install iptables-services                           切换到root用户 安装

service iptables [start|stop|restart|save|status]          之后就可以使用了

vi /etc/sysconfig/iptables                                 修改防火墙配置，如添加端口9908
```    

- 关闭SELinux

以root用户登录，编辑文件 # vi /etc/selinux/config,设置SELINUX=disable


5. ***SSH免密码登录***
    
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
##### 集群搭建方式采用先安装配置好一台样板机，然后再拷贝到其它设备的方式进行
*** 

## 配置样板机

1. ***添加hadoop用户/用户组***
```
groupadd -g 1000 hadoop                    指定用户组ID 1000 （系统用户组ID不低于500）如果ID已存在，就改为其它或去掉 -g 1000
useradd -u 2000 -g hadoop hadoop           指定用户UID 2000 及所属的主组
passwd hadoop                              指定用户密码
```

2. ***后续会用到sudo命令，故需把hadoop用户加入到sudoers中***

```
chmod u+w /etc/sudoers      先修改该配置文件的权限

vi /etc/sudoers             打开该配置文件，找到 root ALL=(ALL) ALL 在其下边添加 hadoop ALL=（ALL）

```

3. ***统一目录结构（所属组和用户均为hadoop）***

```
mkdir /alphastaApp                           上传目录

mkdir /opt/modules                           软件安装目录

chown -R hadoop:hadoop /opt/modules          将该目录的用户组设定为 hadoop
```

4. ***安装和配置JDK***

- JDK安装包（jdk-8u191-linux-x64.tar.gz）              上传 到 /alphastaApp目录下
- cd /alphastaApp                                     验证是否上传成功
- tar -zxvf jdk-8u191-linux-x64.tar.gz -C /usr/lib    将JDK解压到 /usr/lib目录下
- cd /usr/lib      ls                                 验证是否解压成功
- vi /etc/profile                                     打开配置文件，按下 shift + G 快速到文件末尾 按o键新增一行
- 在文件末尾添加如下内容，:wq保存退出（shift +ZZ）
```
export JAVA_HOME=/usr/lib/jdk-8u191-linux-x64
export PATH=$PATH:$JAVA_HOME/bin
```
- source /etc/profile                                 使配置文件生效
- Java -version                                       在任意目录输入此命令验证是否安装成功

***分发已配置好的JDK***

如果profile文件已经配置，各个机器不一样，就选择各自配置profile即可

scp -r /usr/lib/jdk-8u191-linux-x64 root@alphasta01:/usr/lib/
scp -r /etc/profile root@alphasta01:/etc/

5. ***安装和配置 hadoop***

- 上传hadoop 安装包到 /alphastaApp
- tar -zxvf hadoop-2.9.2.tar.gz -C /opt/modules/      解压到/opt/modules目录下
- cd /opt/modules   ls                                切换到/opt/modules目录下查看是否解压成功
 
> 安装完hadoop之后再配置HDFS，（hadoop2.0所有的配置文件都在$HADOOP_HOME/etc/hadoop目录下）

- 将hadoop 添加到环境变量
```
vi /etc/profile                      shift + G 跳转到文件末尾，添加如下内容

export HADOOP_HOME=/opt/modules/hadoop-2.9.2
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin
```

- 配置hadoop，共7个文件(core-site.xml、hdfs-site.xml、mapreduce-site.xml、yarn-site.xml、hadoop-env.sh、mapred-env.sh、yarn-env.sh)
```
cd $HADOOP_HOME/etc/hadoop           切换到hadoop 配置文件目录


第一个文件：core-site.xml

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
  <value>alphasta04:2181,alphasta05:2181,alphasta06:2181</value>
 </property>

 <property>
  <name>ha.zookeeper.session-timeout.ms</name>
  <value>3000</value>
 </property>

</configuration>


第二个文件：hdfs-site.xml 

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
  <value>alphasta01:9000</value>
 </property>

 <!-- nn2的RPC通信地址 -->
 <property>
  <name>dfs.namenode.rpc-address.alphasta.nn2</name>
  <value>alphasta02:9000</value>
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
  <value>qjournal://alphasta04:8485;alphasta05:8485;alphasta06:8485/alphasta</value>
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
  <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider
  </value>
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


第三个文件：mapred-site.xml

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


第四个文件：yarn-site.xml（配置ResourceManager HA）


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
  <value>alphasta04:2181,alphasta05:2181,alphasta06:2181</value>
 </property>

 !--配置与zookeeper的连接地址-->
 <property>
  <name>yarn.resourcemanager.zk-state-store.address</name>
  <value>alphasta04:2181,alphasta05:2181,alphasta06:2181</value>
 </property>

 <property>
  <name>yarn.resourcemanager.store.class</name>
  <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore
  </value>
 </property>

 <property>
  <name>yarn.resourcemanager.zk-address</name>
  <value>alphasta04:2181,alphasta05:2181,alphasta06:2181</value>
 </property>

 <property>
  <name>yarn.resourcemanager.ha.automatic-failover.zk-base-path</name>
  <value>/yarn-leader-election</value>
  <description>Optionalsetting.Thedefaultvalueis/yarn-leader-election
  </description>
 </property>

 <property>
  <name>yarn.nodemanager.aux-services</name>
  <value>mapreduce_shuffle</value>
 </property>
</configuration>


第四个文件：yarn-site.xml（单点 ResourceManager）


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


hadoop-env.sh & mapred-env.sh & yarn-env.sh 配置hadoop守护进程的运行环境

export JAVA_HOME=/opt/jdk1.8.0_121
export CLASS_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib 
export HADOOP_HOME=/opt/hadoop-2.8.0
export HADOOP_PID_DIR=/opt/hadoop-2.8.0/pids 
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native 
export HADOOP_OPTS="$HADOOP_OPTS-Djava.library.path=$HADOOP_HOME/lib/native" 
export HADOOP_PREFIX=$HADOOP_HOME 
export HADOOP_MAPRED_HOME=$HADOOP_HOME 
export HADOOP_COMMON_HOME=$HADOOP_HOME 
export HADOOP_HDFS_HOME=$HADOOP_HOME 
export YARN_HOME=$HADOOP_HOME 
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop 
export HDFS_CONF_DIR=$HADOOP_HOME/etc/hadoop 
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop 
export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native 
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin


配置slaves（slaves文件中存放的被管理者的主机名，规划中为alphasta05，alphasta06，alphasta07）
alphasta01
alphasta02
alphasta03
alphasta04
alphasta05
alphasta06
alphasta07

```

## 分发已配置应用



