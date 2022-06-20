## 1 Atomic原子操作
在 Java 5.0 提供了 java.util.concurrent(简称JUC)包,在此包中增加了在并发编程中很常用的工具类

Java从JDK1.5开始提供了java.util.concurrent.atomic包，方便程序员在多线程环境下，无锁的进行原子操作。原子变量的底层使用了处理器提供的原子指令，但是不同的CPU架构可能提供的原子指令不一样，也有可能需要某种形式的内部锁,所以该方法不能绝对保证线程不被阻塞。

在Atomic包里一共有12个类，四种原子更新方式，分别是原子更新基本类型，原子更新数组，原子更新引用和原子更新字段。Atomic包里的类基本都是使用Unsafe实现的包装类。
* 原子更新基本类型类： AtomicBoolean，AtomicInteger，AtomicLong，AtomicReference
* 原子更新数组类：AtomicIntegerArray，AtomicLongArray
* 原子更新引用类型：AtomicMarkableReference，AtomicStampedReference，AtomicReferenceArray
* 原子更新字段类：AtomicLongFieldUpdater，AtomicIntegerFieldUpdater，AtomicReferenceFieldUpdater






## 2. 预备知识
能够弄懂atomic包下这些原子操作类的实现原理，就要先明白什么是CAS操作。

**什么是CAS?**
使用锁时，线程获取锁是一种悲观锁策略，即假设每一次执行临界区代码都会产生冲突，所以当前线程获取到锁的时候同时也会阻塞其他线程获取该锁。而CAS操作（又称为无锁操作）是一种乐观锁策略，它假设所有线程访问共享资源的时候不会出现冲突，既然不会出现冲突自然而然就不会阻塞其他线程的操作。因此，线程就不会出现阻塞停顿的状态。那么，如果出现冲突了怎么办？无锁操作是使用CAS(compare and swap)又叫做比较交换来鉴别线程是否出现冲突，出现冲突就重试当前操作直到没有冲突为止。

**CAS的操作过程：** 可以参考AtomicInteger.compareAndSet(int expect, int update) 方法
CAS比较交换的过程可以通俗的理解为CAS(V,O,N)，包含三个值分别为：V 内存地址存放的实际值；O 预期的值（旧值）；N 更新的新值。当V和O相同时，也就是说旧值和内存中实际的值相同表明该值没有被其他线程更改过，即该旧值O就是目前来说最新的值了，自然而然可以将新值N赋值给V。反之，V和O不相同，表明该值已经被其他线程改过了则该旧值O不是最新版本的值了，所以不能将新值N赋给V，返回V即可。当多个线程使用CAS操作一个变量是，只有一个线程会成功，并成功更新，其余会失败。失败的线程会重新尝试，当然也可以选择挂起线程
CAS的实现需要硬件指令集的支撑，在JDK1.5后虚拟机才可以使用处理器提供的CMPXCHG指令实现。

**Synchronized VS CAS**
元老级的Synchronized(未优化前)最主要的问题是：在存在线程竞争的情况下会出现线程阻塞和唤醒锁带来的性能问题，因为这是一种互斥同步（阻塞同步）。而CAS并不是武断的间线程挂起，当CAS操作失败后会进行一定的尝试，而非进行耗时的挂起唤醒的操作，因此也叫做非阻塞同步。这是两者主要的区别。



**CAS的问题**
1.ABA问题
因为CAS会检查旧值有没有变化，这里存在这样一个有意思的问题。比如一个旧值A变为了成B，然后再变成A，刚好在做CAS时检查发现旧值并没有变化依然为A，但是实际上的确发生了变化。解决方案可以沿袭数据库中常用的乐观锁方式，添加一个版本号可以解决。原来的变化路径A->B->A就变成了1A->2B->3C。
2.自旋时间过长
使用CAS时非阻塞同步，也就是说不会将线程挂起，会自旋（无非就是一个死循环）进行下一次尝试，如果这里自旋时间过长对性能是很大的消耗。如果JVM能支持处理器提供的pause指令，那么在效率上会有一定的提升。




**CAS的优缺点**
* CAS由于是在硬件层面保证的原子性，不会锁住当前线程，它的效率是很高的。
* CAS虽然很高效的实现了原子操作，但是它依然存在三个问题。

1、ABA问题。CAS在操作值的时候检查值是否已经变化，没有变化的情况下才会进行更新。但是如果一个值原来是A，变成B，又变成A，那么CAS进行检查时会认为这个值没有变化，但是实际上却变化了。ABA问题的解决方法是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加一，那么A－B－A 就变成1A-2B－3A。从Java1.5开始JDK的atomic包里提供了一个类AtomicStampedReference来解决ABA问题。

