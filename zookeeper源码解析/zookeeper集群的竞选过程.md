zookeeper集群竞选leader的流程是从`QuorumPeer#startLeaderElection()`开始，下面一步一步看。
## 初始化
先创建自身的选票，保存服务id，事务id，本机届数，然后获取竞选使用的端口，创建选举算法。
```java
    synchronized public void startLeaderElection() {
    	try {
            //myid是配置文件定义的，getLastLoggedZxid()=>事务id，getCurrentEpoch()=>当前届数
            //事务id和届数是方法返回的，每次集群需要选举时都会更新
    		currentVote = new Vote(myid, getLastLoggedZxid(), getCurrentEpoch());
    	} catch(IOException e) {
    		RuntimeException re = new RuntimeException(e.getMessage());
    		re.setStackTrace(e.getStackTrace());
    		throw re;
    	}
        //根据配置文件myid中的值和server.x中的x找到自身的节点配置信息，
        //即有myid指定自身是哪个节点
        for (QuorumServer p : getView().values()) {
            if (p.id == myid) {
                myQuorumAddr = p.addr;
                break;
            }
        }
        if (myQuorumAddr == null) {
            throw new RuntimeException("My id " + myid + " not in the peer list");
        }
        if (electionType == 0) {
            try {
                udpSocket = new DatagramSocket(myQuorumAddr.getPort());
                responder = new ResponderThread();
                responder.start();
            } catch (SocketException e) {
                throw new RuntimeException(e);
            }
        }
        this.electionAlg = createElectionAlgorithm(electionType);
    }
```
先实例化QuorumCnxManager,再启动Listener线程，再实例化FastLeaderElection。
```java
    protected Election createElectionAlgorithm(int electionAlgorithm){
        Election le=null;
......
        case 3:
            qcm = createCnxnManager();
            QuorumCnxManager.Listener listener = qcm.listener;
            if(listener != null){
                listener.start();
                le = new FastLeaderElection(this, qcm);
            } else {
                LOG.error("Null listener when initializing cnx manager");
            }
            break;
        default:
            assert false;
        }
        return le;
    }
```
`listener.start()`执行后就会启动Listener线程，看下Listener线程吧。在线程中对ip端口进行监听，再`accept()`等待其他节点客户端的连接，然后receiveConnection()处理连接。
```java
        public void run() {
            int numRetries = 0;
            InetSocketAddress addr;
            while((!shutdown) && (numRetries < 3)){
                try {
                    ss = new ServerSocket();
                    //允许端口复用，在设置监听端口号之前调用
                    ss.setReuseAddress(true);
                    if (listenOnAllIPs) {
                        int port = view.get(QuorumCnxManager.this.mySid)
                            .electionAddr.getPort();
                        addr = new InetSocketAddress(port);
                    } else {
                        //server.1=127.0.0.1:1111:1112
                        //server的IP地址:同步数据端口:竞选端口
                        addr = view.get(QuorumCnxManager.this.mySid)
                            .electionAddr;
                    }
                    LOG.info("My election bind port: " + addr.toString());
                    setName(view.get(QuorumCnxManager.this.mySid)
                            .electionAddr.toString());
                    ss.bind(addr);
                    while (!shutdown) {
                        //监听其他节点的连接
                        Socket client = ss.accept();
                        setSockOpts(client);
                        LOG.info("Received connection request "
                                + client.getRemoteSocketAddress());
                        
                        if (quorumSaslAuthEnabled) {
                            receiveConnectionAsync(client);
                        } else {
                            //处理连接
                            receiveConnection(client);
                        }

                        numRetries = 0;
                    }
                } catch (IOException e) {
                    ......
                }
            }
            ......
        }
```
建立连接时都是由大的sid连接小的sid对应的节点，防止重复连接。那这里的连接就有两种情况，首先讨论连接sid比自己小的节点，再讨论sid比自己大的节点连接自己。
```java
    private void handleConnection(Socket sock, DataInputStream din)
            throws IOException {
        Long sid = null;
        try {
            //读取server id，高32位为届数，低32位为事务id
            sid = din.readLong();
            if (sid < 0) { // this is not a server id but a protocol version (see ZOOKEEPER-1633)
                sid = din.readLong();

                // next comes the #bytes in the remainder of the message
                // note that 0 bytes is fine (old servers)
                int num_remaining_bytes = din.readInt();
                if (num_remaining_bytes < 0 || num_remaining_bytes > maxBuffer) {
                    LOG.error("Unreasonable buffer length: {}", num_remaining_bytes);
                    closeSocket(sock);
                    return;
                }
                //创建缓冲区接收发送过来的数据
                byte[] b = new byte[num_remaining_bytes];

                // remove the remainder of the message from din
                int num_read = din.read(b);
                if (num_read != num_remaining_bytes) {
                    LOG.error("Read only " + num_read + " bytes out of " + num_remaining_bytes + " sent by server " + sid);
                }
            }
            if (sid == QuorumPeer.OBSERVER_ID) {
                /*
                 * Choose identifier at random. We need a value to identify
                 * the connection.
                 */
                sid = observerCounter.getAndDecrement();
                LOG.info("Setting arbitrary identifier to observer: " + sid);
            }
        } catch (IOException e) {
            closeSocket(sock);
            LOG.warn("Exception reading or writing challenge: " + e.toString());
            return;
        }

        // do authenticating learner
        LOG.debug("Authenticating learner server.id: {}", sid);
        authServer.authenticate(sock, din);

        //If wins the challenge, then close the new connection.
        //连接时由sid大的节点连接小的节点，防止重复连接
        if (sid < this.mySid) {
           
            SendWorker sw = senderWorkerMap.get(sid);
            if (sw != null) {
                sw.finish();
            }

            LOG.debug("Create new connection to server: " + sid);
            closeSocket(sock);
            connectOne(sid);

            // Otherwise start worker threads to receive data.
        } else {
            //当发现sid大的节点连接我时，准备连接
            //逻辑跟connectOne()差不多，创建发送和接收数据的线程对象，启动线程
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);
            
            if(vsw != null)
                vsw.finish();
            
            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
            
            sw.start();
            rw.start();
            
            return;
        }
    }
```
## 连接到其他节点
首先判断连接存不存在，防止连接多次，然后调用connect()连接对方节点的ip，如果连接成功则处理连接的socket。
```java
    synchronized public void connectOne(long sid){
        //连接之前判断，到对方节点服务器的连接是否已经建立
        if (!connectedToPeer(sid)){
            //获取要连接的地址,sid为要连接的服务器节点id
            InetSocketAddress electionAddr;
            if (view.containsKey(sid)) {
                electionAddr = view.get(sid).electionAddr;
            } else {
                LOG.warn("Invalid server id: " + sid);
                return;
            }
            try {

                LOG.debug("Opening channel to server " + sid);
                Socket sock = new Socket();
                setSockOpts(sock);
                sock.connect(view.get(sid).electionAddr, cnxTO);
                LOG.debug("Connected to server " + sid);

                if (quorumSaslAuthEnabled) {
                    initiateConnectionAsync(sock, sid);
                } else {
                    //建立连接，创建用来接收和发送数据的线程对象
                    initiateConnection(sock, sid);
                }
            } catch (UnresolvedAddressException e) {
                ......
            } catch (IOException e) {
                ......
            }
        } else {
            LOG.debug("There is a connection already for server " + sid);
        }
    }
```
socket的`connect()`之后，就会发送本机节点的sid，然后再创建两个线程对象分别用于发送和接收数据。
```java
    private boolean startConnection(Socket sock, Long sid)
            throws IOException {
        DataOutputStream dout = null;
        DataInputStream din = null;
        try {
            // 发送本机节点的server id,届数+事务
            dout = new DataOutputStream(sock.getOutputStream());
            dout.writeLong(this.mySid);
            dout.flush();

            din = new DataInputStream(
                    new BufferedInputStream(sock.getInputStream()));
        } catch (IOException e) {
            LOG.warn("Ignoring exception reading or writing challenge: ", e);
            closeSocket(sock);
            return false;
        }

        // authenticate learner
        authLearner.authenticate(sock, view.get(sid).hostname);

        // If lost the challenge, then drop the new connection
        if (sid > this.mySid) {
            LOG.info("Have smaller server identifier, so dropping the " +
                     "connection: (" + sid + ", " + this.mySid + ")");
            closeSocket(sock);
            // Otherwise proceed with the connection
        } else {
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);
            //finish()会从senderWorkerMap中将sid对应的SendWorker移除
            if(vsw != null)
                vsw.finish();
            //将服务id对应的SendWorker对象保存下来
            senderWorkerMap.put(sid, sw);
            //putIfAbsent()会到map中检查，如果key对应的value存在则不更新
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
            
            sw.start();
            rw.start();
            
            return true;    
            
        }
        return false;
    }
```
### Listener中发送数据的线程SendWorker

