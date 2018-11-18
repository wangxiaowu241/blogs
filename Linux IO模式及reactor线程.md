注：本文主要参考大神文章，来源：https://segmentfault.com/a/1190000003063859

本文主要是学习大神文章后，自己总结下知识。

# 一、概念说明

在进行解释之前，先说明几个概念。

- 用户空间与内核空间
- 进程切换
- 进程阻塞
- 文件描述符
- 缓存I/O

## 用户空间与内核空间

现在操作系统普遍采用虚拟存储器，对32位操作系统而言，它的寻址空间，也就是虚拟存储空间最大可为4G(2的32次方)，意思是的单个进程的最大地址为4G。操作系统的核心是内核，独立与普通的应用程序（如微信，QQ等），可以访问受限的内存空间，也有访问底层设备的所有权限。为了保证内核进程的安全，用户进程不能直接操作内核空间，操作系统将虚拟空间划分为两部分，一部分为内核空间，供内核进程使用；另一部分为用户空间，供用户进程使用。

简单来说：每个进程的4G空间中，最高的1G都是一样的，是内核空间，剩余的3G为用户进程使用。

为什么要区分内核空间和用户空间？

在CPU的所有指令中，有些指令是比较危险的，如果错用，将导致系统崩溃，比如清理内存，设置时钟等。如果允许所有的程序都可以使用这些命令，系统崩溃概率大大增加。所以CPU指令分为特权指令和非特权指令，对于危险的指令，只允许操作系统及其相关模块使用，普通应用程序只能使用非特权指令。

## 进程切换

为了控制进程的执行，内核必须有能力正在挂起正在运行的进程，并恢复以前的某个进程的执行。这种行为被称为进程切换。因此，可以说，任何进程都是在操作系统的内核支持下运行的，是与内核紧密相关的。

比如说：I/O操作一直都是相对CPU执行来说是很耗时的，这时候就可以通过中断操作，在外部存储设备传输数据时，CPU切换到其他进程继续执行，等到数据传输完毕，再切换回之前进程，避免CPU一直在等待。

进程切换要经过的过程：

1. **保存处理机上下文，包括程序计数器和其他寄存器**
2. **保存PCB**
3. **把进程的PCB移入相应的队列，如就绪、在某事件阻塞等队列**
4. **选择另一个进程执行，并更新其PCB**
5. **更新内存管理的数据结构**
6. **恢复处理机上下文**

**很耗费资源。**

## 进程阻塞

正在执行的进程，由于期待某些事件的发生，或者依赖某些事件的发生，如请求系统资源失败、等待某种操作的完成（如IO操作）、新数据尚未到达或无新工作等，则系统给自动执行阻塞，使当前进程由运行状态转为阻塞状态。可见，进程的阻塞是进程本身的一种主动行为，也只有处于运行状态的进程才可以使自己变成阻塞状态。

**当进程进入阻塞状态时，是不占用CPU资源的。**

## 文件描述符

文件描述符在形式上是一个非负整数。实际上，它是一个索引值，指向内核为每一个进程所维护的该进程打开文件的记录表。当程序打开一个现有文件或者创建一个新文件时，内核向进程返回一个文件描述符。在程序设计中，一些涉及底层的程序编写往往会围绕着文件描述符展开。

Linux的设计思想便是把一切设备都视作文件。因此，文件描述符为在该系列平台上进行设备相关的编程实际上提供了一个统一的方法。

## 缓存I/O

缓存I/O也叫标准I/O，大多数文件系统的默认I/O操作都是缓存I/O。在Linux机制中，数据是先从磁盘或外部设备复制到内核空间的缓冲区，再从内核空间复制到用户空间。

好处：在一定程度上分离了内和空间和用户空间；可以一定程度上减少了和磁盘或外部设备的交互次数，从而提高性能。

坏处：数据在传输过程中需经过两次拷贝操作，一次是从磁盘或外部设备复制到内核空间，另一次是从内核空间复制到用户空间。

