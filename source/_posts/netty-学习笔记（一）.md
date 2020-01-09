---
title: netty 学习笔记（一）
date: 2018-11-05 17:36:00
categories:
		- netty学习笔记
tags:
		- java
		- netty
---

# NETTY-学习笔记（一）

> **netty**  is an advanced framework for creating high-performance networking applications 
> --- *netty in action* 

## netty初见
挑战：如何让自己编写的应用程序支持150000以上的并发？

### 传统的java代码如何实现网络访问
以下是一段传统java网络处理代码

<!--more-->

```java
ServerSocket serverSocket = new ServerSocket(portNumber);
Socket clientSocket = serverSocket.accept();
BufferedReader in = new BufferedReader(
	new InputStreamReader(clientSocket.getInputStream()));
PrintWriter out =
	new PrintWriter(clientSocket.getOutputStream(), true);
String request, response;
while ((request = in.readLine()) != null) {
	if ("Done".equals(request) (
		break;
	}
	response = processRequest(request);
	out.println(response);
} 
```
简而言之，这段代码做了一下几件事。
1. serverSocket 绑定到一个port端口
2. accept()等待连接建立
3. 不断地readLine()读取数据并返回response

从这段代码可以看出，在同一时刻只能处理一个链接，如果要处理多并发，则需要多个线程，每个线程处理一个socket client。
![传统网络模型图](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/%E4%BC%A0%E7%BB%9Fio%E6%A8%A1%E5%9E%8B.png)
如果我们采用传统的网络模型来支持高并发，会造成一下问题
- 首先大部分时间，大部分线程都在等待，线程利用率非常底下。
- 其次每个线程都需要栈内存开销，从64KB-1MB不等，当并发数量过多时，会大大消耗内存资源，造成OutOfMemoryException。
- 线程切换开销等操作同样会消耗过多的cpu资源。

### Java NIO
Java 1.4开始引入一种新的IO API，称为NIO，NIO有两种解释，New Input/Output 和 Non-blocking Input/Output ,但是无所谓了，哪种解释都可以。这里采用New Input/Output，对应的传统IO成为OIO。NIO和OIO最大差别在于socker的read/write操作在没有数据的时候会直接返回。这样将不会造成阻塞。NIO将通过一个selector来选择IOready的socket。
![NIO网络模型](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/nio%E6%A8%A1%E5%9E%8B.png)

好处非常明显。
1. 不会阻塞，由selector检查所有socket状态，而非等待一个socket IO ready。
2. 处理线程和socket解耦，线程可以高效利用起来，比如当IO等待时，线程可以归还到线程池处理别的任务。

### 我们需要netty
从JDK1.4起，Java便已经支持NIO，我们完全可以采用JDK自带的NIO模块来开发网络组建，但是Java的NIO库有些问题，一是与OIO语法非常不同，如果你想从OIO迁移到NIO，那不好意思，恐怕要完完全全重新写一遍代码。二是使用复杂，业务逻辑和网络相关代码过度耦合，开发难度大，维护难度也很大。而netty，很好的解决了这些问题，使用简单，功能强大。

-	设计：为多种网络方式（NIO/OIO）提供统一的API接口，简单强大的线程模型，业务逻辑解耦，易于复用。
-	使用简单：提供详细的文档和丰富的案例，包教包会。
-	性能：和原生API有着更高的吞吐量和更低的延迟，可以通过线程池减少资源使用。
-	健壮性：不会因为过慢过快过量的连接造成OutOfMemoryError。消除NIO应用在高速网络下的读写不平衡。
-	安全：完全支持SSL/TLS和StartTLS
-	社区驱动：netty社区里个个都是人才。

### netty核心组件
**netty**核心组件由以下四个模块构成
- Channels
- Callbacks
- Futures
- Events and handlers
对于netty中的三个概念：resources, logic, 和 notifications

#### netty-Channels
Channels 是Java NIO中一个非常基本的概念
> an open connection to an entity such as a hardware device, a file, a network socket, or a program component that is capable of performing one or more distinct I/O operations, for example reading or writing.

