dubbo在服务消费时调用的方法栈比较深，所以得一边看一边记，还是比较费力的。在dubbo服务发现中，我们看到通过ReferenceConfig#get()返回的是要调用接口的代理对象，因此通过接口的代理对象调用方法时是调用InvocationHandler(InvokerInvocationHandler)#invoke()方法，此时会调用注入的Invoker#invoke()方法，而InvokerInvocationHandler对象的invoker默认是DubboInvoker实现的，因此DubboInvoker#invoke()方法被调用，最终调用子类DubboInvoker#doInvoke()方法。
```java
// DubboInvoker=>AbstractInvoker.java
    public Result invoke(Invocation inv) throws RpcException {
        //设置一系列参数
        ......

        try {
            //调用子类方法
            return doInvoke(invocation);
        } catch (InvocationTargetException e) { // biz exception
            ......
        }
    }
```
这里是调用远程服务有三种，异步无返回值、异步有返回值、同步阻塞获取返回，以同步方式往下分析。
```java
// DubboInvoker
    protected Result doInvoke(final Invocation invocation) throws Throwable {
        RpcInvocation inv = (RpcInvocation) invocation;
        final String methodName = RpcUtils.getMethodName(invocation);
        //接口路径
        inv.setAttachment(Constants.PATH_KEY, getUrl().getPath());
        inv.setAttachment(Constants.VERSION_KEY, version);
        //获取已经建立连接的客户端
        ExchangeClient currentClient;
        if (clients.length == 1) {
            currentClient = clients[0];
        } else {
            currentClient = clients[index.getAndIncrement() % clients.length];
        }
        try {
            //先从invocation中获取，再从URL中获取
            boolean isAsync = RpcUtils.isAsync(getUrl(), invocation);
            boolean isOneway = RpcUtils.isOneway(getUrl(), invocation);
            int timeout = getUrl().getMethodParameter(methodName, Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
            //单向调用，直接调用不获取返回值
            if (isOneway) {
                boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
                currentClient.send(inv, isSent);
                RpcContext.getContext().setFuture(null);
                return new RpcResult();
            } else if (isAsync) {
                //异步调用有返回值需要手动获取
                ResponseFuture future = currentClient.request(inv, timeout);
                RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
                return new RpcResult();
            } else {
                //同步调用，get()会阻塞地获取返回值
                RpcContext.getContext().setFuture(null);
                return (Result) currentClient.request(inv, timeout).get();
            }
        } catch (TimeoutException e) {
            throw new RpcException(RpcException.TIMEOUT_EXCEPTION, "Invoke remote method timeout. method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        } catch (RemotingException e) {
            throw new RpcException(RpcException.NETWORK_EXCEPTION, "Failed to invoke remote method: " + invocation.getMethodName() + ", provider: " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
DubboProtocol#getSharedClient()生成netty客户端时，会使用ReferenceCountExchangeClient装饰客户端ExchangeClient，然后被保存到数组中，因此这里获取的currentClient为ReferenceCountExchangeClient，调用注入的ExchangeClient(HeaderExchangeClient)对象client的request()方法。
```java
// ReferenceCountExchangeClient
    public ResponseFuture request(Object request, int timeout) throws RemotingException {
        return client.request(request, timeout);
    }
```
这里会继续调用内部的ExchangeChannel对象channel的request()方法，那这里的channel怎么来得呢。构造HeaderExchangeClient对象时new出来的，同时构造方法中的client为NettyClient实例对象。
```java
// HeaderExchangeClient.java
    public ResponseFuture request(Object request, int timeout) throws RemotingException {
        return channel.request(request, timeout);
    }

    public HeaderExchangeClient(Client client, boolean needHeartbeat) {
        ....
        this.client = client;
        this.channel = new HeaderExchangeChannel(client);
        ....
    }
