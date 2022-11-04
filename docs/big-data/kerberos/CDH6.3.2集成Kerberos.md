#  CDH6.3.2集成Kerberos

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

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211128163.png)

（１）客户端执行kinit命令，输入Principal及Password，向AS证明身份，并请求获取TGT。
（２）AS检查Database中是否存有客户端输入的Principal，如有则向客户端返回TGT。
（３）客户端获取TGT后，向TGS请求ServerTicket。
（４）TGS收到请求，检查Database中是否存有客户端所请求服务的Principal，如有则向客户端返回ServerTicket。
（５）客户端收到ServerTicket，则向目标服务发起请求。
（６）目标服务收到请求，响应客户端。



## 四.参考doc

CDH enable kerberos: [Kerberos Security Artifacts Overview | 6.3.x | Cloudera Documentation](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cm_sg_principal_keytab.html)

CDH disable kerberos:https://www.sameerahmad.net/blog/disable-kerberos-on-CDH; https://community.cloudera.com/t5/Support-Questions/Disabling-Kerberos/td-p/19654



## 五.Kerberos部署

### 5.1 概述

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

### 5.3 Kerberos安装

在192.168.0.200上安装服务端：

```bash
yum -y install krb5-server openldap-clients
```

在192.168.0.[201-202]上安装客户端：

```bash
pssh -h slaves -P -l root -A "yum -y install krb5-devel krb5-workstation"
```

在安装完上述的软件之后，会在KDC主机上生成配置文件/etc/krb5.conf和/var/kerberos/krb5kdc/kdc.conf，它们分别反映了realm name 以及 domain-to-realm mappings。



### 5.4 修改配置

#### 5.4.1 配置krb5.conf

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



#### 5.4.2 配置kdc.conf

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



#### 5.4.3 配置kadm5.acl

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



### 5.5 创建Kerberos数据库

在192.168.0.200上对数据库进行初始化，默认的数据库路径为`/var/kerberos/krb5kdc`，如果需要重建数据库，将该目录下的principal相关的文件删除即可，请牢记数据库密码。

```bash
kdb5_util create -r XICHUAN.COM -s
```

**说明：**

**[-s]** 表示生成 stash file ，并在其中存储master server key（krb5kdc）
**[-r]** 来指定一个realm name ， 当krb5.conf中定义了多个realm时使用
当 Kerberos database创建好之后，在 /var/kerberos/ 中可以看到生成的 principal相关文件

### 5.6 启动Kerberos服务

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



### 5.7 创建Kerberos管理员principal

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



## 六.CDH集成Kerberos

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





## 七.Kerberos使用

### 7.1 Kerberos主体Principal介绍

在官网上有详细的介绍Principal的介绍，链接为：http://web.mit.edu/kerberos/krb5-1.5/krb5-1.5.4/doc/krb5-user/What-is-a-Kerberos-Principal_003f.html

主体是Kerberos可以为其分配票证的唯一标识。主体可以具有任意数量的组件。每个组件都由一个组件分隔符（通常为 /）分隔。最后一个组成部分是领域，由领域分隔符（通常为@）与主体的其余部分分隔。如果主体中没有领域组件，则将假定主体在使用它的上下文中位于默认领域中

Principal的格式一般为：primary/instance@REALM

primary是Principal的一部分，如果是用户，则为用户名，如果是主机则为主机名。

instance是一个可选字符串，用于限定主服务器。该实例与主实例之间用斜杠（/）隔开。对于用户而言，该实例通常为null，但用户可能还具有一个附加的主体，即一个名为admin的实例，用该实例来管理数据库。主体jennifer@XICHUAN.COM与主体root/admin@XICHUAN.COM完全分开 ，具有单独的密码和单独的权限。对于主机，实例是标准主机名，例如 xichuan.com。

REALM是您的Kerberos领域。在大多数情况下，您的Kerberos域是您的域名，以大写字母表示。例如，机器daffodil.example.com就在领域中EXAMPLE.COM



