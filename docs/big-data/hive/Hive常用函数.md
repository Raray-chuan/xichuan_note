# Hive常用函数

## 1. Hive的参数传递

### 1.1 Hive命令行

- 查看hive命令的参数

```shell
[hadoop@node03 ~]$ hive -help
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151138157.png)

语法结构:

- hive [-hiveconf x=y]* [<-i filename>]* [<-f filename>|<-e query-string>][-S]

- 说明：
  - -i 从文件初始化HQL。
  - -e从命令行执行指定的HQL 
  - -f 执行HQL脚本 
  - -v 输出执行的HQL语句到控制台 
  - -p <port> connect to Hive Server on port number 
  - -hiveconf x=y Use this to set hive/hadoop configuration variables.  设置hive运行时候的参数配置

### 1.2 Hive参数配置方式

- Hive参数大全：
- [官网地址](<https://cwiki.apache.org/confluence/display/Hive/Configuration+Properties>)
- 开发Hive应用时，不可避免地需要设定Hive的参数。设定Hive的参数可以调优HQL代码的执行效率，或帮助定位问题。然而实践中经常遇到的一个问题是，为什么设定的参数没有起作用？这通常是错误的设定方式导致的。

- **对于一般参数，有以下三种设定方式：**
  - 配置文件  hive-site.xml

  - 命令行参数  启动hive客户端的时候可以设置参数

  - 参数声明   进入客户端以后设置的一些参数  set  

- **配置文件**
  - Hive的配置文件包括
    - 用户自定义配置文件：$HIVE_CONF_DIR/hive-site.xml 
    - 默认配置文件：$HIVE_CONF_DIR/hive-default.xml 
  - 用户自定义配置会覆盖默认配置。
  - 另外，Hive也会读入Hadoop的配置，因为Hive是作为Hadoop的客户端启动的，Hive的配置会覆盖Hadoop的配置。配置文件的设定对本机启动的所有Hive进程都有效。

- **命令行参数**：启动Hive（客户端或Server方式）时，可以在命令行添加-hiveconf param=value来设定参数，例如：

```shell
bin/hive --hiveconf hive.root.logger=INFO,console
```

​		这一设定只对本次启动的Session（对于Server方式启动，则是所有请求的Sessions）有效。

- **参数声明**：可以在HQL中使用SET关键字设定参数，例如：

```sql
-- 设置mr中reduce个数
set mapreduce.job.reduces=100;
```

​		这一设定的作用域也是session级的。

- 上述三种设定方式的优先级依次递增。
  - 即参数声明覆盖命令行参数，命令行参数覆盖配置文件设定。
  - 注意某些系统级的参数，例如log4j相关的设定，必须用前两种方式设定，因为那些参数的读取在Session建立以前已经完成了。

```
参数声明  >   命令行参数   >  配置文件参数（hive）
```

### 1.3 使用变量传递参数

- 实际工作当中，我们一般都是将hive的hql语法开发完成之后，就写入到一个脚本里面去，然后定时的通过命令 hive  -f  去执行hive的语法即可
- 然后通过定义变量来传递参数到hive的脚本当中去，那么我们接下来就来看看如何使用hive来传递参数。

- hive0.9以及之前的版本是不支持传参
- hive1.0版本之后支持  hive -f 传递参数

- 在hive当中我们一般可以使用`hivevar`或者`hiveconf`来进行参数的传递

#### 1.3.1 hiveconf使用说明

- hiveconf用于定义HIVE**执行上下文的属性**(配置参数)，可覆盖覆盖hive-site.xml（hive-default.xml）中的参数值，如用户执行目录、日志打印级别、执行队列等。例如我们可以使用hiveconf来覆盖我们的hive属性配置，

- hiveconf变量取值必须要使用hiveconf作为前缀参数，具体格式如下:

```sql
${hiveconf:key} 
bin/hive --hiveconf "mapred.job.queue.name=root.default"
```

#### 1.3.2 hivevar使用说明

- hivevar用于定义HIVE**运行时的变量**替换，类似于JAVA中的“PreparedStatement”，与\${key}配合使用或者与 ${hivevar:key}

- 对于hivevar取值可以不使用前缀hivevar，具体格式如下：

```sql
-- 使用前缀:
 ${hivevar:key}
-- 不使用前缀:
 ${key}
hive --hivevar  name=zhangsan    

${hivevar:name}  
也可以这样取值  ${name}
```

#### 1.3.3 define使用说明

- define与hivevar用途完全一样，还有一种简写“-d

```sql
hive --hiveconf "mapred.job.queue.name=root.default" -d my="201912" --database myhive

-- 执行SQL
hive > select * from myhive.score2 where concat(year, month) = ${my} limit 5;
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151140733.png)

#### 1.3.4 hiveconf与hivevar使用实战

- 需求：hive当中执行以下hql语句，并将'201807'、'80'、'03'用参数的形式全部都传递进去

```sql
select * from student left join score on student.s_id = score.s_id where score.month = '201807' and score.s_score > 80 and score.c_id = 03;
```

##### 第一步：创建student表并加载数据

```sql
hive (myhive)> create external table student
(s_id string, s_name string, s_birth string, s_sex string) row format delimited
fields terminated by '\t';

hive (myhive)> load data local inpath '/opt/install/hivedatas/student.csv' overwrite into table student;
```

##### 第二步：定义hive脚本

- 开发hql脚本，并使用hiveconf和hivevar进行参数传入

- node03执行以下命令定义hql脚本

```sql
cd /opt/instal/hivedatas

vim hivevariable.hql

use myhive;
select * from student left join score on student.s_id = score.s_id where score.month = ${hiveconf:month} and score.s_score > ${hivevar:s_score} and score.c_id = ${c_id};   
```

##### 第三步：调用hive脚本并传递参数

- student、score表内容如下

