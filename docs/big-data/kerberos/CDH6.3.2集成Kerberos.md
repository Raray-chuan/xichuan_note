#  CDH6.3.2集成Kerberos

## 一.参考doc

CDH enable kerberos: [Kerberos Security Artifacts Overview | 6.3.x | Cloudera Documentation](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cm_sg_principal_keytab.html)

CDH disable kerberos:https://www.sameerahmad.net/blog/disable-kerberos-on-CDH; https://community.cloudera.com/t5/Support-Questions/Disabling-Kerberos/td-p/19654



## 二.CDH集成Kerberos

进入Cloudera Manager的**“管理”->“安全”**界面
**1）选择“启用Kerberos”，进入如下界面**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211129108.png)

**2）环境确认（勾选全部）**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211131706.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211132781.png)

**3）填写KDC配置**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211139431.png)

要注意的是：这里的 Kerberos Encryption Types 必须跟KDC实际支持的加密类型匹配（即kdc.conf中的值）

**4）KRB5 信息**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211141730.png)

**5）填写主体名和密码**
注：此处填写的主题名与密码是我们在[Kerberos基础](https://raray-chuan.github.io/xichuan_note/#/docs/big-data/kerberos/Kerberos基础) 中创建的`cloudera-scm/admin@XICHUAN.COM`用户

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211142952.png)

**6）等待导入KDC凭据完成**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211143438.png)

**7）继续**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211143692.png)

**8）重启集群**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211145091.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211146396.png)

**9）完成**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211146415.png)

之后 Cloudera Manager 会自动重启集群服务，启动之后会提示 Kerberos 已启用。在 Cloudera Manager 上启用 Kerberos 的过程中，会自动做以下的事情：

集群中有多少个节点，每个账户就会生成对应个数的 principal ;
为每个对应的 principal 创建 keytab；
部署 keytab 文件到指定的节点中；
在每个服务的配置文件中加入有关 Kerberos 的配置；
启用之后访问集群的所有资源都需要使用相应的账号来访问，否则会无法通过 Kerberos 的 authenticatin。







## 三.Kerberos使用

### 3.1 访问hdfs

1.认证前（访问异常）：

```
[root@node01 ~]# hdfs dfs -ls /
22/10/26 15:55:50 WARN ipc.Client: Exception encountered while connecting to the server : org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]
ls: Failed on local exception: java.io.IOException: org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]; Host Details : local host is: "node01/192.168.1.230"; destination host is: "node01":8020; 
[root@node01 ~]# 
```

2.认证中（此处使用密码认证）：

```
[root@node01 ~]# kinit xichuan/admin
Password for xichuan/admin@XICHUAN.COM: 
[root@node01 ~]#
```

3.认证后（正常访问）：

```
[root@node01 ~]# hdfs dfs -ls /
Found 2 items
drwxrwxrwt   - hdfs supergroup          0 2022-10-25 16:08 /tmp
drwxr-xr-x   - hdfs supergroup          0 2022-10-25 16:15 /user
[root@node01 ~]# 
```



### 3.2 访问hive

1.认证前（访问异常）：

