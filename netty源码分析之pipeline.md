## 初始化

我们知道，channel在初始化的时候就会将channelId，pipeline，unsafe等初始化。 

AbstractChannel的构造方法

```java
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    id = newId();
    unsafe = newUnsafe();
    pipeline = newChannelPipeline();//DefaultChannelPipeline
}
```

其中，pipeline初始化的newChannelPipeline()方法实现如下：

```java
protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}
```

`DefaultChannelPipeline`

```java
protected DefaultChannelPipeline(Channel channel) {
    //保存了channel的引用
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);
	//初始化头结点、尾节点
    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```

默认情况下，也就是pipeline刚初始化的时候，只有两个节点。每一个节点都是一个ChannelHandlerContext对象（TailContext和HeadContext也是ChannelHandlerContext的实现类）。

## 添加新节点

pipeline常用添加新节点方式：addLast

```java
public final ChannelPipeline addLast(String name, ChannelHandler handler) {
    return addLast(null, name, handler);
}

@Override
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
    final AbstractChannelHandlerContext newCtx;
    synchronized (this) {
        //1.校验当前handler是否是@sharable注解标注及是否已被添加
        checkMultiplicity(handler);
		//2.创建新的ChannelHandlerContext
        newCtx = newContext(group, filterName(name, handler), handler);
		//3.将新节点加入pipeline
        addLast0(newCtx);

        // If the registered is false it means that the channel was not registered on an eventloop yet.
        // In this case we add the context to the pipeline and add a task that will call
        // ChannelHandler.handlerAdded(...) once the channel is registered.
        if (!registered) {
            newCtx.setAddPending();
            callHandlerCallbackLater(newCtx, true);
            return this;
        }

        EventExecutor executor = newCtx.executor();
        if (!executor.inEventLoop()) {
            newCtx.setAddPending();
            executor.execute(new Runnable() {
                @Override
                public void run() {
                    //回调用户方法中自定义的handler中的handlerAdded方法
                    callHandlerAdded0(newCtx);
                }
            });
            return this;
        }
    }
    //回调用户方法中自定义的handler中的handlerAdded方法
    callHandlerAdded0(newCtx);
    return this;
}
```

### checkMultiplicity方法

```java
private static void checkMultiplicity(ChannelHandler handler) {
    if (handler instanceof ChannelHandlerAdapter) {
        //强转为ChannelHandlerAdapter
        ChannelHandlerAdapter h = (ChannelHandlerAdapter) handler;
        if (!h.isSharable() && h.added) {
            //如果ChannelHandlerAdapter不是@Sharable注解标识且已经添加过，抛出异常
            throw new ChannelPipelineException(
                    h.getClass().getName() +
                    " is not a @Sharable handler, so can't be added or removed multiple times.");
        }
        //标识此实例已经添加过
        h.added = true;
    }
}
```

### newContext方法

先看`filterName`方法

```java
private String filterName(String name, ChannelHandler handler) {
    if (name == null) {
        //如果名称为空，则自动生成一个name
        return generateName(handler);
    }
    //校验name是否重复
    checkDuplicateName(name);
    return name;
}
```

`generateName`

```java
private String generateName(ChannelHandler handler) {
    //nameCaches是一个FastThreadLocal
    Map<Class<?>, String> cache = nameCaches.get();
    Class<?> handlerType = handler.getClass();
    String name = cache.get(handlerType);
    if (name == null) {
        //name缓存中没有的话，就生成一个新的name，以simpleClassName+"#0"
        name = generateName0(handlerType);
        cache.put(handlerType, name);
    }

    //用户不太可能将同一类型的多个处理程序放置在一起，但避免
    if (context0(name) != null) {
        String baseName = name.substring(0, name.length() - 1); // Strip the trailing '0'.
        for (int i = 1;; i ++) {
            String newName = baseName + i;
            if (context0(newName) == null) {
                name = newName;
                break;
            }
        }
    }
    return name;
}
private static String generateName0(Class<?> handlerType) {
        return StringUtil.simpleClassName(handlerType) + "#0";
}
```

`newContext`

```java
private AbstractChannelHandlerContext newContext(EventExecutorGroup group, String name, ChannelHandler handler) {
    //由前面方法可知，这里group为null
    return new DefaultChannelHandlerContext(this, childExecutor(group), name, handler);
}

private EventExecutor childExecutor(EventExecutorGroup group) {
    //由前面方法可知，这里group为null，所以返回null
    if (group == null) {
        return null;
    }
    //.....
}
```

看一下DefaultChannelHandlerContext的构造函数

```java
DefaultChannelHandlerContext(
        DefaultChannelPipeline pipeline, EventExecutor executor, String name, ChannelHandler handler) {
    //pipeline为当前pipeline对象，executor为null，handler为add Last传入的handler
    //isInbound和isOutbound就是分别看下handler是否是ChannelInboundHandler或ChannelOutboundHandler的实现类
    super(pipeline, executor, name, isInbound(handler), isOutbound(handler));
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
}
```

再追一下父类的构造函数

```java
AbstractChannelHandlerContext(DefaultChannelPipeline pipeline, EventExecutor executor, String name,
                              boolean inbound, boolean outbound) {
    this.name = ObjectUtil.checkNotNull(name, "name");
    this.pipeline = pipeline;
    this.executor = executor;
    this.inbound = inbound;
    this.outbound = outbound;
    // Its ordered if its driven by the EventLoop or the given Executor is an instanceof OrderedEventExecutor.
    ordered = executor == null || executor instanceof OrderedEventExecutor;
}
```

### addLast0方法

channelHandlerContext创建完后，就要把context加入到pipeline中了。

```java
private void addLast0(AbstractChannelHandlerContext newCtx) {
    //双向链表的插入操作，插入后，context就在pipeline中了，在tail节点前
    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;
}
```

### callHandlerAdded0方法，回调用户方法

```java
private void callHandlerAdded0(final AbstractChannelHandlerContext ctx) {
    try {
        // We must call setAddComplete before calling handlerAdded. Otherwise if the handlerAdded method generates
        // any pipeline events ctx.handler() will miss them because the state will not allow it.
        ctx.setAddComplete();
        //context绑定了handler，直接获取context绑定的handler，调用handler的handlerAdded方法。
        ctx.handler().handlerAdded(ctx);
    } catch (Throwable t) {
        boolean removed = false;
        try {
            //回调出现异常，remove掉
            remove0(ctx);
            try {
                ctx.handler().handlerRemoved(ctx);
            } finally {
                ctx.setRemoved();
            }
            removed = true;
        } catch (Throwable t2) {
            if (logger.isWarnEnabled()) {
                logger.warn("Failed to remove a handler: " + ctx.name(), t2);
            }
        }

        if (removed) {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; removed.", t));
        } else {
            fireExceptionCaught(new ChannelPipelineException(
                    ctx.handler().getClass().getName() +
                    ".handlerAdded() has thrown an exception; also failed to remove.", t));
        }
    }
}
```

DefaultChannelHandlerContext其实是一个双向链表结构，pipeline操作其实都是在操作DefaultChannelHandlerContext，DefaultChannelHandlerContext和handler、pipeline有关联，pipeline和channel也有关联关系。