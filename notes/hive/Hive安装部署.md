# Hive安装

## 1 Hive 安装部署

1. 把apache-hive-3.1.2-bin.tar.gz上传到 linux的/opt/software目录下

2. 解压 apache-hive-3.1.2-bin.tar.gz 到/opt/module/目录下面

   ```shell
   tar -zxvf /opt/software/apache-hive-3.1.2-bin.tar.gz -C /opt/module/
   ```

3. 修改apache-hive-3.1.2-bin 的名称为hive

   ```shell
   mv /opt/module/apache-hive-3.1.2-bin/ /opt/module/hive
   ```

4.  修改/etc/profile.d/myenv.sh，添加环境变量

   ```shell
   sudo vim /etc/profile.d/my_env.sh
   ```

   添加内容

   ```shell
   #HIVE_HOME 
   export HIVE_HOME=/opt/module/hive 
   export PATH=$PATH:$HIVE_HOME/bin
   ```

   source一下/etc/profile.d/my_env.sh

   ```shell
   source /etc/profile.d/my_env.sh
   ```

5. 解决日志Jar包冲突，进入/opt/module/hive/lib 目录

   ```shell
   mv log4j-slf4j-impl-2.10.0.jar log4j-slf4j-impl-2.10.0.jar.bak
   ```

## 2 Hive 元素据配置到MySQL

### 2.1 拷贝驱动

将 MySQL 的 JDBC 驱动拷贝到 Hive 的 lib 目录下 

```shell
 cp /opt/software/mysql-connector-java-5.1.48.jar /opt/module/hive/lib/
```

### 2.2 配置Metastore 到 MySQL

在$HIVE_HOME/conf 目录下新建 hive-site.xml 文件 

```shell
vim hive-site.xml
```

添加如下内容

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?> 
<configuration> 
    <property> 
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://node1:3306/metastore?useSSL=false</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionUserName</name> 
        <value>root</value> 
    </property> 
    <property> 
        <name>javax.jdo.option.ConnectionPassword</name> 
        <value>000000</value>
    </property> 
    <property> 
        <name>hive.metastore.warehouse.dir</name> 
        <value>/user/hive/warehouse</value> 
    </property> 
    <property> 
        <name>hive.metastore.schema.verification</name> 
        <value>false</value> 
    </property> 
    <property> 
        <name>hive.server2.thrift.port</name> 
        <value>10000</value> 
    </property>
    <property> 
        <name>hive.server2.thrift.bind.host</name> 
        <value>node1</value> 
    </property> 
    <property> 
        <name>hive.metastore.event.db.notification.api.auth</name> 
        <value>false</value>
    </property>
    <property> 
        <name>hive.cli.print.header</name>
        <value>true</value> 
    </property> 
    <property> 
        <name>hive.cli.print.current.db</name> 
        <value>true</value>
    </property> 
</configuration>
```

## 3 启动hive 客户端

### 3.1初始化元数据库

1. 登录MySQL

   ```shell
   mysql -uroot -p000000
   ```

2. 新建Hive元数据库

   ```shell
   mysql> create database metastore; 
   mysql> quit;
   ```

3. 初始化元数据库

   ```shell
   schematool -initSchema -dbType mysql -verbose
   ```

### 3.2 启动hive客户端

1. 启动Hive客户端

   ```shell
   bin/hive
   ```

2. 查看一下数据库

   ```shell
   hive (default)> show databases; 
   OK
   database_name 
   default
   ```

   

