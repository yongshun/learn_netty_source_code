# Netty 源码分析之 二 贯穿Netty 的大动脉 ── ChannelPipeline (一)

@(Netty)[Netty 源码分析]

[TOC]


----------

## 前言
这篇是 **Netty 源码分析** 的第二篇, 在这篇文章中, 我会为读者详细地分析 Netty 中的 ChannelPipeline 机制.

## Channel 与 ChannelPipeline
相信大家都知道了, 在  Netty 中每个 Channel 都有且仅有一个 ChannelPipeline 与之对应, 它们的组成关系如下:
![Alt text](./Channel 组成.png)


通过上图我们可以看到, 一个 Channel 包含了一个 ChannelPipeline, 而 ChannelPipeline 中又维护了一个由 ChannelHandlerContext 组成的双向链表. 这个链表的头是 HeadContext, 链表的尾是 TailContext, 并且每个 ChannelHandlerContext 中又关联着一个 ChannelHandler.
上面的图示给了我们一个对 ChannelPipeline 的直观认识, 但是实际上 Netty 实现的 Channel 是否真的是这样的呢? 我们继续用源码说话.

在第一章 **Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头** 中, 我们已经知道了一个 Channel 的初始化的基本过程, 下面我们再回顾一下.
下面的代码是 AbstractChannel 构造器:
```
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```
AbstractChannel 有一个 pipeline 字段, 在构造器中会初始化它为 DefaultChannelPipeline的实例. 这里的代码就印证了一点: `每个 Channel 都有一个 ChannelPipeline`.
接着我们跟踪一下 DefaultChannelPipeline 的初始化过程.
首先进入到 DefaultChannelPipeline 构造器中:
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
在 DefaultChannelPipeline 构造器中, 首先将与之关联的 Channel 保存到字段 channel 中, 然后实例化两个 ChannelHandlerContext, 一个是 HeadContext 实例 head, 另一个是 TailContext 实例 tail. 接着将 head 和 tail 互相指向, 构成一个双向链表.
`特别注意到, 我们在开始的示意图中, head 和 tail 并没有包含 ChannelHandler`, 这是因为 HeadContext 和 TailContext 继承于 AbstractChannelHandlerContext 的同时也实现了 ChannelHandler 接口了, 因此它们有 Context 和 Handler 的双重属性.
## ChannelPipeline 的初始化再探
在第一章的时候, 我们已经对 ChannelPipeline 的初始化有了一个大致的了解, 不过当时重点毕竟不在 ChannelPipeline 这里, 因此没有深入地分析它的初始化过程. 那么下面我们就来看一下具体的 ChannelPipeline 的初始化都做了哪些工作吧.

### ChannelPipeline 实例化过程
我们再来回顾一下, 在实例化一个 Channel 时, 会伴随着一个 ChannelPipeline 的实例化, 并且此 Channel 会与这个 ChannelPipeline 相互关联, 这一点可以通过NioSocketChannel 的父类 AbstractChannel 的构造器予以佐证:
```
protected AbstractChannel(Channel parent) {
    this.parent = parent;
    unsafe = newUnsafe();
    pipeline = new DefaultChannelPipeline(this);
}
```
当实例化一个 Channel(这里以 EchoClient 为例, 那么 Channel 就是 NioSocketChannel), 其 pipeline 字段就是我们新创建的 DefaultChannelPipeline 对象, 那么我们就来看一下 DefaultChannelPipeline 的构造方法吧:
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
可以看到, 在 DefaultChannelPipeline 的构造方法中, 将传入的 channel 赋值给字段 this.channel, 接着又实例化了两个特殊的字段: **tail** 与 **head**. 这两个字段是一个双向链表的头和尾. 其实在 DefaultChannelPipeline 中, 维护了一个以 AbstractChannelHandlerContext 为节点的双向链表, 这个链表是 Netty 实现 Pipeline 机制的关键.
再回顾一下 head 和 tail 的类层次结构:
![@HeadContext 类层次结构](./HeadContext 类层次结构.png)
![@TailContext 类层次结构](./TailContext 类层次结构.png)

从类层次结构图中可以很清楚地看到, head 实现了 **ChannelInboundHandler**, 而 tail 实现了 **ChannelOutboundHandler** 接口, 并且它们都实现了 **ChannelHandlerContext**  接口, 因此可以说 **head 和 tail 即是一个 ChannelHandler, 又是一个 ChannelHandlerContext.**

