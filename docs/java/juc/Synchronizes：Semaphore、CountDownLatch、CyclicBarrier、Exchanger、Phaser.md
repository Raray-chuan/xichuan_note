## 1. Semaphore
Semaphore（信号量）是用来控制同时访问特定资源的线程数量，通过协调各个线程，保证合理的使用公共资源。
Semaphore维护了一个许可集，其实就是一定数量的“许可证”。当有线程想要访问共享资源时，需要先获取(acquire)的许可；如果许可不够了，线程需要一直等待，直到许可可用。当线程使用完共享资源后，可以归还(release)许可，以供其它需要的线程使用。

和ReentrantLock类似，Semaphore支持公平/非公平策略。

Semaphore的主要方法摘要：
　　void acquire():从此信号量获取一个许可，在提供一个许可前一直将线程阻塞，否则线程被中断。
　　void release():释放一个许可，将其返回给信号量。
　　int availablePermits():返回此信号量中当前可用的许可数。
　　boolean hasQueuedThreads():查询是否有线程正在等待获取。



**使用**
```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;

//创建一个会实现print queue的类名为 PrintQueue。
class PrintQueue {

    // 声明一个对象为Semaphore，称它为semaphore。
    private final Semaphore semaphore;
    // 实现类的构造函数并初始能保护print quere的访问的semaphore对象的值。
    public PrintQueue() {
        semaphore = new Semaphore(1);
    }

    //实现Implement the printJob()方法，此方法可以模拟打印文档，并接收document对象作为参数。
    public void printJob(Object document) {
//在这方法内，首先，你必须调用acquire()方法获得demaphore。这个方法会抛出 InterruptedException异常，使用必须包含处理这个异常的代码。
        try {
            semaphore.acquire();

//然后，实现能随机等待一段时间的模拟打印文档的行。
            long duration = (long) (Math.random() * 10);

            System.out.printf("%s: PrintQueue: Printing a Job during %d seconds\n", Thread.currentThread().getName(), duration);

            Thread.sleep(duration);

//最后，释放semaphore通过调用semaphore的relaser()方法。
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            semaphore.release();
        }
    }

}

//创建一个名为Job的类并一定实现Runnable 接口。这个类实现把文档传送到打印机的任务。
class Job implements Runnable {
    //声明一个对象为PrintQueue，名为printQueue。
    private PrintQueue printQueue;
    //实现类的构造函数，初始化这个类里的PrintQueue对象。
    public Job(PrintQueue printQueue) {
        this.printQueue = printQueue;
    }

    //实现方法run()。
    @Override
    public void run() {
                //首先， 此方法写信息到操控台表明任务已经开始执行了。
        System.out.printf("%s: Going to print a job\n", Thread.currentThread().getName());
                // 然后，调用PrintQueue 对象的printJob()方法。
        printQueue.printJob(new Object());
                //最后， 此方法写信息到操控台表明它已经结束运行了。
        System.out.printf("%s: The document has been printed\n", Thread.currentThread().getName());

    }
}

public class SemaphoreTest {

    public static void main(String args[]) {

                // 创建PrintQueue对象名为printQueue。
        PrintQueue printQueue = new PrintQueue();
            //创建10个threads。每个线程会执行一个发送文档到print queue的Job对象。
        Thread thread[] = new Thread[10];

        for (int i = 0; i < 10; i++) {
            thread[i] = new Thread(new Job(printQueue), "Thread" + i);
        }

        for (int i = 0; i < 10; i++) {
            thread[i].start();
        }

    }
}
```




## 2. CountDownLatch
在多线程协作完成业务功能时，有时候需要等待其他多个线程完成任务之后，主线程才能继续往下执行业务功能，在这种的业务场景下，通常可以使用Thread类的join方法，让主线程等待被join的线程执行完之后，主线程才能继续往下执行。当然，使用线程间消息通信机制也可以完成。其实，java并发工具类中为我们提供了类似“倒计时”这样的工具类，可以十分方便的完成所说的这种业务场景。

CountDownLatch允许一个或多个线程等待其他线程完成工作。

CountDownLatch相关方法：
* public CountDownLatch(int count) 构造方法会传入一个整型数N，之后调用CountDownLatch的countDown方法会对N减一，知道N减到0的时候，当前调用await方法的线程继续执行。

* await() throws InterruptedException：调用该方法的线程等到构造方法传入的N减到0的时候，才能继续往下执行；

* await(long timeout, TimeUnit unit)：与上面的await方法功能一致，只不过这里有了时间限制，调用该方法的线程等到指定的timeout时间后，不管N是否减至为0，都会继续往下执行；

