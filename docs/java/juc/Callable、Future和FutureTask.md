## 一、Callable 与 Runnable
先说一下java.lang.Runnable吧，它是一个接口，在它里面只声明了一个run()方法：
```java
public interface Runnable {
    public abstract void run();
}
```
由于run()方法返回值为void类型，所以在执行完任务之后无法返回任何结果。

Callable位于java.util.concurrent包下，它也是一个接口，在它里面也只声明了一个方法，只不过这个方法叫做call()：
```java
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     *
     * @return computed result
     * @throws Exception if unable to compute a result
     */
    V call() throws Exception;
}
```
可以看到，这是一个泛型接口，该接口声明了一个名称为call()的方法，同时这个方法可以有返回值V，也可以抛出异常。call()方法返回的类型就是传递进来的V类型。

那么怎么使用Callable呢？一般情况下是配合ExecutorService来使用的，在ExecutorService接口中声明了若干个submit方法的重载版本：
```java
<T> Future<T> submit(Callable<T> task);    //submit提交一个实现Callable接口的任务，并且返回封装了异步计算结果的Future。
<T> Future<T> submit(Runnable task, T result);  //submit提交一个实现Runnable接口的任务，并且指定了在调用Future的get方法时返回的result对象。
Future<?> submit(Runnable task);  //submit提交一个实现Runnable接口的任务，并且返回封装了异步计算结果的Future。
```
因此我们只要创建好我们的线程对象（实现Callable接口或者Runnable接口），然后通过上面3个方法提交给线程池去执行即可。

明显能看到区别：
1.Callable能接受一个泛型，然后在call方法中返回一个这个类型的值。而Runnable的run方法没有返回值
2.Callable的call方法可以抛出异常，而Runnable的run方法不会抛出异常。




**补充：**
实现Runnable接口和实现Callable接口的区别：
1、Runnable是自从java1.1就有了，而Callable是1.5之后才加上去的。
2、Callable规定的方法是call(),Runnable规定的方法是run()。
3、Callable的任务执行后可返回值，而Runnable的任务是不能返回值(是void)。
4、call方法可以抛出异常，run方法不可以。
5、运行Callable任务可以拿到一个Future对象，表示异步计算的结果。它提供了检查计算是否完成的方法，以等待计算的完成，并检索计算的结果。通过Future对象可以了解任务执行情况，可取消任务的执行，还可获取执行结果。
6、加入线程池运行，Runnable使用ExecutorService的execute方法，Callable使用submit方法。





## 二、Future
    Future就是对于具体的Runnable或者Callable任务的执行结果进行取消、查询是否完成、获取结果。必要时可以通过get方法获取执行结果，该方法会阻塞直到任务返回结果。
    Future<V>接口是用来获取异步计算结果的，说白了就是对具体的Runnable或者Callable对象任务执行的结果进行获取(get())，取消(cancel())，判断是否完成等操作。我们看看Future接口的源码：
```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

在Future接口中声明了5个方法，下面依次解释每个方法的作用：
  **cancel**方法用来取消任务，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
  **isCancelled**方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
  **isDone**方法表示任务是否已经完成，若任务完成，则返回true；
  **get()**方法用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
  **get(long timeout, TimeUnit unit)**用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。

也就是说Future提供了三种功能：
1）判断任务是否完成；
2）能够中断任务；
3）能够获取任务执行结果。
因为Future只是一个接口，所以是无法直接用来创建对象使用的，因此就有了下面的FutureTask。


**例子： submit(Callable task)**
```java
public class Main {
　　public static void main(String[] args) throws InterruptedException, ExecutionException {
　　ExecutorService executor = Executors.newFixedThreadPool(2);
　　//创建一个Callable，3秒后返回String类型
　　Callable myCallable = new Callable() {
　　　　@Override
　　　　public String call() throws Exception {
　　　　　　Thread.sleep(3000);
　　　　　　System.out.println("calld方法执行了");
　　　　　　return "call方法返回值";
　　　　}
　　};
　　System.out.println("提交任务之前 "+getStringDate());
　　Future future = executor.submit(myCallable);
　　System.out.println("提交任务之后，获取结果之前 "+getStringDate());
　　System.out.println("获取返回值: "+future.get());
　　System.out.println("获取到结果之后 "+getStringDate());
　　}
　　public static String getStringDate() {
　　　　Date currentTime = new Date();
　　　　SimpleDateFormat formatter = new SimpleDateFormat("HH:mm:ss");
　　　　String dateString = formatter.format(currentTime);
　　　　return dateString;
　　　　}
　　}
```
通过executor.submit提交一个Callable，返回一个Future，然后通过这个Future的get方法取得返回值。
看一下输出：
```
提交任务之前 12:13:01
提交任务之后，获取结果之前 12:13:01
calld方法执行了
获取返回值: call方法返回值
获取到结果之后 12:13:04
```


**例子：submit(Runnable task)**
因为Runnable是没有返回值的，所以如果submit一个Runnable的话，get得到的为null：
```java
Runnable myRunnable = new Runnable() {
　　@Override
　　public void run() {
　　try {
　　　　Thread.sleep(2000);
　　　　System.out.println(Thread.currentThread().getName() + " run time: " + System.currentTimeMillis());
　　} catch (InterruptedException e) {
　　　　e.printStackTrace();
　　}
　　}
};

