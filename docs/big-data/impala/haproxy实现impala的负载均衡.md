# HAProxy实现Impala的负载均衡



## 1.HAProxy安装及启停

**1.1 在集群中选择一个节点，使用yum方式安装HAProxy服务**

```
[root@data01-dev ~]# yum -y install haproxy
```

**1.2 启动与停止HAProxy服务，并将服务添加到自启动列表**

```
[root@data01-dev ~]# service haproxy start
[root@data01-dev ~]# service haproxy stop
[root@data01-dev ~]# chkconfig haproxy on
[root@data01-dev ~]#  service haproxy status
Redirecting to /bin/systemctl status haproxy.service
● haproxy.service - HAProxy Load Balancer
   Loaded: loaded (/usr/lib/systemd/system/haproxy.service; enabled; vendor preset: disabled)
   Active: active (running) since Thu 2022-10-20 19:29:46 CST; 19h ago
 Main PID: 27994 (haproxy-systemd)
   CGroup: /system.slice/haproxy.service
           ├─27994 /usr/sbin/haproxy-systemd-wrapper -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid
           ├─27995 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds
           └─27996 /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

Oct 20 19:29:46 data01-dev systemd[1]: Started HAProxy Load Balancer.
Oct 20 19:29:46 data01-dev haproxy-systemd-wrapper[27994]: haproxy-systemd-wrapper: executing /usr/sbin/haproxy -f /etc/haproxy/haproxy.cfg -p /run/haproxy.pid -Ds

```



**3.HAProxy配置Impala负载均衡**

1.将`/etc/haproxy`目录下的`haproxy.cfg`文件备份，新建`haproxy.cfg`文件，添加如下配置

```
#---------------------------------------------------------------------
# Example configuration for a possible web application.  See the
# full configuration options online.
#
#   http://haproxy.1wt.eu/download/1.4/doc/configuration.txt
#
#---------------------------------------------------------------------

#---------------------------------------------------------------------
# Global settings
#---------------------------------------------------------------------
global
    # to have these messages end up in /var/log/haproxy.log you will
    # need to:
    #
    # 1) configure syslog to accept network log events.  This is done
    #    by adding the '-r' option to the SYSLOGD_OPTIONS in
    #    /etc/sysconfig/syslog
    #
    # 2) configure local2 events to go to the /var/log/haproxy.log
    #   file. A line like the following can be added to
    #   /etc/sysconfig/syslog
    #
    #    local2.*                       /var/log/haproxy.log
    #
    log         127.0.0.1 local2

    chroot      /var/lib/haproxy
    pidfile     /var/run/haproxy.pid
    maxconn     4000
    user        haproxy
    group       haproxy
    daemon

    # turn on stats unix socket
    stats socket /var/lib/haproxy/stats

#---------------------------------------------------------------------
# common defaults that all the 'listen' and 'backend' sections will
# use if not designated in their block
#---------------------------------------------------------------------
defaults
    mode                    http
    log                     global
    option                  httplog
    option                  dontlognull
    option http-server-close
    #option forwardfor       except 127.0.0.0/8
    option                  redispatch
    retries                 3
    timeout http-request    10s
    timeout queue           1m
    timeout connect         10s
    timeout client          1m
    timeout server          1m
    timeout http-keep-alive 10s
    timeout check           10s
    maxconn                 3000

#---------------------------------------------------------------------
# main frontend which proxys to the backends
#---------------------------------------------------------------------
frontend  main *:5000
    acl url_static       path_beg       -i /static /images /javascript /stylesheets
    acl url_static       path_end       -i .jpg .gif .png .css .js

    use_backend static          if url_static
    default_backend             app

#---------------------------------------------------------------------
# static backend for serving up images, stylesheets and such
#---------------------------------------------------------------------
backend static
    balance     roundrobin
    server      static 127.0.0.1:4331 check

#---------------------------------------------------------------------
# round robin balancing between the various backends
#---------------------------------------------------------------------
backend app
    balance     roundrobin
    server  app1 127.0.0.1:5001 check
    server  app2 127.0.0.1:5002 check
    server  app3 127.0.0.1:5003 check
    server  app4 127.0.0.1:5004 check
	
listen status
	bind data01-dev:1080
	mode http
	option httplog
	maxconn 5000
	stats refresh 30s
	stats uri /haproxy
	stats enable
	stats realm Global\ statistics
	stats auth admin:admin
	
listen impala_shell data01-dev:25003
    mode tcp
    option tcplog
    balance leastconn
    #主机列表
    server impala1 impala01-dev:21000
    server impala2 impala02-dev:21000
    server impala3 impala03-dev:21000
    server impala4  data01-dev:21000
    server impala5  data02-dev:21000
    server impala6  data03-dev:21000

listen impala_jdbc data01-dev:25004
    mode tcp
    option tcplog
    balance roundrobin
    #主机列表
    server impala1 impala01-dev:21050
    server impala2 impala02-dev:21050
    server impala3 impala03-dev:21050
    server impala4  data01-dev:21050
    server impala5  data02-dev:21050
    server impala6  data03-dev:21050
	
log 127.0.0.1 local0 info


```

