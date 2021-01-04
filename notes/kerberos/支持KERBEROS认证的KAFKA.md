# 支持Kerberos认证的KAFKA

## 1 KAFKA开启Kerberos认证

### 1.1 开启kerberos认证

除了CDH开启之外，还需要修改kafka如下的配置。

（1）在 Kafka 的配置项搜索“security.inter.broker.protocol”，设置为 SALS_PLAINTEXT

（2）在 Kafka 的配置项搜索“ssl.client.auth”，设置为 none。

### 1.2 验证

#### 客户端命令验证

```shell
[asap@asap185 ~]$ kafka-console-consumer  --bootstrap-server asap184:9092 --topic test --from-beginning
[2020-12-01 17:40:20,464] WARN Bootstrap broker asap184:9092 disconnected (org.apache.kafka.clients.NetworkClient)
[2020-12-01 17:40:20,517] WARN Bootstrap broker asap184:9092 disconnected (org.apache.kafka.clients.NetworkClient)
[2020-12-01 17:40:20,569] WARN Bootstrap broker asap184:9092 disconnected (org.apache.kafka.clients.NetworkClient)
```

从错误可以看出kafka 消费者连接不上。

## 2 访问Kerberos认证的KAFKA

### 2.1 客户端环境认证

**1）在Kerberos 数据库中创建用户主体/实例**

```shell
[asap@asap185 ~]# sudo kadmin.local -q "addprinc asap/asap@HADOOP.COM"
```

**2）创建jass.conf文件**

```shell
[asap@asap184 ~]$ vim /var/lib/kafka/jaas.conf
```

文件内容如下：

```
KafkaClient {
com.sun.security.auth.module.Krb5LoginModule required
useTicketCache=true;
};
```

**3）创建cosumer.properties文件**

```shell
[asap@asap185 ~]# vim /etc/kafka/conf/consumer.properties
```

文件内容如下：

```
security.protocol=SASL_PLAINTEXT
sasl.kerberos.service.name=kafka
```

**4）声明jaas.conf文件路径**

```shell
[asap@asap184 ~]$ export KAFKA_OPTS="-Djava.security.auth.login.config=/var/lib/kafka/jaas.conf"
```

**5）使用kafka-console-consumer消费Kafka topic数据**

```shell
[asap@asap184 ~]$ kafka-console-consumer  --bootstrap-server asap184:9092 --topic test --from-beginning --consumer.config /etc/kafka/conf/consumer.properties
213
123
123
213
wqe
sa
```

### 2.2 java代码认证（凭证无法实现自动续期）

#### 2.2.1 MyProperties 

```java
public class MyProperties extends Properties {
    private Properties properties;

    private static final String JAAS_TEMPLATE =
            "KafkaClient {\n"
                    + "com.sun.security.auth.module.Krb5LoginModule required\n" +
                    "useKeyTab=true\n" +
                    "keyTab=\"%1$s\"\n" +
                    "principal=\"%2$s\";\n"
                    + "};";


    public MyProperties(){
        properties = new Properties();
    }

    public MyProperties self(){
        return this;
    }

    public MyProperties put(String key , String value) {
        if (properties == null) {
            properties = new Properties();
        }

        properties.put(key, value);
        return self();
    }

    public static MyProperties initKerberos(){
        return new MyProperties()
                .put(ConsumerConfig.GROUP_ID_CONFIG, "DemoConsumer")
                .put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                        "org.apache.kafka.common.serialization.StringDeserializer")
                .put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                        "org.apache.kafka.common.serialization.StringDeserializer")
                .put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true")
                .put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "1000")
                .put("security.protocol", "SASL_PLAINTEXT")
                .put("sasl.kerberos.service.name", "kafka");

    }

    public static MyProperties initProducer(){
        return new MyProperties()
                .put(ProducerConfig.ACKS_CONFIG, "all")
                .put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer")
                .put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer")
                .put("security.protocol", "SASL_PLAINTEXT")
                .put("sasl.kerberos.service.name", "kafka");
    }

    public Properties getProperties() {
        return properties;
    }

    //生成jaas.conf临时文件
    public static void configureJAAS(String keyTab, String principal) {
        String content = String.format(JAAS_TEMPLATE, keyTab, principal);

        File jaasConf = null;
        PrintWriter writer = null;

        try {

            jaasConf  = File.createTempFile("jaas", ".conf");
            writer = new PrintWriter(jaasConf);

            writer.println(content);

        } catch (IOException e) {
            // TODO Auto-generated catch block
            e.printStackTrace();

        } finally {

            if (writer != null) {
                writer.close();
            }

            jaasConf.deleteOnExit();
        }

        System.setProperty("java.security.auth.login.config", jaasConf.getAbsolutePath());
    }
}
```

#### 2.2.2 Consumer

