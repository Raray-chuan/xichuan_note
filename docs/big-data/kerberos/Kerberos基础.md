## 一. Kerberos概述

Kerberos是一种计算机网络授权协议，用来在非安全网络中，对个人通信以安全的手段进行身份认证。这个词又指麻省理工学院为这个协议开发的一套计算机软件。软件设计上采用客户端/服务器结构，并且能够进行相互认证，即客户端和服务器端均可对对方进行身份认证。可以用于防止窃听、防止重放攻击、保护数据完整性等场合，是一种应用对称密钥体制进行密钥管理的系统。

Hadoop使用Kerberos作为用户和服务的强身份验证和身份传播的基础。Kerberos是一种计算机网络认证协议，它允许某实体在非安全网络环境下通信，向另一个实体以一种安全的方式证明自己的身份。 Kerberos是第三方认证机制，其中用户和服务依赖于第三方（Kerberos服务器）来对彼此进行身份验证。 Kerberos服务器本身称为密钥分发中心或KDC。 在较高的层面上，它有三个部分：

一个认证服务器（AS）执行初始认证并颁发票证授予票证（TGT）
一个票据授权服务器（TGS）发出基于初始后续服务票证TGT
一个用户主要来自AS请求认证。AS返回使用用户主体的Kerberos密码加密的TGT，该密码仅为用户主体和AS所知。用户主体使用其Kerberos密码在本地解密TGT，从那时起，直到ticket到期，用户主体可以使用TGT从TGS获取服务票据。服务票证允许委托人访问各种服务。

Kerberos简单来说就是一个用于安全认证第三方协议，它采用了传统的共享密钥的方式，实现了在网络环境不一定保证安全的环境下，client和server之间的通信，适用于client/server模型，由MIT开发和实现。

Kerberos服务是单点登录系统，这意味着您对于每个会话只需向服务进行一次自我验证，即可自动保护该会话过程中所有后续事务的安全。

由于每次解密TGT时群集资源（主机或服务）都无法提供密码，因此它们使用称为keytab的特殊文件，该文件包含资源主体的身份验证凭据。 Kerberos服务器控制的主机，用户和服务集称为领域。



## 二. Kerberos术语

