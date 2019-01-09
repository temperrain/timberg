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
alphasta01|192.168.16.91|JDK、hadoop|NameNode、DFSZKFailoverController
alphasta02|192.168.16.92|JDK、hadoop|NameNode、DFSZKFailoverController
alphasta03|192.168.16.93|JDK、hadoop|ResourceManager
alphasta04|192.168.16.94|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta05|192.168.16.95|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain
alphasta06|192.168.16.96|JDK、hadoop、zookeeper|DataNode、NodeManager、JournalNode、QuorumPeerMain

***
##### 集群搭建方式采用先安装配置好一台样板机，然后再拷贝到其它设备的方式进行
*** 

## 样板机配置

### 设置系统环境

1.   **设置机器名**

   以root用户登录，命令行终端使用# vi /etc/sysconfig/network 打开配置文件，根据集群规划分别设置机器名，重启后生效

```
NETWORKING=yes
HOSTNAME=alphasta01
```

1.   **设置IP地址**

   以root用户登录，打开配置文件 # vi/etc/sysconfig/network-scripts/ifcfg-enp0s3，通过以上方法设置网络配置后，使用命令# service network restart 重启网络，并使用 # ifconfig 查看IP是否生效，如果需要设置后远程连接，建议重启机器。

```
    TYPE="Ethernet"                                 以太网类型
    PROXY_METHOD="none"
    BROWSER_ONLY="no"
    BOOTPROTO="static"                              使用静态IP，而不是用DHCP分配IP
    DEFROUTE="yes"
    IPV4_FAILURE_FATAL="no"
    IPV6INIT="yes"
    IPV6_AUTOCONF="yes"
    IPV6_DEFROUTE="yes"
    IPV6_FAILURE_FATAL="no"
    IPV6_ADDR_GEN_MODE="stable-privacy"
    NAME="enp0s3"                                    网络连接名称
    UUID="b6591044-1086-4cc4-af48-876b26c0e739"
    DEVICE="enp0s3"                                  对应第一张网卡
    ONBOOT="yes"                                     是否启动运行
    IPADDR=192.168.16.59                             指定本机IP地址
    GATEWAY=192.168.16.1                             指定网关
    NETMASK=255.255.255.0                            
    DNS1=222.222.222.222                             DNS地址，配置后同步到 /etc/resolv.conf
```

1.  **设置HOST映射文件**

   以root用户登录，在命令终端使用# vi /etc/hosts打开配置文件，根据集群规划添加如下内容

```   
 192.168.16.91     alphasta01
 192.168.16.92     alphasta02
 192.168.16.93     alphasta03
 192.168.16.94     alphasta04
 192.168.16.95     alphasta05 
 192.168.16.96     alphasta06 
```