```
[root@node01 ~]# hive
WARNING: Use "yarn jar" to launch YARN applications.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/log4j-slf4j-impl-2.8.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]

Logging initialized using configuration in jar:file:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/hive-common-2.1.1-cdh6.3.2.jar!/hive-log4j2.properties Async: false
Exception in thread "main" java.lang.RuntimeException: java.io.IOException: Failed on local exception: java.io.IOException: org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]; Host Details : local host is: "node01/192.168.1.230"; destination host is: "node01":8020; 
	at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:604)
	at org.apache.hadoop.hive.ql.session.SessionState.beginStart(SessionState.java:545)
	at org.apache.hadoop.hive.cli.CliDriver.run(CliDriver.java:763)
	at org.apache.hadoop.hive.cli.CliDriver.main(CliDriver.java:699)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.util.RunJar.run(RunJar.java:313)
	at org.apache.hadoop.util.RunJar.main(RunJar.java:227)
Caused by: java.io.IOException: Failed on local exception: java.io.IOException: org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]; Host Details : local host is: "node01/192.168.1.230"; destination host is: "node01":8020; 
	at org.apache.hadoop.net.NetUtils.wrapException(NetUtils.java:808)
	at org.apache.hadoop.ipc.Client.getRpcResponse(Client.java:1503)
	at org.apache.hadoop.ipc.Client.call(Client.java:1445)
	at org.apache.hadoop.ipc.Client.call(Client.java:1355)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:228)
	at org.apache.hadoop.ipc.ProtobufRpcEngine$Invoker.invoke(ProtobufRpcEngine.java:116)
	at com.sun.proxy.$Proxy28.getFileInfo(Unknown Source)
	at org.apache.hadoop.hdfs.protocolPB.ClientNamenodeProtocolTranslatorPB.getFileInfo(ClientNamenodeProtocolTranslatorPB.java:875)
	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
	at java.lang.reflect.Method.invoke(Method.java:498)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invokeMethod(RetryInvocationHandler.java:422)
	at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeMethod(RetryInvocationHandler.java:165)
	at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invoke(RetryInvocationHandler.java:157)
	at org.apache.hadoop.io.retry.RetryInvocationHandler$Call.invokeOnce(RetryInvocationHandler.java:95)
	at org.apache.hadoop.io.retry.RetryInvocationHandler.invoke(RetryInvocationHandler.java:359)
	at com.sun.proxy.$Proxy29.getFileInfo(Unknown Source)
	at org.apache.hadoop.hdfs.DFSClient.getFileInfo(DFSClient.java:1630)
	at org.apache.hadoop.hdfs.DistributedFileSystem$29.doCall(DistributedFileSystem.java:1496)
	at org.apache.hadoop.hdfs.DistributedFileSystem$29.doCall(DistributedFileSystem.java:1493)
	at org.apache.hadoop.fs.FileSystemLinkResolver.resolve(FileSystemLinkResolver.java:81)
	at org.apache.hadoop.hdfs.DistributedFileSystem.getFileStatus(DistributedFileSystem.java:1508)
	at org.apache.hadoop.fs.FileSystem.exists(FileSystem.java:1617)
	at org.apache.hadoop.hive.ql.session.SessionState.createRootHDFSDir(SessionState.java:712)
	at org.apache.hadoop.hive.ql.session.SessionState.createSessionDirs(SessionState.java:650)
	at org.apache.hadoop.hive.ql.session.SessionState.start(SessionState.java:580)
	... 9 more
Caused by: java.io.IOException: org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]
	at org.apache.hadoop.ipc.Client$Connection$1.run(Client.java:756)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1875)
	at org.apache.hadoop.ipc.Client$Connection.handleSaslConnectionFailure(Client.java:719)
	at org.apache.hadoop.ipc.Client$Connection.setupIOstreams(Client.java:812)
	at org.apache.hadoop.ipc.Client$Connection.access$3600(Client.java:410)
	at org.apache.hadoop.ipc.Client.getConnection(Client.java:1560)
	at org.apache.hadoop.ipc.Client.call(Client.java:1391)
	... 33 more
Caused by: org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]
	at org.apache.hadoop.security.SaslRpcClient.selectSaslClient(SaslRpcClient.java:173)
	at org.apache.hadoop.security.SaslRpcClient.saslConnect(SaslRpcClient.java:390)
	at org.apache.hadoop.ipc.Client$Connection.setupSaslConnection(Client.java:614)
	at org.apache.hadoop.ipc.Client$Connection.access$2300(Client.java:410)
	at org.apache.hadoop.ipc.Client$Connection$2.run(Client.java:799)
	at org.apache.hadoop.ipc.Client$Connection$2.run(Client.java:795)
	at java.security.AccessController.doPrivileged(Native Method)
	at javax.security.auth.Subject.doAs(Subject.java:422)
	at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1875)
	at org.apache.hadoop.ipc.Client$Connection.setupIOstreams(Client.java:795)
	... 36 more

```