### 7.2 Kerberos 命令使用

```
kadmin.local 与 kadmin区别：
kadmin.local：需要在 KDC server 上面操作，无需密码即可管理资料库
kadmin：可以在任何一台 KDC 领域的系统上面操作，但是需要输入管理员密码
```



#### 7.2.1登陆kinit

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

#### 7.2.2查询登陆状态klist

**`klist`**

```shell
[root@node01 ~]# klist
Ticket cache: FILE:/tmp/krb5cc_0
Default principal: root/admin@XICHUAN.COM

Valid starting       Expires              Service principal
10/26/2022 15:19:15  10/27/2022 15:19:15  krbtgt/XICHUAN.COM@XICHUAN.COM
	renew until 11/02/2022 15:19:15
```

#### 7.2.3退出登陆kdestroy

**`kdestroy`**

```shell
[root@node01 ~]# kdestroy
[root@node01 ~]# klist
klist: No credentials cache found (filename: /tmp/krb5cc_0)
```

#### 7.2.4登录KDC后台 kadmin.local

**`kadmin.local`**

```shell
[root@node02 etc]# kadmin.local
Authenticating as principal root/admin@XICHUAN.COM with password.
kadmin.local:  

```

如果找不到kadmin.local命令，可以使用find / -name kadmin 来查找
一般的位置为：/usr/bin/kadmin

#### 7.2.5查看用户列表 listprincs

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

#### 7.2.6修改账号密码change_password

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

#### 7.2.7创建用户addprinc

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

#### 7.2.8删除用户delprinc

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

#### 7.2.9导出keytab文件

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

#### 7.2.10查看keytab文件中的用户列表

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

#### 7.2.11更新票据kinit

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

#### 7.2.12查看Ticket详细信息

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



### 7.3 访问hdfs

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



### 7.4 访问hive

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



### 7.5 访问impala-shell

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

### 7.6 错误记录

#### 7.6.1 hdfs相关指令报错

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





### 7.7 Dbever访问Kerberos环境下的Impala

#### 7.7.1 windows下安装kfw客户端

下载地址：[https://web.mit.edu/kerberos/dist/index.html](https://links.jianshu.com/go?to=https%3A%2F%2Fweb.mit.edu%2Fkerberos%2Fdist%2Findex.html)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271555796.png)

安装过程没什么好说的，傻瓜式安装，唯一需要注意的是：

安装之后不要点击重启(其实也可以，但是没必要)！不要打开软件!

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210271558464.png)



####  7.7.2 修改C:\ProgramData\MIT\Kerberos5\krb5.ini文件

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



####  7.7.3 修改环境变量

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



####  7.7.4 修改dbeaver配置文件和连接配置

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



### 7.8 Windows访问HDFS/YARN/HIVESERVER2 等服务的webui

> 参考：https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_sg_browser_access_kerberos_protected_url.html#topic_6_2__section_vj5_gwv_ls

#### 7.8.1 HDFS/YARN/HIVESERVER2开启webui验证

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



#### 7.8.2 修改配置并进行验证后访问

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





### 7.9代码中的认证

#### 7.9.1 spark程序的认证

```scala
  val krb5ConfPath = "D:\\development\\license_dll\\krb5.conf"
  val keyTabPath = "D:\\development\\license_dll\\xichuan.keytab"
  val principle = "xichuan/admin@XICHUAN.COM"

  def kerberosAuth(krb5ConfPath:String,keytabPath:String,principle:String): Unit ={
    val conf = new Configuration
    System.setProperty("java.security.krb5.conf", krb5ConfPath)

    //conf.addResource(new Path("C:\\Users\\Leon\\Desktop\\hive-conf\\hive-site.xml"))
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



#### 7.9.2 Java连接impala的认证

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



#### 7.9.3 Springboot使用hikari连接池并进行Kerberos认证访问Impala

项目与文档地址：https://github.com/Raray-chuan/springboot-kerberos-hikari-impala



## 八.CDH disable Kerberos

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