Future future = executor.submit(myRunnable);
System.out.println("获取的返回值： "+future.get());
```
输出为：
```
pool-1-thread-1 run time: 1493966762524
获取的返回值： null
```



## 三、FutureTask
我们先来看一下FutureTask的实现：
```java
public class FutureTask<V> implements RunnableFuture<V>
```
FutureTask类实现了RunnableFuture接口，我们看一下RunnableFuture接口的实现：
```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
可以看出RunnableFuture继承了Runnable接口和Future接口，而FutureTask实现了RunnableFuture接口。所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。

**分析：**
FutureTask除了实现了Future接口外还实现了Runnable接口，因此FutureTask也可以直接提交给Executor执行。 当然也可以调用线程直接执行（FutureTask.run()）。

接下来我们根据FutureTask.run()的执行时机来分析其所处的3种状态：
（1）未启动，FutureTask.run()方法还没有被执行之前，FutureTask处于未启动状态，当创建一个FutureTask，而且没有执行FutureTask.run()方法前，这个FutureTask也处于未启动状态。
（2）已启动，FutureTask.run()被执行的过程中，FutureTask处于已启动状态。
（3）已完成，FutureTask.run()方法执行完正常结束，或者被取消或者抛出异常而结束，FutureTask都处于完成状态。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206161442473.png)



下面我们再来看看FutureTask的方法执行示意图（方法和Future接口基本是一样的，这里就不过多描述了）

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206161443618.png)

分析：
（1）当FutureTask处于未启动或已启动状态时，如果此时我们执行FutureTask.get()方法将导致调用线程阻塞；当FutureTask处于已完成状态时，执行FutureTask.get()方法将导致调用线程立即返回结果或者抛出异常。
（2）当FutureTask处于未启动状态时，执行FutureTask.cancel()方法将导致此任务永远不会执行。当FutureTask处于已启动状态时，执行cancel(true)方法将以中断执行此任务线程的方式来试图停止任务，如果任务取消成功，cancel(...)返回true；但如果执行cancel(false)方法将不会对正在执行的任务线程产生影响(让线程正常执行到完成)，此时cancel(...)返回false。当任务已经完成，执行cancel(...)方法将返回false。


最后我们给出FutureTask的两种构造函数：
```java
public FutureTask(Callable<V> callable) {
}
public FutureTask(Runnable runnable, V result) {
}
```
事实上，FutureTask是Future接口的一个唯一实现类。


示例：
```java
package com.demo.test;

import java.util.concurrent.Callable;

public class Task implements Callable<Integer>{
    
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        return sum;
    }
}
```
```java
package com.demo.test;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.FutureTask;

public class CallableTest1 {
    
    public static void main(String[] args) {
        //第一种方式
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        executor.submit(futureTask);
        executor.shutdown();
         
        //第二种方式，注意这种方式和第一种方式效果是类似的，只不过一个使用的是ExecutorService，一个使用的是Thread
//        Task task = new Task();
//        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
//        Thread thread = new Thread(futureTask);
//        thread.start();
        
        
         //第三种，使用Callable+Future获取执行结果
 //        ExecutorService executor = Executors.newCachedThreadPool();
        //创建Callable对象任务  
 //       Task task = new Task();
        //提交任务并获取执行结果  
//        Future<Integer> result = executor.submit(task);
        //关闭线程池  
//        executor.shutdown();         
       
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }
         
        System.out.println("主线程在执行任务");
         
        try {
            if(futureTask.get()!=null){  
                System.out.println("task运行结果"+futureTask.get());
            }else{
                System.out.println("future.get()未获取到结果"); 
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }
         
        System.out.println("所有任务执行完毕");
    }
}
```
运行结果：
```
子线程在进行计算
主线程在执行任务
task运行结果4950
所有任务执行完毕
```

