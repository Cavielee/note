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
2. 每次通信都需要打开和关闭 IO 流，降低了效率。
3. （伪异步 BIO）在 read() 方法中，会阻塞读取，直至读取完毕/异常。由于发送者发送缓慢、网络传输缓慢、读取者读取缓慢等原因，将会导致阻塞时间变长，因此会导致线程占用时间长，其他的接入请求不断地添加到消息队列中等待。**并且当消息队列填满时，会拒绝客户端接入请求，客户端会发生大量的连接超时。**
4. （伪异步 BIO）在 write() 方法中，会阻塞写入，直至所有要发送的字节写入完毕/异常。由于接收方处理缓慢，可能会导致 TCP 缓冲区填满，而发送方只能等待 TCP 缓冲区有空余位置才能继续写入。



#### 例子（聊天）

##### Client

缺点：

- Listener 会阻塞在 read() 方法，直到服务端发来信息。

```java
package cn.cavie.nio;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.Socket;
import java.net.UnknownHostException;
import java.nio.charset.Charset;
import java.util.Scanner;

public class BIOClient {
	private Socket socket = null;
	private InputStream is = null;
	private OutputStream os = null;

	private String nickName = "";
	private Charset charset = Charset.forName("UTF-8");
	private static String USER_EXIST = "系统提示：该昵称已存在，请换一个昵称";
	private static String USER_CONTENT_SPILT = "：";

	public BIOClient() throws UnknownHostException, IOException {
		socket = new Socket("localhost", 8080);
		is = socket.getInputStream();
		os = socket.getOutputStream();
	}

	public void session() {
		// 开一个线程读取服务端信息
		new Listener().start();
		// 开一个线程向服务端写信息
		new Writer().start();
	}

	private class Listener extends Thread {

		@Override
		public void run() {
			try {
				byte[] buffer = new byte[1024];
				while (!socket.isClosed()) {
					String message = "";
					// 阻塞等待，直到服务端有信息发送
					int len = is.read(buffer);
					// 服务信息
					message += new String(buffer, 0, len, charset);
					if (USER_EXIST.equals(message)) {
						nickName = "";
					}
					if (!"".equals(message)) {
						System.out.println(message);
					}
				}
			} catch (IOException e) {
			}
		}

	}

	private class Writer extends Thread {

		@Override
		public void run() {

			try {
				// 获取写流
				Scanner sc = new Scanner(System.in);
				// 写入信息
				while (sc.hasNextLine()) {
					String message = sc.nextLine();
					// 不允许发空信息
					if ("".equals(message)) {
						continue;
					}
					if ("".equals(nickName)) {
						nickName = message;
						message = nickName;
					} else if (("exit").equals(message)) { // 如果输入exit则退出
						System.out.println("退出聊天");
						os.write(message.getBytes());
						is.close();
						os.close();
						socket.close();
						break;
					} else {
						message = nickName + USER_CONTENT_SPILT + message;
					}
					// 写信息到服务器
					os.write(message.getBytes());
				}
			} catch (IOException e) {
				e.printStackTrace();
			}
		}

	}

	public static void main(String[] args) throws UnknownHostException, IOException {
		try {
			new BIOClient().session();
		} catch (UnknownHostException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
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
package cn.cavie.nio;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.ServerSocket;
import java.net.Socket;
import java.nio.charset.Charset;
import java.util.HashMap;
import java.util.HashSet;
import java.util.Map;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BIOServer {
	private ServerSocket ss = null;
	private HashSet<String> user = new HashSet<>();
	private HashMap<Socket, String> clients = new HashMap<>();
	private ExecutorService es = Executors.newFixedThreadPool(3);

	private Charset charset = Charset.forName("UTF-8");
	private static String USER_EXIST = "系统提示：该昵称已存在，请换一个昵称";

	public BIOServer(int port) throws IOException {
		ss = new ServerSocket(port);
		// 轮询监听
		while (true) {
			// 阻塞等待，直到有客户端连接
			Socket socket = ss.accept();
			clients.put(socket, "");
			// 如果连接池用完，则阻塞等待，直到有空闲的工作线程才能执行
			es.submit(new Process(socket));
		}
	}

	// 监听客户端发来的信息
	private class Process implements Runnable {
		Socket socket = null;

		public Process(Socket socket) {
			this.socket = socket;
		}

		@Override
		public void run() {
			try {
				InputStream is = socket.getInputStream();
				OutputStream os = socket.getOutputStream();
				byte[] buffer = new byte[1024];
				os.write("请输入昵称".getBytes());
				// 读取信息
				while (true) {
					String message = "";
					// 阻塞等待，直到客户端有信息发送
					int len = is.read(buffer);
					// 读取客户端信息
					message += new String(buffer, 0, len, charset);
					System.out.println(message);
					// 设置昵称
					if (("").equals(clients.get(socket))) {
						// 判断昵称是否存在
						if (user.contains(message)) {
							os.write(USER_EXIST.getBytes());
						} else { // 添加user
							user.add(message);
							clients.put(socket, message);
							System.out.println(clients.size());
							os.write("开始聊天：".getBytes());
							// 广播
							message = "欢迎" + message + "加入群聊！";
							bordercast(socket, message);
						}
					} else if (("exit").equals(message)) {
						try {
							// 广播给其他用户
							message = clients.get(socket) + "已退出群聊";
							bordercast(socket, message);
						} catch (IOException ee) {
							ee.printStackTrace();
						} finally {
							clients.remove(socket);
						}
					} else {
						// 广播给其他用户
						bordercast(socket, message);
					}
				}

			} catch (IOException e) {
				try {
					// 广播给其他用户
					String message = clients.get(socket) + "已退出群聊";
					bordercast(socket, message);
				} catch (IOException ee) {
					ee.printStackTrace();
				} finally {
					clients.remove(socket);
				}
			}
		}

	}

	public void bordercast(Socket socket, String message) throws IOException {
		for (Map.Entry<Socket, String> client : clients.entrySet()) { // 广播出自己以外
			if (!clients.get(socket).equals(client.getValue())) {
				Socket other = client.getKey();
				OutputStream os = other.getOutputStream();
				os.write(message.getBytes());
			}
		}
	}

	public static void main(String[] args) {
		try {
			new BIOServer(8080);
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
}

```



