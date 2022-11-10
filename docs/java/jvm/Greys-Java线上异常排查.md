## 1.软件安装与启动
Greys支持在线安装和本地安装两种安装方案，安装即可用，推荐使用在线安装。

### 1.1 在线安装（推荐）
请复制以下内容，并粘贴到命令行中。
```shell
curl -sLk http://ompc.oss.aliyuncs.com/greys/install.sh|bash
```
命令将会下载启动脚本文件greys.sh到当前目录，你可以放在任何地方或加入到$PATH中

### 1.2 本地安装
在某些情况下，目标服务器无法访问远程阿里云主机，此时你需要自行下载greys的安装文件。
**1）下载最新版本的GREYS**
`http://ompc.oss.aliyuncs.com/greys/release/greys-stable-bin.zip`

**2）解压zip文件后，执行以下命令**
```shell
cd greys
sh ./install-local.sh
```
即完成本地安装。
常见安装问题

**3）下载失败**
通常这样的原因你需要检查你的网络是否畅通，核对是否能正确访问这个网址
```shell
http://ompc.oss.aliyuncs.com/greys/greys.sh
downloading...
download failed!
```

**4）没有权限**
安装脚本首先会将greys文件从阿里云服务器上下载到当前执行脚本的目录，所以你必须要拥有当前目录的写权限。
`permission denied, target directory is not writable.`




### 1.3 启动关闭Greys
```shell
# ./greys.sh <PID>[@IP:PORT]
[root@mvxl52738 ]# greys.sh 57951
_
____  ____ _____ _   _  ___ _____ _____ ____  _____ _| |_ ___  ____  _   _
/ _  |/ ___) ___ | | | |/___|_____|____ |  _ \(____ (_   _) _ \|    \| | | |
( (_| | |   | ____| |_| |___ |     / ___ | | | / ___ | | || |_| | | | | |_| |
\___ |_|   |_____)\__  (___/      \_____|_| |_\_____|  \__)___/|_|_|_|\__  |
(_____|           (____/                                              (____/
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|v|e|r|s|i|o|n|:|1|.|7|.|6|.|6|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
ga?>help
# 查看帮助文档
ga?>shutdown
Greys Server is shut down.
```

### 1.4 远程连接Greys
远程连接，当然也是可以的。配置对应的ip和端口，就能远程协助了。（需要开启agent）
```shell
[root@mvxl52740 greys]# ./greys.sh @10.18.40.183:3658
_
____  ____ _____ _   _  ___ _____ _____ ____  _____ _| |_ ___  ____  _   _
/ _  |/ ___) ___ | | | |/___|_____|____ |  _ \(____ (_   _) _ \|    \| | | |
( (_| | |   | ____| |_| |___ |     / ___ | | | / ___ | | || |_| | | | | |_| |
\___ |_|   |_____)\__  (___/      \_____|_| |_\_____|  \__)___/|_|_|_|\__  |
(_____|           (____/                                              (____/
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|v|e|r|s|i|o|n|:|1|.|7|.|6|.|6|
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
ga?>
ga?>tt -t -n 3 *OrderReportFacadeImpl *
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:12) cost in 68 ms.
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|    INDEX | PROCESS-ID |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1003 |       1003 |  2018-09-15 17:40:24 |         14 |     true |    false |       0x68d36ec |          OrderReportFacadeImpl |            lambda$selectItem$3 |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1004 |       1004 |  2018-09-15 17:40:24 |         15 |     true |    false |       0x68d36ec |          OrderReportFacadeImpl |                     selectItem |
```



## 2.入门知识
### 2.1 会话与任务（不重要）
Greys是一个C/S架构的程序，所以当Client访问到Server时，Server会维护一个session（会话），以及session的心跳、超时机制。事务（Tx）机制则是建立在session的基础上，所有的命令交互都会创建一个事务，并且产生对应的队列进行输出缓冲。
事务伴随着命令的生命周期而存在，命令分两种：
**1）立即返回**
立即返回的命令定义是：敲下命令后Server端立即返回最终结果，后续无持续反馈信息，释放Client对输入的锁定，重新开放让用户输入信息，比如version、sc、sm等。

**2）等待中止**
等待中止的命令则是需要用户主动输入Ctrl+D完成的命令中止操作。命令执行后无法立即返回最终结果，而是不断的将中间产生的输出源源不断的输出到客户端中，这种命令比如stack、monitor等。
当session关闭时，所有挂在session的事务也会立即被关闭。


### 2.2 表达式
Greys相对于HouseMD、BTrace而言最灵活的地方就是在用表达式来灵活的支持不同的问题排查、分析场景。
表达式分两种：`条件表达式`与`观察表达式`
**1) 条件表达式**
   条件表达式用在使用表达式表达TRUE或FALSE的场景，从1.6.0.6版本开始，trace、stack、tt、watch命令都增加了条件表达式支持。
   条件表达式将会使用greys内置的表达式解析引擎，识别OGNL语法。
   特别指出的是，如果你书写了一个错误的条件表达式，greys为了兼容错误会解析为FALSE。
   以下是一些条件表达式使用的例子和预测结果
   `可以对一些值进行判断，如返回值是否为null`
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101727554.png)

**2) 观察表达式**
   观察表达式用在使用表达式表达输出内容的场景，尤其在watch和tt命令中，观察表达式至关重要。
   条件表达式将会使用greys内置的表达式解析引擎，识别OGNL语法，将表达式转换为待输出的对象。
以下是一些观察表达式使用的例子
  1.字符串拼接
   `clazz.name+"."+method.name`

  2.数字运算
  `clazz.name.length()+method.name.length()`



### 2.3 表达式核心变量(重要)

无论是匹配表达式也好、观察表达式也罢，他们核心判断变量都是围绕着一个greys中的通用通知对象Advice进行。
它的简略代码结构如下

```
public class Advice {

    private final ClassLoader loader;
    private final Class<?> clazz;
    private final GaMethod method;
    private final Object target;
    private final Object[] params;
    private final Object returnObj;
    private final Throwable throwExp;
    private final boolean isBefore;
    private final boolean isThrow;
    private final boolean isReturn;
    
    // getter/setter  
}
```

