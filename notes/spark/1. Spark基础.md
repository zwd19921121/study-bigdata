# Spark 基础

## 1 Spark 概述

### 1.1 什么是Spark

Spark 是一个快速(基于内存), 通用, 可扩展的集群计算引擎

并且 Spark 目前已经成为 Apache 最活跃的开源项目, 有超过 1000 个活跃的贡献者.

### 1.2. Spark 特点

#### 1.2.1  快速

与 Hadoop 的 MapReduce 相比, Spark 基于内存的运算是 MapReduce 的 100 倍.基于硬盘的运算也要快 10 倍以上.

Spark 实现了高效的 DAG 执行引擎, 可以通过基于内存来高效处理数据流

#### 1.2.2 易用

Spark 支持 Scala, Java, Python, R 和 SQL 脚本, 并提供了超过 80 种高性能的算法, 非常容易创建并行 App

而且 Spark 支持交互式的 Python 和 Scala 的 shell, 这意味着可以非常方便地在这些 shell 中使用 Spark 集群来验证解决问题的方法, 而不是像以前一样 需要打包, 上传集群, 验证等. 这对于原型开发非常重要.

#### 1.2.3 通用

Spark 结合了SQL, Streaming和复杂分析.

Spark 提供了大量的类库, 包括 SQL 和 DataFrames, 机器学习(MLlib), 图计算(GraphicX), 实时流处理(Spark Streaming) .

可以把这些类库无缝的柔和在一个 App 中.

减少了开发和维护的人力成本以及部署平台的物力成本.

#### 1.2.4 可融合性

Spark 可以非常方便的与其他开源产品进行融合.

比如, Spark 可以使用 Hadoop 的 YARN 和 Appache Mesos 作为它的资源管理和调度器, 并且可以处理所有 Hadoop 支持的数据, 包括 HDFS, HBase等.

### 1.3 Spark 内置模块介绍

![Spark 内置模块介绍](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark内置模块.png)

#### 1.3.1 集群管理器(Cluster Manager)

Spark 设计为可以高效地在一个计算节点到数千个计算节点之间伸缩计算。

为了实现这样的要求，同时获得最大灵活性，Spark 支持在各种集群管理器(Cluster Manager)上运行，目前 Spark 支持 3 种集群管理器:

1. Hadoop YARN(在国内使用最广泛)

2. Apache Mesos(国内使用较少, 国外使用较多)

3. Standalone(Spark 自带的资源调度器, 需要在集群中的每台节点上配置 Spark)

#### 1.3.2 SparkCore

实现了 Spark 的基本功能，包含任务调度、内存管理、错误恢复、与存储系统交互等模块。SparkCore 中还包含了对弹性分布式数据集(Resilient Distributed DataSet，简称RDD)的API定义。

#### 1.3.3 Spark SQL

是 Spark 用来操作结构化数据的程序包。通过SparkSql，我们可以使用 SQL或者Apache Hive 版本的 SQL 方言(HQL)来查询数据。Spark SQL 支持多种数据源，比如 Hive 表、Parquet 以及 JSON 等。

#### 1.3.4  Spark Streaming

是 Spark 提供的对实时数据进行流式计算的组件。提供了用来操作数据流的 API，并且与 Spark Core 中的 RDD API 高度对应。

#### 1.3.5 Spark MLlib

提供常见的机器学习 (ML) 功能的程序库。包括分类、回归、聚类、协同过滤等，还提供了模型评估、数据导入等额外的支持功能。



## 2 Spark 运行模式

### 2.1 Local 模式

Local 模式就是指的只在一台计算机上来运行 Spark.

通常用于测试的目的来使用 Local 模式, 实际的生产环境中不会使用 Local 模式.

#### 2.1.1 解压 Spark 安装包

把安装包上传到/opt/software/下, 并解压到/opt/module/目录下

```shell
tar -zxvf spark-2.1.1-bin-hadoop2.7.tgz -C /opt/module
```

然后复制刚刚解压得到的目录, 并命名为spark-local:

```shell
cp -r spark-2.1.1-bin-hadoop2.7 spark-local
```

