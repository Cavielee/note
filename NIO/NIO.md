### 阻塞与非阻塞

阻塞和非阻塞是事件操作时，结果是否完成才返回。

例如煮水这一个事件：

- 阻塞下意味着要等待水煮沸（事件完成并返回结果）才能做其他事情
- 非阻塞下意味着执行煮水后，就可以继续执行其他事情，不必等待水煮沸这一个结果返回。（一般可以主动去轮询查看结果是否完成）



### 同步和异步

同步和异步都是基于应用程序和操作系统处理 IO 事件所采用的方式。

- 同步是应用程序直接参与 IO 读写的操作。（同一个线程处理，每次处理都是依次一个一个处理）
- 异步是所有的 IO 读写交给操作系统去处理，应用程序只需要被动等待通知。（分发给其他线程处理，多线程形式并发处理）

例如：

- 同步处理两件事情，则需要该线程依照顺序依次执行。
- 异步则使用多线程，把事情扔给子线程去处理，主线程继续处理其他事情。当子线程完成后，会通知主线程去获取结果。

### NIO 和 BIO 区别

| IO模型 | IO               | BIO                        |
| ------ | ---------------- | -------------------------- |
| 方式   | 从硬盘到内存     | 从内存到硬盘               |
| 通信   | 面向流           | 面向缓冲                   |
| 处理   | 阻塞IO（多线程） | 非阻塞IO（反应堆 Reactor） |
| 触发   | 无               | 选择器（轮询机制）         |



### BIO

#### 传统 BIO 流程

在while循环中服务端会调用 accept 方法等待接收客户端的连接请求，一旦接收到一个连接请求，就会将 Socket 扔给一个新的线程处理（对 Socket 进行 read() 以及进行 write()）



#### 伪异步 BIO 流程（传统的改进）

由于每一个请求接入都用一个新的线程处理，这样会导致线程资源大量消耗，甚至可能导致宕机等情况。因此为了控制线程资源，采用了线程池的概念，给定指定数量的线程去处理 Socket，并且指定消息队列去存储等待线程处理的 Socket。



#### 缺点

1. （传统 BIO）由于每建立一次请求就会为其创建一个线程去处理 Socket。当存在海量并发客户端请求接入时，会导致线程资源耗尽（Java 中线程资源很重要）。
2. 同一个用户的每次请求都需要打开和关闭 IO 流（进行读写），降低了效率。
3. （伪异步 BIO）在 read() 方法中，会阻塞读取，直至读取完毕/异常。由于发送者发送缓慢、网络传输缓慢、读取者读取缓慢等原因，将会导致阻塞时间变长，因此会导致线程占用时间长，其他的接入请求不断地添加到消息队列中等待。**并且当消息队列填满时，会拒绝客户端接入请求，客户端会发生大量的连接超时。**
4. （伪异步 BIO）在 write() 方法中，会阻塞写入，直至所有要发送的字节写入完毕/异常。由于接收方处理缓慢，可能会导致 TCP 缓冲区填满，而发送方只能等待 TCP 缓冲区有空余位置才能继续写入。



#### 例子（获取时间服务）

##### Client

缺点：

- Listener 会阻塞在 read() 方法，直到服务端发来信息。

```java
public class BIOTimeClient {

	public static void main(String[] args) {
		int port = 8081;
		Socket socket = null;
		BufferedReader in = null;
		PrintWriter out = null;

		try {
			socket = new Socket("127.0.0.1", 8080);
			in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
			out = new PrintWriter(socket.getOutputStream(), true);

			out.println("QUERY TIME ORDER");
			System.out.println("Send order 2 succeed.");
			String resp = in.readLine();
			System.out.println("Now is : " + resp);
		} catch (Exception e) {
			e.printStackTrace();
		} finally {
			// 资源关闭
			if (in != null) {
				try {
					in.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
				in = null;
			}

			if (out != null) {
				out.close();
				out = null;
			}

			if (socket != null) {
				try {
					socket.close();
				} catch (IOException e) {
					e.printStackTrace();
				}
				socket = null;
			}
		}
	}

}

```