可以简单理解为传输数据的通道，能被open/close,connect/disconnect
#### netty-Callbakcs
Callbakcs(回掉)，是异步编程中的常见概念，这个就不说了。
#### netty-Futures
Futures是一个较为抽象的概念，它作为placeholder，代表一个异步过程完成后的结果值。将来的某个时刻，在异步过程完结后，可以统统Futures访问这个结果。JDK中提供对Future接口的默认实现类非常一般，需要程序员手工检查状态，并会造成阻塞。因此Netty自己提供一个对Future接口的实现，称为ChannelFuture。
通过向ChannelFuture注册Listener，待Future完成后会异步回掉Listener的方法。
ChannelFuture使用示例
```java
Channel channel = ...;
// Does not block
ChannelFuture future = channel.connect(
	new InetSocketAddress("192.168.0.1", 25));
future.addListener(new ChannelFutureListener() {
	@Override
	public void operationComplete(ChannelFuture future) {
		if (future.isSuccess()){
			ByteBuf buffer = Unpooled.copiedBuffer(
				 "Hello",Charset.defaultCharset());
			ChannelFuture wf = future.channel().writeAndFlush(buffer);
			....
		} else {
			Throwable cause = future.cause();
			cause.printStackTrace();
		}
	}
});
```
Callbakcs和Futures共同构成logic部分
#### netty- Events and handlers
**Netty**采用了多种events，这些events用来表示网络连接中各类状态变化和操作的结果。当这些events被触发时，可以采取一系列操作来处理这些events。这些操作可以包括一下几类：
-	日志记录
-	数据转换
-	流控制
-	业务逻辑

**Netty**的events，可以根据inbound data和outbound data，分为以下几类。
- Active or inactive connections
- Data Reads
- User event
- Error event

inbound event 和 outbound event 数据流模型图
![Alt text](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/%E6%95%B0%E6%8D%AE%E6%B5%81%E6%A8%A1%E5%9E%8B%E5%9B%BE.png)
对于每个event，都可以分配一个user-implement handler class来处理。当然Netty为Handler提供了一个基本抽象--ChannelHandler，并且提供一套预先处理好的Handler集合，可以支持HTTP 和 SSL/TLS协议。

## 年轻人的第一个Netty程序
我们来实现一个具体的Netty Server/Client程序，这个程序非常简单，Server只需要简单将Client发送的数据原封不动的返回即可，这个简单的Server，我们称之为EchoServer。我们目的是为了了解Netty框架的使用，因此Server的业务逻辑越简单越好。

### EchoServer代码
EchoServer的业务逻辑通过EchoServerHandler来实现
```java
@Sharable
public class EchoServerHandler extends ChannelInboundHandlerAdapter {
	 @Override
	 public void channelRead(ChannelHandlerContext ctx, Object msg) {
		 ByteBuf in = (ByteBuf) msg;
		 System.out.println(
		 "Server received: " + in.toString(CharsetUtil.UTF_8));
		 ctx.write(in);
	 }
	 @Override
	 public void channelReadComplete(ChannelHandlerContext ctx) {
		 ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
			 .addListener(ChannelFutureListener.CLOSE);
	 }
	 @Override
	 public void exceptionCaught(ChannelHandlerContext ctx, 
											   throwable cause) {
		 cause.printStackTrace();
		 ctx.close();
	 }
}
```
ChannelInboundHandlerAdapter默认实现了ChannelInboundHandler的接口，如果你只关心特定的event而不是所有event，可以通过继承ChannelInboundHandlerAdapter，其余的接口将会采用ChannelInboundHandlerAdapter的默认实现方法。这里我们主要处理以下三个event
- channelRead()当有数据写入时调用
- channelReadComplete()当读入的数据是最后一个的时候调用
- exceptionCaught()异常发生时调用