**现在也有DMA方式，直接从磁盘或外部设备复制到用户空间，不经过内核。**

# 二、IO模式

对于一次IO读取操作而言，数据先被复制到操作系统的内核缓冲区，然后才会从内核空间复制到用户空间。所以说一次IO read操作，会经历以下两个阶段：

1. 等待数据准备：拷贝到操作系统的内核缓冲区
2. 将数据从内核缓冲区拷贝到用户空间的进程中

正因为这两个阶段，Linux系统下有下面几种网络模型：

- **阻塞I/O（blocking I/O）**
- **非阻塞I/O（nonblocking I/O）**
- **I/O多路复用（I/O multiplexing）**
- **信号驱动I/O（signal driven I/O）**
- **异步I/O（asynchronous I/O）**

## 阻塞I/O（blocking I/O）

在Linux中，默认的socket都是阻塞I/O（blocking I/O）。

![1593755892-55c466c2b5fc5_articlex](/Users/wangwangxiaoteng/work/code/github/blogs/1593755892-55c466c2b5fc5_articlex.png)

当用户进程调用了recvfrom这个系统调用方法，kernel内核就开始了IO的第一个阶段，准备数据，对于网络IO来说，很多时候数据在一开始还没完全达到，比如说没有接受到一个完整的数据包，这个时候kernel就要等待足够的数据到来。这个过程需要等待，也就是说数据被拷贝到操作系统的缓冲区的这个过程中，用户进程是主动阻塞的。当kernel中的数据准备完成了，它就会将数据从内核缓冲区拷贝到用户进程的内存中，然后kernel返回OK，用户进程解除blocking状态，才重新运行起来。

**`所以，阻塞I/O的特点是在IO操作的两个阶段都block住了。`**

## 非阻塞I/O（non-blocking I/O）

Linux环境下，可以设置socket为non-blocking。当一个non-blocking I/O执行读操作时，流程如下：

![1505222224-55c466dda9803_articlex](/Users/wangwangxiaoteng/work/code/github/blogs/1505222224-55c466dda9803_articlex.png)

当用户发起read操作时，如果kernel中数据还未准备好，也就是第一个阶段还未完成时，它并不会block用户进程，而是会返回一个error。从用户进程角度来说，它发起一个read操作，在这个阶段它自己并没有block住，而是立刻得到了一个结果。用户进程判断结果是error时，它就知道数据还未准备完成，就可以再次发送read操作。当kernel内核进程中的数据准备完成后，而此时又收到了用户进程的read请求，那么这个时候就block用户进程，kernel将数据从内核缓冲区复制到用户进程中，然后返回OK。

**`所以，非阻塞I/O的特点是在IO操作的第一个阶段，用户进程是不断的询问kernel中的数据是否准备完成，在第二个阶段也是block住的。`**

## I/O多路复用（IO multiplexing）

IO multiplexing多路复用，也就是常说的event-driven IO，主要是通过select、poll、epoll函数来操作IO。select/poll/epoll的好处在于可以同时处理多个网络连接的IO。它的基本原理就是select、poll、epoll会不断轮询所负责的所有socket，当某个socket有数据准备完成了，就通知用户进程。

![1903235121-55c466eb17665_articlex](/Users/wangwangxiaoteng/work/code/github/blogs/1903235121-55c466eb17665_articlex.png)

当用户进程调用selec、poll、epoll函数时，用户进程是block住的，同时kernel会负责监控所有select负责的socket，当任何一个socket中的数据准备完成了，select就会返回，这个时候用户进程就解除block，再调用read操作，kernel再将数据从内和缓冲区拷贝到用户进程。

**`所以，I/O多路复用的特点是在第一个阶段，用户进程可以同时监控多个socket连接，同时block住，当监控的任意一个socket数据准备完成，用户进程就可以进行read操作。`**