接着看一下 HeadContext 的构造器:
```
HeadContext(DefaultChannelPipeline pipeline) {
    super(pipeline, null, HEAD_NAME, false, true);
    unsafe = pipeline.channel().unsafe();
}
```
它调用了父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = false, outbound = true.
TailContext 的构造器与 HeadContext 的相反, 它调用了父类 AbstractChannelHandlerContext 的构造器, 并传入参数 inbound = true, outbound = false.
即 header 是一个 outboundHandler, 而 tail 是一个inboundHandler, 关于这一点, 大家要特别注意, 因为在后面的分析中, 我们会反复用到 inbound 和 outbound 这两个属性.

### 链表的节点 ChannelHandlerContext

### ChannelInitializer 的添加
上面一小节中, 我们已经分析了 Channel 的组成, 其中我们了解到, 最开始的时候 ChannelPipeline 中含有两个 ChannelHandlerContext(同时也是 ChannelHandler), 但是这个 Pipeline并不能实现什么特殊的功能, 因为我们还没有给它添加自定义的 ChannelHandler.
通常来说, 我们在初始化 Bootstrap, 会添加我们自定义的 ChannelHandler, 就以我们熟悉的 EchoClient 来举例吧:
```
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
```
上面代码的初始化过程, 相信大家都不陌生. 在调用 handler 时, 传入了 ChannelInitializer 对象, 它提供了一个 initChannel 方法供我们初始化 ChannelHandler. 那么这个初始化过程是怎样的呢? 下面我们就来揭开它的神秘面纱.

ChannelInitializer 实现了 ChannelHandler, 那么它是在什么时候添加到 ChannelPipeline 中的呢? 进行了一番搜索后, 我们发现它是在 Bootstrap.init 方法中添加到 ChannelPipeline 中的.
其代码如下:
```
@Override
@SuppressWarnings("unchecked")
void init(Channel channel) throws Exception {
    ChannelPipeline p = channel.pipeline();
    p.addLast(handler());
    ...
}
```
上面的代码将 handler() 返回的 ChannelHandler 添加到 Pipeline 中, 而 handler() 返回的是handler 其实就是我们在初始化 Bootstrap 调用 handler 设置的 ChannelInitializer 实例, 因此这里就是将 ChannelInitializer 插入到了 Pipeline 的末端.
此时 Pipeline 的结构如下图所示:
![Alt text](./ChannelPipeline 初始化过程.png)


有朋友可能就有疑惑了, 我明明插入的是一个 ChannelInitializer 实例, 为什么在 ChannelPipeline 中的双向链表中的元素却是一个 ChannelHandlerContext? 为了解答这个问题, 我们继续在代码中寻找答案吧.
我们刚才提到, 在 Bootstrap.init 中会调用 p.addLast() 方法, 将 ChannelInitializer 插入到链表末端:
```
@Override
public ChannelPipeline addLast(EventExecutorGroup group, final String name, ChannelHandler handler) {
    synchronized (this) {
        checkDuplicateName(name); // 检查此 handler 是否有重复的名字

        AbstractChannelHandlerContext newCtx = new DefaultChannelHandlerContext(this, group, name, handler);
        addLast0(name, newCtx);
    }

    return this;
}
```
addLast 有很多重载的方法, 我们关注这个比较重要的方法就可以了.
上面的 addLast 方法中, 首先检查这个 ChannelHandler 的名字是否是重复的, 如果不重复的话, 则为这个 Handler 创建一个对应的 DefaultChannelHandlerContext 实例, 并与之关联起来(Context 中有一个 handler 属性保存着对应的 Handler 实例). 判断此 Handler 是否重名的方法很简单: Netty 中有一个 **name2ctx** Map 字段, key 是 handler 的名字, 而 value 则是 handler 本身. 因此通过如下代码就可以判断一个 handler 是否重名了:
```
private void checkDuplicateName(String name) {
    if (name2ctx.containsKey(name)) {
        throw new IllegalArgumentException("Duplicate handler name: " + name);
    }
}
```
为了添加一个 handler 到 pipeline 中, 必须把此 handler 包装成 ChannelHandlerContext. 因此在上面的代码中我们可以看到新实例化了一个 newCtx 对象, 并将 handler 作为参数传递到构造方法中. 那么我们来看一下实例化的 DefaultChannelHandlerContext 到底有什么玄机吧.
首先看它的构造器:
```
DefaultChannelHandlerContext(
        DefaultChannelPipeline pipeline, EventExecutorGroup group, String name, ChannelHandler handler) {
    super(pipeline, group, name, isInbound(handler), isOutbound(handler));
    if (handler == null) {
        throw new NullPointerException("handler");
    }
    this.handler = handler;
}
```
DefaultChannelHandlerContext 的构造器中, 调用了两个很有意思的方法: **isInbound** 与 **isOutbound**, 这两个方法是做什么的呢?
```
private static boolean isInbound(ChannelHandler handler) {
    return handler instanceof ChannelInboundHandler;
}

private static boolean isOutbound(ChannelHandler handler) {
    return handler instanceof ChannelOutboundHandler;
}
```
从源码中可以看到, 当一个 handler 实现了 ChannelInboundHandler 接口, 则 isInbound 返回真; 相似地, 当一个 handler 实现了 ChannelOutboundHandler 接口, 则 isOutbound 就返回真.
而这两个 boolean 变量会传递到父类 AbstractChannelHandlerContext 中, 并初始化父类的两个字段: **inbound** 与 **outbound**.
那么这里的 ChannelInitializer 所对应的 DefaultChannelHandlerContext 的 inbound 与 inbound 字段分别是什么呢? 那就看一下 ChannelInitializer 到底实现了哪个接口不就行了? 如下是 ChannelInitializer 的类层次结构图:
![Alt text](./ChannelInitializer 类层次结构图.png)
可以清楚地看到, ChannelInitializer 仅仅实现了 ChannelInboundHandler 接口, 因此这里实例化的 DefaultChannelHandlerContext 的 inbound = true, outbound = false.
不就是 inbound 和 outbound 两个字段嘛, 为什么需要这么大费周章地分析一番? 其实这两个字段关系到 pipeline 的事件的流向与分类, 因此是十分关键的, 不过我在这里先卖个关子, 后面我们再来详细分析这两个字段所起的作用. 在这里, 读者只需要记住, **ChannelInitializer 所对应的 DefaultChannelHandlerContext 的 inbound = true, outbound = false** 即可.

