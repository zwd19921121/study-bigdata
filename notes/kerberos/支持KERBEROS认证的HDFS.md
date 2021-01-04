# 支持Kerberos认证的HDFS

## 1 HDFS开启Kerberos认证

### 1.1 开启kerberos认证

### 1.2 验证

#### 1.2.1 客户端命令验证

```shell
[asap@asap185 ~]$  hadoop fs -ls /
20/11/24 15:28:53 WARN security.UserGroupInformation: PriviledgedActionException as:asap (auth:KERBEROS) cause:javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
20/11/24 15:28:53 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
20/11/24 15:28:53 WARN security.UserGroupInformation: PriviledgedActionException as:asap (auth:KERBEROS) cause:java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
ls: Failed on local exception: java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]; Host Details : local host is: "asap185/192.168.1.185"; destination host is: "asap184":8020;
```

#### 1.2.2 WEB-UI环境认证

![web-ui](https://gitee.com/zhdoop/blogImg/raw/master/img/kerberos_hdfs_web-ui.png)

## 2 访问Kerberos认证的HDFS

### 2.1 客户端环境认证

**1）在Kerberos 数据库中创建用户主体/实例**

```shell
[asap@asap185 ~]# sudo kadmin.local -q "addprinc asap/asap@HADOOP.COM"
```

**2）进行用户认证**（会过期，过期后需要重新执行该命令）

```shell
[asap@asap185 ~]# kinit asap/asap@HADOOP.COM
```

**3）访问HDFS**

```shell
[asap@asap185 ~]# hadoop fs -ls /
Found 3 items
drwx------   - hdfs supergroup          0 2019-11-21 21:25 /livy
drwxrwxrwt   - hdfs supergroup          0 2020-10-29 16:34 /tmp
drwxr-xr-x   - hdfs supergroup          0 2019-11-21 21:26 /user
```

如果不进行认证，会报错

```shell
[asap@asap185 ~]$  hadoop fs -ls /
20/11/24 15:28:53 WARN security.UserGroupInformation: PriviledgedActionException as:asap (auth:KERBEROS) cause:javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
20/11/24 15:28:53 WARN ipc.Client: Exception encountered while connecting to the server : javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
20/11/24 15:28:53 WARN security.UserGroupInformation: PriviledgedActionException as:asap (auth:KERBEROS) cause:java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]
ls: Failed on local exception: java.io.IOException: javax.security.sasl.SaslException: GSS initiate failed [Caused by GSSException: No valid credentials provided (Mechanism level: Failed to find any Kerberos tgt)]; Host Details : local host is: "asap185/192.168.1.185"; destination host is: "asap184":8020;
```

**4）Hive查询**

```shell
[asap@asap185 ~]$ hive
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was remov                                                              ed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was remov                                                              ed in 8.0

Logging initialized using configuration in jar:file:/data/cloudera/parcels/CDH-5.9.2-1.cdh5.9.                                                              2.p0.3/jars/hive-common-1.1.0-cdh5.9.2.jar!/hive-log4j.properties
WARNING: Hive CLI is deprecated and migration to Beeline is recommended.
hive>
```

### 2.2 java代码认证（凭证无法实现自动续期）

```java
public class FileSystemCat{
    public static void main(String[] args) throws Exception{
        Configuration conf = new Configuration();
        conf.set("fs.defaultFS","hdfs://asap184:8020");
        conf.setBoolean("hadoop.security.authentication",true);
        conf.set("hadoop.security.authentication","kerberos");
        //conf.set("dfs.client.use.datanode.hostname","true"); //如果从外网访问内网的hdfs集群需要加
        
        System.setProperty("java.security.krb5.conf","D:/krb5.conf");
        UserGroupInformation.setConfiguration(conf);
        UserGroupInformation.loginUserFromKeytab("hdfs/hdfs","D:/hdfs.keytab");
        
        FileSystem fs = FileSystem.get(conf);
        RemoteIterator<LocatedFileStatus> listFiles = fs.listFiles(new Path("/"),true);
        
        //遍历迭代器
        while(listFiles.hasNext()){
            LocatedFileStatus fileStatus = listFiles.next();
             //打印当前文件的名字和大小
            System.out.println(fileStatus.getPath().getName());
            System.out.println(fileStatus.getBlockSize());
        } 
        
        fs.close();
    }
}
```

### 2.3 spark 认证

#### 2.3.1 java代码方式认证（凭证无法实现自动续期）

```java
object Main {
  def main(args: Array[String]): Unit = {
    System.setProperty("java.security.krb5.conf", "D:/krb5.conf")
    val spark = SparkSession
      .builder()
      .appName("testHdfs")
      .master("local[2]")
      .getOrCreate()

     System.setProperty("java.security.krb5.conf", "D:/krb5.conf")
     spark.sparkContext.hadoopConfiguration.set("dfs.client.use.datanode.hostname","true")
     spark.sparkContext.hadoopConfiguration.set("hadoop.security.authorization","true")
     spark.sparkContext.hadoopConfiguration.set("hadoop.security.authentication", "kerberos")
     spark.sparkContext.hadoopConfiguration.set("dfs.namenode.kerberos.principal", "hdfs/_HOST@HADOOP.COM")

      //用户登录
      UserGroupInformation.setConfiguration(spark.sparkContext.hadoopConfiguration)
      UserGroupInformation.loginUserFromKeytab("hdfs/hdfs", "D:/hdfs.keytab")
      val sourceRDD = spark.read
      .text(
      "hdfs://node001:8020/test/*").rdd
    sourceRDD.foreach(println)
  }
}
```

#### 2.3.2 spark 参数方式认证（凭证可以自动续期，yarn模式下）

```shell
spark-submit \
--keytab /root/hdfs.keytab \
--principal "hdfs/hdfs@HADOOP.COM" \
--files /etc/krb5.conf \
--num-executors 2 \
--master yarn \
--deploy-mode client \
--driver-memory 1g \
--class org.asiainfo.com.spark.hdfs.APP \
spark-1.0-SNAPSHOT.jar
--conf spark.hadoop.fs.hdfs.impl.disable.cache=true 
```

* --conf spark.hadoop.fs.hdfs.impl.disable.cache=true  不加好像不能实现自动刷新凭证

  也可以在core-site.xml 加入如下参数

  ```xml
  <property>
  	<name>fs.hdfs.impl.disable.cache</name>
  	<value>true</value>
  </property>
  ```

### 2.4 spark-jobserver 认证

见附件。

