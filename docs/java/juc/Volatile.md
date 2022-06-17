## 一、简介
volatile是Java提供的一种轻量级的同步机制。Java 语言包含两种内在的同步机制：同步块（或方法）和 volatile 变量，相比于synchronized（synchronized通常称为重量级锁），volatile更轻量级，因为它不会引起线程上下文的切换和调度。但是volatile 变量的同步性较差（有时它更简单并且开销更低），而且其使用也更容易出错。




## 二、并发编程的3个基本概念
**（1）原子性**
定义： 即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

原子性是拒绝多线程操作的，不论是多核还是单核，具有原子性的量，同一时刻只能有一个线程来对它进行操作。简而言之，在整个操作过程中不会被线程调度器中断的操作，都可认为是原子性。例如 a=1是原子性操作，但是a++和a +=1就不是原子性操作。Java中的原子

性操作包括：
a. 基本类型的读取和赋值操作，且赋值必须是数字赋值给变量，变量之间的相互赋值不是原子性操作。
b.所有引用reference的赋值操作
c.java.concurrent.Atomic.* 包中所有类的一切操作




**（2）可见性**
定义：指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

在多线程环境下，一个线程对共享变量的操作对其他线程是不可见的。Java提供了volatile来保证可见性，当一个变量被volatile修饰后，表示着线程本地内存无效，当一个线程修改共享变量后他会立即被更新到主内存中，其他线程读取共享变量时，会直接从主内存中读取。当然，synchronize和Lock都可以保证可见性。synchronized和Lock能保证同一时刻只有一个线程获取锁然后执行同步代码，并且在释放锁之前会将对变量的修改刷新到主存当中。因此可以保证可见性。




**（3）有序性**
定义：即程序执行的顺序按照代码的先后顺序执行。

Java内存模型中的有序性可以总结为：如果在本线程内观察，所有操作都是有序的；如果在一个线程中观察另一个线程，所有操作都是无序的。前半句是指“线程内表现为串行语义”，后半句是指“指令重排序”现象和“工作内存主主内存同步延迟”现象。
在Java内存模型中，为了效率是允许编译器和处理器对指令进行重排序，当然重排序不会影响单线程的运行结果，但是对多线程会有影响。Java提供volatile来保证一定的有序性。最著名的例子就是单例模式里面的DCL（双重检查锁）。另外，可以通过synchronized和Lock来保证有序性，synchronized和Lock保证每个时刻是有一个线程执行同步代码，相当于是让线程顺序执行同步代码，自然就保证了有序性。




## 三、锁的互斥和可见性
锁提供了两种主要特性：互斥（mutual exclusion） 和可见性（visibility）。

（1）互斥即一次只允许一个线程持有某个特定的锁，一次就只有一个线程能够使用该共享数据。

（2）可见性要更加复杂一些，它必须确保释放锁之前对共享数据做出的更改对于随后获得该锁的另一个线程是可见的。也即当一条线程修改了共享变量的值，新值对于其他线程来说是可以立即得知的。如果没有同步机制提供的这种可见性保证，线程看到的共享变量可能是修改前的值或不一致的值，这将引发许多严重问题。

要使 volatile 变量提供理想的线程安全，必须同时满足下面两个条件：
a.对变量的写操作不依赖于当前值。
b.该变量没有包含在具有其他变量的不变式中。

实际上，这些条件表明，可以被写入 volatile 变量的这些有效值独立于任何程序的状态，包括变量的当前状态。事实上就是保证操作是原子性操作，才能保证使用volatile关键字的程序在并发时能够正确执行。






## 四、Java的内存模型JMM以及共享变量的可见性
JMM决定一个线程对共享变量的写入何时对另一个线程可见，JMM定义了线程和主内存之间的抽象关系：共享变量存储在主内存(Main Memory)中，每个线程都有一个私有的本地内存（Local Memory），本地内存保存了被该线程使用到的主内存的副本拷贝，线程对变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量。

