# Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (客户端)
@(Netty)[Java, Netty, Netty 源码分析]

[TOC]


这一章是 Netty 源码分析系列的第一章, 我打算在这一章中, 展示一下 Netty 的客户端和服务端的初始化和启动的流程, 给读者一个对 Netty 源码有一个大致的框架上的认识, 而不会深入每个功能模块.
本章会从 Bootstrap/ServerBootstrap 类 入手, 分析 Netty 程序的初始化和启动的流程.
## Bootstrap
Bootstrap 是 Netty 提供的一个便利的工厂类, 我们可以通过它来完成 Netty 的客户端或服务器端的 Netty 初始化.
下面我以 Netty 源码例子中的 Echo 服务器作为例子, 从客户端和服务器端分别分析一下Netty 的程序是如何启动的.
## 客户端部分
### 连接源码
首先, 让我们从客户端方面的代码开始
下面是源码**example/src/main/java/io/netty/example/echo/EchoClient.java** 的客户端部分的启动代码:
```
EventLoopGroup group = new NioEventLoopGroup();
try {
    Bootstrap b = new Bootstrap();
    b.group(group)
     .channel(NioSocketChannel.class)
     .option(ChannelOption.TCP_NODELAY, true)
     .handler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) throws Exception {
             ChannelPipeline p = ch.pipeline();
             p.addLast(new EchoClientHandler());
         }
     });

    // Start the client.
    ChannelFuture f = b.connect(HOST, PORT).sync();

    // Wait until the connection is closed.
    f.channel().closeFuture().sync();
} finally {
    // Shut down the event loop to terminate all threads.
    group.shutdownGracefully();
}
```
从上面的客户端代码虽然简单, 但是却展示了 Netty 客户端初始化时所需的所有内容:
 1. EventLoopGroup: 不论是服务器端还是客户端, 都必须指定 EventLoopGroup. 在这个例子中, 指定了 NioEventLoopGroup, 表示一个 NIO 的EventLoopGroup.
 2. ChannelType: 指定 Channel 的类型. 因为是客户端, 因此使用了 NioSocketChannel.
 3. Handler: 设置数据的处理器.

下面我们深入代码, 看一下客户端通过 Bootstrap 启动后, 都做了哪些工作.

### NioSocketChannel 的初始化过程
在 Netty 中, Channel 是一个 Socket 的抽象, 它为用户提供了关于 Socket 状态(是否是连接还是断开) 以及对 Socket 的读写等操作. 每当 Netty 建立了一个连接后, 都会有一个对应的 Channel 实例.
NioSocketChannel 的类层次结构如下:
![Alt text](./NioSocketChannel 类层次结构.png)


这一小节我们着重分析一下 Channel 的初始化过程.
#### ChannelFactory 和 Channel 类型的确定
除了 TCP 协议以外, Netty 还支持很多其他的连接协议, 并且每种协议还有 NIO(异步 IO) 和 OIO(Old-IO, 即传统的阻塞 IO) 版本的区别. 不同协议不同的阻塞类型的连接都有不同的 Channel 类型与之对应下面是一些常用的 Channel 类型:
 - NioSocketChannel, 代表异步的客户端 TCP Socket 连接.
 - NioServerSocketChannel, 异步的服务器端 TCP Socket 连接.
 - NioDatagramChannel, 异步的 UDP 连接
 - NioSctpChannel, 异步的客户端 Sctp 连接.
 - NioSctpServerChannel, 异步的 Sctp 服务器端连接.
 - OioSocketChannel, 同步的客户端 TCP Socket 连接.
 - OioServerSocketChannel, 同步的服务器端 TCP Socket 连接.
 - OioDatagramChannel, 同步的 UDP 连接
 - OioSctpChannel, 同步的 Sctp 服务器端连接.
 - OioSctpServerChannel, 同步的客户端 TCP Socket 连接.

