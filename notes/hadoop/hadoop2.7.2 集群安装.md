# Hadoop2.7.2 集群搭建

以下操作使用bigdata用户

## 1 Hadoop的分布式模型

Hadoop通常有三种运行模式：本地(独立)模式、伪分布式(Pseudo-distributed)模式和完全分布式(Fully distributed)模式。
安装完成后，Hadoop的默认配置即为本地模式，此时Hadoop使用本地文件系统而非分布式文件系统，而且其也不会启动任何Hadoop守护进程，Map和Reduce任务都作为同一进程的不同部分来执行。
因此，本地模式下的Hadoop仅运行于本机。此模式仅用于开发或调试MapReduce应用程序但却避免了复杂的后续操作。伪分布式模式下，Hadoop将所有进程运行于同一台主机上，但此时Hadoop将使用分布式文件系统，而且各jobs也是由JobTracker服务管理的独立进程。
同时，由于伪分布式的Hadoop集群只有一个节点，因此HDFS的块复制将限制为单个副本，其secondary-master和slave也都将运行于本地主机。此种模式除了并非真正意义的分布式之外，其程序执行逻辑完全类似于完全分布式，因此，常用于开发人员测试程序执行。要真正发挥Hadoop的威力，就得使用完全分布式模式。
由于ZooKeeper实现高可用等依赖于奇数法定数目(an odd-numbered quorum)，因此，完全分布式环境需要至少三个节点

## 2 环境准备

### 2.1 角色分配

| 主机名称 | IP              | 角色                                   | 系统名称                  |
| -------- | --------------- | -------------------------------------- | ------------------------- |
| node1    | 192.168.137.101 | namenode,datanode,nodemanager          | Centos release 7.4 x86_64 |
| node2    | 192.168.137.102 | secondarynamenode,datanode,nodemanager | Centos release 7.4 x86_64 |
| node3    | 192.168.137.103 | resourcemanager,datanode,nodemanager   | Centos release 7.4 x86_64 |

### 2.2 虚拟机环境准备

#### 2.2.1 准备虚拟机

使用vagrant 安装三台虚拟机

