## 一、继承结构

![image-20181102181729701](/Users/wangwangxiaoteng/Library/Application Support/typora-user-images/image-20181102181729701.png)

从上图可以看到，NioEventLoop实际上是一个线程池，继承了netty中的抽象类SingleThreadEventExecutor，追本逐源，而SingleThreadEventExecutor本身又是Excutor的实现类。

## 源码

调用关系：ServeBootstrap.bind()->doBind()->doBind0()->SingleThreadEventExecutor.execute()

```java
public void execute(Runnable task) {
    //判断是否是当前reactor线程
    boolean inEventLoop = inEventLoop();//此时SingleThreadEventExecutor子类中全局变量thread为空
    //向task队列 taskQueue中添加任务  task->Runnable，这里的taskQueue的实现类是LinkedBlockingQueue
    addTask(task);
    if (!inEventLoop) {
        startThread();
        //.....
    }
}
private void startThread() {
    if (state == ST_NOT_STARTED) {
        //CAS 比较只有当字段状态为ST_NOT_STARTED时，CAS获得锁，才调用doStartThread()方法
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            try {
                doStartThread();
            } catch (Throwable cause) {
                STATE_UPDATER.set(this, ST_NOT_STARTED);
                PlatformDependent.throwException(cause);
            }
        }
    }
}
```

SingleThreadEventExecutor的execute方法调用了自己的startThread方法，startThread又调用了doStartThread方法。

```java
private void doStartThread() {
    assert thread == null;
    executor.execute(new Runnable() {
        @Override
        public void run() {
            //将全局变量thread赋值，避免其他线程并发执行
            thread = Thread.currentThread();
            if (interrupted) {
                thread.interrupt();
            }
            	//调用了子类的run方法
                SingleThreadEventExecutor.this.run();
                success = true;
        }
    });
}
```

NioEventLoop正好是SingleThreadEventExecutor的子类，看下NioEventLoop的run方法。

```java
protected void run() {
    //一直在循环
    for (;;) {
        try {
            //判断当前是否有task
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    //无task，select轮询channel
                    //wakenUp=new AomicBoolean();wakenUp初始值为false，每次select将wakenup设置为false
                    select(wakenUp.getAndSet(false));
                    if (wakenUp.get()) {
                        selector.wakeup();
                    }
                default:
            }
            		//.....
                    processSelectedKeys();   
                    runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        // Always handle shutdown even if the loop processing threw an exception.
    }
}
```

暂且先不分析calculateStrategy方法，先看下面的主干。

分别是select方法，processSelectedKeys方法及runAllTasks方法。

从方法名可以简单猜下，这三个方法分别是负责循环处理channel，返回selectedKey，处理selectedKeys，并封装成NioTask，及执行这些NioTasks。

分别仔细看下这三个方法。

### select方法。

```java
private void select(boolean oldWakenUp) throws IOException {
        Selector selector = this.selector;
        try {
            int selectCnt = 0;
            long currentTimeNanos = System.nanoTime();
            long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

            for (;;) {
                long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
                if (timeoutMillis <= 0) {
                    //当select超时时，结束循环
                    if (selectCnt == 0) {
                        selector.selectNow();
                        selectCnt = 1;
                    }
                    break;
                }

                // If a task was submitted when wakenUp value was true, the task didn't get a chance to call
                // Selector#wakeup. So we need to check task queue again before executing select operation.
                // If we don't, the task might be pended until select operation was timed out.
                // It might be pended until idle timeout if IdleStateHandler existed in pipeline.
                //如果有任务队列有任务的时候，且乐观锁获得锁时，退出循环
                if (hasTasks() && wakenUp.compareAndSet(false, true)) { 
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                int selectedKeys = selector.select(timeoutMillis);
                selectCnt ++;
                long time = System.nanoTime();
                if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
                    // timeoutMillis elapsed without anything selected.
                    selectCnt = 1;
                } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
                        selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
                    //当select循环次数超过一定次数时，退出循环，避免空轮训，并且重建selector
                    rebuildSelector();
                    selector = this.selector;

                    // Select again to populate selectedKeys.
                    selector.selectNow();
                    selectCnt = 1;
                    break;
                }

                currentTimeNanos = time;
            }
        } catch (CancelledKeyException e) {
            if (logger.isDebugEnabled()) {
                logger.debug(CancelledKeyException.class.getSimpleName() + " raised by a Selector {} - JDK bug?",
                        selector, e);
            }
            // Harmless exception - log anyway
        }
    }
```

总结select()方法：

1. 循环处理channel
2. 判断当前时间是否超过deadline，超过则跳出循环，避免空轮训
3. 调用selector.select()方法处理连接channel
4. 判断当前轮询次数是否超过阈值，超过则重建selector，避免空轮训bug。重建selector时，如果发生异常，若selectionKey 为AbstractNioChannel或其子类，则关闭channel，否则调用channel的取消注册方法