那么我们是如何设置所需要的 Channel 的类型的呢? 答案是 channel() 方法的调用.
回想一下我们在客户端连接代码的初始化 Bootstrap 中, 会调用 channel() 方法, 传入 NioSocketChannel.class, 这个方法其实就是初始化了一个 BootstrapChannelFactory:
```
public B channel(Class<? extends C> channelClass) {
    if (channelClass == null) {
        throw new NullPointerException("channelClass");
    }
    return channelFactory(new BootstrapChannelFactory<C>(channelClass));
}
```
而 BootstrapChannelFactory 实现了 ChannelFactory 接口, 它提供了唯一的方法, 即 **newChannel**. ChannelFactory, 顾名思义, 就是产生 Channel 的工厂类.
进入到 BootstrapChannelFactory.newChannel 中, 我们看到其实现代码如下:
```
@Override
public T newChannel() {
	// 删除 try 块
    return clazz.newInstance();
}
```
根据上面代码的提示, 我们就可以确定:
 - Bootstrap 中的 ChannelFactory 的实现是 BootstrapChannelFactory
 - 生成的 Channel 的具体类型是 NioSocketChannel. 
Channel 的实例化过程, 其实就是调用的 ChannelFactory#newChannel 方法, 而实例化的 Channel 的具体的类型又是和在初始化 Bootstrap 时传入的 channel() 方法的参数相关. 因此对于我们这个例子中的客户端的 Bootstrap 而言, 生成的的 Channel 实例就是 NioSocketChannel.

#### Channel 实例化
前面我们已经知道了如何确定一个 Channel 的类型, 并且了解到 Channel 是通过工厂方法 ChannelFactory.newChannel() 来实例化的, 那么 ChannelFactory.newChannel() 方法在哪里调用呢?
继续跟踪, 我们发现其调用链是:
```
Bootstrap.connect -> Bootstrap.doConnect -> AbstractBootstrap.initAndRegister
```
在 AbstractBootstrap.initAndRegister 中就调用了 **channelFactory().newChannel()** 来获取一个新的 NioSocketChannel 实例, 其源码如下:
```
final ChannelFuture initAndRegister() {
	// 去掉非关键代码
    final Channel channel = channelFactory().newChannel();
    init(channel);
    ChannelFuture regFuture = group().register(channel);
}
```
在 **newChannel** 中, 通过类对象的 newInstance 来获取一个新 Channel 实例, 因而会调用NioSocketChannel 的默认构造器.
NioSocketChannel 默认构造器代码如下:
```
public NioSocketChannel() {
    this(newSocket(DEFAULT_SELECTOR_PROVIDER));
}
```
`这里的代码比较关键`, 我们看到, 在这个构造器中, 会调用 **newSocket** 来打开一个新的 Java NIO SocketChannel:
```
private static SocketChannel newSocket(SelectorProvider provider) {
    ...
    return provider.openSocketChannel();
}
```
接着会调用父类, 即 AbstractNioByteChannel 的构造器:
```
AbstractNioByteChannel(Channel parent, SelectableChannel ch)
```
并传入参数 parent 为 null, ch 为刚才使用 newSocket 创建的  Java NIO SocketChannel, 因此生成的 NioSocketChannel 的 parent channel 是空的.
```
protected AbstractNioByteChannel(Channel parent, SelectableChannel ch) {
    super(parent, ch, SelectionKey.OP_READ);
}
```
接着会继续调用父类 AbstractNioChannel 的构造器, 并传入了参数 **readInterestOp = SelectionKey.OP_READ**:
```
protected AbstractNioChannel(Channel parent, SelectableChannel ch, int readInterestOp) {
    super(parent);
    this.ch = ch;
    this.readInterestOp = readInterestOp;
    // 省略 try 块
    // 配置 Java NIO SocketChannel 为非阻塞的.
    ch.configureBlocking(false);
}
```
然后继续调用父类 AbstractChannel 的构造器:
```
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```

到这里, 一个完整的 NioSocketChannel 就初始化完成了, 我们可以稍微总结一下构造一个 NioSocketChannel 所需要做的工作:
 - 调用 NioSocketChannel.newSocket(DEFAULT_SELECTOR_PROVIDER) 打开一个新的 Java NIO SocketChannel
 - AbstractChannel(Channel parent) 中初始化 AbstractChannel 的属性: 
  - parent 属性置为 null
  - unsafe 通过newUnsafe() 实例化一个 unsafe 对象, 它的类型是 AbstractNioByteChannel.NioByteUnsafe 内部类
  - pipeline 是 new DefaultChannelPipeline(this) 新创建的实例. `这里体现了:Each channel has its own pipeline and it is created automatically when a new channel is created.`
 - AbstractNioChannel 中的属性:
  - SelectableChannel ch 被设置为 Java SocketChannel, 即 NioSocketChannel#newSocket 返回的 Java NIO SocketChannel.
  - readInterestOp 被设置为 SelectionKey.OP_READ
  - SelectableChannel ch 被配置为非阻塞的 **ch.configureBlocking(false)**
 - NioSocketChannel 中的属性:
  - SocketChannelConfig config = new NioSocketChannelConfig(this, socket.socket())

