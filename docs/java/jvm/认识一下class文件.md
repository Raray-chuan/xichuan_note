## 1.class文件概述 
我们可任意打开一个Class文件（使用Hex Editor等工具打开），内容如下（内容是16进制）：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221613969.png)

十六进制转字符串：http://www.bejson.com/convert/ox2str/
进制转换网址（十六进制转十进制）：http://tool.oschina.net/hexconvert/

参考下图去阅读上面的十六进制文档：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221613896.png)

据上述的叙述，我们可以将class的文件组织结构概括成以下面这个表格（其中u表示u4表示4个无符号字节， u2表示2个无符号字节）：

| 类型             | 名称                         | 数量                    |
| -------------- | -------------------------- | --------------------- |
| u4             | magic(魔数)                  | 1                     |
| u2             | minor_version(JDK次版本号)     | 1                     |
| u2             | major_version(JDK主版本号)     | 1                     |
| u2             | constant_pool_count(常量池数量) | 1                     |
| cp_info        | constan_pool(常量表)          | constant_pool_count-1 |
| u2             | access_flags(访问标志)         | 1                     |
| u2             | this_class(类引用)            | 1                     |
| u2             | super_class（父类引用）          | 1                     |
| u2             | interfaces_count(接口数量)     | 1                     |
| u2             | interfaces(接口数组)           | interfaces_count      |
| u2             | fields_count(字段数量)         | 1                     |
| field_info     | fields(字段表)                | fields_count          |
| u2             | methods_count(方法数量)        | 1                     |
| method_info    | methods(方法表)               | methods_count         |
| u2             | attributes_count(属性数量)     | 1                     |
| attribute_info | attributes(属性表)            | attributes_count      |




### 1.1 魔数
所有的由Java编译器编译而成的class文件的前4个字节都是“0xCAFEBABE”。
它的作用在于：当JVM在尝试加载某个文件到内存中来的时候，会首先判断此class文件有没有JVM认为可以接受的“签名”，即JVM会首先读取文件的前4个字节，判断该4个字节是否是“0xCAFEBABE”，如果是，则JVM会认为可以将此文件当作class文件来加载并使用。

### 1.2 版本号
随着Java本身的发展，Java语言特性和JVM虚拟机也会有相应的更新和增强。目前我们能够用到的JDK版本如：1.5，1.6，1.7，还有现如今的1.8及更高的版本。发布新版本的目的在于：在原有的版本上增加新特性和相应的JVM虚拟机的优化。而随着主版本发布的次版本，则是修改相应主版本上出现的bug。我们平时只需要关注主版本就可以了。

主版本号和次版本号在class文件中各占两个字节，副版本号占用第5、 6两个字节，而主版本号则占用第7， 8两个字节。JDK1.0的主版本号为45，以后的每个新主版本都会在原先版本的基础上加1。若现在使用的是JDK1.7编译出来的class文件，则相应的主版本号应该是51，对应的7，8个字节的十六进制的值应该是 0x33。

一个 JVM实例只能支持特定范围内的主版本号 （Mi 至Mj） 和 0 至特定范围内 （0 至 m） 的副版本号。假设一个 Class 文件的格式版本号为 V， 仅当Mi. 0 ≤ v ≤ Mj . m成立时，这个 Class 文件才可以被此 Java 虚拟机支持。不同版本的 Java 虚拟机实现支持的版本号也不同，高版本号的 Java虚拟机实现可以支持低版本号的 Class 文件，反之则不成立。

JVM在加载class文件的时候，会读取出主版本号，然后比较这个class文件的主版本号和JVM本身的版本号，如果JVM本身的版本号 < class文件的版本号，JVM会认为加载不了这个class文件，会抛出我们经常见到的" java.lang.UnsupportedClassVersionError: Bad version number in .classfile " Error 错误；反之，JVM会认为可以加载此class文件，继续加载此class文件。

JDK版本号信息对照表：
| JDK版本  | 16进制版本号     | 十进制版本号 |
| ------ | ----------- | ------ |
| JDK8   | 00 00 00 34 | 52     |
| JDK7   | 00 00 00 33 | 51     |
| JDK6   | 00 00 00 32 | 50     |
| JDK5   | 00 00 00 31 | 49     |
| JDK1.4 | 00 00 00 30 | 48     |
| JDK1.3 | 00 00 00 2F | 47     |
| JDK1.2 | 00 00 00 2E | 46     |
| JDK1.1 | 00 00 00 2D | 45     |


小贴士：

1.有时候我们在运行程序时会抛出这个Error 误："java.lang.UnsupportedClassVersionError: Bad version number in .classfile"。上面已经揭示了出现这个问题的原因，就是在于当前尝试加载class文件的JVM虚拟机的版本 低于class文件的版本。

解决方法：
a). 重新使用当前jvm编译源代码，然后再运行代码；
b). 将当前JVM虚拟机更新到class文件的版本。



2.怎样查看class文件的版本号？可以借助于文本编辑工具，直接查看该文件的7，8个字节的值，确定class文件是什么版本的。当然快捷的方式使用JDK自带的javap工具，如当前有Math.class 文件，进入此文件所在的目录，然后执行 ”javap -v Math“





### 1.3 常量池计数器

常量池是class文件中非常重要的结构，它描述着整个class文件的字面量信息。常量池是由一组constant_pool结构体数组组成的，而数组的大小则由常量池计数器指定。常量池计数器constant_pool_count 的值 =constant_pool表中的成员数+ 1。constant_pool表的索引值只有在大于 0 且小于constant_pool_count时才会被认为是有效的。

**注意事项：**
常量池计数器默认从1开始而不是从0开始：
当constant_pool_count = 1时，常量池中的cp_info个数为0；当constant_pool_count为n时，常量池中的cp_info个数为n-1。