发送数据的线程就是从queueSendMap中取出要发送的数据，在send()中进行发送，后面想发送数据只要把数据加入queueSendMap保存的队列中就行了。
```java
        public void run() {
            threadCnt.incrementAndGet();
            try {
                //获取sid对应的队列，优先发送queueSendMap中的数据，没有的话就到lastMessageSent里面找
                ArrayBlockingQueue<ByteBuffer> bq = queueSendMap.get(sid);
                if (bq == null || isSendQueueEmpty(bq)) {
                   ByteBuffer b = lastMessageSent.get(sid);
                   if (b != null) {
                       LOG.debug("Attempting to send lastMessage to sid=" + sid);
                       send(b);
                   }
                }
            } catch (IOException e) {
                LOG.error("Failed to send last message. Shutting down thread.", e);
                this.finish();
            }
            
            try {
                while (running && !shutdown && sock != null) {

                    ByteBuffer b = null;
                    try {
                        ArrayBlockingQueue<ByteBuffer> bq = queueSendMap
                                .get(sid);
                        if (bq != null) {
                            b = pollSendQueue(bq, 1000, TimeUnit.MILLISECONDS);
                        } else {
                            LOG.error("No queue of incoming messages for " +
                                      "server " + sid);
                            break;
                        }

                        if(b != null){
                            lastMessageSent.put(sid, b);
                            send(b);
                        }
                    } catch (InterruptedException e) {
                        LOG.warn("Interrupted while waiting for message on queue",
                                e);
                    }
                }
            } catch (Exception e) {
                LOG.warn("Exception when using channel: for id " + sid
                         + " my id = " + QuorumCnxManager.this.mySid
                         + " error = " + e);
            }
            this.finish();
            LOG.warn("Send worker leaving thread");
        }
    }
```
### Listener中接收数据的线程RecvWorker