```
hive (myhive)> select * from student;
OK
student.s_id    student.s_name  student.s_birth student.s_sex
01      赵雷    1990-01-01      男
02      钱电    1990-12-21      男
03      孙风    1990-05-20      男
04      李云    1990-08-06      男
05      周梅    1991-12-01      女
06      吴兰    1992-03-01      女
07      郑竹    1989-07-01      女
08      王菊    1990-01-20      女
Time taken: 0.048 seconds, Fetched: 8 row(s)
hive (myhive)> select * from score;
OK
score.s_id      score.c_id      score.s_score   score.month
01      01      80      201912
01      02      90      201912
01      03      99      201912
02      01      70      201912
02      02      60      201912
02      03      80      201912
03      01      80      201912
03      02      80      201912
03      03      80      201912
04      01      50      201912
04      02      30      201912
04      03      20      201912
05      01      76      201912
05      02      87      201912
06      01      31      201912
06      03      34      201912
07      02      89      201912
07      03      98      201912
```

- node03执行以下命令并

```sql
hive --hiveconf month=201912 --hivevar s_score=80 --hivevar c_id=03  -f /opt/install/hivedatas/hivevariable.hql
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151140432.png)



## 2. Hive的常用函数

### 2.1 系统内置函数

```
1．查看系统自带的函数
hive> show functions;
2．显示自带的函数的用法
hive> desc function upper;
3．详细显示自带的函数的用法
hive> desc function extended upper;
```

### 2.2 数值计算

#### 2.2.1 取整函数: round 

- **语法**: round(double a)
- **返回值**: BIGINT
- **说明**: 返回double类型的整数值部分 （遵循四舍五入）

```sql
hive> select round(3.1415926) from tableName;
3
hive> select round(3.5) from tableName;
4
hive> create table tableName as select round(9542.158) from tableName;
```

#### 2.2.2 指定精度取整函数: round 

- **语法**: round(double a, int d)
- **返回值**: DOUBLE
- **说明**: 返回指定精度d的double类型

```sql
hive> select round(3.1415926, 4) from tableName;
3.1416
```

#### 2.2.3 向下取整函数: floor 

- **语法**: floor(double a)
- **返回值**: BIGINT
- **说明**: 返回等于或者小于该double变量的最大的整数

```sql
hive> select floor(3.1415926) from tableName;
3
hive> select floor(25) from tableName;
25
```

#### 2.2.4 向上取整函数: ceil 

- **语法**: ceil(double a)
- **返回值**: BIGINT
- **说明**: 返回等于或者大于该double变量的最小的整数

```sql
hive> select ceil(3.1415926) from tableName;
4
hive> select ceil(46) from tableName;
46
```

#### 2.2.5 向上取整函数: ceiling 

- **语法**: ceiling(double a)
- **返回值**: BIGINT
- **说明**: 与ceil功能相同

```sql
hive> select ceiling(3.1415926) from tableName;
4
hive> select ceiling(46) from tableName;
46
```

#### 2.2.6 取随机数函数: rand

- **语法**: rand(), rand(int seed)
- **返回值**: double
- **说明**: 返回一个0到1范围内的随机数。如果指定种子seed，则会等到一个稳定的随机数序列

```sql
hive> select rand() from tableName;
0.5577432776034763
hive> select rand() from tableName;
0.6638336467363424
hive> select rand(100) from tableName;
0.7220096548596434
hive> select rand(100) from tableName;
0.7220096548596434
```

### 2.3 日期函数

#### 2.3.1 UNIX时间戳转日期函数: from_unixtime  

- **语法**: from_unixtime(bigint unixtime[, string format])
- **返回值**: string
- **说明**: 转化UNIX时间戳（从1970-01-01 00:00:00 UTC到指定时间的秒数）到当前时区的时间格式

```sql
hive> select from_unixtime(1323308943, 'yyyyMMdd') from tableName;
20111208
```

#### 2.3.2 获取当前UNIX时间戳函数: unix_timestamp

- **语法**: unix_timestamp()
- **返回值**: bigint
- **说明**: 获得当前时区的UNIX时间戳

```sql
hive> select unix_timestamp() from tableName;
1323309615
```

#### 2.3.3 日期转UNIX时间戳函数: unix_timestamp 

- **语法**: unix_timestamp(string date)
- **返回值**: bigint
- **说明**: 转换格式为"yyyy-MM-dd HH:mm:ss"的日期到UNIX时间戳。如果转化失败，则返回0。

```sql
hive> select unix_timestamp('2011-12-07 13:01:03') from tableName;
1323234063
```

#### 2.3.4 指定格式日期转UNIX时间戳函数: unix_timestamp 

- **语法**: unix_timestamp(string date, string pattern)
- **返回值**: bigint
- **说明**: 转换pattern格式的日期到UNIX时间戳。如果转化失败，则返回0。

```sql
hive> select unix_timestamp('20111207 13:01:03','yyyyMMdd HH:mm:ss') from tableName;
1323234063
```

#### 2.3.5 日期时间转日期函数: to_date  

- **语法**: to_date(string datetime)
- **返回值**: string
- **说明**: 返回日期时间字段中的日期部分。

```sql
hive> select to_date('2011-12-08 10:03:01') from tableName;
2011-12-08
```

#### 2.3.6 日期转年函数: year 

- **语法**: year(string date)
- **返回值**: int
- **说明**: 返回日期中的年。

```sql
hive> select year('2011-12-08 10:03:01') from tableName;
2011
hive> select year('2012-12-08') from tableName;
2012
```

#### 2.3.7 日期转月函数: month 

- **语法**: month (string date)
- **返回值**: int
- **说明**: 返回date或datetime中的月份。

```sql
hive> select month('2011-12-08 10:03:01') from tableName;
12
hive> select month('2011-08-08') from tableName;
8
```

#### 2.3.8 日期转天函数: day 

- **语法**: day (string date)
- **返回值**: int
- **说明**: 返回日期中的天。

```sql
hive> select day('2011-12-08 10:03:01') from tableName;
8
hive> select day('2011-12-24') from tableName;
24
```

#### 2.3.9 日期转小时函数: hour 

- **语法**: hour (string date)
- **返回值**: int
- **说明**: 返回日期中的小时。

```sql
hive> select hour('2011-12-08 10:03:01') from tableName;
10
```

#### 2.3.10 日期转分钟函数: minute

- **语法**: minute (string date)
- **返回值**: int
- **说明**: 返回日期中的分钟。

```sql
hive> select minute('2011-12-08 10:03:01') from tableName;
3