**原因：**
在指定class文件规范的时候，将索引#0项常量空出来是有特殊考虑的，这样当：某些数据在特定的情况下想表达“不引用任何一个常量池项”的意思时，就可以将其引用的常量的索引值设置为#0来表示。



### 1.4 常量池数据区

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221615643.png)



### 1.5 访问标志

访问标志，access_flags 是一种掩码标志，用于表示某个类或者接口的访问权限及基础属性。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221615134.png)



### 1.6 类索引

类索引，this_class的值必须是对constant_pool表中项目的一个有效索引值。constant_pool表在这个索引处的项必须为CONSTANT_Class_info 类型常量，表示这个 Class 文件所定义的类或接口。



### 1.7 父类索引

父类索引，对于类来说，super_class 的值必须为 0 或者是对constant_pool 表中项目的一个有效索引值。

如果它的值不为 0，那 constant_pool 表在这个索引处的项必须为CONSTANT_Class_info 类型常量，表示这个 Class 文件所定义的类的直接父类。当前类的直接父类，以及它所有间接父类的access_flag 中都不能带有ACC_FINAL 标记。对于接口来说，它的Class文件的super_class项的值必须是对constant_pool表中项目的一个有效索引值。constant_pool表在这个索引处的项必须为代表 java.lang.Object 的 CONSTANT_Class_info 类型常量 。

如果 Class 文件的 super_class的值为 0，那这个Class文件只可能是定义的是java.lang.Object类，只有它是唯一没有父类的类。



### 1.8 接口计数器

接口计数器，interfaces_count的值表示当前类或接口的【直接父接口数量】。



### 1.9 接口信息数据区

接口表，interfaces[]数组中的每个成员的值必须是一个对constant_pool表中项目的一个有效索引值， 它的长度为 interfaces_count。每个成员interfaces[i] 必须为CONSTANT_Class_info类型常量，其中 【0 ≤ i <interfaces_count】。在interfaces[]数组中，成员所表示的接口顺序和对应的源代码中给定的接口顺序（从左至右）一样，即interfaces[0]对应的是源代码中最左边的接口。



### 1.10 字段计数器

字段计数器，fields_count的值表示当前 Class 文件 fields[]数组的成员个数。 fields[]数组中每一项都是一个field_info结构的数据项，它用于表示该类或接口声明的【类字段】或者【实例字段】。



### 1.11 字段信息数据区

字段表，fields[]数组中的每个成员都必须是一个fields_info结构的数据项，用于表示当前类或接口中某个字段的完整描述。 fields[]数组描述当前类或接口声明的所有字段，但不包括从父类或父接口继承的部分。



### 1.12 方法计数器

方法计数器， methods_count的值表示当前Class 文件 methods[]数组的成员个数。Methods[]数组中每一项都是一个 method_info 结构的数据项。



### 1.13 方法信息数据区

方法表，methods[] 数组中的每个成员都必须是一个 method_info 结构的数据项，用于表示当前类或接口中某个方法的完整描述。

如果某个method_info 结构的access_flags 项既没有设置 ACC_NATIVE 标志也没有设置ACC_ABSTRACT 标志，那么它所对应的方法体就应当可以被 Java 虚拟机直接从当前类加载，而不需要引用其它类。

method_info结构可以表示类和接口中定义的所有方法，包括【实例方法】、【类方法】、【实例初始化方法】和【类或接口初始化方法】。

methods[]数组只描述【当前类或接口中声明的方法】，【不包括从父类或父接口继承的方法】。



### 1.14 属性计数器

属性计数器，attributes_count的值表示当前 Class 文件attributes表的成员个数。attributes表中每一项都是一个attribute_info 结构的数据项。



### 1.15 属性信息数据区

属性表，attributes 表的每个项的值必须是attribute_info结构。

在Java 7 规范里，Class文件结构中的attributes表的项包括下列定义的属性： InnerClasses、 EnclosingMethod 、 Synthetic 、Signature、SourceFile，SourceDebugExtension、Deprecated、RuntimeVisibleAnnotations 、RuntimeInvisibleAnnotations以及BootstrapMethods属性。

对于支持 Class 文件格式版本号为 49.0 或更高的 Java 虚拟机实现，必须正确识别并读取attributes表中的Signature、RuntimeVisibleAnnotations和RuntimeInvisibleAnnotations属性。对于支持Class文件格式版本号为 51.0 或更高的 Java虚拟机实现，必须正确识别并读取 attributes表中的BootstrapMethods属性。Java 7 规范 要求任一 Java 虚拟机实现可以自动忽略 Class 文件的 attributes表中的若干 （甚至全部） 它不可识别的属性项。任何本规范未定义的属性不能影响Class文件的语义，只能提供附加的描述信息 。









## 2.class常量池理解

### 2.1常量池在class文件的什么位置？

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221616524.png)



### 2.2 常量池的里面是怎么组织的？

cp_info：常量池项
constant_pool_count：常量池计算器

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221616213.png)



### 2.3 常量池项 (cp_info) 的结构是什么？

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221617654.png)

JVM虚拟机规定了不同的tag值和不同类型的字面量对应关系如下：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221617213.png)

所以根据cp_info中的tag 不同的值，可以将cp_info 更细化为以下结构体：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221618948.png)

现在让我们看一下细化了的常量池的结构会是类似下图所示的样子：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221618746.png)



### 2.4 int和float数据类型的常量在常量池中是怎样表示和存储的？

Java语言规范规定了 int类型和Float 类型的数据类型占用 4 个字节的空间。那么存在于class字节码文件中的该类型的常量是如何存储的呢？

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221618331.png)

