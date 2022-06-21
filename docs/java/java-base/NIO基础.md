## 一、NIO基础
Java New IO是从Java1.4版本开始引入的一个新的IO api，可以替代以往的标准IO，NIO相比原来的IO有同样的作用和目的，但是使用的方式完全不一样，NIO是面向缓冲区的，基于通道的IO操作，这也让它比传统IO有着更为高效的读写。

### 1.1 IO和NIO的主要区别

| IO    | NIO   |
| ----- | ----- |
| 面向流   | 面向缓冲  |
| 区阻塞IO | 非阻塞IO |
| 无     | 选择器   |


#### 1.1.1 传统IO的流
以下用图来简单理解一下，在传统IO中当App要对网络，磁盘中的文件进行读写的时候，它们必须建立一个连接，流到底是一个什么样的概念呢，我们可以先把它想象成自来水，家里要用自来水，需要有水管，让水从水管过来到家里，起到一个运输的作用。

所以当我们文件中的数据需要输入到App里面时，它们就会建立一个输入的管道。而当我们的App有数据需要写入到文件系统的时候，就会建立一个输出的管道，这两条管道就是我们的输入流和输出流。那水从来没有逆流而上的呀，所以它们都是单向管道。这么一讲，是不是就很好懂了呢😁？
![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151554588.png)






#### 1.1.2 NIO
也是同样的文件系统和App，不过此时把流换成了一个channel，现在我们可以先认为它就是一条铁道，那我们知道铁道本身是不能传递货物的呀，所以我们需要一个载具---火车（也就是缓冲区），App需要的数据就由这个名叫缓冲区的载具运输过来。那火车是可以开过来，也可以开回去的，所以NIO是双向传输的。
![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151605523.png)






### 1.2 Buffer
NIO的核心在于，通道（channel）和缓冲区（buffer）两个。通道是打开到IO设备的连接。使用时需要获取用于连接IO设备的通道以及用于容纳数据的缓冲区，然后通过操作缓冲区对数据进行处理。（其实就是上面那张图的事儿，或者一句话就是一个负责传输，一个负责存储）。

缓冲区是Java.nio包定义好的，所有缓冲区都是Buffer抽象类的子类。Buffer根据数据类型不同，常用子类分别是基本数据类型除了Boolean外的xxxBuffer（IntBuffer，DoubleBuffer···等）。不同的Buffer类它们的管理方式都是相同的，获取对象的方法都是
```
// 创建一个容量为capacity的xxx类型的Buffer对象
static xxxBuffer allocate(int capacity)
```

而且缓冲区提供了两个核心方法：get()和put()，put方法是将数据存入到缓冲区，而get是获取缓冲区的数据。

此时我们用代码看一下
```
public class BufferTest {
    @Test
    public void testBuffer(){
        // 创建缓冲区对象
        ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
    }
}
```
点进去ByteBuffer，会看到这个东西是继承了Buffer类的
```
public abstract class ByteBuffer extends Buffer implements Comparable<ByteBuffer>
```
此时继续点进去Buffer类，第一眼看到的是有几个自带的属性
![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151608558.png)




#### 1.2.1 buffer的基本属性
![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151609164.png)

① capacity容量
表示Buffer的最大数据容量，这个值不能为负。而且创建后是不能更改的。

② limit限制
第一个不能读取或写入的数据的索引，位于此索引后的数据不可读写。这个数值不能为负且不能超过capacity，如上图中第三个缓冲区，在下标为5之后的数据块均不能读写，那limit为5

③ position位置
下一个要读取或写入的数据的索引，这个数值不能为负且不能超过capacity，如图中第二个缓冲区，前面5块写完成，此时第6个数据块的下标为5，所以position为5

④ mark标记/reset重置
mark是一个索引，通过Buffer的mark()方法指定Buffer中一个特定的position后，可以通过reset()方法重置到这个position，这个通过代码来解释会比较好说明





#### 1.2.2 code部分（非常简单）

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151641663.png)

**1.首先我们创建一个缓冲区对象，然后把它的属性打印出来** 

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151642838.png)

```
ByteBuffer byteBuffer = ByteBuffer.allocate(10);
System.out.println(byteBuffer.position());
System.out.println(byteBuffer.capacity());
System.out.println(byteBuffer.limit());

运行结果：0,10,10
```

**2.执行一个put()方法，来把一个字符丢进去**

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151643829.png)