好了，接下来我们要启动这个服务器了。
```java
public class EchoServer {
	 private final int port;
	 public EchoServer(int port) {
		 this.port = port;
	 }
	 public static void main(String[] args) throws Exception {
		 if (args.length != 1) {
			 System.err.println(
			 "Usage: " + EchoServer.class.getSimpleName() +
				 " <port>");
		 }
		 int port = Integer.parseInt(args[0]);
		 new EchoServer(port).start();
	 }
	 public void start() throws Exceptio3n {
		 final EchoServerHandler serverHandler = new EchoServerHandler();
		 EventLoopGroup group = new NioEventLoopGroup();
		 try {
			 ServerBootstrap b = new ServerBootstrap();
			 b.group(group)
				 .channel(NioServerSocketChannel.class)
				 .localAddress(new InetSocketAddress(port))
				 .childHandler(new ChannelInitializer<SocketChannel>(){
					 @Override
					 public void initChannel(SocketChannel ch)
												 throws Exception {
						 ch.pipeline().addLast(serverHandler);
					 }
				 });
			 ChannelFuture f = b.bind().sync();
			 f.channel().closeFuture().sync();
		 } finally {
			 group.shutdownGracefully().sync();
		 }
	 }
}
```

ChannelInitializer之前的都非常好理解，那么这个ChannelInitializer是起什么作用？当Server绑定到port后，会监听连接请求信息，当有连接建立的时候，还生成一个Channel，而ChannelInitializer则是负责对这个Channel初始化的，可见在initChannel中，initializer给Channel添加了一个EchoServerHandler实例。代码的最后，是同步绑定服务器。

### EchoClient代码

```java
@Sharable
public class EchoClientHandler extends
							 SimpleChannelInboundHandler<ByteBuf> {
	 //建立连接是调用
	 @Override
	 public void channelActive(ChannelHandlerContext ctx) {
		 ctx.writeAndFlush(Unpooled.copiedBuffer("Netty rocks!",
		 CharsetUtil.UTF_8);
	 }
	 //从Server读到数据时调用
	 @Override
	 public void channelRead0(ChannelHandlerContext ctx, ByteBuf in) {
		 System.out.println(
			 "Client received: " + in.toString(CharsetUtil.UTF_8));
	 }
	 //异常时调用
	 @Override
	 public void exceptionCaught(ChannelHandlerContext ctx,
													Throwable cause) {
		 cause.printStrackTrace();
		 ctx.close();
	 }
}
```

Client端的Handler和Server端很类似，不过这里是Inbound。
注意， ChannelInboundHandlerAdapter也是存在的，不过这里并不采用，原因是SimpleChannelInboundHandler更简单了，它替我们处理一些内存释放等操作。

现在让我们启动client
```java
public class EchoClient {
	 private final String host;
	 private final int port;
	 public EchoClient(String host, int port) {
		 this.host = host;
		 this.port = port;
	 }
	 public void start() throws Exception {
		 EventLoopGroup group = new NioEventLoopGroup();
		 try {
			 Bootstrap b = new Bootstrap();
			 b.group(group)
				 .channel(NioSocketChannel.class)
				 .remoteAddress(new InetSocketAddress(host, port))
				 .handler(new ChannelInitializer<SocketChannel>() {
					 @Override
					 public void initChannel(SocketChannel ch)
											 throws Exception {
						 ch.pipeline().addLast(
						 new EchoClientHandler());
					 }
				 });
			 ChannelFuture f = b.connect().sync();
			 f.channel().closeFuture().sync();
		 } finally {
			 group.shutdownGracefully().sync();
		 }
	 }
	 public static void main(String[] args) throws Exception {
		 if (args.length != 2) {
			 System.err.println(
					 "Usage: " + EchoClient.class.getSimpleName() +
					 " <host> <port>");
			 return;
		 }
		 String host = args[0];
		 int port = Integer.parseInt(args[1]);
		 new EchoClient(host, port).start();
	 }
}
```
client端的代码与server端非常相似，仅有几处细微差别。

## Netty组件与设计
在初步认识Netty之后，本章详细介绍下Netty的各个组件与设计。

### Channel, EventLoop, and ChannelFuture
Channel, EventLoop, 和 ChannelFuture这三个组件可以理解为对Netty的抽象。
Channel是对Socket的抽象，EventLoop是对Control flow, multithreading, concurrency的抽象，ChannelFuture是对Asynchronous notification的抽象。