* countDown()：使CountDownLatch初始值N减1；

* long getCount()：获取当前CountDownLatch维护的值

  ​


**CountDownLatch的用法**
CountDownLatch典型用法：1、某一线程在开始运行前等待n个线程执行完毕。将CountDownLatch的计数器初始化为new CountDownLatch(n)，每当一个任务线程执行完毕，就将计数器减1 countdownLatch.countDown()，当计数器的值变为0时，在CountDownLatch上await()的线程就会被唤醒。一个典型应用场景就是启动一个服务时，主线程需要等待多个组件加载完毕，之后再继续执行。
CountDownLatch典型用法：2、实现多个线程开始执行任务的最大并行性。注意是并行性，不是并发，强调的是多个线程在某一时刻同时开始执行。类似于赛跑，将多个线程放到起点，等待发令枪响，然后同时开跑。做法是初始化一个共享的CountDownLatch(1)，将其计算器初始化为1，多个线程在开始执行任务前首先countdownlatch.await()，当主线程调用countDown()时，计数器变为0，多个线程同时被唤醒。



**CountDownLatch的不足**
CountDownLatch是一次性的，计算器的值只能在构造方法中初始化一次，之后没有任何机制再次对其设置值，当CountDownLatch使用完毕后，它不能再次被使用。




**栗子**：运动员进行跑步比赛时，假设有6个运动员参与比赛，裁判员在终点会为这6个运动员分别计时，可以想象没当一个运动员到达终点的时候，对于裁判员来说就少了一个计时任务。直到所有运动员都到达终点了，裁判员的任务也才完成。这6个运动员可以类比成6个线程，当线程调用CountDownLatch.countDown方法时就会对计数器的值减一，直到计数器的值为0的时候，裁判员（调用await方法的线程）才能继续往下执行。
```java
public class CountDownLatchTest {
private static CountDownLatch startSignal = new CountDownLatch(1);
//用来表示裁判员需要维护的是6个运动员
private static CountDownLatch endSignal = new CountDownLatch(6);

public static void main(String[] args) throws InterruptedException {
    ExecutorService executorService = Executors.newFixedThreadPool(6);
    for (int i = 0; i < 6; i++) {
        executorService.execute(() -> {
            try {
                System.out.println(Thread.currentThread().getName() + " 运动员等待裁判员响哨！！！");
                startSignal.await();
                System.out.println(Thread.currentThread().getName() + "正在全力冲刺");
                endSignal.countDown();
                System.out.println(Thread.currentThread().getName() + "  到达终点");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });
    }
    System.out.println("裁判员响哨开始啦！！！");
    startSignal.countDown();
    endSignal.await();
    System.out.println("所有运动员到达终点，比赛结束！");
    executorService.shutdown();
}
}

该示例代码中设置了两个CountDownLatch，第一个endSignal用于控制让main线程（裁判员）必须等到其他线程（运动员）让CountDownLatch维护的数值N减到0为止，相当于一个完成信号；另一个startSignal用于让main线程对其他线程进行“发号施令”，相当于一个入口或者开关。

startSignal引用的CountDownLatch初始值为1，而其他线程执行的run方法中都会先通过 startSignal.await()让这些线程都被阻塞，直到main线程通过调用startSignal.countDown();，将值N减1，CountDownLatch维护的数值N为0后，其他线程才能往下执行，并且，每个线程执行的run方法中都会通过endSignal.countDown();对endSignal维护的数值进行减一，由于往线程池提交了6个任务，会被减6次，所以endSignal维护的值最终会变为0，因此main线程在latch.await();阻塞结束，才能继续往下执行。

注意：当调用CountDownLatch的countDown方法时，当前线程是不会被阻塞，会继续往下执行。
```





## 3.  CyclicBarrier
CountDownLatch是一个倒数计数器，在计数器不为0时，所有调用await的线程都会等待，当计数器降为0，线程才会继续执行，且计数器一旦变为0，就不能再重置了。

CyclicBarrier可以认为是一个栅栏，栅栏的作用是什么？就是阻挡前行。

CyclicBarrier是一个可以循环使用的栅栏，它做的事情就是：让线程到达栅栏时被阻塞(调用await方法)，直到到达栅栏的线程数满足指定数量要求时，栅栏才会打开放行，被栅栏拦截的线程才可以执行。