```
String str = "abcde";
byteBuffer.put(str.getBytes());
System.out.println(byteBuffer.position());
System.out.println(byteBuffer.capacity());
System.out.println(byteBuffer.limit());

运行结果：5,10,10
"abcde"长度为5，position已经变化，其它不变
```

**3.使用flip()切换为读模式**

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151643189.png)

```
byteBuffer.flip();
System.out.println(byteBuffer.position());
System.out.println(byteBuffer.capacity());
System.out.println(byteBuffer.limit());

运行结果：0,10,5
```
此时position变成为0了，因为一开始的5,是因为这时候要写的是下标为5的数据块，而转换成读模式后，第一个读的明显是下标为0的数据块呀。limit的数值也变成了5，因为当前能读到的数据从下标为5开始就木有了，所以limit为5

**4.简单获取一下buffer中的数据**
```
byte[] array = new byte[byteBuffer.limit()];
byteBuffer.get(array);
System.out.println(new String(array,0,array.length));

运行结果：abcde
```

**5.mark() & reset()**
```
byte[] array = new byte[byteBuffer.limit()];
byteBuffer.get(array,0,2);
System.out.println(new String(array,0,2));
System.out.println(byteBuffer.position());

byteBuffer.mark();
byteBuffer.get(array,2,2);
System.out.println(new String(array,2,2));
System.out.println(byteBuffer.position());

byteBuffer.reset();
System.out.println(byteBuffer.position());

运行结果：ab，2，cd，4，2
```

其实很简单，就是第一次读取的时候，只是读取了前面两个字符，然后此时position的结果为2，然后再读取后两个，position为4，可是因为我在读取前面2个的时候进行了一个mark操作，它就自动回到我mark之前的那个读取位置而已，就是这么简单

**6.其他的一些方法**
rewind()方法，可重复读，clear()清空缓冲区，不过这个方法的清空缓冲区，是一种被遗忘的状态，就是说，数据仍然还存于缓冲区中，可是自动忽略掉了。此时再次读取数据，是还是可以get()到的。hasRemaining()方法就是表示剩余可操作的数据量还有多少，比如刚刚的mark的那个例子中，我reset回去之后，剩余的可操作数据就是3，因为我只读了ab，还有cde这三个。




#### 1.2.3 直接缓冲区和非直接缓冲区
非直接缓冲区：通过allocate()方法来分配缓冲区。将缓冲区建立在JVM的内存中。
直接缓冲区：通过allocateDirect()方法分配缓冲区，将缓冲区建立在物理内存中。效率更高。

**① 非直接缓冲区**

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151644205.png)

应用程序想要在磁盘中读取数据时，首先它发起请求，让物理磁盘先把它的数据读到内核地址空间当中，之后这个内核空间再将这个数据copy一份到用户地址空间去。然后数据才能通过read()方法将数据返回个应用程序。而应用程序需要写数据进去，也是同理，先写到用户地址空间，然后copy到内核地址空间，再写入磁盘。此时不难发现，这个copy的操作显得十分的多余，所以非直接缓冲区的效率相对来说会低一些。



**② 直接缓冲区**

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151645809.png)

直接缓冲区就真的顾名思义非常直接了，写入的时候，写到物理内存映射文件中，再由它写入物理磁盘，读取也是磁盘把数据读到这个文件然后再由它读取到应用程序中即可。没有了copy的中间过程。





### 1.3 channel

#### 1.3.1 扯一下概念背景

由java.nio.channels包定义，表示IO源与目标打开的链接，它本身不存在直接访问数据的能力，只能和Buffer进行交互

传统的IO由cpu来全权负责，此时这个设计在有大量文件读取操作时，CPU的利用率会被拉的非常低，因为IO操作把CPU的资源都抢占了。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151646111.png)

在这种背景下进行了一些优化，把对cpu的连接取消，转为[DMA(直接内存存取)](https://baike.baidu.com/item/DMA/2385376?fr=aladdin)的方式。当然DMA这个操作本身也是需要CPU进行调度的。不过这个损耗自然就会比大量的IO要小的多。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151647799.png)

此时，就出现了通道这个概念，它是一个完全独立的处理器。专门用来负责文件的IO操作。

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151647732.png)



#### 1.3.2 常用通道

Java为Channel接口提供的主要实现类：

```
FileChannel：用于读取，写入，映射和操作文件的通道
DatagramChannel：通过UDP读写网络中的数据通道
SocketChannel：通过TCP读写网络中的数据通道
ServerSocketChannel：可以监听新进来的TCP连接，对每一个新进来的连接都会创建一个SocketChannel              
```

