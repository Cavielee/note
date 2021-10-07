# IO 区别

IO 操作，即输入（读）输出（写）操作。

以下例子均以读为例：

## 操作系统层面

进程发起读请求，内核收到请求后，会将硬件设备数据拷贝到内核空间中，再将内核空间的数据拷贝进程的空间中。

![操作系统的IO](C:\Users\63190\Desktop\pics\操作系统的IO.png)



### BIO

Blocking IO——同步阻塞IO

![操作系统的BIO](C:\Users\63190\Desktop\pics\操作系统的BIO.png)

进程发起请求后会被挂起，阻塞等待，直到数据拷贝到进程内存空间后，才能继续处理。

### NIO

Non-blocking IO——非阻塞IO

![操作系统的NIO](C:\Users\63190\Desktop\pics\操作系统的NIO.png)

进程发起请求后，如果数据未准备好，进程不会阻塞等待，而是直接返回。

进程会不断主动轮询数据是否准备好（轮询期间可以处理其他事情），当数据准备好后，会再次发起读请求，将内核空间的数据拷贝到进程内存空间（该过程中实际还是会阻塞）。

### 多路复用IO

实际上也是一种非阻塞 IO 模型。

![操作系统的多路复用IO](C:\Users\63190\Desktop\pics\操作系统的多路复用IO.png)

进程通过操作系统的多路复用模型注册一个 IO 事件，并监听该事件。

进程期间可以处理其他事情，直到数据准备好后，内核会触发事件通知进程，进程会发起读请求，将内核空间的数据拷贝到进程内存空间（该过程中会阻塞）。

### 异步IO

Aynchronous IO——异步非阻塞IO

![操作系统的AIO](C:\Users\63190\Desktop\pics\操作系统的AIO.png)

进程发起异步读请求后直接返回，当内核将数据拷贝到进程的内存空间后通知进程可以直接使用数据。

### 阻塞和非阻塞

阻塞和非阻塞实际可以理解为进程进程是否需要等待数据准备好（硬件设备拷贝到内核空间）。

### 同步和异步

同步和异步实际可以理解为是否 IO 结果由谁处理。

同步：由进程去主动获取，如 NIO 需要主动轮询获取；多路复用IO 需要收到通知后主动去获取。

异步：由内核将数据拷贝到进程内存空间后，进程直接使用。



## Java 层面

以网络 IO 为例：

### BIO

1. Socket 服务端调用 accept() 会阻塞等待客户端发起 Socket 连接。
2. 接受到客户端连接请求后，进行相关处理，此时服务端会阻塞等待处理执行完后才能响应其他客户端 Socket 请求。

```java
public static void main(String[] args) throws IOException {
    int port = 8080;
    ServerSocket server = null;
    try {
        // 创建服务端 ServerSocket
        server = new ServerSocket(port);
        Socket socket = null;

        while (true) {
            // 阻塞接受客戶端请求接入
            socket = server.accept();
            // IO处理
            out = new PrintWriter(socket.getOutputStream(), true);
        }
    } finally {
        // 关闭资源
        ...
    }

}
```

### 伪异步 IO

伪异步 IO是对 BIO 的优化。

对于 Socket 请求处理交由线程去执行，使得 Socket 请求处理不会影响 Socket 请求接入。

```java
public static void main(String[] args) throws IOException {
    int port = 8080;
    ServerSocket server = null;
    try {
        // 创建服务端 ServerSocket
        server = new ServerSocket(port);
        Socket socket = null;

        while (true) {
            // 阻塞接受客戶端请求接入
            socket = server.accept();
            // 创建线程去处理 Socket 请求
            new Thread(new xxxHandler(socket)).start();
        }
    } finally {
        // 关闭资源
        ...
    }

}
```

缺陷：

* accept() 还是阻塞的
* 如果为每一个 Socket 请求创建一个线程处理，当高并发请求时，将会耗尽线程资源。
* 如果采用线程池，则当线程内处理过程十分慢，将会导致线程池线程被全部占用而无法响应其他客户端的 Socket 请求（请求超时），导致服务不可用。

### NIO

实际上是一个多路复用 IO模型。其主要有三个组件：

#### 缓冲区 Buffer

Buffer 实际上是一个数组。