这里列一个表格来说明不同变量的含义
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101731624.png)
所有变量都可以在表达式中直接使用，如果在表达式中编写了不符合OGNL脚本语法或者引入了不在表格中的变量，
`对条件表达式、检索表达式而言，则一律当成false来处理
对观察表达式而言，则放弃当前方法调用的处理（不输出）`


### 2.4 JDK类支持（不重要）
JDK的类默认由BootstrapClassLoader负责加载，由于Greys自己也适用了大量的JDK类，所以我不建议使用Greys直接对JDK相关类进行增强、代理。
默认而言，Greys会拒绝执行关于JDK类的操作命令。你需显式用options命令打开。
```shell
ga?>options unsafe true
+--------+--------------+-------------+
| NAME   | BEFORE-VALUE | AFTER-VALUE |
+--------+--------------+-------------+
| unsafe | false        | true        |
+--------+--------------+-------------+
Affect(row-cnt:1) cost in 4 ms.
ga?>
```


### 2.5 模式匹配
一些命令需要对类、方法进行模式匹配过滤，从1.5.4.6及其之后的版本之后，Greys默认支持通配符匹配，目前仅支持*和?两个通配符，正则表达式需要显式指定-E参数激活。
**1）模式匹配举例**
原sc命令的正则表达式匹配
```shell
sc com\.apache\.commons\.lang\.StringUtils
```
在1.5.4.6及其之后的版本中将会默认使用通配符表达式
```shell
sc com.apache.commons.lang.StringUtils
sc *lang.StringUtils
```
若想继续使用正则表达式匹配，需要显式-E参数激活
```shell
sc -E com\.apache\.commons\.lang\.StringUtils
sc -E com\..*StringUtils
```

**2）支持模式匹配的命令**
所有需要模式匹配的命令都支持参数-E，他们分别是：sc、sm、stack、monitor、watch、tt、trace




## 3.Greys命令详解
### 3.1 命令清单
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101733201.png)



### 3.2 help命令
help命令会是你第一个在Greys中使用的命令，也会是今后使用最频繁的命令之一，当你在使用的过程中有任何不熟悉的疑问，请直接help吧～
#### 3.2.1 查看命令清单
进入Greys的欢迎界面后，所有命令都可以通过help获取帮助。此时你直接输入一个help，Greys则会返回所有命令的大概用途介绍。
```shell
ga?>help
+----------+----------------------------------------------------------------------------------+
|       sc | Search all the classes loaded by JVM                                             |
+----------+----------------------------------------------------------------------------------+
|       sm | Search the method of classes loaded by JVM                                       |
+----------+----------------------------------------------------------------------------------+
|  monitor | Monitor the execution of specified Class and its method                          |
+----------+----------------------------------------------------------------------------------+
|    watch | Display the details of specified class and method                                |
+----------+----------------------------------------------------------------------------------+
|       tt | Time Tunnel                                                                      |
+----------+----------------------------------------------------------------------------------+
|    stack | Display the stack trace of specified class and method                            |
+----------+----------------------------------------------------------------------------------+
|   ptrace | Display the detailed thread path stack of specified class and method             |
+----------+----------------------------------------------------------------------------------+
|    trace | Display the detailed thread stack of specified class and method                  |
|       js | Enhanced JavaScript                                                              |
+----------+----------------------------------------------------------------------------------+
|  session | Display current session information                                              |
+----------+----------------------------------------------------------------------------------+
|     quit | Quit Greys console                                                               |
+----------+----------------------------------------------------------------------------------+
|  version | Display Greys version                                                            |
+----------+----------------------------------------------------------------------------------+
|      jvm | Display the target JVM information                                               |
+----------+----------------------------------------------------------------------------------+
|    reset | Reset all the enhanced classes                                                   |
+----------+----------------------------------------------------------------------------------+
| shutdown | Shut down Greys server and exit the console                                      |
+----------+----------------------------------------------------------------------------------+
|     help | Display Greys Help                                                               |
+----------+----------------------------------------------------------------------------------+
Affect(row-cnt:1) cost in 9 ms.
ga?>
```
嗯，囋囋，我知道我的英文翻译很烂，就不用吐槽了。期望能有人能帮我重新打理英文的帮助界面和英文xwiki，小生感激不尽！

#### 3.2.2 查看命令详细帮助
help命令同时也支持对其他命令的一个解释说明，比如我们键入help watch，greys将会返回watch命令的所有参数解释、用法介绍等详细信息。
```shell
ga?>help watch
+---------+----------------------------------------------------------------------------------+
|   USAGE | -[bfesx:En:] class-pattern method-pattern express condition-express              |
|         | Display the details of specified class and method                                |
+---------+----------------------------------------------------------------------------------+
| OPTIONS |              [b] | Watch before invocation                                       |
|         | -----------------+-------------------------------------------------------------- |
|         |              [f] | Watch after invocation                                        |
|         | -----------------+-------------------------------------------------------------- |
|         |              [e] | Watch after throw exception                                   |
|         | -----------------+-------------------------------------------------------------- |
|         |              [s] | Watch after successful invocation                             |
|         | -----------------+-------------------------------------------------------------- |
|         |             [x:] | Expand level of object (0 by default)                         |
|         | -----------------+-------------------------------------------------------------- |
|         |              [E] | Enable regular expression to match (wildcard matching by def  |
|         |                  | ault)                                                         |
|         | -----------------+-------------------------------------------------------------- |
|         |             [n:] | Threshold of execution times                                  |
|         | -----------------+-------------------------------------------------------------- |
|         |    class-pattern | Path and classname of Pattern Matching                        |
|         | -----------------+-------------------------------------------------------------- |
|         |   method-pattern | Method of Pattern Matching                                    |
|         | -----------------+-------------------------------------------------------------- |
|         |          express | express, write by OGNL.                                       |
|         |                  |                                                               |
|         |                  | FOR EXAMPLE    params[0]                                      |
|         |                  |     params[0]+params[1]                                       |
|         |                  |     returnObj                                                 |
|         |                  |     throwExp                                                  |
|         |                  |     target                                                    |
|         |                  |     clazz                                                     |
|         |                  |     method                                                    |
|         |                  |                                                               |
|         |                  | THE STRUCTURE                                                 |
|         |                  |           target : the object                                 |
|         |                  |            clazz : the object's class                         |
|         |                  |           method : the constructor or method                  |
|         |                  |     params[0..n] : the parameters of method                   |
|         |                  |        returnObj : the returned object of method              |
|         |                  |         throwExp : the throw exception of method              |
|         |                  |         isReturn : the method ended by return                 |
|         |                  |          isThrow : the method ended by throwing exception     |
|         | -----------------+-------------------------------------------------------------- |
|         |  condition-expre | Conditional expression by OGNL                                |
|         |               ss |                                                               |
|         |                  | FOR EXAMPLE                                                   |
|         |                  |      TRUE : 1==1                                              |
|         |                  |      TRUE : true                                              |
|         |                  |     FALSE : false                                             |
|         |                  |      TRUE : params.length>=0                                  |
|         |                  |     FALSE : 1==2                                              |
|         |                  |                                                               |
|         |                  | THE STRUCTURE                                                 |
|         |                  |           target : the object                                 |
|         |                  |            clazz : the object's class                         |
|         |                  |           method : the constructor or method                  |
|         |                  |     params[0..n] : the parameters of method                   |
|         |                  |        returnObj : the returned object of method              |
|         |                  |         throwExp : the throw exception of method              |
|         |                  |         isReturn : the method ended by return                 |
|         |                  |          isThrow : the method ended by throwing exception     |
+---------+----------------------------------------------------------------------------------+
| EXAMPLE | watch -Eb org\.apache\.commons\.lang\.StringUtils isBlank params[0]              |
|         | watch -b org.apache.commons.lang.StringUtils isBlank params[0]                   |
|         | watch -f org.apache.commons.lang.StringUtils isBlank returnObj                   |
|         | watch -bf *StringUtils isBlank params[0]                                         |
|         | watch *StringUtils isBlank params[0]                                             |
|         | watch *StringUtils isBlank params[0] params[0].length==1                         |
+---------+----------------------------------------------------------------------------------+
Affect(row-cnt:1) cost in 15 ms.
ga?>
```