![](https://raw.githubusercontent.com/Raray-chuan/xichuan_blog_pic/main/img/202206171155606.png)

对于普通的共享变量来讲，线程A将其修改为某个值发生在线程A的本地内存中，此时还未同步到主内存中去；而线程B已经缓存了该变量的旧值，所以就导致了共享变量值的不一致。解决这种共享变量在多线程模型中的不可见性问题，较粗暴的方式自然就是加锁，但是此处使用synchronized或者Lock这些方式太重量级了，比较合理的方式其实就是volatile。

需要注意的是，JMM是个抽象的内存模型，所以所谓的本地内存，主内存都是抽象概念，并不一定就真实的对应cpu缓存和物理内存






## 五、volatile变量的特性

**（1）保证可见性，不保证原子性**
a.当写一个volatile变量时，JMM会把该线程本地内存中的变量强制刷新到主内存中去；
b.这个写会操作会导致其他线程中的缓存无效。

**（2）禁止指令重排**
重排序是指编译器和处理器为了优化程序性能而对指令序列进行排序的一种手段。重排序需要遵守一定规则：

a.重排序操作不会对存在数据依赖关系的操作进行重排序。
　 比如：a=1;b=a; 这个指令序列，由于第二个操作依赖于第一个操作，所以在编译时和处理器运
行时这两个操作不会被重排序。

b.重排序是为了优化性能，但是不管怎么重排序，单线程下程序的执行结果不能被改变
　 比如：a=1;b=2;c=a+b这三个操作，第一步（a=1)和第二步(b=2)由于不存在数据依赖关系， 所以可能会发生重排序，但是c=a+b这个操作是不会被重排序的，因为需要保证最终的结果一定是c=a+b=3。
重排序在单线程下一定能保证结果的正确性，但是在多线程环境下，可能发生重排序，影响结果，下例中的1和2由于不存在数据依赖关系，则有可能会被重排序，先执行status=true再执行a=2。而此时线程B会顺利到达4处，而线程A中a=2这个操作还未被执行，所以b=a+1的结果也有可能依然等于2。
```java
 class TestVolatile {
    int a = 1;
    boolean status = false;

    /**
     * 状态切换为true
     */
    public void changeStatus() {
        a = 2;//1
        status = true;//2
    }

    /**
     * 若状态为true，则running.
     */
    public void run() {
        if (status) {//3
            int b = a + 1;//4
            System.out.println(b);
        }
    }
}
```

使用volatile关键字修饰共享变量便可以禁止这种重排序。若用volatile修饰共享变量，在编译时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序

volatile禁止指令重排序也有一些规则：
a.当程序执行到volatile变量的读操作或者写操作时，在其前面的操作的更改肯定全部已经进行，且结果已经对后面的操作可见；在其后面的操作肯定还没有进行；

b.在进行指令优化时，不能将在对volatile变量访问的语句放在其后面执行，也不能把volatile变量后面的语句放到其前面执行。

即执行到volatile变量时，其前面的所有语句都执行完，后面所有语句都未执行。且前面语句的结果对volatile变
量及其后面语句可见。





## 六、volatile不适用的场景
**（1）volatile不适合复合操作**
例如，inc++不是一个原子性操作，可以由读取、加、赋值3步组成，所以结果并不能达到30000。
```java
 class Test {
    public volatile int inc = 0;

    public void increase() {
        inc++;
    }

    public static void main(String[] args) {
        final Test test = new Test();
        for (int i = 0; i < 10; i++) {
            new Thread() {
                public void run() {
                    for (int j = 0; j < 1000; j++)
                        test.increase();
                };
            }.start();
            
            while (Thread.activeCount() > 1) //保证前面的线程 都执行完
                Thread.yield();
            System.out.println(test.inc);
        }
    }
}
```


**（2）解决方法**
1.采用synchronized

```java
class Test {
    public int inc = 0;
    
    public synchronized void increase() {
        inc++;
    }
    
    public static void main(String[ ] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        
        while(Thread.activeCount()>1) //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```



2.采用Lock

```java
 class Test {
    public int inc = 0;
    Lock lock = new ReentrantLock( ) ;
    
    public void increase() {
        lock.lock();
        try {
            inc++;
        } finally{
            lock.unlock();
        }
    }
    
    public static void main(String[] args) {
        final Test test = new Test();
        
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        
        while(Thread.activeCount()>1) //保证前面的线程都执行完
            Thread.yield();
        System.out.println(test.inc);
    }
}
```



