## 1. ThreadLocal简介
多线程访问同一个共享变量的时候容易出现并发问题，特别是多个线程对一个变量进行写入的时候，为了保证线程安全，一般使用者在访问共享变量的时候需要进行额外的同步措施才能保证线程安全性。ThreadLocal是除了加锁这种同步方式之外的一种保证一种规避多线程访问出现线程不安全的方法，当我们在创建一个变量后，如果每个线程对其进行访问的时候访问的都是线程自己的变量这样就不会存在线程不安全问题。
　　ThreadLocal是JDK包提供的，它提供线程本地变量，如果创建一乐ThreadLocal变量，那么访问这个变量的每个线程都会有这个变量的一个副本，在实际多线程操作的时候，操作的是自己本地内存中的变量，从而规避了线程安全问题，如下图所示

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201150219.png)


ThreadLocal类中提供了几个方法：
```java
1.public T get() { }
2.public void set(T value) { }
3.public void remove() { }
4.protected T initialValue(){ }
```
ThreadLocal是解决线程安全问题一个很好的思路，它通过为每个线程提供一个独立的变量副本解决了变量并发访问的冲突问题。在很多情况下，ThreadLocal比直接使用synchronized同步机制解决线程安全问题更简单，更方便，且结果程序拥有更高的并发性。





## 2. ThreadLocal的应用场景？
在Java的多线程编程中，为保证多个线程对共享变量的安全访问，通常会使用synchronized来保证同一时刻只有一个线程对共享变量进行操作。这种情况下可以将类变量放到ThreadLocal类型的对象中，使变量在每个线程中都有独立拷贝，不会出现一个线程读取变量时而被另一个线程修改的现象。最常见的ThreadLocal使用场景为用来解决数据库连接、Session管理等。在下面会例举几个场景。






## 3. ThreadLocal简单使用
下面的例子中，开启两个线程，在每个线程内部设置了本地变量的值，然后调用print方法打印当前本地变量的值。如果在打印之后调用本地变量的remove方法会删除本地内存中的变量，代码如下所示
```java
public class ThreadLocalTest {

    static ThreadLocal<String> localVar = new ThreadLocal<>();

    static void print(String str) {
        //打印当前线程中本地内存中本地变量的值
        System.out.println(str + " :" + localVar.get());
        //清除本地内存中的本地变量
        localVar.remove();
    }

    public static void main(String[] args) {
        Thread t1  = new Thread(new Runnable() {
            @Override
            public void run() {
                //设置线程1中本地变量的值
                localVar.set("localVar1");
                //调用打印方法
                print("thread1");
                //打印本地变量
                System.out.println("after remove : " + localVar.get());
            }
        });

        Thread t2  = new Thread(new Runnable() {
            @Override
            public void run() {
                //设置线程1中本地变量的值
                localVar.set("localVar2");
                //调用打印方法
                print("thread2");
                //打印本地变量
                System.out.println("after remove : " + localVar.get());
            }
        });

        t1.start();
        t2.start();
    }
}
```
结果
```java
thread1 :localVar1
thread2 :localVar2
after remove : null
after remove : null
```





## 4. ThreadLocal的实现原理
　　下面是ThreadLocal的类图结构，从图中可知：Thread类中有两个变量threadLocals和inheritableThreadLocals，二者都是ThreadLocal内部类ThreadLocalMap类型的变量，我们通过查看内部内ThreadLocalMap可以发现实际上它类似于一个HashMap。在默认情况下，每个线程中的这两个变量都为null

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201152439.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201153099.png)

只有当线程第一次调用ThreadLocal的set或者get方法的时候才会创建他们（后面我们会查看这两个方法的源码）。除此之外，和我所想的不同的是，每个线程的本地变量不是存放在ThreadLocal实例中，而是放在调用线程的ThreadLocals变量里面（前面也说过，该变量是Thread类的变量）。也就是说，ThreadLocal类型的本地变量是存放在具体的线程空间上，其本身相当于一个装载本地变量的工具壳，通过set方法将value添加到调用线程的threadLocals中，当调用线程调用get方法时候能够从它的threadLocals中取出变量。如果调用线程一直不终止，那么这个本地变量将会一直存放在他的threadLocals中，所以不使用本地变量的时候需要调用remove方法将threadLocals中删除不用的本地变量。下面我们通过查看ThreadLocal的set、get以及remove方法来查看ThreadLocal具体实怎样工作的

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201153301.png)