#### 3.2.3 帮助信息组成
帮助文档分成Usage、Options、Example三个区域，分别是用途说明、参数列表、实际例子
```shell
ga?>help session
+---------+----------------------------------------------------------------------------------+
|   USAGE | -[c:]                                                                            |
|         | Display current session information                                              |
+---------+----------------------------------------------------------------------------------+
| OPTIONS |             [c:] | Modify the character set of session                           |
+---------+----------------------------------------------------------------------------------+
| EXAMPLE | session                                                                          |
|         | session -c GBK                                                                   |
|         | session -c UTF-8                                                                 |
+---------+----------------------------------------------------------------------------------+
Affect(row-cnt:1) cost in 2 ms.
ga?>
```

#### 3.2.4 参数选项说明
[]中的参数为选填项，比如[d]，意思是该命令可接受一个名称为d的选填参数，且不用参数值。
[:]中的参数则为选填，但有值的参数，比如[c:]
class-pattern／method-pattern，这两个参数为隐性参数，即在输入的时候不需要特意声明参数。class-pattern为类路径.类名称的表达式匹配，method-pattern则为方法名的表达式匹配。




### 3.3 sc命令
“Search-Class”的简写，这个命令能搜索出所有已经加载到JVM中的Class信息。

#### 3.3.1 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101735143.png)

#### 3.3.2 使用参考
```shell
ga?>sc -df *alibaba.Address
+------------------+-----------------------------------------------+
|       class-info | com.alibaba.Address                           |
+------------------+-----------------------------------------------+
|      code-source | /Users/vlinux/temp/agent-test/target/         |
+------------------+-----------------------------------------------+
|             name | com.alibaba.Address                           |
+------------------+-----------------------------------------------+
|      isInterface | false                                         |
+------------------+-----------------------------------------------+
|     isAnnotation | false                                         |
+------------------+-----------------------------------------------+
|           isEnum | false                                         |
+------------------+-----------------------------------------------+
| isAnonymousClass | false                                         |
+------------------+-----------------------------------------------+
|          isArray | false                                         |
+------------------+-----------------------------------------------+
|     isLocalClass | false                                         |
+------------------+-----------------------------------------------+
|    isMemberClass | false                                         |
+------------------+-----------------------------------------------+
|      isPrimitive | false                                         |
+------------------+-----------------------------------------------+
|      isSynthetic | false                                         |
+------------------+-----------------------------------------------+
|      simple-name | Address                                       |
+------------------+-----------------------------------------------+
|         modifier | public                                        |
+------------------+-----------------------------------------------+
|       annotation |                                               |
+------------------+-----------------------------------------------+
|       interfaces |                                               |
+------------------+-----------------------------------------------+
|      super-class | com.alibaba.CountObject                       |
|                  |   `-java.lang.Object                          |
+------------------+-----------------------------------------------+
|     class-loader | sun.misc.Launcher$AppClassLoader@2a139a55     |
|                  |   `-sun.misc.Launcher$ExtClassLoader@5fb20bfd |
+------------------+-----------------------------------------------+
|           fields | modifier : private                            |
|                  |     type : java.lang.String                   |
|                  |     name : username                           |
|                  |                                               |
|                  | modifier : private                            |
|                  |     type : int                                |
|                  |     name : addressId                          |
|                  |                                               |
|                  | modifier : private                            |
|                  |     type : java.lang.String                   |
|                  |     name : addressName                        |
|                  |                                               |
+------------------+-----------------------------------------------+
Affect(row-cnt:1) cost in 9 ms.
ga?>
```





### 3.4 sm命令
“Search-Method”的简写，这个命令能搜索出所有已经加载了Class信息的方法信息。

#### 3.4.1 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101736468.png)