**Princal(安全个体)**:Kerberos所管理的一个用户或者一个服务，可以理解为Kerberos中保存的一个账号，其格式通常如下：primary**/**instance@realm

**Realm**：Kerberos所管理的一个领域或范围，称之为一个Realm。

**keytab**：Kerberos中的用户认证，可通过密码或者密钥文件证明身份，keytab指密钥文件。

**KDC（key distribution center）**: 认证过程的票据生成管理服务，其中包含两个服务，AS（Authentication Service）和TGS（Ticket Granting Service）。
**Ticket**：一个记录,客户用它来向服务器证明自己的身份,包括客户标识,会话密钥,时间戳.
**AS(Authentication Server)**：认证服务器, 为client生成TGT的服务。
**TGS(Ticket Grantion Server)**: 许可证服务器, 为client生成某个服务的ticket
**TGT(Ticket-grantion Ticket)** : 用于获取ticket的票据
**AD（Account Database）**: 存储所有client的白名单，只有存在于白名单的client才能顺利申请到ticket



## 三. Kerberos认证流程

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202210211128163.png)

（１）客户端执行kinit命令，输入Principal及Password，向AS证明身份，并请求获取TGT。
（２）AS检查Database中是否存有客户端输入的Principal，如有则向客户端返回TGT。
（３）客户端获取TGT后，向TGS请求ServerTicket。
（４）TGS收到请求，检查Database中是否存有客户端所请求服务的Principal，如有则向客户端返回ServerTicket。
（５）客户端收到ServerTicket，则向目标服务发起请求。
（６）目标服务收到请求，响应客户端。


## 四.Kerberos部署

### 4.1 概述

Cloudera提供了非常简便的Kerberos集成方式，基本做到了自动化部署。

系统：CentOS 7.3

操作用户：admin

角色分布如下：

| 角色                          | 部署节点                |
| --------------------------- | ------------------- |
| KDC, AS, TGS,Kerberos Agent | 192.168.0.200       |
| Kerberos Agent              | 192.168.0.[201-202] |

假设slaves文件如下：

```bash
192.168.0.201
192.168.0.202
```

### 4.3 Kerberos安装

在192.168.0.200上安装服务端：

```bash
yum -y install krb5-server openldap-clients
```

在192.168.0.[201-202]上安装客户端：

```bash
pssh -h slaves -P -l root -A "yum -y install krb5-devel krb5-workstation"
```

在安装完上述的软件之后，会在KDC主机上生成配置文件/etc/krb5.conf和/var/kerberos/krb5kdc/kdc.conf，它们分别反映了realm name 以及 domain-to-realm mappings。



### 4.4 修改配置

#### 4.4.1 配置krb5.conf

**`/etc/krb5.conf`**: 包含Kerberos的配置信息。例如，KDC的位置，Kerberos的admin的realms 等。需要所有使用的Kerberos的机器上的配置文件都同步。这里仅列举需要的基本配置。

```bash
[logging]
 default = FILE:/var/log/krb5libs.log
 kdc = FILE:/var/log/krb5kdc.log
 admin_server = FILE:/var/log/kadmind.log

[libdefaults]
 default_realm = XICHUAN.COM
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 renew_lifetime = 7d
 forwardable = true
 rdns = false
 default_ccache_name = KEYRING:persistent:%{uid}

[realms]
  EXAMPLE.COM = {
   kdc = master01-dev
   admin_server = master01-dev
  }

[domain_realm]
  .example.com = XICHUAN.COM
  example.com = XICHUAN.COM
```

**说明**：

**[logging]**：表示server端的日志的打印位置

**[libdefaults]**：每种连接的默认配置，需要注意以下几个关键的小配置

1. **default_realm** = EXAMPLE.COM 默认的realm，必须跟要配置的realm的名称一致。

2. **udp_preference_limit** = 1 禁止使用udp可以防止一个Hadoop中的错误

3. **ticket_lifetime** 表明凭证生效的时限，一般为24小时。

4. **renew_lifetime** 表明凭证最长可以被延期的时限，一般为一个礼拜。当凭证过期之后，对安全认证的服务的后续访问则会失败

**[realms]**:列举使用的realm。

1.   **kdc**：代表要kdc的位置。格式是 机器:端口

2.   **admin_server**: 代表admin的位置。格式是机器:端口
3.   **default_domain**：代表默认的域名

**[appdefaults]**:可以设定一些针对特定应用的配置，覆盖默认配置。


分发krb5.conf至所有client：

//这一步可以不需要，因为在Cloudera Manager中启用配置krb5.conf后，它会自动帮你部署到客户端。

```bash
pscp -h slaves krb5.conf /tmp
pssh -h slaves "cp /tmp/krb5.conf /etc"
```



#### 4.4.2 配置kdc.conf

默认放在 **`/var/kerberos/krb5kdc/kdc.conf`**

```
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 XICHUAN.COM = {
  #master_key_type = aes256-cts
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes128-cts:normal aes256-cts des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
  max_life = 24h
  max_renewable_life = 7d
 }
```

**说明：**

**XICHUAN.COM**:是设定的realms。名字随意。Kerberos可以支持多个realms，会增加复杂度。本文不探讨。大小写敏感，一般为了识别使用全部大写。这个realms跟机器的host没有大关系。
**max_renewable_life = 8d** 涉及到是否能进行ticket的renwe必须配置。
**master_key_type:和supported_enctypes**默认使用aes256-cts。由于，JAVA使用aes256-cts验证方式需要安装额外的jar包，推荐不使用。
**acl_file**:标注了admin的用户权限。文件格式是:
​	`Kerberos_principal permissions [target_principal][restrictions]`支持通配符等。
**admin_keytab:KDC**进行校验的keytab。后文会提及如何创建。
**supported_enctypes**:支持的校验方式。注意把aes256-cts去掉。



#### 4.4.3 配置kadm5.acl

修改服务端192.168.0.200上的配置文件/var/kerberos/krb5kdc/kadm5.acl，以允许具备匹配条件的admin用户进行远程登录权限：

```
*/admin@XICHUAN.COM *
```

**说明：**

标注了admin的用户权限，需要用户自己创建。文件格式是:
​	`Kerberos_principal permissions [target_principal][restrictions]`
支持通配符等。最简单的写法是:
​	`*/admin@XICHUAN.COM *`
代表名称匹配*/admin@XICHUAN.COM 都认为是admin，权限是 * 代表全部权限。



### 4.5 创建Kerberos数据库

在192.168.0.200上对数据库进行初始化，默认的数据库路径为`/var/kerberos/krb5kdc`，如果需要重建数据库，将该目录下的principal相关的文件删除即可，请牢记数据库密码。

```bash
kdb5_util create -r XICHUAN.COM -s
```

**说明：**

**[-s]** 表示生成 stash file ，并在其中存储master server key（krb5kdc）
**[-r]** 来指定一个realm name ， 当krb5.conf中定义了多个realm时使用
当 Kerberos database创建好之后，在 /var/kerberos/ 中可以看到生成的 principal相关文件

### 4.6 启动Kerberos服务

在192.168.0.200上执行：

```bash
#启动服务命令 
systemctl start krb5kdc 
systemctl start kadmin 

