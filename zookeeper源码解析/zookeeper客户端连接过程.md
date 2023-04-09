---
title: zookeeper客户端连接过程
tags:
notebook: zookeeper源码解析
---
我们都知道zookeeper的客户端连接过程都是从`public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,boolean canBeReadOnly);`开始的，现在就从这个类的构造开始分析。
```java
    public ZooKeeper(String connectString, int sessionTimeout, Watcher watcher,
            boolean canBeReadOnly)
        throws IOException
    {
        LOG.info("Initiating client connection, connectString=" + connectString
                + " sessionTimeout=" + sessionTimeout + " watcher=" + watcher);

        watchManager.defaultWatcher = watcher;
//解析连接字符串
        ConnectStringParser connectStringParser = new ConnectStringParser(
                connectString);
//封装连接信息
        HostProvider hostProvider = new StaticHostProvider(
                connectStringParser.getServerAddresses());
//创建连接上下文
        cnxn = new ClientCnxn(connectStringParser.getChrootPath(),
                hostProvider, sessionTimeout, this, watchManager,
                getClientCnxnSocket(), canBeReadOnly);
        cnxn.start();
    }
    // 启动启动
    public void start() {
        // 连接服务器的线程
        sendThread.start();
        eventThread.start();
    }
```
创建连接上下文的时候，先反射创建ClientCnxnSocketNIO类的实例，保存到ClientCnxn的实例对象中，在构造ClientCnxn的实例对象时创建了两个线程，连接服务器的线程就是这个时候创建的，看下在这个线程中怎么连接的吧。这个方法太长了，先按照执行顺序进行分析吧。
1. 启动的时候发现没有建立连接`!clientCnxnSocket.isConnected()`，于是开始连接`startConnect(serverAddress);`，
```java
    @Override
    void connect(InetSocketAddress addr) throws IOException {
        // 创建socket套接字
        SocketChannel sock = createSock();
        try {
           registerAndConnect(sock, addr);
        } catch (IOException e) {
            LOG.error("Unable to open socket to " + addr);
            sock.close();
            throw e;
        }
        initialized = false;

        /*
         * Reset incomingBuffer
         */
        lenBuffer.clear();
        incomingBuffer = lenBuffer;
    }

    void registerAndConnect(SocketChannel sock, InetSocketAddress addr) throws IOException {
        // 设置sockKey为OP_CONNECT
        sockKey = sock.register(selector, SelectionKey.OP_CONNECT);
        // 连接服务器
        boolean immediateConnect = sock.connect(addr);
        if (immediateConnect) {
            sendThread.primeConnection();
        }
    }
```
2. 注册了OP_CONNECT以后，就要对这个OP_CONNECT事件进行处理，先`sc.finishConnect()`，再发个空包。
```java
    void doTransport(int waitTimeOut, List<Packet> pendingQueue, LinkedList<Packet> outgoingQueue,
                     ClientCnxn cnxn)
            throws IOException, InterruptedException {
        selector.select(waitTimeOut);
        Set<SelectionKey> selected;
        synchronized (this) {
            selected = selector.selectedKeys();
        }
        // Everything below and until we get back to the select is
        // non blocking, so time is effectively a constant. That is
        // Why we just have to do this once, here
        updateNow();
        for (SelectionKey k : selected) {
            SocketChannel sc = ((SocketChannel) k.channel());
            if ((k.readyOps() & SelectionKey.OP_CONNECT) != 0) {
                if (sc.finishConnect()) {
                    updateLastSendAndHeard();
                    sendThread.primeConnection();
                }
            } else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                doIO(pendingQueue, outgoingQueue, cnxn);
            }
        }
        if (sendThread.getZkState().isConnected()) {
            synchronized(outgoingQueue) {
                if (findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                    enableWrite();
                }
            }
        }
        selected.clear();
    }
```
连接到服务器的时候会向outgoingQueue()`private final LinkedList<Packet> outgoingQueue = new LinkedList<Packet>();`队列中添加个空包，同时设置socket为可写的。再来看什么时候从队列中获取空包然后发送给服务器的。
```java
        void primeConnection() throws IOException {
        ......  
                            outgoingQueue.addFirst(new Packet(null, null, conReq,null, null, readOnly));
                            clientCnxnSocket.enableReadWriteOnly();
        ......        
        }
```
在ClientCnxnSocketNIO.java#doIO()方法中，发现socket可写即可以向服务器发送数据时，就会到队列中取出刚刚加入的Packet。
```java
// ClientCnxnSocketNIO.java#doIO()
        if (sockKey.isWritable()) {
            synchronized(outgoingQueue) {
                Packet p = findSendablePacket(outgoingQueue,
                        cnxn.sendThread.clientTunneledAuthenticationInProgress());

                if (p != null) {
                    updateLastSend();
                    // If we already started writing p, p.bb will already exist
                    if (p.bb == null) {
                        if ((p.requestHeader != null) &&
                                (p.requestHeader.getType() != OpCode.ping) &&
                                (p.requestHeader.getType() != OpCode.auth)) {
                            p.requestHeader.setXid(cnxn.getXid());
                        }
                        p.createBB();
                    }
                    // 将ByteBuffer写道SocketChannel
                    sock.write(p.bb);
                    // 如果这个包的数据都写到SocketChannel，就将这个包加到pendingQueue中
                    if (!p.bb.hasRemaining()) {
                        sentCount++;
                        outgoingQueue.removeFirstOccurrence(p);
                        if (p.requestHeader != null
                                && p.requestHeader.getType() != OpCode.ping
                                && p.requestHeader.getType() != OpCode.auth) {
                            synchronized (pendingQueue) {
                                pendingQueue.add(p);
                            }
                        }
                    }
                }
                
            }
        }
```
取出Packet的方法是findSendablePacket()。连接包是没有requestHeader的，根据这个从队列中取出连接包Packet。
```java
    private Packet findSendablePacket(LinkedList<Packet> outgoingQueue,
                                      boolean clientTunneledAuthenticationInProgress) {
        synchronized (outgoingQueue) {
            if (outgoingQueue.isEmpty()) {
                return null;
            }
            // 这里的意思是如果队列中的第一个包在发送中，但是数据没发完，要取出来继续发
            if (outgoingQueue.getFirst().bb != null // If we've already starting sending the first packet, we better finish
                || !clientTunneledAuthenticationInProgress) {
                return outgoingQueue.getFirst();
            }

            ListIterator<Packet> iter = outgoingQueue.listIterator();
            while (iter.hasNext()) {
                Packet p = iter.next();
                if (p.requestHeader == null) {
                    // 找到了连接包Packet
                    iter.remove();
                    outgoingQueue.add(0, p);
                    return p;
                } else {
                    // Non-priming packet: defer it until later, leaving it in the queue
                    // until authentication completes.
                    if (LOG.isDebugEnabled()) {
                        LOG.debug("deferring non-priming packet: " + p +
                                "until SASL authentication completes.");
                    }
                }
            }
            // no sendable packet found.
            return null;
        }
    }
```
现在我们已经取出这个发送给服务器了，再来看客户端怎么接收来自服务器的报文。
```java
// ClientCnxnSocketNIO.java#doIO()
        if (sockKey.isReadable()) {
            // 从socket中读数据到incomingBuffer(ByteBuffer)
            int rc = sock.read(incomingBuffer);
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from server sessionid 0x"
                                + Long.toHexString(sessionId)
                                + ", likely server has closed socket");
            }
            // 有数据的话进行处理
            if (!incomingBuffer.hasRemaining()) {
                incomingBuffer.flip();
                if (incomingBuffer == lenBuffer) {
                    recvCount++;
                    readLength();
                } else if (!initialized) {
                    // 我们的连接包就是在这里被处理的
                    readConnectResult();
                    enableRead();
                    if (findSendablePacket(outgoingQueue,
                            cnxn.sendThread.clientTunneledAuthenticationInProgress()) != null) {
                        // Since SASL authentication has completed (if client is configured to do so),
                        // outgoing packets waiting in the outgoingQueue can now be sent.
                        enableWrite();
                    }
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                    initialized = true;
                } else {
                    sendThread.readResponse(incomingBuffer);
                    lenBuffer.clear();
                    incomingBuffer = lenBuffer;
                    updateLastHeard();
                }
            }
        }
```
如果能正确处理报文，则设置state的值为已连接。`state = (isRO) ?States.CONNECTEDREADONLY : States.CONNECTED;`，再看看zk的失败重连机制。如果不是第一次连接则延时1s再次startConnect()。
```java
                    if (!clientCnxnSocket.isConnected()) {
                        if(!isFirstConnect){
                            try {
                                Thread.sleep(r.nextInt(1000));
                            } catch (InterruptedException e) {
                                LOG.warn("Unexpected exception", e);
                            }
                        }
                        // don't re-establish connection if we are closing
                        if (closing || !state.isAlive()) {
                            break;
                        }
                        if (rwServerAddress != null) {
                            serverAddress = rwServerAddress;
                            rwServerAddress = null;
                        } else {
                            serverAddress = hostProvider.next(1000);
                        }
                        startConnect(serverAddress);
                        clientCnxnSocket.updateLastSendAndHeard();
                    }
```
**总结下，zk是通过nio的方式连接服务器的，创建socket时设置连接事件，再添加个空包到队列中，然后发给服务器，对服务器返回的报文进行处理后建立连接，之后就可以与服务器进行通信啦。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**