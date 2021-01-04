# CDH启用Kerberos认证

## 1 环境准备

以三节点为例

<table>
    <tr>
        <th>节点</th>
        <th>192.168.1.184</th>
        <th>192.168.1.185</th>
        <th>192.168.1.186</th>
    </tr>
     <tr>
        <td>主机名</th>
        <td>asap184</td>
        <td>asap185</td>
        <td>asap186</td>
    </tr>
 	<tr>
        <td>服务</th>
        <td>cloudera-scm-server<br/>cloudera-scm-agent<br/>kerberos client</td>
        <td>cloudera-scm-agent<br/>kerberos server</td>
        <td>cloudera-scm-agent<br/>kerberos client</td>
    </tr>
</table>

## 2 CDH安装

## 3 Kerberos 安装

### 3.1 server节点安装kerberos 相关软件

```shell
[asap@asap185 ~]# sudo yum install -y krb5-server krb5-workstation krb5-libs
#查看结果
[asap@asap185 ~]# sudo rpm -qa | grep krb5
krb5-workstation-1.10.3-65.el6.x86_64
krb5-libs-1.10.3-65.el6.x86_64
krb5-server-1.10.3-65.el6.x86_64
```

### 3.2 client 节点安装

```shell
[asap@asap184 ~]# sudo yum install -y krb5-workstation krb5-libs
[asap@asap186 ~]# sudo yum install -y krb5-workstation krb5-libs
#查看结果
[asap@asap184 ~]# sudo rpm -qa | grep krb5
krb5-workstation-1.10.3-65.el6.x86_64
krb5-libs-1.10.3-65.el6.x86_64
```

### 3.3 配置kerberos

需要配置的文件有两个为 kdc.conf 和 krb5.conf , 配置只是需要 **Server** 服务节点配置，即 asap185。

1）kdc 配置

<table bgcolor=#F2F2F2>
    <tr>
    	<td>
            [asap@asap185 ~]# sudo vim /var/kerberos/krb5kdc/kdc.conf <br/>
			[kdcdefaults]  <br/>
			kdc_ports = 88 <br/>
			kdc_tcp_ports = 88 <br/>
			[realms] <br/>
            <font color=red>HADOOP.COM</font> = { <br/>
 				#master_key_type = aes256-cts<br/>
 				acl_file = /var/kerberos/krb5kdc/kadm5.acl<br/>
 				dict_file = /usr/share/dict/words<br/>
 				admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab<br/>
 				<font color=red>max_life = 1d<br/>
 				max_renewable_life = 7d</font><br/>
            supported_enctypes = <font color=red>aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal</font><br/>
} <br/>
        </td>
    </tr>
</table>

 说明：

**HADOOP.COM**: realm名称，Kerberos支持多个realm，一般全写大写。

**acl_file**: admin的用户权限

**admin_keytab**: KDC进行校验的keytab

**supported_enctypes**: 支持的校验方式，注意把<font color=red>aes256-cts</font>去掉，JAVA使用 aes256-cts 验证方式需要安装额外的jar包，所以这里不用。

2）krb5文件配置

<table bgcolor=#F2F2F2>
    <tr>
    	<td>
            [asap@asap185 ~]# sudo vim /etc/krb5.conf <br/>
			includedir /etc/krb5.conf.d/  <br/>
			 <br/>
			[logging] <br/>
            default = FILE:/var/log/krb5libs.log <br/>
            kdc = FILE:/var/log/krb5kdc.log <br/>
            admin_server = FILE:/var/log/kadmind.log <br/>
            <br/>
            [libdefaults] <br/>
            dns_lookup_realm = false <br/>
            ticket_lifetime = 24h <br/>
            renew_lifetime = 7d <br/>
            forwardable = true <br/>
            rdns = false <br/>
            pkinit_anchors = /etc/pki/tls/certs/ca-bundle.crt <br/>
            <font color=red>default_realm = HADOOP.COM </font><br/>
            <font color=red>#default_ccache_name = KEYRING:persistent:%{uid}</font><br/>
            <font color=red>udp_preference_limit = 1</font>
            <br/>
            [realms] <br/>
            <font color=red> HADOOP.COM </font> = { <br/>
            kdc = <font color=red>asap185</font> <br/>
            admin_server = <font color=red>asap185</font> <br/>
            } <br/>
            <br/>
            [domain_realm] <br/>
            # .example.com = EXAMPLE.COM<br/>
            # example.com = EXAMPLE.COM <br/>            
        </td>
    </tr>
</table>

说明：

**default_realm**:  默认的realm，设置Kerberos 应用程序的默认领域，必须跟要配置的realm的名称一致。

**ticket_lifetime**: 表明凭证生效时限，一般为 24 小时。

**renew_lifetime**: 表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败。

**udp_preference_limit= 1**:  禁止使用 udp，可以防止一个 Hadoop 中的错误。

**realms**: 配置使用的 realm，如果有多个领域，只需向 [realms] 节添加其他的语句。