2.认证中（此处使用密码认证）：

```
[root@node01 ~]# kinit xichuan/admin
Password for xichuan/admin@XICHUAN.COM: 
[root@node01 ~]#
```

3.认证后（正常访问）：

```
[root@node01 ~]# hive
WARNING: Use "yarn jar" to launch YARN applications.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/log4j-slf4j-impl-2.8.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/slf4j-log4j12-1.7.25.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]

Logging initialized using configuration in jar:file:/opt/cloudera/parcels/CDH-6.3.2-1.cdh6.3.2.p0.1605554/jars/hive-common-2.1.1-cdh6.3.2.jar!/hive-log4j2.properties Async: false

WARNING: Hive CLI is deprecated and migration to Beeline is recommended.
hive> show databases;
OK
default
test_dws
Time taken: 2.369 seconds, Fetched: 4 row(s)
hive> select * from test_dws.test_table limit 1;
OK
NULL	1919	3	1	1	P	2630	480										HUAWEI_JSCC	ZTT	    JSCC		SA8200RFIV110		35042026		NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULLNULL	NULL	NULL	NULL		NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL	NULL		NULL		NULL	NULL	NULLNULL	NULL	0	1641440662000	FT_ECID	1	NULL	FTTB3Q1G40000	LX-2021122
Time taken: 21.826 seconds, Fetched: 1 row(s)
hive> 

```



### 3.3 访问impala-shell

1.认证前（无法访问）：

```
[root@node01 ~]# impala-shell
Starting Impala Shell without Kerberos authentication
Opened TCP connection to node01:21000
Error connecting: TTransportException, TSocket read 0 bytes
***********************************************************************************
Welcome to the Impala shell.
(Impala Shell v3.2.0-cdh6.3.2 (1bb9836) built on Fri Nov  8 07:22:06 PST 2019)

To see a summary of a query's progress that updates in real-time, run 'set
LIVE_PROGRESS=1;'.
***********************************************************************************
[Not connected] >
```

2.认证中（此处使用密码认证）：

```
[root@node01 ~]# kinit xichuan/admin
Password for xichuan/admin@XICHUAN.COM: 
[root@node01 ~]#
```

3.认证后（正常访问）：

```
root@node01 ~]# impala-shell
Starting Impala Shell without Kerberos authentication
Opened TCP connection to node01:21000
Error connecting: TTransportException, TSocket read 0 bytes
Kerberos ticket found in the credentials cache, retrying the connection with a secure transport.
Opened TCP connection to node01:21000
Connected to node01:21000
Server version: impalad version 3.2.0-cdh6.3.2 RELEASE (build 1bb9836227301b839a32c6bc230e35439d5984ac)
***********************************************************************************
Welcome to the Impala shell.
(Impala Shell v3.2.0-cdh6.3.2 (1bb9836) built on Fri Nov  8 07:22:06 PST 2019)

To see live updates on a query's progress, run 'set LIVE_SUMMARY=1;'.
***********************************************************************************
[node01:21000] default> 
[node01:21000] default> select * from  kudu_test_dws.test_table2;
Query: select * from  kudu_test_dws.test_table2
Query submitted at: 2022-10-26 16:29:26 (Coordinator: http://node01:25000)
Query progress can be monitored at: http://node01:25000/query_plan?query_id=af489e22bc4c3f46:c0d08c6100000000
+----------------------------------+-------------+-------------+------------+----------------+------------+---------------+----------------+------------------++---------------+---------------+----------------+---------------+-----------------+-------------------+----------------+------+---------+---------+-----------+----------+------------+
| id                               | range_month | total_count | good_count | total_td_count | gross_time | net_test_time | good_test_time | good_td_sequence || start_time    | end_time      | lot_start_time | lot_end_time  | handler_id      | loadboard         | loadboard_type | year | month   | qualter | work_week | week_day | run_date   |
+----------------------------------+-------------+-------------+------------+----------------+------------+---------------+----------------+------------------++---------------+---------------+----------------+---------------+-----------------+-------------------+----------------+------+---------+---------+-----------+----------+------------+
| 55e9b624ab7ecc10b4595c791f49917a | 8           | 226         | 57         | 205            | 1838000    | 1191436       | 591308         | 52               || 1628291104000 | 1628292942000 | 1628221346000  | 1628292942000 | PTSK-O-011-UF30 | V93-G12BX2-PH04/2 |                | 2021 | 2021-08 | 2021-03 | 2021-31   | 6        | 2021-08-07 |
| d802906d9817897f54383a492e7464b6 | 8           | 250         | 76         | 216            | 1786000    | 1304322       | 706299         | 62               || 1628750593000 | 1628752379000 | 1628750593000  | 1628752379000 | PTSK-O-011-UF30 | V93-G12BX2-PH04/2 |                | 2021 | 2021-08 | 2021-03 | 2021-32   | 4        | 2021-08-12 |
| d9f613e8c6924c77493304d265ca9fa6 | 8           | 1344        | 1137       | 690            | 8890000    | 7728323       | 5845658        | 502              || 1628230300000 | 1628239190000 | 1628230300000  | 1628294556000 | PTSK-O-011-UF30 | V93-G12BX2-PH04/2 |                | 2021 | 2021-08 | 2021-03 | 2021-31   | 5        | 2021-08-06 |
+----------------------------------+-------------+-------------+------------+----------------+------------+---------------+----------------+------------------++---------------+---------------+----------------+---------------+-----------------+-------------------+----------------+------+---------+---------+-----------+----------+------------+
Fetched 3 row(s) in 0.39s
[node01:21000] default> 


```