当多个线程都达到了指定点后，才能继续往下继续执行。这就有点像报数的感觉，假设6个线程就相当于6个运动员，到赛道起点时会报数进行统计，如果刚好是6的话，这一波就凑齐了，才能往下执行。这里的6个线程，也就是计数器的初始值6，是通过CyclicBarrier的构造方法传入的。

CyclicBarrier的主要方法：
* await() throws InterruptedException, BrokenBarrierException  等到所有的线程都到达指定的临界点；
* await(long timeout, TimeUnit unit) throws InterruptedException, BrokenBarrierException, TimeoutException 与上面的await方法功能基本一致，只不过这里有超时限制，阻塞等待直至到达超时时间为止；
* int getNumberWaiting()获取当前有多少个线程阻塞等待在临界点上；
* boolean isBroken()用于查询阻塞等待的线程是否被中断
* void reset()将屏障重置为初始状态。如果当前有线程正在临界点等待的话，将抛出BrokenBarrierException。  

另外需要注意的是，CyclicBarrier提供了这样的构造方法：
```java
public CyclicBarrier(int parties, Runnable barrierAction)
```

可以用来，当指定的线程都到达了指定的临界点的时，接下来执行的操作可以由barrierAction传入即可。



**栗子**：6个运动员准备跑步比赛，运动员在赛跑需要在起点做好准备，当裁判发现所有运动员准备完毕后，就举起发令枪，比赛开始。这里的起跑线就是屏障，是临界点，而这6个运动员就类比成线程的话，就是这6个线程都必须到达指定点了，意味着凑齐了一波，然后才能继续执行，否则每个线程都得阻塞等待，直至凑齐一波即可。
```java
public class CyclicBarrierTest {
    public static void main(String[] args) {

        int N = 6;  // 运动员数
        CyclicBarrier cb = new CyclicBarrier(N, new Runnable() {
            @Override
            public void run() {
                System.out.println("所有运动员已准备完毕，发令枪：跑！");
            }
        });

        for (int i = 0; i < N; i++) {
            Thread t = new Thread(new PrepareWork(cb), "运动员[" + i + "]");
            t.start();
        }
    }


private static class PrepareWork implements Runnable {​
        private CyclicBarrier cb;

        PrepareWork(CyclicBarrier cb) {
            this.cb = cb;
        }

        @Override
        public void run() {

            try {
                Thread.sleep(500);
                System.out.println(Thread.currentThread().getName() + ": 准备完成");
                cb.await();          // 在栅栏等待
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }
}
```
从输出结果可以看出，当6个运动员（线程）都到达了指定的临界点（barrier）时候，才能继续往下执行，否则，则会阻塞等待在调用await()处。
（在CyclicBarrier构造函数中传入N参数，代表当由N个线程调用cb.await();  方法，CyclicBarrier才会往下执行barrierAction）

**CyclicBarrier对异常的处理**
线程在阻塞过程中，可能被中断，那么既然CyclicBarrier放行的条件是等待的线程数达到指定数目，万一线程被中断导致最终的等待线程数达不到栅栏的要求怎么办？
```java
public int await() throws InterruptedException, BrokenBarrierException {
    //...
}
```
可以看到，这个方法除了抛出InterruptedException异常外，还会抛出BrokenBarrierException。
BrokenBarrierException表示当前的CyclicBarrier已经损坏了，等不到所有线程都到达栅栏了，所以已经在等待的线程也没必要再等了，可以散伙了。

出现以下几种情况之一时，当前等待线程会抛出BrokenBarrierException异常：
* 其它某个正在await等待的线程被中断了；
* 其它某个正在await等待的线程超时了；
* 某个线程重置了CyclicBarrier；

另外，只要正在Barrier上等待的任一线程抛出了异常，那么Barrier就会认为肯定是凑不齐所有线程了，就会将栅栏置为损坏（Broken）状态，并传播BrokenBarrierException给其它所有正在等待（await）的线程。



**异常情况模拟：**

```java
public class CyclicBarrierTest {
    public static void main(String[] args) throws InterruptedException {

        int N = 6;  // 运动员数
        CyclicBarrier cb = new CyclicBarrier(N, new Runnable() {
            @Override
            public void run() {
                System.out.println("所有运动员已准备完毕，发令枪：跑！");
            }
        });

        List<Thread> list = new ArrayList<>();
        for (int i = 0; i < N; i++) {
            Thread t = new Thread(new PrepareWork(cb), "运动员[" + i + "]");
            list.add(t);
            t.start();
            if (i == 3) {
                t.interrupt();  // 运动员[3]置中断标志位
            }
        }

        Thread.sleep(2000);
        System.out.println("Barrier是否损坏：" + cb.isBroken());
    }
```