#### 2.1.2 运行官方求PI 的实例

```shell
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master local[2] \
./examples/jars/spark-examples_2.11-2.1.1.jar 100
```

注意:

* 如果你的shell是使用的zsh, 则需要把local[2]加上引号:'local[2]'

说明:

1. 使用spark-submit来发布应用程序.

2. 语法:

```shell
./bin/spark-submit \
--class <main-class> \
--master <master-url> \
--deploy-mode <deploy-mode> \
--conf <key>=<value> \
... # other options
<application-jar> \
[application-arguments]
```

* --master 指定 master 的地址，默认为local. 表示在本机运行.

* --class 你的应用的启动类 (如 org.apache.spark.examples.SparkPi)
* --deploy-mode 是否发布你的驱动到 worker节点(cluster 模式) 或者作为一个本地客户端 (client 模式) (default: client)
* --conf: 任意的 Spark 配置属性， 格式key=value. 如果值包含空格，可以加引号"key=value"
* application-jar: 打包好的应用 jar,包含依赖. 这个 URL 在集群中全局可见。 比如hdfs:// 共享存储系统， 如果是 file:// path， 那么所有的节点的path都包含同样的jar
* application-arguments: 传给main()方法的参数
*  --executor-memory 1G 指定每个executor可用内存为1G
* --total-executor-cores 6 指定所有executor使用的cpu核数为6个
* --executor-cores 表示每个executor使用的 cpu 的核数
* 关于 Master URL 的说明