当创建好 Context 后, 就将这个 Context 插入到 Pipeline 的双向链表中:
```
private void addLast0(final String name, AbstractChannelHandlerContext newCtx) {
    checkMultiplicity(newCtx);

    AbstractChannelHandlerContext prev = tail.prev;
    newCtx.prev = prev;
    newCtx.next = tail;
    prev.next = newCtx;
    tail.prev = newCtx;

    name2ctx.put(name, newCtx);

    callHandlerAdded(newCtx);
}
```
显然, 这个代码就是典型的双向链表的插入操作了. 当调用了 addLast 方法后, Netty 就会将此 handler 添加到双向链表中 tail 元素之前的位置.

## 自定义 ChannelHandler 的添加过程
在上一小节中, 我们已经分析了一个 ChannelInitializer 如何插入到 Pipeline 中的, 接下来就来探讨一下 ChannelInitializer 在哪里被调用, ChannelInitializer 的作用, 以及我们自定义的 ChannelHandler 是如何插入到 Pipeline 中的.

在 **Netty 源码分析之 一 揭开 Bootstrap 神秘的红盖头 (客户端)** 一章的 **channel 的注册过程** 小节中, 我们已经分析过 Channel 的注册过程了, 这里我们再简单地复习一下:
 - 首先在 AbstractBootstrap.initAndRegister中, 通过 group().register(channel), 调用 MultithreadEventLoopGroup.register 方法
 - 在MultithreadEventLoopGroup.register 中, 通过 next() 获取一个可用的 SingleThreadEventLoop, 然后调用它的 register
 - 在 SingleThreadEventLoop.register 中, 通过 channel.unsafe().register(this, promise) 来获取 channel 的 unsafe() 底层操作对象, 然后调用它的 register.
 - 在 AbstractUnsafe.register 方法中, 调用 register0 方法注册 Channel
 - 在 AbstractUnsafe.register0 中, 调用 AbstractNioChannel#doRegister 方法
 - AbstractNioChannel.doRegister 方法通过 javaChannel().register(eventLoop().selector, 0, this) 将 Channel 对应的 Java NIO SockerChannel 注册到一个 eventLoop 的 Selector 中, 并且将当前 Channel 作为 attachment.

