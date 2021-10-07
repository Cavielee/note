# RSocket源码分析



## 启动服务端

**自动装配：**

```java
public class RSocketServerAutoConfiguration {
	@ConditionalOnProperty(prefix = "spring.rsocket.server", name = "port")
	static class EmbeddedServerAutoConfiguration {
        @Bean
		@ConditionalOnMissingBean
		RSocketServerBootstrap rSocketServerBootstrap(RSocketServerFactory rSocketServerFactory,
				RSocketMessageHandler rSocketMessageHandler) {
			return new RSocketServerBootstrap(rSocketServerFactory, rSocketMessageHandler.responder());
		}
	}
}
```

**RSocketServerFactory：**

1. 管理RSocket服务的配置，比如端口。
2. 创建RSocket服务，需要参数SocketAcceptor。

**SocketAcceptor：**

1. 接收创建连接的消息（payload），返回处理远程请求的响应者（responder）。

**RSocketMessageHandler：**

1. 管理消息编解码RSocketStrategies。
2. 管理消息处理器（AbstractMethodMessageHandler.handlerMethods）。

**RSocketServerBootstrap:**

1. 管理RSocket服务启动和关闭。



## 连接服务端

**示例代码：**

```java
public RSocketRequester connect(RSocketRequester.Builder rsocketRequesterBuilder, RSocketStrategies strategies) {
    String client = UUID.randomUUID().toString();
    log.info("Connecting using client ID: {}", client);

    SocketAcceptor acceptor = RSocketMessageHandler.responder(strategies, new ClientHandler());

    RSocketRequester rsocketRequester = rsocketRequesterBuilder.setupRoute("shell-client")
            .setupData(client)
            .rsocketStrategies(strategies)
            .rsocketConnector(connector -> connector.acceptor(acceptor).resume(new Resume()))
            .connectTcp("localhost", 7000)
            .block();

    rsocketRequester.rsocket()
            .onClose()
            .doOnError(er -> log.error("connection closed.", er))
            .doFinally(c -> log.info("connection disconnected"))
            .subscribe();
    
    return rsocketRequester;
}
```

**RSocketMessageHandler.responder：**

1. RSocketMessageHandler的工厂方法，提供消息编解码器、消息处理器，构造SocketAcceptor。
2. SocketAcceptor需要实现accept接口，返回responder，responder通过RSocketMessageHandler.createResponder构建。

**RSocketMessageHandler.createResponder：**

1. 创建MessagingRSocket，MessagingRSocket.handle()调用消息处理器处理消息。

**rsocketRequesterBuilder.setupRoute().setupData()：**

1. 创建连接时附加的信息

**connector.resume(new Resume())：**

1. 连接断开后，自动恢复

**RSocketRequester：**

1. 负责发送消息

   

## 发送消息

**示例代码：**

```java
private RSocketRequester rsocketRequester;

public void requestResponse() throws InterruptedException {
	this.rsocketRequester
        .route("request-response")
        .data(new Message())
        .retrieveMono(Message.class)
        .subscribe(m -> {
            log.info("\nResponse was: {}", m);
        });
}
```

1. 四种交互模式：Request-Response、Request-Stream、Channel、Fire-and-Forget。示例中演示的是Request-Response。
2. RSocketRequester的实现类为DefaultRSocketRequester，通过持有的rsocketClient:RSocketClientAdapter来发送消息。



**RSocketClientAdapter：**

```java
class RSocketClientAdapter implements RSocketClient {
    private final RSocket rsocket;
    
    public Mono<Payload> requestResponse(Mono<Payload> payloadMono) {
        return payloadMono.flatMap(rsocket::requestResponse);
    }
}
```

1. rsocket:RSocket的实现类为RSocketRequester。



**RSocketRequester：**

```java
class RSocketRequester implements RSocket {
    public Mono<Payload> requestResponse(Payload payload) {
    	return new RequestResponseRequesterMono(payload, this);
    }
}
```



**RequestResponseRequesterMono：**

```java
final class RequestResponseRequesterMono extends Mono<Payload> implements Subscription {
    
    public final void request(long n) {
    	sendFirstPayload(this.payload, n);
    }
    
    void sendFirstPayload(Payload payload, long initialRequestN) {
    	int streamId = = sm.addAndGetNextStreamId(this);
        sendReleasingPayload(streamId, FrameType.REQUEST_RESPONSE, this.mtu, payload, connection, allocator, true);
    }
    
    static void sendReleasingPayload(
      int streamId,
      FrameType frameType,
      int mtu,
      Payload payload,
      DuplexConnection connection,
      ByteBufAllocator allocator,
      boolean requester) {
    	boolean fragmentable = isFragmentable(mtu, data, metadata, false);
        if (fragmentable) {
        	ByteBuf first = FragmentationUtils.encodeFirstFragment(
                allocator, mtu, frameType, streamId, hasMetadata, slicedMetadata, slicedData);
            connection.sendFrame(streamId, first);
            while (slicedData.isReadable() || slicedMetadata.isReadable()) {
                final ByteBuf following =
                      FragmentationUtils.encodeFollowsFragment(
                          allocator, mtu, streamId, complete, slicedMetadata, slicedData);
                connection.sendFrame(streamId, following);
          	}
        } else {
            ...
        }
    }
}
```

1. 数据包太大的时候，会分成多个Frame发送。



## 接收消息

**处理消息的两种形式：**

1. 请求-响应式，发送请求后在Subscriber中处理，比如发送消息中给出的示例。
2. 通过@MessageMapping或@ConnectMapping注册处理函数。

所以，我们分析一下接收消息到处理消息的流程。



