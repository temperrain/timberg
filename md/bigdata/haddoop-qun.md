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
alphasta01|192.168.16.181|JDK、hadoop|NameNode、DFSZKFailoverController
alphasta02|192.168.16.182|JDK、hadoop|NameNode、DFSZKFailoverController
alphasta03|192.168.16.183|JDK、hadoop|ResourceManager
alphasta04|192.168.16.184|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta05|192.168.16.185|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta06|192.168.16.186|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain

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
 IPADDR=192.168.16.59                             指定本机IP地址
 GATEWAY=192.168.16.1                             指定网关
 NETMASK=255.255.255.0                            
 DNS1=222.222.222.222                             DNS地址，配置后同步到 /etc/resolv.conf
```

3. **设置HOST映射文件**

以root用户登录，在命令终端使用# vi /etc/hosts打开配置文件，根据集群规划添加如下内容，设置完成后，使用#ping alphasta01检测。

```   
192.168.16.181     alphasta01
192.168.16.182     alphasta02
192.168.16.183     alphasta03
192.168.16.184     alphasta04
192.168.16.185     alphasta05 
192.168.16.186     alphasta06 
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

- 配置hadoop，共6个文件
```
cd $HADOOP_HOME/etc/hadoop           切换到hadoop 配置文件目录


第一个文件： hadoop.env.sh

说明：修改Java的导出路径，如（export JAVA_HOME=/usr/java/jdk1.8.0_80）
vi hadoop.env.sh 修改JAVA_HOEM的值     退出保存


第二个文件： core-site.xml

说明：第一个property配置的是nameservice，nameservice包含两个NameNode，一个是Active状态，另一个额是Standby状态，在集群当中，DataNode不直接连向某NameNode，而是与NameService直接相连。（****注意：<property></property>与<configuration></configuration>之间以Tab缩进，<property></property>里面的属性也用Tab缩进，否则可能会有问题。

<configuration>
     <!-- 指定hdfs的nameservice为ns1 -->
       <property>                                          
            <name>fs.defaultFS</name>
            <value>hdfs://ns1</value>
       </property>
     <!-- 指定hadoop临时目录 -->
       <property>
            <name>hadoop.tmp.dir</name>
            <value>/itcast/hadoop-2.2.0/tmp</value>
       </property>
     <!-- 指定zookeeper地址 -->
       <property>
            <name>ha.zookeeper.quorum</name>
            <value>itcast04:2181,itcast05:2181,itcast06:2181</value>   
       </property>
    </configuration>



## 分发已配置应用