##### Server（传统BIO）

缺点：

- accept() 方法会阻塞等待，直到有客户端连接。
- 如果线程池用完，则必须等待有空闲线程池才能处理请求。
- 处理过程中，read() 方法会阻塞直到客户端有信息发送，才能往下执行。

```java
public class BIOTimeServer {

	public static void main(String[] args) throws IOException {
		int port = 8080;
		ServerSocket server = null;

		try {
			// 创建服务端 ServerSocket
			server = new ServerSocket(port);
			System.out.println("The time server is start in port :" + port);
			Socket socket = null;

			while (true) {
				// 阻塞接受客戶端请求接入
				socket = server.accept();
				// 创建线程处理 Socket 请求
				new Thread(new BIOTimeServerHandler(socket)).start();
			}
		} finally {
			if (server != null) {
				System.out.println("The time server close");
				server.close();
				server = null;
			}
		}

	}

}

/**
 * 处理逻辑
 * 
 * @author created by Cavielee
 * @date 2018年12月24日 下午6:31:14
 */
public class BIOTimeServerHandler implements Runnable {

	private Socket socket;

	public BIOTimeServerHandler(Socket socket) {
		super();
		this.socket = socket;
	}

	public void run() {
		BufferedReader in = null;
		PrintWriter out = null;

		try {
			in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
			out = new PrintWriter(socket.getOutputStream(), true);

			String currentTime = null;
			String body = null;

			while (true) {
				body = in.readLine();
				if (body == null) {
					break;

				}
				System.out.println("The time server receive order : " + body);
				currentTime = "QUERY TIME ORDER".equals(body) ? new Date(System.currentTimeMillis()).toString()
						: "BAD ORDER";
				out.println(currentTime);
			}
		} catch (Exception e) {
			// 关闭资源
			if (in != null) {
				try {
					in.close();
				} catch (IOException e1) {
					e1.printStackTrace();
				}
			}
			if (out != null) {
				out.close();
				out = null;
			}
			if (socket != null) {
				try {
					socket.close();
				} catch (IOException e1) {
					e1.printStackTrace();
				}
				socket = null;
			}
		}

	}

}


```



### NIO

同步非阻塞式IO，采用了事件驱动的思想来实现了一个多路复用器（Selector）。 

![1545618546998](C:\Users\37\AppData\Roaming\Typora\typora-user-images\1545618546998.png)

#### 优点

- 创建通道 Channel，通过通道（双向的）可以进行读写，读写完后通道归还，不会销毁（一个 SocketChannel 代表客户端和服务端的交互通道，当交互结束时，通道就会销毁）。
- 多路复用器 Selector （实际类似于一个线程）， 不断的轮询各个通道，看是否有通道准备好（可连接、可读、可写等事件），提高了响应，并把请求任务分发给处理线程。不会出现 BIO 那样，如果同一时间客户端请求过多要等待服务端处理完才能接受新的请求。NIO 统一的把所有连接请求作为事件注册到 Selector 上，使得请求不会阻塞在外面，接受所有的接入请求，并作为事件有序（会为每个事件派发一个 SelectionKey）的处理。（同步非阻塞）
- 客户端可以不停的往缓冲区写数据，不需要等待服务端处理完成后才能继续写，服务端只要检查到缓冲区数据准备好了，就会安排处理数据（触发一个可读/可写的事件到 Selector 上，由 Selector 轮询进行处理），不需要阻塞等待数据写完。因此可以在空闲的时候处理其他请求。（非阻塞）

#### 阻塞与非阻塞 IO

Java IO 的各种流是阻塞的。这意味着，当一个线程调用 read() 或 write() 时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。