获取channel的一种方式是对支持通道的对象调用getChannel()方法，支持类如下
```     
FileInputStream
FileOutputStream
RandomAccessFile
DatagramSocket
Socket
ServerSocket
```
获取的其他方式是使用Files类的静态方法newByteChannel()获取字节通道。再或者是通过通道的静态方法open()打开并返回指定通道。



#### 1.3.3 常用方法和简单使用

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151648033.png)

**① 使用非直接缓冲区完成文件复制**
```
// 创建输入输出流对象
FileInputStream fileInputStream = new FileInputStream("testPic.jpg");
FileOutputStream fileOutputStream = new FileOutputStream("testPic2.jpg");

// 通过流对象获取通道channel
FileChannel inChannel = fileInputStream.getChannel();
FileChannel outChannel = fileOutputStream.getChannel();

// 创建指定大小的缓冲区
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

// 将通道中的数据写入到缓冲区中
while (inChannel.read(byteBuffer) != -1){

    // 切换成读取模式
    byteBuffer.flip();
    // 将缓冲区中的数据写到输出通道
    outChannel.write(byteBuffer);

    // 清空缓冲区
    byteBuffer.clear();

}
//回收资源(这里为了省时间直接抛出去了，反正这段不太重要)
outChannel.close();
inChannel.close();
fileInputStream.close();
fileOutputStream.close();

运行结果：就自然是复制了一个testPic2出来啦
```
因为代码本身不难，注释已经写得比较详细，就不展开了

**② 使用直接缓冲区来完成文件的复制**

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151649602.png)

注意这里的StandardOpenOption是一个枚举，表示模式，很显然这里是要选择READ读取模式。

```
FileChannel inChannel = FileChannel.open(Paths.get("testPic.jpg",StandardOpenOption.READ));
FileChannel outChannel = FileChannel.open(Paths.get("testPic2.jpg"),StandardOpenOption.WRITE,StandardOpenOption.READ,StandardOpenOption.CREATE);
// 进行内存映射
MappedByteBuffer inMappedBuffer = inChannel.map(FileChannel.MapMode.READ_ONLY, 0, inChannel.size());
MappedByteBuffer outMapBuffer = outChannel.map(FileChannel.MapMode.READ_WRITE, 0, inChannel.size());

// 对缓冲区进行数据的读写操作
byte[] array = new byte[inMappedBuffer.limit()];
inMappedBuffer.get(array);
outMapBuffer.put(array);

// 回收资源
inChannel.close();
outChannel.close();
```
如果需要看一下它们两个的时间差，自己用最常规的系统时间来瞧瞧就好，在这里就不再加上了。






## 二、NIO非阻塞式网络通信
传统的IO流都是阻塞式的，当一个线程调用read或者write时，该线程被阻塞，直到数据被读取或者写入，该线程在此期间都是不能执行其他任务的，因此，在完成网络通信进行IO操作时，线程被阻塞，所以服务器端必须为每个客户端提供一个独立线程进行处理，当服务器端需要处理大量客户端时，性能将会急剧下降。

NIO是非阻塞的，当线程从某通道进行读写数据时，若没有数据可用，该线程可以进行其他任务。线程通常将非阻塞IO的空闲时间用于在其他通道上执行IO操作，所以单独的线程可以管理多个输入和输出通道。因此NIO可以让服务器端使用一个或有限几个线程来同时处理连接到服务器端的所有客户端。




### 2.1 Selector
这个选择器其实就是在客户端和服务端之间引入一个通道的注册器，比如现在我的客户端要像服务端传输数据了，客户端会给选择器去发送一个channel的注册请求，注册完成后，Selector就会去监控这个channel的IO状态（读写，连接）。只有当通道中的数据完全准备就绪，Selector才会将数据分配到服务端的某个线程去处理。

这种非阻塞性的流程就可以更好地去使用CPU的资源。提高CPU的工作效率。这个可以用收快递来说明。如果你一开始就告诉我半小时后过来取快递，而我在这时候已经到目的地了，我有可能就原地不动站着等半个小时。这个期间啥地都去不了，可是你是到了之后，才打电话告诉我过来取，那我就有了更多的自由时间。




### 2.2 code（阻塞性IO的网络通信）
现在我们来演示一下阻塞性IO的网络通信

