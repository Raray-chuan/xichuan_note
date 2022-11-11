## 1.前言
在工作突然有一个需求。线上运维的一个tomcat的web项目，运行的程序不正常。需要修改代码。可是这个项目代码非常的老，并且公司存储的源代码跟线上的不一致。
我了个擦，没有源代码但是还要结局客户的问题。只能到线上将对应程序的class文件拷贝到本地进行修改，每修改一部分就上传到线上覆盖掉之前的class文件，重启tomcat进行测试。（过程想当麻烦）
修改class字节码文件用到 IDEA工具来反编译class进行查看代码，javassist工具进行修改。
修改method中的方法时，主要是对 书写的代码 格式有很多要求

Java 字节码以二进制的形式存储在 .class 文件中，每一个 .class 文件包含一个 Java 类或接口。Javaassist 就是一个用来 处理 Java 字节码的类库。它可以在一个已经编译好的类中添加新的方法，或者是修改已有的方法，并且不需要对字节码方面有深入的了解。同时也可以去生成一个新的类对象，通过完全手动的方式。



## 2.重要方法介绍
首先需要引入jar包：
```xml
<dependency>
<groupId>org.javassist</groupId>
<artifactId>javassist</artifactId>
<version>3.27.0-GA</version>
</dependency>
```

在 Javassist 中，类 Javaassit.CtClass 表示 class 文件。一个 GtClass (编译时类）对象可以处理一个 class 文件，ClassPool是 CtClass 对象的容器。它按需读取类文件来构造 CtClass 对象，并且保存 CtClass 对象以便以后使用。
需要注意的是 ClassPool 会在内存中维护所有被它创建过的 CtClass，当 CtClass 数量过多时，会占用大量的内存，API中给出的解决方案是 有意识的调用CtClass的detach()方法以释放内存。

### 1）ClassPool需要关注的方法：
```
getDefault : 返回默认的ClassPool 是单例模式的，一般通过该方法创建我们的ClassPool；
appendClassPath, insertClassPath : 将一个ClassPath加到类搜索路径的末尾位置 或 插入到起始位置。通常通过该方法写入额外的类搜索路径，以解决多个类加载器环境中找不到类的尴尬；
toClass : 将修改后的CtClass加载至当前线程的上下文类加载器中，CtClass的toClass方法是通过调用本方法实现。需要注意的是一旦调用该方法，则无法继续修改已经被加载的class；
get , getCtClass : 根据类路径名获取该类的CtClass对象，用于后续的编辑。
```

### 2）CtClass需要关注的方法：
```
freeze : 冻结一个类，使其不可修改；
isFrozen : 判断一个类是否已被冻结；
prune : 删除类不必要的属性，以减少内存占用。调用该方法后，许多方法无法将无法正常使用，慎用；
defrost : 解冻一个类，使其可以被修改。如果事先知道一个类会被defrost， 则禁止调用 prune 方法；
detach : 将该class从ClassPool中删除；
writeFile : 根据CtClass生成 .class 文件；
toClass : 通过类加载器加载该CtClass。
```
上面我们创建一个新的方法使用了CtMethod类。CtMthod代表类中的某个方法，可以通过CtClass提供的API获取或者CtNewMethod新建，通过CtMethod对象可以实现对方法的修改。

### 3）CtMethod中的一些重要方法：
```
insertBefore : 在方法的起始位置插入代码；
insterAfter : 在方法的所有 return 语句前插入代码以确保语句能够被执行，除非遇到exception；
insertAt : 在指定的位置插入代码；
setBody : 将方法的内容设置为要写入的代码，当方法被 abstract修饰时，该修饰符被移除；
make : 创建一个新的方法。
```


### 4）写method的 body代码注意点：
a.如果方法要使用 参数的话 不能直接 使用 。 需要用 $1,$2,$3 来代替。 1-2-3代表前后顺序
```
列如：
方法上有 (int a,int b) a和b两个参数
那么在 setBody方法中要 使用 a 和 b
就需要用 $1=>a  $2=>b 使用$1,$2来代替 ， $0代码的是this

$args ：$args 指的是方法所有参数的数组类似Object[]，需要注意$args[0]对应的是$1,而不是$0

$r：指的是方法返回值的类型，主要用在类型的转型上

$w：$w代表一个包装类型。主要用在转型上。比如：Integer i = ($w)5;如果该类型不是基本类型，则会忽略

$type：返回结果值的类型

具体还有很多的符号可以使用，但是不同符号在不同的场景下会有不同的含义，所以在这里就不在赘述，可以看javassist 的说明文档。
http://www.javassist.org/tutorial/tutorial2.html
```

b.在写入某些对象时要加上包全路径名称
```
列如： Date ， List ， Map ， 还有一些自定义的pojo类或者 service
// 第三方自己定义的 service 和 pojo 可以在最开始通过
cPool.importPackage("com.gdzy.JZFW.service"); 来进行导入
```

c.在使用<>这样的泛型定义 标识时 要使用 /* */ 将其包括起来
```
列如： List<String> 要写成 List/*<String>*/
```

d.还有一个问题是我需改的class文件需要再重新上线到tomcat中运行，但是我每次修改完运行时都会报一个 线程没有正常结束 的错误，导致tomcat启动不了，最后发现是某些类型的定义不能使用 引用类型 只能 使用 基本类型
```
列如： Float，Long
不能使用 Float.valueOf() 只能使用 Float.parseFloat() 来进行转换类型
```




## 3.示例
### 1）修改class文件中的某个方法
此方法只能整体修改 method的所有内容。目前没有找到可以局部修改代码的方法。
```
import java.io.IOException;

import javassist.CannotCompileException;
import javassist.ClassPool;
import javassist.CtClass;
import javassist.CtField;
import javassist.CtMethod;
import javassist.CtNewMethod;
import javassist.NotFoundException;

public class UpdateMethod {

    private static String pathName = "D:\\Java\\xxxxx\\test\\bin";
    private static String className = "com.lucumt.Test1";
    
    public static void main(String[] args) {
        updateMethod();
    }

    public static void updateMethod(){
        try {
            ClassPool cPool = new ClassPool(true);
            //如果该文件引入了其它类，需要利用类似如下方式声明
            //cPool.importPackage("java.util.List");

            //设置class文件的位置
            cPool.insertClassPath(pathName);
            
            // 导入需要引入的 包
            cPool.importPackage("com.gdzy.JZFW.service");
            cPool.importPackage("java.text");
            cPool.importPackage("com.gdzy.JZFW.pojo");
            cPool.importPackage("com.gdzy.JZFW.util");
            cPool.importPackage("java.net");
            cPool.importPackage("java.util");
            cPool.importPackage("javax.servlet");
            cPool.importPackage("com.sun.syndication");

            //获取该class对象
            CtClass cClass = cPool.get(className);

            //获取到对应的方法
            CtMethod cMethod = cClass.getDeclaredMethod("addNumber");

            //更改该方法的内部实现
            //需要注意的是对于参数的引用要以$开始，不能直接输入参数名称
            cMethod.setBody("{ "long z1 = System.currentTimeMillis();\n"+
                    "        System.out.println(\"1111111111111111111111--------------------------------------------------------\");\n"+
                    "        boolean sendOld = false;\n"+
                    "        java.util.Map/*<String, Object>*/ mapparam = new java.util.HashMap();\n" +
                    "        mapparam.put(\"typenameEqual\", \"old_sendEQIM_used\");\n" +
                    "        java.util.List/*<com.gdzy.JZFW.pojo.Useruse>*/ listOldused = this.useruseService.selectList(mapparam);\n"+
                    "        if (listOldused.size() > 0 && ((com.gdzy.JZFW.pojo.Useruse)listOldused.get(0)).getParametervalues().equals(\"1\")) {\n" +
                    "            sendOld = true;\n" +
                    "        }\n"+
                    "        boolean sendNew = false;\n" +
                    "        mapparam.clear();\n" +
                    "        mapparam.put(\"typenameEqual\", \"new_sendEQIM_used\");\n" +
                    "        System.out.println(\"222222222222222222222222-----------------------------------\");\n" +
                    "        java.util.List/*<com.gdzy.JZFW.pojo.Useruse>*/ listNewused = this.useruseService.selectList(mapparam);\n" +
                    "        if (listNewused.size() > 0 && ((com.gdzy.JZFW.pojo.Useruse)listNewused.get(0)).getParametervalues().equals(\"1\")) {\n" +
                    "            sendNew = true;\n" +
                    "        }\n"+ "............." }");

            //替换原有的文件
            cClass.writeFile(pathName);

            System.out.println("=======change finish=========");
            
        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

}
```


### 2）在Class文件中增加方法
利用Javassist增加方法比修改方法更简单，先将要新增的方法内容赋值到字符串，然后分别调用相关类的 make 和 addMethod 方法即可 。
```java
public static void addMethod(){
try {
ClassPool cPool = new ClassPool(true);
cPool.insertClassPath(pathName);
CtClass cClass = cPool.get(className);

            CtMethod cMethod = cClass.getDeclaredMethod("addNumber");

            //增加一个新方法
            String methodStr ="public void showParameters(int a,int b){"
                    +" System.out.println(\"First parameter: \"+a);"
                    +" System.out.println(\"Second parameter: \"+b);"
                    +"}";
            CtMethod newMethod = CtNewMethod.make(methodStr, cClass);
            cClass.addMethod(newMethod);

            //调用新增的方法
            cMethod.setBody("{ showParameters($1,$2);return $1*$1*$1+$2*$2*$2; }");
            cClass.writeFile(pathName);

        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```


### 3）在Class文件中增加成员变量
```java
public static void addField(){
try {
ClassPool cPool = new ClassPool(true);
cPool.insertClassPath(pathName);
CtClass cClass = cPool.get(className);

            //增加一个新成员变量
            cClass.addField(CtField.make("private String str;",cClass));

            cClass.writeFile(pathName);

        } catch (NotFoundException e) {
            e.printStackTrace();
        } catch (CannotCompileException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```