### 3.4 错误记录

#### 3.4.1 hdfs相关指令报错

错误信息 : `AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]`

```
[root@node01 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: root/admin@XICHUAN.COM

Valid starting       Expires              Service principal
10/26/2022 15:19:15  10/27/2022 15:19:15  krbtgt/XICHUAN.COM@XICHUAN.COM
	renew until 11/02/2022 15:19:15
[root@node01 ~]#
[root@node01 ~]# hadoop fs -ls /
2021-03-28 20:23:27,667 WARN ipc.Client: Exception encountered while connecting to the server : org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]
ls: DestHost:destPort master01:8020 , LocalHost:localPort master01/192.xx.xx:0. Failed on local exception: java.io.IOException: org.apache.hadoop.security.AccessControlException: Client cannot authenticate via:[TOKEN, KERBEROS]
```

**解决办法:**
3. 修改 Kerboeros配置文件 /etc/krb5.conf , 注释掉 : `default_ccache_name`属性 .然后执行kdestroy,重新kinit .
4. ![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041655833.png)





### 3.5 Dbever访问Kerberos环境下的Impala

#### 3.5.1 windows下安装kfw客户端

下载地址：[https://web.mit.edu/kerberos/dist/index.html](https://links.jianshu.com/go?to=https%3A%2F%2Fweb.mit.edu%2Fkerberos%2Fdist%2Findex.html)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271555796.png)

安装过程没什么好说的，傻瓜式安装，唯一需要注意的是：

安装之后不要点击重启(其实也可以，但是没必要)！不要打开软件!

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271558464.png)



####  3.5.2 修改C:\ProgramData\MIT\Kerberos5\krb5.ini文件

kfw启动时会读取`C:\ProgramData\MIT\Kerberos5\krb5.ini`的配置文件，我们需要把它配置成和集群中一样的配置

1.连接你的集群krb5kdc和kadmin服务所在的机器，复制/etc/krb5.conf中的配置

`vim /etc/krb5.conf`

```shell
[libdefaults]
 dns_lookup_realm = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 pkinit_anchors = FILE:/etc/pki/tls/certs/ca-bundle.crt
 default_realm = XICHUAN.COM
 # default_ccache_name = KEYRING:persistent:%{uid}

[realms]
 XICHUAN.COM = {
  kdc = node02
  admin_server = node02
 }

[domain_realm]
 .example.com = XICHUAN.COM
 example.com = XICHUAN.COM

```



