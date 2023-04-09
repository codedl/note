---
title: zookeeper服务端启动过程
tags:
notebook: zookeeper源码解析
---
现在看看zk服务端的启动过程，逻辑比较长，但不是很复杂，待会也能看到zk的代码在开发中还是值得借鉴的。所有的程序入口点都在main()，就从这里开始看起，go!  
首先对QuorumPeerMain类进行实例化，然后开始运行。
```java
    public static void main(String[] args) {
        QuorumPeerMain main = new QuorumPeerMain();
        try {
            main.initializeAndRun(args);
        } catch (IllegalArgumentException e) {
            LOG.error("Invalid arguments, exiting abnormally", e);
            LOG.info(USAGE);
            System.err.println(USAGE);
            System.exit(2);
        } catch (ConfigException e) {
            LOG.error("Invalid config, exiting abnormally", e);
            System.err.println("Invalid config, exiting abnormally");
            System.exit(2);
        } catch (Exception e) {
            LOG.error("Unexpected exception, exiting abnormally", e);
            System.exit(1);
        }
        LOG.info("Exiting normally");
        System.exit(0);
    }
```
### 初始化
还记得启动zk时传了配置文件路径作为参数吗，这里就是对参数的解析，解析完成后判断是否为集群，刚开始先从单机看起，所有走的是`ZooKeeperServerMain.main(args);`。
```java
    protected void initializeAndRun(String[] args)
        throws ConfigException, IOException
    {
        QuorumPeerConfig config = new QuorumPeerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        }

        // Start and schedule the the purge task
        DatadirCleanupManager purgeMgr = new DatadirCleanupManager(config
                .getDataDir(), config.getDataLogDir(), config
                .getSnapRetainCount(), config.getPurgeInterval());
        purgeMgr.start();

        if (args.length == 1 && config.servers.size() > 0) {
            runFromConfig(config);
        } else {
            LOG.warn("Either no config or no quorum defined in config, running "
                    + " in standalone mode");
            // there is only server in the quorum -- run as standalone
            ZooKeeperServerMain.main(args);
        }
    }
```
看看配置文件怎么被解析的吧，平时自己也可以试着写一些配置文件。根据路径生成File对象，将配置文件的配置项读到Properties中，然后zk会对这个Properties进行处理`parseProperties(cfg);`。
```java
    public void parse(String path) throws ConfigException {
        File configFile = new File(path);

        LOG.info("Reading configuration from: " + configFile);

        try {
            if (!configFile.exists()) {
                throw new IllegalArgumentException(configFile.toString()
                        + " file is missing");
            }

            Properties cfg = new Properties();
            FileInputStream in = new FileInputStream(configFile);
            try {
                cfg.load(in);
            } finally {
                in.close();
            }

            parseProperties(cfg);
        } catch (IOException e) {
            throw new ConfigException("Error processing " + path, e);
        } catch (IllegalArgumentException e) {
            throw new ConfigException("Error processing " + path, e);
        }
    }
```
这个方法就很长了，我就截取一部分关键的代码了。首先便利所有的属性，设置集群中每个节点的地址，并判断是否为observer模式，再根据端口号设置监听的IP地址，配置myid。
```java
// parseProperties
    for (Entry<Object, Object> entry : zkProp.entrySet()) {
        // 对所有属性进行遍历，依次获取key和value
        String key = entry.getKey().toString().trim();
        String value = entry.getValue().toString().trim();
        // 将获取到的key或现有的key对比，设置配置项到zk
        if (key.equals("dataDir")) {
            dataDir = value;
        } 
        ...
        // 如果是集群配置的话
        else if (key.startsWith("server.")) {
            // 小数点后面的数字作为sid
            int dot = key.indexOf('.');
            long sid = Long.parseLong(key.substring(dot + 1));
            String parts[] = splitWithLeadingHostname(value);
            if ((parts.length != 2) && (parts.length != 3) && (parts.length !=4)) {
                LOG.error(value
                    + " does not have the form host:port or host:port:port " +
                    " or host:port:port:type");
            }
            // 依次设置ip，数据端口，竞选端口
            LearnerType type = null;
            String hostname = parts[0];
            Integer port = Integer.parseInt(parts[1]);
            Integer electionPort = null;
            if (parts.length > 2){
                electionPort=Integer.parseInt(parts[2]);
            }
            // 判断是否为observer模式
            if (parts.length > 3){
                if (parts[3].toLowerCase().equals("observer")) {
                    type = LearnerType.OBSERVER;
                } else if (parts[3].toLowerCase().equals("participant")) {
                    type = LearnerType.PARTICIPANT;
                } else {
                    throw new ConfigException("Unrecognised peertype: " + value);
                }
            }
            // 保存当前的服务启动模式
            if (type == LearnerType.OBSERVER){
                observers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
            } else {
                servers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
            }
        }
        // 根据clientPort设置服务端启动时监听的地址
        if (clientPortAddress != null) {
            this.clientPortAddress = new InetSocketAddress(
                    InetAddress.getByName(clientPortAddress), clientPort);
        } else {
            this.clientPortAddress = new InetSocketAddress(clientPort);
        }
        // 读取服务对应的id，保存下来。
        File myIdFile = new File(dataDir, "myid");
        if (!myIdFile.exists()) {
            throw new IllegalArgumentException(myIdFile.toString()
                    + " file is missing");
        }
        BufferedReader br = new BufferedReader(new FileReader(myIdFile));
        String myIdString;
        try {
            myIdString = br.readLine();
        } finally {
            br.close();
        }
        try {
            serverId = Long.parseLong(myIdString);
            MDC.put("myid", myIdString);
        } catch (NumberFormatException e) {
            throw new IllegalArgumentException("serverid " + myIdString
                    + " is not a number");
        }                
    }
```
因为咱们不是单机启动，所以进入ZooKeeperServerMain的initializeAndRun()方法，解析配置文件的过程和上面一样，直接看启动的方法`runFromConfig(config);`。
```java
// ZooKeeperServerMain.java
    protected void initializeAndRun(String[] args) throws ConfigException, IOException
    {
        try {
            ManagedUtil.registerLog4jMBeans();
        } catch (JMException e) {
            LOG.warn("Unable to register log4j JMX control", e);
        }

        ServerConfig config = new ServerConfig();
        if (args.length == 1) {
            config.parse(args[0]);
        } else {
            config.parse(args);
        }

        runFromConfig(config);
    }
```
这是比较关键的方法了，启动的逻辑都在这里。先对ZooKeeperServer类进行实例化，创建快照文件处理对象FileTxnSnapLog，通过nio监听来自客户端的连接，再启动zk服务器。
```java
    public void runFromConfig(ServerConfig config) throws IOException {
        LOG.info("Starting server");
        FileTxnSnapLog txnLog = null;
        try {
            // Note that this thread isn't going to be doing anything else,
            // so rather than spawning another thread, we will just call
            // run() in this thread.
            // create a file logger url from the command line args
            // zk服务器的实例话
            final ZooKeeperServer zkServer = new ZooKeeperServer();
            // Registers shutdown handler which will be used to know the
            // server error or shutdown state changes.
            final CountDownLatch shutdownLatch = new CountDownLatch(1);
            zkServer.registerServerShutdownHandler(
                    new ZooKeeperServerShutdownHandler(shutdownLatch));
            // FileTxnSnapLog是用来处理日志和数据的
            txnLog = new FileTxnSnapLog(new File(config.dataLogDir), new File(
                    config.dataDir));
            txnLog.setServerStats(zkServer.serverStats());
            zkServer.setTxnLogFactory(txnLog);
            zkServer.setTickTime(config.tickTime);
            zkServer.setMinSessionTimeout(config.minSessionTimeout);
            zkServer.setMaxSessionTimeout(config.maxSessionTimeout);
            // 创建socket监听客户端连接
            cnxnFactory = ServerCnxnFactory.createFactory();
            // 打开socket
            cnxnFactory.configure(config.getClientPortAddress(),
                    config.getMaxClientCnxns());
            // 启动zk服务
            cnxnFactory.startup(zkServer);
            // Watch status of ZooKeeper server. It will do a graceful shutdown
            // if the server is not running or hits an internal error.
            shutdownLatch.await();
            shutdown();

            cnxnFactory.join();
            if (zkServer.canShutdown()) {
                zkServer.shutdown(true);
            }
        } catch (InterruptedException e) {
            // warn, but generally this is ok
            LOG.warn("Server interrupted", e);
        } finally {
            if (txnLog != null) {
                txnLog.close();
            }
        }
    }
```
1. 创建socket句柄，这里能看到ServerCnxnFactory是由NIOServerCnxnFactory实现的。
```java
    static public ServerCnxnFactory createFactory() throws IOException {
        String serverCnxnFactoryName =
            System.getProperty(ZOOKEEPER_SERVER_CNXN_FACTORY);
        if (serverCnxnFactoryName == null) {
            serverCnxnFactoryName = NIOServerCnxnFactory.class.getName();
        }
        try {
            ServerCnxnFactory serverCnxnFactory = (ServerCnxnFactory) Class.forName(serverCnxnFactoryName)
                    .getDeclaredConstructor().newInstance();
            LOG.info("Using {} as server connection factory", serverCnxnFactoryName);
            return serverCnxnFactory;
        } catch (Exception e) {
            IOException ioe = new IOException("Couldn't instantiate "
                    + serverCnxnFactoryName);
            ioe.initCause(e);
            throw ioe;
        }
    }
```
2. 将当前对象NIOServerCnxnFactory作为线程保存下来，这里就是nio编程框架了，打开ServerSocketChannel，绑定ip地址，设置OP_ACCEPT等待连接事件，监听客户端连接请求。
```java
    public void configure(InetSocketAddress addr, int maxcc) throws IOException {
        configureSaslLogin();

        thread = new ZooKeeperThread(this, "NIOServerCxn.Factory:" + addr);
        thread.setDaemon(true);
        maxClientCnxns = maxcc;
        this.ss = ServerSocketChannel.open();
        ss.socket().setReuseAddress(true);
        LOG.info("binding to port " + addr);
        ss.socket().bind(addr);
        ss.configureBlocking(false);
        ss.register(selector, SelectionKey.OP_ACCEPT);
    }
```
接下来就是zookeeper服务启动了，首先创建NIOServerCnxnFactory线程，再从快照文件中加载数据，最后创建请求处理器链。
```java
    public void startup(ZooKeeperServer zks) throws IOException,InterruptedException {
        // 启动NIOServerCnxnFactory线程
        start();
        setZooKeeperServer(zks);
        // 从快照文件中加载数据
        zks.startdata();
        // 创建请求处理器链
        zks.startup();
    }
```
### socket数据处理
这个线程主要做的事情就是监听客户端的连接请求，与客户端进行数据交互，核心方法是doIO()。
```java
// NIOServerCnxnFactory
    public void run() {
        while (!ss.socket().isClosed()) {
            try {
                selector.select(1000);
                Set<SelectionKey> selected;
                synchronized (this) {
                    // 获取所有监听的key
                    selected = selector.selectedKeys();
                }
                ArrayList<SelectionKey> selectedList = new ArrayList<SelectionKey>(
                        selected);
                Collections.shuffle(selectedList);
                for (SelectionKey k : selectedList) {
                    // 处理客户端发送过来的连接请求
                    if ((k.readyOps() & SelectionKey.OP_ACCEPT) != 0) {
                        SocketChannel sc = ((ServerSocketChannel) k
                                .channel()).accept();
                        InetAddress ia = sc.socket().getInetAddress();
                        int cnxncount = getClientCnxnCount(ia);
                        // 不能超过最大连接数，默认60
                        if (maxClientCnxns > 0 && cnxncount >= maxClientCnxns){
                            LOG.warn("Too many connections from " + ia
                                     + " - max is " + maxClientCnxns );
                            sc.close();
                        } else {
                            // 创建连接，将socket注册到selector，监听读事件
                            LOG.info("Accepted socket connection from "
                                     + sc.socket().getRemoteSocketAddress());
                            sc.configureBlocking(false);
                            // 注册读事件，接收客户端发过来的数据
                            SelectionKey sk = sc.register(selector,
                                    SelectionKey.OP_READ);
                            NIOServerCnxn cnxn = createConnection(sc, sk);
                            // 将创建的NIOServerCnxn对象保存到SelectionKey上面，NIOServerCnxn对象持有连接的SocketChannel
                            // 可以通过NIOServerCnxn对象的SocketChannel想指定的客户端返回数据
                            sk.attach(cnxn);
                            // 保存连接
                            addCnxn(cnxn);
                        }
                    } 
                    // 与客户端进行io数据交换
                    else if ((k.readyOps() & (SelectionKey.OP_READ | SelectionKey.OP_WRITE)) != 0) {
                        // 获取客户端与服务端建立连接时绑定的NIOServerCnxn
                        // NIOServerCnxn中SocketChannel写数据就可以返回给发送数据的客户端
                        NIOServerCnxn c = (NIOServerCnxn) k.attachment();
                        c.doIO(k);
                    } else {
                        if (LOG.isDebugEnabled()) {
                            LOG.debug("Unexpected ops in select "
                                      + k.readyOps());
                        }
                    }
                }
                selected.clear();
            } catch (RuntimeException e) {
                LOG.warn("Ignoring unexpected runtime exception", e);
            } catch (Exception e) {
                LOG.warn("Ignoring exception", e);
            }
        }
        closeAll();
        LOG.info("NIOServerCnxn factory exited run method");
    }
```
这里只看读事件，即客户端向服务端发送数据的socket处理，处理读取的数据是在readPayload()方法中。
```java
    void doIO(SelectionKey k) throws InterruptedException {
        try {
            if (isSocketOpen() == false) {
                LOG.warn("trying to do i/o on a null socket for session:0x"
                         + Long.toHexString(sessionId));

                return;
            }
            if (k.isReadable()) {
                // 从socket中读取数据
                int rc = sock.read(incomingBuffer);
                if (rc < 0) {
                    throw new EndOfStreamException(
                            "Unable to read additional data from client sessionid 0x"
                            + Long.toHexString(sessionId)
                            + ", likely client has closed socket");
                }
                if (incomingBuffer.remaining() == 0) {
                    boolean isPayload;
                    if (incomingBuffer == lenBuffer) { // start of next request
                        incomingBuffer.flip();
                        isPayload = readLength(k);
                        incomingBuffer.clear();
                    } else {
                        // continuation
                        isPayload = true;
                    }
                    // 处理读取到的数据
                    if (isPayload) { // not the case for 4letterword
                        readPayload();
                    }
                    else {
                        // four letter words take care
                        // need not do anything else
                        return;
                    }
                }
            
            }
        ......
    }
```
这里就是初步的处理客户端的数据请求，在processPacket()中对客户端的数据字节流进行反序列化，得到RequestHeader，再根据RequestHeader的类型进行不同的处理。如果是普通的请求，像创建一个节点，就生成Request，设置自身为所有者`si.setOwner(ServerCnxn.me);`，再提交请求，由firstProcessor负责处理`firstProcessor.processRequest(si);`，接下来就是请求处理器链的逻辑啦，从这里也能看到nio编程的思想就是，将数据的收发与处理分开，这样的话可以极大的提高数据的吞吐和处理效率，因为处理数据是在另外一个线程中的。
```java
    private void readPayload() throws IOException, InterruptedException {
        if (incomingBuffer.remaining() != 0) { // have we read length bytes?
            int rc = sock.read(incomingBuffer); // sock is non-blocking, so ok
            if (rc < 0) {
                throw new EndOfStreamException(
                        "Unable to read additional data from client sessionid 0x"
                        + Long.toHexString(sessionId)
                        + ", likely client has closed socket");
            }
        }
        if (incomingBuffer.remaining() == 0) { // have we read length bytes?
            packetReceived();
            incomingBuffer.flip();
            // 读取连接请求，处理连接，创建会话
            if (!initialized) {
                readConnectRequest();
            } else {
                // 读取请求
                readRequest();
            }
            lenBuffer.clear();
            incomingBuffer = lenBuffer;
        }
    }
```
### 从快照文件中加载数据
加载数据的本质是FileTxnSnapLog.java类中的restore()方法，先反序列化快照文件。
```java
    public long restore(DataTree dt, Map<Long, Integer> sessions, 
            PlayBackListener listener) throws IOException {
        snapLog.deserialize(dt, sessions);
        return fastForwardFromEdits(dt, sessions, listener);
    }
```
反序列化的逻辑也很清晰，就是获取snapDir配置的路径下所有的快照文件，然后对文件字节流进行反序列化，再加载到DataTree中。之后就是获取事务文件，找到比快照文件zxid大的事务文件，然后依次处理
```java
    public long deserialize(DataTree dt, Map<Long, Integer> sessions)
            throws IOException {
        // we run through 100 snapshots (not all of them)
        // if we cannot get it running within 100 snapshots
        // we should  give up
        List<File> snapList = findNValidSnapshots(100);
        if (snapList.size() == 0) {
            return -1L;
        }
        File snap = null;
        boolean foundValid = false;
        for (int i = 0; i < snapList.size(); i++) {
            snap = snapList.get(i);
            InputStream snapIS = null;
            CheckedInputStream crcIn = null;
            try {
                LOG.info("Reading snapshot " + snap);
                snapIS = new BufferedInputStream(new FileInputStream(snap));
                crcIn = new CheckedInputStream(snapIS, new Adler32());
                InputArchive ia = BinaryInputArchive.getArchive(crcIn);
                deserialize(dt,sessions, ia);
                long checkSum = crcIn.getChecksum().getValue();
                long val = ia.readLong("val");
                if (val != checkSum) {
                    throw new IOException("CRC corruption in snapshot :  " + snap);
                }
                foundValid = true;
                break;
            } catch(IOException e) {
                LOG.warn("problem reading snap file " + snap, e);
            } finally {
                if (snapIS != null) 
                    snapIS.close();
                if (crcIn != null) 
                    crcIn.close();
            } 
        }
        if (!foundValid) {
            throw new IOException("Not able to find valid snapshots in " + snapDir);
        }
        dt.lastProcessedZxid = Util.getZxidFromName(snap.getName(), SNAPSHOT_FILE_PREFIX);
        return dt.lastProcessedZxid;
    }
```
获取配置路径下所有的事物文件，然后找到比lastProcessedZxid大的事务文件，再依次处理。
```java
    public long fastForwardFromEdits(DataTree dt, Map<Long, Integer> sessions,
                                     PlayBackListener listener) throws IOException {
        FileTxnLog txnLog = new FileTxnLog(dataDir);
        // 找到比lastProcessedZxid大的事务文件
        TxnIterator itr = txnLog.read(dt.lastProcessedZxid+1);
        long highestZxid = dt.lastProcessedZxid;
        TxnHeader hdr;
        try {
            while (true) {
                // iterator points to 
                // the first valid txn when initialized
                hdr = itr.getHeader();
                if (hdr == null) {
                    //empty logs 
                    return dt.lastProcessedZxid;
                }
                if (hdr.getZxid() < highestZxid && highestZxid != 0) {
                    LOG.error("{}(higestZxid) > {}(next log) for type {}",
                            new Object[] { highestZxid, hdr.getZxid(),
                                    hdr.getType() });
                } else {
                    highestZxid = hdr.getZxid();
                }
                try {
                    processTransaction(hdr,dt,sessions, itr.getTxn());
                } catch(KeeperException.NoNodeException e) {
                   throw new IOException("Failed to process transaction type: " +
                         hdr.getType() + " error: " + e.getMessage(), e);
                }
                listener.onTxnLoaded(hdr, itr.getTxn());
                if (!itr.next()) 
                    break;
            }
        } finally {
            if (itr != null) {
                itr.close();
            }
        }
        return highestZxid;
    }
```
请求处理器链对客户端请求的处理我们下篇再说，因为这个篇幅太长了。
**总结一下，zk服务端启动是先加载配置文件，然后启动线程监听客户端连接，与客户端连接后进行数据传输，再就是读取快照文件和事务文件中的数据到内存中。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**