#### 2.2.1 client（阻塞性IO）
这个代码大家可以尝试这删除sChannel.shutdownOutput()，此时会发现在启动好server，运行client程序的时候，程序也会阻塞，这是因为这时服务端并无法确定你是否已经发送完成数据了，所以client端也产生了阻塞，双方就一直僵持。

还有一种方法是解阻塞，之后进行阐述。
```
// 1.获取通道
SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("你的IP地址",9898));
// 2.创建文件通道
FileChannel inChannel = FileChannel.open(Paths.get("C:/Users/Administrator/Desktop/testPic.jpg"),StandardOpenOption.READ);
// 3.分配指定大小的缓冲区
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

// 4.发送数据,需要读取文件
while (inChannel.read(byteBuffer) != -1){
    byteBuffer.flip();
    // 将buffer的数据写入到通道中
    sChannel.write(byteBuffer);
    byteBuffer.clear();
}

// 主动告诉服务端，数据已经发送完毕
sChannel.shutdownOutput();

while (sChannel.read(byteBuffer) != -1){
        byteBuffer.flip();
        System.out.println("接收服务端数据成功···");
        byteBuffer.clear();
    }

// 5.关闭通道
inChannel.close();
sChannel.close();
```

#### 2.2.2 server（阻塞性IO）
```
// 1.获取通道
ServerSocketChannel ssChannel = ServerSocketChannel.open();
// 创建一个输出通道，将读取到的数据写入到输出通道中，保存为testPic2
FileChannel outChannel = FileChannel.open(Paths.get("testPic2.jpg"),StandardOpenOption.WRITE,StandardOpenOption.CREATE);
// 2.绑定端口
ssChannel.bind(new InetSocketAddress(9898));
// 3.等待客户端连接，连接成功时会得到一个通道
SocketChannel sChannel = ssChannel.accept();
// 4.创建缓冲区
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
// 5.接收客户端的数据存储到本地
while (sChannel.read(byteBuffer) != -1){
    byteBuffer.flip();
    outChannel.write(byteBuffer);
    byteBuffer.clear();
}

// 发送反馈给客户端
    // 向缓冲区中写入应答信息
    byteBuffer.put("服务端接收数据成功".getBytes());
    byteBuffer.flip();
    sChannel.write(byteBuffer);

// 关闭通道
sChannel.close();
outChannel.close();
byteBuffer.clear();
```

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151650679.png)

然后再当我们的客户端运行起来，就会进行copy操作

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151651925.png)





### 2.3 Selector完成非阻塞IO
使用NIO完成网络通信需要三个核心对象：
channel：java.nio.channels.Channel接口，SocketChannel，ServerSocketChannel，DatagramChannel
管道相关：Pipe.SinkChannel，Pine.SourceChannel
buffer：负责存储数据
Selector：其中Selector是SelectableChannel的多路复用器，主要是用于监控SelectableChannel的IO状态

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151652247.png)

#### 2.3.1 client（非阻塞）

```
// 1.获取通道，默认是阻塞的
SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("192.168.80.1",9898));

// 1.1 将阻塞的套接字变成非阻塞
sChannel.configureBlocking(false);

// 2.创建指定大小的缓冲区
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
// 3.发送数据给服务端，直接将数据存储到缓冲区
byteBuffer.put(new Date().toString().getBytes());
// 4.将缓冲区的数据写入到sChannel
byteBuffer.flip();
sChannel.write(byteBuffer);
byteBuffer.clear();

// 关闭
sChannel.close();
```




#### 2.3.2 server（非阻塞）
代码的注释中已经解释了整个过程的做法，这里就不一一展开了。