```
那继续往下看HeaderExchangeChannel#request()方法，好像离发送请求数据越来越近了，这里会先构造Request和DefaultFuture，再调用内部注入的channel(NettyClient)的send()方法。
```java
// HeaderExchangeChannel
    public ResponseFuture request(Object request, int timeout) throws RemotingException {
        if (closed) {
            throw new RemotingException(this.getLocalAddress(), null, "Failed to send request " + request + ", cause: The channel " + this + " is closed!");
        }
        // create request.
        Request req = new Request();
        req.setVersion("2.0.0");
        req.setTwoWay(true);
        req.setData(request);
        DefaultFuture future = new DefaultFuture(channel, req, timeout);
        try {
            channel.send(req);
        } catch (RemotingException e) {
            future.cancel();
            throw e;
        }
        return future;
    }
```
NettyClient的继承关系是NettyClient=>AbstractClient=>AbstractPeer，此时内部AbstractClient#send()方法被调用。这里就是先getChannel()获取通道再调用send()，再分析这个channel怎么来的，在doConnect()时如果连接服务端成功则能获取Channel，默认为NettyChannel。
```java
// AbstractClient
    public void send(Object message, boolean sent) throws RemotingException {
        if (send_reconnect && !isConnected()) {
            connect();
        }
        //getChannel()由子类NettyClient
        Channel channel = getChannel();
        //TODO Can the value returned by getChannel() be null? need improvement.
        if (channel == null || !channel.isConnected()) {
            throw new RemotingException(this, "message can not send, because channel is closed . url:" + getUrl());
        }
        channel.send(message, sent);
    }
```
NettyChannel#send()中就会把请求发送给服务端了。
```java
// NettyChannel
    public void send(Object message, boolean sent) throws RemotingException {
        super.send(message, sent);

        boolean success = true;
        int timeout = 0;
        try {
            ChannelFuture future = channel.write(message);
            if (sent) {
                timeout = getUrl().getPositiveParameter(Constants.TIMEOUT_KEY, Constants.DEFAULT_TIMEOUT);
                success = future.await(timeout);
            }
            Throwable cause = future.getCause();
            if (cause != null) {
                throw cause;
            }
        } catch (Throwable e) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress() + ", cause: " + e.getMessage(), e);
        }

        if (!success) {
            throw new RemotingException(this, "Failed to send message " + message + " to " + getRemoteAddress()
                    + "in timeout(" + timeout + "ms) limit");
        }
    }
```
至此服务消费者向服务提供者发送调用服务的请求完成，最终返回的在HeaderExchangeChannel#request()构建的DefaultFuture。还记得DubboInvoker#doInvoke()吗，这里的get()获取服务端响应的代码如下，就是不断轮询判断服务端是否响应，超时则抛出异常。
```java
// DefaultFuture
    public Object get(int timeout) throws RemotingException {
        if (timeout <= 0) {
            timeout = Constants.DEFAULT_TIMEOUT;
        }
        if (!isDone()) {
            long start = System.currentTimeMillis();
            lock.lock();
            try {
                while (!isDone()) {
                    done.await(timeout, TimeUnit.MILLISECONDS);
                    if (isDone() || System.currentTimeMillis() - start > timeout) {
                        break;
                    }
                }
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            } finally {
                lock.unlock();
            }
            if (!isDone()) {
                throw new TimeoutException(sent > 0, channel, getTimeoutMessage(false));
            }
        }
        return returnFromResponse();
    }
