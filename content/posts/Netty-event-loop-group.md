---
title: "Netty 源码解析 （一）：NioEventLoopGroup"
date: 2024-07-10
draft: true
metaAlignment: center
categories:
- Netty
tags:
- Netty 源码
- NioEventLoopGroup
---

<!--more-->

Netty 使用示例
```java
public final class EchoServer {

    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 100)
             .handler(new LoggingHandler(LogLevel.INFO))
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     if (sslCtx != null) {
                         p.addLast(sslCtx.newHandler(ch.alloc()));
                     }
                     p.addLast(new EchoServerHandler());
                 }
             });
            ChannelFuture f = b.bind(8007).sync();
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

NioEventLoopGroup 构造方法

```java
public NioEventLoopGroup() {
    this(0);
}

public NioEventLoopGroup(int nThreads) {
    this(nThreads, null);
}

public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory) {
    this(nThreads, threadFactory, SelectorProvider.provider());
}

public NioEventLoopGroup(
        int nThreads, ThreadFactory threadFactory, final SelectorProvider selectorProvider) {
    this(nThreads, threadFactory, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
}

public NioEventLoopGroup(int nThreads, ThreadFactory threadFactory,
                         final SelectorProvider selectorProvider, final SelectStrategyFactory selectStrategyFactory) {
    super(nThreads, threadFactory, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
}

/**
 * 如果没有设置程序启动参数（或者说没有指定key=”io.netty.eventLoopThreads”的属性值），那么默认情况下线程的个数为cpu的核数乘以2。
private static final int DEFAULT_EVENT_LOOP_THREADS;
    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt("io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));
}
 */
protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
    super(nThreads == 0? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
}


```

设置线程数 N 为 cpu 数量的 2 倍
设置线程工厂：DefaultThreadFactory
创建 N 个 NioEventLoop 线程
    provider = selectorProvider;
    selector = selectorTuple.selector;
    DefaultThreadFactory.newThread()，传入Runnable，重新 run 方法，调用 NioEventLoop 的 run 方法
    设置 taskQueue = new LinkedBlockingQueue<>(maxPendingTasks)
    