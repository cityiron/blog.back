---
title: Netty权威指南-客户端创建
date: 2019-09-25 22:17:33
tags: [netty客户端, netty, 学习, 原创]
categories: netty
---

[netty] 学习Netty-客户端创建

<!-- more -->

步骤1 用户线程创建 Bootstrap 实例，通过 API 设置创建客户端相关的参数，异步发起客户端连接
步骤2 创建处理客户端连接、 I/O 读写的 Reactor 线程组 NioEventLoopGroup 。可以通过构造函数指定 I/O 线程的个数，默认为 CPU 内核数的2倍

```java
    /**
     * The {@link EventLoopGroup} which is used to handle all the events for the to-be-created
     * {@link Channel}
     */
    public B group(EventLoopGroup group) {
        if (group == null) {
            throw new NullPointerException("group");
        }
        if (this.group != null) {
            throw new IllegalStateException("group set already");
        }
        this.group = group;
        return self();
    }
```

```java
    /**
     * Allow to specify a {@link ChannelOption} which is used for the {@link Channel} instances once they got
     * created. Use a value of {@code null} to remove a previous set {@link ChannelOption}.
     */
    public <T> B option(ChannelOption<T> option, T value) {
        if (option == null) {
            throw new NullPointerException("option");
        }
        if (value == null) {
            synchronized (options) {
                options.remove(option);
            }
        } else {
            synchronized (options) {
                options.put(option, value);
            }
        }
        return self();
    }
```

步骤3 通过 Bootstrap 的 ChannelFactory 和用户指定的 Channel 类型创建用于客户端连接的 NioSocketChannel ，它的功能类似于JDK NIO类库提供的 SocketChannel
步骤4 创建默认的 Channel Handler Piepline，用于调度和执行网络事件
步骤5 异步发起 TCP 连接，判断连接是否成功。如果成功，则直接将 NioSocketChannel 注册到多路复用器上，监听读操作位，用于数据报读取和消息发送；
如果没有立即连接成功，则注册连接监听位到多路复用器，等待连接结果
步骤6 注册对应到网络监听状态位到多路复用器
步骤7 由多路复用器在 I/O 现场中轮询各 Channel，处理连接结果
步骤8 如果连接成功，设置 Future 结果，发送连接成功事件，触发 ChannelPipeline 执行
步骤9 由 ChannelPipeline 调度执行系统和用户到 ChannelHandler，执行业务逻辑

```java
    public void connect(final int port, final String host) throws Exception {
        EventLoopGroup worker = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap(); // 1
            b.group(worker) // 2
                .channel(NioSocketChannel.class)
                .option(ChannelOption.TCP_NODELAY, true)
                .handler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) throws Exception {
                        ch.pipeline().addLast(new StringDecoder());
                        handlerAdapterMap.entrySet().stream()
                            .forEach(handlerAdapter -> ch.pipeline()
                                .addLast(handlerAdapter.getKey(), handlerAdapter.getValue()));
                        ch.pipeline().addLast(new StringEncoder());
                    }

                });

            ChannelFuture channelFuture = b.connect(host, port);

            while (true) {
                TimeUnit.SECONDS.sleep(5);
                channelFuture.channel().writeAndFlush("filter result");
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            worker.shutdownGracefully();
        }

    }
```

>  未完，有空待续，尴尬