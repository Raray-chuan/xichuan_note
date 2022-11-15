# Hive 主流文件存储格式对比

## 1. hive的SerDe

### 1.1 hive的SerDe是什么

​	Serde是 `Serializer/Deserializer`的简写。hive使用Serde进行行对象的序列与反序列化。最后实现把文件内容映射到 hive 表中的字段数据类型。

​	为了更好的阐述使用 SerDe 的场景，我们需要了解一下 Hive 是如何读数据的(类似于 HDFS 中数据的读写操作)：

```
HDFS files –> InputFileFormat –> <key, value> –> Deserializer –> Row object

Row object –> Serializer –> <key, value> –> OutputFileFormat –> HDFS files
```



### 1.2 hive的SerDe 类型

- Hive 中内置`org.apache.hadoop.hive.serde2`库，内部封装了很多不同的SerDe类型。
- hive创建表时， 通过自定义的SerDe或使用Hive内置的SerDe类型指定数据的序列化和反序列化方式。

```sql
CREATE [EXTERNAL] TABLE [IF NOT EXISTS] table_name 
[(col_name data_type [COMMENT col_comment], ...)] [COMMENT table_comment] [PARTITIONED BY (col_name data_type [COMMENT col_comment], ...)] 
[CLUSTERED BY (col_name, col_name, ...) 
[SORTED BY (col_name [ASC|DESC], ...)] INTO num_buckets BUCKETS] 
[ROW FORMAT row_format] 
[STORED AS file_format] 
[LOCATION hdfs_path]
```

