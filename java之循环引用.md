
在学习java内存模型及垃圾回收时提到了引用计数法无法解决循环引用的问题，心里一直在思考怎么才是循环引用。
netty中的循环引用的例子。

例如：NioServerSocketChannel类中有内部类NioServerSocketChannelConfig。每一个NioServerSocketChannel实例对象都有全局变量NioServerSocketChannelConfig的实例对象，而NioServerSocketChannelConfig实例对象在构造的时候也是需要将NioServerSocketChannel实例对象引用进去。也就是NioServerSocketChannel实例对象和NioServerSocketChannelConfig实例对象相互引用了。也就是循环引用。

```java
public class NioServerSocketChannel extends AbstractNioMessageChannel
                             implements io.netty.channel.socket.ServerSocketChannel {

    private static final ChannelMetadata METADATA = new ChannelMetadata(false, 16);
    private static final SelectorProvider DEFAULT_SELECTOR_PROVIDER = SelectorProvider.provider();

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(NioServerSocketChannel.class);

  
	//NioServerSocketChannel实例对象的全局变量ServerSocketChannelConfig的实例
    private final ServerSocketChannelConfig config;
```

```java
public NioServerSocketChannel(ServerSocketChannel channel) {
    super(null, channel, SelectionKey.OP_ACCEPT);
    //在初始化config的时候将自己(NioServerSocketChannel的实例对象)也传入了
    config = new NioServerSocketChannelConfig(this, javaChannel().socket());
}
```

```java
private NioServerSocketChannelConfig(NioServerSocketChannel channel, ServerSocket javaSocket) {
    super(channel, javaSocket);
}
```