## RPC（Remote Procedure Call）

RPC —— 远程服务调用。

用途：在分布式中，JVM_A 调用 JVM_B 的对象方法，使得调用该对象的方法如同在 JVM_A 本地一样。

原理：实际上底层是通过客户端和服务端之间的 Socket 通信，ServerSocket 通过监听端口，并根据客户端传来的参数进行相应的处理（调用对应的对象方法），并返回相应结果给客户端。

* 编写服务端服务，并将其通过某个服务机的端口暴露出去供客户端调用。
* 编写客户端程序，客户端通过指定服务所在的主机和端口号、将请求封装并序列化，最终通过网络协议发送到服务端。
* 服务端解析和反序列化请求，调用服务端上的服务，将结果序列化并返回给客户端。
* 客户端接收并反序列化服务端返回的结果，反馈给用户。

为了屏蔽底层实现：

客户端操作封装成 Stub 对象，Stub对象做的事情是建立到服务端Skeleton对象的Socket连接。将客户端的方法调用转换为字符串标识传递给Skeleton对象。并且同步阻塞等待服务端返回结果。

服务端操作封装成 Skeleton 对象，Skeleton对象做的事情是将服务实现传入构造参数，获取客户端通过socket传过来的方法调用字符串标识，将请求转发到具体的服务上面。获取结果之后返回给客户端。

二者实际都是动态代理对象。



### RMI 实现

1. 服务端创建 Registry 对象，并将服务对象注册到 Registry 上

```java
// 参数为 ServerSocket 的监听端口，接受客户端的请求
Registry registry=LocateRegistry.createRegistry(1099);

// 绑定服务URL及对应服务
IOperation iOperation=new OperationImpl();
Naming.rebind("rmi://127.0.0.1:1099/Operation",iOperation);
```

Registry 对象：用一个 Map 集合存储 Key:服务接口URL，Value:对应服务



2. 创建远程服务对象

```java
/**
 * 服务端接口必须实现java.rmi.Remote
 */
public interface IOperation extends Remote{

    /**
     * 远程接口上的方法必须抛出RemoteException，因为网络通信是不稳定的，不能吃掉异常
     * @param a
     * @param b
     * @return
     */
    int add(int a, int b) throws RemoteException;

}
```



3. 服务端实现远程服务对象的实现类

```java
public class OperationImpl extends UnicastRemoteObject implements IOperation{

    public OperationImpl() throws RemoteException {
        super();
    }

    @Override
    public int add(int a, int b) throws RemoteException{
        return a+b;
    }

}
```



4. 客户端调用

```java
public class Client {

    public static void main(String args[]) throws Exception{
        // 创建相应的Stub代理对象
        IOperation iOperation= (IOperation) Naming.lookup("rmi://127.0.0.1:1099/Operation");
        // 调用Stub对象方法，实际底层使用Socket去服务端的Skeleton调用相应的对象方法
        System.out.println(iOperation.add(1,1));
    }

}
```



# RPC

Http 是一种应用层协议

用于客户端和服务端的通信，客户端一般为浏览器。

Http 协议是一种请求响应的协议。

通信流程：

1. 建立连接
2. 客户端请求
3. 服务端响应

Http 1.0时，该协议是短连接。意味着客户端每次发起请求前都会先和服务端建立连接，然后发送请求，服务端处理完后会响应请求，收到响应后连接就会断开。因此缺点很明显：频繁创建、销毁连接。

Http 1.1支持一种伪长链接。为了避免短连接导致频繁创建、销毁连接。Http 1.1提供了一种心跳机制，客户端每隔一段时间会发送一次心跳包（特殊的请求），当客户端一段时间没有收到服务端的心跳响应，或者服务端没收到客户端的心跳请求，则会断开连接。

缺点：实际上不是一种真正意义上的长链接，例如 IM（通信）系统中，服务端不能主动推送数据给客户端，需要客户端不断轮询发起请求去获取数据，服务端收到请求才会响应。

Http 是无状态协议，意味着即使是来自于同一个客户端，但由于是不同的连接，无法进行识别。因此为了解决这个问题，通过服务端Session和客户端Cookie进行状态保留。

Http 协议由两个部分组成：协议头和协议体，因此像上述 Http 1.1实现IM系统时，服务端只关心请求体，客户端只关心响应体，但由于协议必须有协议头，因此导致协议头属于不必要的内容，额外增加网络传输负担。



WebSocket 是基于 Http 协议之上的协议，首先通过 Http 建立连接，然后升级为 WebSocket 协议。

WebSocket 是真正意义上的长链接，是一种双向通信的连接，客户端可以发起请求，服务端也可以主动推送。



RMI——远程方法调用，Java 提供的两个Java进程的方法调用。

客户端本地调用方法（stub），底层实际是通过socket方式将调用的方法信息序列化传输到服务端。

服务端收到skeleton收到信息后反序列化得到调用的方法，并反射调用，然后将结果序列化返回给客户端。





RPC——远程过程调用，实际和RMI相似，但RPC一般是跨语言。



WebService 也可以看做是一种 RPC 框架，但 RPC 一般是基于 Socket 数据传输，而 WebService 则是通过 http 数据传输，



内网服务与服务之间的调用一般推荐使用 RPC 框架。