-- second 返回秒
hive> select second('2011-12-08 10:03:01') from tableName;
1
```

#### 2.3.11 日期转周函数: weekofyear

- **语法**: weekofyear (string date)
- **返回值**: int
- **说明**: 返回日期在当前的周数。

```sql
hive> select weekofyear('2011-12-08 10:03:01') from tableName;
49
```

#### 2.3.12 日期比较函数: datediff 

- **语法**: datediff(string enddate, string startdate)
- **返回值**: int
- **说明**: 返回结束日期减去开始日期的天数。

```sql
hive> select datediff('2012-12-08','2012-05-09') from tableName;
213
```

#### 2.3.13 日期增加函数: date_add 

- **语法**: date_add(string startdate, int days)
- **返回值**: string
- **说明**: 返回开始日期startdate增加days天后的日期。

```sql
hive> select date_add('2012-12-08',10) from tableName;
2012-12-18
```

#### 2.3.14 日期减少函数: date_sub 

- **语法**: date_sub (string startdate, int days)
- **返回值**: string
- **说明**: 返回开始日期startdate减少days天后的日期。

```sql
hive> select date_sub('2012-12-08',10) from tableName;
2012-11-28
```

### 2.4 条件函数（重点）

#### 2.4.1 If函数: if 

- **语法**: if(boolean testCondition, T valueTrue, T valueFalseOrNull)
- **返回值**: T
- **说明**: 当条件testCondition为TRUE时，返回valueTrue；否则返回valueFalseOrNull

```sql
hive> select if(1=2,100,200) from tableName;
200
hive> select if(1=1,100,200) from tableName;
100
```

#### 2.4.2 非空查找函数: COALESCE

- **语法**: COALESCE(T v1, T v2, …)
- **返回值**: T
- **说明**: 返回参数中的第一个非空值；如果所有值都为NULL，那么返回NULL

```sql
hive> select COALESCE(null,'100','50') from tableName;
100
```

#### 2.4.3 条件判断函数：CASE 

- **语法**: CASE a WHEN b THEN c [WHEN d THEN e]* [ELSE f] END
- **返回值**: T
- **说明**：如果a等于b，那么返回c；如果a等于d，那么返回e；否则返回f

```sql
hive> select case 100 when 50 then 'tom' when 100 then 'mary' else 'tim' end from tableName;
mary
hive> Select case 200 when 50 then 'tom' when 100 then 'mary' else 'tim' end from tableName;
tim
```

#### 2.4.4 条件判断函数：CASE

- **语法**: CASE WHEN a THEN b [WHEN c THEN d]* [ELSE e] END
- **返回值**: T
- **说明**：如果a为TRUE,则返回b；如果c为TRUE，则返回d；否则返回e

```sql
hive> select case when 1=2 then 'tom' when 2=2 then 'mary' else 'tim' end from tableName;
mary
hive> select case when 1=1 then 'tom' when 2=2 then 'mary' else 'tim' end from tableName;
tom
```

### 2.5 字符串函数

#### 2.5.1 字符串长度函数：length

- **语法**: length(string A)
- **返回值**: int
- **说明**：返回字符串A的长度

```sql
hive> select length('abcedfg') from tableName;
```

#### 2.5.2 字符串反转函数：reverse

- **语法**: reverse(string A)
- **返回值**: string
- **说明**：返回字符串A的反转结果

```sql
hive> select reverse('abcedfg') from tableName;
gfdecba
```

#### 2.5.3 字符串连接函数：concat

- **语法**: concat(string A, string B…)
- **返回值**: string
- **说明**：返回输入字符串连接后的结果，支持任意个输入字符串

```sql
hive> select concat('abc','def','gh') from tableName;
abcdefgh
```

#### 2.5.4 字符串连接并指定字符串分隔符：concat_ws

- **语法**: concat_ws(string SEP, string A, string B…)
- **返回值**: string
- **说明**：返回输入字符串连接后的结果，SEP表示各个字符串间的分隔符

```sql
hive> select concat_ws(',','abc','def','gh') from tableName;
abc,def,gh
```

#### 2.5.5 字符串截取函数：substr

- **语法**: substr(string A, int start), substring(string A, int start)
- **返回值**: string
- **说明**：返回字符串A从start位置到结尾的字符串

```sql
hive> select substr('abcde',3) from tableName;
cde
hive> select substring('abcde',3) from tableName;
cde
hive> select substr('abcde',-1) from tableName;  （和ORACLE相同）
e
```

#### 2.5.6 字符串截取函数：substr, substring 

- **语法**: substr(string A, int start, int len),substring(string A, int start, int len)
- **返回值**: string
- **说明**：返回字符串A从start位置开始，长度为len的字符串

```sql
hive> select substr('abcde',3,2) from tableName;
cd
hive> select substring('abcde',3,2) from tableName;
cd
hive>select substring('abcde',-3,2) from tableName;
cd
```

#### 2.5.7 字符串转大写函数：upper, ucase  

- **语法**: upper(string A) ucase(string A)
- **返回值**: string
- **说明**：返回字符串A的大写格式

```sql
hive> select upper('abSEd') from tableName;
ABSED
hive> select ucase('abSEd') from tableName;
ABSED
```

#### 2.5.8 字符串转小写函数：lower, lcase  

- **语法**: lower(string A) lcase(string A)
- **返回值**: string
- **说明**：返回字符串A的小写格式

```sql
hive> select lower('abSEd') from tableName;
absed
hive> select lcase('abSEd') from tableName;
absed
```

#### 2.5.9 去空格函数：trim 

- **语法**: trim(string A)
- **返回值**: string
- **说明**：去除字符串两边的空格

```sql
hive> select trim(' ab c ') from tableName;
ab c
```

#### 2.5.10 url解析函数  parse_url

- **语法**:
  parse_url(string urlString, string partToExtract [, string keyToExtract])
- **返回值**: string
- **说明**：返回URL中指定的部分。partToExtract的有效值为：HOST, PATH,
  QUERY, REF, PROTOCOL, AUTHORITY, FILE, and USERINFO.

```sql
hive> select parse_url
('https://www.tableName.com/path1/p.php?k1=v1&k2=v2#Ref1', 'HOST') 
from tableName;
www.tableName.com 
hive> select parse_url
('https://www.tableName.com/path1/p.php?k1=v1&k2=v2#Ref1', 'QUERY', 'k1')
 from tableName;