### NIO

同步非阻塞式IO，采用了事件驱动的思想来实现了一个多路复用器（Selector）。 

#### 优点

- 创建通道 Channel，通过通道（双向的）可以进行读写，读写完后通道归还，不会销毁（一个 SocketChannel 代表客户端和服务端的交互通道，当交互结束时，通道就会销毁）。
- 多路复用器 Selector （实际类似于一个线程）， 不断的轮询各个通道，看是否有通道准备好（可连接、可读、可写等事件），提高了响应，并把请求任务分发给处理线程。不会出现 BIO 那样，如果同一时间客户端请求过多要等待服务端处理完才能接受新的请求。NIO 统一的把所有连接请求作为事件注册到 Selector 上，使得请求不会阻塞在外面，接受所有的接入请求，并作为事件有序（会为每个事件派发一个 SelectionKey）的处理。（同步非阻塞）
- 客户端可以不停的往缓冲区写数据，不需要等待服务端处理完成后才能继续写，服务端只要检查到缓冲区数据准备好了，就会安排处理数据（触发一个可读/可写的事件到 Selector 上，由 Selector 轮询进行处理），不需要阻塞等待数据写完。因此可以在空闲的时候处理其他请求。（非阻塞）

#### 阻塞与非阻塞 IO

Java IO 的各种流是阻塞的。这意味着，当一个线程调用 read() 或 write() 时，该线程被阻塞，直到有一些数据被读取，或数据完全写入。该线程在此期间不能再干任何事情了。

Java NIO 的非阻塞模式，当某个通道准备好数据，就会触发事件。Selector 轮询处理这些事件。使得没有数据可用时，就什么都不会获取，也就不会阻塞导致其他请求无法处理。所以直至数据变得可读/可写之前，Selector 可以继续处理其他事件。

#### 通道

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
- 调用 `selector.selectedKeys()` 获得发生事件的 SelectionKey，并分发给处理线程处理。这种选择机制，使得一个单独的线程很容易的管理多个通道。（多路复用）。
- 处理线程只需要根据 SelectionKey 监听的事件类型进行相应的处理。

#### 缓冲区（Buffer）

缓冲区实际上是一个容器对象，更直接的说，其实就是一个数组，在 NIO 库中，所有数据都是用缓冲区处理的。在读取数据时，它是直接读到缓冲区中的；在写入数据时，它也是写入到缓冲区中的；任何时候访问 NIO 中的数据，都是将它放到缓冲区中。而在面向流 I/O 系统中，所有数据都是直接写入或者直接将数据读到 Stream 对象中。

##### 缓冲区状态变量

- capacity：最大容量；
- position：当前已经读写的字节数；
- limit：还可以读写的字节数。



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

#### 例子（聊天）

##### Client