可以看出IO多路复用和blocking IO的处理思路基本一致，但是它的优势在于可以同时处理多个连接，在高并发时候有非常高的性能。但是在并发量比较小的时候，它的性能并不一定比多线程+blocking IO要强，可能还要更弱一些，因为blocking IO只需要一次system call，即在kernel将数据完整的拷贝到用户进程后。而IO multiplexing在select函数返回及kernel拷贝数据到用户进程完成后分别有一次system call。

在IO multiplexing model 中，实际上每一个socket个都设置成为non-blocking，但是，如上图所示，用户进程实际上是block住的，只不过是被select函数block住的，而不是被socket IO block住的。



## 异步IO （asynchronous IO）

异步IO实际上用的比较少，它的基本思路是，用户进程发起read操作后，就立刻返回做其他的事情了。由kernel负责接收数据并将数据拷贝到用户进程中，拷贝完成后，kernel给用户进程发送一个system call，用户进程再处理IO。

![1311869885-55c466fac00ba_articlex](/Users/wangwangxiaoteng/work/code/github/blogs/1311869885-55c466fac00ba_articlex.png)

**`可以看出，从用户进程角度来看，异步IO的特点是在IO的两个阶段，接收数据及数据拷贝到用户进程都是没有被block住的。`**



## 总结

### blocking和non-blocking的区别

从IO操作的两个阶段来看，区别在于数据准备的阶段，blocking IO中用户进程也是block住的，而non-blocking IO中用户进程是没有block住的。

## synchronous与asynchronous的区别

在说明synchronous IO和asynchronous IO的区别之前，需要先给出两者的定义。POSIX的定义是这样子的：
\- A synchronous I/O operation causes the requesting process to be blocked until that I/O operation completes;
\- An asynchronous I/O operation does not cause the requesting process to be blocked;

两者的区别就在于synchronous IO做”IO operation”的时候会将process阻塞。按照这个定义，之前所述的blocking IO，non-blocking IO，IO multiplexing都属于synchronous IO。

有人会说，non-blocking IO并没有被block啊。这里有个非常“狡猾”的地方，**定义中所指的”IO operation”是指真实的IO操作**，就是例子中的recvfrom这个system call。non-blocking IO在执行recvfrom这个system call的时候，如果kernel的数据没有准备好，这时候不会block进程。但是，当kernel中数据准备好的时候，recvfrom会将数据从kernel拷贝到用户内存中，这个时候进程是被block了，在这段时间内，进程是被block的。

**`就是说，synchronous与asynchronous的区别在于从kernel拷贝数据到用户进程的时候是不是block的`**

**各个IO Model的比较如图所示：**

![2109320510-55c4670795194_articlex](/Users/wangwangxiaoteng/work/code/github/blogs/2109320510-55c4670795194_articlex.png)

通过上面的图片，可以发现non-blocking IO和asynchronous IO的区别还是很明显的。在non-blocking IO中，虽然进程大部分时间都不会被block，但是它仍然要求进程去主动的check，并且当数据准备完成以后，也需要进程主动的再次调用recvfrom来将数据拷贝到用户内存。而asynchronous IO则完全不同。它就像是用户进程将整个IO操作交给了他人（kernel）完成，然后他人做完后发信号通知。在此期间，用户进程不需要去检查IO操作的状态，也不需要主动的去拷贝数据。

# reactor线程

为了处理TCP的连接，出现了两种竞争关系的体系模型，分别是基于多线程的模型和事件驱动模型。

### 基于线程的模型（Thread-Based Architecture）

传统的blocking IO，一个线程只能处理一个socket，为了处理多个socket，必须使用多线程去处理。由于线程的创建、销毁都极耗费资源，以及要控制正在执行的线程数量，避免CPU频繁切换导致上下文切换的开销，我们开始使用线程池技术。

### 事件驱动模型（Event-Driven Architecture）

在事件驱动模型中，单个线程通过select、poll、epoll等处理所负责的多个socket