### 关于 unsafe 字段的初始化
我们简单地提到了, 在实例化 NioSocketChannel 的过程中, 会在父类 AbstractChannel 的构造器中, 调用 newUnsafe() 来获取一个 unsafe 实例. 那么 unsafe 是怎么初始化的呢? 它的作用是什么?
其实 unsafe 特别关键, 它封装了对 Java 底层 Socket 的操作, 因此实际上是沟通 Netty 上层和 Java 底层的重要的桥梁.

那么我们就来看一下 Unsafe 接口所提供的方法吧:
```
interface Unsafe {
    SocketAddress localAddress();
    SocketAddress remoteAddress();
    void register(EventLoop eventLoop, ChannelPromise promise);
    void bind(SocketAddress localAddress, ChannelPromise promise);
    void connect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise);
    void disconnect(ChannelPromise promise);
    void close(ChannelPromise promise);
    void closeForcibly();
    void deregister(ChannelPromise promise);
    void beginRead();
    void write(Object msg, ChannelPromise promise);
    void flush();
    ChannelPromise voidPromise();
    ChannelOutboundBuffer outboundBuffer();
}
```
一看便知, 这些方法其实都会对应到相关的 Java 底层的 Socket 的操作.
回到 AbstractChannel 的构造方法中, 在这里调用了 newUnsafe() 获取一个新的 unsafe 对象, 而 newUnsafe 方法在 NioSocketChannel 中被重写了:
```
@Override
protected AbstractNioUnsafe newUnsafe() {
    return new NioSocketChannelUnsafe();
}
```
NioSocketChannel.newUnsafe 方法会返回一个 NioSocketChannelUnsafe 实例. 从这里我们就可以确定了, 在实例化的 NioSocketChannel 中的 unsafe 字段, 其实是一个 NioSocketChannelUnsafe 的实例.
### 关于 pipeline 的初始化
上面我们分析了一个 Channel (在这个例子中是 NioSocketChannel) 的大体初始化过程, 但是我们漏掉了一个关键的部分, 即 ChannelPipeline 的初始化. 
根据 `Each channel has its own pipeline and it is created automatically when a new channel is created.`, 我们知道, 在实例化一个 Channel 时, 必然伴随着实例化一个 ChannelPipeline. 而我们确实在 AbstractChannel 的构造器看到了 pipeline 字段被初始化为 DefaultChannelPipeline 的实例. 那么我们就来看一下, DefaultChannelPipeline 构造器做了哪些工作吧:
```
public DefaultChannelPipeline(AbstractChannel channel) {
    if (channel == null) {
        throw new NullPointerException("channel");
    }
    this.channel = channel;

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```
我们调用 DefaultChannelPipeline 的构造器, 传入了一个 channel, 而这个 channel 其实就是我们实例化的 NioSocketChannel, DefaultChannelPipeline 会将这个 NioSocketChannel 对象保存在channel 字段中. DefaultChannelPipeline 中, 还有两个特殊的字段, 即 head 和 tail, 而这两个字段是一个双向链表的头和尾. 其实在 DefaultChannelPipeline 中, 维护了一个以 AbstractChannelHandlerContext 为节点的双向链表, 这个链表是 Netty 实现 Pipeline 机制的关键. 关于 DefaultChannelPipeline 中的双向链表以及它所起的作用, 我在这里暂时不表, 在 **Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline** 中会有详细的分析.

HeadContext 的继承层次结构如下所示:
![Alt text](./HeadContext.png)
TailContext 的继承层次结构如下所示:
![Alt text](./TailContext.png)

我们可以看到, 链表中 head 是一个 **ChannelOutboundHandler**, 而 tail 则是一个 **ChannelInboundHandler**.
接着看一下 HeadContext 的构造器:
```
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
}
```
它调用了父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = false, outbound = true.
TailContext 的构造器与 HeadContext 的相反, 它调用了父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = true, outbound = false.
即 header 是一个 outboundHandler, 而 tail 是一个inboundHandler, 关于这一点, 大家要特别注意, 因为在分析到 Netty Pipeline 时, 我们会反复用到 inbound 和 outbound 这两个属性.