**DuplexConnection：**

1. RSocketRequester和RSocketResponder构造时需要提供参数DuplexConnection。
2. tcp连接的实现为TcpDuplexConnection，如果开启自动重连，实现为ResumableDuplexConnection，持有activeConnection:TcpDuplexConnection。



**ResumableDuplexConnection：**

```java
public class ResumableDuplexConnection extends Flux<ByteBuf> implements DuplexConnection, Subscription {
    void initConnection(DuplexConnection nextConnection) {
        final FrameReceivingSubscriber frameReceivingSubscriber =
            new FrameReceivingSubscriber(tag, resumableFramesStore, receiveSubscriber);
        nextConnection.receive().subscribe(frameReceivingSubscriber);
    }
}
```



**FrameReceivingSubscriber：**

```java
class FrameReceivingSubscriber implements CoreSubscriber<ByteBuf> {

  final CoreSubscriber<? super ByteBuf> actual;

  private FrameReceivingSubscriber(
      String tag, ResumableFramesStore store, CoreSubscriber<? super ByteBuf> actual) {
    this.tag = tag;
    this.resumableFramesStore = store;
    this.actual = actual;
  }

  @Override
  public void onNext(ByteBuf frame) {
    actual.onNext(frame);
  }
}
```

1. FrameReceivingSubscriber包装了CoreSubscriber actual，actual的类型为ClientServerInputMultiplexer。



**ClientServerInputMultiplexer：**

```java
class ClientServerInputMultiplexer implements CoreSubscriber<ByteBuf> {
    private final InternalDuplexConnection serverReceiver;
    private final InternalDuplexConnection clientReceiver;
    private final DuplexConnection serverConnection;
    private final DuplexConnection clientConnection;
    private final DuplexConnection source;
    private final boolean isClient;
    
    public void onNext(ByteBuf frame) {
        int streamId = FrameHeaderCodec.streamId(frame);
        final Type type;
        if (streamId == 0) {
            switch (FrameHeaderCodec.frameType(frame)) {
                case LEASE:
                case KEEPALIVE:
                case ERROR:
                    type = isClient ? Type.CLIENT : Type.SERVER;
                    break;
                default:
                    type = isClient ? Type.SERVER : Type.CLIENT;
            }
        } else if ((streamId & 0b1) == 0) {
            type = Type.SERVER;
        } else {
            type = Type.CLIENT;
        }

        switch (type) {
            case CLIENT:
                clientReceiver.onNext(frame);
                break;
            case SERVER:
                serverReceiver.onNext(frame);
                break;
        }
	}
}
```

1. clientReceiver和serverReceiver应该对应了开始提到了处理消息的两种形式。



**回到RSocketRequester的构造逻辑：**

```java
class RSocketRequester extends RequesterResponderSupport implements RSocket {
    RSocketRequester(DuplexConnection connection, ...) {
		connection.receive().subscribe(this::handleIncomingFrames, e -> {});
    }
    
    private void handleIncomingFrames(ByteBuf frame) {
        int streamId = FrameHeaderCodec.streamId(frame);
        FrameType type = FrameHeaderCodec.frameType(frame);
        if (streamId == 0) {
        	handleStreamZero(type, frame);
        } else {
        	handleFrame(streamId, type, frame);
        }
	}
	
	private void handleFrame(int streamId, FrameType type, ByteBuf frame) {
        FrameHandler receiver = this.get(streamId);

        switch (type) {
            case NEXT_COMPLETE:
            	receiver.handleNext(frame, false, true);
            	break;
            case NEXT:
            	boolean hasFollows = FrameHeaderCodec.hasFollows(frame);
            	receiver.handleNext(frame, hasFollows, false);
            	break;
            ...
        }
    }
}

class FireAndForgetResponderSubscriber implements CoreSubscriber<Void>, ResponderFrameHandler {
 @Override
  public void handleNext(ByteBuf followingFrame, boolean hasFollows, boolean isLastPayload) {
    final CompositeByteBuf frames = this.frames;

    ReassemblyUtils.addFollowingFrame(
          frames, followingFrame, hasFollows, this.maxInboundPayloadSize);

    if (!hasFollows) {
      this.requesterResponderSupport.remove(this.streamId, this);
      this.frames = null;

      Payload payload = this.payloadDecoder.apply(frames);
      frames.release();

      Mono<Void> source = this.handler.fireAndForget(payload);
      source.subscribe(this);
    }
  }
}
```

1. FrameHandler实现包括FireAndForgetResponderSubscriber、RequestResponseResponderSubscriber等等，可以看出和交互方式有对应关系。
2. NEXT和NEXT_COMPLETE表明一个消息会拆成多个Frame，在接收端完成组装。



## 管理执行线程

在某些业务场景，希望为消息绑定处理线程。比如，根据玩家id分配线程，保证单个玩家的消息顺序处理，多个玩家的消息并行处理。为此，需要在消息中传递玩家标识，用于确定线程，并为处理逻辑绑定线程。



### 在消息中传递玩家标识

一个玩家创建一个客户端，服务端接收请求时需要知道消息是谁传过来的。这个标识可以在消息头中传递，逻辑处理函数通过方法参数获取这个标识。一种方法是通过@Header指定参数获取哪一个header，参考：[rsocket-messagemapping](https://docs.spring.io/spring-framework/docs/5.3.5/reference/html/web-reactive.html#rsocket-annot-messagemapping)，另一种方式是定义HandlerMethodArgumentResolver，这样省去在每个处理函数上加@Header注解的麻烦。





### 为消息处理器指定执行线程



### 为请求响应指定执行线程