**1、set方法源码**
```java
public void set(T value) {
    //(1)获取当前线程（调用者线程）
    Thread t = Thread.currentThread();
    //(2)以当前线程作为key值，去查找对应的线程变量，找到对应的map
    ThreadLocalMap map = getMap(t);
    //(3)如果map不为null，就直接添加本地变量，key为当前线程，值为添加的本地变量值
    if (map != null)
        map.set(this, value);
    //(4)如果map为null，说明首次添加，需要首先创建出对应的map
    else
        createMap(t, value);
}
```
在上面的代码中，(2)处调用getMap方法获得当前线程对应的threadLocals(参照上面的图示和文字说明)，该方法代码如下
```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals; //获取线程自己的变量threadLocals，并绑定到当前调用线程的成员变量threadLocals上
}
```

如果调用getMap方法返回值不为null，就直接将value值设置到threadLocals中（key为当前线程引用，值为本地变量）；如果getMap方法返回null说明是第一次调用set方法（前面说到过，threadLocals默认值为null，只有调用set方法的时候才会创建map），这个时候就需要调用createMap方法创建threadLocals，该方法如下所示
```java
 void createMap(Thread t, T firstValue) {
     t.threadLocals = new ThreadLocalMap(this, firstValue);
 }
```
createMap方法不仅创建了threadLocals，同时也将要添加的本地变量值添加到了threadLocals中。


**2、get方法源码**
　　在get方法的实现中，首先获取当前调用者线程，如果当前线程的threadLocals不为null，就直接返回当前线程绑定的本地变量值，否则执行setInitialValue方法初始化threadLocals变量。在setInitialValue方法中，类似于set方法的实现，都是判断当前线程的threadLocals变量是否为null，是则添加本地变量（这个时候由于是初始化，所以添加的值为null），否则创建threadLocals变量，同样添加的值为null。
```java
public T get() {
    //(1)获取当前线程
    Thread t = Thread.currentThread();
    //(2)获取当前线程的threadLocals变量
    ThreadLocalMap map = getMap(t);
    //(3)如果threadLocals变量不为null，就可以在map中查找到本地变量的值
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //(4)执行到此处，threadLocals为null，调用该更改初始化当前线程的threadLocals变量
    return setInitialValue();
}

private T setInitialValue() {
    //protected T initialValue() {return null;}
    T value = initialValue();
    //获取当前线程
    Thread t = Thread.currentThread();
    //以当前线程作为key值，去查找对应的线程变量，找到对应的map
    ThreadLocalMap map = getMap(t);
    //如果map不为null，就直接添加本地变量，key为当前线程，值为添加的本地变量值
    if (map != null)
        map.set(this, value);
    //如果map为null，说明首次添加，需要首先创建出对应的map
    else
        createMap(t, value);
    return value;
}
```


**3、remove方法的实现**
remove方法判断该当前线程对应的threadLocals变量是否为null，不为null就直接删除当前线程中指定的threadLocals变量
```java
public void remove() {
    //获取当前线程绑定的threadLocals
     ThreadLocalMap m = getMap(Thread.currentThread());
     //如果map不为null，就移除当前线程中指定ThreadLocal实例的本地变量
     if (m != null)
         m.remove(this);
 }
```


**4、如下图所示**：每个线程内部有一个名为threadLocals的成员变量，该变量的类型为ThreadLocal.ThreadLocalMap类型（类似于一个HashMap），其中的key为当前定义的ThreadLocal变量的this引用，value为我们使用set方法设置的值。每个线程的本地变量存放在自己的本地内存变量threadLocals中，如果当前线程一直不消亡，那么这些本地变量就会一直存在（所以可能会导致内存溢出），因此使用完毕需要将其remove掉。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201154553.png)





## 5. ThreadLocal使用有哪些坑及注意事项
我经常在网上看到骇人听闻的标题，ThreadLocal导致内存泄漏，这通常让一些刚开始对ThreadLocal理解不透彻的开发者，不敢贸然使用。越不用，越陌生。这样就让我们错失了更好的实现方案，所以敢于引入新技术，敢于踩坑，才能不断进步。
我们来看下为什么说ThreadLocal会引起内存泄漏，什么场景下会导致内存泄漏？

先回顾下什么叫内存泄漏，对应的什么叫内存溢出
①Memory overflow:内存溢出，没有足够的内存提供申请者使用。
②Memory leak:内存泄漏，程序申请内存后，无法释放已申请的内存空间，内存泄漏的堆积终将导致内存溢出。
显然是TreadLocal在不规范使用的情况下导致了内存没有释放。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201155076.png)

红框里我们看到了一个特殊的类WeakReference，同样这个类，应用开发者也同样很少使用，这里简单介绍下吧