```
**总结下发送请求的过程：首先获取已经建立连接的netty客户端，然后构建Request和DefaultFuture，通过netty通道将请求发送给netty服务端之后，DefaultFuture#get()会超时等待，默认超时时间1S。**

接下来看服务端怎么处理调用请求的。服务提供者收到服务消费者的调用请求后，首先在DubboCodec#decodeBody()对编码后的请求字节流数据进行解码，得到调用远程服务的Request对象。
```java
    protected Object decodeBody(Channel channel, InputStream is, byte[] header) throws IOException {
        byte flag = header[2], proto = (byte) (flag & SERIALIZATION_MASK);
        Serialization s = CodecSupport.getSerialization(channel.getUrl(), proto);
        // get request id.
        long id = Bytes.bytes2long(header, 4);
        if ((flag & FLAG_REQUEST) == 0) {
           ......
        } else {
            // decode request.
            Request req = new Request(id);
            req.setVersion("2.0.0");
            req.setTwoWay((flag & FLAG_TWOWAY) != 0);
            if ((flag & FLAG_EVENT) != 0) {
                req.setEvent(Request.HEARTBEAT_EVENT);
            }
            try {
                Object data;
                if (req.isHeartbeat()) {
                    data = decodeHeartbeatData(channel, deserialize(s, channel.getUrl(), is));
                } else if (req.isEvent()) {
                    data = decodeEventData(channel, deserialize(s, channel.getUrl(), is));
                } else {
                    DecodeableRpcInvocation inv;
                    if (channel.getUrl().getParameter(
                            Constants.DECODE_IN_IO_THREAD_KEY,
                            Constants.DEFAULT_DECODE_IN_IO_THREAD)) {
                        inv = new DecodeableRpcInvocation(channel, req, is, proto);
                        inv.decode();
                    } else {
                        inv = new DecodeableRpcInvocation(channel, req,
                                new UnsafeByteArrayInputStream(readMessageData(is)), proto);
                    }
                    data = inv;
                }
                req.setData(data);
            } catch (Throwable t) {
                if (log.isWarnEnabled()) {
                    log.warn("Decode request failed: " + t.getMessage(), t);
                }
                // bad request
                req.setBroken(true);
                req.setData(t);
            }
            return req;
        }
    }
```
在创建netty服务器的时候会调用NettyServer#doOpen()创建NettyHandler，当前服务端收到请求数据时钩子函数messageReceived会被调用。
```java
//NettyHandler 
    public void messageReceived(ChannelHandlerContext ctx, MessageEvent e) throws Exception {
        NettyChannel channel = NettyChannel.getOrAddChannel(ctx.getChannel(), url, handler);
        try {
            handler.received(channel, e.getMessage());
        } finally {
            NettyChannel.removeChannelIfDisconnected(ctx.getChannel());
        }
    }
```
服务端处理请求是分层的，消费者调用提供者的请求在AllChannelHandler#received()中被处理的，此时请求会通过线程池调度线程执行。
```java
// AllChannelHandler
    public void received(Channel channel, Object message) throws RemotingException {
        ExecutorService cexecutor = getExecutorService();
        try {
            cexecutor.execute(new ChannelEventRunnable(channel, handler, ChannelState.RECEIVED, message));
        } catch (Throwable t) {
            ......
        }
    }
```
这里会用DecodeHandler对请求解码后到HeaderExchangeHandler#received()中继续处理。
```java
// ChannelEventRunnable
    public void run() {
        switch (state) {
            ......
            case RECEIVED:
                try {
                    handler.received(channel, message);
                } catch (Exception e) {
                    logger.warn("ChannelEventRunnable handle " + state + " operation error, channel is " + channel
                            + ", message is " + message, e);
                }
                break;
            ......
        }
    }
```
发现收到的消息是请求Request时，调用handleRequest()处理。
```java
// HeaderExchangeHandler
    public void received(Channel channel, Object message) throws RemotingException {
        channel.setAttribute(KEY_READ_TIMESTAMP, System.currentTimeMillis());
        ExchangeChannel exchangeChannel = HeaderExchangeChannel.getOrAddChannel(channel);
        try {
            if (message instanceof Request) {
                // handle request.
                Request request = (Request) message;
                if (request.isEvent()) {
                    handlerEvent(channel, request);
                } else {
                    if (request.isTwoWay()) {
                        Response response = handleRequest(exchangeChannel, request);
                        channel.send(response);
                    } else {
                        handler.received(exchangeChannel, request.getData());
                    }
                }
            } else if (message instanceof Response) {
                handleResponse(channel, (Response) message);
            } else if (message instanceof String) {
                if (isClientSide(channel)) {
                    Exception e = new Exception("Dubbo client can not supported string message: " + message + " in channel: " + channel + ", url: " + channel.getUrl());
                    logger.error(e.getMessage(), e);
                } else {
                    String echo = handler.telnet(channel, (String) message);
                    if (echo != null && echo.length() > 0) {
                        channel.send(echo);
                    }
                }
            } else {
                handler.received(exchangeChannel, message);
            }
        } finally {
            HeaderExchangeChannel.removeChannelIfDisconnected(channel);
        }
    }