2、并发越高，失败的次数会越多，CAS如果长时间不成功，会极大的增加CPU的开销。因此CAS不适合竞争十分频繁的场景。

3、只能保证一个共享变量的原子操作。当对多个共享变量操作时，CAS就无法保证操作的原子性，这时就可以用锁，或者把多个共享变量合并成一个共享变量来操作。比如有两个共享变量i＝2,j=a，合并一下ij=2a，然后用CAS来操作ij。从Java1.5开始JDK提供了AtomicReference类来保证引用对象的原子性，你可以把多个变量放在一个对象里来进行CAS操作。





## 3. 原子更新基本类型类
用于通过原子的方式更新基本类型，Atomic包提供了以下三个类：
* AtomicBoolean：原子更新布尔类型。

* AtomicInteger：原子更新整型。

* AtomicLong：原子更新长整型。

  ​

AtomicInteger的常用方法如下：
* int addAndGet(int delta) ：以原子方式将输入的数值与实例中的值（AtomicInteger里的value）相加，并返回结果

* boolean compareAndSet(int expect, int update) ：如果输入的数值等于预期值，则以原子方式将该值设置为输入的值。

* int getAndIncrement()：以原子方式将当前值加1，注意：这里返回的是自增前的值。

* void lazySet(int newValue)：最终会设置成newValue，使用lazySet设置值后，可能导致其他线程在之后的一小段时间内还是可以读到旧的值。

* int getAndSet(int newValue)：以原子方式设置为newValue的值，并返回旧值。

  ​

AtomicInteger例子代码如下：
```java
  import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerTest {

	static AtomicInteger ai = new AtomicInteger(1);

	public static void main(String[] args) {
		System.out.println(ai.getAndIncrement());
		System.out.println(ai.get());
	}

}
```
输出
```
1
2
```



**餐后甜点**
Atomic包提供了三种基本类型的原子更新，但是Java的基本类型里还有char，float和double等。那么问题来了，如何原子的更新其他的基本类型呢？Atomic包里的类基本都是使用Unsafe实现的，让我们一起看下Unsafe的源码，发现Unsafe只提供了三种CAS方法，compareAndSwapObject，compareAndSwapInt和compareAndSwapLong，再看AtomicBoolean源码，发现其是先把Boolean转换成整型，再使用compareAndSwapInt进行CAS，所以原子更新double也可以用类似的思路来实现。






## 4. 原子更新数组类
通过原子的方式更新数组里的某个元素，Atomic包提供了以下三个类：
* AtomicIntegerArray：原子更新整型数组里的元素。

* AtomicLongArray：原子更新长整型数组里的元素。

* AtomicReferenceArray：原子更新引用类型数组里的元素。

* AtomicIntegerArray类主要是提供原子的方式更新数组里的整型，其常用方法如下

* int addAndGet(int i, int delta)：以原子方式将输入值与数组中索引i的元素相加。

* boolean compareAndSet(int i, int expect, int update)：如果当前值等于预期值，则以原子方式将数组位置i的元素设置成update值。

  ​

实例代码如下：
```java
  public class AtomicIntegerArrayTest {

  static int[] value = new int[] { 1, 2 };

  static AtomicIntegerArray ai = new AtomicIntegerArray(value);

  public static void main(String[] args) {
  	ai.getAndSet(0, 3);
  	System.out.println(ai.get(0));
  	          System.out.println(value[0]);
  }

}
```
输出
```
3
1
```

AtomicIntegerArray类需要注意的是，数组value通过构造方法传递进去，然后AtomicIntegerArray会将当前数组复制一份，所以当AtomicIntegerArray对内部的数组元素进行修改时，不会影响到传入的数组。






## 5. 原子更新引用类型
  原子更新基本类型的AtomicInteger，只能更新一个变量，如果要原子的更新多个变量，就需要使用这个原子更新引用类型提供的类。Atomic包提供了以下三个类：
* AtomicReference：原子更新引用类型。

* AtomicReferenceFieldUpdater：原子更新引用类型里的字段。

* AtomicMarkableReference：原子更新带有标记位的引用类型。可以原子的更新一个布尔类型的标记位和引用类型。构造方法是AtomicMarkableReference(V initialRef, boolean initialMark)

  ​

AtomicReference的使用例子代码如下：
```java
  public class AtomicReferenceTest {

  public static AtomicReference<user> atomicUserRef = new AtomicReference</user><user>();

  public static void main(String[] args) {
  	User user = new User("conan", 15);
  	atomicUserRef.set(user);
  	User updateUser = new User("Shinichi", 17);
  	atomicUserRef.compareAndSet(user, updateUser);
  	System.out.println(atomicUserRef.get().getName());
  	System.out.println(atomicUserRef.get().getOld());
  }

  static class User {
  	private String name;
  	private int old;
  	
  	public User(String name, int old) {
  		this.name = name;
  		this.old = old;
  	}
  	
  	public String getName() {
  		return name;
  	}
  	
  	public int getOld() {
  		return old;
  	}
  }
  }
```
  输出
