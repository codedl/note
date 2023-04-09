zookeeper集群运行是由配置文件决定的，因此还得从配置文件解析开始看起，解析的方法是`QuorumPeerConfig#parseProperties()`。首先获取集群中每个节点的信息，判断是否为OBSERVER观察者，通过myid文件指定自身节点的信息。发现是集群的配置之后就`runFromConfig()`开始集群启动zookeeper了。
```java
    public void parseProperties(Properties zkProp) throws IOException, ConfigException {
        else if (key.startsWith("server.")) {
                //.的位置
                int dot = key.indexOf('.');
                // sid为 server.x 的x，跟myid中数字可以对应起来指定自身节点的信息
                long sid = Long.parseLong(key.substring(dot + 1));
                String parts[] = splitWithLeadingHostname(value);
                if ((parts.length != 2) && (parts.length != 3) && (parts.length !=4)) {
                    LOG.error(value
                       + " does not have the form host:port or host:port:port " +
                       " or host:port:port:type");
                }
                LearnerType type = null;
                // IP地址:数据端口:竞选端口(:observers)
                String hostname = parts[0];
                Integer port = Integer.parseInt(parts[1]);
                Integer electionPort = null;
                if (parts.length > 2){
                	electionPort=Integer.parseInt(parts[2]);
                }
                if (parts.length > 3){
                    if (parts[3].toLowerCase().equals("observer")) {
                        type = LearnerType.OBSERVER;
                    } else if (parts[3].toLowerCase().equals("participant")) {
                        type = LearnerType.PARTICIPANT;
                    } else {
                        throw new ConfigException("Unrecognised peertype: " + value);
                    }
                }
                // 保存每个节点的信息
                if (type == LearnerType.OBSERVER){
                    observers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
                } else {
                    servers.put(Long.valueOf(sid), new QuorumServer(sid, hostname, port, electionPort, type));
                }
            }
        }
        ......
        //创建集群验证器对象
        quorumVerifier = new QuorumMaj(servers.size());
        ......
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
        
        // Warn about inconsistent peer type
        //LearnerType可以取 OBSERVER观察者 或 PARTICIPANT参与者
        LearnerType roleByServersList = observers.containsKey(serverId) ? LearnerType.OBSERVER
                : LearnerType.PARTICIPANT;
        if (roleByServersList != peerType) {
            LOG.warn("Peer type from servers list (" + roleByServersList
                    + ") doesn't match peerType (" + peerType
                    + "). Defaulting to servers list.");

            peerType = roleByServersList;
        }
        ......
    }
```
### follower与leader的连接
先创建socket句柄，生成QuorumPeer实例对象，然后启动start()。
```java
    public void runFromConfig(QuorumPeerConfig config) throws IOException {
      try {
          ManagedUtil.registerLog4jMBeans();
      } catch (JMException e) {
          LOG.warn("Unable to register log4j JMX control", e);
      }
  
      LOG.info("Starting quorum peer");
      try {
          //创建客户端通信用的socket，配置端口、最大连接数
          ServerCnxnFactory cnxnFactory = ServerCnxnFactory.createFactory();
          cnxnFactory.configure(config.getClientPortAddress(),
                                config.getMaxClientCnxns());

          quorumPeer = getQuorumPeer();

          ......//一系列set方法

          quorumPeer.start();
          quorumPeer.join();
      } catch (InterruptedException e) {
          // warn, but generally this is ok
          LOG.warn("Quorum Peer interrupted", e);
      }
    }
```
启动时做了四件事：
1. 加载快照文件到内存
2. 启动客户端通信线程
3. 开始竞选
4. 启动节点通信线程
```java
    @Override
    public synchronized void start() {
        loadDataBase();
        cnxnFactory.start();        
        startLeaderElection();
        super.start();
    }
```
集群中follower与leader的连接就在通信的线程里面。在run()中根据节点状态进入不同代码段，先看领袖和追随者怎么执行的，都是先实例化对象，再执行方法。如果是leader就执行Leader#lead()方法，如果是follower就执行Follower#followLeader()方法。
```java
    public void run() {
            while (running) {
                switch (getPeerState()) {
                case LOOKING:
                    ......
                    break;
                case OBSERVING:
                    ......
                    break;
                case FOLLOWING:
                    try {
                        LOG.info("FOLLOWING");
                        setFollower(makeFollower(logFactory));
                        follower.followLeader();
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                    } finally {
                        follower.shutdown();
                        setFollower(null);
                        setPeerState(ServerState.LOOKING);
                    }
                    break;
                case LEADING:
                    LOG.info("LEADING");
                    try {
                        setLeader(makeLeader(logFactory));
                        leader.lead();
                        setLeader(null);
                    } catch (Exception e) {
                        LOG.warn("Unexpected exception",e);
                    } finally {
                        if (leader != null) {
                            leader.shutdown("Forcing shutdown");
                            setLeader(null);
                        }
                        setPeerState(ServerState.LOOKING);
                    }
                    break;
                }
            }
    }
```
再看下整个通信的过程：
1. leader启动监听其他节点连接的线程，通过accept()监听集群中其他节点的连接，当有其他节点连接过来时，创建LearnerHandler线程进行通信。
```java
    void lead() throws IOException, InterruptedException {
        ......
        cnxAcceptor = new LearnerCnxAcceptor();
        cnxAcceptor.start();
        ......
    }
    // LearnerCnxAcceptor
        public void run() {
            try {
                while (!stop) {
                    try{
                        //监听集群中其他follower节点的连接
                        Socket s = ss.accept();
                        // start with the initLimit, once the ack is processed
                        // in LearnerHandler switch to the syncLimit
                        s.setSoTimeout(self.tickTime * self.initLimit);
                        s.setTcpNoDelay(nodelay);

                        BufferedInputStream is = new BufferedInputStream(
                                s.getInputStream());
                        //开启新的线程，即每个follower节点都有一个对应的线程
                        LearnerHandler fh = new LearnerHandler(s, is, Leader.this);
                        fh.start();
                    } catch (SocketException e) {
                        if (stop) {
                            LOG.info("exception while shutting down acceptor: "
                                    + e);

                            // When Leader.shutdown() calls ss.close(),
                            // the call to accept throws an exception.
                            // We catch and set stop to true.
                            stop = true;
                        } else {
                            throw e;
                        }
                    } catch (SaslException e){
                        LOG.error("Exception while connecting to quorum learner", e);
                    }
                }
            } catch (Exception e) {
                LOG.warn("Exception while accepting follower", e);
            }
        }
```
2. follower找到集群中的领袖，向领袖发起连接，然后注册到领袖。
```java
    void followLeader() throws InterruptedException {
        self.end_fle = Time.currentElapsedTime();
        long electionTimeTaken = self.end_fle - self.start_fle;
        self.setElectionTimeTaken(electionTimeTaken);
        LOG.info("FOLLOWING - LEADER ELECTION TOOK - {}", electionTimeTaken);
        self.start_fle = 0;
        self.end_fle = 0;
        fzk.registerJMX(new FollowerBean(this, zk), self.jmxLocalPeerBean);
        try {
            //找到领袖节点
            QuorumServer leaderServer = findLeader();            
            try {
                // 连接领袖节点
                connectToLeader(leaderServer.addr, leaderServer.hostname);
                long newEpochZxid = registerWithLeader(Leader.FOLLOWERINFO);

                //check to see if the leader zxid is lower than ours
                //this should never happen but is just a safety check
                long newEpoch = ZxidUtils.getEpochFromZxid(newEpochZxid);
                if (newEpoch < self.getAcceptedEpoch()) {
                    LOG.error("Proposed leader epoch " + ZxidUtils.zxidToString(newEpochZxid)
                            + " is less than our accepted epoch " + ZxidUtils.zxidToString(self.getAcceptedEpoch()));
                    throw new IOException("Error: Epoch of leader is lower");
                }
                syncWithLeader(newEpochZxid);                
                QuorumPacket qp = new QuorumPacket();
                while (this.isRunning()) {
                    readPacket(qp);
                    processPacket(qp);
                }
            } catch (Exception e) {
                LOG.warn("Exception when following the leader", e);
                try {
                    sock.close();
                } catch (IOException e1) {
                    e1.printStackTrace();
                }
    
                // clear pending revalidations
                pendingRevalidations.clear();
            }
        } finally {
            zk.unregisterJMX((Learner)this);
        }
    }
```
先构造发送给服务器的包，将本节点sid发过去，再等待服务器响应，然后发送ack应答包。
```java
   protected long registerWithLeader(int pktType) throws IOException{
        /*
         * Send follower info, including last zxid and sid
         */
    	long lastLoggedZxid = self.getLastLoggedZxid();
        // 构造发送给服务器的数据包
        QuorumPacket qp = new QuorumPacket();                
        qp.setType(pktType);
        qp.setZxid(ZxidUtils.makeZxid(self.getAcceptedEpoch(), 0));
        
        /*
         * Add sid to payload
         */
        //发送leader节点的数据
        LearnerInfo li = new LearnerInfo(self.getId(), 0x10000);
        ByteArrayOutputStream bsid = new ByteArrayOutputStream();
        BinaryOutputArchive boa = BinaryOutputArchive.getArchive(bsid);
        boa.writeRecord(li, "LearnerInfo");
        qp.setData(bsid.toByteArray());
        //发送给服务器
        writePacket(qp, true);
        //接收来自leader的数据
        readPacket(qp);        
        final long newEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
        //服务端返回的包为Leader.LEADERINFO
		if (qp.getType() == Leader.LEADERINFO) {
        	// we are connected to a 1.0 server so accept the new epoch and read the next packet
        	leaderProtocolVersion = ByteBuffer.wrap(qp.getData()).getInt();
        	byte epochBytes[] = new byte[4];
        	final ByteBuffer wrappedEpochBytes = ByteBuffer.wrap(epochBytes);
        	if (newEpoch > self.getAcceptedEpoch()) {
        		wrappedEpochBytes.putInt((int)self.getCurrentEpoch());
        		self.setAcceptedEpoch(newEpoch);
        	} else if (newEpoch == self.getAcceptedEpoch()) {
        		// since we have already acked an epoch equal to the leaders, we cannot ack
        		// again, but we still need to send our lastZxid to the leader so that we can
        		// sync with it if it does assume leadership of the epoch.
        		// the -1 indicates that this reply should not count as an ack for the new epoch
                wrappedEpochBytes.putInt(-1);
        	} else {
        		throw new IOException("Leaders epoch, " + newEpoch + " is less than accepted epoch, " + self.getAcceptedEpoch());
        	}
            //写个应答包给服务端
        	QuorumPacket ackNewEpoch = new QuorumPacket(Leader.ACKEPOCH, lastLoggedZxid, epochBytes, null);
        	writePacket(ackNewEpoch, true);
            return ZxidUtils.makeZxid(newEpoch, 0);
        } else {
        	if (newEpoch > self.getAcceptedEpoch()) {
        		self.setAcceptedEpoch(newEpoch);
        	}
            if (qp.getType() != Leader.NEWLEADER) {
                LOG.error("First packet should have been NEWLEADER");
                throw new IOException("First packet should have been NEWLEADER");
            }
            return qp.getZxid();
        }
    }
``` 
3. 服务端收到包反序列化成QuorumPacket对象,在跟这个节点通信的线程中记录节点的sid，更新选举的届数Epoch，发送标识为Leader.LEADERINFO的应答包给客户端，等待客户端的ack应当包。
```java
    public void run() {
        try {
            leader.addLearnerHandler(this);
            tickOfNextAckDeadline = leader.self.tick.get()
                    + leader.self.initLimit + leader.self.syncLimit;

            ia = BinaryInputArchive.getArchive(bufferedInput);
            bufferedOutput = new BufferedOutputStream(sock.getOutputStream());
            oa = BinaryOutputArchive.getArchive(bufferedOutput);
            // 将接收到的字节流数据反序列化成QuorumPacket对象
            QuorumPacket qp = new QuorumPacket();
            ia.readRecord(qp, "packet");
            //不是追随者或观察者的话直接报错
            if(qp.getType() != Leader.FOLLOWERINFO && qp.getType() != Leader.OBSERVERINFO){
            	LOG.error("First packet " + qp.toString()
                        + " is not FOLLOWERINFO or OBSERVERINFO!");
                return;
            }
            byte learnerInfoData[] = qp.getData();
            //更新sid,这是跟集群中其他节点通信的线程，sid的值要跟这个节点相同
            if (learnerInfoData != null) {
                // 如果是8个字节，即一个long
            	if (learnerInfoData.length == 8) {
            		ByteBuffer bbsid = ByteBuffer.wrap(learnerInfoData);
            		this.sid = bbsid.getLong();
            	} else {
                    // 发送的是LearnerInfo，通过反序列化得到
            		LearnerInfo li = new LearnerInfo();
            		ByteBufferInputStream.byteBuffer2Record(ByteBuffer.wrap(learnerInfoData), li);
            		this.sid = li.getServerid();
            		this.version = li.getProtocolVersion();
            	}
            } else {
            	this.sid = leader.followerCounter.getAndDecrement();
            }

            LOG.info("Follower sid: " + sid + " : info : "
                    + leader.self.quorumPeers.get(sid));
                        
            if (qp.getType() == Leader.OBSERVERINFO) {
                  learnerType = LearnerType.OBSERVER;
            }            
            // zxid的高32位表示届数epoch，低32位表示事务
            long lastAcceptedEpoch = ZxidUtils.getEpochFromZxid(qp.getZxid());
            
            long peerLastZxid;
            StateSummary ss = null;
            long zxid = qp.getZxid();
            //创建新的届数newEpoch
            long newEpoch = leader.getEpochToPropose(this.getSid(), lastAcceptedEpoch);
            
            if (this.getVersion() < 0x10000) {
                // we are going to have to extrapolate the epoch information
                long epoch = ZxidUtils.getEpochFromZxid(zxid);
                ss = new StateSummary(epoch, zxid);
                // fake the message
                leader.waitForEpochAck(this.getSid(), ss);
            } else {
                //将0x10000存到这个数组
                byte ver[] = new byte[4];
                ByteBuffer.wrap(ver).putInt(0x10000);
                //构造返回给其他节点的QuorumPacket
                QuorumPacket newEpochPacket = new QuorumPacket(Leader.LEADERINFO, ZxidUtils.makeZxid(newEpoch, 0), ver, null);
                oa.writeRecord(newEpochPacket, "packet");
                bufferedOutput.flush();
                //读取来自客户端的应答包
                QuorumPacket ackEpochPacket = new QuorumPacket();
                ia.readRecord(ackEpochPacket, "packet");
                if (ackEpochPacket.getType() != Leader.ACKEPOCH) {
                    LOG.error(ackEpochPacket.toString()
                            + " is not ACKEPOCH");
                    return;
				}
                ByteBuffer bbepoch = ByteBuffer.wrap(ackEpochPacket.getData());
                ss = new StateSummary(bbepoch.getInt(), ackEpochPacket.getZxid());
                leader.waitForEpochAck(this.getSid(), ss);
            }
        }
    }
```
### 数据同步
zookeeper集群创建成功后，主要就是LearnerHandler和Learner这两个线程之间的通信，接下来看看数据同步的过程。首先在`FinalRequestProcessor#processRequest()`中会将集群请求保存下来`ZKDatabase#addCommittedProposal()`
```java
    public void addCommittedProposal(Request request) {
        WriteLock wl = logLock.writeLock();
        try {
            wl.lock();
            //commitLogCount=500;即默认保存最近的500个包
            if (committedLog.size() > commitLogCount) {
                //500次事务之前的包会被移除
                committedLog.removeFirst();
                //minCommittedLog保存最小的事务id
                minCommittedLog = committedLog.getFirst().packet.getZxid();
            }
            if (committedLog.size() == 0) {
                minCommittedLog = request.zxid;
                maxCommittedLog = request.zxid;
            }

            //
            byte[] data = SerializeUtils.serializeRequest(request);
            QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data, null);
            Proposal p = new Proposal();
            p.packet = pp;
            p.request = request;
            //集群中每次更新的提议都会被加入到committedLog中保存，同步的时候会用到
            committedLog.add(p);
            //maxCommittedLog是最大的事务id
            maxCommittedLog = p.packet.getZxid();
        } finally {
            wl.unlock();
        }
    }
```
然后再回到线程观察数据同步的过程。这里主要是确定包的类型，然后leader发送，follower处理并答复。
1. 将来自follower节点的事务和leader节点的事务进行比较，一共有四种情况。
   + 如果相等以为数据是相同的不用再同步了，此时发送的是Leader.DIFF
   + follower节点的事务id在leader的最小与最大之间，需要从peerLastZxid(follower节点的当前事务)开始同步，发送的是Leader.DIFF
   + follower节点的事务id比leader节点保存最大事务还大，需要把leader节点的事务id大的事务清除掉，说明zookeeper集群是以leader节点为主的，发送的是Leader.TRUNC
   + follower节点的事务id比leader保存的最小事务还小，此时会丢弃follower节点的数据，将leader节点的数据同步过去，发送的是Leader.SNAP