v1
```

#### 2.5.11 json解析  get_json_object 

- **语法**: get_json_object(string json_string, string path)
- **返回值**: string
- **说明**：解析json的字符串json_string,返回path指定的内容。如果输入的json字符串无效，那么返回NULL。

```sql
hive> select  get_json_object('{"store":{"fruit":\[{"weight":8,"type":"apple"},{"weight":9,"type":"pear"}], "bicycle":{"price":19.95,"color":"red"} },"email":"amy@only_for_json_udf_test.net","owner":"amy"}','$.owner') from tableName;
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151141177.png)

#### 2.5.12 重复字符串函数：repeat 

- **语法**: repeat(string str, int n)
- **返回值**: string
- **说明**：返回重复n次后的str字符串

```sql
hive> select repeat('abc', 5) from tableName;
abcabcabcabcabc
```

#### 2.5.13 分割字符串函数: split   

- **语法**: split(string str, string pat)
- **返回值**: array
- **说明**: 按照pat字符串分割str，会返回分割后的字符串数组

```sql
hive> select split('abtcdtef','t') from tableName;
["ab","cd","ef"]
```

### 2.6 集合统计函数

#### 2.6.1 个数统计函数: count  

- **语法**: count(*), count(expr), count(DISTINCT expr[, expr_.])
- **返回值**：Int
- **说明**: count(*)统计检索出的行的个数，包括NULL值的行；count(expr)返回指定字段的非空值的个数；count(DISTINCT
  expr[, expr_.])返回指定字段的不同的非空值的个数

```sql
hive> select count(*) from tableName;
20
hive> select count(distinct t) from tableName;
10
```

#### 2.6.2 总和统计函数: sum 

- **语法**: sum(col), sum(DISTINCT col)
- **返回值**: double
- **说明**: sum(col)统计结果集中col的相加的结果；sum(DISTINCT col)统计结果中col不同值相加的结果

```sql
hive> select sum(t) from tableName;
100
hive> select sum(distinct t) from tableName;
70
```

#### 2.6.3 平均值统计函数: avg   

- **语法**: avg(col), avg(DISTINCT col)
- **返回值**: double
- **说明**: avg(col)统计结果集中col的平均值；avg(DISTINCT col)统计结果中col不同值相加的平均值

```sql
hive> select avg(t) from tableName;
50
hive> select avg (distinct t) from tableName;
30
```

#### 2.6.4 最小值统计函数: min 

- **语法**: min(col)
- **返回值**: double
- **说明**: 统计结果集中col字段的最小值

```sql
hive> select min(t) from tableName;
20
```

#### 2.6.5 最大值统计函数: max  

- **语法**: max(col)
- **返回值**: double
- **说明**: 统计结果集中col字段的最大值

```sql
hive> select max(t) from tableName;
120
```

### 2.7 复合类型构建函数

#### 2.7.1 Map类型构建: map  

- **语法**: map (key1, value1, key2, value2, …)
- **说明**：根据输入的key和value对构建map类型

```sql
-- 建表
create table score_map(name string, score map<string, int>)
row format delimited fields terminated by '\t' 
collection items terminated by ',' 
map keys terminated by ':';

-- 创建数据内容如下并加载数据
cd /opt/install/hivedatas/
vim score_map.txt

zhangsan	数学:80,语文:89,英语:95
lisi	语文:60,数学:80,英语:99

-- 加载数据到hive表当中去
load data local inpath '/opt/install/hivedatas/score_map.txt' overwrite into table score_map;

-- map结构数据访问：
-- 获取所有的value：
select name,map_values(score) from score_map;

-- 获取所有的key：
select name,map_keys(score) from score_map;

-- 按照key来进行获取value值
select name,score["数学"]  from score_map;

-- 查看map元素个数
select name,size(score) from score_map;

-- 构建一个map
select map(1, 'zs', 2, 'lisi');
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151142672.png)

#### 2.7.2 Struct类型构建: struct

- **语法**: struct(val1, val2, val3, …)
- **说明**：根据输入的参数构建结构体struct类型，似于C语言中的结构体，内部数据通过X.X来获取，假设我
- 数据格式是这样的，电影ABC，有1254人评价过，打分为7.4分

```sql
-- 创建struct表
hive> create table movie_score(name string, info struct<number:int,score:float>)
row format delimited fields terminated by "\t"  
collection items terminated by ":"; 

-- 加载数据
cd /opt/install/hivedatas/
vim struct.txt

-- 电影ABC，有1254人评价过，打分为7.4分
ABC	1254:7.4  
DEF	256:4.9  
XYZ	456:5.4

-- 加载数据
load data local inpath '/opt/install/hivedatas/struct.txt' overwrite into table movie_score;