```
  Shinichi
  17
```





## 6. 原子更新字段类
  如果我们只需要某个类里的某个字段，那么就需要使用原子更新字段类，Atomic包提供了以下三个类：
* AtomicIntegerFieldUpdater：原子更新整型的字段的更新器。

* AtomicLongFieldUpdater：原子更新长整型字段的更新器。

* AtomicStampedReference：原子更新带有版本号的引用类型。该类将整数值与引用关联起来，可用于原子的更数据和数据的版本号，可以解决使用CAS进行原子更新时，可能出现的ABA问题。



原子更新字段类都是抽象类，每次使用都时候必须使用静态方法newUpdater创建一个更新器。原子更新类的字段的必须使用public volatile修饰符。AtomicIntegerFieldUpdater的例子代码如下：
```java
  public class AtomicIntegerFieldUpdaterTest {

  private static AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater
  		.newUpdater(User.class, "old");

  public static void main(String[] args) {
  	User conan = new User("conan", 10);
  	System.out.println(a.getAndIncrement(conan));
  	System.out.println(a.get(conan));
  }

  public static class User {
  	private String name;
  	public volatile int old;
  	
  	public User(String name, int old) {
  		this.name = name;
  		this.old = old;
  	}
  	
  	public String getName() {
  		return name;
  	}
  	
  	public int getOld() {
  		return old;
  	}
  }
  }
```
  输出
```
  10
  11
```




## 7. AtomicInteger 实现自旋锁
```java
  /**
 * 使用AtomicInteger实现自旋锁
    */
    public class SpinLock {

    private AtomicInteger state = new AtomicInteger(0);

    /**
     * 自旋等待直到获得许可
        */
        public void lock(){
        for (;;){
        	//CAS指令要锁总线，效率很差。所以我们通过一个if判断避免了多次使用CAS指令。
        	if (state.get() == 1) {
        		continue;
        	} else if(state.compareAndSet(0, 1)){
        		return;
        	}
        }
        }

    public void unlock(){
    	state.set(0);
    }
    }
```

原理很简单，就是一直CAS抢锁，如果抢不到，就一直死循环，直到抢到了才退出这个循环。

自旋锁实现起来非常简单，如果关键区的执行时间很短，往往自旋等待会是一种比较高效的做法，它可以避免线程的频繁切换和调度。但如果关键区的执行时间很长，那这种做法就会大量地浪费CPU资源。

针对关键区执行时间长的情况，该怎么办呢？





## 8. 实现可等待的锁
  如果关键区的执行时间很长，自旋的锁会大量地浪费CPU资源，我们可以这样改进：当一个线程拿不到锁的时候，就让这个线程先休眠等待。这样，CPU就不会白白地空转了。大致步骤如下：
* 需要一个容器，如果线程抢不到锁，就把线程挂起来，并记录到这个容器里。
* 当一个线程放弃了锁，得从容器里找出一个挂起的线程，把它恢复了。
```java
  /**
 * 使用AtomicInteger实现可等待锁
    */
    public class BlockLock implements Lock {

    private AtomicInteger state = new AtomicInteger(0);
    private ConcurrentLinkedQueue<Thread> waiters = new ConcurrentLinkedQueue<>();

    @Override
    public void lock() {
        if (state.compareAndSet(0, 1)) {
            return;
        }
        //放到等待队列
        waiters.add(Thread.currentThread());
        
        for (;;) {
            if (state.get() == 0) {
                if (state.compareAndSet(0, 1)) {
                    waiters.remove(Thread.currentThread());
                    return;
                }
            } else {
                LockSupport.park();     //挂起线程
            }
        }
    }

    @Override
    public void unlock() {
        state.set(0);
        //唤醒等待队列的第一个线程
        Thread waiterHead = waiters.peek();
        if(waiterHead != null){
            LockSupport.unpark(waiterHead);     //唤醒线程
        }
    }
    }
```
我们引入了一个 waitList，用于存储抢不到锁的线程，让它挂起。这里我们先借用一下JDK里的ConcurrentLinkedQueue，因为这个Queue也是使用CAS操作实现的无锁队列，所以并不会引入JDK里的其他锁机制。如果大家去看AbstractQueuedSynchronizer的实现，就会发现，它的acquire()方法的逻辑与上面的实现是一样的。

不过上面的代码是不是没问题了呢？如果一个线程在还未调用park挂起之前，是不是有可能被其他线程先调用一遍unpark？这就是唤醒发生在休眠之前。发生这样的情况会不会带来问题呢？