而我们自定义 ChannelHandler 的添加过程, 发生在 AbstractUnsafe.register0 中, 在这个方法中调用了 pipeline.fireChannelRegistered() 方法, 其实现如下:
```
@Override
public ChannelPipeline fireChannelRegistered() {
    head.fireChannelRegistered();
    return this;
}
```
上面的代码很简单, 就是调用了 head.fireChannelRegistered() 方法而已.
>`关于上面代码的 head.fireXXX 的调用形式, 是 Netty 中 Pipeline 传递事件的常用方式, 我们以后会经常看到.`

还记得 head 的 类层次结构图不, head 是一个 AbstractChannelHandlerContext 实例, 并且它没有重写 fireChannelRegistered 方法, 因此 head.fireChannelRegistered 其实是调用的 AbstractChannelHandlerContext.fireChannelRegistered:
```
@Override
public ChannelHandlerContext fireChannelRegistered() {
    final AbstractChannelHandlerContext next = findContextInbound();
    EventExecutor executor = next.executor();
    if (executor.inEventLoop()) {
        next.invokeChannelRegistered();
    } else {
        executor.execute(new OneTimeTask() {
            @Override
            public void run() {
                next.invokeChannelRegistered();
            }
        });
    }
    return this;
}
```
这个方法的第一句是调用 findContextInbound 获取一个 Context, 那么它返回的 Context 到底是什么呢? 我们跟进代码中看一下:
```
private AbstractChannelHandlerContext findContextInbound() {
    AbstractChannelHandlerContext ctx = this;
    do {
        ctx = ctx.next;
    } while (!ctx.inbound);
    return ctx;
}
```
很显然, 这个代码会从 head 开始遍历 Pipeline 的双向链表, 然后找到第一个属性 inbound 为 true 的 ChannelHandlerContext 实例. 想起来了没? 我们在前面分析 ChannelInitializer 时, 花了大量的笔墨来分析了 inbound 和 outbound 属性, 你看现在这里就用上了. 回想一下, ChannelInitializer 实现了 ChannelInboudHandler, 因此它所对应的 ChannelHandlerContext 的 inbound 属性就是 true, 因此这里返回就是 ChannelInitializer 实例所对应的 ChannelHandlerContext. 即:
![Alt text](./自定义 ChannelHandler 分析1.png)


当获取到 inbound 的 Context 后, 就调用它的 invokeChannelRegistered 方法:
```
private void invokeChannelRegistered() {
    try {
        ((ChannelInboundHandler) handler()).channelRegistered(this);
    } catch (Throwable t) {
        notifyHandlerException(t);
    }
}
```
 我们已经强调过了, 每个 ChannelHandler 都与一个 ChannelHandlerContext 关联, 我们可以通过 ChannelHandlerContext 获取到对应的 ChannelHandler. 因此很显然了, 这里 handler() 返回的, 其实就是一开始我们实例化的 ChannelInitializer 对象, 并接着调用了 ChannelInitializer.channelRegistered 方法. 看到这里, 读者朋友是否会觉得有点眼熟呢? ChannelInitializer.channelRegistered 这个方法我们在第一章的时候已经大量地接触了, 但是我们并没有深入地分析这个方法的调用过程, 那么在这里读者朋友应该对它的调用有了更加深入的了解了吧.
 
那么这个方法中又有什么玄机呢? 继续看代码:
```
@Override
@SuppressWarnings("unchecked")
public final void channelRegistered(ChannelHandlerContext ctx) throws Exception {
    initChannel((C) ctx.channel());
    ctx.pipeline().remove(this);
    ctx.fireChannelRegistered();
}
```
initChannel 这个方法我们很熟悉了吧, 它就是我们在初始化 Bootstrap 时, 调用 handler 方法传入的匿名内部类所实现的方法:
```
.handler(new ChannelInitializer<SocketChannel>() {
     @Override
     public void initChannel(SocketChannel ch) throws Exception {
         ChannelPipeline p = ch.pipeline();
         p.addLast(new EchoClientHandler());
     }
 });
```
因此当调用了这个方法后, 我们自定义的 ChannelHandler 就插入到 Pipeline 了, 此时的 Pipeline 如下图所示:
![Alt text](./自定义 ChannelHandler 分析2.png)


当添加了自定义的 ChannelHandler 后, 会删除 ChannelInitializer 这个 ChannelHandler, 即 "ctx.pipeline().remove(this)", 因此最后的 Pipeline 如下:
![Alt text](./自定义 ChannelHandler 分析3.png)

好了, 到了这里, 我们的 **自定义 ChannelHandler 的添加过程** 也分析的查不多了.