-- hive当中查询数据
hive> select * from movie_score;  
hive> select info.number, info.score from movie_score;  
OK  
1254    7.4  
256     4.9  
456     5.4  

-- 构建一个struct
select struct(1, 'anzhulababy', 'moon', 1.68);
```

![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151142649.png)

#### 2.7.3 Array类型构建: array

- **语法**: array(val1, val2, …)
- **说明**：根据输入的参数构建数组array类型

```sql
hive> create table person(name string, work_locations array<string>)
ROW FORMAT DELIMITED FIELDS TERMINATED BY '\t'
COLLECTION ITEMS TERMINATED BY ',';

-- 加载数据到person表当中去
cd /opt/install/hivedatas/
vim person.txt

-- 数据内容格式如下
biansutao	beijing,shanghai,tianjin,hangzhou
linan	changchun,chengdu,wuhan

-- 加载数据
hive > load  data local inpath '/opt/install/hivedatas/person.txt' overwrite into table person;

-- 查询所有数据数据
hive > select * from person;

-- 按照下表索引进行查询
hive > select work_locations[0] from person;

-- 查询所有集合数据
hive  > select work_locations from person; 

-- 查询元素个数
hive >  select size(work_locations) from person;   

-- 构建array
select array(1, 2, 1);
select array(1, 'a', 1.0);
select array(1, 2, 1.0);
```

### 2.8 复杂类型长度统计函数

#### 2.8.1 Map类型长度函数: size(Map<k .V>)

- **语法**: size(Map<k .V>)
- **返回值**: int
- **说明**: 返回map类型的长度

```sql
hive> select size(map(1, 'zs', 2, 'anzhulababy')) from tableName;
2
```

#### 2.8.2 array类型长度函数: size(Array<T>)

- **语法**: size(Array<T>)
- **返回值**: int
- **说明**: 返回array类型的长度

```sql
hive> select size(t) from arr_table2;
4
```

#### 2.8.3 类型转换函数  

- **类型转换函数**: cast
- **语法**: cast(expr as <type>)
- **返回值**: Expected "=" to follow "type"
- **说明**: 返回转换后的数据类型

```sql
hive> select cast('1' as bigint) from tableName;
1
```

### 2.9 行转列

#### 2.9.1 相关函数说明

- CONCAT(string A/col, string B/col…)：返回输入字符串连接后的结果，支持任意个输入字符串;

- CONCAT_WS(separator, str1, str2,...)：它是一个特殊形式的 CONCAT()。
  - 第一个参数剩余参数间的分隔符。分隔符可以是与剩余参数一样的字符串。如果分隔符是 NULL，返回值也将为 NULL。
  - 这个函数会跳过分隔符参数后的任何 NULL 和空字符串。分隔符将被加到被连接的字符串之间;

- COLLECT_SET(col)：函数只接受基本数据类型，它的主要作用是将某字段的值进行去重汇总，产生**array**类型字段。

#### 2.9.2 数据准备

- 数据准备

| name    | constellation | blood_type |
| ------- | ------------- | ---------- |
| 孙悟空     | 白羊座           | A          |
| 老王      | 射手座           | A          |
| 宋宋      | 白羊座           | B          |
| 猪八戒     | 白羊座           | A          |
| 按住啦baby | 射手座           | A          |

#### 2.9.3 需求

- 把星座和血型一样的人归类到一起。结果如下：

```
射手座,A            老王|按住啦baby
白羊座,A            孙悟空|猪八戒
白羊座,B            宋宋
```

#### 2.9.4 创建表数据文件

- node03服务器执行以下命令创建文件，注意数据使用\t进行分割

```shell
cd /opt/install/hivedatas
vim constellation.txt
```

```
孙悟空	白羊座	A
老王	射手座	A
宋宋	白羊座	B       
猪八戒	白羊座	A
凤姐	射手座	A
```

#### 2.9.5 创建hive表并导入数据

- 创建hive表并加载数据

```sql
hive (hive_explode)> create table person_info(name string, constellation string,  blood_type string) row format delimited fields terminated by "\t";
```

- 加载数据

```sql
hive (hive_explode)> load data local inpath '/opt/install/hivedatas/constellation.txt' into table person_info;
```

#### 2.9.6 按需求查询数据

```sql
hive (hive_explode)> select t1.base, concat_ws('|', collect_set(t1.name)) name 
from    
(select name, concat(constellation, "," , blood_type) base from person_info) t1 
group by t1.base;
```

### 2.10 列转行

#### 2.10.1 函数说明

- EXPLODE(col)：将hive一列中复杂的array或者map结构拆分成多行。

- LATERAL VIEW
  - 用法：LATERAL VIEW udtf(expression) tableAlias AS columnAlias
  - 解释：用于和split, explode等UDTF一起使用，它能够将一列数据拆成多行数据，在此基础上可以对拆分后的数据进行聚合。

#### 2.10.2 数据准备

- 数据内容如下，字段之间都是使用\t进行分割

```shell
cd /opt/install/hivedatas

vim movie.txt

《疑犯追踪》	悬疑,动作,科幻,剧情
《Lie to me》	悬疑,警匪,动作,心理,剧情
《战狼2》	战争,动作,灾难
```

#### 2.10.3 需求

- 将电影分类中的数组数据展开。结果如下：

```
《疑犯追踪》	悬疑
《疑犯追踪》	动作
《疑犯追踪》	科幻
《疑犯追踪》	剧情
《Lie to me》	悬疑
《Lie to me》	警匪
《Lie to me》	动作
《Lie to me》	心理
《Lie to me》	剧情
《战狼2》	战争
《战狼2》	动作
《战狼2》	灾难
```

#### 2.10.4 创建hive表并导入数据

- 创建hive表

```sql
hive (hive_explode)> create table movie_info(movie string, category array<string>) 
row format delimited fields terminated by "\t" 
collection items terminated by ",";
```

- 加载数据

```sql
load data local inpath "/opt/install/hivedatas/movie.txt" into table movie_info;
```

#### 2.10.5 按需求查询数据

```sql
hive (hive_explode)> select movie, category_name from movie_info 
lateral view explode(category) table_tmp as category_name;
```

### 2.11 lateral view、explode练习

#### 2.11.1 explode函数将Map和Array字段数据进行拆分

- explode还可以用于将hive一列中复杂的array或者map结构拆分成多行

```sql
-- 需求：现在有数据格式如下
zhangsan	child1,child2,child3,child4	k1:v1,k2:v2
lisi	child5,child6,child7,child8	 k3:v3,k4:v4