## 再看processSelectedKeys方法。

```java
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized();
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}
private void  processSelectedKeysOptimized() {
    for (int i = 0; i < selectedKeys.size; ++i) {
        final SelectionKey k = selectedKeys.keys[i];
        // null out entry in the array to allow to have it GC'ed once the Channel close
        // See https://github.com/netty/netty/issues/2363
        selectedKeys.keys[i] = null;

        final Object a = k.attachment();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (needsToSelectAgain) {
            // null out entries in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.reset(i + 1);

            selectAgain();
            i = -1;
        }
    }
}
private void processSelectedKeysPlain(Set<SelectionKey> selectedKeys) {
    // check if the set is empty and if so just return to not create garbage by
    // creating a new Iterator every time even if there is nothing to process.
    // See https://github.com/netty/netty/issues/597
    //判空，
    if (selectedKeys.isEmpty()) {
        return;
    }

    Iterator<SelectionKey> i = selectedKeys.iterator();
    for (;;) {
        //遍历SelectionKey
        final SelectionKey k = i.next();
        final Object a = k.attachment();
        i.remove();

        if (a instanceof AbstractNioChannel) {
            processSelectedKey(k, (AbstractNioChannel) a);
        } else {
            @SuppressWarnings("unchecked")
            NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
            processSelectedKey(k, task);
        }

        if (!i.hasNext()) {
            break;
        }

        if (needsToSelectAgain) {
            selectAgain();
            selectedKeys = selector.selectedKeys();

            // Create the iterator again to avoid ConcurrentModificationException
            if (selectedKeys.isEmpty()) {
                break;
            } else {
                i = selectedKeys.iterator();
            }
        }
    }
}
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
   
    try {
        int readyOps = k.readyOps();
        // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
        // the NIO JDK channel implementation may throw a NotYetConnectedException.
        if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
            // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
            // See https://github.com/netty/netty/issues/924
            int ops = k.interestOps();
            ops &= ~SelectionKey.OP_CONNECT;
            k.interestOps(ops);
			//完成连接
            unsafe.finishConnect();
        }

        // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
        if ((readyOps & SelectionKey.OP_WRITE) != 0) {
            // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
            //刷新buffer，空出来给队列的buffer使用
            ch.unsafe().forceFlush();
        }

        // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
        // to a spin loop
        if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
            //调用read方法，处理接收的消息
            unsafe.read();
        }
    } catch (CancelledKeyException ignored) {
        unsafe.close(unsafe.voidPromise());
    }
}
private static void processSelectedKey(SelectionKey k, NioTask<SelectableChannel> task) {
    int state = 0;
    try {
        //这里NioTask为接口，未找到其实现类，不知道这里究竟做了什么操作
        task.channelReady(k.channel(), k);
        state = 1;
    } catch (Exception e) {
        k.cancel();
        invokeChannelUnregistered(task, k, e);
        state = 2;
    } finally {
        switch (state) {
        case 0:
            k.cancel();
            invokeChannelUnregistered(task, k, null);
            break;
        case 1:
            if (!k.isValid()) { // Cancelled by channelReady()
                invokeChannelUnregistered(task, k, null);
            }
            break;
        }
    }
}
```

总结processSelectedKeys方法：

1. 遍历SelectionKey
2. 如果SelectionKey.attachment为AbstractNioChannel或其子类，先调用channel的finnishConnect方法，说明连接已建立，再强制刷新buffer，供队列存储task用，再执行unsafe.read()方法，处理消息。
3. 如果SelectKey.attachment为NioTask，调用NioTask的channelReady方法。这里没有找到NioTask的具体实现类，无法得知里面做了哪些操作。

分析一下processSelectedKey方法中，调用了Unsafe的子类NioMessageUnsafe.read()方法。

```java
public void read() {
        assert eventLoop().inEventLoop();
        final ChannelConfig config = config();
        final ChannelPipeline pipeline = pipeline();
        final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
        allocHandle.reset(config);

        boolean closed = false;
        Throwable exception = null;
        //..........
    	//这个doReadMessages就是通过NioServerSocketChannel和SocketChannel构造了一个NioSocketChannel，并将NioSocketChannel加进readBuf（是一个list集合，泛型为Object）中
        int localRead = doReadMessages(readBuf);

            int size = readBuf.size();
            for (int i = 0; i < size; i ++) {
                readPending = false;
                //channelPipeline（最后跳转到ServerBootstrapAcceptor.channelRead方法）处理刚刚添加的NioSocketChannel
                pipeline.fireChannelRead(readBuf.get(i));
            }
            readBuf.clear();
            allocHandle.readComplete();
    		//调用处理读取完毕的方法
            pipeline.fireChannelReadComplete();

            if (exception != null) {
                closed = closeOnReadError(exception);
				//调用处理异常的方法
                pipeline.fireExceptionCaught(exception);
            }

            if (closed) {
                inputShutdown = true;
                if (isOpen()) {
                    close(voidPromise());
                }
            }
        } 
    }
}
```