### 关于 EventLoop 初始化
回到最开始的 EchoClient.java 代码中, 我们在一开始就实例化了一个 NioEventLoopGroup 对象, 因此我们就从它的构造器中追踪一下 EventLoop 的初始化过程.
首先来看一下 NioEventLoopGroup 的类继承层次:
![Alt text](./NioEventLoopGroup 类层次结构.png)

NioEventLoop 有几个重载的构造器, 不过内容都没有什么区别, 最终都是调用的父类MultithreadEventLoopGroup构造器:
```
protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    super(nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}
```
其中有一点有意思的地方是, 如果我们传入的线程数 nThreads 是0, 那么 Netty 会为我们设置默认的线程数 DEFAULT_EVENT_LOOP_THREADS, 而这个默认的线程数是怎么确定的呢?
其实很简单, 在静态代码块中, 会首先确定 DEFAULT_EVENT_LOOP_THREADS 的值:
```
static {
    DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
            "io.netty.eventLoopThreads", Runtime.getRuntime().availableProcessors() * 2));
}
```
Netty 会首先从系统属性中获取 "io.netty.eventLoopThreads" 的值, 如果我们没有设置它的话, 那么就返回默认值: 处理器核心数 * 2.

回到MultithreadEventLoopGroup构造器中, 这个构造器会继续调用父类 MultithreadEventExecutorGroup 的构造器:
```
protected MultithreadEventExecutorGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    // 去掉了参数检查, 异常处理 等代码.
    children = new SingleThreadEventExecutor[nThreads];
    if (isPowerOfTwo(children.length)) {
        chooser = new PowerOfTwoEventExecutorChooser();
    } else {
        chooser = new GenericEventExecutorChooser();
    }

    for (int i = 0; i < nThreads; i ++) {
        children[i] = newChild(threadFactory, args);
    }
}
```
根据代码, 我们就很清楚 MultithreadEventExecutorGroup 中的处理逻辑了:
 - 创建一个大小为 nThreads 的 SingleThreadEventExecutor 数组
 - 根据 nThreads 的大小, 创建不同的 Chooser, 即如果 nThreads 是 2 的幂, 则使用 PowerOfTwoEventExecutorChooser, 反之使用 GenericEventExecutorChooser. 不论使用哪个 Chooser, 它们的功能都是一样的, 即从 children 数组中选出一个合适的 EventExecutor 实例.
 - 调用 newChhild 方法初始化 children 数组.

根据上面的代码, 我们知道, MultithreadEventExecutorGroup 内部维护了一个 EventExecutor 数组, Netty 的 EventLoopGroup 的实现机制其实就建立在 MultithreadEventExecutorGroup 之上. 每当 Netty 需要一个 EventLoop 时, 会调用 next() 方法获取一个可用的 EventLoop.
上面代码的最后一部分是 newChild 方法, 这个是一个抽象方法, 它的任务是实例化 EventLoop 对象. 我们跟踪一下它的代码, 可以发现, 这个方法在 NioEventLoopGroup 类中实现了, 其内容很简单:
```
@Override
protected EventExecutor newChild(
        ThreadFactory threadFactory, Object... args) throws Exception {
    return new NioEventLoop(this, threadFactory, (SelectorProvider) args[0]);
}
```
其实就是实例化一个 NioEventLoop 对象, 然后返回它.

最后总结一下整个 EventLoopGroup 的初始化过程吧:
 - EventLoopGroup(其实是MultithreadEventExecutorGroup) 内部维护一个类型为 EventExecutor children 数组, 其大小是 nThreads, 这样就构成了一个线程池
 - 如果我们在实例化 NioEventLoopGroup 时, 如果指定线程池大小, 则 nThreads 就是指定的值, 反之是处理器核心数 * 2
 - MultithreadEventExecutorGroup 中会调用 newChild 抽象方法来初始化 children 数组
 - 抽象方法 newChild 是在 NioEventLoopGroup 中实现的, 它返回一个 NioEventLoop 实例.
 - NioEventLoop 属性:
  - SelectorProvider provider 属性: NioEventLoopGroup 构造器中通过 SelectorProvider.provider() 获取一个 SelectorProvider
  - Selector selector 属性: NioEventLoop 构造器中通过调用通过 selector = provider.openSelector() 获取一个 selector 对象.