2.修改`C:\ProgramData\MIT\Kerberos5\krb5.ini`的配置文件

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271604092.png)



####  3.5.3 修改环境变量

dbeaver会读取我们的环境变量 $KRB5CCNAME 来获取kfw的缓存

**1.在C盘创建temp文件夹**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271605779.png)

**2.增加环境变量**

`KRB5CCNAME=C:\temp\krb5cache`

`KRB5_CONFIG=C:\ProgramData\MIT\Kerberos5\krb5.ini`

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271607427.png)



**3.打开kfw软件登陆**

确认可以登陆后重启windows

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041657666.png)

重启后我们会发现这里多了个文件：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271610914.png)



**4.查看登录状态**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041658559.png)



####  3.5.4 修改dbeaver配置文件和连接配置

**1.在DBeaver的安装目录下找到dbeaver.ini文件**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271615886.png)

**2.在dbeaver.ini中后面添加**

```
-Djavax.security.auth.useSubjectCredsOnly=false

-Djava.security.krb5.conf=C:\ProgramData\MIT\Kerberos5\krb5.ini

-Dsun.security.krb5.debug=true
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271635532.png)

切记

`-Djava.security.krb5.conf=C:\ProgramData\MIT\Kerberos5\krb5.ini`

不要加引号「“”」！！！



**3.Dbever登录impala**

点击编辑驱动设置

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041700223.png)



修改URL模板为：

```shell
jdbc:impala://{host}:{port}/{database};AuthMech=1;KrbRealm=XICHUAN.COM;KrbHostFQDN={host};KrbServiceName=impala;KrbAuthType=2
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271640403.png)



添加impalaJDBC文件

impalaJDBC下载：[https://mvnrepository.com/artifact/com.cloudera/ImpalaJDBC41/2.6.3](https://links.jianshu.com/go?to=https%3A%2F%2Fmvnrepository.com%2Fartifact%2Fcom.cloudera%2FImpalaJDBC41%2F2.6.3)





测试连接：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041700168.png)

OK！可以愉快的用DBeaver写sql了！



### 3.6 Windows访问HDFS/YARN/HIVESERVER2 等服务的webui

> 参考：https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_sg_browser_access_kerberos_protected_url.html#topic_6_2__section_vj5_gwv_ls

#### 3.6.1 HDFS/YARN/HIVESERVER2开启webui验证

**1.修改CDH中的配置：**

hdfs 开启 `Enable Kerberos Authentication for HTTP Web-Consoles`

下图中的配置选项选中，使其生效

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281052048.png)

yarn开启 `Enable Kerberos Authentication for HTTP Web-Consoles`

下图中的配置选项选中，使其生效

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281053585.png)

hive设置 `hive.server2.webui.use.spnego=true`

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281055596.png)

**2.重启CDH**



**3.访问相应的页面**

访问hdfs webui

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281057131.png)

访问yarn webui

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281057407.png)

访问hive webui

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281058529.png)



#### 3.6.2 修改配置并进行验证后访问

> 注：此处只设置了Firefox，其他浏览器设置，参考本章节参考链接自行设置

**1.修改Firefox配置**

由于chrome浏览器中 kerberos 相关配置比较复杂，建议配置使用firefox浏览器。打开firefox浏览器，在地址栏输入`about:config`，然后搜索并配置如下两个参数：
`network.auth.use-sspi`：将值改为`false`；
`network.negotiate-auth.trusted-uris`：将值为集群节点ip或主机名；

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281103844.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281104903.png)

**2.打开kfw软件登陆**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041657666.png)



**3.再次访问页面**

访问hdfs webui

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281106083.png)

访问yarn webui

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281107487.png)

访问hive webui

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210281107994.png)





### 3.7代码中的认证

