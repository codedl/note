---
title: zookeeper客户端指令执行过程
tags:
notebook: zookeeper源码解析
---
运行zookeeper客户端的启动脚本可以向服务器发起连接，执行zk指令创建节点，就从这里开始看起，可以看到程序main方法在`org.apache.zookeeper.ZooKeeperMain`类中。
```
ZOOBIN="${BASH_SOURCE-$0}"
ZOOBIN="$(dirname "${ZOOBIN}")"
ZOOBINDIR="$(cd "${ZOOBIN}"; pwd)"

if [ -e "$ZOOBIN/../libexec/zkEnv.sh" ]; then
  . "$ZOOBINDIR"/../libexec/zkEnv.sh
else
  . "$ZOOBINDIR"/zkEnv.sh
fi

"$JAVA" "-Dzookeeper.log.dir=${ZOO_LOG_DIR}" "-Dzookeeper.root.logger=${ZOO_LOG4J_PROP}" \
     -cp "$CLASSPATH" $CLIENT_JVMFLAGS $JVMFLAGS \
     org.apache.zookeeper.ZooKeeperMain "$@"
```
在ZooKeeperMain类的main中，先生成ZooKeeperMain类的实例对象，在运行run()方法。
```java
// ZooKeeperMain.java
    public static void main(String args[]) throws KeeperException, IOException, InterruptedException
    {
        ZooKeeperMain main = new ZooKeeperMain(args);
        main.run();
    }
```
在构造ZooKeeperMain实例时，先解析main方法的参数，再连接到zk上。
```java
    public ZooKeeperMain(String args[]) throws IOException, InterruptedException {
        cl.parseOptions(args);
        System.out.println("Connecting to " + cl.getOption("server"));
        connectToZK(cl.getOption("server"));
    }
```
连接到zk服务器实际上就是创建ZooKeeper的实例对象，这个过程已经分析过啦。
```java
    protected void connectToZK(String newHost) throws InterruptedException, IOException {
        if (zk != null && zk.getState().isAlive()) {
            zk.close();
        }
        host = newHost;
        boolean readOnly = cl.getOption("readonly") != null;
        zk = new ZooKeeper(host,
                 Integer.parseInt(cl.getOption("timeout")),
                 new MyWatcher(), readOnly);
    }
```
再看下run方法吧，如果启动zk客户端时输入了相关指令，则执行这个指令，没有的话就循环等待用户的输入。
```java
   void run() throws KeeperException, IOException, InterruptedException {
        if (cl.getCommand() == null) {
            System.out.println("Welcome to ZooKeeper!");

            boolean jlinemissing = false;
            // only use jline if it's in the classpath
            try {
                // idea无法输入我就用Java的Scanner改写了，逻辑还是一样的，都是从console中读取一行
                String line;
                Scanner sc = new Scanner(System.in);
                while ((line = sc.nextLine()) != null) {
                    executeLine(line);
                }

                Class<?> consoleC = Class.forName("jline.ConsoleReader");
                Class<?> completorC =
                    Class.forName("org.apache.zookeeper.JLineZNodeCompletor");

                System.out.println("JLine support is enabled");

                Object console =
                    consoleC.getConstructor().newInstance();

                Object completor =
                    completorC.getConstructor(ZooKeeper.class).newInstance(zk);
                Method addCompletor = consoleC.getMethod("addCompletor",
                        Class.forName("jline.Completor"));
                addCompletor.invoke(console, completor);
                Method readLine = consoleC.getMethod("readLine", String.class);

                while ((line = (String)readLine.invoke(console, getPrompt())) != null) {
                    executeLine(line);
                }
            } catch (ClassNotFoundException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (NoSuchMethodException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (InvocationTargetException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (IllegalAccessException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            } catch (InstantiationException e) {
                LOG.debug("Unable to start jline", e);
                jlinemissing = true;
            }

            if (jlinemissing) {
                System.out.println("JLine support is disabled");
                BufferedReader br =
                    new BufferedReader(new InputStreamReader(System.in));

                String line;
                while ((line = br.readLine()) != null) {
                    executeLine(line);
                }
            }
        } else {
            // Command line args non-null.  Run what was passed.
            processCmd(cl);
        }
    }
```
当读取到用户的输入时，就会对用户输入的指令进行解析，解析的过程在executeLine()方法中，先从console中获取用户的输入，解析成指令，在进行处理`processCmd(cl)`。
```java
    public void executeLine(String line)
    throws InterruptedException, IOException, KeeperException {
      if (!line.equals("")) {
        cl.parseCommand(line);
        addToHistory(commandCount,line);
        processCmd(cl);
        commandCount++;
      }
    }
```
接着看怎么处理的，处理的方法是`processZKCmd()`，这个方法很长，我截取了create指令对应的部分代码。默认创建模式为PERSISTENT，可以根据"-e"或"-s"来确定，最后调用ZooKeeper类中的方法`public String create(final String path, byte data[], List<ACL> acl,
CreateMode createMode)`。
```java
// ZooKeeperMain.java#processZKCmd
        if (cmd.equals("create") && args.length >= 3) {
            int first = 0;
            CreateMode flags = CreateMode.PERSISTENT;
            if ((args[1].equals("-e") && args[2].equals("-s"))
                    || (args[1]).equals("-s") && (args[2].equals("-e"))) {
                first+=2;
                flags = CreateMode.EPHEMERAL_SEQUENTIAL;
            } else if (args[1].equals("-e")) {
                first++;
                flags = CreateMode.EPHEMERAL;
            } else if (args[1].equals("-s")) {
                first++;
                flags = CreateMode.PERSISTENT_SEQUENTIAL;
            }
            if (args.length == first + 4) {
                acl = parseACLs(args[first+3]);
            }
            path = args[first + 1];
            String newPath = zk.create(path, args[first+2].getBytes(), acl, flags);
            System.err.println("Created " + newPath);
        }
```
在create()方法中，先创建RequestHeader请求头，设置类型为`ZooDefs.OpCode.create`，再通过ClientCnxn.java类中的submitRequest()法方法提交请求`public ReplyHeader submitRequest(RequestHeader h, Record request, Record response, WatchRegistration watchRegistration)`