```java
public class Comsumer {
    private static String TOPIC_NAME = "test";

    public static void main(String[] args) {

        System.setProperty("java.security.krb5.conf", Thread.currentThread()
                .getContextClassLoader().getResource("krb5.conf").getPath());
        //初始化jaas.conf文件
        MyProperties.configureJAAS(Thread.currentThread().getContextClassLoader().getResource("asap.keytab").getPath(), "asap/asap@HADOOP.COM");

        System.setProperty("javax.security.auth.useSubjectCredsOnly", "false");

        MyProperties props = MyProperties.initKerberos();

        props.put(
                ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                "asap184:9092");

        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(
                props.getProperties());
        consumer.subscribe(Arrays.asList(TOPIC_NAME));
        /*
         * TopicPartition partition0= new TopicPartition(TOPIC_NAME, 0);
         *
         * TopicPartition partition1= new TopicPartition(TOPIC_NAME, 1);
         *
         * TopicPartition partition2= new TopicPartition(TOPIC_NAME, 2);
         */

        // consumer.assign(Arrays.asList(partition0,partition1, partition2));

        ConsumerRecords<String, String> records = null;

        while (true) {
            try {
                Thread.sleep(1000);

                System.out.println();
                records = consumer.poll(Long.MAX_VALUE);

                for (ConsumerRecord<String, String> record : records) {

                    System.out.println("Receivedmessage: (" + record.key()
                            + "," + record.value() + ") at offset "
                            + record.offset());

                }

            } catch (InterruptedException e) {
                // TODO Auto-generated catch block
                e.printStackTrace();
            }
        }
        /*
         * while (true){
         *
         * try {
         *
         * Thread.sleep(10000l);
         *
         * System.out.println();
         *
         * records = consumer.poll(Long.MAX_VALUE);
         *
         * for (ConsumerRecord<String, String> record : records) {
         *
         * System.out.println("Receivedmessage: (" + record.key() + "," +
         * record.value() + ") at offset " + record.offset());
         *
         * }
         *
         * } **catch** (**InterruptedException** e){
         *
         * e.printStackTrace();
         *
         * }
         */
    }
}
```

#### 2.2.3 Producer

```shell
public class Producer {
    //发送topic
    public static String TOPIC_NAME = "test";

    public static void main(String[] args) {

        System.setProperty("java.security.krb5.conf",
                Thread.currentThread().getContextClassLoader().getResource("krb5.conf").getPath());
        //初始化jaas.conf文件
        MyProperties.configureJAAS(Thread.currentThread().getContextClassLoader().getResource("asap.keytab").getPath(), "asap/asap@HADOOP.COM");
        System.setProperty("javax.security.auth.useSubjectCredsOnly", "false");

        //System.setProperty("sun.security.krb5.debug","true");

        //初始化kerberos环境
        MyProperties props = MyProperties.initProducer();

        //kafka brokers地址
        props.put(
                ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                "asap184:9092");

        org.apache.kafka.clients.producer.Producer<String, String> producer = new KafkaProducer<String, String>(
                props.getProperties());
        for (int i = 0; i < 10; i++) {

            String key = "key-" + i;

            String message = "{\"id\":\"hello\"}";

            ProducerRecord record = new ProducerRecord<String, String>(
                    TOPIC_NAME, key, message);

            producer.send(record);

            System.out.println(key + "----" + message);

        }

        producer.close();

    }
}
```

#### 2.2.4 pom

```xml
 <dependencies>
        <dependency>
            <groupId>org.apache.kafka</groupId>
            <artifactId>kafka_2.11</artifactId>
            <version>2.2.0</version>
        </dependency>
    </dependencies>
```

### 2.3 spark 认证

#### 2.3.1 java代码修改

需要增加如下三个参数。

```java
 resultMap.+=("security.protocol" -> "SASL_PLAINTEXT")
 resultMap.+=("sasl.mechanism" -> "GSSAPI")
 resultMap.+=("sasl.kerberos.service.name" -> getRequiredPropBy("sasl-kerberos-service"))
```

#### 2.3.2 submit 参数配置（凭证可以自动续期，yarn模式下）

```shell
/data/cloudera/parcels/SPARK2/bin/spark2-submit  \
--master yarn \
--deploy-mode client \
--executor-memory 2g \
--executor-cores 2 \
--driver-memory 2g \
--num-executors 2 \
--queue default  \
--principal "asap/asap@HADOOP.COM" \
--keytab ./asap.keytab \
--files "/data/asap/asap-mamd/job-server/conf/jaas.conf" \
--driver-java-options "-Djava.security.auth.login.config=/data/asap/asap-mamd/job-server/conf/jaas.conf" \
--conf "spark.executor.extraJavaOptions=-Djava.security.auth.login.config=./jaas.conf" \
--class org.asiainfo.com.spark.kafka.ScalaKafkaStreamingYarn \
/data/asap/asap-mamd/job-server/conf/spark-1.0-SNAPSHOT.jar
```

* --conf spark.hadoop.fs.hdfs.impl.disable.cache=true  不加好像不能实现自动刷新凭证

  也可以在core-site.xml 加入如下参数

  ```xml
  <property>
  	<name>fs.hdfs.impl.disable.cache</name>
  	<value>true</value>
  </property>
  ```

#### 2.3.3 jaas.conf

```
KafkaClient {
com.sun.security.auth.module.Krb5LoginModule required
  useTicketCache=false
  useKeyTab=true
  principal="asap/asap@HADOOP.COM"
  keyTab="/data/asap/asap-mamd/job-server/conf/asap.keytab"
  renewTicket=true
  storeKey=true;
};
```

或者

```shell
KafkaClient {
 com.sun.security.auth.module.Krb5LoginModule required
 useKeyTab=false
 useTicketCache=true
 renewTicket=true;
};
```

### 2.4 spark-jobserver 认证

见附件