1. 开启 Selector
2. 建立 SocketChannel
3. 把 SocketChannel 设置为非阻塞
4. 把该 SocketChannel 注册到 Selector，设置为关注可读事件。
5. 当服务端发送信息事，就会注册一个事件（可读）
6. 轮询 selector.select() ，检查是否有通道准备好（是否有事件），有则交给 process() 处理（处理过程是阻塞的，可以通过线程池改善）。

```java
package cn.cavie.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.UnknownHostException;
import java.nio.ByteBuffer;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.SocketChannel;
import java.nio.charset.Charset;
import java.util.Iterator;
import java.util.Scanner;
import java.util.Set;

public class NIOClient {
	private InetSocketAddress serverAddress = new InetSocketAddress("localhost", 8080);
	private Selector selector = null;
	private SocketChannel client = null;

	private String nickName = "";
	private Charset charset = Charset.forName("UTF-8");
	private static String USER_EXIST = "系统提示：该昵称已存在，请换一个昵称";
	private static String USER_CONTENT_SPILT = ":";
	private boolean isExit = false;

	public NIOClient() throws UnknownHostException, IOException {
		// 打开Selector
		selector = Selector.open();
		// 连接服务端
		client = SocketChannel.open(serverAddress);
		// 设置为非阻塞
		client.configureBlocking(false);
		// 把该通道注册到 Selector 并监听可读事件
		client.register(selector, SelectionKey.OP_READ);
	}

	public void session() {
		// 开一个线程读取服务端信息
		new Listener().start();
		// 开一个线程向服务端写信息
		new Writer().start();
	}
	
	public void exit() throws IOException {
		client.close();
		selector.close();
	}

	private class Listener extends Thread {

		@Override
		public void run() {
			try {
				while (!isExit) {
                    // 检查是否有事件（当有事件时，证明有通道已经准备好）
					int readyChannel = selector.select();
					
					// 获得触发事件的通道，返回这些通道的标识 SelectionKey
					Set<SelectionKey> keys = selector.selectedKeys();
					Iterator<SelectionKey> iterator = keys.iterator();
                    // 处理事件
					while (iterator.hasNext()) {
						SelectionKey key = iterator.next();
                        // 已处理要删除事件
						iterator.remove();
						process(key);
					}

				}
			} catch (IOException e) {
			}
		}

		public void process(SelectionKey key) throws IOException {
			// 如果是客户端可读事件
			if (key.isReadable()) {
				// 返回该SelectionKey 对应的 channel，其中有数据要读
				SocketChannel client = (SocketChannel) key.channel();
				ByteBuffer buffer = ByteBuffer.allocate(1024);
				StringBuilder content = new StringBuilder();
				while (client.read(buffer) > 0) {
					buffer.flip();
					content.append(charset.decode(buffer));
				}
				String message = content.toString();
				if (USER_EXIST.equals(message)) {
					nickName = "";
				}
				System.out.println(message);
				// 将此对应的 channel 设置为准备下一次接收数据
				key.interestOps(SelectionKey.OP_READ);
			}
			// 如果是服务端可写事件
			else if (key.isWritable()) {

			}
		}

	}

	private class Writer extends Thread {

		@Override
		public void run() {

			try {
				// 获取写流
				Scanner sc = new Scanner(System.in);
				// 写入信息
				while (!isExit && sc.hasNextLine()) {
					String message = sc.nextLine();
					// 不允许发空信息
					if ("".equals(message)) {
						continue;
					}
					if ("".equals(nickName)) {
						nickName = message;
						message = nickName;
					} else if (("exit").equals(message)) { // 如果输入exit则退出
						System.out.println("退出聊天");
						message = nickName + USER_CONTENT_SPILT + message;
						isExit = true;
						// 写信息到服务器
						client.write(charset.encode(message));
						exit();
						
					} else {
						message = nickName + USER_CONTENT_SPILT + message;
						// 写信息到服务器
						client.write(charset.encode(message));
					}
				}
			} catch (IOException e) {
				e.printStackTrace();
			}
		}

	}

	public static void main(String[] args) throws UnknownHostException, IOException {
		try {
			new NIOClient().session();
		} catch (UnknownHostException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}
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
package cn.cavie.nio;

import java.io.IOException;
import java.net.InetSocketAddress;
import java.net.ServerSocket;
import java.nio.ByteBuffer;
import java.nio.channels.Channel;
import java.nio.channels.SelectionKey;
import java.nio.channels.Selector;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.nio.charset.Charset;
import java.util.HashSet;
import java.util.Iterator;
import java.util.Set;

public class NIOServer {
	ServerSocket ss = null;
	HashSet<String> users = new HashSet<>();
	Selector selector = null;
	ServerSocketChannel server;
	private Charset charset = Charset.forName("UTF-8");
	private static String USER_EXIST = "系统提示：该昵称已存在，请换一个昵称";
	private static String USER_CONTENT_SPILT = ":";

	public NIOServer(int port) throws IOException {
		// 打开通道
		server = ServerSocketChannel.open();
		server.bind(new InetSocketAddress(port));
        // 设置为非阻塞
		server.configureBlocking(false);

		// 等于服务叫号大厅
		selector = Selector.open();

		// 把该通道注册到 Selector 并监听连接建立事件
		server.register(selector, SelectionKey.OP_ACCEPT);

		System.out.println("服务端已启动，端口为：" + port);
	}

	public void listener() throws IOException {
		// 轮询，非阻塞
		while (true) {
			// 检查是否有事件（当有事件时，证明有通道已经准备好）
			int wait = selector.select();

			// 获得触发事件的通道，返回这些通道的标识 SelectionKey
			Set<SelectionKey> keys = selector.selectedKeys();
			Iterator<SelectionKey> iterator = keys.iterator();
            // 处理事件
			while (iterator.hasNext()) {
				SelectionKey key = iterator.next();
                // 已处理要删除事件
				iterator.remove();
				process(key);
			}

		}
	}

	public void process(SelectionKey key) throws IOException {
		// 如果是连接建立事件(客户端已经准备好，并可以实现交互)
		if (key.isAcceptable()) {
			ServerSocketChannel server = (ServerSocketChannel) key.channel();
            // 为每一个客户端连接建立，创建一条通道SocketChannel
			SocketChannel client = server.accept();
			// 非阻塞模式
			client.configureBlocking(false);
			// 注册该通道到 Selector，并设置该通道监听可读事件
			client.register(selector, SelectionKey.OP_READ);

			// 将此对应的 channel 设置为准备接受其他客户请求
			// key.interestOps(SelectionKey.OP_ACCEPT);

			client.write(charset.encode("请输入昵称"));
		}
		// 如果是客户端可读事件
		else if (key.isReadable()) {
			// 返回该SelectionKey 对应的 channel，其中有数据要读
			SocketChannel client = (SocketChannel) key.channel();
			ByteBuffer buffer = ByteBuffer.allocate(1024);
			StringBuilder content = new StringBuilder();
			try {
				while (client.read(buffer) > 0) {
					buffer.flip();
					content.append(charset.decode(buffer));
				}
				// 将此对应的 channel 设置为准备下一次接收数据
				// key.interestOps(SelectionKey.OP_READ);
			} catch (IOException e) {
				clientExit(key);
			}
			// 处理信息
			if (content.length() > 0) {
				String[] arrayContent = content.toString().split(USER_CONTENT_SPILT);
				// 注册用户
				if (arrayContent != null && arrayContent.length == 1) {
					String nickName = arrayContent[0];
					if (users.contains(nickName)) {
						client.write(charset.encode(USER_EXIST));
					} else {
						users.add(nickName);
						// 广播
						String message = "欢迎" + nickName + "加入群聊！当前在线人数" + onlineCount() + "人";
						bordercast(client, message);
					}
				}
				// 注册完了，发送消息
				else if (arrayContent != null && arrayContent.length > 1) {
					String nickName = arrayContent[0];
					String message = "";
					// 用户退出
					if ("exit".equals(arrayContent[1])) {
						message += nickName + "已退出群聊。";
						users.remove(nickName);
					} else {
						message += nickName + USER_CONTENT_SPILT + arrayContent[1];
					}
					bordercast(client, message);
				}
			}

		}
		// 如果是服务端可写事件
		else if (key.isWritable()) {

		}
	}

	// 用户退出
	public void clientExit(SelectionKey key) throws IOException {
		key.cancel();
		if (key.channel() != null) {
			key.channel().close();
		}
	}

	// 统计当前在线用户
	public int onlineCount() {
		int count = 0;
		for (SelectionKey key : selector.keys()) {
			Channel target = key.channel();

			if (target instanceof SocketChannel) {
				count++;
			}
		}
		return count;
	}

    // 广播
	public void bordercast(SocketChannel client, String message) throws IOException {
		for (SelectionKey key : selector.keys()) {
			Channel target = key.channel();

			if (!(target instanceof ServerSocketChannel) && target != client) {
				((SocketChannel) target).write(charset.encode(message));
			}
		}
	}

	public static void main(String[] args) {
		try {
			new NIOServer(8080).listener();
		} catch (IOException e) {
			e.printStackTrace();
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