- 如上创建表语句， 使用`row format 参数说明SerDe的类型`.
- 你可以创建表时使用用户**自定义的Serde或者native Serde**， **如果 ROW FORMAT没有指定或者指定了 ROW FORMAT DELIMITED就会使用native Serde**。
- [Hive SerDes](https://cwiki.apache.org/confluence/display/Hive/SerDe):
  - Avro (Hive 0.9.1 and later)
  - ORC (Hive 0.11 and later)
  - RegEx
  - Thrift
  - Parquet (Hive 0.13 and later)
  - CSV (Hive 0.14 and later)
  - MultiDelimitSerDe


## 2. 存储文件的压缩比测试

### 2.1 测试数据 

~~~
https://github.com/Raray-chuan/xichuan_blog_pic/blob/main/file/hive/hive-log.txt

log.txt 大小为18.1 M
~~~

### 2.2 TextFile

* 创建表，存储数据格式为**TextFile**

~~~sql
create table log_text (
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
row format delimited fields terminated by '\t'
stored as textfile ;
~~~

* 向表中加载数据

~~~sql
load data local inpath '/home/hadoop/log.txt' into table log_text ;
~~~

* 查看表的数据量大小

~~~shell
dfs -du -h /user/hive/warehouse/log_text;

+------------------------------------------------+--+
|                   DFS Output                   |
+------------------------------------------------+--+
| 18.1 M  /user/hive/warehouse/log_text/log.txt  |
+------------------------------------------------+--+
~~~



### 2.3 Parquet

* 创建表，存储数据格式为 **parquet**

~~~sql
create table log_parquet  (
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
row format delimited fields terminated by '\t'
stored as parquet;
~~~

* 向表中加载数据

~~~sql
insert into table log_parquet select * from log_text;
~~~

* 查看表的数据量大小

~~~shell
dfs -du -h /user/hive/warehouse/log_parquet;

+----------------------------------------------------+--+
|                     DFS Output                     |
+----------------------------------------------------+--+
| 13.1 M  /user/hive/warehouse/log_parquet/000000_0  |
+----------------------------------------------------+--+
~~~



### 2.4  ORC

- 创建表，存储数据格式为ORC

```sql
create table log_orc  (
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
row format delimited fields terminated by '\t'
stored as orc  ;
```

- 向表中加载数据

```sql
insert into table log_orc select * from log_text ;
```

- 查看表的数据量大小

```shell
dfs -du -h /user/hive/warehouse/log_orc;
+-----------------------------------------------+--+
|                  DFS Output                   |
+-----------------------------------------------+--+
| 2.8 M  /user/hive/warehouse/log_orc/000000_0  |
+-----------------------------------------------+--+
```

### 2.5 存储文件的压缩比总结

~~~
ORC >  Parquet >  textFile
~~~



## 3. 存储文件的查询速度测试

### 3.1  TextFile

~~~sql
select count(*) from log_text;
+---------+--+
|   _c0   |
+---------+--+
| 100000  |
+---------+--+
1 row selected (16.99 seconds)
~~~



### 3.2 Parquet

~~~sql
select count(*) from log_parquet;
+---------+--+
|   _c0   |
+---------+--+
| 100000  |
+---------+--+
1 row selected (17.994 seconds)
~~~



### 3.3 ORC

~~~sql
select count(*) from log_orc;
+---------+--+
|   _c0   |
+---------+--+
| 100000  |
+---------+--+
1 row selected (15.943 seconds)
~~~

### 3.4 存储文件的查询速度总结

~~~
ORC > TextFile > Parquet
~~~



## 4. 存储和压缩结合

* 使用压缩的优势是可以最小化所需要的磁盘存储空间，以及减少磁盘和网络io操作

* 官网地址
  * https://cwiki.apache.org/confluence/display/Hive/LanguageManual+ORC

* ORC支持三种压缩：ZLIB,SNAPPY,NONE。最后一种就是不压缩，`orc默认采用的是ZLIB压缩`。



### 4.1  创建一个非压缩的的ORC存储方式表

* 1、创建一个非压缩的的ORC表

~~~
create table log_orc_none (
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
row format delimited fields terminated by '\t'
stored as orc tblproperties("orc.compress"="NONE") ;
~~~

* 2、加载数据

~~~sql
insert into table log_orc_none select * from log_text ;
~~~

* 3、查看表的数据量大小

~~~shell
dfs -du -h /user/hive/warehouse/log_orc_none;
+----------------------------------------------------+--+
|                     DFS Output                     |
+----------------------------------------------------+--+
| 7.7 M  /user/hive/warehouse/log_orc_none/000000_0  |
+----------------------------------------------------+--+
~~~



### 4.2  创建一个snappy压缩的ORC存储方式表

* 1、创建一个snappy压缩的的ORC表

~~~sql
create table log_orc_snappy (
track_time string,
url string,
session_id string,
referer string,
ip string,
end_user_id string,
city_id string
)
row format delimited fields terminated by '\t'
stored as orc tblproperties("orc.compress"="SNAPPY") ;
~~~

* 2、加载数据

~~~sql
insert into table log_orc_snappy select * from log_text ;
~~~

* 3、查看表的数据量大小

~~~shell
dfs -du -h /user/hive/warehouse/log_orc_snappy;
+------------------------------------------------------+--+
|                      DFS Output                      |
+------------------------------------------------------+--+
| 3.8 M  /user/hive/warehouse/log_orc_snappy/000000_0  |
+------------------------------------------------------+--+
~~~



### 4.3  创建一个ZLIB压缩的ORC存储方式表

* 不指定压缩格式的就是默认的采用ZLIB压缩
  * 可以参考上面创建的 log_orc 表
* 查看表的数据量大小

~~~shell
dfs -du -h /user/hive/warehouse/log_orc;
+-----------------------------------------------+--+
|                  DFS Output                   |
+-----------------------------------------------+--+
| 2.8 M  /user/hive/warehouse/log_orc/000000_0  |
+-----------------------------------------------+--+
~~~

### 4.4 存储方式和压缩总结

* orc 默认的压缩方式ZLIB比Snappy压缩的还小。

* 在实际的项目开发当中，hive表的数据存储格式一般选择：orc或parquet。

* 由于snappy的压缩和解压缩 效率都比较高，`压缩方式一般选择snappy`






