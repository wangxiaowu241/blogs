## 介绍

netty是一款基于NIO的高性能网络框架，它对JDK中的NIO做了封装和优化，提供了更好的性能的同时，降低了使用的难度。netty支持NIO中的select、poll、epoll（仅Linux）等。关于这三者及BIO、NIO、AIO的介绍请看https://segmentfault.com/a/1190000003063859

同时，netty还支持HTTP、HTTPS、websocket协议，并提供了开箱即用的一些基类。 

## 官网服务端最佳实践

```java
public class DiscardServer {
    
    private int port;
    
    public DiscardServer(int port) {
        this.port = port;
    }
    
    public void run() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(); // (1)
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap(); // (2)
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class) // (3)
             .childHandler(new ChannelInitializer<SocketChannel>() { // (4)
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ch.pipeline().addLast(new DiscardServerHandler());
                 }
             })
             .option(ChannelOption.SO_BACKLOG, 128)          // (5)
             .childOption(ChannelOption.SO_KEEPALIVE, true); // (6)
    
            // Bind and start to accept incoming connections.
            ChannelFuture f = b.bind(port).sync(); // (7)
    
            // Wait until the server socket is closed.
            // In this example, this does not happen, but you can do that to gracefully
            // shut down your server.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
            bossGroup.shutdownGracefully();
        }
    }
    
    public static void main(String[] args) throws Exception {
        int port;
        if (args.length > 0) {
            port = Integer.parseInt(args[0]);
        } else {
            port = 8080;
        }
        new DiscardServer(port).run();
    }
}
```

1. NioEventLoopGroup是一个多线程事件驱动IO操作类。netty对不同的传输提供了多样的EventLoopGroup实现，我们这里是将netty作为服务端使用，因此这里有两个NioEventLoopGroup。其中一个称为boss，负责接收连接请求，另外一个称为worker，负责处理具体的channel。EventLoopGroup是基于JDK的线程池进行封装的实现，boss和worker可以设置不同的线程数，默认情况下，boss线程池中线程数量为1个，worker中的线程数量为2*CPU。
2. ServerBootstrap是服务端启动的引导类。
3. 指定使用NioServerSocketChannel来建立请求连接。
4. 构造一系列channelHandler处理链来组成ChannelPipeline。
5. option用来配置一些channel的参数，配置的参数会被ChannelConfig使用。
6. 另外：示例中的DiscardServerHandler是继承了ChannelInboundHandlerAdapter，复写了channelRead和exceptionCaught方法。

## 官网客户端最佳实践

```java
public class TimeClient {
    public static void main(String[] args) throws Exception {
        String host = args[0];
        int port = Integer.parseInt(args[1]);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        
        try {
            Bootstrap b = new Bootstrap(); // (1)
            b.group(workerGroup); // (2)
            b.channel(NioSocketChannel.class); // (3)
            b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
            b.handler(new ChannelInitializer<SocketChannel>() {
                @Override
                public void initChannel(SocketChannel ch) throws Exception {
                    ch.pipeline().addLast(new TimeClientHandler());
                }
            });
            
            // Start the client.
            ChannelFuture f = b.connect(host, port).sync(); // (5)

            // Wait until the connection is closed.
            f.channel().closeFuture().sync();
        } finally {
            workerGroup.shutdownGracefully();
        }
    }
}
```

1.  客户端引导类使用Bootstrap。
2. 客户端只需要一个NioEventLoopGroup即可,因为无需单独处理连接和IO操作。
3. 指定建立连接的方式为NioSocketChannel。
4. option用来配置一些channel的参数，配置的参数会被ChannelConfig使用。
5. 连接到指定的IP和端口



## Netty的核心组件简介

1. ServerBootStrap、BootStrap

   - ServerBootStrap是服务端应用启动引导类，使用这个类只需绑定本地端口即可，简单的使用方法即是使用main方法调用即可，无需依赖Tomcat、WebLogic容器等。


   - BootStrap是客户端应用启动引导类，此类需要绑定连接到服务端的host和port，也可以使用main方法直接启动即可，不依赖于Tomcat、Jboss容器。