### channel 的注册过程
在前面的分析中, 我们提到, channel 会在 Bootstrap.initAndRegister 中进行初始化, 但是这个方法还会将初始化好的 Channel 注册到 EventGroup 中. 接下来我们就来分析一下 Channel 注册的过程.
回顾一下 AbstractBootstrap.initAndRegister 方法:
```
final ChannelFuture initAndRegister() {
	// 去掉非关键代码
    final Channel channel = channelFactory().newChannel();
    init(channel);
    ChannelFuture regFuture = group().register(channel);
}
```
当Channel 初始化后, 会紧接着调用 group().register() 方法来注册 Channel, 我们继续跟踪的话, 会发现其调用链如下:
AbstractBootstrap.initAndRegister -> MultithreadEventLoopGroup.register -> SingleThreadEventLoop.register -> AbstractUnsafe.register
通过跟踪调用链, 最终我们发现是调用到了 unsafe 的 register 方法, 那么接下来我们就仔细看一下 AbstractUnsafe.register 方法中到底做了什么:
```
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
	// 省略条件判断和错误处理
    AbstractChannel.this.eventLoop = eventLoop;
    register0(promise);
}
```
首先, 将 eventLoop 赋值给 Channel 的 eventLoop 属性, 而我们知道这个 eventLoop 对象其实是 MultithreadEventLoopGroup.next() 方法获取的, 根据我们前面 **关于 EventLoop 初始化** 小节中, 我们可以确定 next() 方法返回的 eventLoop 对象是 NioEventLoop 实例.
register 方法接着调用了 register0 方法:
```
private void register0(ChannelPromise promise) {
    boolean firstRegistration = neverRegistered;
    doRegister();
    neverRegistered = false;
    registered = true;
    safeSetSuccess(promise);
    pipeline.fireChannelRegistered();
    // Only fire a channelActive if the channel has never been registered. This prevents firing
    // multiple channel actives if the channel is deregistered and re-registered.
    if (firstRegistration && isActive()) {
        pipeline.fireChannelActive();
    }
}
```
register0 又调用了 AbstractNioChannel.doRegister:
```
@Override
protected void doRegister() throws Exception {
	// 省略错误处理
    selectionKey = javaChannel().register(eventLoop().selector, 0, this);
}
```
javaChannel() 这个方法在前面我们已经知道了, 它返回的是一个 Java NIO SocketChannel, 这里我们将这个 SocketChannel 注册到与 eventLoop 关联的 selector 上了.

我们总结一下 Channel 的注册过程:
 - 首先在 AbstractBootstrap.initAndRegister中, 通过 group().register(channel), 调用 MultithreadEventLoopGroup.register 方法
 - 在MultithreadEventLoopGroup.register 中, 通过 next() 获取一个可用的 SingleThreadEventLoop, 然后调用它的 register
 - 在 SingleThreadEventLoop.register 中, 通过 channel.unsafe().register(this, promise) 来获取 channel 的 unsafe() 底层操作对象, 然后调用它的 register.
 - 在 AbstractUnsafe.register 方法中, 调用 register0 方法注册 Channel
 - 在 AbstractUnsafe.register0 中, 调用 AbstractNioChannel.doRegister 方法
 - AbstractNioChannel.doRegister 方法通过 javaChannel().register(eventLoop().selector, 0, this) 将 Channel 对应的 Java NIO SockerChannel 注册到一个 eventLoop 的 Selector 中, 并且将当前 Channel 作为 attachment.

总的来说, Channel 注册过程所做的工作就是将 Channel 与对应的 EventLoop 关联, 因此这也体现了, 在 Netty 中, 每个 Channel 都会关联一个特定的 EventLoop, 并且这个 Channel 中的所有 IO 操作都是在这个 EventLoop 中执行的; 当关联好 Channel 和 EventLoop 后, 会继续调用底层的 Java NIO SocketChannel 的 register 方法, 将底层的 Java NIO SocketChannel 注册到指定的 selector 中. 通过这两步, 就完成了 Netty Channel 的注册过程.