#### 3.7.1 spark程序的认证
**windows本地运行:**
```scala
  val krb5ConfPath = "D:\\development\\license_dll\\krb5.conf"
  val keyTabPath = "D:\\development\\license_dll\\xichuan.keytab"
  val principle = "xichuan/admin@XICHUAN.COM"

  def kerberosAuth(krb5ConfPath:String,keytabPath:String,principle:String): Unit ={
    val conf = new Configuration
    System.setProperty("java.security.krb5.conf", krb5ConfPath)

    //conf.addResource(new Path("C:\\Users\\xichuan\\Desktop\\hive-conf\\hive-site.xml"))
    conf.set("hadoop.security.authentication", "Kerberos")
    conf.set("fs.hdfs.impl", "org.apache.hadoop.hdfs.DistributedFileSystem")
    UserGroupInformation.setConfiguration(conf)
    val loginInfo = UserGroupInformation.loginUserFromKeytabAndReturnUGI(principle, keytabPath)

    if (loginInfo.hasKerberosCredentials) {
      println("kerberos authentication success!")
      println("login user: "+loginInfo.getUserName())
    } else {
      println("kerberos authentication fail!")
    }
  }

```

**spark-submit集群运行:**

```shell
##1.可以本地kinit 认证一个用户（会过期）

##2.spark-submit添加
--principal xichuan/admin@XICHUAN.COM \
--keytab /opt/xichuan.keytab \
```
提交命令:
```shell
spark-submit \
--master yarn \
--deploy-mode cluster \
--conf spark.sql.shuffle.partitions=200 \
--principal xichuan/admin@XICHUAN.COM \
--keytab /opt/xichuan.keytab \
--num-executors 1 \
--executor-memory 2G  \
--executor-cores 1 \
--queue root.default \
--class com.xichuan.dev.TestSparkHive.scala /opt/xichuan/spark-test.jar 
```

#### 3.7.2 flink程序的认证
**flink-session:**
```shell
$ vim flink-conf.yaml
security.kerberos.login.use-ticket-cache: true
security.kerberos.login.keytab: /opt/xichuan.keytab
security.kerberos.login.principal: xichuan/admin@XICHUAN.COM
security.kerberos.login.contexts: Client
```

```scala
#从kafka接收数据写入到hdfs中，同时受kerberos+ranger权限管控
val sink: StreamingFileSink[String] = StreamingFileSink
      .forRowFormat(new Path("hdfs://temp/xichuan/kerberos.test"), new SimpleStringEncoder[String]("UTF-8"))
      .withRollingPolicy(
        DefaultRollingPolicy.builder()
          .withRolloverInterval(TimeUnit.SECONDS.toMillis(15))
          .withInactivityInterval(TimeUnit.SECONDS.toMillis(5))
          .withMaxPartSize(1024 * 1024 * 1024)
          .build())
      .build()
    sinkstream.addSink(sink)
```