**CountDownLatch与CyclicBarrier的比较**
CountDownLatch与CyclicBarrier都是用于控制并发的工具类，都可以理解成维护的就是一个计数器，但是这两者还是各有不同侧重点的：
* CountDownLatch一般用于某个线程A等待若干个其他线程执行完任务之后，它才执行；而CyclicBarrier一般用于一组线程互相等待至某个状态，然后这一组线程再同时执行；CountDownLatch强调一个线程等多个线程完成某件事情。CyclicBarrier是多个线程互等，等大家都完成，再携手共进。
* CountDownLatch方法比较少，操作比较简单，而CyclicBarrier提供的方法更多，比如能够通过getNumberWaiting()，isBroken()这些方法获取当前多个线程的状态，并且CyclicBarrier的构造方法可以传入barrierAction，指定当所有线程都到达时执行的业务功能；
* CountDownLatch是不能复用的，而CyclicLatch是可以复用的。









## 4. Exchanger
Exchanger可以用来在两个线程之间交换持有的对象。当Exchanger在一个线程中调用exchange方法之后，会等待另外的线程调用同样的exchange方法，两个线程都调用exchange方法之后，传入的参数就会交换。



**两个主要方法**
`public V exchange(V x) throws InterruptedException`

当这个方法被调用的时候，当前线程将会等待直到其他的线程调用同样的方法。当其他的线程调用exchange之后，当前线程将会继续执行。
在等待过程中，如果有其他的线程interrupt当前线程，则会抛出InterruptedException。

`public V exchange(V x, long timeout, TimeUnit unit) throws InterruptedException, TimeoutException`

多了一个timeout时间。如果在timeout时间之内没有其他线程调用exchange方法，抛出TimeoutException。



**栗子：**
我们先定义一个带交换的类：
然后定义两个Runnable，在run方法中调用exchange方法：
```java
public class ExchangerTest {

    public static void main(String[] args) {
        Exchanger<CustBook> exchanger = new Exchanger<>();
        // Starting two threads
        new Thread(new ExchangerOne(exchanger)).start();
        new Thread(new ExchangerTwo(exchanger)).start();
    }
}
public class CustBook {

    private String name;
}
public class ExchangerOne implements Runnable{

    Exchanger<CustBook> ex;

    ExchangerOne(Exchanger<CustBook> ex){
      this.ex=ex;
    }

    @Override
    public void run() {
    CustBook custBook= new CustBook();
        custBook.setName("book one");

        try {
            CustBook exhangeCustBook=ex.exchange(custBook);
            log.info(exhangeCustBook.getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
public class ExchangerTwo implements Runnable{

    Exchanger<CustBook> ex;

    ExchangerTwo(Exchanger<CustBook> ex){
      this.ex=ex;
    }

    @Override
    public void run() {
    CustBook custBook= new CustBook();
        custBook.setName("book two");

        try {
            CustBook exhangeCustBook=ex.exchange(custBook);
            log.info(exhangeCustBook.getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```






## 5. Phaser
Phaser是一个同步工具类，适用于一些需要分阶段的任务的处理。它的功能与 CyclicBarrier和CountDownLatch类似，类似于一个多阶段的栅栏，并且功能更强大，我们来比较下这三者的功能：

| CountDownLatch | 倒数计数器，初始时设定计数器值，线程可以在计数器上等待，当计数器值归0后，所有等待的线程继续执行 |
| -------------- | ---------------------------------------- |
| CyclicBarrier  | 循环栅栏，初始时设定参与线程数，当线程到达栅栏后，会等待其它线程的到达，当到达栅栏的总数满足指定数后，所有等待的线程继续执行 |
| Phaser         | 多阶段栅栏，可以在初始时设定参与线程数，也可以中途注册/注销参与者，当到达的参与者数量满足栅栏设定的数量后，会进行阶段升级（advance） |

**相关概念：**
**phase(阶段)**
Phaser也有栅栏，在Phaser中，栅栏的名称叫做phase(阶段)，在任意时间点，Phaser只处于某一个phase(阶段)，初始阶段为0，最大达到Integerr.MAX_VALUE，然后再次归零。当所有parties参与者都到达后，phase值会递增。

**parties(参与者)**
Phaser既可以在初始构造时指定参与者的数量，也可以中途通过register、bulkRegister、arriveAndDeregister等方法注册/注销参与者。