-- 字段之间使用\t分割，需求将所有的child进行拆开成为一列
+----------+--+
| mychild  |
+----------+--+
| child1   |
| child2   |
| child3   |
| child4   |
| child5   |
| child6   |
| child7   |
| child8   |
+----------+--+

-- 将map的key和value也进行拆开，成为如下结果
+-----------+-------------+--+
| mymapkey  | mymapvalue  |
+-----------+-------------+--+
| k1        | v1          |
| k2        | v2          |
| k3        | v3          |
| k4        | v4          |
+-----------+-------------+--+
```

##### 第一步：创建hive数据库

- 创建hive数据库

```
hive (default)> create database hive_explode;
hive (default)> use hive_explode;
```

##### 第二步：创建hive表

```sql
hive (hive_explode)> create table hive_explode.t3(name string, children array<string>, address Map<string, string>) row format delimited fields terminated by '\t' collection items terminated by ','  map keys terminated by ':' stored as textFile;
```

##### 第三步：加载数据

- node03执行以下命令创建表数据文件

```sql
cd  /opt/install/hivedatas/

vim maparray

-- 数据内容格式如下
zhangsan	child1,child2,child3,child4	k1:v1,k2:v2
lisi	child5,child6,child7,child8	k3:v3,k4:v4
```

- hive表当中加载数据

```sql
hive (hive_explode)> load data local inpath '/opt/install/hivedatas/maparray' into table hive_explode.t3;
```

##### 第四步：使用explode将hive当中数据拆开

- 将array当中的数据拆分开

```sql
hive (hive_explode)> SELECT explode(children) AS myChild FROM hive_explode.t3;
```

- 将map当中的数据拆分开

```sql
hive (hive_explode)> SELECT explode(address) AS (myMapKey, myMapValue) FROM hive_explode.t3;
```

#### 2.11.2 使用explode拆分json字符串

- 需求：现在有一些数据格式如下：

```
a:shandong,b:beijing,c:hebei|1,2,3,4,5,6,7,8,9|[{"source":"7fresh","monthSales":4900,"userCount":1900,"score":"9.9"},{"source":"jd","monthSales":2090,"userCount":78981,"score":"9.8"},{"source":"jdmart","monthSales":6987,"userCount":1600,"score":"9.0"}]
```

- 其中字段与字段之间的分隔符是 | 

- 我们要解析得到所有的monthSales对应的值为以下这一列（行转列）

```
4900
2090
6987
```

##### 第一步：创建hive表

```sql
hive (hive_explode)> create table hive_explode.explode_lateral_view (area string, goods_id string, sale_info string) ROW FORMAT DELIMITED FIELDS TERMINATED BY '|' STORED AS textfile;
```

##### 第二步：准备数据并加载数据

- 准备数据如下

```shell
cd /opt/install/hivedatas

vim explode_json