**domain_realm**: 集群域名与 Kerberosrealm 的映射关系，单个 realm 的情况下，可忽略。

3）同步krb5到Client节点

```shell
[asap@asap185 ~]# sudo xsync /etc/krb5.conf
```

### 3.4 生成Kerberos 数据库

在 server 节点执行

```shell
[asap@asap185 ~]# sudo kdb5_util create -s
Loading random data
Initializing database '/var/kerberos/krb5kdc/principal' for realm 'HADOOP.COM',
master key name 'K/M@HADOOP.COM'
You will be prompted for the database Master Password.
It is important that you NOT FORGET this password.
Enter KDC database master key: （输入密码）
Re-enter KDC database master key to verify:（确认密码）
```

创建完成后/var/kerberos/krb5kdc 目录下会生成对应的文件

<table bgcolor=#F2F2F2>
    <tr>
    	<td>
        	[asap@asap185 ~]# ls /var/kerberos/krb5kdc/ <br/>
            kadm5.acl kdc.conf <font color=red>principal principal.kadm5 principal.kadm5.lock principal.ok</font>
        </td>
    </tr>
</table>

### 3.5 赋予Kerberos 管理员所有权限

<table bgcolor=#F2F2F2>
    <tr>
    	<td>
        	[asap@asap185 ~]# sudo vim /var/kerberos/krb5kdc/kadm5.acl <br/>
            #修改为以下内容： <br/>
            <font color=red>*/admin@HADOOP.COM *</font>
        </td>
    </tr>
</table>

说明：

***/admin**： admin 实例的全部主体

**@HADOOP.COM**: realm

*****: 全部权限

这个授权的意思：就是授予 admin 实例的全部主体对应 HADOOP.COM 领域的全部权限。也就是创建 Kerberos 主体的时候如果实例为 admin,就具有 HADOOP.COM 领域的全部权限，比如创建如下的主体 user1/admin 就拥有全部的 HADOOP.COM 领域的权限。

### 3.6 启动Kerberos服务

```shell
#启动 krb5kdc
[asap@asap185 ~]# sudo systemctl start krb5kdc
正在启动 Kerberos 5 KDC： [确定] #启动 kadmin
[asap@asap185 ~]# sudo systemctl start kadmin
正在启动 Kerberos 5 Admin Server： [确定] #设置开机自启
[asap@asap185 ~]# sudo systemctl enable krb5kdc
#查看是否设置为开机自启
[asap@asap185 ~]# sudo systemctl is-enabled krb5kdc
[asap@asap185 ~]# sudo systemctl enable kadmin
#查看是否设置为开机自启
[asap@asap185 ~]# sudo systemctl is-enabled kadmin
```

注意：启动失败时可以通过/var/log/krb5kdc.log 和/var/log/kadmind.log 来查看。



## 4 Kerberos 数据库操作

### 4.1 登录 Kerberos 数据库

1）本地登录（无需认证）

```shell
[asap@asap185 ~]# kadmin.local 
Authenticating as principal root/admin@HADOOP.COM with password.
kadmin.local:
```

2）远程登录（需进行主体认证）

```shell
[asap@asap185 ~]# kadmin
Authenticating as principal admin/admin@HADOOP.COM with password.
Password for admin/admin@HADOOP.COM: 
kadmin:
```

退出输入：exit

### 4.2 创建Kerberos 主体

```shell
[asap@asap185 ~]$ sudo kadmin.local -q "addprinc asap/asap"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for asap/asap@HADOOP.COM; defaulting to no policy
Enter password for principal "asap/asap@HADOOP.COM":
Re-enter password for principal "asap/asap@HADOOP.COM":
Principal "asap/asap@HADOOP.COM" created.
```

### 4.3 修改主体的密码

```shell
[asap@asap185 ~]$ sudo kadmin.local -q "cpw asap/asap"
Authenticating as principal root/admin@HADOOP.COM with password.
Enter password for principal "asap/asap@HADOOP.COM":
Re-enter password for principal "asap/asap@HADOOP.COM":
Password for "asap/asap@HADOOP.COM" changed.
```

### 4.4 查看所有主体

```shell
[asap@asap185 ~]# sudo kadmin.local -q "list_principals"
Authenticating as principal root/admin@HADOOP.COM with password.
K/M@HADOOP.COM
asap/asap@HADOOP.COM
kadmin/admin@HADOOP.COM
kadmin/changepw@HADOOP.COM
kadmin/hadoop105@HADOOP.COM
kiprop/hadoop105@HADOOP.COM
krbtgt/HADOOP.COM@HADOOP.COM
```

## 5 Kerberos 主体认证

Kerberos 提供了两种认证方式，一种是通过输入密码认证，另一种是通过 keytab 密钥文件认证，但两种方式不可同时使用。

### 5.1 密码认证

1）使用 kinit 进行主体认证

```shell
[asap@asap184 ~]# kinit asap/asap
Password for asap/asap@HADOOP.COM:
```

