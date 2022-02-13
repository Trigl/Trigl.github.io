---
layout:     post
title:      "使用 Netty 写一个 HTTP Server"
date:       2018-07-13
author:     "Ink Bai"
header-img: "/img/post/netty-httpserve.jpg"
catalog:    true
tags:
    - Netty
---

> 前一段时间需要写一个 HTTP Server，之前用 Akka 实现了一个，但是 Netty 更能支撑高并发，连 Spark 2.0 也把 Akka 换成了 Netty 进行网络传输，所以现在再撸一个 Netty 的 Server。

闲话少说，直接开搞，首先将 Netty 加入 sbt：

```scala
"io.netty" % "netty-all" % "4.1.25.Final"
```

我们需要定义两个组 `bossGroup` 和 `workerGroup`，这个两个组实际上是两个线程组，负责的不同。`bossGroup` 负责获取客户端连接，接收到之后会将该连接转发到 `workerGroup` 进行处理。

```scala
// if there are multiple `ServerBootstrap`, we should set more than one thread
val bossGroup = new NioEventLoopGroup(1)
val workerGroup = new NioEventLoopGroup()
```

然后通过一个启动服务类 `ServerBootstrap` 进行初始化启动：

```scala
val http = new ServerBootstrap()
  .group(bossGroup, workerGroup)
  .channel(classOf[NioServerSocketChannel])
  .childHandler(new HttpServerInitializer(writer))
```

其中 `HttpServerInitializer` 是我们自定义实现的一个服务初始化器，继承了 `ChannelInitializer` 类，这个类通过 `ChannelPipieline` 设置初始化链，包括超时时间、编解码和处理 HTTP 请求的设置。

```scala
class HttpServerInitializer(private val writer: EventWriter) extends ChannelInitializer[SocketChannel] {
  private val logger = Logger(this.getClass)

  override def initChannel(ch: SocketChannel) {
    logger.debug("Initialize http server channel.")
    ch.pipeline.addLast(
      new ReadTimeoutHandler(1),
      new HttpRequestDecoder(8192, 8192, 8192),
      new HttpResponseEncoder(),
      new HttpServerHandler(writer)
    )
  }
}
```

这里的 `HttpServerHandler` 就是主要的 HTTP 请求处理类，类似于 Spring 中的拦截器，包含处理请求、转发的相关功能：

```scala
class HttpServerHandler(private val writer: EventWriter)
  extends SimpleChannelInboundHandler[Object] {
  private val logger = Logger(this.getClass)

  private def forwardMessage(req: HttpRequest) {
    if (req.decoderResult.isSuccess) {
      val uri = req.uri
      logger.debug(s"Forward message, url=$uri")
      writer.putEvent(System.currentTimeMillis, uri)
    } else {
      logger.error(s"Request decode failure, result=${req.decoderResult}")
    }
  }

  override def channelReadComplete(ctx: ChannelHandlerContext) {
    ctx.flush
  }

  override def channelRead0(ctx: ChannelHandlerContext, msg: Object) {
    if (msg.isInstanceOf[HttpRequest]) {
      logger.debug("Flush and finish requests.")
      val response = new DefaultFullHttpResponse(HttpVersion.HTTP_1_1, HttpResponseStatus.NO_CONTENT, Unpooled.EMPTY_BUFFER)
      ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE)
      forwardMessage(msg.asInstanceOf[HttpRequest])
    }
  }

  override def exceptionCaught(ctx: ChannelHandlerContext, error: Throwable) {
    logger.error(s"Http handle error, error=$error")
    ctx.close
  }

}
```

注意这里是通过 `channelRead0` 处理请求并给客户端返回，我在这里的处理是不论什么内容进来，都给客户端返回一个 `NO_CONTENT`，而实际的处理会进行转发，这样是为了防止阻塞，首先给客户端返回结果，然后具体的一些耗时操作放在 `forwardMessage` 内。

上面的内容就是一些初始化启动类，准备好这些以后我们需要把初始化服务器绑定到服务端口，这个端口用来接收 HTTP 请求：

```scala
val f = http.bind(port).sync().channel()
logger.info(s"Start server, port=$port")
f.closeFuture().sync()
```

如果接收 HTTP 请求的时候出错，我们还需要释放资源：

```scala
bossGroup.shutdownGracefully()
workerGroup.shutdownGracefully()
```

以上完整实现了一个 HTTP Server，具体的过程是：

1. 定义两个线程组： `bossGroup` 和 `workerGroup`
2. 准备一个继承自 `ChannelInitializer` 服务处理初始化类，这个类设置超时编解码以及具体处理 HTTP 请求的类
3. 定义一个服务启动类 `ServerBootstrap`，设置线程组和 handler
4. 把服务启动类绑定到某个端口号，HTTP Server 启动

完整代码见 [producer](https://github.com/Trigl/ripple/tree/master/producer)，我是通过 Netty 的 HTTP Server 接收 GET 请求，处理之后将消息发送到 AWS Kinesis（类似于 Kafka）中。