举例：建下面的类 IntAndFloatTest.java，在这个类中，我们声明了五个变量，但是取值就两种int类型的10 和Float类型的11f。
```java
 package com.dev.jvm;
 public class IntAndFloatTest {

     private final int a = 10; 
     private final int b = 10; 
     //private int c = 20;
     private float c = 11f; 
     private float d = 11f; 
     private float e = 11f;

 }
```

然后用编译器编译成IntAndFloatTest.class字节码文件，我们通过javap -v IntAndFloatTest指令来看一下其常量池中的信息，可以看到虽然我们在代码中写了两次10 和三次11f，但是常量池中，就只有一个常量10 和一个常量11f，如下图所示:

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221619117.png)

从结果上可以看到常量池第#8 个常量池项(cp_info) 就是CONSTANT_Integer_info，值为10；第#23个常量池项(cp_info) 就是CONSTANT_Float_info，值为11f。(常量池中其他的东西先别纠结啦， 我们后面会一一讲解的哦)。

代码中所有用到 int 类型 10 的地方，会使用指向常量池的指针值#8 定位到第#8 个常量池项(cp_info)， 即值为 10的结构体CONSTANT_Integer_info，而用到float类型的11f时，也会指向常量池的指针值#23来定位到第#23个常量池项(cp_info) 即值为11f的结构体CONSTANT_Float_info。
如下图所示：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221620535.png)





### 2.5 long和 double数据类型的常量在常量池中是怎样表示和存储的？

Java语言规范规定了 long 类型和 double类型的数据类型占用8 个字节的空间。那么存在于class字节码文件中的该类型的常量是如何存储的呢？

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221620011.png)

举例：建下面的类 LongAndDoubleTest.java，在这个类中，我们声明了六个变量，但是取值就两种Long 类型的-6076574518398440533L 和Double 类型的10. 1234567890D。
```java
package com.dev.jvm;
  public class LongAndDoubleTest {

    private long a = -6076574518398440533L; 
    private long b = -6076574518398440533L; 
    private long c = -6076574518398440533L; 
    private double d = 10.1234567890D;
    private double e = 10.1234567890D; 
    private double f = 10.1234567890D; 
 }
```

然后用编译器编译成 LongAndDoubleTest.class 字节码文件，我们通过javap -v LongAndDoubleTest指令来看一下其常量池中的信息，可以看到虽然我们在代码中写了三次-6076574518398440533L 和三次10. 1234567890D，但是常量池中，就只有一个常量-6076574518398440533L 和一个常量10. 1234567890D，如下图所示:

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221621415.png)

从结果上可以看到常量池第 #18 个常量池项(cp_info) 就是CONSTANT_Long_info，值为-6076574518398440533L ；第 #26个常量池项(cp_info) 就是CONSTANT_Double_info，值为10. 1234567890D。(常量池中其他的东西先别纠结啦，我们会面会一一讲解的哦)。

代码中所有用到 long 类型-6076574518398440533L 的地方，会使用指向常量池的指针值#18 定位到第 #18 个常量池项(cp_info)， 即值为-6076574518398440533L 的结构体CONSTANT_Long_info，而用到double类型的10. 1234567890D时，也会指向常量池的指针值#26来定位到第 #26 个常量池项(cp_info) 即值为10. 1234567890D的结构体CONSTANT_Double_info。如
下图所示：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221621002.png)



### 2.6 String类型的字符串常量在常量池中是怎样表示和存储的？

对于字符串而言，JVM会将字符串类型的字面量以UTF-8 编码格式存储到在class字节码文件中。这么说可能有点摸不着北，我们先从直观的Java源码中中出现的用双引号"" 括起来的字符串来看，在编译器编译的时候，都会将这些字符串转换成CONSTANT_String_info结构体，然后放置于常量池中。其结构如下所示：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221621927.png)

如上图所示的结构体，CONSTANT_String_info结构体中的string_index的值指向了CONSTANT_Utf8_info结构体，而字符串的utf-8编码数据就在这个结构体（CONSTANT_Utf8_info）之中。如下图所示：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221622231.png)

请看一例，定义一个简单的StringTest.java类，然后在这个类里加一个"JVM原理" 字符串，然后，我们来看看它在class文件中是怎样组织的。
```java
 package com.dev.jvm;

 public class StringTest { 
     private String s1 = "JVM原理"; 
     private String s2 = "JVM原理"; 
     private String s3 = "JVM原理"; 
     private String s4 = "JVM原理";
 }
```

将Java源码编译成StringTest.class文件后，在此文件的目录下执行 javap -v StringTest 命令，会看到如下的常量池信息的轮廓：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221622858.png)

(PS :使用javap -v 指令能看到易于我们阅读的信息，查看真正的字节码文件可以使用HEXWin、NOTEPAD++、UtraEdit 等工具。)

在面的图中，我们可以看到CONSTANT_String_info结构体位于常量池的第#15个索引位置。而存放"Java虚拟机原理" 字符串的 UTF-8编码格式的字节数组被放到CONSTANT_Utf8_info结构体中，该结构体位于常量池的第#16个索引位置。上面的图只是看了个轮廓，让我们再深入地看一下它们的组织吧。请看下图：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221623231.png)

由上图可见：“JVM原理”的UTF-8编码的数组是：4A564D E5 8E 9FE7 90 86，并且存入了CONSTANT_Utf8_info结构体中。



### 2.7 类文件中定义的类名和类中使用到的类在常量池中是怎样被组织和存储的？

JVM会将某个Java 类中所有使用到了的类的完全限定名 以二进制形式的完全限定名 封装成CONSTANT_Class_info结构体中，然后将其放置到常量池里。CONSTANT_Class_info 的tag值为 7。 其结构如下：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221623427.png)