#### 3.4.2 使用参考
```shell
ga?>sm -d *alibaba.Address *
+-----------------------------+-------------------------------------------------+
| DECLARED-CLASS              | VISIBLE-METHOD                                  |
+-----------------------------+-------------------------------------------------+
| com.alibaba.Address         | declaring-class : class com.alibaba.Address     |
|                             |     method-name : getAddressId                  |
|                             |        modifier : public                        |
|                             |      annotation :                               |
|                             |      parameters :                               |
|                             |          return : int                           |
|                             |      exceptions :                               |
+-----------------------------+-------------------------------------------------+
| com.alibaba.Address         | declaring-class : class com.alibaba.Address     |
|                             |     method-name : getAddressName                |
|                             |        modifier : public                        |
|                             |      annotation :                               |
|                             |      parameters :                               |
|                             |          return : java.lang.String              |
|                             |      exceptions :                               |
+-----------------------------+-------------------------------------------------+
| com.alibaba.Address         | declaring-class : class com.alibaba.CountObject |
|   `-com.alibaba.CountObject |     method-name : finalize                      |
|                             |        modifier : protected                     |
|                             |      annotation :                               |
|                             |      parameters :                               |
|                             |          return : void                          |
|                             |      exceptions : java.lang.Throwable           |
+-----------------------------+-------------------------------------------------+
| com.alibaba.Address         | declaring-class : class com.alibaba.CountObject |
|   `-com.alibaba.CountObject |     method-name : count                         |
|                             |        modifier : public                        |
|                             |      annotation :                               |
|                             |      parameters :                               |
|                             |          return : int                           |
|                             |      exceptions :                               |
+-----------------------------+-------------------------------------------------+
Affect(row-cnt:4) cost in 9 ms.
ga?>
```





### 3.5 monitor命令(监控方法成功失败的次数)
对匹配class-pattern／method-pattern的类.方法的调用进行监控。

monitor命令是介绍到的第一个非实时返回命令，实时返回命令是输入之后立即返回，而非实时返回的命令，则是不断的等待目标Java进程返回信息，直到用户输入Ctrl+D为止。服务端是以任务的形式在后台跑任务，植入的代码随着任务的中止而停止执行，所以任务关闭后，不会对原有性能产生太大影响，而且原则上，任何Greys的命令也不会引起任何原有业务逻辑的改变。

#### 3.5.1 监控的维度说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101737372.png)

#### 3.5.2 参数说明
方法拥有一个命名参数[c:]，意思是统计周期（cycle of output），拥有一个整形的参数值
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101737442.png)

#### 3.5.3 使用参考
```shell
ga?>monitor -c 5 *alibaba*Test printAddress
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 22 ms.
+-----------+-------+--------+-------+---------+------+-----------+------------+------------+------------+
| TIMESTAMP | CLASS | METHOD | TOTAL | SUCCESS | FAIL | FAIL-RATE | AVG-RT(ms) | MIN-RT(ms) | MAX-RT(ms) |
+-----------+-------+--------+-------+---------+------+-----------+------------+------------+------------+

+---------------------+-----------------------+--------------+-------+---------+------+-----------+------------+------------+------------+
| TIMESTAMP           | CLASS                 | METHOD       | TOTAL | SUCCESS | FAIL | FAIL-RATE | AVG-RT(ms) | MIN-RT(ms) | MAX-RT(ms) |
+---------------------+-----------------------+--------------+-------+---------+------+-----------+------------+------------+------------+
| 2015-12-06 16:34:56 | com.alibaba.AgentTest | printAddress | 5     | 3       | 2    | 40.00%    | 0.20       | 0          | 1          |
+---------------------+-----------------------+--------------+-------+---------+------+-----------+------------+------------+------------+

+---------------------+-----------------------+--------------+-------+---------+------+-----------+------------+------------+------------+
| TIMESTAMP           | CLASS                 | METHOD       | TOTAL | SUCCESS | FAIL | FAIL-RATE | AVG-RT(ms) | MIN-RT(ms) | MAX-RT(ms) |
+---------------------+-----------------------+--------------+-------+---------+------+-----------+------------+------------+------------+
| 2015-12-06 16:35:01 | com.alibaba.AgentTest | printAddress | 5     | 3       | 2    | 40.00%    | 0.00       | 0          | 0          |
+---------------------+-----------------------+--------------+-------+---------+------+-----------+------------+------------+------------+

ga?>
```



### 3.6 trace命令
命令能主动搜索class-pattern／method-pattern所渲染的方法调用路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。

#### 3.6.1 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101738274.png)

#### 3.6.2 注意事项
trace能方便的帮助你定位和发现因RT高而导致的性能问题缺陷，但其每次只能跟踪一级方法的调用链路，目前暂时没有精力去解决往下几个层级的调用。如果真有需求可以Issues我。