3.采用java并发包中的原子操作类，原子操作类是通过CAS循环的方式来保证其原子性的

```java
 class Test {
    public AtomicInteger inc = new AtomicInteger();
    
    public void increase() {
        inc.getAndIncrement();
    }
    
    public static void main(String[] args) {
        final Test test = new Test();
        for(int i=0;i<10;i++){
            new Thread( ){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase( ) ;
                };
            }.start();
        }
        
        while(Thread. activeCount()>1) //保证前面的线程都执行完
            Thread.yield();
        System. out. println(test.inc);
    }
}
```





## 七、volatile原理

volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令，lock前缀指令实际上相当于一个内存屏障（也成内存栅栏），内存屏障会提供3个功能：

I. 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内
存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；

II. 它会强制将对缓存的修改操作立即写入主存；

III. 如果是写操作，它会导致其他CPU中对应的缓存行无效。





## 八、单例模式的双重锁为什么要加volatile
```java
class TestInstance {
    private volatile static TestInstance instance;

    public static TestInstance getInstance() { // 1
        if (instance == null) {//2
            synchronized (TestInstance.class) {//3
                if (instance == null) { //4
                    instance = new TestInstance();//5
                }
            }
        }
        return instance;//6
    }
}
```

需要volatile关键字的原因是，在并发情况下，如果没有volatile关键字，在第5行会出现问题。instance = new TestInstance();可以分解为3行伪代码
```
a.memory = allocate() //分配内存

b. ctorInstanc(memory) //初始化对象

c. instance = memory //设置instance指向刚分配的地址
```
上面的代码在编译运行时，可能会出现重排序从a-b-c排序为a-c-b。在多线程的情况下会出现以下问题。当线程A在执行第5行代码时，B线程进来执行到第2行代码。假设此时A执行的过程中发生了指令重排序，即先执行了a和c，没有执行b。那么由于A线程执行了c导致instance指向了一段地址，所以B线程判断instance不为null，会直接跳到第6行并返回一个未初始化的对象。





## 九. volatile的适用场景
**模式 #1：状态标志**
也许实现 volatile 变量的规范使用仅仅是使用一个布尔状态标志，用于指示发生了一个重要的一次性事件，例如完成初始化或请求停机。
```java
volatile boolean shutdownRequested;
  
...
  
public void shutdown() { 
  shutdownRequested = true; 
}
  
public void doWork() { 
  while (!shutdownRequested) { 
    // do stuff
  }
}
```
线程1执行doWork()的过程中，可能有另外的线程2调用了shutdown，所以boolean变量必须是volatile。
而如果使用 synchronized 块编写循环要比使用 volatile 状态标志编写麻烦很多。由于 volatile 简化了编码，并且状态标志并不依赖于程序内任何其他状态，因此此处非常适合使用 volatile。
这种类型的状态标记的一个公共特性是：通常只有一种状态转换；shutdownRequested 标志从false 转换为true，然后程序停止。这种模式可以扩展到来回转换的状态标志，但是只有在转换周期不被察觉的情况下才能扩展（从false 到true，再转换到false）。此外，还需要某些原子状态转换机制，例如原子变量。



**模式 #2：一次性安全发布（one-time safe publication）**
在缺乏同步的情况下，可能会遇到某个对象引用的更新值（由另一个线程写入）和该对象状态的旧值同时存在。
这就是造成著名的双重检查锁定（double-checked-locking）问题的根源，其中对象引用在没有同步的情况下进行读操作，产生的问题是您可能会看到一个更新的引用，但是仍然会通过该引用看到不完全构造的对象。