#加入开机启动项 
systemctl enable krb5kdc 
systemctl enable kadmin
```

**查看进程：**

```bash
[root@master01-dev ~]# netstat -anpl|grep kadmin
tcp        0      0 0.0.0.0:749             0.0.0.0:*               LISTEN      67465/kadmind       
tcp        0      0 0.0.0.0:464             0.0.0.0:*               LISTEN      67465/kadmind       
tcp6       0      0 :::749                  :::*                    LISTEN      67465/kadmind       
tcp6       0      0 :::464                  :::*                    LISTEN      67465/kadmind       
udp        0      0 0.0.0.0:464             0.0.0.0:*                           67465/kadmind       
udp6       0      0 :::464                  :::*                                67465/kadmind       
[root@master01-dev ~]# netstat -anpl|grep kdc
tcp        0      0 0.0.0.0:88              0.0.0.0:*               LISTEN      67402/krb5kdc       
tcp6       0      0 :::88                   :::*                    LISTEN      67402/krb5kdc       
udp        0      0 0.0.0.0:88              0.0.0.0:*                           67402/krb5kdc       
udp6       0      0 :::88                   :::*                                67402/krb5kdc  
```



### 4.7 创建Kerberos管理员principal

```bash
 #需要设置两次密码 
kadmin.local -q "addprinc root/admin"
#使用cloudera-scm/admin作为CM创建其它principals的超级用户
kadmin.local -q "addprinc cloudera-scm/admin"
```

principal的名字的第二部分是admin,那么根据之前配置的acl文件，该principal就拥有administrative privileges，这个账号将会被CDH用来生成其他用户/服务的principal。

`注意需要先kinit保证已经有principal缓存。`
```bash
[root@master01-dev ~]# kinit cloudera-scm/admin
Password for cloudera-scm/admin@XICHUAN.COM: 
[root@master01-dev ~]# klist 
Ticket cache: KEYRING:persistent:0:0
Default principal: cloudera-scm/admin@XICHUAN.COM

Valid starting       Expires              Service principal
09/01/2022 11:51:17  09/02/2017 11:51:16  krbtgt/BOCLOUD.COM@XICHUAN.COM
	renew until 09/08/2022 11:51:16