#### 3.6.3 使用参考
```shell
ga?>trace com.alibaba.manager.DefaultAddressManager toStringPass2
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 19 ms.
`---+Tracing for : thread_name="agent-test-address-printer" thread_id=0xb;is_daemon=false;priority=5;
`---+[0,0ms]com.alibaba.manager.DefaultAddressManager:toStringPass2()
+---[0,0ms]com.alibaba.Address:getAddressId()
+---[0,0ms]com.alibaba.Address:getAddressId()
+---[0,0ms]java.lang.Integer:valueOf()
+---[0,0ms]com.alibaba.Address:getAddressName()
+---[0,0ms]com.alibaba.Address:count()
+---[0,0ms]java.lang.Integer:valueOf()
`---[0,0ms]java.lang.String:format()
```
是不是很眼熟，没错，在JProfiler等收费软件中你曾经见识过类似的功能，这里你通过命令就能打印出指定调用路径。
[10,1ms]的含义，10所代表的含义是：当前节点的整体耗时；1的含义是：当前节点在当前步骤的耗时；两者之间用逗号分割，单位为毫秒。





### 3.7 ptrace命令
#### 3.7.1 命令解释
命令为trace命令的强化版，通过指定渲染路径来完成对方法执行路径的渲染过程
命令能主动搜索tracing-path-pattern所渲染的路径，渲染和统计整个调用链路上的所有性能开销和追踪调用链路。

#### 3.7.2 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101739533.png)

#### 3.7.3 使用例子
```shell
ga?>ptrace -t *alibaba*Test printAddress --path=*alibaba*
Press Ctrl+D to abort.
Affect(class-cnt:10 , method-cnt:36) cost in 148 ms.
`---+pTracing for : thread_name="agent-test-address-printer" thread_id=0xb;is_daemon=false;priority=5;process=1004;
`---+[2,2ms]com.alibaba.AgentTest:printAddress(); index=1021;
+---+[1,1ms]com.alibaba.manager.DefaultAddressManager:newAddress(); index=1014;
|   +---[1,1ms]com.alibaba.CountObject:<init>(); index=1012;
|   `---[1,0ms]com.alibaba.Address:<init>(); index=1013;
+---+[2,1ms]com.alibaba.manager.DefaultAddressManager:toString(); index=1020;
|   +---+[2,1ms]com.alibaba.manager.DefaultAddressManager:toStringPass1(); index=1019;
|   |   +---+[2,1ms]com.alibaba.manager.DefaultAddressManager:toStringPass2(); index=1017;
|   |   |   +---[1,0ms]com.alibaba.Address:getAddressId(); index=1015;
|   |   |   +---+[1,0ms]com.alibaba.manager.DefaultAddressManager:throwRuntimeException(); index=1016;
|   |   |   |   `---[1,0ms]throw:java.lang.RuntimeException
|   |   |   `---[1,0ms]throw:java.lang.RuntimeException
|   |   +---[2,0ms]com.alibaba.AddressException:<init>(); index=1018;
|   |   `---[2,0ms]throw:com.alibaba.AddressException
|   `---[2,0ms]throw:com.alibaba.AddressException
`---[2,0ms]throw:com.alibaba.AddressException
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|    INDEX | PROCESS-ID |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1012 |       1004 |  2015-12-06 16:46:49 |          0 |     true |    false |        0x943cff |                    CountObject |                         <init> |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1013 |       1004 |  2015-12-06 16:46:49 |          0 |     true |    false |        0x943cff |                        Address |                         <init> |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1014 |       1004 |  2015-12-06 16:46:49 |          1 |     true |    false |      0x6833b8a5 |          DefaultAddressManager |                     newAddress |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1015 |       1004 |  2015-12-06 16:46:49 |          0 |     true |    false |        0x943cff |                        Address |                   getAddressId |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1016 |       1004 |  2015-12-06 16:46:49 |          0 |    false |     true |      0x6833b8a5 |          DefaultAddressManager |          throwRuntimeException |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1017 |       1004 |  2015-12-06 16:46:49 |          0 |    false |     true |      0x6833b8a5 |          DefaultAddressManager |                  toStringPass2 |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1018 |       1004 |  2015-12-06 16:46:49 |          0 |     true |    false |      0x67e7a923 |               AddressException |                         <init> |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1019 |       1004 |  2015-12-06 16:46:49 |          1 |    false |     true |      0x6833b8a5 |          DefaultAddressManager |                  toStringPass1 |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1020 |       1004 |  2015-12-06 16:46:49 |          1 |    false |     true |      0x6833b8a5 |          DefaultAddressManager |                       toString |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1021 |       1004 |  2015-12-06 16:46:49 |          2 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
```



### 3.8 watch命令
能方便的让你观察到指定方法的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写OGNL表达式进行对应变量的查看。

#### 3.8.1 参数说明
watch的参数比较多，主要是因为它能在4个不同的场景观察对象
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101740035.png)
这里重点要说明的是观察表达式，观察表达式的构成主要由OGNL表达式组成，所以你可以这样写params[0]+"$"+target，只要是一个合法的OGNL表达式，都能被正常支持。
观察的维度也比较多，主要体现在参数advice的数据结构上。Advice参数最主要是封装了通知节点的所有信息。参考表达式核心变量中关于该节点的描述:https://github.com/oldmanpushcart/greys-anatomy/wiki/greys-pdf#advice-class。

#### 3.8.2 使用参考
```shell
ga?>watch -b *Test printAddress '"params[0]="+params[0]'
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 32 ms.
params[0]=3163
params[0]=3164
params[0]=3165
params[0]=3166
```

这里需要说明的一个参数x，这个参数决定了输出的结果的层级遍历输出对象，当加上这个参数之后，greys会将输出的对象按照指定层级进行剥开。-x 1表明展开第1层级。
```shell
ga?>watch -s com.alibaba.manager.DefaultAddressManager newAddress returnObj -x 1
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 34 ms.
@Address[
username=@String[dukun],
addressId=@Integer[3244],
addressName=@String[ADDRESS],
]
```




### 3.9 tt命令
时间隧道命令是我在使用watch命令进行问题排查的时候衍生出来的想法。watch虽然很方便和灵活，但需要提前想清楚观察表达式的拼写，这对排查问题而言要求太高，因为很多时候我们并不清楚问题出自于何方，只能靠蛛丝马迹进行猜测。
这个时候如果能记录下当时方法调用的所有入参和返回值、抛出的异常会对整个问题的思考与判断非常有帮助。
于是乎，TimeTunnel命令就诞生了。

#### 3.9.1 记录方法的调用
**1) 基本用法**
   对于一个最基本的使用来说，就是记录下当前方法的每次调用环境现场。
```shell
   ga?>tt -t -n 3 *Test printAddress
   Press Ctrl+D to abort.
   Affect(class-cnt:1 , method-cnt:1) cost in 33 ms.
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |    INDEX | PROCESS-ID |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1036 |       1009 |  2015-12-06 16:57:06 |          1 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1037 |       1010 |  2015-12-06 16:57:07 |          0 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1038 |       1011 |  2015-12-06 16:57:08 |          0 |     true |    false |       0x2062a3d |                      AgentTest |                   printAddress |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   ga?>
```

2) 命令参数解析
   -t :tt命令有很多个主参数，-t就是其中之一。这个参数的表明希望记录下类*Test的print方法的每次执行情况。
   -n 3 :当你执行一个调用量不高的方法时可能你还能有足够的时间用CTRL+D中断tt命令记录的过程，但如果遇到调用量非常大的方法，瞬间就能将你的JVM内存撑爆。
此时你可以通过-n参数指定你需要记录的次数，当达到记录次数时greys会主动中断tt命令的记录过程，避免人工操作无法停止的情况。

#### 3.9.2 表格字段说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101743466.png)

#### 3.9.3 条件表达式
不知道大家是否有在使用过程中遇到以下困惑
似乎很难区分出重载的方法
我只需要观察特定参数，但是tt却全部都给我记录了下来

从1.6.0.6版本之后，应广大妇女群众的要求，增加了条件表达式，这样你可以通过简单的条件表达式解决上边的困惑。
条件表达式也是用OGNL来编写，核心的判断对象依然是Advice对象。
除了tt命令之外，watch、trace、stack命令也都支持条件表达式

**1)解决方法重载**
`tt -t *Test print params[0].length==1`

通过制定参数个数的形式解决不同的方法签名，如果参数个数一样，你还可以这样写
`tt -t *Test print 'params[1].class == Integer.class'`


**2) 解决指定参数**
`tt -t *Test print params[0].mobile=="13989838402"`


**3) 检索调用记录**
   当你用tt记录了一大片的时间片段之后，你希望能从中筛选出自己需要的时间片段，这个时候你就需要对现有记录进行检索。
   假设我们有这些记录
```shell
   ga?>tt -l
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |    INDEX | PROCESS-ID |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1047 |       1020 |  2015-12-06 17:03:00 |          1 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1048 |       1021 |  2015-12-06 17:03:01 |          0 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1049 |       1022 |  2015-12-06 17:03:01 |          1 |     true |    false |       0x2062a3d |                      AgentTest |                   printAddress |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1050 |       1023 |  2015-12-06 17:03:01 |          0 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1051 |       1024 |  2015-12-06 17:03:02 |          1 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1052 |       1025 |  2015-12-06 17:03:02 |          1 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1053 |       1026 |  2015-12-06 17:03:02 |          0 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1054 |       1027 |  2015-12-06 17:03:03 |          0 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1055 |       1028 |  2015-12-06 17:03:03 |          0 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   |     1056 |       1029 |  2015-12-06 17:03:03 |          0 |     true |    false |       0x2062a3d |                      AgentTest |                      printUser |
   +----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
   Affect(row-cnt:10) cost in 3 ms.
   ga?>