在 NIO 进行读写时都是对缓冲区进行处理。在读取数据时，它是直接读到缓冲区中的；在写入数据时，它也是写入到缓冲区中的。

Buffer 抽象对象有不同的实现，如最常用的 ByteBuffer，实际上根据操作的基本数据类型实现不同的 Buffer 子类。

##### 缓冲区状态变量

Buffer 提供以下变量记录缓冲区内部状态的变化。

| 字段     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 指定了可以存储在缓冲区中的最大数据容量，实际上为底层数组的大小或者是可以使用底层数组的容量。在缓冲区创建时就被设定并且不能改变。 |
| Limit    | 指定还有多少数据需要取出(在从缓冲区写入通道时)，或者还有多少空间可以放入数据(在从通道读入缓冲区时)。 |
| Position | 指定下一个将要被写入或者读取的元素索引，它的值由get() / put()方法自动更新，在新创建一个Buffer 对象时，position 被初始化为0。 |

三个属性存在相对大小关系：0 <= position <= limit <= capacity。

```
FileInputStream fin = new FileInputStream("E://test.txt");
//创建文件的操作管道
FileChannel fc = fin.getChannel();
// 1.分配一个10 个大小缓冲区，说白了就是分配一个10 个大小的byte 数组
ByteBuffer buffer = ByteBuffer.allocate(10);
// 2.从通道中读取数据，实际上就是将读取的数据写入到 buffer 中
fc.read(buffer);
// 3.准备操作之前，先锁定操作范围
buffer.flip();
// 4.判断有没有可读数据
while (buffer.remaining() > 0) {
	// 5.从通道中读取数据，实际上就是将读取的数据写入到 buffer 中
	byte b = buffer.get();
}
//可以理解为解锁
buffer.clear();
```



客户端可以不停的往缓冲区写数据，不需要等待服务端处理完成后才能继续写，服务端只要检查到缓冲区数据准备好了，就会安排处理数据（触发一个可读/可写的事件到 Selector 上，由 Selector 轮询进行处理），不需要阻塞等待数据写完。因此可以在空闲的时候处理其他请求。

#### 通道 Channel

BIO 通信是通过流的形式（InputStream、OutputStream），每次通信只能读或者写，如读取 read() 会阻塞等待，直到读取完流中所有数据才能处理下一次读写。

NIO 则通过 Channel 的形式，实际上是对 InputStream 和 OutputStream的模拟。Channel 的读是通过将数据读取到缓冲区 Buffer，然后程序直接读取 Buffer 当前可读数据；程序写入数据，只需要将数据写入到 Buffer，Channel 的写会将 Buffer 的数据发送。

因此读取时，程序只需读取当前 Buffer 可读的数据，不需要像 BIO 那样阻塞等待数据全部从流中读取完；写入时，程序只需要把数据写入到 Buffer 中，不需要像 BIO 一样阻塞等待数据全部写入完到流中。

> 上述阻塞等待过程可以理解为：流数据在网络传输需要耗时。

BIO 中客户端与服务端通信，需要分别使用 InputStream、OutputStream 流。

客户端和服务端通过 Channel 进行通信，每次进行读写需要通过 Channel，直到通信结束，通道就会销毁。

Channel 包括以下类型：

- FileChannel：从文件中读写数据；
- DatagramChannel：通过 UDP 读写网络中数据；
- SocketChannel：通过 TCP 读写网络中数据；
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个  SocketChannel。



#### 多路复用器 Selector

对于每一个客户端的连接都会创建一个 Channel，并将 Channel 会注册到 Selector，需要声明任何操作都是以事件作为驱动，如客户端连接上了服务端就会触发 SelectionKey.OP_CONNECT 事件，

实际上可以理解为一个单线程，不断轮询

-  （实际类似于一个线程）， 不断的轮询各个通道，看是否有通道准备好（可连接、可读、可写等事件），提高了响应，并把请求任务分发给处理线程。不会出现 BIO 那样，如果同一时间客户端请求过多要等待服务端处理完才能接受新的请求。NIO 统一的把所有连接请求作为事件注册到 Selector 上，使得请求不会阻塞在外面，接受所有的接入请求，并作为事件有序（会为每个事件派发一个 SelectionKey）的处理。（同步非阻塞）
- 

#### 非阻塞体现