ServerBootStrap内部类ServerBootstrapAcceptor.channelRead()方法。

```java
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    //这里的msg就是前面传入的NioSocketChannel，所以可以直接强转
    final Channel child = (Channel) msg;
	//将ServerBootstrap启动时配置的childHandler加入此child的pipeline中
    child.pipeline().addLast(childHandler);

    setChannelOptions(child, childOptions, logger);

    for (Entry<AttributeKey<?>, Object> e: childAttrs) {
        child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
    }

    try {
        //NioEventLoopGroup的register方法
        childGroup.register(child).addListener(new ChannelFutureListener() {
            @Override
            public void operationComplete(ChannelFuture future) throws Exception {
                if (!future.isSuccess()) {
                    forceClose(child, future.cause());
                }
            }
        });
    } catch (Throwable t) {
        forceClose(child, t);
    }
}
```

NioEventLoopGroup父类MultithreadEventLoopGroup的register方法

```java
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}
```

register()方法就跳转到NioEventLoop的register()方法了，之前文章已分析，这里不做介绍。

关注点应该在next()方法上。

```java
public EventExecutor next() {
    //这个choose的初始化是        chooser = chooserFactory.newChooser(children);
    //chooserFactory的实例类是DefaultEventExecutorChooserFactory
    return chooser.next();
}
public EventExecutorChooser newChooser(EventExecutor[] executors) {
    if (isPowerOfTwo(executors.length)) {
        //如果线程的数量是2的次幂数
        return new PowerOfTwoEventExecutorChooser(executors);
    } else {
        return new GenericEventExecutorChooser(executors);
    }
}

private static boolean isPowerOfTwo(int val) {
    return (val & -val) == val;
}

private static final class PowerOfTwoEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTwoEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }

    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
```

两种类型的选择器在选择reactor线程的时候，都是通过Round-Robin轮询的方式选择reactor线程，唯一不同的是，`PowerOfTowEventExecutorChooser`是通过与运算，而`GenericEventExecutorChooser`是通过取余运算，与运算的效率要高于求余运算。

### 最后看一下runAllTasks方法

```java
protected boolean runAllTasks() {
    assert inEventLoop();
    boolean fetchedAll;
    boolean ranAtLeastOne = false;

    do {
        //至少执行一次
        //从定时任务队列中获得对列头部
        fetchedAll = fetchFromScheduledTaskQueue();
        if (//执行队列中 的所有任务
            runAllTasksFrom(taskQueue)
        ) {
            ranAtLeastOne = true;
        }
    } while (!fetchedAll); // keep on processing until we fetched all scheduled tasks.

    if (ranAtLeastOne) {
        lastExecutionTime = ScheduledFutureTask.nanoTime();
    }
    afterRunningAllTasks();
    return ranAtLeastOne;
}
private boolean fetchFromScheduledTaskQueue() {
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    //从scheduledTaskQueue中获得head节点任务
    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    while (scheduledTask != null) {
        //获取到头节点
        if (!taskQueue.offer(scheduledTask)) {
            //如果taskQueue没加成功head节点任务，将head节点任务重新放入scheduledTaskQueue
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
        scheduledTask  = pollScheduledTask(nanoTime);
    }
    return true;
}
protected final boolean runAllTasksFrom(Queue<Runnable> taskQueue) {
    //从taskQueue中获取头节点
    Runnable task = pollTaskFrom(taskQueue);
    if (task == null) {
        return false;
    }
    //遍历taskQueue执行NioTask
    for (;;) {
        //安全执行task任务
        safeExecute(task);
        //再从taskQueue中获取头部节点任务
        task = pollTaskFrom(taskQueue);
        if (task == null) {
            return true;
        }
    }
}
```

safeExecute方法其实就是调用了Runnable的run方法，只不过会忽略抛出的所有异常

```java
protected static void safeExecute(Runnable task) {
    try {
        task.run();
    } catch (Throwable t) {
        logger.warn("A task raised an exception. Task: {}", task, t);
    }
}
```

总结一下：

select方法主要是轮询处理NIO中的channel，processSelectedKeys方法主要是处理SelectedKey，SelectedKey分为AbstractNioChannel和NioTask。runAllTasks方法就是worker线程具体处理NioTask的方法了。
