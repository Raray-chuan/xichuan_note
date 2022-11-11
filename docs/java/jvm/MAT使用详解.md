## 1. Mat的作用
   MAT是Memory Analyzer tool的缩写，是一种快速，功能丰富的Java堆分析工具，能帮助你查找内存泄漏和减少内存消耗。很多情况下，我们需要处理测试提供的hprof文件，分析内存相关问题，那么MAT也绝对是不二之选。 Eclipse可以下载插件结合使用，也可以作为一个独立分析工具使用；


## 2. Mat的使用步骤
打开Mat后File>OpenHeapDump打开一个对应的dump文件后，此时对应的打开后结果如图所示：
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111401637.png)
默认情况下打开该dump文件后，直接展示的就是一个Overview(概览)的页签，其中可以看到上面标注为（1,2）的地方所对应的图标与Overview页签中所对应的部分图标是相似的；如果你不小心关掉了Overview的页签，那么直接单击当前dump页签第一行导航栏的第一个 I字的图标即可，同理，如果此时想要打开Histogram，那么在不打开Overview的情况下，直接点击第一行导航栏的第二个图标即可；......


## 3. Overview下功能解释
   Overview页签下分别包含了：Actions，Reports，Step By Step 三大块功能；每一块功能下的子集所对应的作用分别是：
```
1.Actions：
   Histogram 列出每个类所对应的对象个数，以及所占用的内存大小；
   Dominator Tree 以占用总内存的百分比的方式来列举出所有的实例对象，注意这个地方是直接列举出的对应的对象而不是类，这个视图是用来发现大内存对象的
   Top Consumers：按照类和包分组的方式展示出占用内存最大的一个对象
   Duplicate Classes：检测由多个类加载器所加载的类信息（用来查找重复的类）
2.Reports：
   Leak Suspects：通过MAT自动分析当前内存泄露的主要原因
   Top Components：Top组件，列出大于总堆1%的组件的报告
3.Step By Step：
   Component Report：组件报告,分析属于公共根包或类加载器的对象；
上述所有被标注加粗的部分，是内存溢出dump分析时较为常用的功能点也是下面主要讲解的内容。
```

### 3.1 Histogram
通过Histogram 列出每个类所对应的对象个数，以及所占用的内存大小；
此处选中一个ClassName单击后，通过左上角Inspector可以看到当前类的回收情况，内存地址，等
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111403976.png)
补充解释：
```
字段一：表示当前类所对应的对象数量
字段二：Shallow Size是对象本身占据的内存的大小，不包含其引用的对象。对于常规对象（非数组）的Shallow Size由其成员变量的数量和类型来定，而数组的ShallowSize由数组类型和数组长度来决定，它为数组元素大小的总和；
字段三：Retained Size=当前对象大小+当前对象可直接或间接引用到的对象的大小总和。(间接引用的含义：A->B->C,C就是间接引用) ，并且排除被GC Roots直接或者间接引用的对象；
```
关于红框内的Statics，Attributes，Classhierarchy,Value则分别表示当前类的静态变量，属性，当前类的层次结构图，以及当前类所对应的值Value；

注意：当前Histogram的列属性：ClassName，Objects，ShallowHeap，RetainedHeap这几个列属性下面都是有提供一个输入框，通过该输入框可以进行相关类的检索，比如：在ClassName下输入一个正则.quark.那么则获取到所有包路径为quark的类信息；


### 3.2 Dominator Tree
以占用总内存的百分比的方式来列举出所有的实例对象，可以用来发现大内存对象；
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111404024.png)
如上图所示：可以看到ConcurrentHashMap@0x60191cfa8这个对象占据了98.92%的堆大小，所以基本就可以断定，当前项目之所以会down机的主要原因是，ConcurrentHashMap溢出所导致的问题；

那么当我们需要查看，当前该ConcurrentHashMap@0x60191cfa8对象都引用了那些数据，以及当前该对象是被那几个对象所引用的，如何查看？

鼠标在当前所要查看的对象右键，点击List Objects可以看到分别提供了：with outgoing references（查看当前该对象的所有的引用信息） 和 with incoming references（查看当前该对象是被那几个对象所引用的） ；
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111404312.png)


### 3.3 Leak Suspects
通过MAT自动分析当前内存泄露的主要原因
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111405744.png)
可以看到，当前MAT所给出内存泄露的主要原因是：当前实例java.util.concurrent.ConcurrentHashMap被加载自system class loader，共占用了 98.92%的堆内存，这个实例被引用自org.apache.ignite.internal.processors.cache.binary.CacheObjectBinaryProcessorImpl并且这个CacheObjectBinaryProcessorImpl这个对象是加载自LaunchedURLClassLoader这个类加载器；

并且还给出了所对应的主要关键词是：
```
java.util.concurrent.ConcurrentHashMap$Node[]
java.util.concurrent.ConcurrentHashMap
org.springframework.boot.loader.LaunchedURLClassLoader @ 0x6000a6860
```
基本上可以说是很详细了，一语中的，如果想要查看明细，可以直接点击detail，里面有更详细的说明，如下图所示：
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111405418.png)
上图分别说明了到该内存泄漏的对象的最快路径，也就是列出了当前ConcurrentHashMapConcurrentHashMap@0x60191cfa8这个对象所对应的被引用关系：可以看到当前引起内存泄漏的ConcurrentHashMap被CacheObjectBinaryProcessorImpl@0x60191cea8这个对象的metadataLocCache这个属性所引用，而CacheObjectBinaryProcessorImpl这个对象又被GridKernalContextImpl @ 0x601821bf8这个对象的cacheObjeProc这个属性所引用，以此递推；
除此之外，还有以下三个被隐藏的信息，点击即可查看明细：

Accumulated Objects in Dominator Tree （主控树中的累积对象），Accumulated Objects by Class in Dominator Tree（主控树中的按类累积对象 ，All Accumulated Objects by Class （按类列出所有的累积对象）


### 3.4 Overview功能说明结尾
通过上述的解释应该对当前Overview下的功能使用已经有了一个大概的了解，需要注意的是，Histogram 以及Dominator Tree时所主要提及的Shallow Size以及Retained Size以及在所列出的对象上右键查看引用关系，GCROOTS，以及左上角所展示的属性明细等功能 是适用于所有的功能模块的，后续不再赘述；




## 4. 一级导航栏功能说明
查看完上述关于Overview中的功能说明后，此处再来看一下Overview中不包含的一些功能

### 4.1 Thread_Overview
如下图所示，点击一级导航栏的第5个图标，可以用来查看当前进程dump时的所有线程的堆栈信息，通过分析下面所对应的堆栈信息，可以很快速的定位到对应的线程所执行的方法等层级关系，以此来定位对应的异常问题；
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111406620.png)

### 4.2 OQL
用于查询Java堆的类SQL查询语言
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111407088.png)

### 4.3 Heap Dump Overview
点击一级导航栏的第6个图标的下拉框下的 Heap Dump Overview，可以查看全局的内存占用信息
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111407783.png)


### 4.4 Find Object by address
查看指定内存地址所对应的对象信息；
![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111408768.png)




## 5. 常见溢出的几个场景
```
1、线程所引用对象溢出
2、静态属性对象溢出
```
线程栈所引用对象溢出的场景，如下：
   ![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202211111408424.png)
Mat各功能内还有很多小的子功能，使用过程中可逐步尝试，此处不再赘述