```java
            peerLastZxid = ss.getLastZxid();
            
            /* the default to send to the follower */
            int packetToSend = Leader.SNAP;
            long zxidToSend = 0;
            long leaderLastZxid = 0;
            /** the packets that the follower needs to get updates from **/
            long updates = peerLastZxid;
            
            /* we are sending the diff check if we have proposals in memory to be able to 
             * send a diff to the 
             */ 
            ReentrantReadWriteLock lock = leader.zk.getZKDatabase().getLogLock();
            ReadLock rl = lock.readLock();
            try {
                rl.lock();
                //获取当前leader节点中最大和最小的事务id
                final long maxCommittedLog = leader.zk.getZKDatabase().getmaxCommittedLog();
                final long minCommittedLog = leader.zk.getZKDatabase().getminCommittedLog();
                LOG.info("Synchronizing with Follower sid: " + sid
                        +" maxCommittedLog=0x"+Long.toHexString(maxCommittedLog)
                        +" minCommittedLog=0x"+Long.toHexString(minCommittedLog)
                        +" peerLastZxid=0x"+Long.toHexString(peerLastZxid));

                LinkedList<Proposal> proposals = leader.zk.getZKDatabase().getCommittedLog();
                //相等意味着follower与leader的数据相同不用同步
                if (peerLastZxid == leader.zk.getZKDatabase().getDataTreeLastProcessedZxid()) {
                    // Follower is already sync with us, send empty diff
                    LOG.info("leader and follower are in sync, zxid=0x{}",
                            Long.toHexString(peerLastZxid));
                    packetToSend = Leader.DIFF;
                    zxidToSend = peerLastZxid;
                } else if (proposals.size() != 0) {
                    LOG.debug("proposal size is {}", proposals.size());
                    //follower节点的事务id在leader的最小与最大之间，需要从peerLastZxid开始同步
                    if ((maxCommittedLog >= peerLastZxid)
                            && (minCommittedLog <= peerLastZxid)) {
                        LOG.debug("Sending proposals to follower");

                        // as we look through proposals, this variable keeps track of previous
                        // proposal Id.
                        long prevProposalZxid = minCommittedLog;

                        // Keep track of whether we are about to send the first packet.
                        // Before sending the first packet, we have to tell the learner
                        // whether to expect a trunc or a diff
                        boolean firstPacket=true;

                        // If we are here, we can use committedLog to sync with
                        // follower. Then we only need to decide whether to
                        // send trunc or not
                        packetToSend = Leader.DIFF;
                        zxidToSend = maxCommittedLog;

                        for (Proposal propose: proposals) {
                            // skip the proposals the peer already has
                            //跳过已有的事务
                            if (propose.packet.getZxid() <= peerLastZxid) {
                                prevProposalZxid = propose.packet.getZxid();
                                continue;
                            } else {
                                // If we are sending the first packet, figure out whether to trunc
                                // in case the follower has some proposals that the leader doesn't
                                if (firstPacket) {
                                    firstPacket = false;
                                    // Does the peer have some proposals that the leader hasn't seen yet
                                    if (prevProposalZxid < peerLastZxid) {
                                        // send a trunc message before sending the diff
                                        packetToSend = Leader.TRUNC;                                        
                                        zxidToSend = prevProposalZxid;
                                        updates = zxidToSend;
                                    }
                                }
                                //将需要同步的事务添加到队列
                                queuePacket(propose.packet);
                                QuorumPacket qcommit = new QuorumPacket(Leader.COMMIT, propose.packet.getZxid(),
                                        null, null);
                                queuePacket(qcommit);
                            }
                        }
                      //让follower节点丢弃比leader节点新的数据，因为leader是集群选举出来的
                      //leader节点的数据跟集群中大部分节点的数据都是同步，所以以leader节点数据为主
                    } else if (peerLastZxid > maxCommittedLog) {
                        LOG.debug("Sending TRUNC to follower zxidToSend=0x{} updates=0x{}",
                                Long.toHexString(maxCommittedLog),
                                Long.toHexString(updates));

                        packetToSend = Leader.TRUNC;
                        zxidToSend = maxCommittedLog;
                        updates = zxidToSend;
                    } else {
                        LOG.warn("Unhandled proposal scenario");
                    }
                } else {
                    // just let the state transfer happen
                    LOG.debug("proposals is empty");
                }               

                LOG.info("Sending " + Leader.getPacketType(packetToSend));
                leaderLastZxid = leader.startForwarding(this, updates);

            } finally {
                rl.unlock();
            }
```
2. leader发送NEWLEADER包，等待follower答复；follower设置届数，并写到文件currentEpoch里，然后发送ack应答包给leader
```java
// leader$LearnerHandler
            //发送Leader.NEWLEADER类型的QuorumPacket
             QuorumPacket newLeaderQP = new QuorumPacket(Leader.NEWLEADER,
                    ZxidUtils.makeZxid(newEpoch, 0), null, null);
             if (getVersion() < 0x10000) {
                oa.writeRecord(newLeaderQP, "packet");
            } else {
                queuedPackets.add(newLeaderQP);
            }
            bufferedOutput.flush();

// follower$syncWithLeader()
            case Leader.NEWLEADER: // Getting NEWLEADER here instead of in discovery 
    
            //找到快照文件
            File updating = new File(self.getTxnFactory().getSnapDir(),
                                QuorumPeer.UPDATING_EPOCH_FILENAME);
            if (!updating.exists() && !updating.createNewFile()) {
                throw new IOException("Failed to create " +
                                        updating.toString());
            }
            //如果是diff包，表示follower与leader之间存在差异，不用加载快照文件
            if (snapshotNeeded) {
                zk.takeSnapshot();
            }
            //设置届数，并将届数保存到文件currentEpoch
            self.setCurrentEpoch(newEpoch);
            if (!updating.delete()) {
                throw new IOException("Failed to delete " +
                                        updating.toString());
            }
            writeToTxnLog = true; //Anything after this needs to go to the transaction log, not applied directly in memory
            isPreZAB1_0 = false;
            //返回leader应答包，表示操作完成
            writePacket(new QuorumPacket(Leader.ACK, newLeaderZxid, null, null), true);
            break;
```
3. leader发送SNAP快照包，follower清空当前数据库，加载leader发送过来的快照，设置事务id
```java
// leader$LearnerHandler
            if (packetToSend == Leader.SNAP) {
                zxidToSend = leader.zk.getZKDatabase().getDataTreeLastProcessedZxid();
            }
            oa.writeRecord(new QuorumPacket(packetToSend, zxidToSend, null, null), "packet");
            bufferedOutput.flush();
// follower$syncWithLeader()
            else if (qp.getType() == Leader.SNAP) {
                LOG.info("Getting a snapshot from leader 0x" + Long.toHexString(qp.getZxid()));
                // The leader is going to dump the database
                // clear our own database and read
                zk.getZKDatabase().clear();
                zk.getZKDatabase().deserializeSnapshot(leaderIs);
                String signature = leaderIs.readString("signature");
                if (!signature.equals("BenWasHere")) {
                    LOG.error("Missing signature. Got " + signature);
                    throw new IOException("Missing signature");                   
                }
                zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
            }
```
4. leader发送TRUNC包，follower根据leader发送的事务id进行回滚，回滚到leader发送的事务id。
```java
// leader$LearnerHandler
            else if (peerLastZxid > maxCommittedLog) {
                LOG.debug("Sending TRUNC to follower zxidToSend=0x{} updates=0x{}",
                        Long.toHexString(maxCommittedLog),
                        Long.toHexString(updates));

                packetToSend = Leader.TRUNC;
                zxidToSend = maxCommittedLog;
                updates = zxidToSend;
            } 
// follower$syncWithLeader()            
            else if (qp.getType() == Leader.TRUNC) {
                //we need to truncate the log to the lastzxid of the leader
                LOG.warn("Truncating log to get in sync with the leader 0x"
                        + Long.toHexString(qp.getZxid()));
                boolean truncated=zk.getZKDatabase().truncateLog(qp.getZxid());
                if (!truncated) {
                    // not able to truncate the log
                    LOG.error("Not able to truncate the log "
                            + Long.toHexString(qp.getZxid()));
                    System.exit(13);
                }
                zk.getZKDatabase().setlastProcessedZxid(qp.getZxid());
            }
```
5. leader发送的COMMIT包，follower判断是否要写事务日志，如果不需要就直接更新内存，需要的话就加到队列，之后再根据节点的类型判断是否要提交事务。Follower需要提交事务，Observer不要提交。
```java
// follower$syncWithLeader()            
        case Leader.COMMIT:
            //是否需要写事务日志
            if (!writeToTxnLog) {
                pif = packetsNotCommitted.peekFirst();
                if (pif.hdr.getZxid() != qp.getZxid()) {
                    LOG.warn("Committing " + qp.getZxid() + ", but next proposal is " + pif.hdr.getZxid());
                } else {
                    //直接更新内存
                    zk.processTxn(pif.hdr, pif.rec);
                    packetsNotCommitted.remove();
                }
            } else {
                packetsCommitted.add(qp.getZxid());
            }
        break;
.....
        if (zk instanceof FollowerZooKeeperServer) {
            FollowerZooKeeperServer fzk = (FollowerZooKeeperServer)zk;
            for(PacketInFlight p: packetsNotCommitted) {
                fzk.logRequest(p.hdr, p.rec);
            }
            //提交所有事务
            for(Long zxid: packetsCommitted) {
                fzk.commit(zxid);
            }
            //observer会更新内存，不会提交事务
        } else if (zk instanceof ObserverZooKeeperServer) {
            // Similar to follower, we need to log requests between the snapshot
            // and UPTODATE
            ObserverZooKeeperServer ozk = (ObserverZooKeeperServer) zk;
            for (PacketInFlight p : packetsNotCommitted) {
                Long zxid = packetsCommitted.peekFirst();
                if (p.hdr.getZxid() != zxid) {
                    // log warning message if there is no matching commit
                    // old leader send outstanding proposal to observer
                    LOG.warn("Committing " + Long.toHexString(zxid)
                            + ", but next proposal is "
                            + Long.toHexString(p.hdr.getZxid()));
                    continue;
                }
                packetsCommitted.remove();
                Request request = new Request(null, p.hdr.getClientId(),
                        p.hdr.getCxid(), p.hdr.getType(), null, null);
                request.txn = p.rec;
                request.hdr = p.hdr;
                ozk.commitRequest(request);
            }
        } else {
            // New server type need to handle in-flight packets
            throw new UnsupportedOperationException("Unknown server type");
        }                
```
6. leader在循环中处理集群中的写数据请求，再同步给follower节点，follower节点先读数据包再处理。
```java
// leader$LearnerHandler
            while (true) {
                qp = new QuorumPacket();
                ia.readRecord(qp, "packet");

                long traceMask = ZooTrace.SERVER_PACKET_TRACE_MASK;
                if (qp.getType() == Leader.PING) {
                    traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
                }
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logQuorumPacket(LOG, traceMask, 'i', qp);
                }
                tickOfNextAckDeadline = leader.self.tick.get() + leader.self.syncLimit;


                ByteBuffer bb;
                long sessionId;
                int cxid;
                int type;

                switch (qp.getType()) {
                case Leader.ACK:
                    if (this.learnerType == LearnerType.OBSERVER) {
                        if (LOG.isDebugEnabled()) {
                            LOG.debug("Received ACK from Observer  " + this.sid);
                        }
                    }
                    syncLimitCheck.updateAck(qp.getZxid());
                    leader.processAck(this.sid, qp.getZxid(), sock.getLocalSocketAddress());
                    break;
                case Leader.PING:
                    // Process the touches
                    ByteArrayInputStream bis = new ByteArrayInputStream(qp
                            .getData());
                    DataInputStream dis = new DataInputStream(bis);
                    while (dis.available() > 0) {
                        long sess = dis.readLong();
                        int to = dis.readInt();
                        leader.zk.touch(sess, to);
                    }
                    break;
                case Leader.REVALIDATE:
                    bis = new ByteArrayInputStream(qp.getData());
                    dis = new DataInputStream(bis);
                    long id = dis.readLong();
                    int to = dis.readInt();
                    ByteArrayOutputStream bos = new ByteArrayOutputStream();
                    DataOutputStream dos = new DataOutputStream(bos);
                    dos.writeLong(id);
                    boolean valid = leader.zk.touch(id, to);
                    if (valid) {
                        try {
                            //set the session owner
                            // as the follower that
                            // owns the session
                            leader.zk.setOwner(id, this);
                        } catch (SessionExpiredException e) {
                            LOG.error("Somehow session " + Long.toHexString(id) + " expired right after being renewed! (impossible)", e);
                        }
                    }
                    if (LOG.isTraceEnabled()) {
                        ZooTrace.logTraceMessage(LOG,
                                                 ZooTrace.SESSION_TRACE_MASK,
                                                 "Session 0x" + Long.toHexString(id)
                                                 + " is valid: "+ valid);
                    }
                    dos.writeBoolean(valid);
                    qp.setData(bos.toByteArray());
                    queuedPackets.add(qp);
                    break;
                case Leader.REQUEST:                    
                    bb = ByteBuffer.wrap(qp.getData());
                    sessionId = bb.getLong();
                    cxid = bb.getInt();
                    type = bb.getInt();
                    bb = bb.slice();
                    Request si;
                    if(type == OpCode.sync){
                        si = new LearnerSyncRequest(this, sessionId, cxid, type, bb, qp.getAuthinfo());
                    } else {
                        si = new Request(null, sessionId, cxid, type, bb, qp.getAuthinfo());
                    }
                    si.setOwner(this);
                    leader.zk.submitRequest(si);
                    break;
                default:
                    LOG.warn("unexpected quorum packet, type: {}", packetToString(qp));
                    break;
                }
            }
// follower$syncWithLeader()            
            while (this.isRunning()) {
                readPacket(qp);
                processPacket(qp);
            }
```