Tips：类的完全限定名和二进制形式的完全限定名

在某个Java源码中，我们会使用很多个类，比如我们定义了一个 ClassTest的类，并把它放到com. dev. jvm 包下，则 ClassTest类的完全限定名为com. dev. jvm. ClassTest，将JVM编译器将类编译成class文件后，此完全限定名在class文件中，是以二进制形式的完全限定名存储的，即它会把完全限定符的". "换成"/" ，即在class文件中存储的 ClassTest类的完全限定名称是"com/dev/jvm/ClassTest"。因为这种形式的完全限定名是放在了class二进制形式的字节码文件中，所以就称之为 二进制形式的完全限定名。

举例，我们定义一个很简单的ClassTest类，来看一下常量池是怎么对类的完全限定名进行存储的。
```java
 package com.jvm;
 import  java.util.Date;
 public class ClassTest { 
    private Date date =new Date();
 }
```

将Java源码编译成ClassTest. class文件后，在此文件的目录下执行 javap -v ClassTest 命令，会看到如下的常量池信息的轮廓：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221624082.png)

如上图所示，在ClassTest. class文件的常量池中，共有 3 个CONSTANT_Class_info结构体，分别表示ClassTest 中用到的Class信息。 我们就看其中一个表示com/jvm. ClassTest的CONSTANT_Class_info 结构体。它在常量池中的位置是#1，它的name_index值为#2，它指向了常量池的第2 个常量池项，如下所示:

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221624466.png)

CONSTANT_Class_info结构体中的name_index值是2，则它指向了常量池的第二个常量池项，第二个常量池项必须是Constant_Utf8_info,它存储了以utf-8编码编码的“com/jvm/ClassTest”字符串。
（其实类文件中定义的类名和类中使用到的类在常量池中是由两个结构体构成的，一个是CONSTANT_Class_info，一个是Constant_Utf8_info，CONSTANT_Class_info常量池项的name_index指向Constant_Utf8_info常量池项）

**注意：**
对于某个类而言，其class文件中至少要有两个CONSTANT_Class_info常量池项，用来表示自己的类信息和其父类信息。(除了java.lang.Object类除外，其他的任何类都会默认继承自java.lang.Object）如果类声明实现了某些接口，那么接口的信息也会生成对应的CONSTANT_Class_info常量池项。

除此之外，如果在类中使用到了其他的类，只有真正使用到了相应的类，JDK编译器才会将类的信息组成CONSTANT_Class_info常量池项放置到常量池中。
```java
package com.dev.jvm; import java.util.Date; 
public  class Other{ 
    private Date date; 
    public Other()  { 
        Date da;
    }
}
```


上述的Other的类，在JDK将其编译成class文件时，常量池中并没有java.util.Date对应的CONSTANT_Class_info常量池项，为什么呢?

在Other类中虽然定义了Date类型的两个变量date、da，但是JDK编译的时候，认为你只是声明了“Ljava/util/Date”类型的变量，并没有实际使用到Ljava/util/Date类。将类信息放置到常量池中的目的，是为了在后续的代码中有可能会反复用到它。很显然，JDK在编译Other类的时候，会解析到Date类有没有用到，发现该类在代码中就没有用到过，所以就认为没有必要将它的信息放置到常量池中了。

将上述的Other类改写一下，仅使用new Date()，如下图所示：
```java
package com.dev.jvm; 
import java.util.Date; 
public  class Other{ 
   public Other()
   {
       new Date();
   }
 }
```

这时候使用javap -v Other ，可以查看到常量池中有表示java/util/Date的常量池项：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221625878.png)


**总结：**
1. 对于某个类或接口而言，其自身、父类和继承或实现的接口的信息会被直接组装成CONSTANT_Class_info常量池项放置到常量池中；
2. 类中或接口中使用到了其他的类，只有在类中实际使用到了该类时，该类的信息才会在常量池中有对应的CONSTANT_Class_info常量池项；
3. 类中或接口中仅仅定义某种类型的变量，JDK只会将变量的类型描述信息以UTF-8字符串组成CONSTANT_Utf8_info常量池项放置到常量池中，上面在类中的private Date date;JDK编译器只会将表示date的数据类型的“Ljava/util/Date”字符串放置到常量池中。





### 2.8 哪些字面量会进入常量池中？
**结论：**
1. final类型的8种基本类型的值会进入常量池。
2. 非final类型（包括static的）的8种基本类型的值，只有double、float、long的值会进入常量池。
3. 常量池中包含的字符串类型字面量（双引号引起来的字符串值）。测试代码：
```java
     public class Test{
        private int int_num = 110;
        private char char_num = 'a'; 
        private short short_num = 120; 
        private float float_num = 130.0f; 
        private double double_num = 140.0; 
        private byte byte_num = 111; 
        private long long_num = 3333L; 
        private long long_delay_num; 
        private boolean boolean_flage = true;

   public void init() { 
       this.long_delay_num = 5555L; 
   }
  }
```

使用javap命令打印的结果如下：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221625628.png)







## 3. class文件中的引用和特殊字符串
### 3.1 符号引用
符号引用以一组符号来描述所引用的目标，符号可以是任何形式的字面量，只要使用时能够无歧义的定位到目标即可。

例如，在Class文件中它以CONSTANT_Class_info、 CONSTANT_Fieldref_info、CONSTANT_Methodref_info等类型的常量出现。

符号引用与虚拟机的内存布局无关， 引用的目标并不一定加载到内存中。

在Java中，一个java类将会编译成一个class文件。在编译时，java类并不知道所引用的类的实际地址，因此只能使用符号引用来代替。