a:shandong,b:beijing,c:hebei|1,2,3,4,5,6,7,8,9|[{"source":"7fresh","monthSales":4900,"userCount":1900,"score":"9.9"},{"source":"jd","monthSales":2090,"userCount":78981,"score":"9.8"},{"source":"jdmart","monthSales":6987,"userCount":1600,"score":"9.0"}]
```

- 加载数据到hive表当中去

```sql
hive (hive_explode)> load data local inpath '/opt/install/hivedatas/explode_json' overwrite into table hive_explode.explode_lateral_view;
```

##### 第三步：使用explode拆分Array

```sql
hive (hive_explode)> select explode(split(goods_id, ',')) as goods_id from hive_explode.explode_lateral_view;
```

##### 第四步：使用explode拆解Map

```sql
hive (hive_explode)> select explode(split(area, ',')) as area from hive_explode.explode_lateral_view;
```

##### 第五步：拆解json字段

```sql
hive (hive_explode)> select explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{')) as sale_info from hive_explode.explode_lateral_view;
```

 ![](https://gcore.jsdelivr.net/gh/Raray-chuan/xichuan_blog_pic@main/img/202211151143251.png)

- 然后我们想用get_json_object来获取key为monthSales的数据：

```sql
hive (hive_explode)> select get_json_object(explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{')),'$.monthSales') as sale_info from hive_explode.explode_lateral_view;

-- 然后出现异常FAILED: SemanticException [Error 10081]: UDTF's are not supported outside the SELECT clause, nor nested in expressions
-- UDTF explode不能写在别的函数内

-- 如果你这么写，想查两个字段，select explode(split(area,',')) as area,good_id from explode_lateral_view;
-- 会报错FAILED: SemanticException 1:40 Only a single expression in the SELECT clause is supported with UDTF's. Error encountered near token 'good_id'
-- 使用UDTF的时候，只支持一个字段，这时候就需要LATERAL VIEW出场了
```

#### 2.11.3 配合LATERAL  VIEW使用

- lateral view用于和split、explode等UDTF一起使用的，能将一行数据拆分成多行数据
- 在此基础上可以对拆分的数据进行聚合
- lateral view首先为原始表的每行调用UDTF，UDTF会把一行拆分成一行或者多行，lateral view在把结果组合，产生一个支持别名表的**虚拟表**。

- 配合lateral view查询多个字段

```sql
hive (hive_explode)> select goods_id2, sale_info from explode_lateral_view 
LATERAL VIEW explode(split(goods_id, ','))goods as goods_id2;
```

- 其中LATERAL VIEW explode(split(goods_id,','))goods相当于一个虚拟表，与原表explode_lateral_view**笛卡尔积关联**。

- 也可以多重使用，如下，也是三个表笛卡尔积的结果

```sql
hive (hive_explode)> select goods_id2, sale_info, area2 from explode_lateral_view 
LATERAL VIEW explode(split(goods_id, ','))goods as goods_id2 
LATERAL VIEW explode(split(area,','))area as area2;
```

- 最终，我们可以通过下面的句子，把这个json格式的一行数据，完全转换成二维表的方式展现

```sql
hive (hive_explode)> select 
get_json_object(concat('{',sale_info_1,'}'),'$.source') as source, get_json_object(concat('{',sale_info_1,'}'),'$.monthSales') as monthSales, get_json_object(concat('{',sale_info_1,'}'),'$.userCount') as userCount,  get_json_object(concat('{',sale_info_1,'}'),'$.score') as score 
from explode_lateral_view   
LATERAL VIEW explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{'))sale_info as sale_info_1;
```

- 总结：
  - Lateral View通常和UDTF一起出现，为了解决UDTF不允许在select字段的问题。 
  - Multiple Lateral View可以实现类似笛卡尔乘积。 
  - Outer关键字可以把不输出的UDTF的空结果，输出成NULL，防止丢失数据。

### 2.12 reflect函数

- reflect函数可以支持在sql中调用java中的自带函数，秒杀一切udf函数。

#### 2.12.1 使用java.lang.Math当中的Max求两列中最大值

- 创建hive表

```sql
hive (hive_explode)> create table test_udf(col1 int,col2 int) 
row format delimited fields terminated by ',';
```

- 准备数据并加载数据

```shell
cd /opt/install/hivedatas
vim test_udf

1,2
4,3
6,4
7,5
5,6
```

- 加载数据

```sql
hive (hive_explode)> load data local inpath '/opt/install/hivedatas/test_udf' overwrite into table test_udf;
```

- 使用java.lang.Math当中的Max求两列当中的最大值

```sql
hive (hive_explode)> select reflect("java.lang.Math","max", col1, col2) from test_udf;
```

#### 2.12.2 不同记录执行不同的java内置函数

- 创建hive表

```sql
hive (hive_explode)> create table test_udf2(class_name string, method_name string, col1 int, col2 int) row format delimited fields terminated by ',';
```

- 准备数据

```shell
cd /export/servers/hivedatas
vim test_udf2

java.lang.Math,min,1,2
java.lang.Math,max,2,3
```

- 加载数据

```sql
hive (hive_explode)> load data local inpath '/opt/install/hivedatas/test_udf2' overwrite into table test_udf2;
```

- 执行查询

```sql
hive (hive_explode)> select reflect(class_name, method_name, col1, col2) from test_udf2;
```

#### 2.12.3 判断是否为数字

- 使用apache commons中的函数，commons下的jar已经包含在hadoop的classpath中，所以可以直接使用。

- 使用方式如下：

```sql
hive (hive_explode)> select reflect("org.apache.commons.lang.math.NumberUtils", "isNumber", "123");
```

### 2.13 分析函数—分组求topN

#### 2.13.1 分析函数的作用

- 对于一些比较复杂的数据求取过程，我们可能就要用到分析函数
- 分析函数主要用于`分组求topN或者求取百分比，或者进行数据的切片`等等，我们都可以使用分析函数来解决

#### 2.13.2 常用的分析函数

1、ROW_NUMBER()：

- 从1开始，按照顺序，给分组内的记录加序列；
  - 比如，按照pv降序排列，生成分组内每天的pv名次,ROW_NUMBER()的应用场景非常多
  - 再比如，获取分组内排序第一的记录;
  - 获取一个session中的第一条refer等。 

2、RANK() ：

- 生成数据项在分组中的排名，排名相等会在名次中留下空位 

3、DENSE_RANK() ：

- 生成数据项在分组中的排名，排名相等会在名次中不会留下空位 

4、CUME_DIST ：

- 小于等于当前值的行数/分组内总行数。比如，统计小于等于当前薪水的人数，所占总人数的比例 

5、PERCENT_RANK ：

- 分组内当前行的RANK值/分组内总行数

6、NTILE(n) ：

- 用于将分组数据按照顺序切分成n片，返回当前切片值
- 如果切片不均匀，默认增加第一个切片的分布。
- NTILE不支持ROWS BETWEEN，比如 NTILE(2) OVER(PARTITION BY cookieid ORDER BY createtime ROWS BETWEEN 3 PRECEDING AND CURRENT ROW)

#### 2.13.3 需求描述

- 现有数据内容格式如下，分别对应三个字段，cookieid，createtime ，pv
- 求取每个cookie访问pv前三名的数据记录，其实就是分组求topN，求取每组当中的前三个值

```
cookie1,2015-04-10,1
cookie1,2015-04-11,5
cookie1,2015-04-12,7
cookie1,2015-04-13,3
cookie1,2015-04-14,2
cookie1,2015-04-15,4
cookie1,2015-04-16,4
cookie2,2015-04-10,2
cookie2,2015-04-11,3
cookie2,2015-04-12,5
cookie2,2015-04-13,6
cookie2,2015-04-14,3
cookie2,2015-04-15,9
cookie2,2015-04-16,7
```

##### 第一步：创建数据库表

- 在hive当中创建数据库表

```sql
CREATE EXTERNAL TABLE cookie_pv (
cookieid string,
createtime string, 
pv INT
) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;
```

##### 第二步：准备数据并加载

- node03执行以下命令，创建数据，并加载到hive表当中去

```shell
cd /opt/install/hivedatas
vim cookiepv.txt

cookie1,2015-04-10,1
cookie1,2015-04-11,5
cookie1,2015-04-12,7
cookie1,2015-04-13,3
cookie1,2015-04-14,2
cookie1,2015-04-15,4
cookie1,2015-04-16,4
cookie2,2015-04-10,2
cookie2,2015-04-11,3
cookie2,2015-04-12,5
cookie2,2015-04-13,6
cookie2,2015-04-14,3
cookie2,2015-04-15,9
cookie2,2015-04-16,7
```

- 加载数据到hive表当中去

```sql
load data local inpath '/opt/install/hivedatas/cookiepv.txt' overwrite into table  cookie_pv;
```

##### 第三步：使用分析函数来求取每个cookie访问PV的前三条记录

```sql
select * from (
SELECT 
cookieid,
createtime,
pv,
RANK() OVER(PARTITION BY cookieid ORDER BY pv desc) AS rn1,
DENSE_RANK() OVER(PARTITION BY cookieid ORDER BY pv desc) AS rn2,
ROW_NUMBER() OVER(PARTITION BY cookieid ORDER BY pv DESC) AS rn3 
FROM cookie_pv 
) temp where temp.rn1 <= 3;
```

### 2.14 hive自定义函数

#### 2.14.1 自定义函数的基本介绍

- Hive 自带了一些函数，比如：max/min等，但是数量有限，自己可以通过自定义UDF来方便的扩展。

- 当Hive提供的内置函数无法满足你的业务处理需要时，此时就可以考虑使用用户自定义函数（UDF：user-defined function）

- 根据用户自定义函数类别分为以下三种：
  - UDF（User-Defined-Function） 一进一出
  - UDAF（User-Defined Aggregation Function）  聚集函数，多进一出，类似于：count/max/min

  - UDTF（User-Defined Table-Generating Functions） 一进多出，如lateral view explode()

​                如lateral view explode()

- 官方文档地址

  https://cwiki.apache.org/confluence/display/Hive/HivePlugins

- 编程步骤：

​        （1）继承org.apache.hadoop.hive.ql.UDF

​        （2）需要实现evaluate函数；evaluate函数支持重载；

- 注意事项

​        （1）UDF必须要有返回类型，可以返回null，但是返回类型不能为void；

​        （2）UDF中常用Text/LongWritable等类型，不推荐使用java类型；

####  2.14.2 自定义函数开发

##### 第一步：创建maven java 工程，并导入jar包

```xml
	<repositories>
        <repository>
            <id>cloudera</id>
            <url>https://repository.cloudera.com/artifactory/cloudera-repos/</url>
        </repository>
    </repositories>
    
    <dependencies>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-client</artifactId>
            <version>2.6.0-mr1-cdh5.14.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-common</artifactId>
            <version>2.6.0-cdh5.14.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-hdfs</artifactId>
            <version>2.6.0-cdh5.14.2</version>
        </dependency>

        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-mapreduce-client-core</artifactId>
            <version>2.6.0-cdh5.14.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-exec</artifactId>
            <version>1.1.0-cdh5.14.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-jdbc</artifactId>
            <version>1.1.0-cdh5.14.2</version>
        </dependency>
        <dependency>
            <groupId>org.apache.hive</groupId>
            <artifactId>hive-cli</artifactId>
            <version>1.1.0-cdh5.14.2</version>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                    <!--    <verbal>true</verbal>-->
                </configuration>
            </plugin>
        </plugins>
    </build>
```

##### 第二步：开发java类继承UDF，并重载evaluate 方法

```java
package com.udf.MyUDF;

public class MyUDF extends UDF {
     public Text evaluate(final Text s) {
         if (null == s) {
             return null;
         }
         //返回大写字母         
         return new Text(s.toString().toUpperCase());
     }
 }
```

##### 第三步：项目打包

- 使用maven的package进行打包
- 将我们打包好的jar包上传到node03服务器的/opt/install/hive-1.1.0-cdh5.14.2/lib 这个路径下

##### 第四步：添加我们的jar包

- 重命名我们的jar包名称

```shell
cd /opt/install/hive-1.1.0-cdh5.14.2/lib
mv original-day_hive_udf-1.0-SNAPSHOT.jar udf.jar
```

- hive的客户端添加我们的jar包

```sql
0: jdbc:hive2://node03:10000> add jar /opt/install/hive-1.1.0-cdh5.14.2/lib/udf.jar;
```

##### 第五步：设置函数与我们的自定义函数关联

```sql
0: jdbc:hive2://node03:10000> create temporary function touppercase as 'com.opt.udf.MyUDF';
```

##### 第六步：使用自定义函数

```sql
0: jdbc:hive2://node03:10000>select tolowercase('abc');
```

- hive当中如何创建永久函数

- 在hive当中添加临时函数，需要我们每次进入hive客户端的时候都需要添加以下，退出hive客户端临时函数就会失效，那么我们也可以创建永久函数来让其不会失效
- 创建永久函数

```sql
-- 1、指定数据库，将我们的函数创建到指定的数据库下面
0: jdbc:hive2://node03:10000>use myhive;

-- 2、使用add jar添加我们的jar包到hive当中来
0: jdbc:hive2://node03:10000>add jar /opt/install/hive-1.1.0-cdh5.14.2/lib/udf.jar;

-- 3、查看我们添加的所有的jar包
0: jdbc:hive2://node03:10000>list jars;

-- 4、创建永久函数，与我们的函数进行关联
0: jdbc:hive2://node03:10000>create function myuppercase as 'com.opt.udf.MyUDF';

-- 5、查看我们的永久函数
0: jdbc:hive2://node03:10000>show functions like 'my*';

-- 6、使用永久函数
0: jdbc:hive2://node03:10000>select myhive.myuppercase('helloworld');

-- 7、删除永久函数
0: jdbc:hive2://node03:10000>drop function myhive.myuppercase;

-- 8、查看函数
 show functions like 'my*';
```



 

