# IM项目说明文档

# 一、netty

## (一)netty是什么

1）本质：JBoss做的一个Jar包

2）目的：快速开发高性能、高可靠性的网络服务器和客户端程序

3）优点：提供异步的、事件驱动的网络应用程序框架和工具

通俗的说：一个好用的处理Socket的东西，netty可以进行socket的服务端和客户端的编程，Netty是建立在NIO基础之上的。

## (二)netty相关概念

### 1.socket

Socket是一个抽象的接口，可以理解为网络中连接的两端 ，网络上的两个程序通过一个双向的通信连接实现数据的交换，这个连接的一端称为一个socket

### 2.NIO和BIO

**BIO**：同步阻塞式IO，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，当然可以通过线程池机制改善。  

**NIO**（None Blocking IO ）：同步非阻塞式IO，服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。

NIO核心在于：通道和缓冲区（Buffer），通道表示IO源daoIO设备（例如：文件，套接字）的连接，若需要时使用NIO需要获取IO设备中的通道以及用于容纳的数据缓冲区，对数据进行处理  缓冲区底层就是数组  简而言之：Channel 负责传输，Buffer负责存储 

NIO与BIO最大的**区别**就是只需要开启一个线程就可以处理来自多个客户端的IO事件   

**各自应用场景**  

（1）NIO适合处理连接数目特别多，但是连接比较短（轻操作）的场景，Jetty，Mina，ZooKeeper等都是基于java nio实现。

（2）BIO方式适用于连接数目比较小且固定的场景，这种方式对服务器资源要求比较高，并发局限于应用中。

### 3.channel



## (三)netty应用场景

 1）分布式进程通信

例如: hadoop、dubbo、akka等具有分布式功能的框架，底层RPC通信都是基于netty实现的

 2）游戏服务器开发
最新的游戏服务器有部分公司可能已经开始采用netty4.x 或 netty5.x

# 二、IM项目

## 1.项目目录