| 类型   | 回收时间                 | 应用场景                                    |
| ---- | -------------------- | --------------------------------------- |
| 强引用  | 一直存活，除非GC Roots不可达   | 所有程序的场景，基本对象，自定义对象等                     |
| 软引用  | 内存不足时会被回收            | 一般用在对内存非常敏感的资源上，用作缓存的场景比较多，例如：网页缓存、图片缓存 |
| 弱引用  | 只能存活到下一次GC前          | 生命周期很短的对象，例如ThreadLocal中的Key。           |
| 虚引用  | 随时会被回收， 创建了可能很快就会被回收 | 可能被JVM团队内部用来跟踪JVM的垃圾回收活动                |


既然WeakReference在下一次gc即将被回收，那么我们的程序为什么没有出问题呢？

①所以我们测试下弱引用的回收机制：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201156855.png)

这一种存在强引用不会被回收。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201157674.png)




这里没有强引用将会被回收。
上面演示了弱引用的回收情况，下面我们看下ThreadLocal的弱引用回收情况。

②ThreadLocal的弱引用回收情况

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201158995.png)


如上图所示，我们在作为key的ThreadLocal对象没有外部强引用，下一次gc必将产生key值为null的数据，若线程没有及时结束必然出现，一条强引用链
Threadref–>Thread–>ThreadLocalMap–>Entry，所以这将导致内存泄漏。
下面我们模拟复现ThreadLocal导致内存泄漏：
1.为了效果更佳明显我们将我们的treadlocals的存储值value设置为1万字符串的列表：
```java
class ThreadLocalMemory {
    // Thread local variable containing each thread's ID
    public ThreadLocal<List<Object>> threadId = new ThreadLocal<List<Object>>() {
        @Override
        protected List<Object> initialValue() {
            List<Object> list = new ArrayList<Object>();
            for (int i = 0; i < 10000; i++) {
                list.add(String.valueOf(i));
            }
            return list;
        }
    };
    // Returns the current thread's unique ID, assigning it if necessary
    public List<Object> get() {
        return threadId.get();
    }
    // remove currentid
    public void remove() {
        threadId.remove();
    }
}
```

测试代码如下：
```java
public static void main(String[] args)
            throws InterruptedException {

        //  为了复现key被回收的场景，我们使用临时变量
        ThreadLocalMemory memeory = new ThreadLocalMemory();
    
        // 调用
        incrementSameThreadId(memeory);
    
        System.out.println("GC前：key:" + memeory.threadId);
        System.out.println("GC前：value-size:" + refelectThreadLocals(Thread.currentThread()));
    
        // 设置为null，调用gc并不一定触发垃圾回收，但是可以通过java提供的一些工具进行手工触发gc回收。
        memeory.threadId = null;
        System.gc();
    
        System.out.println("GC后：key:" + memeory.threadId);
        System.out.println("GC后：value-size:" + refelectThreadLocals(Thread.currentThread()));
    
        // 模拟线程一直运行
        while (true) {
        }
    }
```

此时我们如何知道内存中存在memory leak呢？
我们可以借助jdk提供的一些命令dump当前堆内存，命令如下：
`jmap -dump:live,format=b,file=heap.bin <pid>`



然后我们借助MAT可视化分析工具，来查看对内存，分析对象实例的存活状态：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201220181.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201221157.png)

首先打开我们工具提示我们的内存泄漏分析：

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201221347.png)


   这里我们可以确定的是ThreadLocalMap实例的Entry.value是没有被回收的。
    最后我们要确定Entry.key是否还在？打开Dominator Tree，搜索我们的ThreadLocalMemory，发现并没有存活的实例。




![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201222137.png)

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206201223954.png)

以上我们复现了ThreadLocal不正当使用，引起的内存泄漏。

所以我们总结了使用ThreadLocal时会发生内存泄漏的前提条件：
①ThreadLocal引用被设置为null，且后面没有set，get,remove操作。
②线程一直运行，不停止。（线程池）
③触发了垃圾回收。（Minor GC或Full GC）

我们看到ThreadLocal出现内存泄漏条件还是很苛刻的，所以我们只要破坏其中一个条件就可以避免内存泄漏，单但为了更好的避免这种情况的发生我们使用ThreadLocal时遵守以下两个小原则:
①ThreadLocal申明为private static final。
     Private与final 尽可能不让他人修改变更引用，
     Static 表示为类属性，只有在程序结束才会被回收。
②ThreadLocal使用后务必调用remove方法。
    最简单有效的方法是使用后将其移除。

