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