[VirtualBox+Vagrant环境搭建](https://www.wiz.cn/wapp/recent/?docGuid=4afdc355-fe03-4e4e-b5b0-4b95422205d9&cmd=km%2C)

####  2.2.2 关闭防火墙以及hosts配置

#### 2.2.3 创建bigdata 用户以及配置用户的具有root权限

[建立sudo用户](https://www.wiz.cn/wapp/recent/?docGuid=4400a8f1-f4d4-4175-ad36-0b666f353292&cmd=km%2C)

#### 2.2.4 在/opt 目录下创建文件夹

```shell
# 创建文件夹
sudo mkdir module
sudo mkdir software
#修改 module、software 文件夹的所有者
sudo chown bigdata:bigdata module/ software/
```

#### 2.2.5 配置所有的机器ssh互信

[centos7配置免密](https://www.wiz.cn/wapp/recent/?docGuid=8883c7fd-b69d-4de9-85b3-6d1d993075e5&cmd=km%2C)

#### 2.2.6 同步时间，配置yum和阿里镜像源

[集群时间同步](https://www.wiz.cn/wapp/recent/?docGuid=59c69ec1-92ed-4237-a4c2-57c8c57a4508&cmd=km%2C)

[centos配置国内yum源](https://www.wiz.cn/wapp/recent/?docGuid=d8216935-eff1-4616-9e09-d6841ffe2601&cmd=km%2C)

## 3 安装jdk和配置环境变量

[Centos7搭建java环境](https://www.wiz.cn/wapp/recent/?docGuid=989a8191-ed81-4220-81aa-e9198749f580&cmd=km%2C)

## 4 安装并配置hadoop

**Hadoop 下载地址**

https://archive.apache.org/dist/hadoop/common/hadoop-2.7.2/

### 4.1 安装Hadoop并配置环境变量(node1上)

```shell
$tar -zxvf hadoop-2.6.0.tar.gz -C /opt/module/
$cd /opt/module/

#配置环境变量
sudo vim /etc/profile.d/myenv.sh
source /etc/profile.d/myenv.sh

#HADOOP_HOME
export HADOOP_HOME=/opt/module/hadoop-2.6.0
export PATH=$HADOOP_HOME/bin:$HADOOP_HOME/sbin:$PATH
```

### 4.2 修改配置文件

***注意：所有文件均位于/usr/local/hadoop/etc/hadoop路径下***

#### 3.2.1 hadoop-env.sh

```
24 # The java implementation to use.
25 export JAVA_HOME=export JAVA_HOME=/opt/module/jdk1.8.0_261 #将JAVA_HOME改为固定路径
```

#### 3.2.2 core-site.xml

```xml
<configuration>
   <!-- 指定HDFS老大（namenode）的通信地址 -->
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://node1:9000</value>
    </property>
    <!-- 指定hadoop运行时产生文件的存储路径 -->
    <property>
        <name>hadoop.tmp.dir</name>
        <value>/opt/module/hadoop-2.6.0/data</value>
    </property>
</configuration>
```

#### 3.2.3 hdfs-site.xml

```xml
<configuration>
    <!-- 设置namenode的http通讯地址 -->
    <property>
       <name>dfs.namenode.http-address</name>
       <value>node1:50070</value>
    </property>
    <!-- 设置secondarynamenode的http通讯地址 -->
    <property>
       <name>dfs.namenode.secondary.http-address</name>
       <value>node2:50090</value>
    </property>
     <!-- 设置namenode存放的路径 -->
    <property>
       <name>dfs.namenode.name.dir</name>
       <value>/opt/module/hadoop-2.6.0/data/namenode</value>
    </property>
    <!-- 设置hdfs副本数量 -->
    <property>
       <name>dfs.replication</name>
       <value>1</value>
    </property>
    <!-- 设置datanode存放的路径 -->
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/opt/module/hadoop-2.6.0/data/datanode</value>
    </property>
</configuration>
```

#### 3.2.4 mapred-site.xml

```shell
mv mapred-site.xml.template mapred-site.xml
```

```xml
<configuration>
    <!-- 通知框架MR使用YARN -->
    <property>
       <name>mapreduce.framework.name</name>
       <value>yarn</value>
    </property>
</configuration>
```

#### 3.2.5 yarn-site.xml

```xml
<configuration>
    <!-- 设置 resourcemanager 在哪个节点-->
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>node3</value>
    </property>

    <!-- reducer取数据的方式是mapreduce_shuffle -->
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>

    <property>
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
        <value>org.apache.hadoop.mapred.ShuffleHandler</value>
    </property>
</configuration>
```

#### 3.2.6 slaves

```
node1
node2
node3
```

#### 3.2.7 创建相关目录

```shell
mkdir /opt/module/hadoop-2.6.0/data
mkdir /opt/module/hadoop-2.6.0/data/datanode
mkdir /opt/module/hadoop-2.6.0/data/namenode
```

### 4.3 复制Hadoop 安装目录及环境配置文件到其他主机

```shell
xsync.sh hadoop-2.6.0/
sudo /home/bigdata/bin/xsync.sh /etc/profile.d/myenv.sh

#每台主机
source /etc/profile.d/myenv.sh
```

**xsync.sh脚本**

```shell
#! /bin/bash
#1. 判断参数的个数
if [ $# -lt 1 ]
then
        echo Not Enough Arguement!
        exit
fi

#2. 遍历集群所有机器
for host in node1 node2 node3
do
    echo ============== $host ===========
        # 遍历所有的目录，挨个发送
        for file in $@
        do
                #4.判断文件是否存在
                if [ -e $file ]
                then
                        #5. 获取父目录
                        pdir=$(cd -P $(dirname $file); pwd)
                        #6. 获取当前文件的名称
                        fname=$(basename $file)
                        ssh $host "mkdir -p $pdir"
                        rsync -av $pdir/$fname $host:$pdir
                else
                        echo $file does not exists!
                fi
        done
done
```

## 5 启动Hadoop

### 5.1 格式化名称节点(node1)

```shell
hdfs namenode -format
20/10/13 02:36:55 INFO common.Storage: Storage directory /opt/module/hadoop-2.6.0/data/namenode has been successfully formatted. #这行信息表明对应的存储已经格式化成功。
20/10/13 02:36:56 INFO namenode.NNStorageRetentionManager: Going to retain 1 images with txid >= 0
20/10/13 02:36:56 INFO util.ExitUtil: Exiting with status 0
20/10/13 02:36:56 INFO namenode.NameNode: SHUTDOWN_MSG:
/************************************************************
SHUTDOWN_MSG: Shutting down NameNode at node1/192.168.137.101
************************************************************/
```

### 5.2 启动dfs及yarn

```shell
#node1上执行
sbin/start-dfs.sh
#node3 上执行（因为ResourceManager在node3上）
sbin/start-yarn.sh
```

### 5.3 查看结果

```shell
[bigdata@node1 logs]$ xcall.sh jps
------node1-----
4614 NameNode
5462 Jps
5065 NodeManager
3549 Worker
3455 Master
4735 DataNode
------node2-----
4096 Jps
3430 Worker
3798 SecondaryNameNode
3705 DataNode
3867 NodeManager
------node3-----
3617 NodeManager
3252 Worker
3845 ResourceManager
3518 DataNode
4142 Jps
```

***其中 xcall.sh 脚本如下***

/home/bigdata/bin/xcall.sh

```shell
#! /bin/bash
for i in node1 node2 node3
do
  echo ------$i-----
  ssh $i "$*"
done
```

## 6 测试结果

### 6.1 查看集群状态

```shell
hdfs dfsadmin -report

Configured Capacity: 128782970880 (119.94 GB)
Present Capacity: 115143544832 (107.24 GB)
DFS Remaining: 115143532544 (107.24 GB)
DFS Used: 12288 (12 KB)
DFS Used%: 0.00%
Under replicated blocks: 0
Blocks with corrupt replicas: 0
Missing blocks: 0

-------------------------------------------------
Live datanodes (3):

Name: 192.168.137.102:50010 (node2)
Hostname: node2
Decommission Status : Normal
Configured Capacity: 42927656960 (39.98 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 4364705792 (4.06 GB)
DFS Remaining: 38562947072 (35.91 GB)
DFS Used%: 0.00%
DFS Remaining%: 89.83%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Tue Oct 13 02:48:35 UTC 2020


Name: 192.168.137.101:50010 (node1)
Hostname: node1
Decommission Status : Normal
Configured Capacity: 42927656960 (39.98 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 4910219264 (4.57 GB)
DFS Remaining: 38017433600 (35.41 GB)
DFS Used%: 0.00%
DFS Remaining%: 88.56%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Tue Oct 13 02:48:35 UTC 2020


Name: 192.168.137.103:50010 (node3)
Hostname: node3
Decommission Status : Normal
Configured Capacity: 42927656960 (39.98 GB)
DFS Used: 4096 (4 KB)
Non DFS Used: 4364500992 (4.06 GB)
DFS Remaining: 38563151872 (35.91 GB)
DFS Used%: 0.00%
DFS Remaining%: 89.83%
Configured Cache Capacity: 0 (0 B)
Cache Used: 0 (0 B)
Cache Remaining: 0 (0 B)
Cache Used%: 100.00%
Cache Remaining%: 0.00%
Xceivers: 1
Last contact: Tue Oct 13 02:48:35 UTC 2020
```

### 6.2 测试YARN

可以访问YARN的管理界面，验证YARN，如下图所示：

![YARN管理界面](https://gitee.com/zhdoop/blogImg/raw/master/img/Yarn-WEBUI.PNG)

### 6.3 测试查看HDFS

![HDFS界面](https://gitee.com/zhdoop/blogImg/raw/master/img/HDFS界面.PNG)