Java NIO 的非阻塞模式，当某个通道准备好数据，就会触发事件。Selector 轮询处理这些事件。使得没有数据可用时，就什么都不会获取，也就不会阻塞导致其他请求无法处理。所以直至数据变得可读/可写之前，Selector 可以继续处理其他事件。

#### 通道（Channel）

通道 Channel 是对原 I/O 包中的流的模拟，可以通过它读取和写入数据。

通道与流的不同之处在于，流只能在一个方向上移动，(一个流必须是 InputStream 或者 OutputStream 的子类)，而通道是双向的，可以用于读、写或者同时用于读写。

通道包括以下类型：

- FileChannel：从文件中读写数据；
- DatagramChannel：通过 UDP 读写网络中数据；
- SocketChannel：通过 TCP 读写网络中数据；
- ServerSocketChannel：可以监听新进来的 TCP 连接，对每一个新进来的连接都会创建一个 SocketChannel。

![img](http://static.oschina.net/uploads/img/201510/23094528_OF9c.jpg)

![NIO2.png](https://github.com/Cavielee/note/blob/master/pics/NIO2.png?raw=true)

#### 选择器（Selector）

- Selector 可以看做是一个单线程，通道会注册到 Selector 中并设置该通道监听的事件（可连接、可读、可写等），通道注册后会返回一个 SelectionKey（标识该通道）。
- Selector 轮询所有的通道，调用 `selector.select()` （会阻塞，直到有通道监听的事件发生）。
- 调用 `selector.selectedKeys()` 获得发生事件的 SelectionKey，并分发给处理线程处理（可以使用线程池实现处理线程，否则会阻塞）。这种选择机制，使得一个单独的线程很容易的管理多个通道，并且不会限制一定数量的客户端接入。（多路复用）。
- 处理线程只需要根据 SelectionKey 监听的事件类型进行相应的处理。

#### 缓冲区（Buffer）

缓冲区实际上是一个容器对象，更直接的说，其实就是一个数组，在 NIO 库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的；在写入数据时，它也是写入到缓冲区中的；任何时候访问 NIO 中的数据，都是将它放到缓冲区中。而在面向流 I/O 系统中，所有数据都是直接写入或者直接将数据读到 Stream 对象中。

##### 缓冲区状态变量

| 字段     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| Capacity | 容量，即可以容纳的最大数据量；在缓冲区创建时被设定并且不能改变 |
| Limit    | 表示缓冲区的当前终点，不能对缓冲区超过极限的位置进行读写操作。且极限是可以修改的 |
| Position | 位置，下一个要被读或写的元素的索引，每次读写缓冲区数据时都会改变改值，为下次读写作准备 |
| Mark     | 标记，调用mark()来设置mark=position，再调用reset()可以让position恢复到标记的位置 |



##### 实例化方法

通过 ByteBuffer.allocate() 来实例化 ByteBuffer。

| 方法                                        | 描述                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| allocate(int capacity)                      | 从堆空间中分配一个容量大小为capacity的byte数组作为缓冲区的byte数据存储器 |
| allocateDirect(int capacity)                | 是不使用JVM堆栈而是通过操作系统来创建内存块用作缓冲区，它与当前操作系统能够更好的耦合，因此能进一步提高I/O操作速度。但是分配直接缓冲区的系统开销很大，因此只有在缓冲区较大并长期存在，或者需要经常重用时，才使用这种缓冲区 |
| wrap(byte[] array)                          | 这个缓冲区的数据会存放在byte数组中，bytes数组或buff缓冲区任何一方中数据的改动都会影响另一方。其实ByteBuffer底层本来就有一个bytes数组负责来保存buffer缓冲区中的数据，通过allocate方法系统会帮你构造一个byte数组 |
| wrap(byte[] array,  int offset, int length) | 在上一个方法的基础上可以指定偏移量和长度，这个offset也就是包装后byteBuffer的position，而length呢就是limit-position的大小，从而我们可以得到limit的位置为length+position(offset) |



##### 常用方法

| 方法                                    | 描述                                                         |
| --------------------------------------- | ------------------------------------------------------------ |
| mark()                                  | 标记 mark=posistion                                          |
| reset()                                 | 把position设置成mark的值，相当于之前做过一个标记，现在要退回到之前标记的地方 |
| clear()                                 | position = 0;limit = capacity;mark = -1;  有点初始化的味道，但是并不影响底层byte数组的内容 |
| flip()                                  | limit = position;position = 0;mark = -1;  翻转，也就是让flip之后的position到limit这块区域变成之前的0到position这块，翻转就是将一个处于存数据状态的缓冲区变为一个处于准备取数据的状态 |
| rewind()                                | 把position设为0，mark设为-1，不改变limit的值                 |
| remaining()                             | return limit - position;返回limit和position之间相对位置差    |
| hasRemaining()                          | return position < limit返回是否还有未读内容                  |
| compact()                               | 把从position到limit中的内容移到0到limit-position的区域内，position和limit的取值也分别变成limit-position、capacity。如果先将positon设置到limit，再compact，那么相当于clear() |
| get()                                   | 相对读，从position位置读取一个byte，并将position+1，为下次读写作准备 |
| get(int index)                          | 绝对读，读取byteBuffer底层的bytes中下标为index的byte，不改变position |
| get(byte[] dst, int offset, int length) | 从position位置开始相对读，读length个byte，并写入dst下标从offset到offset+length的区域 |
| put(byte b)                             | 相对写，向position的位置写入一个byte，并将postion+1，为下次读写作准备 |
| put(int index, byte b)                  | 绝对写，向byteBuffer底层的bytes中下标为index的位置插入byte b，不改变position |
| put(ByteBuffer src)                     | 用相对写，把src中可读的部分（也就是position到limit）写入此byteBuffer |
| put(byte[] src, int offset, int length) | 从src数组中的offset到offset+length区域读取数据并使用相对写写入此byteBuffer |

状态变量的改变过程举例：

① 新建一个大小为 8 个字节的缓冲区，此时 position 为 0，而 limit = capacity = 8。capacity 变量不会改变，下面的讨论会忽略它。

![img](https://github.com/Cavielee/Interview-Notebook/raw/master/pics/1bea398f-17a7-4f67-a90b-9e2d243eaa9a.png)

② 从输入通道中读取 3 个字节数据写入缓冲区中，此时 position 移动设为 3，limit 保持不变。

![img](https://github.com/Cavielee/Interview-Notebook/raw/master/pics/4628274c-25b6-4053-97cf-d1239b44c43d.png)

③ 以下图例为已经从输入通道读取了 5 个字节数据写入缓冲区中。在将缓冲区的数据写到输出通道之前，需要先调用 flip() 方法，这个方法将 limit 设置为当前 position，并将 position 设置为 0。

![img](https://github.com/Cavielee/Interview-Notebook/raw/master/pics/4628274c-25b6-4053-97cf-d1239b44c43d.png)

④ 从缓冲区中取 4 个字节到输出缓冲中，此时 position 设为 4。

![img](https://github.com/Cavielee/Interview-Notebook/raw/master/pics/952e06bd-5a65-4cab-82e4-dd1536462f38.png)

⑤ 最后需要调用 clear() 方法来清空缓冲区，此时 position 和 limit 都被设置为最初位置。

![img](https://github.com/Cavielee/Interview-Notebook/raw/master/pics/67bf5487-c45d-49b6-b9c0-a058d8c68902.png)



#### 例子（获取时间服务）

##### Client

1. 开启 Selector
2. 建立 SocketChannel
3. 把 SocketChannel 设置为非阻塞
4. 把该 SocketChannel 注册到 Selector，设置为关注可读事件。
5. 当服务端发送信息事，就会注册一个事件（可读）
6. 轮询 selector.select() ，检查是否有通道准备好（是否有事件），有则交给 process() 处理（处理过程是阻塞的，可以通过线程池改善）。

```java
public class NIOTimeClient {

	public static void main(String[] args) {
		int port = 8080;

		TimeClient timeClient = new TimeClient(port);

		new Thread(timeClient).start();
	}

}

public class TimeClient implements Runnable {
	private SocketChannel socketChannel = null;
	private Selector selector = null;
	private boolean stop;

	public TimeClient(int port) {
		try {
			selector = Selector.open();
			socketChannel = socketChannel.open();
			socketChannel.configureBlocking(false);
			doConnect(port);
		} catch (IOException e) {
			e.printStackTrace();
			System.exit(1);
		}

	}

	private void doConnect(int port) throws IOException {
		if (socketChannel.connect(new InetSocketAddress("127.0.0.1", port))) {
			socketChannel.register(selector, SelectionKey.OP_READ);
			doWrite(socketChannel);
		} else {
			// 三次握手还没建立成功，继续监听连接事件
			socketChannel.register(selector, SelectionKey.OP_CONNECT);
		}
	}

	private void doWrite(SocketChannel sc) throws IOException {
		byte[] bytes = "QUERY TIME ORDER".getBytes();
		ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
		writeBuffer.put(bytes);
		writeBuffer.flip();
		sc.write(writeBuffer);
		if (!writeBuffer.hasRemaining()) {
			System.out.println("Send order 2 server succeed");
		}
	}

	public void run() {
		while (!stop) {
			try {
				selector.select(1000);
				Set<SelectionKey> selectedKeys = selector.selectedKeys();
				Iterator<SelectionKey> it = selectedKeys.iterator();
				SelectionKey key = null;
				while (it.hasNext()) {
					key = it.next();
					it.remove();
					try {
						handle(key);
					} catch (Exception e) {
						if (key != null) {
							key.cancel();
							if (key.channel() != null) {
								key.channel().close();
							}
						}
					}
				}
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
		// 关闭资源
		if (selector != null) {
			try {
				selector.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}

	private void handle(SelectionKey key) throws IOException {
		if (key.isValid()) {
			SocketChannel sc = (SocketChannel) key.channel();
			if (key.isConnectable()) {
				if (sc.finishConnect()) {
					socketChannel.register(selector, SelectionKey.OP_READ);
					doWrite(sc);
				} else {
					System.exit(1);
				}
			}
			if (key.isReadable()) {
				ByteBuffer readBuffer = ByteBuffer.allocate(1024);
				int readBytes = sc.read(readBuffer);
				byte[] bytes = new byte[readBytes];
				if (readBytes > 0) {
					readBuffer.flip();
					readBuffer.get(bytes);
					String body = new String(bytes, "UTF-8");
					System.out.println("Now is :" + body);
					this.stop();
				} else if (readBytes < 0) {
					key.cancel();
					sc.close();
				}
			}
		}
	}

	private void stop() {
		this.stop = true;
	}
}
```

##### Server

1. 开启一条 ServerSocketChannel，绑定监听的端口以及设置为非阻塞模式
2. 开启 selector
3. 把 ServerSocketChannel 注册到selector上，并设置该 channel 关注接受客户端请求
4. selector 轮询查看是否有通道准备好（以事件形式判定）
5. 当服务端发送信息事，就会注册一个事件（可读）
6. 轮询 selector.select() ，检查是否有通道准备好（是否有事件），有则交给 process() 处理（处理过程是阻塞的，可以通过线程池改善）。

```java
public class NIOTimeServer {

	public static void main(String[] args) {
		int port = 8080;

		MultiplexerTimeServer timeServer = new MultiplexerTimeServer(port);
		new Thread(timeServer).start();
		System.out.println("Time server start");
	}
}

/**
 * 处理逻辑
 * 
 * @author created by Cavielee
 * @date 2018年12月25日 上午11:39:18
 */
public class MultiplexerTimeServer implements Runnable {
	private volatile boolean stop;
	private ServerSocketChannel serverChannel = null;
	private Selector selector = null;

	public MultiplexerTimeServer(int port) {
		try {
			selector = selector.open();
			serverChannel = ServerSocketChannel.open();
			serverChannel.configureBlocking(false);
			serverChannel.socket().bind(new InetSocketAddress(port));
			serverChannel.register(selector, SelectionKey.OP_ACCEPT);
		} catch (IOException e) {
			e.printStackTrace();
			System.exit(1);
		}
	}

	private void stop() {
		this.stop = true;
	}

	public void run() {
		while (!stop) {
			try {
				int num = selector.select(1000);
				Set<SelectionKey> selectedKeys = selector.selectedKeys();
				Iterator<SelectionKey> it = selectedKeys.iterator();
				SelectionKey key = null;
				while (it.hasNext()) {
					key = it.next();
					it.remove();
					try {
						handle(key);
					} catch (Exception e) {
						if (key != null) {
							key.cancel();
							if (key.channel() != null) {
								key.channel().close();
							}
						}
					}
				}
			} catch (Throwable t) {
				t.printStackTrace();
			}
		}
		// 关闭资源
		if (selector != null) {
			try {
				selector.close();
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}

	private void handle(SelectionKey key) throws IOException {
		if (key.isValid()) {
			// 处理接入请求
			if (key.isAcceptable()) {
				ServerSocketChannel serverChannel = (ServerSocketChannel) key.channel();
				SocketChannel sc = serverChannel.accept();
				sc.configureBlocking(false);
				sc.register(selector, SelectionKey.OP_READ);
			}

			// 处理读取请求
			if (key.isReadable()) {
				SocketChannel sc = (SocketChannel) key.channel();
				ByteBuffer readBuffer = ByteBuffer.allocate(1024);
				int readBytes = sc.read(readBuffer);
				if (readBytes > 0) {
					readBuffer.flip();
					byte[] bytes = new byte[readBytes];
					readBuffer.get(bytes);
					String body = new String(bytes, "UTF-8");
					System.out.println("The time server receive order : " + body);
					String currentTime = "QUERY TIME ORDER".equals(body)
							? new Date(System.currentTimeMillis()).toString()
							: "BAD ORDER";
					doWrite(sc, currentTime);
				} else if (readBytes < 0) {
					// 关闭资源
					key.cancel();
					sc.close();
				} else {

				}
			}
		}
	}

	private void doWrite(SocketChannel sc, String response) throws IOException {
		if (response != null && !response.trim().isEmpty()) {
			byte[] bytes = response.getBytes();
			ByteBuffer writeBuffer = ByteBuffer.allocate(1024);
			writeBuffer.put(bytes);
			writeBuffer.flip();
			sc.write(writeBuffer);
		}
	}
}

```

#### 例子（读取文件）

```java
public class NIOReadFile {
    // 为要读取的文件创建 FileInputStream，之后通过 FileInputStream 获取输入 FileChannel；
    FileInputStream fin = new FileInputStream("readandshow.txt");
    FileChannel fic = fin.getChannel();
    // 创建一个容量为 1024 的 Buffer；
    ByteBuffer buffer = ByteBuffer.allocate(1024);
    // 将数据从输入 FileChannel 写入到 Buffer 中，如果没有数据的话，read() 方法会返回 -1；
    int r = fcin.read(buffer);
    if (r == -1) {
         break;
    }
    // 为要写入的文件创建 FileOutputStream，之后通过 FileOutputStream 获取输出 FileChannel

    FileOutputStream fout = new FileOutputStream("writesomebytes.txt");
    FileChannel foc = fout.getChannel();
    // 调用 flip() 切换读写

    buffer.flip();
    // 把 Buffer 中的数据读取到输出 FileChannel 中

    foc.write(buffer);
    // 最后调用 clear() 重置缓冲区

    buffer.clear();
}
```



### 