#### Channel
Channel定义了基础的IO操作，包括 bind(), connect(), read(), 和write()
Netty预定义了一些Channel，用来简化应用开发。
- EmbeddedChannel
- LocalServerChannel
- NioDatagramChannel
- NioSctpChannel
- NioSocketChannel
使用这些Channel可以简化程序开发。

#### EventLoop
EventLoop则是定义了connection的生命周期对事件处理的抽象。这里在一个高层次介绍Channels, EventLoops, Threads 以及EventLoopGroups之间的关系。
- 一个 EventLoopGroup 包含一个或多个 EventLoops.
- 一个 EventLoop 会绑定一个单独的线程.
- 所有的 I/O 事件都会被EventLoop的线程处理.
- 一个 Channel 会注册都一个EventLoop上.
- 一个 EventLoop 可能会拥有多个Channel.

![EventLoopGroup模型图](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/EventLoopGroup%E6%A8%A1%E5%9E%8B.png)

#### ChannelFuture
ChannelFuture负责在处理完成后的回掉工作。可以通过其addListener()注册一个ChannelFutureListener，用于在操作完成后进行回调（无论结果是否成功）。

### ChannelHandler 和 ChannelPipeline
ChannelHandler 和 ChannelHandler 两个组件主要处理数据流和业务逻辑。

#### ChannelHandler 
从ChannelHandler的角度来看，这是最核心的类，因为其直接负责业务逻辑代码的处理。通过EventDriven机制，ChannelHandler处理各种类型的event，从而实现业务逻辑。从上面例子可以见，Netty一样实现多个默认类，来简化应用开发。

#### ChannelPipeline
ChannelPipeline可以理解为存放链式ChannelHandler的容器，我们将Socket视为外部，java application视作内部，ChannelHandler分为两类，ChannelInboundHandler表示数据从socket到java，ChannelOutBoundHandler则相反。handler在pipeline中按链式存储，数据或者事件从一个handler传递到下一个handler，虽然两类handler混在一个pipeline中，但是两者不会混淆，netty能分辩两者，流入数据只会被ChannelInboundHandler处理，流出数据只会被ChannelOutboundHandler处理。
![enter image description here](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/ChannelPipelinewithinboundandoutboundChannelHandlers.png)

当一个handler被添加到pipeline中，会被绑定一个ChannelHandlerContext，context代表handler和pipeline的一种绑定关系，你可以使用context获得底层Channel来进行操作，这种一般用来写输出数据，这会导致数据从pipeline tail开始。第二中则是将数据写入到context中，这将使下一个handler来处理数据。

netty为开发者提供了二者adapter类，方法均作了默认实现，不做任何处理传递到下一个handler，开发者只需覆盖自己感兴趣的方法，其他采用默认实现即可。

* ChannelHandlerAdapter
* ChannelInboundHandlerAdapter 
* ChannelOutboundHandlerAdapter 
* ChannelDuplexHandlerAdapter

特殊handler：encoder和decoder。输出数据需要encode，从object转换成byte，输入数据需要decode，从byte转换成object。netty提供encoder/decoder不是ChannelInboundHandler就是ChannelOutboundHandler。

### Bootstrapping
netty bootstrap类是提供应用网络层配置容器，可以绑定程序到给定端口或者连接到一个正在监听的主机端口上。前者一般称为server而后者一般称为client，netty也有两类bootstrap，bootstrapserver和bootstrap。
两者差别除了上述行为外，还有一点很重要的，Bootstrap只有一个EventLoopGroup，而Bootstrap有两个EventLoopGroup。
server需要两个不同EventLoopGroup，前者只有一个ServerChannel，用来创建对每一个连接创建Channel，并交由第二个EventLoopGroup中的EventLoop处理。应该就是运用到多路io复用技术。
![enter image description here](http://ch-blog-img.oss-cn-shanghai.aliyuncs.com/blog/img/Server%20with%20two%20EventLoopGroups.png)