```
**Kerberos客户端支持两种，一是使用 principal + Password，二是使用 principal + keytab**

**principal + Password适合用户进行交互式应用，例如hadoop fs -ls这种，后者适合服务，例如yarn的rm、nm等。**
**principal + keytab就类似于ssh免密码登录，登录时不需要密码了。**



## 五.Kerberos使用

### 5.1 Kerberos主体Principal介绍

在官网上有详细的介绍Principal的介绍，链接为：http://web.mit.edu/kerberos/krb5-1.5/krb5-1.5.4/doc/krb5-user/What-is-a-Kerberos-Principal_003f.html

主体是Kerberos可以为其分配票证的唯一标识。主体可以具有任意数量的组件。每个组件都由一个组件分隔符（通常为 /）分隔。最后一个组成部分是领域，由领域分隔符（通常为@）与主体的其余部分分隔。如果主体中没有领域组件，则将假定主体在使用它的上下文中位于默认领域中

Principal的格式一般为：primary/instance@REALM

primary是Principal的一部分，如果是用户，则为用户名，如果是主机则为主机名。

instance是一个可选字符串，用于限定主服务器。该实例与主实例之间用斜杠（/）隔开。对于用户而言，该实例通常为null，但用户可能还具有一个附加的主体，即一个名为admin的实例，用该实例来管理数据库。主体jennifer@XICHUAN.COM与主体root/admin@XICHUAN.COM完全分开 ，具有单独的密码和单独的权限。对于主机，实例是标准主机名，例如 xichuan.com。

REALM是您的Kerberos领域。在大多数情况下，您的Kerberos域是您的域名，以大写字母表示。例如，机器daffodil.example.com就在领域中EXAMPLE.COM



### 5.2 Kerberos 命令使用

```
kadmin.local 与 kadmin区别：
kadmin.local：需要在 KDC server 上面操作，无需密码即可管理资料库
kadmin：可以在任何一台 KDC 领域的系统上面操作，但是需要输入管理员密码
```



#### 5.2.1登陆kinit

**`kinit root/admin@XICHUAN.COM`**

```shell
[root@node01 ~]# kinit root/admin@XICHUAN.COM
Password for root/admin@XICHUAN.COM: root
[root@node01 ~]# 
```

或者使用keytap登陆

**`kinit -kt /opt/xichuan.keytab xichuan/admin`**

```shell
[root@node01 ~]# kinit -kt /opt/xichuan.keytab xichuan/admin
[root@node01 ~]# 
```

#### 5.2.2查询登陆状态klist

**`klist`**

```shell
[root@node01 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: root/admin@XICHUAN.COM

Valid starting       Expires              Service principal
10/26/2022 15:19:15  10/27/2022 15:19:15  krbtgt/XICHUAN.COM@XICHUAN.COM
	renew until 11/02/2022 15:19:15
```

#### 5.2.3退出登陆kdestroy

**`kdestroy`**

```shell
[root@node01 ~]# kdestroy
[root@node01 ~]# klist
klist: No credentials cache found (filename: /tmp/krb5cc_0)
```

#### 5.2.4登录KDC后台 kadmin.local

**`kadmin.local`**

```shell
[root@node02 etc]# kadmin.local
Authenticating as principal root/admin@XICHUAN.COM with password.
kadmin.local:  

```

如果找不到kadmin.local命令，可以使用find / -name kadmin 来查找
一般的位置为：/usr/bin/kadmin

#### 5.2.5查看用户列表 listprincs

**`listprincs`**

```shell
[root@node01 ~]# kadmin
Authenticating as principal root/admin@XICHUAN.COM with password.
Password for root/admin@XICHUAN.COM: 
kadmin:  listprincs
HTTP/node01@XICHUAN.COM
HTTP/node02@XICHUAN.COM
HTTP/node03@XICHUAN.COM
K/M@XICHUAN.COM
cloudera-scm/admin@XICHUAN.COM
hdfs/node01@XICHUAN.COM
hdfs/node02@XICHUAN.COM
hdfs/node03@XICHUAN.COM
hive/node01@XICHUAN.COM
httpfs/node01@XICHUAN.COM
hue/node03@XICHUAN.COM
impala/node01@XICHUAN.COM
impala/node02@XICHUAN.COM
impala/node03@XICHUAN.COM
kadmin/admin@XICHUAN.COM
kadmin/changepw@XICHUAN.COM
kadmin/node02@XICHUAN.COM
kiprop/node02@XICHUAN.COM
krbtgt/XICHUAN.COM@XICHUAN.COM
kudu/node01@XICHUAN.COM
kudu/node02@XICHUAN.COM
kudu/node03@XICHUAN.COM
mapred/node01@XICHUAN.COM
root/admin@XICHUAN.COM
yarn/node01@XICHUAN.COM
yarn/node02@XICHUAN.COM
yarn/node03@XICHUAN.COM
zookeeper/node01@XICHUAN.COM
zookeeper/node02@XICHUAN.COM
zookeeper/node03@XICHUAN.COM
kadmin:  

```

#### 5.2.6修改账号密码change_password

**`change_password root/admin@XICHUAN.COM`**

```shell
[root@node01 ~]# kadmin
Authenticating as principal root/admin@XICHUAN.COM with password.
Password for root/admin@XICHUAN.COM: 
kadmin:  change_password root/admin@XICHUAN.COM
Enter password for principal "root/admin@XICHUAN.COM": root
Re-enter password for principal "root/admin@XICHUAN.COM": root
Password for "root/admin@XICHUAN.COM" changed.
kadmin: 

```

#### 5.2.7创建用户addprinc

**`addprinc -pw root xichuan/admin   #创建用户名hadoop用户,密码为123456 `**

```shell
[root@node01 ~]# kadmin
Authenticating as principal root/admin@XICHUAN.COM with password.
Password for root/admin@XICHUAN.COM: root
kadmin:  addprinc -pw root xichuan/admin   #创建用户名hadoop用户,密码为123456 
WARNING: no policy specified for xichuan/admin@XICHUAN.COM; defaulting to no policy
Principal "xichuan/admin@XICHUAN.COM" created.


```

创建用户时，不带REALM时，使用默认的REALM

#### 5.2.8删除用户delprinc

**`delete_principal test/admin`**

```shell
[root@node01 ~]# kadmin
Authenticating as principal root/admin@XICHUAN.COM with password.
Password for root/admin@XICHUAN.COM: root
kadmin:  delete_principal test/admin
Are you sure you want to delete the principal "test/admin@XICHUAN.COM"? (yes/no): yes
Principal "test/admin@XICHUAN.COM" deleted.
Make sure that you have removed this principal from all ACLs before reusing.
kadmin:

```

#### 5.2.9导出keytab文件

**`ktadd -k /opt/xichuan.keytab -norandkey xichuan/admin@XICHUAN.COM`**

```shell
[root@node02 etc]# kadmin.local
Authenticating as principal root/admin@XICHUAN.COM with password.
kadmin.local:  ktadd -k /opt/xichuan.keytab -norandkey xichuan/admin@XICHUAN.COM
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type aes256-cts-hmac-sha1-96 added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type aes128-cts-hmac-sha1-96 added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type des3-cbc-sha1 added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type arcfour-hmac added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type camellia256-cts-cmac added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type camellia128-cts-cmac added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type des-hmac-sha1 added to keytab WRFILE:/opt/xichuan.keytab.
Entry for principal xichuan/admin@XICHUAN.COM with kvno 1, encryption type des-cbc-md5 added to keytab WRFILE:/opt/xichuan.keytab.


```

`-norandkey` 表示密码不改变

默认导出的目录为当前目录，在kadmin.local命令上操作

#### 5.2.10查看keytab文件中的用户列表

**`klist -ket /opt/xichuan.keytab`**

```shell
[root@node02 opt]#  klist -ket /opt/xichuan.keytab 
Keytab name: FILE:/opt/xichuan.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (aes256-cts-hmac-sha1-96) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (aes128-cts-hmac-sha1-96) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (des3-cbc-sha1) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (arcfour-hmac) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (camellia256-cts-cmac) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (camellia128-cts-cmac) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (des-hmac-sha1) 
   1 10/26/2022 15:41:35 xichuan/admin@XICHUAN.COM (des-cbc-md5) 


```

#### 5.2.11更新票据kinit

**`kinit -R`**

```shell
[root@node01 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: xichuan/admin@XICHUAN.COM

Valid starting       Expires              Service principal
10/26/2022 15:45:07  10/27/2022 15:45:07  krbtgt/XICHUAN.COM@XICHUAN.COM
	renew until 11/02/2022 15:45:07
[root@node01 ~]# kinit -R
[root@node01 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: xichuan/admin@XICHUAN.COM

Valid starting       Expires              Service principal
10/26/2022 15:45:54  10/27/2022 15:45:54  krbtgt/XICHUAN.COM@XICHUAN.COM
	renew until 11/02/2022 15:45:07


```

可以看到这两个Ticket的Expires时间不一样

#### 5.2.12查看Ticket详细信息

**`getprinc xichuan/admin`**

```shell
[root@node01 ~]# kadmin
Authenticating as principal xichuan/admin@XICHUAN.COM with password.
Password for xichuan/admin@XICHUAN.COM: root
kadmin:  getprinc xichuan/admin
Principal: xichuan/admin@XICHUAN.COM
Expiration date: [never]
Last password change: Wed Oct 26 15:24:39 CST 2022
Password expiration date: [never]
Maximum ticket life: 1 day 01:00:00
Maximum renewable life: 8 days 00:00:00
Last modified: Wed Oct 26 15:24:39 CST 2022 (root/admin@XICHUAN.COM)
Last successful authentication: [never]
Last failed authentication: [never]
Failed password attempts: 0
Number of keys: 8
Key: vno 1, aes256-cts-hmac-sha1-96
Key: vno 1, aes128-cts-hmac-sha1-96
Key: vno 1, des3-cbc-sha1
Key: vno 1, arcfour-hmac
Key: vno 1, camellia256-cts-cmac
Key: vno 1, camellia128-cts-cmac
Key: vno 1, des-hmac-sha1
Key: vno 1, des-cbc-md5
MKey: vno 1
Attributes:
Policy: [none]
kadmin:
```