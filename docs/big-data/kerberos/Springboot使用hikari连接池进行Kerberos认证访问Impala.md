# springboot-kerberos-hikari-impala
Springboot使用hikari连接池并进行Kerberos认证访问Impala的演示项目



Springboot使用hikari连接池并进行Kerberos认证访问Impala的demo地址:https://github.com/Raray-chuan/springboot-kerberos-hikari-impala

修改后的Hikari源码地址:https://github.com/Raray-chuan/HikariCP-4.0.3



```
springboot版本:2.7.5
HikariCP版本:4.0.3
impala驱动版本:2.6.3
CDH集群的Hadoop版本:3.0.0-cdh6.3.2
```



## 1. Hikari源码改造,以支持Kerberos认证

如果想了解为什么要修改Hikari来支持Kerberos认证，以及如何修改Hikari源码，请到下面地址查看:`https://github.com/Raray-chuan/HikariCP-4.0.3 `, 此项目中有详细的readme解答

如果不想手动修改源码以及编译，而且HikariCP版本为4.0.3，在项目的resources目录中有编译好的HikariCP-4.0.3.jar可供使用





## 2. 准备测试的hive数据



**1.在hive中创建表与导入数据：**

```shell
create database test_xichuan_db;

CREATE TABLE test_xichuan_db.test_xichuan_table (
  start_time String,
  id STRING
)
row format delimited fields terminated by ','
stored as textfile;


测试数据：sample.txt
20211107,id1
20211207,id2
20211210,id3
20211214,id4
20211220,id5
20211222,id6
20211228,id7
20211101,id7
20211227,id4
20211228,id3
20211229,id2
20211230,id1


LOAD DATA LOCAL INPATH '/home/xichuan/sample.txt'
OVERWRITE INTO TABLE test_xichuan_db.test_xichuan_table;

```



![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041519082.png)



**2.在impala中查询数据**

```shell
select * from test_xichuan_db.test_xichuan_table;
```

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041526242.png)



## 3.项目说明

此项目是一个很简单的springboot的demo项目，项目结构以及每个注解就不详细说了，而且项目中是通过principal+keytab的方式进行认证的，Kerberos如何使用，请参考:https://raray-chuan.github.io/xichuan_note/#/docs/big-data/kerberos/CDH6.3.2集成Kerberos



此项目很简单，我们在DataSourceConfig中自定义dataSource这个spring bean,这样我们使用mybatis进行impala查询的时候，会自动获取到dataSource这个bean，就可以不用修改代码了。



我们修改了Hikari的源码，在HikariConfig类中添加了Kerberos四个参数，分别是：

```
authenticationType:安全验证的类型,如果值是"kerberos"，则进行Kerberos认证，如果为null，则不进行认证
krb5FilePath:krb5.conf文件的路径
principal:principal的名称
keytabPath:对应principal的keytab的路径
```

这样我们在创建HikariDataSource的时候，就可以使用HikariDataSource(HikariConfig configuration)这个构造方法创建即可将Kerberos参数传入其中。DataSourceConfig代码如下：

```java
package com.xichuan.dev.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import javax.sql.DataSource;
import java.sql.*;

/**
 * @Author Xichuan
 * @Date 2022/11/1 15:15
 * @Description
 */
@Configuration
public class DataSourceConfig {
    private Logger logger = LoggerFactory.getLogger(DataSourceConfig.class);

    @Value("${authentication.type}")
    private String authenticationType;

    @Value("${authentication.kerberos.krb5FilePath}")
    private String krb5FilePath;

    @Value("${authentication.kerberos.principal}")
    private String principal;

    @Value("${authentication.kerberos.keytabPath}")
    private String keytabPath;

    /**
     * inint datasource
     * @return
     */
    @Bean
    public DataSource dataSource(DataSourceProperties dataSourceProperties) throws SQLException {
        HikariConfig config = new HikariConfig();
        //kerberos config
        config.setAuthenticationType(authenticationType);
        config.setKrb5FilePath(krb5FilePath);
        config.setPrincipal(principal);
        config.setKeytabPath(keytabPath);

        //jdbc and pool config
        config.setJdbcUrl(dataSourceProperties.getUrl());
        config.setDriverClassName(dataSourceProperties.getDriverClassName());
        config.setUsername(dataSourceProperties.getUsername());
        config.setPassword(dataSourceProperties.getPassword());
        config.setPoolName(dataSourceProperties.getPoolName());
        config.setReadOnly(dataSourceProperties.isReadOnly());
        config.setAutoCommit(dataSourceProperties.isAutoCommit());
        config.setMaximumPoolSize(dataSourceProperties.getMaximumPoolSize());
        //maxLifetime 池中连接最长生命周期
        config.setMaxLifetime(dataSourceProperties.getMaxLifetime());
        //等待来自池的连接的最大毫秒数 30000
        config.setIdleTimeout(dataSourceProperties.getIdleTimeout());
        //连接将被测试活动的最大时间量
        config.setValidationTimeout(dataSourceProperties.getValidationTimeout());


        HikariDataSource dataSource = new HikariDataSource(config);
        logger.info("init new dataSource: {}", dataSource);
        return dataSource;
    }
}
```









## 4. 修改application-prod.yml中的参数，以符合自己的测试环境

```sh
# datasource and pool
# 修改为自己的impala链接地址
datasource.xichuan.url=jdbc:impala://node01:21050/;AuthMech=1;KrbRealm=XICHUAN.COM;KrbHostFQDN=node01;KrbServiceName=impala
datasource.xichuan.driver-class-name=com.cloudera.impala.jdbc41.Driver
datasource.xichuan.username=
datasource.xichuan.password=
datasource.xichuan.pool-name=xichuan-pool
datasource.xichuan.read-only=false
datasource.xichuan.auto-commit=true
datasource.xichuan.maximum-pool-size=3
# 此处是演示值，让connection的最长生存时长为35秒
datasource.xichuan.max-lifetime=35000
datasource.xichuan.idle-timeout=10000
datasource.xichuan.validation-timeout=5000


# kerberos
# 此值为null，则不进行Kerberos认证，如果此值为kerberos，则进行kerberos认证
authentication.type=kerberos
# 修改为服务器中的krb5.conf地址，或者是本地的krb5.conf地址
authentication.kerberos.krb5FilePath=D:\\development\\license_dll\\krb5.conf
# 修改为自己的principle名称
authentication.kerberos.principal=xichuan/admin@XICHUAN.COM
# 修改为服务器中的keytab地址，或者是本地的keytab地址
authentication.kerberos.keytabPath=D:\\development\\license_dll\\xichuan.keytab
```





## 5. 项目验证演示

**1.启动项目**

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041537566.png)

我们可以看出刚启动项目后，Hikari连接池已经进行了Kerberos认证，并创建了三个Connection



**2.访问数据**

访问接口：`http://localhost:18080/lot_operation/all_data`

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041542280.png)

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041543639.png)

我们可以看出访问访问数据一切正常



**3.验证HikariPool中Connection失效后，自动Kerberos验证并创建新的Connection**

我们设置的`datasource.xichuan.max-lifetime=35000`，所以每过`35秒`，`HikariPool`中的`Connection`就会失效，并重新生成新的`Connection`,我们查看日志：

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211041559195.png)

通过日志可以看出，`HikariPool`中的`Connection`每过35秒就会失效，但`HikariPool`会通过Kerberos认证，并创建新的`Connection`添加到`HikariPool`中。

