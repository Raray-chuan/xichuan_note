## BTrace 是什么
BTrace 是检查和解决线上的问题的杀器，BTrace 可以通过编写脚本的方式，获取程序执行过程中的一切信息，并且，注意了，不用重启服务，是的，不用重启服务。写好脚本，直接用命令执行即可，不用动原程序的代码。



## 原理
总体来说，BTrace 是基于动态字节码修改技术(Hotswap)来实现运行时 java 程序的跟踪和替换。大体的原理可以用下面的公式描述：Client(Java compile api + attach api) + Agent（脚本解析引擎 + ASM + JDK6 Instumentation） + Socket其实 BTrace 就是使用了 java attach api 附加 agent.jar ，然后使用脚本解析引擎+asm来重写指定类的字节码，再使用 instrument 实现对原有类的替换。




## 安装和配置
本次安装和配置在 Linux Ubuntu 14.04 下进行。目前 BTrace 的最新版本为 1.3.9，代码托管在 [github] 上。第一步，在github 上下载 releases 版 btrace-bin-1.3.9.tgz，zip 版的没有 build 目录。第二步，解压 btrace-bin-1.3.9.tgz 到一个目录即可，例如 /home/fengzheng/soft/btrace , 到这一步其实就可以用了，只是执行脚本的时候需要在 btrace 命令前加上绝对路径，如果想在任意目录可执行，进行下一步第三步，配置环境变量，配置的环境变量包括 JAVA_HOME和 BTRACE_HOME ，例如我的配置如下：
```bash
export JAVA_HOME=/root/soft/jdk1.8.0_111
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib 
export PATH=${JAVA_HOME}/bin:$PATH
export BTRACE_HOME=/root/soft/btrace
export PATH=$PATH:$BTRACE_HOME/bin
```

之后执行命令 source /etc/profile ，使环境变量立即生效。接下来在任意目录执行 btrace命令，都可以执行成功了。



## 简单测试用例　　
btrace 最简单的语法是 btrace $pid script.java，所以需要知道要探测的 Java程序的进程id，然后编写一个探测脚本即可。
1. 写一个常驻内存的 Java 程序，这里写了一个无限循环，每隔5秒钟输出一组计算结果，内容如下：
```java
public class NumberUtil {
 
    public int sum(){
        int result = 0;
        for(int i = 0; i< 100; i++){
            result += i * i;
        }
        return result;
    }
 
    public static void main(String[] args){
        while (true) {
            Thread.currentThread().setName("计算");
            NumberUtil util = new NumberUtil();
            int result = util.sum();
            System.out.println(result);
            try {
                Thread.sleep(5000);
            }catch (InterruptedException e){
 
            }
        }
    }
}
```

顺便说一下命令行编译和运行 Java 的过程：
**编译：**javac -d . NumberUtil.java，定位到 NumberUtil.java 所在目录，然后执行此命令行，将会在当前目录（.表示当前目录）生成包名所示的目录结构，kite/lab/utils/NumberUtil.class
**执行：**java kite.lab.utils.NumberUtil 即可　　

2. 执行上面的程序后，可用 jps 命令查看 pid(一般情况下用哪个账号启动的程序，就要用哪个账号执行 jps ，root 账号除外)，执行 jps 命令看到如下结果：
```bash
root@ubuntu:/root/codes/btrace# jps
10906 Jps
10860 NumberUtil
```

3. 可以看到刚刚执行的 java 进程为 10860　　
4. 编写 btrace 脚本，脚本内容简单如下：
```java
import com.sun.btrace.annotations.*;
import static com.sun.btrace.BTraceUtils.Strings.strcat;
import static com.sun.btrace.BTraceUtils.jstack;
import static com.sun.btrace.BTraceUtils.println;
import static com.sun.btrace.BTraceUtils.str;
@BTrace
public class NumberUtilBTrace {
    
    @OnMethod(
            clazz="kite.lab.utils.NumberUtil",
            method="sum",
            location=@Location(Kind.RETURN)
    )
    public static void func(@Return int result) {
        println("trace: =======================");
        println(strcat("result:", str(result)));
        jstack();
    }
}
```
意思是在执行结束后（location=@Location(Kind.RETURN) 表示执行结束）输出结果和堆栈信息　　

5. 预编译：执行之前可以用预编译命令检查脚本的正确性，预编译命令为 btracec，它是一个 javac-like 命令，btracec NumberUtilBTrace.java

6. 调用命令行执行，btrace 10860 NumberUtilBTrace.java ，(如果要保存到本地文件中，可以使用转向命令 btrace 10860 NumberUtilBTrace.java > mylog.log )打印的信息如下
```bash
trace: =======================
result:328350
kite.lab.utils.NumberUtil.sum(NumberUtil.java:16)
kite.lab.utils.NumberUtil.main(NumberUtil.java:27)
```

7. 按ctrl + c ，会给出退出提示，再按 1 退出





## 使用场景
BTrace 是一个事后工具，所谓事后工具就是在服务已经上线了，但是发现存在以下问题的时候，可以用 BTrace。
* 比如哪些方法执行太慢，例如监控执行时间超过1s的方法
* 查看哪些方法调用了 System.gc() ，调用栈是怎样的
* 查看方法参数或对象属性
* 哪些方法发生了异常
  多说一点，为了更好解决问题，最好还要配合事前准备和进行中监控，事前准备就是埋点嘛，在一些可能出现问题的方法中进行日志输出，进行中监控就是利用一些实时监控工具，例如 VisualVM 、jmc 这些带界面的工具或者 jdk 提供的命令行工具等，再高级一点的就是利用 Graphite 这样的Metrics 工具配合 web 界面展示出来。