2. NioEventLoopGroup、NioEventLoop

   服务端一般可使用一个或两个NioEventLoopGroup（推荐）来处理channel（等同于JDK中的socket）的连接及IO操作。NioEventLoopGroup是NIO的EventLoopGroup具体实现，而EventLoopGroup也继承了ScheduledExecutorService，即NioEventLoopGroup本身是一个线程池，那么NioEventLoopGroup也提供了一些构造方法去构造NioEventLoopGroup。一般boss的NioEventLoopGroup的线程大小为1，worker的NioEventLoopGroup为2*cpu。boss的NioEventLoopGroup负责处理 客户端的连接请求，worker的NioEventLoopGroup负责处理具体的IO操作等。

   EventLoop 定义了 Netty 的核心抽象，用于处理连接的生命周期中所发生的事件。NioEventLoop是具体处理channel连接及IO操作的类，一个channel在其生命周期只和一个NioEventLoop绑定，一个NioEventLoop可以同时处理多个channel。NioEventLoop使用select（）不断轮询其管理的channel，直至接收到客户端的数据。

3. NioServerSocketChannel、NioSocketChannel

   NioServerSocketChannel是netty为非阻塞同步IO提供的服务端socket连接类，NioSocketChannel是netty为非阻塞同步IO提供的客户端socket连接类。

   bootstrap类在引导服务时指定channel类型，服务端接收新连接时便使用NioServerSocketChannel创建新的channel，客户端使用NioSocketChannel建立连接。

4. ChannelHandler、ChannelInboundHandler、ChannelOutboundHandler

   - ChannelHandler接口为处理入站和出站数据的应用程序逻辑的容器。
   - ChannelInboundHandler、ChannelOutboundHandler是ChannelHandler的子接口，分别是进站时处理类的接口和出站时处理类的接口。即ChannelInboundHandler提供了服务端接收数据时处理的接口，提供了一些方法，如channelRegisteredchannelUnregistered，channelActive，channelInactive，channelRead方法等，ChannelOutboundHandler提供了服务端返回数据时做一些处理的接口。

5. ChannelInboundHandlerAdapter、ChannelOutboundHandlerAdapter

   ChannelInboundHandlerAdapter和ChannelOutboundHandlerAdapter分别是ChannelInboundHandler和ChannelOutboundHandler的基础实现类，我们自定义的handler最好继承这两个，尽可能只重写自己需要的方法。

6. ChannelPipeline

   ChannelPipeline控制着ChannelHandler的流转，所有的ChannelHandler都在ChannelPipeline上流转。

7. ChannelHandlerContext

   ChannelHandlerContext是ChannelPipeline与ChannelHandler之间的上下文。


8. ByteBuf

   ByteBuf是netty提供的操作字节码的类，比JDK中的ByteBuffer提供了更为方便的操作字节码的方法。ByteBuf中同时保存了readIndex和writeIndex，读写互不干扰。

9. ChannelFuture

   ChannelFuture是netty中异步IO操作的结果，在netty中所有的IO操作都是异步的，在使用ChannelFuture时需先判断操作已结束且成功。即isDone&&isSuccess。

10. SocketChannel、ServerSocketChannel

    SocketChannel等同于JDK中的socket，ServerSocketChannel等同于JDK中的ServerSocket。

## 服务端启动解析

源码分析参考自美团大神的文章https://www.jianshu.com/p/c5068caab217

官网的netty服务端demo中，（3）、（4）、（5）、（6）均为服务端的一些配置。

 ChannelFuture f = b.bind(port).sync(); // (7) 这里才是真正的启动过程。

点进去查看源码。

```java
public ChannelFuture bind(int inetPort) {
    return bind(new InetSocketAddress(inetPort));
}
```

创建了一个InetSocketAddress新对象，并调用了bind()方法。

```java
public ChannelFuture bind(SocketAddress localAddress) {
    //校验，EventLoopGroup和ChannelFactory不能为空。
    validate();
    if (localAddress == null) {
        throw new NullPointerException("localAddress");
    }
    return doBind(localAddress);
}
```

首先就是validate方法，校验EventLoopGroup和ChannelFactory不能为空。EventLoopGroup是我们在（2）就已经绑定了的，ChannelFactory先不用管。先继续看大致流程。