![img](http://chuantu.biz/t6/350/1532931428x1822611263.png) 

服务器端代码目录

![img](http://chuantu.biz/t6/350/1532932229x1822611251.png) 

客户端代码目录

![img](http://chuantu.biz/t6/350/1532933228x-1404817724.png) 

## 2.项目流程

1.启动项目之后，会先去初始化IMServer和IMWebSocketServer两个bean，**这个项目在网页上使用的IMWebSocketServer。**

2.客户端：进入主页面之后，点击单聊，进入chat页面中，在chat.jsp嵌入的JS代码中，建立了与服务端的连接，并且加入有关于chat聊天的业务代码，**其中核心是JS通过DWR**，获取了后端的代码中的DwrController，而DwrController又调用了DwrConnert接口，实现了**connect（）**创建连接，**close（）**关闭连接，**pushMessage（）**发送消息三个方法，从而实现了客户端的基本操作。客户端本身不需要发送接收请求，服务端会自行推送消息给客户端。

3.服务器：核心代码在IMWebSocketServerHandler类里面的channelRead0的方法去调用receiveMessages方法，然后根据信息的的protocol来匹配不同的处理方法，通过ImConnert去完成不同的处理方法。

```
channelRead0()方法用于处理接收到的信息
```

```
private void receiveMessages(ChannelHandlerContext hander, MessageWrapper wrapper) {
   //设置消息来源为Websocket
   wrapper.setSource(Constants.ImserverConfig.WEBSOCKET);//Constants常量
    //根据wrapper.protocol来决定处理方法
    if (wrapper.isConnect()) {//请求连接
           connertor.connect(hander, wrapper); 
    } else if (wrapper.isClose()) {//关闭连接
       connertor.close(hander,wrapper);
    } else if (wrapper.isHeartbeat()) {//发送心跳包给客户端
       connertor.heartbeatToClient(hander,wrapper);
    }else if (wrapper.isGroup()) {//一个群组群发
       connertor.pushGroupMessage(wrapper);
    }else if (wrapper.isSend()) {//客户发送请求
       connertor.pushMessage(wrapper);
    } else if (wrapper.isReply()) {//客户发送回复
       connertor.pushMessage(wrapper.getSessionId(),wrapper);
    }  
}
```

## 3.相关代码描述

ServerBootstrap：启动NIO服务的辅助启动类

NioEventLoopGroup：用来处理IO操作的多线程事件循环器

handler:监听channel动作以及状态改变

childerhandler():添加handler来监听已连接的客户端的channel动作和状态

chinnelpipelin：交互信息通过它进行传递，可以addLast（addFitst）多个handler

```
ImWebsocketServer和ImWebsocketServerHandler
```

```
import static io.netty.buffer.Unpooled.wrappedBuffer;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.buffer.ByteBuf;
import io.netty.channel.Channel;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.ChannelPipeline;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.MessageToMessageDecoder;
import io.netty.handler.codec.MessageToMessageEncoder;
import io.netty.handler.codec.http.HttpObjectAggregator;
import io.netty.handler.codec.http.HttpServerCodec;
import io.netty.handler.codec.http.websocketx.BinaryWebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketFrame;
import io.netty.handler.codec.http.websocketx.WebSocketServerProtocolHandler;
import io.netty.handler.codec.http.websocketx.extensions.compression.WebSocketServerCompressionHandler;
import io.netty.handler.codec.protobuf.ProtobufDecoder;
import io.netty.handler.stream.ChunkedWriteHandler;
import io.netty.handler.timeout.IdleStateHandler;

import java.io.IOException;
import java.net.ServerSocket;
import java.net.Socket;
import java.util.List;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.google.protobuf.MessageLite;
import com.google.protobuf.MessageLiteOrBuilder;
import com.zhuoan.constant.Constants;
import com.zhuoan.server.connertor.impl.ImConnertorImpl;
import com.zhuoan.server.model.proto.MessageProto;
import com.zhuoan.server.proxy.MessageProxy;


public class ImWebsocketServer  {

    private final static Logger log = LoggerFactory.getLogger(ImWebsocketServer.class);

    //译码器
    private ProtobufDecoder decoder = new ProtobufDecoder(MessageProto.Model.getDefaultInstance());
    //代理
    private MessageProxy proxy = null;
    private ImConnertorImpl connertor; 
    private int port;
    //EventLoopGroup处理IO操作多线程事件循环器   boss接收进来的连接 worker处理已经被接收的连接
    private final EventLoopGroup bossGroup = new NioEventLoopGroup();
    private final EventLoopGroup workerGroup = new NioEventLoopGroup();
    private Channel channel;
    
    public void init() throws Exception {
        log.info("start qiqiim websocketserver ...");

        // Server 服务启动
        /**
         * ServerBootstrap 是一个启动NIO服务的辅助启动类
         */
        ServerBootstrap bootstrap = new ServerBootstrap();
        /***
         * NioEventLoopGroup 是用来处理I/O操作的多线程事件循环器，
         * Netty提供了许多不同的EventLoopGroup的实现用来处理不同传输协议。
         * 第一个经常被叫做‘boss’，用来接收进来的连接。
         * 第二个经常被叫做‘worker’，用来处理已经被接收的连接， 一旦‘boss’接收到连接，就会把连接信息注册到‘worker’上。
         * 如何知道多少个线程已经被使用，如何映射到已经创建的Channels上都需要依赖于EventLoopGroup的实现，
         * 并且可以通过构造函数来配置他们的关系。
         */
        bootstrap.group(bossGroup, workerGroup);
        /***
         * ServerSocketChannel以NIO的selector为基础进行实现的，用来接收新的连接
         * 这里告诉Channel如何获取新的连接.
         */
        bootstrap.channel(NioServerSocketChannel.class);

        /***
         * 这里的事件处理类经常会被用来处理一个最近的已经接收的Channel。
         * ChannelInitializer是一个特殊的处理类，
         * 他的目的是帮助使用者配置一个新的Channel。
         * 也许你想通过增加一些处理类比如NettyServerHandler来配置一个新的Channel
         * 或者其对应的ChannelPipeline来实现你的网络程序。 当你的程序变的复杂时，可能你会增加更多的处理类到pipline上，
         * 然后提取这些匿名类到最顶层的类上。
         */
        bootstrap.childHandler(new ChannelInitializer<SocketChannel>() {
            @Override
            public void initChannel(SocketChannel ch) throws Exception {

                /**
                 * ChannelPipeline是ChannelHandler的容器，它负责ChannelHandler的管理和事件拦截与调度。
                 * Netty的ChannelPipeline和ChannelHandler机制类似于Servlet和Filter过滤器，这类拦截器实际上是职责链模式的一种变形，
                 * 主要是为了方便事件的拦截和用户业务逻辑的定制。Netty的Channel过滤器实现原理与Servlet Filter机制一致，
                 * 它将Channel的数据管道抽象为ChannelPipeline，消息在ChannelPipeline中流动和传递。
                 * ChannelPipeline持有I/O事件拦截器ChannelHandler的链表，由ChannelHandler对I/O事件进行拦截和处理，
                 * 可以方便地通过新增和删除ChannelHandler来实现不同的业务逻辑定制，不需要对已有的ChannelHandler进行修改，
                 * 能够实现对修改封闭和对扩展的支持。
                 */
                ChannelPipeline pipeline = ch.pipeline();


                //websocket在请求连接后, 会有一个http握手请求,

              // HTTP请求的解码和编码
               pipeline.addLast(new HttpServerCodec());
               // 把多个消息转换为一个单一的FullHttpRequest或是FullHttpResponse，
               // 原因是HTTP解码器会在每个HTTP消息中生成多个消息对象HttpRequest/HttpResponse,HttpContent,LastHttpContent
               pipeline.addLast(new HttpObjectAggregator(Constants.ImserverConfig.MAX_AGGREGATED_CONTENT_LENGTH));
               // 主要用于处理大数据流，比如一个1G大小的文件如果你直接传输肯定会撑暴jvm内存的; 增加之后就不用考虑这个问题了
               pipeline.addLast(new ChunkedWriteHandler());//Chunked分块的
               // WebSocket数据压缩
               pipeline.addLast(new WebSocketServerCompressionHandler());
               // 协议包长度限制
               pipeline.addLast(new WebSocketServerProtocolHandler("/ws", null, true, Constants.ImserverConfig.MAX_FRAME_LENGTH));
               // 协议包解码
               pipeline.addLast(new MessageToMessageDecoder<WebSocketFrame>() {
                   @Override
                   protected void decode(ChannelHandlerContext ctx, WebSocketFrame frame, List<Object> objs) throws Exception {
                       ByteBuf buf = ((BinaryWebSocketFrame) frame).content();
                       objs.add(buf);
                       buf.retain();
                   }
               });
               // 协议包编码
               pipeline.addLast(new MessageToMessageEncoder<MessageLiteOrBuilder>() {
                   @Override
                   protected void encode(ChannelHandlerContext ctx, MessageLiteOrBuilder msg, List<Object> out) throws Exception {
                       ByteBuf result = null;
                       if (msg instanceof MessageLite) {
                           result = wrappedBuffer(((MessageLite) msg).toByteArray());
                       }
                       if (msg instanceof MessageLite.Builder) {
                           result = wrappedBuffer(((MessageLite.Builder) msg).build().toByteArray());
                       }
                       // 然后下面再转成websocket二进制流，因为客户端不能直接解析protobuf编码生成的
                       WebSocketFrame frame = new BinaryWebSocketFrame(result);
                       out.add(frame);
                   }
               });
               // 协议包解码时指定Protobuf字节数实例化为CommonProtocol类型
               pipeline.addLast(decoder);
               pipeline.addLast(new IdleStateHandler(Constants.ImserverConfig.READ_IDLE_TIME,Constants.ImserverConfig.WRITE_IDLE_TIME,0));

               // 业务处理器，proxy是消息盒子，connertor连接器
               pipeline.addLast(new ImWebSocketServerHandler(proxy,connertor));
              
            }
        });
        
        // 可选参数
        /***
         * option()是提供给NioServerSocketChannel用来接收进来的连接。
         * childOption()是提供给由父管道ServerChannel接收到的连接，
         */
       bootstrap.childOption(ChannelOption.TCP_NODELAY, true);
        // 绑定接口，同步等待成功
        log.info("start qiqiim websocketserver at port[" + port + "].");

        /** 绑定端口并启动去接收进来的连接 **/
        ChannelFuture future = bootstrap.bind(port).sync();
       channel = future.channel();
        future.addListener(new ChannelFutureListener() {
            public void operationComplete(ChannelFuture future) throws Exception {
                if (future.isSuccess()) {
                    log.info("websocketserver have success bind to " + port);
                } else {
                    log.error("websocketserver fail bind to " + port);
                }
            }
        });
       // future.channel().closeFuture().syncUninterruptibly();
    }


    public void destroy() {
        log.info("destroy qiqiim websocketserver ...");
        // 释放线程池资源
        if (channel != null) {
         channel.close();
      }
        bossGroup.shutdownGracefully();
        workerGroup.shutdownGracefully();
        log.info("destroy qiqiim webscoketserver complate.");
    }

    public void setPort(int port) {
        this.port = port;
    }

   public void setProxy(MessageProxy proxy) {
      this.proxy = proxy;
   }
 

    public void setConnertor(ImConnertorImpl connertor) {
      this.connertor = connertor;
   }
    
    
}
```

```
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.channel.ChannelHandler.Sharable;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;

import org.apache.commons.lang.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import com.zhuoan.constant.Constants;
import com.zhuoan.server.connertor.impl.ImConnertorImpl;
import com.zhuoan.server.model.MessageWrapper;
import com.zhuoan.server.model.proto.MessageProto;
import com.zhuoan.server.proxy.MessageProxy;
import com.zhuoan.util.ImUtils;
@Sharable
public class ImWebSocketServerHandler   extends SimpleChannelInboundHandler<MessageProto.Model>{

   private final static Logger log = LoggerFactory.getLogger(ImWebSocketServerHandler.class);
    private ImConnertorImpl connertor = null;//服务器的连接控制器，负责处理服务器的连接，断开，发送消息等等的请求
    private MessageProxy proxy = null;

    public ImWebSocketServerHandler(MessageProxy proxy, ImConnertorImpl connertor) {
        this.connertor = connertor;
        this.proxy = proxy;
    }
   
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object o) throws Exception {
        String sessionId = ctx.channel().attr(Constants.SessionConfig.SERVER_SESSION_ID).get();
       //发送心跳包
       if (o instanceof IdleStateEvent && ((IdleStateEvent) o).state().equals(IdleState.WRITER_IDLE)) {
           if(StringUtils.isNotEmpty(sessionId)){
              MessageProto.Model.Builder builder = MessageProto.Model.newBuilder();
              builder.setCmd(Constants.CmdType.HEARTBEAT);
               builder.setMsgtype(Constants.ProtobufType.SEND);
              ctx.channel().writeAndFlush(builder);
           } 
          log.debug(IdleState.WRITER_IDLE +"... from "+sessionId+"-->"+ctx.channel().remoteAddress()+" nid:" +ctx.channel().id().asShortText());
       } 
           
       //如果心跳请求发出70秒内没收到响应，则关闭连接
       if ( o instanceof IdleStateEvent && ((IdleStateEvent) o).state().equals(IdleState.READER_IDLE)){
         log.debug(IdleState.READER_IDLE +"... from "+sessionId+" nid:" +ctx.channel().id().asShortText());
         Long lastTime = (Long) ctx.channel().attr(Constants.SessionConfig.SERVER_SESSION_HEARBEAT).get();
          if(lastTime == null || ((System.currentTimeMillis() - lastTime)/1000>= Constants.ImserverConfig.PING_TIME_OUT))
          {
             connertor.close(ctx);
          }
          //ctx.channel().attr(Constants.SessionConfig.SERVER_SESSION_HEARBEAT).set(null);
       }
   }

   @Override
    //channelRead0()方法用于处理接收到的信息
   protected void channelRead0(ChannelHandlerContext ctx, MessageProto.Model message)
         throws Exception {
        try {
            String sessionId = connertor.getChannelSessionId(ctx);
                // inbound
                if (message.getMsgtype() == Constants.ProtobufType.SEND) {//如果是请求类型
                   ctx.channel().attr(Constants.SessionConfig.SERVER_SESSION_HEARBEAT).set(System.currentTimeMillis());//心跳包
                    MessageWrapper wrapper = proxy.convertToMessageWrapper(sessionId, message);//信息封装   （wrapper封装   convert 转换）
                    if (wrapper != null)
                        receiveMessages(ctx, wrapper);
                }
                // outbound
                if (message.getMsgtype() == Constants.ProtobufType.REPLY) {
                   MessageWrapper wrapper = proxy.convertToMessageWrapper(sessionId, message);
                   if (wrapper != null)
                      receiveMessages(ctx, wrapper);
                }
           } catch (Exception e) {
               log.error("ImWebSocketServerHandler channerRead error.", e);
               throw e;
           }       
   }
   
   
   public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
       log.info("ImWebSocketServerHandler  join from "+ImUtils.getRemoteAddress(ctx)+" nid:" + ctx.channel().id().asShortText());
    }

    public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
        log.debug("ImWebSocketServerHandler Disconnected from {" +ctx.channel().remoteAddress()+"--->"+ ctx.channel().localAddress() + "}");
    }

    public void channelActive(ChannelHandlerContext ctx) throws Exception {
        super.channelActive(ctx);
        log.debug("ImWebSocketServerHandler channelActive from (" + ImUtils.getRemoteAddress(ctx) + ")");
    }

    public void channelInactive(ChannelHandlerContext ctx) throws Exception {
        super.channelInactive(ctx);
        log.debug("ImWebSocketServerHandler channelInactive from (" + ImUtils.getRemoteAddress(ctx) + ")");
        String sessionId = connertor.getChannelSessionId(ctx);
        receiveMessages(ctx,new MessageWrapper(MessageWrapper.MessageProtocol.CLOSE, sessionId,null, null));  
    }

    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
        log.warn("ImWebSocketServerHandler (" + ImUtils.getRemoteAddress(ctx) + ") -> Unexpected exception from downstream." + cause);
    }


    /**
     * to send message
     *
     * @param hander
     * @param wrapper
     */
    private void receiveMessages(ChannelHandlerContext hander, MessageWrapper wrapper) {
       //设置消息来源为Websocket
       wrapper.setSource(Constants.ImserverConfig.WEBSOCKET);//Constants常量
        //根据wrapper.protocol来决定处理方法
        if (wrapper.isConnect()) {//请求连接
               connertor.connect(hander, wrapper); 
        } else if (wrapper.isClose()) {//关闭连接
           connertor.close(hander,wrapper);
        } else if (wrapper.isHeartbeat()) {//发送心跳包给客户端
           connertor.heartbeatToClient(hander,wrapper);
        }else if (wrapper.isGroup()) {//一个群组群发
           connertor.pushGroupMessage(wrapper);
        }else if (wrapper.isSend()) {//客户发送请求
           connertor.pushMessage(wrapper);
        } else if (wrapper.isReply()) {//客户发送回复
           connertor.pushMessage(wrapper.getSessionId(),wrapper);
        }  
    }
}
```