接收到数据时，先读取发送的数据长度length，再读取length个字节的数据，然后封装成Message保存到recvQueue(ArrayBlockingQueue<Message>)
```java
    public void run() {
            threadCnt.incrementAndGet();
            try {
                while (running && !shutdown && sock != null) {
                    /**
                     * Reads the first int to determine the length of the
                     * message
                     */
                    //读取要接受的字节数length,再根据length读取接收到的数据
                    int length = din.readInt();
                    if (length <= 0 || length > PACKETMAXSIZE) {
                        throw new IOException(
                                "Received packet with invalid packet: "
                                        + length);
                    }
                    /**
                     * Allocates a new ByteBuffer to receive the message
                     */
                    byte[] msgArray = new byte[length];
                    din.readFully(msgArray, 0, length);
                    ByteBuffer message = ByteBuffer.wrap(msgArray);
                    addToRecvQueue(new Message(message.duplicate(), sid));
                }
            } catch (Exception e) {
                LOG.warn("Connection broken for id " + sid + ", my id = "
                         + QuorumCnxManager.this.mySid + ", error = " , e);
            } finally {
                LOG.warn("Interrupting SendWorker");
                sw.finish();
                if (sock != null) {
                    closeSocket(sock);
                }
            }
        }
    }
```
## 接受其他节点的连接
其他节点来连接我时，就好处理很多了，依然是创建发送和接收的线程对象，保存下来启动即可。
```java
        if (sid < this.mySid) {
           ......
        } else {
            //当发现sid大的节点连接我时，准备连接
            //逻辑跟connectOne()差不多，创建发送和接收数据的线程对象，启动线程
            SendWorker sw = new SendWorker(sock, sid);
            RecvWorker rw = new RecvWorker(sock, din, sid, sw);
            sw.setRecv(rw);

            SendWorker vsw = senderWorkerMap.get(sid);
            
            if(vsw != null)
                vsw.finish();
            
            senderWorkerMap.put(sid, sw);
            queueSendMap.putIfAbsent(sid, new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY));
            
            sw.start();
            rw.start();
            
            return;
        }
```
## 创建算法
到这里为止总算把Listener线程看完了，可见这是个处理socket连接进行通信的线程。接下来是对FastLeaderElection的实例化，这里就是创建两个队列分别用作接收和发送，那这里的两个队列跟前面的两个队列有啥不一样的呢，往下看才知道。
```java
    public FastLeaderElection(QuorumPeer self, QuorumCnxManager manager){
        this.stop = false;
        this.manager = manager;
        starter(self, manager);
    }
    private void starter(QuorumPeer self, QuorumCnxManager manager) {
        this.self = self;
        proposedLeader = -1;
        proposedZxid = -1;

        sendqueue = new LinkedBlockingQueue<ToSend>();
        recvqueue = new LinkedBlockingQueue<Notification>();
        this.messenger = new Messenger(manager);
    }
```
Messenger实例化的时候会创建发送和接收用的线程，分别看下这两个线程。
```java
        Messenger(QuorumCnxManager manager) {
            //发送线程
            this.ws = new WorkerSender(manager);

            Thread t = new Thread(this.ws,
                    "WorkerSender[myid=" + self.getId() + "]");
            t.setDaemon(true);
            t.start();
            //接收线程
            this.wr = new WorkerReceiver(manager);

            t = new Thread(this.wr,
                    "WorkerReceiver[myid=" + self.getId() + "]");
            t.setDaemon(true);
            t.start();
        }
```
### Messenger中发送数据的线程WorkerSender
首先从sendqueue中取出ToSend对象，根据ToSend对象的属性构建ByteBuffer，最终通过queueSendMap根据sid找到发送数据时保存数据的队列ArrayBlockingQueue<ByteBuffer>，然后将ByteBuffer加入到这个队列，此时就会进入上面Listener中发送数据的线程。那这里我们就能知道怎么回事，首先WorkerSender线程将ToSend对象从sendqueue队列取出来，转成ByteBuffer，然后加入到queueSendMap保存的队列中，在SendWorker线程会将ByteBuffer取出来，使用内部保存的socket发送出去。
```java
// FastLeaderElection$WorkerSender
            public void run() {
                while (!stop) {
                    try {
                        ToSend m = sendqueue.poll(3000, TimeUnit.MILLISECONDS);
                        if(m == null) continue;

                        process(m);
                    } catch (InterruptedException e) {
                        break;
                    }
                }
                LOG.info("WorkerSender is down");
            }
// FastLeaderElection$WorkerSender            
            void process(ToSend m) {
                ByteBuffer requestBuffer = buildMsg(m.state.ordinal(), 
                                                        m.leader,
                                                        m.zxid, 
                                                        m.electionEpoch, 
                                                        m.peerEpoch);
                manager.toSend(m.sid, requestBuffer);
            }
// QuorumCnxManager           
            public void toSend(Long sid, ByteBuffer b) {
                /*
                    * If sending message to myself, then simply enqueue it (loopback).
                    */
                //如果是发送给自己的就直接加入队列recvQueue(ArrayBlockingQueue<Message>)
                if (this.mySid == sid) {
                        b.position(0);
                        addToRecvQueue(new Message(b.duplicate(), sid));
                    /*
                        * Otherwise send to the corresponding thread to send.
                        */
                } else {
                        /*
                        * Start a new connection if doesn't have one already.
                        */
                        ArrayBlockingQueue<ByteBuffer> bq = new ArrayBlockingQueue<ByteBuffer>(SEND_CAPACITY);
                        ArrayBlockingQueue<ByteBuffer> bqExisting = queueSendMap.putIfAbsent(sid, bq);
                        if (bqExisting != null) {
                            addToSendQueue(bqExisting, b);
                        } else {
                            addToSendQueue(bq, b);
                        }
                        connectOne(sid);
                        
                }
            }
// QuorumCnxManager
    private void addToSendQueue(ArrayBlockingQueue<ByteBuffer> queue,
          ByteBuffer buffer) {
        //如果队列中没有数据容量了，就移除第一个
        if (queue.remainingCapacity() == 0) {
            try {
                queue.remove();
            } catch (NoSuchElementException ne) {
                // element could be removed by poll()
                LOG.debug("Trying to remove from an empty " +
                        "Queue. Ignoring exception " + ne);
            }
        }
        try {
            //加入队列
            queue.add(buffer);
        } catch (IllegalStateException ie) {
            // This should never happen
            LOG.error("Unable to insert an element in the queue " + ie);
        }
    }           
```
### Messenger中接收数据的线程WorkerReceiver
这里涉及到竞选的逻辑可以待会再细看，先留意接收数据的过程吧。首先是在RecvWorker线程接收到数据，把数据从字节流封装成Message保存到recvQueue(ArrayBlockingQueue<Message>)中，然后在WorkerReceiver线程中将数据取出来写到recvqueue。其实这样设计就可以把收发数据与处理数据分离，提高效率。
```java
        public void run() {

                Message response;
                while (!stop) {
                    // Sleeps on receive
                    try{
                        //取出收到的数据，这里的数据是经过封装的Message对象
                        response = manager.pollRecvQueue(3000, TimeUnit.MILLISECONDS);
                        if(response == null) continue;

                        if(!validVoter(response.sid)){
                            Vote current = self.getCurrentVote();
                            ToSend notmsg = new ToSend(ToSend.mType.notification,
                                    current.getId(),
                                    current.getZxid(),
                                    logicalclock.get(),
                                    self.getPeerState(),
                                    response.sid,
                                    current.getPeerEpoch());

                            sendqueue.offer(notmsg);
                        } else {
                            // Receive new message
                            if (LOG.isDebugEnabled()) {
                                LOG.debug("Receive new notification message. My id = "
                                        + self.getId());
                            }

                            /*
                             * We check for 28 bytes for backward compatibility
                             */
                            if (response.buffer.capacity() < 28) {
                                LOG.error("Got a short response: "
                                        + response.buffer.capacity());
                                continue;
                            }
                            boolean backCompatibility = (response.buffer.capacity() == 28);
                            response.buffer.clear();

                            // Instantiate Notification and set its attributes
                            Notification n = new Notification();
                            
                            // State of peer that sent this message
                            QuorumPeer.ServerState ackstate = QuorumPeer.ServerState.LOOKING;
                            switch (response.buffer.getInt()) {
                            case 0:
                                ackstate = QuorumPeer.ServerState.LOOKING;
                                break;
                            case 1:
                                ackstate = QuorumPeer.ServerState.FOLLOWING;
                                break;
                            case 2:
                                ackstate = QuorumPeer.ServerState.LEADING;
                                break;
                            case 3:
                                ackstate = QuorumPeer.ServerState.OBSERVING;
                                break;
                            default:
                                continue;
                            }
                            
                            n.leader = response.buffer.getLong();
                            n.zxid = response.buffer.getLong();
                            n.electionEpoch = response.buffer.getLong();
                            n.state = ackstate;
                            n.sid = response.sid;
                            if(!backCompatibility){
                                n.peerEpoch = response.buffer.getLong();
                            } else {
                                if(LOG.isInfoEnabled()){
                                    LOG.info("Backward compatibility mode, server id=" + n.sid);
                                }
                                n.peerEpoch = ZxidUtils.getEpochFromZxid(n.zxid);
                            }

                            /*
                             * Version added in 3.4.6
                             */

                            n.version = (response.buffer.remaining() >= 4) ? 
                                         response.buffer.getInt() : 0x0;

                            /*
                             * Print notification info
                             */
                            if(LOG.isInfoEnabled()){
                                printNotification(n);
                            }

                            /*
                             * If this server is looking, then send proposed leader
                             */

                            if(self.getPeerState() == QuorumPeer.ServerState.LOOKING){
                                recvqueue.offer(n);

                                /*
                                 * Send a notification back if the peer that sent this
                                 * message is also looking and its logical clock is
                                 * lagging behind.
                                 */
                                //lookForLeader()方法被调用时logicalclock++
                                if((ackstate == QuorumPeer.ServerState.LOOKING)
                                        && (n.electionEpoch < logicalclock.get())){
                                    Vote v = getVote();
                                    ToSend notmsg = new ToSend(ToSend.mType.notification,
                                            v.getId(),
                                            v.getZxid(),
                                            logicalclock.get(),
                                            self.getPeerState(),
                                            response.sid,
                                            v.getPeerEpoch());
                                    sendqueue.offer(notmsg);
                                }
                            } else {
                                /*
                                 * If this server is not looking, but the one that sent the ack
                                 * is looking, then send back what it believes to be the leader.
                                 */
                                Vote current = self.getCurrentVote();
                                if(ackstate == QuorumPeer.ServerState.LOOKING){
                                    if(LOG.isDebugEnabled()){
                                        LOG.debug("Sending new notification. My id =  " +
                                                self.getId() + " recipient=" +
                                                response.sid + " zxid=0x" +
                                                Long.toHexString(current.getZxid()) +
                                                " leader=" + current.getId());
                                    }
                                    
                                    ToSend notmsg;
                                    if(n.version > 0x0) {
                                        notmsg = new ToSend(
                                                ToSend.mType.notification,
                                                current.getId(),
                                                current.getZxid(),
                                                current.getElectionEpoch(),
                                                self.getPeerState(),
                                                response.sid,
                                                current.getPeerEpoch());
                                        
                                    } else {
                                        Vote bcVote = self.getBCVote();
                                        notmsg = new ToSend(
                                                ToSend.mType.notification,
                                                bcVote.getId(),
                                                bcVote.getZxid(),
                                                bcVote.getElectionEpoch(),
                                                self.getPeerState(),
                                                response.sid,
                                                bcVote.getPeerEpoch());
                                    }
                                    sendqueue.offer(notmsg);
                                }
                            }
                        }
                    } catch (InterruptedException e) {
                        System.out.println("Interrupted Exception while waiting for new message" +
                                e.toString());
                    }
                }
                LOG.info("WorkerReceiver is down");
            }
        }
```
## 竞选leader
前面铺垫这么多，其实都是在将集群中如何进行socket通信收发数据的，现在开始进入正题，就是leader节点的竞选过程了。在集群启动过程中会通过`FastLeaderElection#lookForLeader()`方法开始竞选。
```java
// QuorumPeer
    public void run() {
        ......
        case LOOKING:
        LOG.info("LOOKING");

        if (Boolean.getBoolean("readonlymode.enabled")) {
            ......
            } else {
            try {
                setBCVote(null);
                setCurrentVote(makeLEStrategy().lookForLeader());
            } catch (Exception e) {
                LOG.warn("Unexpected exception", e);
                setPeerState(ServerState.LOOKING);
            }
        }
        break;
        ......
    }
```
分析这个方法就对了，zookeeper集群竞选leader最核心的方法就是lookForLeader()。首先每次选举都会更新logicalclock，因此logicalclock表示选举轮数的，logicalclock越高表示节点处理的投票最多，选举状态最新。然后给自己投票`updateProposal()`，`updateProposal()`方法做的事情就是更新提议的leader，leader节点包含的属性是sid，zxid，epoch这三个。当收到选票时，先跟对方比较logicalclock，如果比自己高，则将对方选票跟自己的提议选票进行比较，比较的条件是先比较选举的届数，届数相同比较事务id，事务id相同比较服务sid，发现对方比自己大就是哟ing对方提议的leader选票；如果发现对方logicalclock比自己小，就不作任何处理；如果logicalclock相同，直接比较选票。这个过程就是找到集群中最高的选票。然后把收到的选票加入recvset保存下来，进行过半验证。过半验证时，先加入自己的提议选票，再遍历选票箱子中已有的选票，看看哪张选票过半了，将选票过半的节点视作leader。选出了leader以后，会继续等待一段时间，看看集群中有没有比自己还要大的选票，没有的话找到leader了。退出循环到外层继续运行。

