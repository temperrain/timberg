# hadoop 3节点集群搭建

## 基础环境

系统/软件 | 版本号
-|:-:
JDK | 1.8 
hadoop | 2.9.2
Linux系统 | CentOS 7

## 集群规划

主机名|IP |安装的软件|运行的进程
-|:-|:-|:-
master|192.168.23.191|JDK、hadoop|ResourceManager、NameNode、SecondaryNameNode
slave1|192.168.23.192|JDK、hadoop|DataNode、NodeManager
slave2|192.168.23.193|JDK、hadoop|DataNode、NodeManager

## 设置系统环境

1. #### 设置机器名

> 用户root

> 根据集群规划设置机器名

> 重启后生效

```
hostname master                            临时生效
hostnamectl set-hostname master            永久生效，需重启
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
IPADDR=192.168.23.191                           指定本机IP地址
GATEWAY=192.168.16.1                            指定网关
NETMASK=255.255.255.0                            
DNS1=222.222.222.222                            DNS地址，配置后同步到 /etc/resolv.conf
```

3. #### 设置HOST映射文件

> 用户root

> 配置文件 /etc/hosts

> 根据集群规划添加如下内容，并使用 #ping master检测

```   
192.168.23.191     master
192.168.23.192     slave1
192.168.23.193     slave2

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
ssh-copy-id master
ssh-copy-id slave1
ssh-copy-id slave2
```
- 验证是否免密成功 ssh master 

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
scp -r /usr/lib/jdk1.8.0_191 root@master:/usr/lib/
scp -r /etc/profile root@master:/etc/
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
                <value>hdfs://master:9000</value>
        </property>

        <!-- 指定hadoop临时目录 需创建-->
        <property>
                <name>hadoop.tmp.dir</name>
                <value>/opt/modules/hadoop-2.9.2/tmp</value>
        </property>

</configuration>
```

> 第二个文件：hdfs-site.xml

```
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>

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
		<value>1</value>
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

	<property>
		<name>yarn.resourcemanager.hostname</name>
		<value>master</value>
	</property>

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
slave1
slave2
```
- 分发已配置的应用
    - 分别将配置好的hadoop文件分发到 slave1 、slave2

```
scp -r /opt/modules/hadoop-2.9.2 root@slave1:/opt/modules/
scp -r /opt/modules/hadoop-2.9.2 root@slave2:/opt/modules/
```
> 传输完后到对应的目录查看是否有相应的hadoop-2.9.2目录,并将配置文件也分发到所有节点

```
scp -r /etc/profile root@slave1:/etc/profile
scp -r /etc/profile root@slave2:/etc/profile
```
> 传输完成使生效   source /etc/profile

> 验证是否传输成功及生效 echo $HADOOP_HOME验证

> 至此，hadoop的配置文件已经全部配置完毕


## 启动hadoop集群

1. #### 格式化hdfs

> 节点 master

> 命令 hdfs namenode -format

> 看到类似tmp/dfs/name has been successfully formatted描述信息时说明格式化成功

2. #### 启动hdfs

> 节点 master

> [root@master hadoop-2.9.2]$ sbin/sh start-dfs.sh start

3. #### 启动yarn

> 运行节点 master

> [root@master hadoop-2.9.2]$ sbin/start-yarn.sh

### 全部启动后，查看进程

- 每个节点的进程都跟规划中的一致