```
// 1.获取通道
ServerSocketChannel ssChannel = ServerSocketChannel.open();
// 2.将阻塞的套接字设置为非阻塞的
ssChannel.configureBlocking(false);
// 3.绑定端口号
ssChannel.bind(new InetSocketAddress(9898));

// 4.创建选择器对象
Selector selector = Selector.open();

// 5.将通道注册到选择器上（这里的第二个参数为selectionKey），下面有解释
// 此时选择器就开始监听这个通道的接收时间，此时接收工作准备就绪，才开始下一步的操作
ssChannel.register(selector,SelectionKey.OP_ACCEPT);

// 6.通过轮询的方式获取选择器上准备就绪的事件
// 如果大于0，至少有一个SelectionKey准备就绪
while (selector.select() > 0){
    // 7.获取当前选择器中所有注册的selectionKey（已经准备就绪的监听事件）
    Iterator<SelectionKey> selectionKeyIterator = selector.selectedKeys().iterator();
    // 迭代获取已经准备就绪的选择键
    while (selectionKeyIterator.hasNext()){

        // 8.获取已经准备就绪的事件
        SelectionKey selectionKey = selectionKeyIterator.next();
        if (selectionKey.isAcceptable()){
            // 9.调用accept方法
            SocketChannel sChannel = ssChannel.accept();
            // 将sChannel设置为非阻塞
            // 再次强调，整个过程不能有任何一条阻塞通道
            sChannel.configureBlocking(false);

            // 进行数据接收工作，而且把sChannel也注册上选择器让选择器来监听
            sChannel.register(selector,SelectionKey.OP_READ);
        }else if (selectionKey.isReadable()){
            // 如果读状态已经准备就绪，就开始读取数据
            // 10.获取当前选择器上读状态准备就绪的通道
            SocketChannel sChannel = (SocketChannel) selectionKey.channel();
            // 11.读取客户端发送的数据，需要先创建缓冲区
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

            // 12.读取缓冲区的数据
            while (sChannel.read(byteBuffer) > 0){
                byteBuffer.flip();
                // 这里sChannel.read(byteBuffer)就是这个字节数组的长度
                System.out.println(new String(byteBuffer.array(),0,sChannel.read(byteBuffer)));

                // 清空缓冲区
                byteBuffer.clear();
            }
        }
        // 当selectionKey使用完毕需要移除，否则会一直优先
        selectionKeyIterator.remove();
    }

}
```

当调用register方法将通道注册到选择器时，选择器对通道的监听事件需要通过第二个参数ops决定

```
读：SelectionKey.OP_READ(1)
写：SelectionKey.OP_WRITE(4)
连接：SelectionKey.OP_CONNECT(8)
接收：SelectionKey.OP_ACCEPT(16)
```

若注册时不仅仅只有一个监听事件，则需要用位或操作符连接

`int selectionKeySet = SelectionKey.OP_READ|SelectionKey.OP_WRITE`



而关于这个selectionKey，它表示着SelectableChannel和Selectr之间的注册关系。它也有一系列对应的方法

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151659683.png)




#### 2.3.3 客户端的改造
引入Scanner接收输入信息，不过请注意，在测试代码中输入IDEA需要进行一些设置，具体做法是在Help-Edit Custom VM Option中加入一行
`-Deditable.java.test.console=true`

这样就可以输入了。

```
// 1.获取通道，默认是阻塞的
SocketChannel sChannel = SocketChannel.open(new InetSocketAddress("192.168.80.1",9898));

// 1.1 将阻塞的套接字变成非阻塞
sChannel.configureBlocking(false);

// 2.创建指定大小的缓冲区
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

Scanner scanner = new Scanner(System.in);
while (scanner.hasNext()){
    String str = scanner.next();
    // 3.发送数据给服务端，直接将数据存储到缓冲区
    byteBuffer.put((new Date().toString()+str).getBytes());
    // 4.将缓冲区的数据写入到sChannel
    byteBuffer.flip();
    sChannel.write(byteBuffer);
    byteBuffer.clear();
}
// 关闭
sChannel.close();
```
这样就完成了一个问答模式的网络通信。








### 2.4 Pipe管道
Java NIO中的管道是两个线程之间的单向数据连接，Pipe有一个source管道和一个sink管道，数据会被写到sink，从source中获取

![](https://raray-chuan.github.io/xichuan_blog_pic/img/202206151700211.png)

```
// 1.获取管道
Pipe pipe = Pipe.open();

// 2.创建缓冲区对象
ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
// 3.获取sink通道
Pipe.SinkChannel sinkChannel = pipe.sink();
byteBuffer.put("通过单向管道传输数据".getBytes());

// 4.将数据写入sinkChannel
byteBuffer.flip();
sinkChannel.write(byteBuffer);
// 5.读取缓冲区中的数据
Pipe.SourceChannel sourceChannel = pipe.source();
// 6.读取sourceChannel中的数据放入到缓冲区
byteBuffer.flip();
sourceChannel.read(byteBuffer);
System.out.println(new String(byteBuffer.array(),0,sourceChannel.read(byteBuffer)));

sourceChannel.close();
sinkChannel.close();
    
运行结果就是打印了我们的那串字符"通过单向管道传输数据"
```
