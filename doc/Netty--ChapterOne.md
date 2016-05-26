
##What is Netty?
Netty is a non-blocking, event driven, networking framework. In reality what this means for you is that Netty uses threads to process IO events. 

##阻塞式IO
阻塞式IO使用socket和server scoket来实现数据交互，通用的处理框架可以理解为一个client conection在Server端对应一个处理Thread。受限于CPU和CPU，并发处理线程数受限。

~~~
public class PlainOioServer {
    public void serve(int port) throws IOException {
        final ServerSocket socket = new ServerSocket(port);
        try {
            while (true) {
                final Socket clientSocket = socket.accept();
                System.out.println("Accepted connection from " + clientSocket);
                new Thread(new Runnable() {public void run() {
                        OutputStream out;
                        try {
                            out = clientSocket.getOutputStream();
                            out.write("Hi!\r\n".getBytes(Charset.forName("UTF-8")));
                            out.flush();
                            clientSocket.close();
                        } catch (IOException e) {
                            e.printStackTrace();
                            try {
                                clientSocket.close();
                            } catch (IOException ex) {
                                // ignore on close
                            }
                        }
                    }
                }).start();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
~~~

##Java NIO 

~~~
public class NIOServer {
        // 通道管理器
        private Selector selector;

        public void initServer(int port) throws Exception {
                // 获得一个ServerSocket通道
                ServerSocketChannel serverChannel = ServerSocketChannel.open();
                // 设置通道为 非阻塞
                serverChannel.configureBlocking(false);
                // 将该通道对于的serverSocket绑定到port端口
                serverChannel.socket().bind(new InetSocketAddress(port));
                // 获得一道管理器
                this.selector = Selector.open();

                // 将通道管理器和该通道绑定，并为该通道注册selectionKey.OP_ACCEPT事件
                // 注册该事件后，当事件到达的时候，selector.select()会返回，
                // 如果事件没有到达selector.select()会一直阻塞

                serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        }

        // 采用轮训的方式监听selector上是否有需要处理的事件，如果有，进行处理
        public void listen() throws Exception {
                System.out.println("start server");
                // 轮询访问selector
                while (true) {
                        // 当注册事件到达时，方法返回，否则该方法会一直阻塞
                        selector.select();
                        // 获得selector中选中的相的迭代器，选中的相为注册的事件
                        Iterator ite = this.selector.selectedKeys().iterator();
                        while (ite.hasNext()) {
                                SelectionKey key = (SelectionKey) ite.next();
                                // 删除已选的key 以防重负处理
                                ite.remove();
                                // 客户端请求连接事件
                                if (key.isAcceptable()) {
                                        ServerSocketChannel server = (ServerSocketChannel) key.channel();
                                        // 获得和客户端连接的通道
                                        SocketChannel channel = server.accept();
                                        // 设置成非阻塞
                                        channel.configureBlocking(false);
                                        // 在这里可以发送消息给客户端
                                        channel.write(ByteBuffer.wrap(new String("hello client").getBytes()));
                                        // 在客户端 连接成功之后，为了可以接收到客户端的信息，需要给通道设置读的权限
                                        channel.register(this.selector, SelectionKey.OP_READ);
                                        // 获得了可读的事件

                                } else if (key.isReadable()) {
                                        read(key);
                                }

                        }
                }
        }

        // 处理 读取客户端发来的信息事件
        private void read(SelectionKey key) throws Exception {
                // 服务器可读消息，得到事件发生的socket通道
                SocketChannel channel = (SocketChannel) key.channel();
                // 穿件读取的缓冲区
                ByteBuffer buffer = ByteBuffer.allocate(10);
                channel.read(buffer);
                byte[] data = buffer.array();
                String msg = new String(data).trim();
                System.out.println("server receive from client: " + msg);
                ByteBuffer outBuffer = ByteBuffer.wrap(msg.getBytes());
                channel.write(outBuffer);
        }
}  
~~~

##Netty BIO

~~~
public class NettyOioServer {

    public void server(int port) throws Exception {
        final ByteBuf buf = Unpooled.wrappedBuffer(
                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));

        // Use OioEventLoopGroup Ito allow blocking mode (Old-IO)
        EventLoopGroup group = new OioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)
                    .channel(OioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(port))
                            //Specify ChannelInitializer that will be called for each accepted connection
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(
                                    //Add ChannelHandler to intercept events and allow to react on them
                                    new ChannelInboundHandlerAdapter() {
                                        @Override
                                        public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                            System.out.println("--active--");
                                            //Write message to client and add ChannelFutureListener to close connection once message written
                                            ctx.write(buf.duplicate()).addListener(ChannelFutureListener.CLOSE);
                                        }
                                    });
                        }
                    });

            /**
             * Bind server to accept connections
             */
            ChannelFuture f = b.bind().sync();
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }


    public static void main(String args[]) throws Exception {

        int port = 9898;
        NettyOioServer server = new NettyOioServer();
        System.out.println("bind port " + port);
        server.server(port);
    }
}
~~~



##Netty NIO

~~~
public class NettyNioServer {

    public void server(int port) throws Exception {
        final ByteBuf buf = Unpooled.wrappedBuffer(
                Unpooled.copiedBuffer("Hi!\r\n", Charset.forName("UTF-8")));

        EventLoopGroup group = new NioEventLoopGroup();
        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(group)
                    .channel(NioServerSocketChannel.class)
                    .localAddress(new InetSocketAddress(port))
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        public void initChannel(SocketChannel ch) throws Exception {
                            ch.pipeline().addLast(
                                    new ChannelInboundHandlerAdapter() {
                                        @Override
                                        public void channelActive(ChannelHandlerContext ctx) throws Exception {
                                            System.out.println("--active--");
                                            //Write message to client and add ChannelFutureListener to close connection once message written
                                            ctx.write(buf.duplicate()).addListener(ChannelFutureListener.CLOSE);
                                        }
                                    });
                        }
                    });

            /**
             * Bind server to accept connections
             */
            ChannelFuture f = b.bind().sync();
            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully().sync();
        }
    }


    public static void main(String args[]) throws Exception {

        int port = 9898;
        NettyOioServer server = new NettyOioServer();
        System.out.println("bind port " + port);
        server.server(port);
    }
}
~~~
 

##An example

The EventLoopGroup may contain more then one EventLoop, but if so depends on configuration. Each Channel will have one EventLoop „bind“ to it once it was created which will never change.


ChannelHandler can be thought of as any piece of code that processes data coming and going through the ChannelPipeline. 
here is a clear distinction between inbound (ChannelInboundHandler) and outbound (ChannelOutboundHandler)