### handler 的添加过程
Netty 的一个强大和灵活之处就是基于 Pipeline 的自定义 handler 机制. 基于此, 我们可以像添加插件一样自由组合各种各样的 handler 来完成业务逻辑. 例如我们需要处理 HTTP 数据, 那么就可以在 pipeline 前添加一个 Http 的编解码的 Handler, 然后接着添加我们自己的业务逻辑的 handler, 这样网络上的数据流就向通过一个管道一样, 从不同的 handler 中流过并进行编解码, 最终在到达我们自定义的 handler 中.
既然说到这里, 有些读者朋友肯定会好奇, 既然这个 pipeline 机制是这么的强大, 那么它是怎么实现的呢? 不过我这里不打算详细展开 Netty 的 ChannelPipeline 的实现机制(具体的细节会在后续的章节中展示), 我在这一小节中, 从简单的入手, 展示一下我们自定义的 handler 是如何以及何时添加到 ChannelPipeline 中的.
首先让我们看一下如下的代码片段:
```
...
.handler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) throws Exception {
         ChannelPipeline p = ch.pipeline();
         if (sslCtx != null) {
             p.addLast(sslCtx.newHandler(ch.alloc(), HOST, PORT));
         }
         //p.addLast(new LoggingHandler(LogLevel.INFO));
         p.addLast(new EchoClientHandler());
     }
 });
```
这个代码片段就是实现了 handler 的添加功能. 我们看到, Bootstrap.handler 方法接收一个 ChannelHandler, 而我们传递的是一个 派生于 ChannelInitializer 的匿名类, 它正好也实现了 ChannelHandler 接口. 我们来看一下, ChannelInitializer  类内到底有什么玄机:
```
@Sharable
public abstract class ChannelInitializer<C extends Channel> extends ChannelInboundHandlerAdapter {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(ChannelInitializer.class);
    protected abstract void initChannel(C ch) throws Exception;

    @Override
    @SuppressWarnings("unchecked")
    public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
        initChannel((C) ctx.channel());
        ctx.pipeline().remove(this);
        ctx.fireChannelRegistered();
    }
    ...
}
```
ChannelInitializer 是一个抽象类, 它有一个抽象的方法 initChannel, 我们正是实现了这个方法, 并在这个方法中添加的自定义的 handler 的. 那么 initChannel 是哪里被调用的呢? 答案是 ChannelInitializer.channelRegistered 方法中. 
我们来关注一下 channelRegistered 方法. 从上面的源码中, 我们可以看到, 在 channelRegistered 方法中, 会调用 initChannel 方法, 将自定义的 handler 添加到 ChannelPipeline 中, 然后调用 ctx.pipeline().remove(this) 将自己从 ChannelPipeline 中删除. 上面的分析过程, 可以用如下图片展示:
一开始, ChannelPipeline 中只有三个 handler, head, tail 和我们添加的 ChannelInitializer.
![Alt text](./1477130291691.png)
接着 initChannel 方法调用后, 添加了自定义的 handler:
![Alt text](./1477130295919.png)
最后将 ChannelInitializer 删除:
![Alt text](./1477130299722.png)

分析到这里, 我们已经简单了解了自定义的 handler 是如何添加到 ChannelPipeline 中的, 不过限于主题与篇幅的原因, 我没有在这里详细展开 ChannelPipeline 的底层机制, 我打算在下一篇 **Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline** 中对这个问题进行深入的探讨.

### 客户端连接分析
经过上面的各种分析后, 我们大致了解了 Netty 初始化时, 所做的工作, 那么接下来我们就直奔主题, 分析一下客户端是如何发起 TCP 连接的.

