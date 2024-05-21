tomcat对请求进行处理分三个阶段，分别是io交互过程、数据加工过程、servlet处理过程，今天先看io过程。tomcat启动完成后会创建两个线程，一个是accept线程监听socket连接，一个是select线程轮询socket事件。
+ accept  
代码简化后如下，逻辑就是调用`endpoint(NioEndpoint)`的`serverSocketAccept()`方法监听来自客户端的连接请求，如果有客户端请求服务器，则会先建立连接，此时会返回一个socket。然后`setSocketOptions`中进行处理。
```java
    public void run() {

        try {
            while (!stopCalled) {

                .....

                try {

                    U socket = null;
                    try {
                        // 阻塞监听来自socket的连接
                        socket = endpoint.serverSocketAccept();
                    } catch (Exception ioe) {
                        ......
                    }
                    errorDelay = 0;

                    if (!stopCalled && !endpoint.isPaused()) {
                        //处理socket
                        if (!endpoint.setSocketOptions(socket)) {
                            endpoint.closeSocket(socket);
                        }
                    } else {
                        endpoint.destroySocket(socket);
                    }
                } catch (Throwable t) {
                    ......
                }
            }
        } finally {
            stopLatch.countDown();
        }
        state = AcceptorState.ENDED;
    }
```
处理的过程是创建NioSocketWrapper封装NioChannel、NioEndpoit、SocketChannel，将socket设置为非阻塞模式(read/write会立刻返回)，使用内置的`poller(Poller)`进行注册。
```java
    protected boolean setSocketOptions(SocketChannel socket) {
        NioSocketWrapper socketWrapper = null;
        try {
            NioChannel channel = null;
            if (nioChannels != null) {
                channel = nioChannels.pop();
            }
            if (channel == null) {
                SocketBufferHandler bufhandler = new SocketBufferHandler(
                        socketProperties.getAppReadBufSize(),
                        socketProperties.getAppWriteBufSize(),
                        socketProperties.getDirectBuffer());
                if (isSSLEnabled()) {
                    channel = new SecureNioChannel(bufhandler, this);
                } else {
                    channel = new NioChannel(bufhandler);
                }
            }
            NioSocketWrapper newWrapper = new NioSocketWrapper(channel, this);
            channel.reset(socket, newWrapper);
            connections.put(socket, newWrapper);
            socketWrapper = newWrapper;

            socket.configureBlocking(false);
            if (getUnixDomainSocketPath() == null) {
                socketProperties.setProperties(socket.socket());
            }

            socketWrapper.setReadTimeout(getConnectionTimeout());
            socketWrapper.setWriteTimeout(getConnectionTimeout());
            socketWrapper.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
            poller.register(socketWrapper);
            return true;
        } catch (Throwable t) {
            ......
        }
        return false;
    }
```
注册到Poller的过程使用Java中生产、消费模式，先创建PollerEvent，再添加到events`private final SynchronizedQueue<PollerEvent> events =new SynchronizedQueue<>();`,然后唤醒selector，接下来看select线程表演。
```java
        public void register(final NioSocketWrapper socketWrapper) {
            socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
            PollerEvent pollerEvent = createPollerEvent(socketWrapper, OP_REGISTER);
            addEvent(pollerEvent);
        }
        private void addEvent(PollerEvent event) {
            events.offer(event);
            if (wakeupCounter.incrementAndGet() == 0) {
                selector.wakeup();
            }
        }
```
+ select  
Poller的select线程现在还在调用select等待客户端的连接，收到`wakeup()`信号后，立刻被唤醒，然后到`events()`方法进行处理。`events()`做的事情就是注册socket的Read事件`sc.register(getSelector(), SelectionKey.OP_READ, socketWrapper);`,同事绑定封装的NioSocketWrapper到socket上面。
```java
public void run() {
            while (true) {

                boolean hasEvents = false;

                try {
                    if (!close) {
                        hasEvents = events();
                        if (wakeupCounter.getAndSet(-1) > 0) {
                            keyCount = selector.selectNow();
                        } else {
                            keyCount = selector.select(selectorTimeout);
                        }
                        wakeupCounter.set(0);
                    }
                    ......
                    if (keyCount == 0) {
                        hasEvents = (hasEvents | events());
                    }
                } catch (Throwable x) {
                    ......
                }

                Iterator<SelectionKey> iterator =
                    keyCount > 0 ? selector.selectedKeys().iterator() : null;
                while (iterator != null && iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    iterator.remove();
                    NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                    if (socketWrapper != null) {
                        processKey(sk, socketWrapper);
                    }
                }

                timeout(keyCount,hasEvents);
            }

            getStopLatch().countDown();
        }
```
此时第一遍循环结束，Poller的`run`开始第二次循环用于读取来自客户端发送的请求数据，当socket可读时，select返回可读的socket数量keyCount`private AtomicLong wakeupCounter = new AtomicLong(0);`，然后读取socket数据`processKey(sk, socketWrapper);`。
```java
                while (iterator != null && iterator.hasNext()) {
                    SelectionKey sk = iterator.next();
                    iterator.remove();
                    NioSocketWrapper socketWrapper = (NioSocketWrapper) sk.attachment();
                    // Attachment may be null if another thread has called
                    // cancelledKey()
                    if (socketWrapper != null) {
                        processKey(sk, socketWrapper);
                    }
                }
```
接下来会创建SocketProcessor交由tomcat内置线程池进行异步处理，再往下就是读取客户端的数据加工成Request对象，这个后面再看。
```java
    public boolean processSocket(SocketWrapperBase<S> socketWrapper,
            SocketEvent event, boolean dispatch) {
        try {
            if (socketWrapper == null) {
                return false;
            }
            SocketProcessorBase<S> sc = null;
            if (processorCache != null) {
                sc = processorCache.pop();
            }
            if (sc == null) {
                sc = createSocketProcessor(socketWrapper, event);
            } else {
                sc.reset(socketWrapper, event);
            }
            Executor executor = getExecutor();
            if (dispatch && executor != null) {
                executor.execute(sc);
            } else {
                sc.run();
            }
        } ......
        return true;
    }
```