```
我需要筛选出printAddress方法的调用信息
```shell
ga?>tt -s method.name=="printAddress"
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|    INDEX | PROCESS-ID |            TIMESTAMP |   COST(ms) |   IS-RET |   IS-EXP |          OBJECT |                          CLASS |                         METHOD |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1049 |       1022 |  2015-12-06 17:03:01 |          1 |     true |    false |       0x2062a3d |                      AgentTest |                   printAddress |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1052 |       1025 |  2015-12-06 17:03:02 |          1 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
|     1055 |       1028 |  2015-12-06 17:03:03 |          0 |    false |     true |       0x2062a3d |                      AgentTest |                   printAddress |
+----------+------------+----------------------+------------+----------+----------+-----------------+--------------------------------+--------------------------------+
Affect(row-cnt:3) cost in 8 ms.
ga?>
```
你需要一个-s参数。同样的，搜索表达式的核心对象依旧是Advice对象。

**4) 查看调用信息**
   对于具体一个时间片的信息而言，你可以通过-i参数后边跟着对应的INDEX编号查看到他的详细信息。
```
   ga?>tt -i 1055
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |           INDEX | 1055                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |      PROCESS-ID | 1028                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |      GMT-CREATE | 2015-12-06 17:03:03                                                                                                                                    |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |        COST(ms) | 0                                                                                                                                                      |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |          OBJECT | 0x2062a3d                                                                                                                                              |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |           CLASS | com.alibaba.AgentTest                                                                                                                                  |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |          METHOD | printAddress                                                                                                                                           |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |       IS-RETURN | false                                                                                                                                                  |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |    IS-EXCEPTION | true                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |   PARAMETERS[0] | 3789                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   | THROW-EXCEPTION | com.alibaba.AddressException: java.lang.RuntimeException: test                                                                                         |
   |                 |     at com.alibaba.manager.DefaultAddressManager.toStringPass1(DefaultAddressManager.java:22)                                                          |
   |                 |     at com.alibaba.manager.DefaultAddressManager.toString(DefaultAddressManager.java:15)                                                               |
   |                 |     at com.alibaba.AgentTest.printAddress(AgentTest.java:80)                                                                                           |
   |                 |     at com.alibaba.AgentTest.access$300(AgentTest.java:7)                                                                                              |
   |                 |     at com.alibaba.AgentTest$3.null(Unknown Source)                                                                                                    |
   |                 | Caused by: java.lang.RuntimeException: test                                                                                                            |
   |                 |     at com.alibaba.manager.DefaultAddressManager.throwRuntimeException(DefaultAddressManager.java:39)                                                  |
   |                 |     at com.alibaba.manager.DefaultAddressManager.toStringPass2(DefaultAddressManager.java:29)                                                          |
   |                 |     at com.alibaba.manager.DefaultAddressManager.toStringPass1(DefaultAddressManager.java:20)                                                          |
   |                 |     ... 4 more                                                                                                                                         |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |           STACK | thread_name="agent-test-address-printer" thread_id=0xb;is_daemon=false;priority=5;                                                                     |
   |                 |     @com.alibaba.AgentTest.access$300()                                                                                                                |
   |                 |         at com.alibaba.AgentTest$3.null(null:-1)                                                                                                       |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   Affect(row-cnt:1) cost in 7 ms.
   ga?>
```

**5) 重做一次调用**
   当你少少做了一些调整之后，你可能需要前端系统重新触发一次你的调用，此时得求爷爷告奶奶的需要前端配合联调的同学再次发起一次调用。而有些场景下，这个调用不是这么好触发的。
   tt命令由于保存了当时调用的所有现场信息，所以我们可以自己主动对一个INDEX编号的时间片自主发起一次调用，从而解放你的沟通成本。此时你需要-p参数。
```
   ga?>tt -i 1055 -p
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |           INDEX | 1055                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |      PROCESS-ID | 1028                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |      GMT-CREATE | 2015-12-06 17:03:03                                                                                                                                    |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |        COST(ms) | 1                                                                                                                                                      |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |          OBJECT | 0x2062a3d                                                                                                                                              |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |           CLASS | com.alibaba.AgentTest                                                                                                                                  |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |          METHOD | printAddress                                                                                                                                           |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |       IS-RETURN | true                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |    IS-EXCEPTION | false                                                                                                                                                  |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |   PARAMETERS[0] | 3789                                                                                                                                                   |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |      RETURN-OBJ | 1                                                                                                                                                      |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   |           STACK | thread_name="ga-command-execute-daemon" thread_id=0x22;is_daemon=true;priority=9;                                                                      |
   |                 |     @com.github.ompc.greys.core.command.TimeTunnelCommand$6.action()                                                                                   |
   |                 |         at com.github.ompc.greys.core.server.DefaultCommandHandler.execute(DefaultCommandHandler.java:210)                                             |
   |                 |         at com.github.ompc.greys.core.server.DefaultCommandHandler.executeCommand(DefaultCommandHandler.java:87)                                       |
   |                 |         at com.github.ompc.greys.core.server.GaServer$4.run(GaServer.java:332)                                                                         |
   |                 |         at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1142)                                                             |
   |                 |         at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:617)                                                             |
   |                 |         at java.lang.Thread.run(Thread.java:745)                                                                                                       |
   +-----------------+--------------------------------------------------------------------------------------------------------------------------------------------------------+
   Time fragment[1055] successfully replayed.
   Affect(row-cnt:1) cost in 2 ms.
   ga?>