这里举个例子来说明这个过程，假如有三台服务器，epoch、zxid、sid分别是0,0,1;0,0,2;0,0,3，整个过程如下:
1. 第一个服务器给自己投票，设置自己为leader，此时leader节点为0,0,1；然后发送给其他两台服务器
2. 第二台服务器设置的leader为0,0,2，并且发送自己的选票；收到第一台服务器的选票后，依次比较，epoch都是0，zxid都是0，但是自己的sid=2比第一台服务器的sid=1大，于是将投给自己的选票0,0,2发送给另外两台服务器
3. 第三台服务器设置的leader为0,0,3，并且发送自己的选票；收到第一台服务器的选票后，依次比较，epoch都是0，zxid都是0，但是自己的sid=3比第一台服务器的sid=1大，于是将投给自己的选票0,0,3发送给另外两台服务器；收到第二台服务器的选票后，依次比较，epoch都是0，zxid都是0，但是自己的sid=3比第一台服务器的sid=2大，于是将投给自己的选票0,0,3发送给另外两台服务器；这样第三台服务器一共向集群中发送了三张选票
4. 第一台服务器的选票变化为：首先收到第二台服务器的选票002保存下来，并将leader票更新为002，此时票箱中选票为2->002；当第三台发送了三张选票给集群时，首先第一台服务器添加一张选票，即3->003;当第二台服务器收到第三台服务器选票时，就会使用第三台服务器选票003，也会向集群发送003选票，此时第一台收到第二台服务器的选票为2->003，就会把2->002更新成2->003，此时选票箱有[2->002,3->003]；当收到第三台服务器的选票时，发现比自己大，于是就使用第三台服务器的选票003最为提议票；此时第一台服务器明显满足过半机制，选择第三台003作为leader
5. 第二台和第三台依次类推
```java
    public Vote lookForLeader() throws InterruptedException {
        ......
            //投票箱，key是服务器id，value是对应的投票
            HashMap<Long, Vote> recvset = new HashMap<Long, Vote>();

            HashMap<Long, Vote> outofelection = new HashMap<Long, Vote>();

            int notTimeout = finalizeWait;

            synchronized(this){
                logicalclock.incrementAndGet();
                //给自己投票，先把自己当成leader
                updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());
            }

            LOG.info("New election. My id =  " + self.getId() +
                    ", proposed zxid=0x" + Long.toHexString(proposedZxid));
            //发送给集群其他节点
            sendNotifications();

            /*
             * Loop in which we exchange notifications until we find a leader
             */

            while ((self.getPeerState() == ServerState.LOOKING) &&
                    (!stop)){
                /*
                 * Remove next notification from queue, times out after 2 times
                 * the termination time
                 */
                Notification n = recvqueue.poll(notTimeout,
                        TimeUnit.MILLISECONDS);

                /*
                 * Sends more notifications if haven't received enough.
                 * Otherwise processes new notification.
                 */
                if(n == null){
                    //先判断是否是数据没有发送完
                    if(manager.haveDelivered()){
                        //没有发送完就继续发送数据
                        sendNotifications();
                    } else {
                        //没有要发送的数据说明socket异常了，说明发送了但是没成功，需要重新连接
                        manager.connectAll();
                    }

                    /*
                     * Exponential backoff
                     */
                    int tmpTimeOut = notTimeout*2;
                    notTimeout = (tmpTimeOut < maxNotificationInterval?
                            tmpTimeOut : maxNotificationInterval);
                    LOG.info("Notification time out: " + notTimeout);
                }
                else if(validVoter(n.sid) && validVoter(n.leader)) {
                    /*
                     * Only proceed if the vote comes from a replica in the
                     * voting view for a replica in the voting view.
                     */
                    switch (n.state) {
                    case LOOKING:
                        // If notification > current, replace and send messages out
                        //如果对方的选票轮数比自己高
                        if (n.electionEpoch > logicalclock.get()) {
                            //更新成选举次数最多的轮数
                            logicalclock.set(n.electionEpoch);
                            //清空选票
                            recvset.clear();
                            if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                    getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
                                //如果对方节点比自己新，使用对方节点的选票数据
                                updateProposal(n.leader, n.zxid, n.peerEpoch);
                            } else {
                                //投票给自己
                                updateProposal(getInitId(),
                                        getInitLastLoggedZxid(),
                                        getPeerEpoch());
                            }
                            //重新发送选票
                            sendNotifications();
                            //对方节点选票轮数比自己小，忽略
                        } else if (n.electionEpoch < logicalclock.get()) {
                            if(LOG.isDebugEnabled()){
                                LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                                        + Long.toHexString(n.electionEpoch)
                                        + ", logicalclock=0x" + Long.toHexString(logicalclock.get()));
                            }
                            break;
                            //对方节点选票轮数和我相同，直接比较leader(服务id)，zxid(事务id)，epoch(届数)
                        } else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                proposedLeader, proposedZxid, proposedEpoch)) {
                            updateProposal(n.leader, n.zxid, n.peerEpoch);
                            sendNotifications();
                        }

                        if(LOG.isDebugEnabled()){
                            LOG.debug("Adding vote: from=" + n.sid +
                                    ", proposed leader=" + n.leader +
                                    ", proposed zxid=0x" + Long.toHexString(n.zxid) +
                                    ", proposed election epoch=0x" + Long.toHexString(n.electionEpoch));
                        }
                        //保存接收到的选票
                        recvset.put(n.sid, new Vote(n.leader, n.zxid, n.electionEpoch, n.peerEpoch));
                        //过半验证选举leader
                        if (termPredicate(recvset,
                                new Vote(proposedLeader, proposedZxid,
                                        logicalclock.get(), proposedEpoch))) {

                            // Verify if there is any change in the proposed leader
                            //再选举出leader以后，判断是否有sid，zxid，epoch更大的节点
                            while((n = recvqueue.poll(finalizeWait,
                                    TimeUnit.MILLISECONDS)) != null){
                                if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
                                        proposedLeader, proposedZxid, proposedEpoch)){
                                    recvqueue.put(n);
                                    break;
                                }
                            }

                            /*
                             * This predicate is true once we don't read any new
                             * relevant message from the reception queue
                             */
                            if (n == null) {
                                //n=null说明没有票了，设置自身节点的状态，要么leader，要么follower或observer
                                //在外面的线程中根据节点状态走不同代码
                                self.setPeerState((proposedLeader == self.getId()) ?
                                        ServerState.LEADING: learningState());

                                Vote endVote = new Vote(proposedLeader,
                                                        proposedZxid,
                                                        logicalclock.get(),
                                                        proposedEpoch);
                                leaveInstance(endVote);
                                return endVote;
                            }
                        }
                        break;
                        ......
```
**总结下，竞选leader时，zookeeper集群每个节点都开了四个线程，两个线程用于发送和接收字节数据，两个线程用于处理发送和接收到的数据。竞选时过程就是先给自己投票，再把自己的票和对方的票比较，将合适的发往集群，一直循环直到选出leader为止。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**