主要配置了HAProxy的http状态管理界面、impalashell和impalajdbc的负载均衡。



## 2. 重启HAProxy服务

```
[root@data01-dev haproxy]# service haproxy restart
```



## 3.浏览器访问

访问地址： `http://{hostname}:1080/stats`



## 4.Impala Shell测试

使用多个终端同时访问，并执行SQL语句，查看是否会通过HAProxy服务自动负载到其它Impala Daemon节点

1.使用Impala shell访问HAProxy服务的25003端口，命令如下

```
[root@data01-dev ~]# impala-shell -i data01-dev:25003
```

 

2.打开第一个终端访问并执行SQL

```
[root@data01-dev ~]# impala-shell -i data01-dev:25003
Starting Impala Shell without Kerberos authentication
Opened TCP connection to data01-dev:25003
Connected to data01-dev:25003
Server version: impalad version 3.2.0-cdh6.3.2 RELEASE (build 1bb9836227301b839a32c6bc230e35439d5984ac)
***********************************************************************************
Welcome to the Impala shell.
(Impala Shell v3.2.0-cdh6.3.2 (1bb9836) built on Fri Nov  8 07:22:06 PST 2019)

Want to know what version of Impala you're connected to? Run the VERSION command to
find out!
***********************************************************************************
[data01-dev:25003] default> select * from test_xac_tmp.xichuan_test limit 1;
Connection lost, reconnecting...
Opened TCP connection to data01-dev:25003
Query: select * from test_xac_tmp.xichuan_test limit 1
Query submitted at: 2022-10-21 14:46:20 (Coordinator: http://data02-dev:25000)
Query progress can be monitored at: http://data02-dev:25000/query_plan?query_id=074d7571e7cd831e:c2f9b34300000000
+----+-----------+------+
| id | continent | area |
+----+-----------+------+
| 1  | a         | 1    |
+----+-----------+------+
Fetched 1 row(s) in 5.04s

```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211456379.png)

3.同时打开第二个终端访问并执行SQL

```
[root@master01-dev ~]# impala-shell -i data01-dev:25003
Starting Impala Shell without Kerberos authentication
Opened TCP connection to data01-dev:25003
Connected to data01-dev:25003
Server version: impalad version 3.2.0-cdh6.3.2 RELEASE (build 1bb9836227301b839a32c6bc230e35439d5984ac)
***********************************************************************************
Welcome to the Impala shell.
(Impala Shell v3.2.0-cdh6.3.2 (1bb9836) built on Fri Nov  8 07:22:06 PST 2019)

You can change the Impala daemon that you're connected to by using the CONNECT
command.To see how Impala will plan to run your query without actually executing
it, use the EXPLAIN command. You can change the level of detail in the EXPLAIN
output by setting the EXPLAIN_LEVEL query option.
***********************************************************************************
[data01-dev:25003] default> select * from test_xac_tmp.xichuan_test limit 1;
Connection lost, reconnecting...
Opened TCP connection to data01-dev:25003
Query: select * from test_xac_tmp.xichuan_test limit 1
Query submitted at: 2022-10-21 14:46:26 (Coordinator: http://data03-dev:25000)
Query progress can be monitored at: http://data03-dev:25000/query_plan?query_id=c740bb9f6d8c326f:d585881000000000
+----+-----------+------+
| id | continent | area |
+----+-----------+------+
| 1  | a         | 1    |
+----+-----------+------+
Fetched 1 row(s) in 0.03s

```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202210211457407.png)

通过以上测试可以看到，两个终端执行的SQL不在同一个Impala Daemon，这样就实现了Impala Daemon服务的负载均衡。



## 5.Impala JDBC测试

这里Java的测试工程就不详细描述如何创建了

配置JDBC的地址为HAProxy服务所在的IP端口为25004