```
你会发现结果虽然一样，但调用的路径发生了变化，有原来的程序发起变成了greys自己的内部线程发起的调用了。
得益于Greys的ClassLoader隔离策略，Greys在内部自己发起线程请求调用的时候，依然采用的是目标类所归属的ClassLoader，所以在OSGI、Tomcat容器等场景下，Greys依然能正确的重做此次调用。

**6) 需要强调的点**
1.ThreadLocal信息丢失
   很多框架偷偷的将一些环境变量信息塞到了发起调用线程的ThreadLocal中，由于调用线程发生了变化，这些ThreadLocal线程信息无法通过greys保存，所以这些信息将会丢失。
   一些常见的CASE比如：阿里鹰眼系统的TraceId、阿里全链路平台的压测流量标记位。

2.引用的对象
需要强调的是，tt命令是将当前环境的对象引用保存起来，但仅仅也只能保存一个引用而已。如果方法内部对入参进行了变更，或者返回的对象经过了后续的处理，那么在tt查看的时候将无法看到当时最准确的值。这也是为什么watch命令存在的意义。




### 3.10 stack命令
很多时候我们都知道一个方法被执行，但这个方法被执行的路径非常多。或者你根本就不知道这个方法是从那里被执行了，正在郁闷，正在彷徨。此时你需要的是stack命令。

#### 3.10.1 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101746721.png)

#### 3.10.2 使用例子
```
ga?>stack com.alibaba.manager.DefaultAddressManager newAddress
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:1) cost in 36 ms.
thread_name="agent-test-address-printer" thread_id=0xb;is_daemon=false;priority=5;
@java.lang.reflect.Method.invoke()
at com.alibaba.manager.DefaultAddressManager.newAddress(DefaultAddressManager.java:-1)
at com.alibaba.AgentTest.printAddress(AgentTest.java:73)
at com.alibaba.AgentTest.access$300(AgentTest.java:7)
at com.alibaba.AgentTest$3.null(null:-1)
```





### 3.11 js命令
js命令几经波折，几乎在两个重要版本中绝迹，所以我不得不先介绍这个命令的历史背景。

在GREYS的1.5版本时代，字节码增强使用的是javassist进行，里面的黑盒严重，我比较难介入和调试其中的字节码生成过程。在这种情况下我们发现了一个JavaScript引擎rhino与Javassist结合存在严重漏洞的问题却没有更多调试的手段。

考虑到性能、后续扩展的发展，GREYS从1.6开始替换成asm，之后我拥有了最精细的字节码控制权限。在1.7.4.0版本中我恢复了GREYS对JavaScript的支持，并通过了之前BUG的测试。一切正常。

现有的GREYS命令不能完全满足所有人的需求，有一些需要根据业务场景进行个性化处理的场景，现有的GREYS命令就比较难支持了。

比如我需要查看目标JVM中所有SQL的执行情况，因为java.sql.PreparedStatement在不同的JDBC协议实现下，存储的原生SQL获取的方式也不尽相同，JDBC规范中并未提供标准的API获取原生的SQL。这个时候需要对原生SQL进行统计就必须进行个性化开发。

#### 3.11.1 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101747901.png)

#### 3.11.2 使用例子
**1) 运行第一个JavaScript**
1.目标
   我们编写一个watch.js脚本，这个脚本的主要是在方法运行之前输出方法的大概信息。类似于
   `watch -b *Test print* clazz.name+"."+method.name+"()"`

2.步骤
首先我们先生成一个脚本文件
`touch /tmp/watch.js`

然后往里面写入以下内容
```
require(['greys'], function (greys) {
greys.watching({

        before: function (o, a) {
            o.println(a.clazz.name+"."+a.method.name+"()");
        },
    
    });
})
```
接下来启动GREYS之行命令运行
```
js *Test print* /tmp/watch.js
```
执行效果
```
ga?>js *Test print* /tmp/watch.js
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:2) cost in 35 ms.
com.alibaba.AgentTest.printAddress()
com.alibaba.AgentTest.printUser()
com.alibaba.AgentTest.printAddress()
com.alibaba.AgentTest.printUser()
com.alibaba.AgentTest.printUser()
com.alibaba.AgentTest.printAddress()
```

**2) 运行一个远程的JavaScript**
除了本地临时写代码之外，你也可以将平时积累好的脚本代码放在远程服务端（比如Github），可以使用远程加载的方式运行。
 提前准备了一个watch.js文件，内容和原来差不多

```javascript
require(['greys'], function (greys) {
   greys.watching({

       before: function (o, a) {
           o.println('call from remote script: '+a.clazz.name+"."+a.method.name+"()");
       },

   });
}
```



1.执行命令

```
js *Test print* watch.js
```
2.执行效果
```
ga?>js *Test print* watch.js
Press Ctrl+D to abort.
Affect(class-cnt:1 , method-cnt:2) cost in 43 ms.
call from remote script: com.alibaba.AgentTest.printUser()
call from remote script: com.alibaba.AgentTest.printAddress()
call from remote script: com.alibaba.AgentTest.printUser()
call from remote script: com.alibaba.AgentTest.printAddress()
```
更详细的命令帮助可以见JavaScriptSupport:https://github.com/oldmanpushcart/greys-anatomy/wiki/JavaScriptSupport





### 3.12 version命令
这估计是最好理解的一个命令了，输出当前Greys的版本号，这里输出的版本号不是Client的版本，而是当前加载到目标Java进程中的Greys版本。




### 3.13 quit命令
这里说明下与shutdown命令的区别，quit命令仅仅是将客户端关闭，而不会将目标Java进程中的与Greys的Server关闭。所以如果仅仅是希望简单的退出Greys控制台，则使用quit命令足矣。
一旦Greys控制台退出，控制台所绑定的Session将会被关闭，Session上所有存活的事务也都会被中止并释放。




### 3.14 shutdown命令
命令执行后将会完成两件事：
关闭Greys在目标Java所加载的Socket服务，所占用的端口将会被释放，同时，本地的Greys控制台也因为远程Socket关闭而主动退出。
重置所有被Greys所增强的类。同reset命令。




### 3.15 reset命令
重置指定被Greys所增强的类。




### 3.16 session命令
会话命令是在1.6版本之后新增，整个命令的定位是维护好会话级的参数。目前可修改的就一个字符集。

#### 3.16.1 参数说明
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211101750668.png)
#### 3.16.2 使用例子
**1) 直接查看会话信息**
```
   ga?>session
   +------------+------------------+
   |   JAVA_PID | 8609             |
   +------------+------------------+
   | SESSION_ID | 2                |
   +------------+------------------+
   |   DURATION | 300000           |
   +------------+------------------+
   |    CHARSET | UTF-8            |
   +------------+------------------+
   |     PROMPT | ga?>             |
   +------------+------------------+
   |       FROM | /127.0.0.1:58186 |
   +------------+------------------+
   |         TO | /127.0.0.1:3658  |
   +------------+------------------+
   Affect(row-cnt:1) cost in 0 ms.
   ga?>
```

**2) 修改字符集**
```
   ga?>session -c GBK
   change charset before[UTF-8] -> new[GBK]
   Affect(row-cnt:1) cost in 26 ms.
```



### 3.17 jvm命令
   查看当前JVM的信息，无参数。
```
   ga?>jvm
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |           CATEGORY | INFO                                                                                                |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |            RUNTIME |              MACHINE-NAME : 25428@vlinux-air.local                                                  |
   |                    |            JVM-START-TIME : 2015-06-16 22:12:20                                                     |
   |                    |   MANAGEMENT-SPEC-VERSION : 1.2                                                                     |
   |                    |                 SPEC-NAME : Java Virtual Machine Specification                                      |
   |                    |               SPEC-VENDOR : Oracle Corporation                                                      |
   |                    |              SPEC-VERSION : 1.8                                                                     |
   |                    |                   VM-NAME : Java HotSpot(TM) 64-Bit Server VM                                       |
   |                    |                 VM-VENDOR : Oracle Corporation                                                      |
   |                    |                VM-VERSION : 25.25-b02                                                               |
   |                    |           INPUT-ARGUMENTS : []                                                                      |
   |                    |                CLASS-PATH : .                                                                       |
   |                    |           BOOT-CLASS-PATH : /Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/Home/jre/li  |
   |                    |                             b/resources.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Conte |
   |                    |                             nts/Home/jre/lib/rt.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_25.j |
   |                    |                             dk/Contents/Home/jre/lib/sunrsasign.jar:/Library/Java/JavaVirtualMachin |
   |                    |                             es/jdk1.8.0_25.jdk/Contents/Home/jre/lib/jsse.jar:/Library/Java/JavaVir |
   |                    |                             tualMachines/jdk1.8.0_25.jdk/Contents/Home/jre/lib/jce.jar:/Library/Jav |
   |                    |                             a/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/Home/jre/lib/charsets.ja |
   |                    |                             r:/Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/Home/jre/l |
   |                    |                             ib/jfr.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_25.jdk/Contents/H |
   |                    |                             ome/jre/classes                                                         |
   |                    |              LIBRARY-PATH : /Users/vlinux/Library/Java/Extensions:/Library/Java/Extensions:/Networ  |
   |                    |                             k/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java |
   |                    |                             :.                                                                      |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |      CLASS-LOADING |        LOADED-CLASS-COUNT : 3045                                                                    |
   |                    |  TOTAL-LOADED-CLASS-COUNT : 3045                                                                    |
   |                    |      UNLOADED-CLASS-COUNT : 0                                                                       |
   |                    |                IS-VERBOSE : false                                                                   |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |        COMPILATION |                      NAME : HotSpot 64-Bit Tiered Compilers                                         |
   |                    |        TOTAL-COMPILE-TIME : 8903(ms)                                                                |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   | GARBAGE-COLLECTORS |               PS Scavenge : 4/40(ms)                                                                |
   |                    |              [count/time]                                                                           |
   |                    |              PS MarkSweep : 0/0(ms)                                                                 |
   |                    |              [count/time]                                                                           |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |    MEMORY-MANAGERS |          CodeCacheManager : Code Cache                                                              |
   |                    |         Metaspace Manager : Metaspace                                                               |
   |                    |                             Compressed Class Space                                                  |
   |                    |               PS Scavenge : PS Eden Space                                                           |
   |                    |                             PS Survivor Space                                                       |
   |                    |              PS MarkSweep : PS Eden Space                                                           |
   |                    |                             PS Survivor Space                                                       |
   |                    |                             PS Old Gen                                                              |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |             MEMORY |         HEAP-MEMORY-USAGE : 163053568/134217728/1908932608/54796080                                 |
   |                    | [committed/init/max/used]                                                                           |
   |                    |      NO-HEAP-MEMORY-USAGE : 32571392/2555904/-1/31757608                                            |
   |                    | [committed/init/max/used]                                                                           |
   |                    |    PENDING-FINALIZE-COUNT : 0                                                                       |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |   OPERATING-SYSTEM |                        OS : Mac OS X                                                                |
   |                    |                      ARCH : x86_64                                                                  |
   |                    |          PROCESSORS-COUNT : 4                                                                       |
   |                    |              LOAD-AVERAGE : 2.50634765625                                                           |
   |                    |                   VERSION : 10.10.3                                                                 |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   |             THREAD |                     COUNT : 8                                                                       |
   |                    |              DAEMON-COUNT : 7                                                                       |
   |                    |                LIVE-COUNT : 9                                                                       |
   |                    |             STARTED-COUNT : 19                                                                      |
   +--------------------+-----------------------------------------------------------------------------------------------------+
   Affect cost in 22 ms.
```