2）查看认证凭证

```shell
[asap@asap184 ~]$ klist
Ticket cache: FILE:/tmp/krb5cc_600
Default principal: asap/asap@HADOOP.COM

Valid starting       Expires              Service principal
11/23/2020 17:53:33  11/24/2020 17:53:33  krbtgt/HADOOP.COM@HADOOP.COM
        renew until 11/30/2020 17:53:33
```

### 5.2 keytab 密钥文件认证

1）生成主体 admin/admin 的 keytab 文件到指定目录/root/admin.keytab

```shell
[asap@asap185 ~]# sudo kadmin.local -q "xst -k ./asap.keytab asap/asap@HADOOP.COM"
```

2）使用 keytab 进行认证

```shell
#需要注意asap.keytab 的权限，如果不符合得修改
[asap@asap185 ~]# kinit -kt ./asap.keytab asap/asap
```

3）查看认证凭证

```shell
[asap@asap184 ~]$ klist
Ticket cache: FILE:/tmp/krb5cc_600
Default principal: asap/asap@HADOOP.COM

Valid starting       Expires              Service principal
11/23/2020 17:53:33  11/24/2020 17:54:23  krbtgt/HADOOP.COM@HADOOP.COM
        renew until 11/30/2020 17:54:23
```

### 5.3 销毁凭证

```shell
[asap@asap184 ~]# kdestroy
[asap@asap184 ~]# klist 
klist: No credentials cache found (ticket cache FILE:/tmp/krb5cc_0)
```

## 6. CDH 启用 Kerberos 安全认证

### 6.1 为CM 创建管理员主体/实体

```shell
[asap@asap185 ~]# sudo kadmin.local -q "addprinc cloudera-scm/admin"
Authenticating as principal root/admin@HADOOP.COM with password.
WARNING: no policy specified for cloudera-scm/admin @HADOOP.COM; defaulting to no 
policy
Enter password for principal " cloudera-scm/admin @HADOOP.COM": （输入密码）
Re-enter password for principal " cloudera-scm/admin @HADOOP.COM": （确认密码）
Principal " cloudera-scm/admin @HADOOP.COM" created.
```

### 6.2 启用Kerberos

![启用Kerberos](https://gitee.com/zhdoop/blogImg/raw/master/img/启动kerberos.png)

### 6.3 环境确认（勾选全部）

![环境确认](https://gitee.com/zhdoop/blogImg/raw/master/img/kerberos打钩.png)

### 6.4 填写配置

Kerberos 加密类型:aes128-cts、des3-hmac-sha1、arcfour-hmac

![填写配置](https://gitee.com/zhdoop/blogImg/raw/master/img/kerberos环境配置.png)

### 6.5 KRB5配置

![KRB5配置](https://gitee.com/zhdoop/blogImg/raw/master/img/krb5配置.png)

### 6.6 填写主体和密码

![填写主体和密码](https://gitee.com/zhdoop/blogImg/raw/master/img/输入用户凭证.png)

### 6.7 继续

后边都点继续，完成5，6，7，8，9步，安装就完成了。

### 6.8 查看主体

```shell
[asap@asap185 ~]$ sudo  kadmin.local -q "list_principals"
Authenticating as principal root/admin@HADOOP.COM with password.
HTTP/asap184@HADOOP.COM
HTTP/asap185@HADOOP.COM
K/M@HADOOP.COM
asap/asap@HADOOP.COM
cloudera-scm/admin@HADOOP.COM
hdfs/asap184@HADOOP.COM
hdfs/asap185@HADOOP.COM
hdfs/hdfs@HADOOP.COM
hive/asap184@HADOOP.COM
hue/asap184@HADOOP.COM
kadmin/admin@HADOOP.COM
kadmin/asap185@HADOOP.COM
kadmin/changepw@HADOOP.COM
kafka/asap184@HADOOP.COM
kafka/asap185@HADOOP.COM
kiprop/asap185@HADOOP.COM
krbtgt/HADOOP.COM@HADOOP.COM
mapred/asap184@HADOOP.COM
spark/asap185@HADOOP.COM
yarn/asap184@HADOOP.COM
yarn/asap185@HADOOP.COM
zookeeper/asap184@HADOOP.COM
```

## 7 Kerberos 安全环境实操

在启用 Kerberos 之后，系统与系统（flume-kafka）之间的通讯，以及用户与系统（user-hdfs）之间的通讯都需要先进行安全认证，认证通过之后方可进行通讯。

故在启用 Kerberos 后，数仓中使用的脚本等，均需要加入一步安全认证的操作，才能正常工作。

### 7.1 用户访问服务认证

开启 Kerberos 安全认证之后，日常的访问服务（例如访问 HDFS，消费 Kafka topic 等）都需要先进行安全认证

**1）在Kerberos 数据库中创建用户主体/实例**

```shell
[asap@asap185 ~]# sudo kadmin.local -q "addprinc asap/asap@HADOOP.COM"
```

**2）进行用户认证**

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