首先, 客户端通过调用 **Bootstrap** 的 **connect** 方法进行连接.
在 connect 中, 会进行一些参数检查后, 最终调用的是 **doConnect0** 方法, 其实现如下:
```
private static void doConnect0(
        final ChannelFuture regFuture, final Channel channel,
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

    // This method is invoked before channelRegistered() is triggered.  Give user handlers a chance to set up
    // the pipeline in its channelRegistered() implementation.
    channel.eventLoop().execute(new Runnable() {
        @Override
        public void run() {
            if (regFuture.isSuccess()) {
                if (localAddress == null) {
                    channel.connect(remoteAddress, promise);
                } else {
                    channel.connect(remoteAddress, localAddress, promise);
                }
                promise.addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
            } else {
                promise.setFailure(regFuture.cause());
            }
        }
    });
}
```
在 doConnect0 中, 会在 event loop 线程中调用 Channel 的 connect 方法, 而这个 Channel 的具体类型是什么呢? 我们在 Channel 初始化这一小节中已经分析过了, 这里 channel 的类型就是 **NioSocketChannel**.
进行跟踪到 channel.connect 中, 我们发现它调用的是 DefaultChannelPipeline#connect, 而, pipeline 的 connect 代码如下:
```
@Override
public ChannelFuture connect(SocketAddress remoteAddress) {
    return tail.connect(remoteAddress);
}
```
而 tail 字段, 我们已经分析过了, 是一个 TailContext 的实例, 而 TailContext 又是 AbstractChannelHandlerContext 的子类, 并且没有实现 connect 方法, 因此这里调用的其实是 AbstractChannelHandlerContext.connect, 我们看一下这个方法的实现:
```
@Override
public ChannelFuture connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

	// 删除的参数检查的代码
    final AbstractChannelHandlerContext next = findContextOutbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeConnect(remoteAddress, localAddress, promise);
    } else {
        safeExecute(executor, new OneTimeTask() {
            @Override
            public void run() {
                next.invokeConnect(remoteAddress, localAddress, promise);
            }
        }, promise, null);
    }

    return promise;
}
```
上面的代码中有一个关键的地方, 即 **final AbstractChannelHandlerContext next = findContextOutbound()**, 这里调用 **findContextOutbound** 方法, 从  DefaultChannelPipeline 内的双向链表的 tail 开始, 不断向前寻找第一个 outbound 为 true 的 AbstractChannelHandlerContext, 然后调用它的 invokeConnect 方法, 其代码如下:
```
private void invokeConnect(SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) {
    // 忽略 try 块
    ((ChannelOutboundHandler) handler()).connect(this, remoteAddress, localAddress, promise);
}
```
还记得我们在 "关于 pipeline 的初始化" 这一小节分析的的内容吗? 我们提到, 在 DefaultChannelPipeline 的构造器中, 会实例化两个对象: head 和 tail, 并形成了双向链表的头和尾. head 是 HeadContext 的实例, 它实现了 ChannelOutboundHandler 接口, 并且它的 outbound 字段为 true. 因此在 findContextOutbound 中, 找到的 AbstractChannelHandlerContext 对象其实就是 head. 进而在 invokeConnect 方法中, 我们向上转换为 ChannelOutboundHandler 就是没问题的了.
而又因为 HeadContext 重写了 connect 方法, 因此实际上调用的是 HeadContext.connect. 我们接着跟踪到 HeadContext.connect, 其代码如下:
```
@Override
public void connect(
        ChannelHandlerContext ctx,
        SocketAddress remoteAddress, SocketAddress localAddress,
        ChannelPromise promise) throws Exception {
    unsafe.connect(remoteAddress, localAddress, promise);
}
```
这个 connect 方法很简单, 仅仅调用了 unsafe 的 connect 方法. 而 unsafe 又是什么呢?
回顾一下 HeadContext 的构造器, 我们发现 unsafe 是 pipeline.channel().unsafe() 返回的, 而 Channel 的 unsafe 字段, 在这个例子中, 我们已经知道了, 其实是 AbstractNioByteChannel.NioByteUnsafe 内部类. 兜兜转转了一大圈, 我们找到了创建 Socket 连接的关键代码.
进行跟踪 NioByteUnsafe -> AbstractNioUnsafe.connect:
```
@Override
public final void connect(
        final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    boolean wasActive = isActive();
    if (doConnect(remoteAddress, localAddress)) {
        fulfillConnectPromise(promise, wasActive);
    } else {
        ...
    }
}
```
AbstractNioUnsafe.connect 的实现如上代码所示, 在这个 connect 方法中, 调用了 doConnect 方法, `注意, 这个方法并不是 AbstractNioUnsafe 的方法, 而是 AbstractNioChannel 的抽象方法.` doConnect 方法是在 NioSocketChannel 中实现的, 因此进入到 **NioSocketChannel.doConnect** 中:
```
@Override
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        javaChannel().socket().bind(localAddress);
    }

    boolean success = false;
    try {
        boolean connected = javaChannel().connect(remoteAddress);
        if (!connected) {
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}
```
我们终于看到的最关键的部分了, 庆祝一下!
上面的代码不用多说, 首先是获取 Java NIO SocketChannel, 即我们已经分析过的, 从 NioSocketChannel.newSocket 返回的 SocketChannel 对象; 然后是调用 SocketChannel.connect 方法完成 Java NIO 层面上的 Socket 的连接.
最后, 上面的代码流程可以用如下时序图直观地展示:
![Alt text](./Netty 客户端的连接时序图.png)
