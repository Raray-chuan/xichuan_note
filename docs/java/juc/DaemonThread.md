在Java中有两类线程，分别是User Thread（用户线程）和Daemon Thread（守护线程） 。
用户线程很好理解，我们日常开发中编写的业务逻辑代码，运行起来都是一个个用户线程。而守护线程相对来说则要特别理解一下。

## 1.1 什么是守护线程
在操作系统里面是没有所谓的守护线程的概念的，只有守护进程一说。但是Java语言机制是构建在JVM的基础之上的，这一机制意味着Java平台是把操作系统的底层给屏蔽了起来，所以它可以在它自己的虚拟的平台里面构造出对自己有利的机制。而Java语言或者说平台的设计者多多少少是收到Unix操作系统思想的影响，而守护线程机制又是对JVM这样的平台凑合，于是守护线程应运而生。

所谓的守护线程，指的是程序运行时在后台提供的一种通用服务的线程。比如垃圾回收线程就是一个很称职的守护者，并且这种线程并不属于程序中不可或缺的部分。因此，当所有的非守护线程结束时，程序也就终止了，同时会杀死进程中的所有守护线程。反过来说，只要任何非守护线程还在运行，程序就不会终止。

事实上，User Thread（用户线程）和Daemon Thread（守护线程）从本质上来说并没有什么区别，唯一的不同之处就在于虚拟机的离开：如果用户线程已经全部退出运行了，只剩下守护线程存在了，虚拟机也就退出了。 因为没有了被守护者，守护线程也就没有工作可做了，也就没有继续运行程序的必要了。




## 1.2 守护线程的使用与注意事项
守护线程并非只有虚拟机内部可以提供，用户也可以手动将一个用户线程设定/转换为守护线程。

在Thread类中提供了一个setDaemon(true)方法来将一个普通的线程（用户线程）设置为守护线程。
```java
public final void setDaemon(boolean on);
```

在使用的过程中，有几点需要注意：
1.thread.setDaemon(true)必须在thread.start()之前设置，否则会抛出一个IllegalThreadStateException异常。这也就意味着不能把正在运行的常规线程设置为守护线程。 这点与操作系统中的守护进程有着明显的区别，守护进程是创建后，让进程摆脱原会话的控制+让进程摆脱原进程组的控制+让进程摆脱原控制终端的控制；所以说寄托于虚拟机的语言机制跟系统级语言有着本质上面的区别。

2.在Daemon线程中产生的新线程也是Daemon的。关于这一点又是与操作系统中的守护进程有着本质的区别：守护进程fork()出来的子进程不再是守护进程，尽管它把父进程的进程相关信息复制过去了，但是子进程的进程的父进程不是init进程，所谓的守护进程本质上说就是，当父进程挂掉，init就会收养该进程，然后文件0、1和2都是/dev/null，当前目录到/。

3.不是所有的应用都可以分配给Daemon线程来进行服务的，比如读写操作或者计算逻辑。因为这种应用可能在Daemon Thread还没来得及进行操作时，虚拟机已经退出了。这也就意味着，守护线程应该永远不去访问固有资源，如文件、数据库，因为它会在任何时候甚至在一个操作的中间发生中断。


下面以一个完成文件输出的守护线程任务作为例子：
```java
import java.io.*;  

class TestRunnable implements Runnable {
    public void run(){
        try {
            Thread.sleep(1000); // 守护线程阻塞1秒后运行  
            File f = new File("daemon.txt");
            FileOutputStream os = new FileOutputStream(f,true);
            os.write("daemon".getBytes());
        } catch(IOException e1) {  
            e1.printStackTrace();  
        } catch(InterruptedException e2) {  
            e2.printStackTrace();  
        }  
    }  
}  

public class TestDemo2 {
    public static void main(String[] args) throws InterruptedException {
        Runnable tr = new TestRunnable();
        Thread thread = new Thread(tr);
        thread.setDaemon(true); // 设置守护线程（必须在thread.start()之前）
        thread.start(); // 开始执行分进程
    }
}
```
上面这段代码的运行结果是文件daemon.txt中没有daemon字符串。
但是如果把thread.setDaemon(true);这行代码注释掉，文件daemon.txt是可以被写入daemon字符串的，因为这个时候这个线程就是普通的用户线程了。
简单理解就是，JRE判断程序是否执行结束的标准是所有的前台线程（用户线程）执行完毕了，而不管后台线程（守护线程）的状态。




## 1.3 守护线程的应用场景
前面说了那么多，那么Daemon Thread的实际应用在那里呢？举个例子，Web服务器中的Servlet，在容器启动时，后台都会初始化一个服务线程，即调度线程，负责处理http请求，然后每个请求过来，调度线程就会从线程池中取出一个工作者线程来处理该请求，从而实现并发控制的目的。也就是说，一个实际应用在Java的线程池中的调度线程。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206171143376.png)



## 1.4 总结
从我的理解，守护线程就是用来告诉JVM，我的这个线程是一个低级别的线程，不需要等待它运行完才退出，让JVM喜欢什么时候退出就退出，不用管这个线程。
在日常的业务相关的CRUD开发中，其实并不会关注到守护线程这个概念，也几乎不会用上。
但是如果要往更高的地方走的话，这些深层次的概念还是要了解一下的，比如一些框架的底层实现。