#### 3.7.3 Java连接impala的认证
```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.security.UserGroupInformation;

import java.io.IOException;
import java.security.PrivilegedAction;
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.ResultSet;
import java.sql.Statement;
/**
 * @Author Xichuan
 * @Date 2022/10/28 17:53
 * @Description
 */
public class TestKerberosImpala {
        public static final String KRB5_CONF = "D:\\development\\license_dll\\krb5.conf";
        public static final String PRINCIPAL = "xichuan/admin@XICHUAN.COM";
        public static final String KEYTAB = "D:\\development\\license_dll\\xichuan.keytab";
        public static String connectionUrl = "jdbc:impala://node01:21050/;AuthMech=1;KrbRealm=XICHUAN.COM;KrbHostFQDN=node01;KrbServiceName=impala";
        public static String jdbcDriverName = "com.cloudera.impala.jdbc41.Driver";

        public static void main(String[] args) throws Exception {
            UserGroupInformation loginUser = kerberosAuth(KRB5_CONF,KEYTAB,PRINCIPAL);

            int result = loginUser.doAs((PrivilegedAction<Integer>) () -> {
                int result1 = 0;
                try {
                    Class.forName(jdbcDriverName);
                } catch (ClassNotFoundException e) {
                    e.printStackTrace();
                }
                try (Connection con = DriverManager.getConnection(connectionUrl)) {
                    Statement stmt = con.createStatement();
                    ResultSet rs = stmt.executeQuery("SELECT count(1) FROM test_dws.dws_test_id");
                    while (rs.next()) {
                        result1 = rs.getInt(1);
                    }
                    stmt.close();
                    con.close();
                } catch (Exception e) {
                    e.printStackTrace();
                }
                return result1;
            });
            System.out.println("count: "+ result);
        }

    /**
     * kerberos authentication
     * @param krb5ConfPath
     * @param keyTabPath
     * @param principle
     * @return
     * @throws IOException
     */
        public static UserGroupInformation kerberosAuth(String krb5ConfPath, String keyTabPath, String principle) throws IOException {
            System.setProperty("java.security.krb5.conf", krb5ConfPath);
            Configuration conf = new Configuration();
            conf.set("hadoop.security.authentication", "Kerberos");
            UserGroupInformation.setConfiguration(conf);
            UserGroupInformation loginInfo = UserGroupInformation.loginUserFromKeytabAndReturnUGI(principle, keyTabPath);


            if (loginInfo.hasKerberosCredentials()) {
                System.out.println("kerberos authentication success!");
                System.out.println("login user: "+loginInfo.getUserName());
            } else {
                System.out.println("kerberos authentication fail!");
            }

            return loginInfo;
        }
}
```



#### 3.7.4 Springboot使用hikari连接池并进行Kerberos认证访问Impala

项目与文档地址：https://github.com/Raray-chuan/springboot-kerberos-hikari-impala



## 四.CDH disable Kerberos

禁用Kerberos，由于没有按钮可以直接操作，需要我们手动一个个修改开启了Kerberos的组件的配置。修改步骤按以下来：

**1.先关闭集群(如果yarn上有任务则等待停止，或手动停止)**



**2.修改zookeeper的配置**
enableSecurity取消勾选，`Enable Kerberos Authentication`取消勾选，在zoo.cfg 的`Server 高级配置代码段（安全阀`）写入`skipACL: yes`
如下图：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211413315.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211415868.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211415452.png)

**3.修改HDFS配置**
修改`hadoop.security.authentication`为`simple`，`hadoop.security.authorization`取消勾选，`dfs.datanode.address`从`1004`修改为`50010`，`dfs.datanode.http.address`从`1006`修改为`50075`，`dfs.datanode.data.dir.perm`从`700`修改为`755`。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211417117.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211418318.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211418032.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211418418.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211418829.png)

**4.修改Kudu配置**
`enable_security`取消勾选

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211419448.png)

**5.修改Kafka配置**
`kerberos.auth.enable`取消勾选

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211420381.png)

**6.修改HBase配置(如有此组件，进行修改)**
`hbase.security.authentication`选择`simple`，`hbase.security.authorization`取消勾选，`hbase.thrift.security.qop`选择`none`。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211421799.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211421178.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211421371.png)

**7.修改Solr配置(如有此组件，进行修改)**
`solr Secure Authentication`选择`simple`



**8.修改Hue配置(如有此组件，进行修改)**
进入HUE的实例页面，停止`“Kerberos Ticket Renewer”`后删除Hue实例中的`“Kerberos Ticket Renewer”`服务。



**9.以上的组件有的就做相关的操作，没有的组件略过，要注意去除所有的Kerberos相关的配置，不然CM的管理-安全页面还是会显示已启用Kerberos**



**10.修改完相关的Kerberos配置后，Cloudera Management Service会显示配置已更改，需要重启，重启Cloudera Management Service，然后管理-安全页面就会显示禁用Kerberos**



**11.启动zookeeper，然后连接客户端，删除几个路径：**

```bash
rmr /rmstore/ZKRMStateRoot
rmr /hbase
```

然后删除zookeeper里在zoo.cfg 的Server 高级配置代码段（安全阀）中的skipACL: yes内容，重启zookeeper。



**12.启动所有服务，完成禁用Kerberos。**