```java
//注意volatile！！！！！！！！！！！！！！！！！ 
private volatile static Singleton instace;  
  
public static Singleton getInstance(){  
  //第一次null检查   
  if(instance == null){      
    synchronized(Singleton.class) {  //1   
      //第二次null检查    
      if(instance == null){     //2 
        instance = new Singleton();//3 
      } 
    }      
  } 
  return instance; 
}
```
如果不用volatile，则因为内存模型允许所谓的“无序写入”，可能导致失败。——某个线程可能会获得一个未完全初始化的实例。
考察上述代码中的 //3 行。此行代码创建了一个 Singleton 对象并初始化变量 instance 来引用此对象。这行代码的问题是：在Singleton 构造函数体执行之前，变量instance 可能成为非 null 的！

什么？这一说法可能让您始料未及，但事实确实如此。
在解释这个现象如何发生前，请先暂时接受这一事实，我们先来考察一下双重检查锁定是如何被破坏的。假设上述代码执行以下事件序列：
1.线程 1 进入 getInstance() 方法。
2.由于 instance 为 null，线程 1 在 //1 处进入synchronized 块。
3.线程 1 前进到 //3 处，但在构造函数执行之前，使实例成为非null。
4.线程 1 被线程 2 预占。
5.线程 2 检查实例是否为 null。因为实例不为 null，线程 2 将instance 引用返回，返回一个构造完整但部分初始化了的Singleton 对象。
6.线程 2 被线程 1 预占。
7.线程 1 通过运行 Singleton 对象的构造函数并将引用返回给它，来完成对该对象的初始化。



**模式 #3：独立观察（independent observation）**
安全使用 volatile 的另一种简单模式是：定期 “发布” 观察结果供程序内部使用。【例如】假设有一种环境传感器能够感觉环境温度。一个后台线程可能会每隔几秒读取一次该传感器，并更新包含当前文档的 volatile 变量。然后，其他线程可以读取这个变量，从而随时能够看到最新的温度值。
使用该模式的另一种应用程序就是收集程序的统计信息。【例】如下代码展示了身份验证机制如何记忆最近一次登录的用户的名字。将反复使用lastUser 引用来发布值，以供程序的其他部分使用。

```java
public class UserManager {
  public volatile String lastUser; //发布的信息
  
  public boolean authenticate(String user, String password) {
    boolean valid = passwordIsValid(user, password);
    if (valid) {
      User u = new User();
      activeUsers.add(u);
      lastUser = user;
    }
    return valid;
  }
} 
```



**模式 #4：“volatile bean” 模式**
volatile bean 模式的基本原理是：很多框架为易变数据的持有者（例如 HttpSession）提供了容器，但是放入这些容器中的对象必须是线程安全的。
在 volatile bean 模式中，JavaBean 的所有数据成员都是 volatile 类型的，并且 getter 和 setter 方法必须非常普通——即不包含约束！

```java
@ThreadSafe
public class Person {
  private volatile String firstName;
  private volatile String lastName;
  private volatile int age;
  
  public String getFirstName() { return firstName; }
  public String getLastName() { return lastName; }
  public int getAge() { return age; }
  
  public void setFirstName(String firstName) { 
    this.firstName = firstName;
  }
  
  public void setLastName(String lastName) { 
    this.lastName = lastName;
  }
  
  public void setAge(int age) { 
    this.age = age;
  }
}
```



**模式 #5：开销较低的“读－写锁”策略**
如果读操作远远超过写操作，您可以结合使用内部锁和 volatile 变量来减少公共代码路径的开销。
如下显示的线程安全的计数器，使用 synchronized 确保增量操作是原子的，并使用 volatile 保证当前结果的可见性。如果更新不频繁的话，该方法可实现更好的性能，因为读路径的开销仅仅涉及 volatile 读操作，这通常要优于一个无竞争的锁获取的开销。

```java
@ThreadSafe
public class CheesyCounter {
  // Employs the cheap read-write lock trick
  // All mutative operations MUST be done with the 'this' lock held
  @GuardedBy("this") private volatile int value;
  
  //读操作，没有synchronized，提高性能
  public int getValue() { 
    return value; 
  } 
  
  //写操作，必须synchronized。因为x++不是原子操作
  public synchronized int increment() {
    return value++;
  }
```
使用锁进行所有变化的操作，使用 volatile 进行只读操作。
其中，锁一次只允许一个线程访问值，volatile 允许多个线程执行读操作