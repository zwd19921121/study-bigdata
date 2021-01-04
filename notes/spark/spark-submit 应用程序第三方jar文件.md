# spark-submit 应用程序第三方jar文件

## 1 第一种方式：打包到jar应用程序

**操作：**将第三方jar文件打包到最终形成的spark应用程序jar文件中

**应用场景：**第三方jar文件比较小，应用的地方比较少

将依赖打入jar的方式：https://blog.csdn.net/xiao__gui/article/details/47341385#

## 2 第二种方式：spark-submit 参数 --jars

**操作:** 使用spark-submit提交命令的参数: --jars

要求：

1、使用spark-submit命令的机器上存在对应的jar文件

2、至于集群中其他机器上的服务需要该jar文件的时候，通过driver提供的一个http接口来获取该jar文件的(例如：http://192.168.187.146:50206/jars/mysql-connector-java-5.1.27-bin.jar Added By User)

```shell
## 配置参数：--jars JARS
如下示例：
$ bin/spark-shell --jars /opt/cdh-5.3.6/hive/lib/mysql-connector-java-5.1.27-bin.jar
```

**应用场景：**要求本地必须要有对应的jar文件



## 3 第三种方式：spark-submit 参数 --packages

**操作：**使用spark-submit提交命令的参数: --packages

```shell
## 配置参数：--packages  jar包的maven地址
如下示例：
$ bin/spark-shell --packages  mysql:mysql-connector-java:5.1.27 --repositories http://maven.aliyun.com/nexus/content/groups/public/

## --repositories 为mysql-connector-java包的maven地址，若不给定，则会使用该机器安装的maven默认源中下载
## 若依赖多个包，则重复上述jar包写法，中间以逗号分隔
## 默认下载的包位于当前用户根目录下的.ivy/jars文件夹中
```

**应用场景：**本地可以没有，集群中服务需要该包的的时候，都是从给定的maven地址，直接下载

## 4 第四种方式：添加到spark的环境变量

**操作：**更改Spark的配置信息:SPARK_CLASSPATH, 将第三方的jar文件添加到SPARK_CLASSPATH环境变量中

**注意事项**：要求Spark应用运行的所有机器上必须存在被添加的第三方jar文件

```shell
A.创建一个保存第三方jar文件的文件夹:
 命令：$ mkdir external_jars

B.修改Spark配置信息
 命令：$ vim conf/spark-env.sh
修改内容：SPARK_CLASSPATH=$SPARK_CLASSPATH:/opt/cdh-5.3.6/spark/external_jars/*

C.将依赖的jar文件copy到新建的文件夹中
命令：$ cp /opt/cdh-5.3.6/hive/lib/mysql-connector-java-5.1.27-bin.jar ./external_jars/
```

**应用场景：**依赖的jar包特别多，写命令方式比较繁琐，被依赖包应用的场景也多的情况下

备注：（只针对spark on yarn(cluster)模式）

**spark on yarn(cluster)，如果应用依赖第三方jar文件**

**最终解决方案：**将第三方的jar文件copy到${HADOOP_HOME}/share/hadoop/common/lib文件夹中(Hadoop集群中所有机器均要求copy)