**arrive(到达) / advance(进阶)**
Phaser注册完parties（参与者）之后，参与者的初始状态是unarrived的，当参与者到达（arrive）当前阶段（phase）后，状态就会变成arrived。当阶段的到达参与者数满足条件后（注册的数量等于到达的数量），阶段就会发生进阶（advance）——也就是phase值+1。

**Termination（终止）**
代表当前Phaser对象达到终止状态。

**Tiering（分层）**
Phaser支持分层（Tiering） —— 一种树形结构，通过构造函数可以指定当前待构造的Phaser对象的父结点。之所以引入Tiering，是因为当一个Phaser有大量参与者（parties）的时候，内部的同步操作会使性能急剧下降，而分层可以降低竞争，从而减小因同步导致的额外开销。
在一个分层Phasers的树结构中，注册和撤销子Phaser或父Phaser是自动被管理的。当一个Phaser参与者（parties）数量变成0时，如果有该Phaser有父结点，就会将它从父结点中溢移除。

**核心方法：**
* arriveAndDeregister() 该方法立即返回下一阶段的序号，并且其它线程需要等待的个数减一，    取消自己的注册、把当前线程从之后需要等待的成员中移除。    如果该Phaser是另外一个Phaser的子Phaser（层次化Phaser），    并且该操作导致当前Phaser的成员数为0，则该操作也会将当前Phaser从其父Phaser中移除。
* arrive() 某个参与者完成任务后调用，该方法不作任何等待，直接返回下一阶段的序号。
  awaitAdvance(int phase) 该方法等待某一阶段执行完毕。    如果当前阶段不等于指定的阶段或者该Phaser已经被终止，则立即返回。    该阶段数一般由arrive()方法或者arriveAndDeregister()方法返回。    返回下一阶段的序号，或者返回参数指定的值（如果该参数为负数），或者直接返回当前阶段序号（如果当前Phaser已经被终止）。
* awaitAdvanceInterruptibly(int phase) 效果与awaitAdvance(int phase)相当，    唯一的不同在于若该线程在该方法等待时被中断，则该方法抛出InterruptedException。
* awaitAdvanceInterruptibly(int phase, long timeout, TimeUnit unit)     效果与awaitAdvanceInterruptibly(int phase)相当，     区别在于如果超时则抛出TimeoutException。
* bulkRegister(int parties) 动态调整注册任务parties的数量。如果当前phaser已经被终止，则该方法无效，并返回负数。    如果调用该方法时，onAdvance方法正在执行，则该方法等待其执行完毕。    如果该Phaser有父Phaser则指定的party数大于0，且之前该Phaser的party数为0，那么该Phaser会被注册到其父Phaser中。
* forceTermination() 强制让该Phaser进入终止状态。    已经注册的party数不受影响。如果该Phaser有子Phaser，则其所有的子Phaser均进入终止状态。    如果该Phaser已经处于终止状态，该方法调用不造成任何影响。


**栗子**：3个线程，4个阶段，每个阶段都并发处理
```java
import java.util.concurrent.Phaser;

public class PhaserTest {
    public static void main(String[] args) {
        int parties = 3;
        int phases = 4;
        final Phaser phaser = new Phaser(parties) {
            @Override
            //每个阶段结束时
            protected boolean onAdvance(int phase, int registeredParties) {
                System.out.println("====== Phase : " + phase + "  end ======");
                return registeredParties == 0;
            }
        };
        for (int i = 0; i < parties; i++) {
            int threadId = i;
            Thread thread = new Thread(() -> {
                for (int phase = 0; phase < phases; phase++) {
                    if (phase == 0) {
                        System.out.println(String.format("第一阶段操作  Thread %s, phase %s", threadId, phase));
                    }
                    if (phase == 1) {
                        System.out.println(String.format("第二阶段操作  Thread %s, phase %s", threadId, phase));
                    }
                    if (phase == 2) {
                        System.out.println(String.format("第三阶段操作  Thread %s, phase %s", threadId, phase));
                    }
                    if (phase == 3) {
                        System.out.println(String.format("第四阶段操作  Thread %s, phase %s", threadId, phase));
                    }
         /**
          * arriveAndAwaitAdvance() 当前线程当前阶段执行完毕，等待其它线程完成当前阶段。
          * 如果当前线程是该阶段最后一个未到达的，则该方法直接返回下一个阶段的序号（阶段序号从0开始），
          * 同时其它线程的该方法也返回下一个阶段的序号。
          **/
                    int nextPhaser = phaser.arriveAndAwaitAdvance();

                }
            });
            thread.start();
        }
    }
}
```