```java
    public String create(final String path, byte data[], List<ACL> acl,
            CreateMode createMode)
        throws KeeperException, InterruptedException
    {
        final String clientPath = path;
        PathUtils.validatePath(clientPath, createMode.isSequential());

        final String serverPath = prependChroot(clientPath);

        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.create);
        CreateRequest request = new CreateRequest();
        CreateResponse response = new CreateResponse();
        request.setData(data);
        request.setFlags(createMode.toFlag());
        request.setPath(serverPath);
        if (acl != null && acl.size() == 0) {
            throw new KeeperException.InvalidACLException();
        }
        request.setAcl(acl);
        ReplyHeader r = cnxn.submitRequest(h, request, response, null);
        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
        if (cnxn.chrootPath == null) {
            return response.getPath();
        } else {
            return response.getPath().substring(cnxn.chrootPath.length());
        }
    }
```
再来怎么提交请求，这里请求来了并不是马上就发给服务端，而是先加入到队列中，再同步地发送过去，可以解决异步问题。逻辑就是生成发送到服务器的Packet包，再一直阻塞`packet.wait()`直到服务器对这个包进行了响应。
```java
    public ReplyHeader submitRequest(RequestHeader h, Record request,
            Record response, WatchRegistration watchRegistration)
            throws InterruptedException {
        ReplyHeader r = new ReplyHeader();
        Packet packet = queuePacket(h, r, request, response, null, null, null,
                    null, watchRegistration);
        synchronized (packet) {
            // 阻塞，直到服务器响应，即packet的notify()方法被调用
            while (!packet.finished) {
                packet.wait();
            }
        }
        return r;
    }

    Packet queuePacket(RequestHeader h, ReplyHeader r, Record request,
            Record response, AsyncCallback cb, String clientPath,
            String serverPath, Object ctx, WatchRegistration watchRegistration)
    {
        Packet packet = null;

        // Note that we do not generate the Xid for the packet yet. It is
        // generated later at send-time, by an implementation of ClientCnxnSocket::doIO(),
        // where the packet is actually sent.
        synchronized (outgoingQueue) {
            packet = new Packet(h, r, request, response, watchRegistration);
            packet.cb = cb;
            packet.ctx = ctx;
            packet.clientPath = clientPath;
            packet.serverPath = serverPath;
            if (!state.isAlive() || closing) {
                conLossPacket(packet);
            } else {
                // If the client is asking to close the session then
                // mark as closing
                if (h.getType() == OpCode.closeSession) {
                    closing = true;
                }
                outgoingQueue.add(packet);
            }
        }
        sendThread.getClientCnxnSocket().wakeupCnxn();
        return packet;
    }
```
包到队列中然后再发送给服务器的过程之前讲过了，现在看怎么处理来自服务器的响应，处理逻辑在ClientCnxn$SendThread#readResponse()方法中。对来自服务器的字节流进行反序列化，然后获取队列中的Packet，设置xid,zxid，最后在finishPacket()中调用notifyAll()，解除之前的阻塞，客户端就能拿到来自服务器的响应啦。
```java
// ClientCnxn$SendThread#readResponse()
        void readResponse(ByteBuffer incomingBuffer) throws IOException {
            ByteBufferInputStream bbis = new ByteBufferInputStream(
                    incomingBuffer);
            BinaryInputArchive bbia = BinaryInputArchive.getArchive(bbis);
            ReplyHeader replyHdr = new ReplyHeader();

            replyHdr.deserialize(bbia, "header");
            if (replyHdr.getXid() == -2) {
                // -2 is the xid for pings
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Got ping response for sessionid: 0x"
                            + Long.toHexString(sessionId)
                            + " after "
                            + ((System.nanoTime() - lastPingSentNs) / 1000000)
                            + "ms");
                }
                return;
            }
            if (replyHdr.getXid() == -4) {
                // -4 is the xid for AuthPacket               
                if(replyHdr.getErr() == KeeperException.Code.AUTHFAILED.intValue()) {
                    state = States.AUTH_FAILED;                    
                    eventThread.queueEvent( new WatchedEvent(Watcher.Event.EventType.None, 
                            Watcher.Event.KeeperState.AuthFailed, null) );            		            		
                }
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Got auth sessionid:0x"
                            + Long.toHexString(sessionId));
                }
                return;
            }
            if (replyHdr.getXid() == -1) {
                // -1 means notification
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Got notification sessionid:0x"
                        + Long.toHexString(sessionId));
                }
                WatcherEvent event = new WatcherEvent();
                event.deserialize(bbia, "response");

                // convert from a server path to a client path
                if (chrootPath != null) {
                    String serverPath = event.getPath();
                    if(serverPath.compareTo(chrootPath)==0)
                        event.setPath("/");
                    else if (serverPath.length() > chrootPath.length())
                        event.setPath(serverPath.substring(chrootPath.length()));
                    else {
                    	LOG.warn("Got server path " + event.getPath()
                    			+ " which is too short for chroot path "
                    			+ chrootPath);
                    }
                }

                WatchedEvent we = new WatchedEvent(event);
                if (LOG.isDebugEnabled()) {
                    LOG.debug("Got " + we + " for sessionid 0x"
                            + Long.toHexString(sessionId));
                }

                eventThread.queueEvent( we );
                return;
            }

            // If SASL authentication is currently in progress, construct and
            // send a response packet immediately, rather than queuing a
            // response as with other packets.
            if (clientTunneledAuthenticationInProgress()) {
                GetSASLRequest request = new GetSASLRequest();
                request.deserialize(bbia,"token");
                zooKeeperSaslClient.respondToServer(request.getToken(),
                  ClientCnxn.this);
                return;
            }

            Packet packet;
            synchronized (pendingQueue) {
                if (pendingQueue.size() == 0) {
                    throw new IOException("Nothing in the queue, but got "
                            + replyHdr.getXid());
                }
                packet = pendingQueue.remove();
            }
            /*
             * Since requests are processed in order, we better get a response
             * to the first request!
             */
            try {
                if (packet.requestHeader.getXid() != replyHdr.getXid()) {
                    packet.replyHeader.setErr(
                            KeeperException.Code.CONNECTIONLOSS.intValue());
                    throw new IOException("Xid out of order. Got Xid "
                            + replyHdr.getXid() + " with err " +
                            + replyHdr.getErr() +
                            " expected Xid "
                            + packet.requestHeader.getXid()
                            + " for a packet with details: "
                            + packet );
                }

                packet.replyHeader.setXid(replyHdr.getXid());
                packet.replyHeader.setErr(replyHdr.getErr());
                packet.replyHeader.setZxid(replyHdr.getZxid());
                if (replyHdr.getZxid() > 0) {
                    lastZxid = replyHdr.getZxid();
                }
                if (packet.response != null && replyHdr.getErr() == 0) {
                    packet.response.deserialize(bbia, "response");
                }

                if (LOG.isDebugEnabled()) {
                    LOG.debug("Reading reply sessionid:0x"
                            + Long.toHexString(sessionId) + ", packet:: " + packet);
                }
            } finally {
                finishPacket(packet);
            }
        }
```
**总结一下，在客户端线程中会读取用户的命令行输入，然后解析成创建节点的指令，加入到队列中阻塞等待，在发送数据的线程从队列中将包取出发送给服务器，服务器执行了指令返回响应给客户端以后，停止阻塞。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**