```
在handleRequest()中会调用DubboProtocol中的属性requestHandler引用的匿名内部类对象中的reply()方法。
```java
// HeaderExchangeHandler
    Response handleRequest(ExchangeChannel channel, Request req) throws RemotingException {
        Response res = new Response(req.getId(), req.getVersion());
        if (req.isBroken()) {
            Object data = req.getData();

            String msg;
            if (data == null) msg = null;
            else if (data instanceof Throwable) msg = StringUtils.toString((Throwable) data);
            else msg = data.toString();
            res.setErrorMessage("Fail to decode request due to: " + msg);
            res.setStatus(Response.BAD_REQUEST);

            return res;
        }
        // find handler by message class.
        //获取请求数据，即客户端创建的RpcInvocation对象
        Object msg = req.getData();
        try {
            // handle data.
            //调用接口方法
            Object result = handler.reply(channel, msg);
            res.setStatus(Response.OK);
            res.setResult(result);
        } catch (Throwable e) {
            res.setStatus(Response.SERVICE_ERROR);
            res.setErrorMessage(StringUtils.toString(e));
        }
        return res;
    }
```
这里其实就是获取Invoker，再调用Invoker#invoke()。
```java
// DubboProtocol.java
    private ExchangeHandler requestHandler = new ExchangeHandlerAdapter() {

        @Override
        public Object reply(ExchangeChannel channel, Object message) throws RemotingException {
            if (message instanceof Invocation) {
                Invocation inv = (Invocation) message;
                Invoker<?> invoker = getInvoker(channel, inv);
                ......
                return invoker.invoke(inv);
            }
            ......
        }
        
    };
```
在DubboProtocol#export()暴露接口服务时会将serviceKey对应的DubboExporter添加到exporterMap(ConcurrentHashMap<String, Exporter<?>>)中，此时就可以通过serviceKey获取到对应的DubboExporter，再通过DubboExporter获取Invoker，Invoker是在ServiceConfig中的doExportUrlsFor1Protocol()中生成的。
```java
    Invoker<?> getInvoker(Channel channel, Invocation inv) throws RemotingException {
        ......
        String serviceKey = serviceKey(port, path, inv.getAttachments().get(Constants.VERSION_KEY), inv.getAttachments().get(Constants.GROUP_KEY));

        DubboExporter<?> exporter = (DubboExporter<?>) exporterMap.get(serviceKey);

        ......

        return exporter.getInvoker();
    }
```
找到Invoker以后就会调用invoke()，最终调用子类的doInvoke()方法，wrapper是动态生成的，但是逻辑就是调用接口实现类对象中的方法。
```java
    public Result invoke(Invocation invocation) throws RpcException {
        try {
            //调用子类的doInvoke()方法
            return new RpcResult(doInvoke(proxy, invocation.getMethodName(), invocation.getParameterTypes(), invocation.getArguments()));
        } catch (InvocationTargetException e) {
            return new RpcResult(e.getTargetException());
        } catch (Throwable e) {
            throw new RpcException("Failed to invoke remote proxy method " + invocation.getMethodName() + " to " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }

    public <T> Invoker<T> getInvoker(T proxy, Class<T> type, URL url) {
        // TODO Wrapper cannot handle this scenario correctly: the classname contains '$'
        final Wrapper wrapper = Wrapper.getWrapper(proxy.getClass().getName().indexOf('$') < 0 ? proxy.getClass() : type);
        return new AbstractProxyInvoker<T>(proxy, type, url) {
            @Override
            protected Object doInvoke(T proxy, String methodName,
                                      Class<?>[] parameterTypes,
                                      Object[] arguments) throws Throwable {
                return wrapper.invokeMethod(proxy, methodName, parameterTypes, arguments);
            }
        };
    }
```
调用方法获取到返回值以后就会在HeaderExchangeHandler#received()中将响应发给服务消费者，服务消费者再以类似的过程对响应数据进行解码，返回到应用层。
**总结下服务提供者处理调用请求的过程：首先对请求字节流数据进行解码，得到请求Request，然后到保存导出模块的map根据serviceKey获取到代理对象，最终通过代理对象调用接口实现类对象的方法，将返回值发送给服务提供者**