比如org.simple.People类引用了org.simple.Language类，在编译时People类并不知道Language类的实际内存地址，因此只能使用符号org.simple.Language（假设是这个，当然实际中是由类似于CONSTANT_Class_info的常量来表示的）来表示Language类的地址。

各种虚拟机实现的内存布局可能有所不同，但是它们能接受的符号引用都是一致的，因为符号引用的字面量形式明确定义在Java虚拟机规范的Class文件格式中。




### 3.2 直接引用
直接引用可以是：
1. 直接指向目标的指针（比如，指向“类型”【Class对象】、类变量、类方法的直接引用可能是指向方法区的指针）
2. 相对偏移量（比如，指向实例变量、实例方法的直接引用都是偏移量）
3. 一个能间接定位到目标的句柄
     直接引用是和虚拟机的布局相关的， 同一个符号引用在不同的虚拟机实例上翻译出来的直接引用一般不会相同。 如果有了直接引用， 那引用的目标必定已经被加载入内存中了。




### 3.3 引用替换的时机
符号引用替换为直接引用的操作发生在类加载过程(加载 -> 连接(验证、准备、解析) -> 初始化)中的解析阶段，会将符号引用转换(替换)为对应的直接引用，放入运行时常量池中。

后面我们会通过一些问题，来更深入的了解class常量池相关的知识点！！！




### 3.4 特殊字符串字面量
特殊字符串包括三种： 类的全限定名， 字段和方法的描述符， 特殊方法的方法名。 下面我们就分别介绍这三种特殊字符串。

**类的全限定名**
Object类，在源文件中的全限定名是 java.lang.Object。
而class文件中的全限定名是将点号替换成“/” 。 也就是 java/lang/Object 。
源文件中一个类的名字， 在class文件中是用全限定名表述的。



**描述符**
`各类型的描述符`
对于字段的数据类型，其描述符主要有以下几种:
* 基本数据类型（byte、char、double、float、int、long、short、boolean）：除 long 和boolean，其他基本数据类型的描述符用对应单词的大写首字母表示。long 用 J 表示，boolean 用 Z 表示。

* void：描述符是 V。

* 对象类型：描述符用字符L加上对象的全限定名表示，如 String 类型的描述符为Ljava/lang/String。