| Master URL        | Meaning                                                      |
| ----------------- | ------------------------------------------------------------ |
| Local             | Run Spark locally with one worker thread (i.e. no parallelism at all). |
| local[K]          | Run Spark locally with K worker threads (ideally, set this to the number of cores on your machine). |
| local[*]          | Run Spark locally with as many worker threads as logical cores on your machine. |
| spark://HOST:PORT | Connect to the given [Spark standalone cluster](http://spark.apache.org/docs/2.1.1/spark-standalone.html) master. The port must be whichever one your master is configured to use, which is 7077 by default. |
| mesos://HOST:PORT | Connect to the given [Mesos](http://spark.apache.org/docs/2.1.1/running-on-mesos.html) cluster. The port must be whichever one your is configured to use, which is 5050 by default. Or, for a Mesos cluster using ZooKeeper, use mesos://zk://.... To submit with --deploy-mode cluster, the HOST:PORT should be configured to connect to the [MesosClusterDispatcher](http://spark.apache.org/docs/2.1.1/running-on-mesos.html#cluster-mode). |
| yarn              | Connect to a [YARN](http://spark.apache.org/docs/2.1.1/running-on-yarn.html)cluster in client or cluster mode depending on the value of --deploy-mode. The cluster location will be found based on the HADOOP_CONF_DIR or YARN_CONF_DIR variable. |

##### 2.1.2.1 结果展示

该算法是利用蒙特·卡罗算法求PI

![PI 结果](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Local-PI-result.png)

##### 2.1.2.2 备注：也可以使用run-examples来运行

```shell
bin/run-example SparkPi 100
```

#### 2.1.3 使用Spark-shell

Spark-shell 是 Spark 给我们提供的交互式命令窗口(类似于 Scala 的 REPL)

本案例在 Spark-shell 中使用 Spark 来统计文件中各个单词的数量.

##### 步骤1： 创建2个文件

```shell
mkdir input
cd input
touch 1.txt
touch 2.txt
# 分别在1.txt 和 2.txt 内输入一些单词
```

##### 步骤2： 打开 Spark-shell

```shell
bin/spark-shell
```

![spark-shell](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Local-spark-shell.png)

##### 步骤3：查看进程和通过web查看应用程序运行情况

![spark-sehll进程](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Local-spark-shell进程.png)

web ui 地址是: http://hadoop201:4040

![web-ui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Locak-spark-shell-webui.png)

##### 步骤4：运行wordcount 程序

```shell
sc.textFile("input/").flatmap(_.split(" ")).map((_,1)).reduceByKey(_ + _).collect
```

![wordcount结果](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Local-spark-shell-wordcount-result.png)

##### 步骤5：登录hadoop201:4040查看程序运行

![wordcount-webui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Local-wordcount-webui.png)

#### 2.1.4 提交流程

Spark 通用运行简易流程

![Spark提交流程](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-spark提交流程.png)

#### 2.1.5 wordcount 数据流程分析:

1. textFile("input")：读取本地文件input文件夹数据；

2. flatMap(_.split(" "))：压平操作，按照空格分割符将一行数据映射成一个个单词；

3. map((_,1))：对每一个元素操作，将单词映射为元组；

4. reduceByKey(_+_)：按照key将值进行聚合，相加；

5. collect：将数据收集到Driver端展示。

### 2.2 Spark 核心概念介绍

#### 2.2.1 Master

Spark 特有资源调度系统的 Leader。掌管着整个集群的资源信息，类似于 Yarn 框架中的 ResourceManager，主要功能：

1. 监听 Worker，看 Worker 是否正常工作；

2. Master 对 Worker、Application 等的管理(接收 Worker 的注册并管理所有的Worker，接收 Client 提交的 Application，调度等待的 Application 并向Worker 提交)。

#### 2.2.2 Work

Spark 特有资源调度系统的 Slave，有多个。每个 Slave 掌管着所在节点的资源信息，类似于 Yarn 框架中的 NodeManager，主要功能：

1. 通过 RegisterWorker 注册到 Master；

2. 定时发送心跳给 Master；

3. 根据 Master 发送的 Application 配置进程环境，并启动 ExecutorBackend(执行 Task 所需的临时进程)

#### 2.2.3 driver program(驱动程序)

每个 Spark 应用程序都包含一个*驱动程序*, 驱动程序负责把并行操作发布到集群上.

驱动程序包含 Spark 应用程序中的*主函数*, 定义了分布式数据集以应用在集群中.

在前面的wordcount案例集中, spark-shell 就是我们的驱动程序, 所以我们可以在其中键入我们任何想要的操作, 然后由他负责发布.

驱动程序通过SparkContext对象来访问 Spark, SparkContext对象相当于一个到 Spark 集群的连接.

在 spark-shell 中, 会自动创建一个SparkContext对象, 并把这个对象命名为sc.

![Driver](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-driver-program.png)

#### 2.2.4  executor(执行器)

SparkContext对象一旦成功连接到集群管理器, 就可以获取到集群中每个节点上的执行器(executor).

执行器是一个进程(进程名: ExecutorBackend, 运行在 Worker 节点上), 用来执行计算和为应用程序存储数据.

然后, Spark 会发送应用程序代码(比如:jar包)到每个执行器. 最后, SparkContext对象发送任务到执行器开始执行程序.

![executor](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-executor.png)

#### 2.2.5 1.2.4  RDDs(Resilient Distributed Dataset) 弹性分布式数据集

一旦拥有了SparkContext对象, 就可以使用它来创建 RDD 了. 在前面的例子中, 我们调用sc.textFile(...)来创建了一个 RDD, 表示文件中的每一行文本. 我们可以对这些文本行运行各种各样的操作.

#### 2.2.6 cluster managers(集群管理器)

为了在一个 Spark 集群上运行计算, SparkContext对象可以连接到几种集群管理器(Spark’s own standalone cluster manager, Mesos or YARN).

集群管理器负责跨应用程序分配资源.

#### 2.2.7 专业术语列表

| Term            | Meaning                                                      |
| --------------- | ------------------------------------------------------------ |
| Application     | User program built on Spark. Consists of a *driver program* and *executors* on the cluster. (构建于 Spark 之上的应用程序. 包含驱动程序和运行在集群上的执行器) |
| Application jar | A jar containing the user’s Spark application. In some cases users will want to create an “uber jar” containing their application along with its dependencies. The user’s jar should never include Hadoop or Spark libraries, however, these will be added at runtime. |
| Driver program  | The thread running the main() function of the application and creating the SparkContext |
| Cluster manager | An external service for acquiring resources on the cluster (e.g. standalone manager, Mesos, YARN) |
| Deploy mode     | Distinguishes where the driver process runs. In “cluster” mode, the framework launches the driver inside of the cluster. In “client” mode, the submitter launches the driver outside of the cluster. |
| Worker node     | Any node that  can run application code in the cluster       |
| Executor        | A process  launched for an application on a worker node, that runs tasks and keeps data  in memory or disk storage across them. Each application has its own  executors. |
| Task            | A unit of work that will be sent to one executor             |
| Job             | A parallel computation consisting of multiple tasks that gets spawned in response to a Spark action (e.g. save, collect); you’ll see this term used in the driver’s logs. |
| Stage           | Each job gets divided into smaller sets of tasks called *stages* that depend on each other (similar to the map and reduce stages in MapReduce); you’ll see this term used in the driver’s logs. |

### 2.3 Standalone 模式

构建一个由 Master + Slave 构成的 Spark 集群，Spark 运行在集群中。

这个要和 Hadoop 中的 Standalone 区别开来. 这里的 Standalone 是指只用 Spark 来搭建一个集群, 不需要借助其他的框架.是相对于 Yarn 和 Mesos 来说的.

#### 2.3.1 配置Standalone 模式

##### 步骤1：复制 spark, 并命名为spark-standalone

```shell
cp -r spark-2.1.1-bin-hadoop2.7 spark-standalone
```

##### 步骤2：进入配置文件目录conf, 配置spark-evn.sh

```shell
cd conf/
cp spark-env.sh.template spark-env.sh
```

在spark-env.sh 文件中配置如下内容:

```shell
SPARK_MASTER_HOST=hadoop201
SPARK_MASTER_PORT=7077 # 默认端口就是7077, 可以省略不配
```

##### 步骤3: 修改 slaves 文件, 添加 worker 节点

```shell
cp slaves.template slaves
```

在slave 文件中配置如下的内容:

```shell
hadoop201
hadoop202
hadoop203
```

##### 步骤4: 分发spark-standalone

##### 步骤5: 启动 Spark 集群

```shell
sbin/start-all.sh
```



可能碰到的问题

*  如果启动的时候报:JAVA_HOME is not set, 则在sbin/spark-config.sh中添加入JAVA_HOME变量即可. 不要忘记分发修改的文件

![](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-standalone问题.png)

![问题解决](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-stabdalone安装问题解决.png)

##### 步骤6: 在网页中查看 Spark 集群情况

地址:htttp://hadoop201:8080

![web-ui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-standalobe-web-ui.png)

#### 2.3.2 使用 Standalone 模式运行计算 PI 的程序

```shell
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop201:7077 \
--executor-memory 1G \
--total-executor-cores 6 \
--executor-cores 2 \
./examples/jars/spark-examples_2.11-2.1.1.jar 100
```

![pi](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-standalone-PI.png)

#### 2.3.3 在 Standalone 模式下启动 Spark-shell

```shell
bin/spark-shell \
--master spark://hadoop201:7077
```

说明:

*   --master spark://hadoop201:7077指定要连接的集群的master

![standalone](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-standalone1.png)

![standalone-webui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-standalone2.png)

执行wordcount 程序

```shell
sc.textFile("input/").flatMap(_.split(" ")).map((_,1)).reduceByKey(_+_).collect
res4: Array[(String, Int)] = Array((are,2), (how,2), (hello,4), (atguigu,2), (world,2), (you,2))
```

注意：

*  每个worker节点上要有相同的文件夹:input/, 否则会报文件不存在的异常	

#### 2.3.4 配置 Spark 任务历史服务器(为 Standalone 模式配置)

在 Spark-shell 没有退出之前, 我们是可以看到正在执行的任务的日志情况:http://hadoop201:4040. 但是退出 Spark-shell 之后, 执行的所有任务记录全部丢失.

所以需要配置任务的历史服务器, 方便在任何需要的时候去查看日志.

##### 步骤1：配置spark-default.conf文件, 开启 Log

```shell
cp spark-defaults.conf.template spark-defaults.conf
```

在spark-defaults.conf文件中, 添加如下内容:

```shell
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://hadoop201:9000/spark-job-log
```

注意:

hdfs://hadoop201:9000/spark-job-log 目录必须提前存在, 名字随意

##### 步骤2: 修改spark-env.sh文件，添加如下配置.

```shell
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=30 -Dspark.history.fs.logDirectory=hdfs://hadoop201:9000/spark-job-log"
```

##### 步骤3：分发配置文件

##### 步骤4: 启动历史服务

需要先启动HDFS，然后在启动

```shell
sbin/start-history-server.sh
```

![JPS](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-JPS.png)

ui 地址:http://hadoop201:18080

![web-ui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-history-webui.png)

##### 步骤5: 启动任务, 查看历史服务器

```shell
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master spark://hadoop201:7077 \
--executor-memory 1G \
--total-executor-cores 6 \
./examples/jars/spark-examples_2.11-2.1.1.jar 100
```

![web-ui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-history-task-web-ui.png)

#### 2.3.5 HA配置(为 Mater 配置)

由于 master 只有一个, 所以也有单点故障问题.

![HA](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-HA.png)

可以启动多个 master, 先启动的处于 Active 状态, 其他的都处于 Standby 状态

##### 步骤1：给 spark-env.sh 添加如下配置

```shell
# 注释掉如下内容：
#SPARK_MASTER_HOST=hadoop201
#SPARK_MASTER_PORT=7077
# 添加上如下内容：
export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=hadoop201:2181,hadoop202:2181,hadoop203:2181 -Dspark.deploy.zookeeper.dir=/spark"
```

##### 步骤2: 分发配置文件

##### 步骤3: 启动 Zookeeper

##### 步骤4: 在 hadoop201 启动全部节点

```shell
sbin/start-all.sh
```

会在当前节点启动一个 master

##### 步骤5: 在 hadoop202 启动一个 master

```shell
sbin/start-master.sh
```

##### 步骤6: 查看 master 的状态

![master1](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-HA-master1.png)

![master2](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-HA-master2.png)

##### 步骤7： 杀死 hadoop201 的 master 进程

hadoop202 的 master 会自动切换成 Active

![master3](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-HA-master3.png)

#### 2.3.6 Standalone 工作模式图解

![Standalone 工作模式图解](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark_standalone_工作图解.png)

### 2.4 Yarn模式

#### 2.4.1 Yarn 模式概述

Spark 客户端可以直接连接 Yarn，不需要额外构建Spark集群。

有 client 和 cluster 两种模式，主要区别在于：Driver 程序的运行节点不同。

* client：Driver程序运行在客户端，适用于交互、调试，希望立即看到app的输出

* cluster：Driver程序运行在由 RM（ResourceManager）启动的 AM（AplicationMaster）上, 适用于生产环境。

工作模式介绍:

![Yarn运行模式](https://gitee.com/zhdoop/blogImg/raw/master/img/Yarn运行模式.png)

#### 2.4.2 Yarn 模式配置

##### 步骤1：修改 hadoop 配置文件 yarn-site.xml, 添加如下内容：

由于咱们的测试环境的虚拟机内存太少, 防止将来任务被意外杀死, 配置所以做如下配置.

```xml
<!--是否启动一个线程检查每个任务正使用的物理内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
<property>
    <name>yarn.nodemanager.pmem-check-enabled</name>
    <value>false</value>
</property>
<!--是否启动一个线程检查每个任务正使用的虚拟内存量，如果任务超出分配值，则直接将其杀掉，默认是true -->
<property>
    <name>yarn.nodemanager.vmem-check-enabled</name>
    <value>false</value>
</property>
```

修改后分发配置文件

##### 步骤2: 复制 spark, 并命名为spark-yarn

```shell
cp -r spark-standalone spark-yarn
```

##### 步骤3: 修改spark-evn.sh文件

去掉 master 的 HA 配置, 日志服务的配置保留着.

并添加如下配置: 告诉 spark 客户端 yarn 相关配置

```shell
YARN_CONF_DIR=/opt/module/hadoop-2.7.2/etc/hadoop
```

##### 步骤4: 执行一段程序

```shell
bin/spark-submit \
--class org.apache.spark.examples.SparkPi \
--master yarn \
--deploy-mode client \
./examples/jars/spark-examples_2.11-2.1.1.jar 100
```

http://hadoop202:8088

![web-ui](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Yarn-webui.png)

#### 2.4.3 日志服务

在前面的页面中点击 history 无法直接连接到 spark 的日志.

可以在spark-default.conf中添加如下配置达到上述目的

```shell
spark.yarn.historyServer.address=hadoop201:18080
spark.history.ui.port=18080
```

![history1](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Yarn-history-ui1.png)

![history2](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Yarn-history-ui2.png)

可能碰到的问题:

如果在 yarn 日志端无法查看到具体的日志, 则在yarn-site.xml中添加如下配置

![问题](https://gitee.com/zhdoop/blogImg/raw/master/img/Spark-Yarn-history-question.png)

```xml
<property>
    <name>yarn.log.server.url</name>
    <value>http://hadoop201:19888/jobhistory/logs</value>
</property>
```

### 2.5 Mesos 模式

Spark客户端直接连接 Mesos；不需要额外构建 Spark 集群。

国内应用比较少，更多的是运用yarn调度。

### 2.6 几种运行模式的对比

| 模式       | Spark 安装机器数 | 需启动的进程   | 所属者 |
| ---------- | ---------------- | -------------- | ------ |
| Local      | 1                | 无             | Spark  |
| Standalone | 多台             | Master及Worker | Spark  |
| Yarn       | 1                | Yarn及HDFS     | Hadoop |



## 3 案例实操

Spark Shell 仅在测试和验证我们的程序时使用的较多，在生产环境中，通常会在 IDE 中编制程序，然后打成 jar 包，然后提交到集群，最常用的是创建一个 Maven 项目，利用 Maven 来管理 jar 包的依赖。

### 3.1 编写 WordCount 程序

#### 步骤1:创建 maven 项目, 导入依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.spark</groupId>
        <artifactId>spark-core_2.11</artifactId>
        <version>2.1.1</version>
    </dependency>
</dependencies>
<build>
    <plugins>
        <!-- 打包插件, 否则 scala 类不会编译并打包进去 -->
        <plugin>
            <groupId>net.alchim31.maven</groupId>
            <artifactId>scala-maven-plugin</artifactId>
            <version>3.4.6</version>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>testCompile</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

#### 步骤2: 创建WordCount.scala

```scala
object WordCount {
    def main(args: Array[String]): Unit = {
        // 1. 创建 SparkConf对象, 并设置 App名字
        val conf: SparkConf = new SparkConf().setAppName("WordCount")
        // 2. 创建SparkContext对象
        val sc = new SparkContext(conf)
        // 3. 使用sc创建RDD并执行相应的transformation和action
        sc.textFile("/input")
            .flatMap(_.split(" "))
            .map((_, 1))
            .reduceByKey(_ + _)
            .saveAsTextFile("/result")
        // 4. 关闭连接
       sc.stop()
    }
}
```

### 3.2 测试

#### 3.2.1 测试1: 打包到 Linux 测试

使用 maven 的打包命令打包. 然后测试

```shell
bin/spark-submit --class day01.WordCount --master yarn input/spark_test-1.0-SNAPSHOT.jar
```

#### 3.2.2 测试2: idea 本地直接提交应用

使用 local 模式执行，相当于代码是在 window 下执行的.

```scala
package day01

import org.apache.spark.{SparkConf, SparkContext}

object WordCount {
    def main(args: Array[String]): Unit = {
        // 1. 创建 SparkConf对象, 并设置 App名字, 并设置为 local 模式
        val conf: SparkConf = new SparkConf().setAppName("WordCount").setMaster("local[*]")
        // 2. 创建SparkContext对象
        val sc = new SparkContext(conf)
        // 3. 使用sc创建RDD并执行相应的transformation和action
        val wordAndCount: Array[(String, Int)] = sc.textFile(ClassLoader.getSystemResource("words.txt").getPath)
            .flatMap(_.split(" "))
            .map((_, 1))
            .reduceByKey(_ + _)
            .collect()
        wordAndCount.foreach(println)
        // 4. 关闭连接
        sc.stop()
    }
}
```