然后调用了doBind()方法。

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    final ChannelFuture regFuture = initAndRegister();
    //.......省略些判断等细枝末节
    final Channel channel = regFuture.channel();
    //.......省略些判断等细枝末节
  	doBind0(regFuture, channel, localAddress, promise);
}
```

从源码中可以看出，doBind（）方法主要做了两件事情，第一件就是initAndRegister，从方法名可猜测是初始化以及注册，另一件是绑定了什么。

先看下initAndRegister方法。

```java
final ChannelFuture initAndRegister() {
    //...创建了一个新的channel
    Channel channel = channelFactory.newChannel();
    //...初始化channel
    init(channel);
    //...
    ChannelFuture regFuture = config().group().register(channel);
}
```

initAndRegister方法主要做了三件事，一是创建了新的channel，二是初始化channel，三是注册了channel。每个方法我们都进去细看一下。

注意，通过debug可以发现，这个channelFactory在之前就已new出来了，实现类为ReflectiveChannelFactory，我们看下这个实现类的newChannel方法。

```java
public T newChannel() {
    try {
        //这个class就是之前指定的channel类型，NioServerSocketChannel
        return clazz.getConstructor().newInstance();
    } catch (Throwable t) {
        throw new ChannelException("Unable to create Channel from class " + clazz, t);
    }
}
```

这个方法其实就是调用了默认构造函数，new了一个新对象。

这里看一下NioServerSocketChannel的默认构造函数。

```java
public NioServerSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

再看一下这里调用的super父类方法。

```java
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    //readInterestOp->SelectionKey.OP_ACCEPT
    this.readInterestOp = readInterestOp;
   //...
   ch.configureBlocking(false);//设置为非阻塞IO，典型的NIO
}
```

这里的 `readInterestOp` 即前面层层传入的 `SelectionKey.OP_ACCEPT`，接下来重点分析 `super(parent);`(这里的parent其实是null，由前面写死传入)。

再看一下 这里的super()方法。

```java
protected AbstractChannel(Channel parent) {
    //AbstractChannel与ChannelId、Unsafe、ChannelPipeline绑定
    this.parent = parent;
    id = newId();//DefaultChannelId
    unsafe = newUnsafe();//NioMessageUnsafe
    pipeline = newChannelPipeline();//DefaultChannelPipeline
}
```

这里new了三个对象，分别是ChannelId，Unsafe，DefaultChannelPipeline。这里只是new了三个对象，先不着急他们是做什么的，继续追代码。

再看一下init（）方法。

```java
void init(Channel channel) throws Exception {
    //这里的channel就是刚刚创建的channel->NioServerSocketChannel
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }
	//这个pipeline是NioServerSocketChannel的父类的构造方法中new出来的DefaultChannelPipeline
    ChannelPipeline p = channel.pipeline();
	//这里的childGroup就是new出来的workerEventLoopGroup
    final EventLoopGroup currentChildGroup = childGroup;
    //childHandler--->server端 new出的ChannelInitializer
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
    }

    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(final Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

这个方法太长了，分成几个部分理解。

这一部分主要是先分别通过options0方法和attrs方法获取到options和attrs，再设置到ChannelConfig或Channel中。

```java
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        setChannelOptions(channel, options, logger);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }
```

再看这一部分。

```java
ChannelPipeline p = channel.pipeline();

final EventLoopGroup currentChildGroup = childGroup;
final ChannelHandler currentChildHandler = childHandler;
final Entry<ChannelOption<?>, Object>[] currentChildOptions;
final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
synchronized (childOptions) {
    currentChildOptions = childOptions.entrySet().toArray(newOptionArray(0));
}
synchronized (childAttrs) {
    currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(0));
}