* 数组类型：每增加一个维度则在对应的字段描述符前增加一个 [ ，如一维数组 int[] 的描述符为 [I，二维数组 String[][] 的描述符为 [[Ljava/lang/String 。

  ![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221626226.png)

  ​



**字段描述符**
字段的描述符就是字段的类型所对应的字符或字符串。
如：
```
int i 中， 字段i的描述符就是 I
Object o中， 字段o的描述符就是 Ljava/lang/Object;
double[][] d中， 字段d的描述符就是 [[D
```



**方法描述符**
方法的描述符比较复杂， 包括所有参数的类型列表和方法返回值。 它的格式是这样的：

```
(参数1类型 参数2类型 参数3类型 ...)返回值类型
```



**注意事项：**
不管是参数的类型还是返回值类型， 都是使用对应字符和对应字符串来表示的， 并且参数列表使用小括号括起来， 并且各个参数类型之间没有空格， 参数列表和返回值类型之间也没有空格。

方法描述符举例说明如下：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206221627095.png)



**特殊方法的方法名**
首先要明确一下， 这里的特殊方法是指的类的构造方法和类型初始化方法。
构造方法就不用多说了， 至于类型的初始化方法， 对应到源码中就是静态初始化块。 也就是说，静态初始化块， 在class文件中是以一个方法表述的， 这个方法同样有方法描述符和方法名，具体如下：

* 类的构造方法的方法名使用字符串 表示
* 静态初始化方法的方法名使用字符串 表示。
* 除了这两种特殊的方法外， 其他普通方法的方法名， 和源文件中的方法名相同。



**总结**

1. 方法和字段的描述符中， 不包括字段名和方法名， 字段描述符中只包括字段类型， 方法描述符中只包括参数列表和返回值类型。
2. 无论method()是静态方法还是实例方法， 它的方法描述符都是相同的。 尽管实例方法除了传递自身定义的参数，还需要额外传递参数this，但是这一点不是由方法描述符来表达的。参数this的传递，是由Java虚拟机实现在调用实例方法所使用的指令中实现的隐式传递。








## 4. 通过javap命令分析java指令
### 4.1 javap命令简述
javap是jdk自带的反解析工具。它的作用就是根据class字节码文件，反解析出当前类对应的code区（汇编指令）、本地变量表、异常表和代码行偏移量映射表、常量池等等信息。

当然这些信息中，有些信息（如本地变量表、指令和代码行偏移量映射表、常量池中方法的参数名称等等）需要在使用javac编译成class文件时，指定参数才能输出，比如，你直接javac xx.java，就不会在生成对应的局部变量表等信息，如果你使用javac -g xx.java就可以生成所有相关信息了。如果你使用的eclipse，则默认情况下，eclipse在编译时会帮你生成局部变量表、指令和代码行偏移量映射表等信息的。

通过反编译生成的汇编代码，我们可以深入的了解java代码的工作机制。比如我们可以查看i++；这行代码实际运行时是先获取变量i的值，然后将这个值加1，最后再将加1后的值赋值给变量i。通过局部变量表，我们可以查看局部变量的作用域范围、所在槽位等信息，甚至可以看到槽位复用等信息。

javap的用法格式：
```bash
javap <options> <classes>
```

其中classes就是你要反编译的class文件。
在命令行中直接输入javap或javap -help可以看到javap的options有如下选项：
```bash
 -help  --help  -?        输出此用法消息
 -version                 版本信息，其实是当前javap所在jdk的版本信息，不是class在哪 个jdk下生成的。
 -v  -verbose             输出附加信息（包括行号、本地变量表，反汇编等详细信息）
 -l                         输出行号和本地变量表
 -public                    仅显示公共类和成员
 -protected               显示受保护的/公共类和成员
 -package                 显示程序包/受保护的/公共类 和成员 (默认)
 -p  -private             显示所有类和成员
 -c                       对代码进行反汇编
 -s                       输出内部类型签名
 -sysinfo                 显示正在处理的类的系统信息 (路径， 大小， 日期， MD5 散 列)
 -constants               显示静态最终常量
 -classpath <path>        指定查找用户类文件的位置
 -bootclasspath <path>    覆盖引导类文件的位置
```

一般常用的是-v -l -c三个选项。
* javap -v classxx，不仅会输出行号、本地变量表信息、反编译汇编代码，还会输出当前类用到的常量池等信息。
* javap -l 会输出行号和本地变量表信息。
* javap -c 会对当前class字节码进行反编译生成汇编代码。

查看汇编代码时，需要知道里面的jvm指令，可以参考官方文档：
https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html
另外通过jclasslib工具也可以看到上面这些信息，而且是可视化的，效果更好一些。



### 4.2 javap测试及内容详解
前面已经介绍过javap输出的内容有哪些，东西比较多，这里主要介绍其中code区(汇编指令)、局部变量表和代码行偏移映射三个部分。

如果需要分析更多的信息，可以使用javap -v进行查看。

另外，为了更方便理解，所有汇编指令不单拎出来讲解，而是在反汇编代码中以注释的方式讲解。
下面写段代码测试一下：

例子1： 分析一下下面的代码反汇编之后结果：

```java
public class TestDate {

    private int count = 0;

    public static void main(String[] args) { 
        TestDate testDate = new TestDate(); 
        testDate.test1();
    }
    
    public void test1(){
        Date date = new Date(); 
        String name1 = "wangerbei"; 
        test2(date，name1); 
        System.out.println(date+name1); 
    }
    public void test2(Date dateP，String name2){ 
        dateP = null;
        name2 = "zhangsan";
    }
    public void test3(){ 
        count++;
    }
    
    public void  test4(){ 
        int a = 0;
        {
            int b = 0;
            b = a+1;
        }
        int c = a+1;
    }
}
```
上面代码通过JAVAC -g 生成class文件，然后通过javap命令对字节码进行反汇编：
```bash
$ javap -c -l TestDate
```
得到下面内容(指令等部分是我参照着官方文档总结的)：
```java
Warning: Binary file TestDate contains com.justest.test.TestDate
Compiled from "TestDate.java"
public class com.justest.test.TestDate { 
  //默认的构造方法，在构造方法执行时主要完成一些初始化操作，包括一些成员变量的初始化赋值 等操作
  public com.justest.test.TestDate();
    Code:
       0: aload_0 //从本地变量表中加载索引为0的变量的值，也即this的引用，压入栈 
       1: invokespecial #10  //出栈，调用java/lang/Object."<init>":()V 初始化 对象，就是this指定的对象的<init>方法完成初始化
       4: aload_0  // 4到6表示，调用this.count = 0，也即为count复制为0。这里this 引用入栈
        5: iconst_0 //将常量0，压入到操作数栈
       6: putfield     //出栈前面压入的两个值（this引用，常量值0）， 将0取出，并赋 值给count
       9: return 
//指令与代码行数的偏移对应关系，每一行第一个数字对应代码行数，第二个数字对应前面code中指 令前面的数字
    LineNumberTable:
      line 5: 0
      line 7: 4
      line 5: 9 
    //局部变量表，start+length表示这个变量在字节码中的生命周期起始和结束的偏移位置 （this生命周期从头0到结尾10），slot就是这个变量在局部变量表中的槽位（槽位可复用）， name就是变量名称，Signatur局部变量类型描述
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
         0      10     0  this   Lcom/justest/test/TestDate;

  public static void main(java.lang.String[]);
    Code:
// new指令，创建一个class com/justest/test/TestDate对象，new指令并不能完全创建一 个对象，对象只有在初，只有在调用初始化方法完成后（也就是调用了invokespecial指令之后）， 对象才创建成功，
       0: new  //创建对象，并将对象引用压入栈
       3: dup //将操作数栈定的数据复制一份，并压入栈，此时栈中有两个引用值
       4: invokespecial #20  //pop出栈引用值，调用其构造函数，完成对象的初始化 
       7: astore_1 //pop出栈引用值，将其（引用）赋值给局部变量表中的变量testDate 
       8: aload_1  //将testDate的引用值压入栈，因为testDate.test1();调用了 testDate，这里使用aload_1从局部变量表中获得对应的变量testDate的值并压入操作数栈 
       9: invokevirtual #21 // Method test1:()V  引用出栈，调用testDate的 test1()方法
      12: return //整个main方法结束返回
    LineNumberTable:
      line 10: 0
      line 11: 8
      line 12: 12
    //局部变量表，testDate只有在创建完成并赋值后，才开始声明周期 
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
         0      13     0  args   [Ljava/lang/String;
         8       5     1 testDate   Lcom/justest/test/TestDate;

  public void test1();
    Code:
       0: new           #27                 // 0到7创建Date对象，并赋值给date 变量
       3: dup
       4: invokespecial #29                 // Method java/util/Date." <init>":()V
       7: astore_1
       8: ldc           #30     // String wangerbei，将常量“wangerbei”压入栈 
      10: astore_2  //将栈中的“wangerbei”pop出，赋值给name1
      11: aload_0 //11到14，对应test2(date，name1);默认前面加this.
      12: aload_1 //从局部变量表中取出date变量
      13: aload_2 //取出name1变量
      14: invokevirtual #32                 // Method test2: 
(Ljava/util/Date;Ljava/lang/String;)V  调用test2方法
  // 17到38对应System.out.println(date+name1);
   17: getstatic     #36                 // Field 
java/lang/System.out:Ljava/io/PrintStream; 
  //20到35是jvm中的优化手段，多个字符串变量相加，不会两两创建一个字符串对象，而使用 StringBuilder来创建一个对象
      20: new           #42                 // class 
java/lang/StringBuilder
      23: dup
      24: invokespecial #44                 // Method 
java/lang/StringBuilder."<init>":()V
      27: aload_1
      28: invokevirtual #45                 // Method 
java/lang/StringBuilder.append: 
(Ljava/lang/Object;)Ljava/lang/StringBuilder;
      31: aload_2
      32: invokevirtual #49                 // Method 
java/lang/StringBuilder.append: 
(Ljava/lang/String;)Ljava/lang/StringBuilder;
      35: invokevirtual #52                 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      38: invokevirtual #56                 // Method 
java/io/PrintStream.println:(Ljava/lang/String;)V  invokevirtual指令表示基于 类调用方法
      41: return
    LineNumberTable:
      line 15: 0
      line 16: 8
      line 17: 11
      line 18: 17
      line 19: 41
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      42     0  this   Lcom/justest/test/TestDate;
             8      34     1  date   Ljava/util/Date;
            11      31     2 name1   Ljava/lang/String;

  public void test2(java.util.Date， java.lang.String);
    Code:
       0: aconst_null //将一个null值压入栈
       1: astore_1 //将null赋值给dateP
       2: ldc           #66       // String zhangsan 从常量池中取出字符 串“zhangsan”压入栈中
       4: astore_2 //将字符串赋值给name2
       5: return
    LineNumberTable:
      line 22: 0
      line 23: 2
      line 24: 5
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0       6     0  this   Lcom/justest/test/TestDate;
             0       6     1 dateP   Ljava/util/Date;
             0       6     2 name2   Ljava/lang/String;

  public void test3();
    Code:
       0: aload_0 //取出this，压入栈
       1: dup   //复制操作数栈栈顶的值，并压入栈，此时有两个this对象引用值在操作数组 栈
       2: getfield #12// Field count:I this出栈，并获取其count字段，然后压入栈， 此时栈中有一个this和一个count的值
       5: iconst_1 //取出一个int常量1，压入操作数栈
       6: iadd  // 从栈中取出count和1，将count值和1相加，结果入栈
       7: putfield      #12 // Field count:I  一次弹出两个，第一个弹出的是上一步 计算值，第二个弹出的this，将值赋值给this的count字段
      10: return
    LineNumberTable:
      line 27: 0
      line 28: 10
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      11     0  this   Lcom/justest/test/TestDate;
 public void test4();
    Code:
       0: iconst_0
       1: istore_1
       2: iconst_0
       3: istore_2
       4: iload_1
       5: iconst_1
       6: iadd
       7: istore_2
       8: iload_1
       9: iconst_1
      10: iadd
      11: istore_2
      12: return
    LineNumberTable:
      line 33: 0
      line 35: 2
      line 36: 4
      line 38: 8
      line 39: 12 
    //看下面，b和c的槽位slot一样，这是因为b的作用域就在方法块中，方法块结束，局部变量表 中的槽位就被释放，后面的变量就可以复用这个槽位
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      13     0  this   Lcom/justest/test/TestDate;
             2      11     1     a   I
             4       4     2     b   I
            12       1     2     c   I
}
```



例子2： 下面一个例子
先有一个User类：

```java
public class User { 
    private String name; 
    private int age;
    
    public String getName() { 
        return name;
    }
    
    public void setName(String name) { 
        this.name = name;
        }
    
    public int getAge() { 
        return age;
    }
    
    public void setAge(int age) { 
        this.age = age;
    }
}
```
然后写一个操作User对象的测试类：
```java
public class TestUser {

    private int count;

    public void test(int a){ 
        count = count + a;
    }
    
    public User initUser(int age，String name){ 
        User user = new User(); 
        user.setAge(age); 
        user.setName(name);
        return user;
    }
    
    public void changeUser(User user，String newName){ 
        user.setName(newName);
    }
}
```

先javac -g 编译成class文件。
然后对TestUser类进行反汇编：
`$ javap -c -l TestUser`
得到反汇编结果如下：
```java
Warning: Binary file TestUser contains com.justest.test.TestUser
Compiled from "TestUser.java"
public class com.justest.test.TestUser {

    //默认的构造函数
    public com.justest.test.TestUser();
      Code:
       0: aload_0
       1: invokespecial #10                 // Method java/lang/Object." <init>":()V
       4: return
    LineNumberTable: 
      line 3: 0
    LocalVariableTable:
     Start  Length  Slot  Name   Signature
      0       5     0  this   Lcom/justest/test/TestUser;
    public void test(int);     
    Code:
       0: aload_0 //取this对应的对应引用值，压入操作数栈
       1: dup //复制栈顶的数据，压入栈，此时栈中有两个值，都是this对象引用 
       2: getfield      #18 // 引用出栈，通过引用获得对应count的值，并压入栈 
       5: iload_1 //从局部变量表中取得a的值，压入栈中
       6: iadd //弹出栈中的count值和a的值，进行加操作，并将结果压入栈
       7: putfield      #18 // 经过上一步操作后，栈中有两个值，栈顶为上一步操作结 果，栈顶下面是this引用，这一步putfield指令，用于将栈顶的值赋值给引用对象的count字段 
      10: return //return void
     LineNumberTable: 
      line 8: 0 
      line 9: 10     
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      11     0  this   Lcom/justest/test/TestUser; 
             0      11     1     a   I           
  public com.justest.test.User initUser(int， java.lang.String);                  
      Code:
       0: new           #23   // class com/justest/test/User 创建User对象， 并将引用压入栈
       3: dup //复制栈顶值，再次压入栈，栈中有两个User对象的地址引用
       4: invokespecial #25   // Method com/justest/test/User."<init>":()V 调用user对象初始化
       7: astore_3 //从栈中pop出User对象的引用值，并赋值给局部变量表中user变量 
       8: aload_3 //从局部变量表中获得user的值，也就是User对象的地址引用，压入栈中 
       9: iload_1 //从局部变量表中获得a的值，并压入栈中，注意aload和iload的区别，一 个取值是对象引用，一个是取int类型数据
      10: invokevirtual #26  // Method com/justest/test/User.setAge:(I)V 
操作数栈pop出两个值，一个是User对象引用，一个是a的值，调用setAge方法，并将a的值传给这 
个方法，setAge操作的就是堆中对象的字段了
      13: aload_3 //同7，压入栈
      14: aload_2 //从局部变量表取出name，压入栈
      15: invokevirtual #29  // MethodUser.setName:(Ljava/lang/String;)V 
操作数栈pop出两个值，一个是User对象引用，一个是name的值，调用setName方法，并将a的值传 给这个方法，setName操作的就是堆中对象的字段了
      18: aload_3 //从局部变量取出User引用，压入栈
      19: areturn //areturn指令用于返回一个对象的引用，也就是上一步中User的引用，这 个返回值将会被压入调用当前方法的那个方法的栈中objectref is popped from the operand stack of the current frame ([§2.6] 
(https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-2.html#jvms-2.6)) and pushed onto the operand stack of the frame of the invoker                            
    LineNumberTable: 
      line 12: 0 
      line 13: 8 
      line 14: 13 
      line 15: 18
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0      20     0  this   Lcom/justest/test/TestUser; 
             0      20     1   age   I
             0      20     2  name   Ljava/lang/String;
             8      12     3  user   Lcom/justest/test/User;
  public void changeUser(com.justest.test.User， java.lang.String);
    Code:
       0: aload_1 //局部变量表中取出this，也即TestUser对象引用，压入栈
       1: aload_2 //局部变量表中取出newName，压入栈
       2: invokevirtual #29 // Method User.setName:(Ljava/lang/String;)V pop出栈newName值和TestUser引用，调用其setName方法，并将newName的值传给这个方法 
       5: return
    LineNumberTable: 
      line 19: 0 
      line 20: 5       
     LocalVariableTable:
      Start  Length  Slot  Name   Signature
             0       6     0  this   Lcom/justest/test/TestUser; 
             0       6     1  user   Lcom/justest/test/User;
             0       6     2 newName   Ljava/lang/String;      
 public static void main(java.lang.String[]);      
    Code:
       0: new      #1 // class com/justest/test/TestUser 创建TestUser对象， 将引用压入栈
       3: dup //复制引用，压入栈
       4: invokespecial #43   // Method "<init>":()V 引用值出栈，调用构造方法， 对象初始化
       7: astore_1 //引用值出栈，赋值给局部变量表中变量tu
       8: aload_1 //取出tu值，压入栈
       9: bipush    10 //将int值10压入栈
      11: ldc           #44   // String wangerbei 从常量池中取出“wangerbei” 压入栈
      13: invokevirtual #46    // Method 
initUser(ILjava/lang/String;)Lcom/justest/test/User; 调用tu的initUser方法，并 返回User对象 ，出栈三个值：tu引用，10和“wangerbei”，并且initUser方法的返回值，即 User的引用，也会被压入栈中，参考前面initUser中的areturn指令
      16: astore_2 //User引用出栈，赋值给user变量
      17: aload_1 //取出tu值，压入栈
      18: aload_2 //取出user值，压入栈
      19: ldc           #48     // String lisi 从常量池中取出“lisi”压入栈 
      21: invokevirtual #50     // Method changeUser: 
(Lcom/justest/test/User;Ljava/lang/String;)V 调用tu的changeUser方法，并将user 引用和lisi传给这个方法
      24: return //return void

 LineNumberTable: 
      line 23: 0 
      line 24: 8 
      line 25: 17 
      line 26: 24
 LocalVariableTable:     
      Start  Length  Slot  Name   Signature
             0      25     0  args   [Ljava/lang/String;
             8      17     1    tu   Lcom/justest/test/TestUser; 
            17       8     2  user   Lcom/justest/test/User;                    
}
```



### 4.3 总结

1、通过javap命令可以查看一个java类反汇编、常量池、变量表、指令代码行号表等等信息。

2、平常，我们比较关注的是java类中每个方法的反汇编中的指令操作过程，这些指令都是顺序执行的，可以参考官方文档查看每个指令的含义，很简单：
https://docs.oracle.com/javase/specs/jvms/se7/html/jvms-6.html#jvms-6.5.areturn

3、通过对前面两个例子代码反汇编中各个指令操作的分析，可以发现，一个方法的执行通常会涉及下面几块内存的操作：
（1）java栈：局部变量表、操作数栈。这些操作基本上都值操作。
（2）java堆：通过对象的地址引用去操作。
（3）常量池。
（4）其他如帧数据区、 方法区（jdk1.8之前，常量池也在方法区）等部分，测试中没有显示出来，这里说明一下。

在做值相关操作时：
一个指令，可以从局部变量表、常量池、堆中对象、方法调用、系统调用中等取得数据，这些数据（可能是指，可能是对象的引用）被压入操作数栈。

一个指令，也可以从操作数数栈中取出一到多个值（pop多次），完成赋值、加减乘除、方法传参、系
统调用等等操作。