1. 读取/写入都是面向缓冲区 Buffer，不需要阻塞等待流传输过程。
2. 服务端不需要阻塞等待一个客户端请求处理完后才能接收下一个客户端请求，通过 Selector 可以接收多个客户端请求，多个客户端可以不停的往缓冲区写入，当缓冲区数据准备好了，会注册一个事件（可读、可写等）到 Selector，服务端收到后就会进行处理。



### AIO

服务端：AsynchronousServerSocketChannel
客服端：AsynchronousSocketChannel
用户处理器：CompletionHandler 接口，这个接口实现应用程序向操作系统发起 IO 请求，操作系统处理完后回调方法进行具体逻辑处理。否则做自己该做的事情，



在 IO 多路复用模型中，事件循环将文件句柄的状态事件通知给用户线程，由用户线程去读取数据、处理数据。

而在异步IO模型（Proactor设计模式）中，当用户线程收到通知时，数据已经被内核读取完毕，并放在了用户线程指定的缓冲区内，内核在IO完成后通知用户线程直接使用即可。

```java
/**
* AIO 服务端
*/
public class AIOServer {
    private final int port;
    public static void main(String args[]) {
        int port = 8000;
        new AIOServer(port);
    }
    public AIOServer(int port) {
        this.port = port;
        listen();
    }
    private void listen() {
        try {
            ExecutorService executorService = Executors.newCachedThreadPool();
            AsynchronousChannelGroup threadGroup = AsynchronousChannelGroup.withCachedThreadPool(executorService, 1);
            final AsynchronousServerSocketChannel server = AsynchronousServerSocketChannel.open(threadGroup);
            server.bind(new InetSocketAddress(port));
            System.out.println("服务已启动，监听端口" + port);
            server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Object>(){
                final ByteBuffer buffer = ByteBuffer.allocateDirect(1024);
                public void completed(AsynchronousSocketChannel result, Object attachment){
                    System.out.println("IO 操作成功，开始获取数据");
                    try {
                        buffer.clear();
                        result.read(buffer).get();
                        buffer.flip();
                        result.write(buffer);
                        buffer.flip();
                    } catch (Exception e) {
                        System.out.println(e.toString());
                    } finally {
                        try {
                            result.close();
                            server.accept(null, this);
                        } catch (Exception e) {
                            System.out.println(e.toString());
                        }
                    }
                    System.out.println("操作完成");
                }
                @Override
                public void failed(Throwable exc, Object attachment) {
                    System.out.println("IO 操作是失败: " + exc);
                }
            });
            try {
                Thread.sleep(Integer.MAX_VALUE);
            } catch (InterruptedException ex) {
                System.out.println(ex);
            }
        } catch (IOException e) {
            System.out.println(e);
        }
    }
}

/**
* AIO 客户端
*/
public class AIOClient {
    private final AsynchronousSocketChannel client;
    public AIOClient() throws Exception{
        client = AsynchronousSocketChannel.open();
    }
    public void connect(String host,int port)throws Exception{
        client.connect(new InetSocketAddress(host,port),null,new CompletionHandler<Void,Void>() {
            @Override
            public void completed(Void result, Void attachment) {
                try {
                    client.write(ByteBuffer.wrap("这是一条测试数据".getBytes())).get();
                    System.out.println("已发送至服务器");
                } catch (Exception ex) {
                    ex.printStackTrace();
                }
            }
            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });
        final ByteBuffer bb = ByteBuffer.allocate(1024);
        client.read(bb, null, new CompletionHandler<Integer,Object>(){
            @Override
            public void completed(Integer result, Object attachment) {
                System.out.println("IO 操作完成" + result);
                System.out.println("获取反馈结果" + new String(bb.array()));
            }
            @Override
            public void failed(Throwable exc, Object attachment) {
                exc.printStackTrace();
            }
        }
                   );
        try {
            Thread.sleep(Integer.MAX_VALUE);
        } catch (InterruptedException ex) {
            System.out.println(ex);
        }
    }
    public static void main(String args[])throws Exception{
        new AIOClient().connect("localhost",8000);
    }
}
```

可以看到当 AIOChannel 没有收到自己关注的事件通知时会继续处理自己逻辑，当收到后操作系统会通知并回调指定 CompletionHandler 的处理逻辑进行处理。