p.addLast(new ChannelInitializer<Channel>() {
    @Override
    public void initChannel(final Channel ch) throws Exception {
        final ChannelPipeline pipeline = ch.pipeline();
        //这里的config是ServerBootstrapConfig，绑定了当前的ServerBootstrap
        //这里的config.handler，其实就是ServerBootstrap中调用.handler()方法绑定的那个handler，可能为空，不调用时就为空。
        ChannelHandler handler = config.handler();
        if (handler != null) {
            pipeline.addLast(handler);
        }

        ch.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                pipeline.addLast(new ServerBootstrapAcceptor(
                        ch, currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
            }
        });
    }
});
```

可以看到，这里构造的currentChildGroup、currentChildHandler、currentChildOptions、currentChildAttrs都是用来构造ServerBootstrapAcceptor的，这个ServerBootstrapAcceptor从名字上就可以看出来，这是一个接入器，专门接受新请求，把新的请求扔给某个事件循环器，我们先不做过多分析。

总结一下init方法：

1. 构造出NioServerSocketChannel，channel内部绑定了channelId、channelPipeline、以及Unsafe。
2. 将ServerBootstrap绑定的option设置到channel绑定的ChannelConfig中
3. 将ServerBootstrap绑定的attr设置到channel的attr中
4. 如果ServerBootstrap调用了handler方法，将绑定的handler添加到pipeline最后面
5. 异步构造一个ServerBootstrapAcceptor，并将其添加到pipeline最后面。

register方法分析。

```java
//这里的config是ServerBootstrapConfig，绑定了当前的ServerBootstrap，调用了group方法就是获取到workerEvenetLoopGroup
ChannelFuture regFuture = config().group().register(channel);
```

点击register方法

SingleThreadEventLoop

```java
public ChannelFuture register(Channel channel) {
    //构造了一个DefaultChannelPromise，包含了当前NioServerSocketChannel以及当前ServerBootstrap
    //DefaultChannelPromise是ChannelFuture的一个实现类
    return register(new DefaultChannelPromise(channel, this));
}
```

构造了一个DefaultChannelPromise对象再调用register方法。

```java
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    //这里的unsafe具体实现类是NioMessageUnsafe
    promise.channel().unsafe().register(this, promise);
    return promise;
}
```

从上面的方法可以看到，unsafe()方法的返回值再次调用了register方法，点击查看详情。

AbstractUnsafe,AbstractChannel的内部类

```java
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    //....
    //eventLoop.inEventLoop()就是判断是否是当前线程
    if (eventLoop.inEventLoop()) {
        //是当前线程就直接处理
        register0(promise);
    } else {
        try {
            //不是当前线程就开个线程再处理
            eventLoop.execute(new Runnable() {
                @Override
                public void run() {
                    register0(promise);
                }
            });
        } catch (Throwable t) {
            logger.warn(
                    "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                    AbstractChannel.this, t);
            closeForcibly();
            closeFuture.setClosed();
            safeSetFailure(promise, t);
        }
    }
}
```

不同的逻辑都调用了register0方法

```java
private void register0(ChannelPromise promise) {
    try {
     	//.....
        boolean firstRegistration = neverRegistered;
        doRegister();
        neverRegistered = false;
        registered = true;
        //handle added
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        //handle registered
        pipeline.fireChannelRegistered();
       
        if (isActive()) {
            if (firstRegistration) {
                
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                beginRead();
            }
        }
    } catch (Throwable t) {
        // Close the channel directly to avoid FD leak.
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```



```java
@Override
protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            //javaChannel()方法返回的是NioServerSocketChannel
            selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                // Force the Selector to select now as the "canceled" SelectionKey may still be
                // cached and not removed because no Select.select(..) operation was called yet.
                eventLoop().selectNow();
                selected = true;
            } else {
                // We forced a select operation on the selector before but the SelectionKey is still cached
                // for whatever reason. JDK bug ?
                throw e;
            }
        }
    }
}
```



```java
public final SelectionKey register(Selector sel, int ops,
                                   Object att)
    throws ClosedChannelException
{
//这里的att是AbstractNioChannel
    synchronized (regLock) {
        if (!isOpen())
            throw new ClosedChannelException();
        if ((ops & ~validOps()) != 0)
            throw new IllegalArgumentException();
        if (blocking)
            throw new IllegalBlockingModeException();
        SelectionKey k = findKey(sel);
        if (k != null) {
            k.interestOps(ops);
            k.attach(att);
        }
        if (k == null) {
            // New registration
            synchronized (keyLock) {
                if (!isOpen())
                    throw new ClosedChannelException();
                k = ((AbstractSelector)sel).register(this, ops, att);
                addKey(k);
            }
        }
        return k;
    }
}
```

register方法总结：

1. 构造了一个DefaultChannelPromise类，这个类是ChannelFuture的实现类
2. 判断当前线程是否是启动时的线程，是则直接调用register0方法，否则新开线程调用register0方法，原因赞不理解。
3. 在某种条件下（未知，详见AbstractSelectableChannel的register方法和findKey方法），将AbstractNioChannel的实现类（这里其实是NioServerSocketChannel）绑定到SelectionKey
4. 调用handler()方法绑定的channelhandler中的channelAdded、channelRegister等方法

至于doBind0方法，比较复杂，请参考另一篇文章《netty源码之NioEventLoop》。

总结：

netty启动一个服务经过的所有过程：

1.设置启动参数，如负责接收请求的boss线程组，负责处理请求的worker线程组，设置channel类型，绑定端口等。

2.创建server对应的channel，创建各大组件，包括ChannelConfig，ChannelId，ChannelPipeline，ChannelHandler，Unsafe等。

3.初始化server对应的channel，设置attr及options等，给server的channel添加接入器，触发register事件，调用handler（）方法中绑定的channelhandler的channelAdd、channelRegister等方法。

4.调用到jdk底层做端口绑定，并触发active事件，active触发的时候，真正做服务端口绑定



参考自大神博客，大神原文贴上https://www.jianshu.com/p/c5068caab217