## 使用限制
为了保证trace语句只读,最小化对被检测程序造成影响， BTrace对trace脚本有一些限制(比如不能改变被trace代码中的状态)
* BTrace class不能新建类, 新建数组, 抛异常, 捕获异常,
* 不能调用实例方法以及静态方法(com.sun.btrace.BTraceUtils除外)
* 不能将目标程序和对象赋值给BTrace的实例和静态field
* 不能定义外部, 内部, 匿名, 本地类
* 不能有同步块和方法
* 不能有循环
* 不能实现接口, 不能扩展类
* 不能使用assert语句, 不能使用class字面值







## 拦截方法定义
@OnMethod 可以指定 clazz 、method、location。由此组成了在什么时机（location 决定）监控某个类/某些类（clazz 决定）下的某个方法/某些方法(method 决定)。

**如何定位**
1. 精准定位 :直接定位到一个类下的一个方法，上面测试用的例子就是
2. 正则表达式定位 :正则表达式在两个"/" 之间，例如下面的例子,监控 javax.swing 包下的所有方法，注意正式环境中，范围尽可能小一点，太大了性能会有影响。

```java
@OnMethod(clazz="/javax\\.swing\\..*/", method="/.*/")
public static void swingMethods( @ProbeClassName String probeClass, @ProbeMethodName String probeMethod) {
   print("entered " + probeClass + "."  + probeMethod);
}
```
通过在拦截函数的定义里注入@ProbeClassName String probeClass, @ProbeMethodName String probeMethod 参数，告诉脚本实际匹配到的类和方法名。

3. 按接口或继承类定位
     例如要匹配继承或实现了 com.kite.base 的接口或基类的，只要在类前加上 + 号就可以了，例如 :@OnMethod(clazz="+com.kite.base", method="doSome")
4. 按注解定位
     在前面加上 @ 即可，例如@OnMethod(clazz="@javax.jws.WebService", method="@javax.jws.WebMethod")　　






## 拦截时机
拦截时机由 location 决定，当然也可为同一个定位加入多个拦截时机，即可以在进入方法时拦截、方法返回时拦截、抛出异常时拦截
1. Kind.Entry与Kind.Return
     分别表示函数的开始和返回，不写 location 的情况下，默认为 Kind.Entry,仅获取参数值，可以用 Kind.Entry ，要获取返回值或执行时间就要用 Kind.Return

2. Kind.Error, Kind.Throw和 Kind.Catch
     表示异常被 throw 、异常被捕获还有异常发生但是没有被捕获的情况，在拦截函数的参数定义里注入一个Throwable的参数，代表异常

```java
@OnMethod(clazz = "com.kite.demo", location = @Location(value = Kind.LINE, line = 20))
public static void onBind() {
 
   println("执行到第20行");
 
}
```

```java
@OnMethod(clazz = "java.net.ServerSocket", method = "bind", location =@Location(Kind.ERROR)) 
public static void onBind(Throwable exception, @Duration long duration){ 

}
```

3. Kind.Call 和 Kind.Line　　
     Kind.Call 表示被监控的方法调用了哪些其他方法，例如：

```java
@OnMethod(clazz = "com.kite",
            method = "login",
            location = @Location(value = Kind.CALL, clazz = "/.*/", method = "/.*/", where = Where.AFTER))
    public static void onBind(@Self Object self, @TargetInstance Object instance, @TargetMethodOrField String method, @Duration long duration){
        println(strcat("self: ", str(self)));
        println(strcat("instance: ", str(instance)));
        println(strcat("method: ", str(method)));
        println(strcat("duration(ms): ", str(duration / 1000000)));
    }
```

@Self 表示当前监控的函数所在类，如果是静态类则为空，@TargetInstance 表示函数中调用的方法或属性所在的类，如果是静态方法则为空，@TargetMethodOrField 表示调用的方法或属性，如果要获取执行时间，那么 where 必须设置为 Where.AFTER
Kind.Line 监测类是否执行到了设置的行数，例如：

```java
@OnMethod(clazz = "com.kite.demo", location = @Location(value = Kind.LINE, line = 20))
public static void onBind() {
 
   println("执行到第20行");
 
}
```






## 几个例子
查看谁调用了GC
```java
@OnMethod(clazz = "java.lang.System", method = "gc")
    public static void onSystemGC() {
        println("entered System.gc()");
        jstack();
    }
```

打印耗时超过100ms的方法
```java
@OnMethod(clazz = "/com\\.kite\\.controller\\..*/",method = "/.*/",location = @Location(Kind.RETURN))
    public static void slowQuery(@ProbeClassName String pcn,@ProbeMethodName String probeMethod, @Duration long duration){
        if(duration > 1000000 * 100){
            println(strcat("类：", pcn));
            println(strcat("方法：", probeMethod));
            println(strcat("时长：", str(duration / 1000000)));
        }
    }
```
BTrace 提供了一系列的 sample, 可到 github 上查看。






## 注意问题
如果出现 Unable to open socket file: target process not responding or HotSpot VM not loaded 这个问题，可能的原因是执行 BTrace 脚本的用户和 Java 进程运行的用户不是同一个，使用 ps -aux | grep $pid查看一下 Java 进程的执行用户，保证和 BTrace 脚本执行用户相同即可　　





来自：

[BTrace : Java 线上问题排查神器](https://www.cnblogs.com/fengzheng/p/7416942.html)

