### 创建处理器链
zookeeper集群先创建处理器链，再使用每个处理器处理来自客户端的请求。leader节点的处理器链是在`LeaderZooKeeperServer#setupRequestProcessors()`方法中创建的，其实就是构建一个链表:PrepRequestProcessor->ProposalRequestProcessor(SyncRequestProcessor->AckRequestProcessor)->CommitProcessor->Leader.ToBeAppliedRequestProcessor->FinalRequestProcessor，其中ProposalRequestProcessor含有SyncRequestProcessor->AckRequestProcessor两个处理器。当客户端发送一个请求到zookeeper集群时，就会使用每个处理器进行处理。
```java
// LeaderZooKeeperServer
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor toBeAppliedProcessor = new Leader.ToBeAppliedRequestProcessor(
                finalProcessor, getLeader().toBeApplied);
        commitProcessor = new CommitProcessor(toBeAppliedProcessor,
                Long.toString(getServerId()), false,
                getZooKeeperServerListener());
        commitProcessor.start();
        ProposalRequestProcessor proposalProcessor = new ProposalRequestProcessor(this,
                commitProcessor);
        proposalProcessor.initialize();
        firstProcessor = new PrepRequestProcessor(this, proposalProcessor);
        ((PrepRequestProcessor)firstProcessor).start();
    }
```
在处理请求时还得跟follower节点和observer节点同步，因此还得看下这两个节点怎么创建处理器链的。先看下follower节点怎么创建处理器链的，这里创建了两条处理器链：FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor和SyncRequestProcessor->SendAckRequestProcessor。
```java
// FollowerZooKeeperServer
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        commitProcessor = new CommitProcessor(finalProcessor,
                Long.toString(getServerId()), true,
                getZooKeeperServerListener());
        commitProcessor.start();
        firstProcessor = new FollowerRequestProcessor(this, commitProcessor);
        ((FollowerRequestProcessor) firstProcessor).start();
        syncProcessor = new SyncRequestProcessor(this,
                new SendAckRequestProcessor((Learner)getFollower()));
        syncProcessor.start();
    }
```
再看observe节点，ObserverRequestProcessor->CommitProcessor->FinalRequestProcessor，根据配置文件中syncEnabled的值决定是否构建SyncRequestProcessor处理器，默认为true。
```java
// ObserverZooKeeperServer
    protected void setupRequestProcessors() {      
        // We might consider changing the processor behaviour of 
        // Observers to, for example, remove the disk sync requirements.
        // Currently, they behave almost exactly the same as followers.
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        commitProcessor = new CommitProcessor(finalProcessor,
                Long.toString(getServerId()), true,
                getZooKeeperServerListener());
        commitProcessor.start();
        firstProcessor = new ObserverRequestProcessor(this, commitProcessor);
        ((ObserverRequestProcessor) firstProcessor).start();
    
        if (syncRequestProcessorEnabled) {
            syncProcessor = new SyncRequestProcessor(this, null);
            syncProcessor.start();
        }
    }
```
### leader节点处理请求的过程
现在集群中的请求处理器链都创建好了，先来看下leader节点怎么处理集群的请求的，以create指令为例进行说明。
1. PrepRequestProcessor：leader节点的`PrepRequestProcessor#pRequest()`中调用`$pRequest2Txn()`方法对请求进行校验，判断创建的路径是否合法，然后在`PrepRequestProcessor#pRequest()`让下个请求处理器(ProposalRequestProcessor)进行处理`nextProcessor.processRequest(request);`
2. ProposalRequestProcessor：这里我们先看leader节点怎么处理请求的。leader节点收到请求后，在ProposalRequestProcessor处理中会先将请求转发给下个处理器CommitProcessor，再创建提议，然后使用自身的SyncRequestProcessor处理器进行处理。
```java
    public void processRequest(Request request) throws RequestProcessorException {
        if(request instanceof LearnerSyncRequest){
            zks.getLeader().processSync((LearnerSyncRequest)request);
          //leader节点收到的写数据指令在这里处理的
        } else {
                nextProcessor.processRequest(request);
            if (request.hdr != null) {
                // We need to sync and get consensus on any transactions
                try {
                    zks.getLeader().propose(request);
                } catch (XidRolloverException e) {
                    throw new RequestProcessorException(e.getMessage(), e);
                }
                syncProcessor.processRequest(request);
            }
        }
    }
```
+ CommitProcessor：顾名思义，这是一个提交请求的处理器，本身不对请求进行实质的处理。整个处理请求的过程是，
   1. 在`processRequest()`中将请求加入queuedRequests(LinkedList<Request>)队列，然后`notifyAll()`通知线程解除阻塞
   2. 在线程中将请求取出，保留到nextPending，再次wait()等待请求被提交
   3. 当其他线程调用`commit()`提交请求时会`notifyAll()`解除阻塞，将请求加入toProcess(ArrayList<Request>)
   4. 在下次while循环时使用使用下个处理器进行处理`nextProcessor.processRequest(toProcess.get(i));`
```java
    public void run() {
        try {
            //nextPending表示从队列中取出的将要在当前线程中处理的请求
            Request nextPending = null;            
            while (!finished) {
                //toProcess是个集合，保存需要下个处理器处理的请求，leader节点会将请求加入toProcess
                int len = toProcess.size();
                for (int i = 0; i < len; i++) {
                    nextProcessor.processRequest(toProcess.get(i));
                }
                toProcess.clear();
                synchronized (this) {
                    //queuedRequests保存的上个处理器ProposalRequestProcessor传进来的请求
                    if ((queuedRequests.size() == 0 || nextPending != null)
                            && committedRequests.size() == 0) {
                        //processRequest()把请求加入队列或commit()提交请求时会notifyAll()解除阻塞
                        wait();
                        continue;
                    }
                    // First check and see if the commit came in for the pending
                    // request
                    //请求提交后会加入toProcess中到下个处理器进行处理
                    if ((queuedRequests.size() == 0 || nextPending != null)
                            && committedRequests.size() > 0) {
                        Request r = committedRequests.remove();
                      
                        if (nextPending != null
                                && nextPending.sessionId == r.sessionId
                                && nextPending.cxid == r.cxid) {
                            // we want to send our version of the request.
                            // the pointer to the connection in the request
                            nextPending.hdr = r.hdr;
                            nextPending.txn = r.txn;
                            nextPending.zxid = r.zxid;
                            toProcess.add(nextPending);
                            nextPending = null;
                        } else {
                            // this request came from someone else so just
                            // send the commit packet
                            toProcess.add(r);
                        }
                    }
                }

                // We haven't matched the pending requests, so go back to
                // waiting
                if (nextPending != null) {
                    continue;
                }
                //请求加入队列时会将请求取出
                synchronized (this) {
                    // Process the next requests in the queuedRequests
                    while (nextPending == null && queuedRequests.size() > 0) {
                        //取出加入队列的请求
                        Request request = queuedRequests.remove();
                        //如果是对集群中节点进行操作的请求给nextPending
                        switch (request.type) {
                        case OpCode.create:
                        case OpCode.delete:
                        case OpCode.setData:
                        case OpCode.multi:
                        case OpCode.setACL:
                        case OpCode.createSession:
                        case OpCode.closeSession:
                            nextPending = request;
                            break;
                        case OpCode.sync:
                            //leader节点matchSyncs为false，follower和observer节点为true
                            if (matchSyncs) {
                                //follower和observer节点会在当前线程中处理
                                nextPending = request;
                            } else {
                                //leader节点会把请求加入toProcess，交给下个处理器处理
                                toProcess.add(request);
                            }
                            break;
                        default:
                            toProcess.add(request);
                        }
                    }
                }
            }
        } catch (InterruptedException e) {
            LOG.warn("Interrupted exception while waiting", e);
        } catch (Throwable e) {
            LOG.error("Unexpected exception causing CommitProcessor to exit", e);
        }
        LOG.info("CommitProcessor exited loop!");
    }
```
+ 创建提议：将请求封装成Proposal对象后，加入队列，再广播给集群中每个follower节点
```java
    public Proposal propose(Request request) throws XidRolloverException {
        /**
         * Address the rollover issue. All lower 32bits set indicate a new leader
         * election. Force a re-election instead. See ZOOKEEPER-1277
         */
        //事务id的高32位为空，则重新竞选
        if ((request.zxid & 0xffffffffL) == 0xffffffffL) {
            String msg =
                    "zxid lower 32 bits have rolled over, forcing re-election, and therefore new epoch start";
            shutdown(msg);
            throw new XidRolloverException(msg);
        }
        byte[] data = SerializeUtils.serializeRequest(request);
        proposalStats.setLastProposalSize(data.length);
        QuorumPacket pp = new QuorumPacket(Leader.PROPOSAL, request.zxid, data, null);
        //将请求封装成Proposal后，加入队列在发送出去
        Proposal p = new Proposal();
        p.packet = pp;
        p.request = request;
        synchronized (this) {
            if (LOG.isDebugEnabled()) {
                LOG.debug("Proposing:: " + request);
            }

            lastProposed = p.packet.getZxid();
            outstandingProposals.put(lastProposed, p);
            sendPacket(pp);
        }
        return p;
    }
    //发送给每个follower节点，queuePacket()会将包加入到与follower通信的线程中
    void sendPacket(QuorumPacket qp) {
        synchronized (forwardingFollowers) {
            for (LearnerHandler f : forwardingFollowers) {                
                f.queuePacket(qp);
            }
        }
    }
```
那么follower节点怎么处理PROPOSAL提议包的呢。在`Follower#followLeader()`中会接收leader节点发过来得包，然后在`processPacket()`中进行处理。首先将请求加入队列pendingTxns`LinkedBlockingQueue<Request>`，再通过SyncRequestProcessor处理器进行处理，这个处理器怎么处理的，下文会详细介绍。SyncRequestProcessor处理器处理完请求后，就到SendAckRequestProcessor处理器的`processRequest()`，这里创建了ACK应答包，写道发送缓冲区，而在SyncRequestProcessor的flush()中，发现下个处理器实现了Flushable接口，就会调用`flush()`将包发给leader节点。
+ 在ProposalRequestProcessor处理器中含有SyncRequestProcessor->AckRequestProcessor的处理器链，会依次对请求进行处理。
  + SyncRequestProcessor
   1. 队列中没有请求时会take()一直阻塞，当前队列中来了新的请求时就从队列中取出请求进行处理
   2. 记录日志，写到配置项dataDir+log前缀的文件缓存区中，如果日志到了一定次数，就将内存快照序列化到快照文件，处理完后将请求加入toFlush
   3. 下次循环时toFlush不为空，就会使用poll()取请求，如果还有请求就继续处理，没有请求就将缓冲区内容刷到日志文件，使用下个处理器进行处理
    ```java
    public void run() {
        try {
            int logCount = 0;

            // we do this in an attempt to ensure that not all of the servers
            // in the ensemble take a snapshot at the same time
            setRandRoll(r.nextInt(snapCount/2));
            while (true) {
                Request si = null;
                if (toFlush.isEmpty()) {
                    //take()为会阻塞的方法，如果队列为空就会一直阻塞，当有请求时就会解除阻塞将请求取出
                    si = queuedRequests.take();
                } else {
                    //poll()是不会阻塞的，没有元素直接返回null
                    si = queuedRequests.poll();
                    //队列中的请求被处理完时，就会flush()将缓存区数据刷到文件，注意这里是高效的批量操作
                    if (si == null) {
                        flush(toFlush);
                        continue;
                    }
                }
                if (si == requestOfDeath) {
                    break;
                }
                if (si != null) {
                    // track the number of records written to the log
                    //记录日志
                    if (zks.getZKDatabase().append(si)) {
                        logCount++;
                        if (logCount > (snapCount / 2 + randRoll)) {
                            setRandRoll(r.nextInt(snapCount/2));
                            // roll the log
                            //日志回滚，将缓冲区内容刷到文件，下次来请求时就会重新创建日志文件
                            zks.getZKDatabase().rollLog();
                            // take a snapshot
                            //在线程中将内存数据序列化到快照文件
                            if (snapInProcess != null && snapInProcess.isAlive()) {
                                LOG.warn("Too busy to snap, skipping");
                            } else {
                                snapInProcess = new ZooKeeperThread("Snapshot Thread") {
                                        public void run() {
                                            try {
                                                zks.takeSnapshot();
                                            } catch(Exception e) {
                                                LOG.warn("Unexpected exception", e);
                                            }
                                        }
                                    };
                                snapInProcess.start();
                            }
                            logCount = 0;
                        }
                    } else if (toFlush.isEmpty()) {
                        // optimization for read heavy workloads
                        // iff this is a read, and there are no pending
                        // flushes (writes), then just pass this to the next
                        // processor
                        if (nextProcessor != null) {
                            nextProcessor.processRequest(si);
                            if (nextProcessor instanceof Flushable) {
                                ((Flushable)nextProcessor).flush();
                            }
                        }
                        continue;
                    }
                    //当记录的处理请求超过1000时就会写到日志，下次来请求时就会回滚
                    toFlush.add(si);
                    if (toFlush.size() > 1000) {
                        flush(toFlush);
                    }
                }
            }
        } catch (Throwable t) {
            handleException(this.getName(), t);
            running = false;
        }
        LOG.info("SyncRequestProcessor exited!");
    }
    ```
  + AckRequestProcessor：这个请求处理器主要调用Leader类中得`processAck()`进行处理，会在两个地方被调用。一处是SyncRequestProcessor处理器处理完请求后，目的是把leader节点得sid加入ackSet集合；另外一处是在跟集群中节点通信得线程中，将发送应答包得follower节点得sid加入ackSet，然后进行过半验证。
    1. 将节点得sid加入ackSet，然后进行过半验证，即是否有半数以上得节点附议
    2. 集群提交事务，向集群中其他follower节点发送COMMIT包，follower节点收到后在`FollowerZooKeeperServer#commit()`中进行处理，过程与leader节点类似；向observer节点发送INFORM包，observer节点`ObserverZooKeeperServer#commitRequest()`方法中进行处理的，过程差不多，先写事务日志，生成快照，更新内存。
    3. 自身节点提交事务，让CommitProcessor通过nextProcessor(Leader.ToBeAppliedRequestProcessor)处理请求
    ```java
    synchronized public void processAck(long sid, long zxid, SocketAddress followerAddr) {
        ......
        //取出Proposal
        Proposal p = outstandingProposals.get(zxid);
        if (p == null) {
            LOG.warn("Trying to commit future proposal: zxid 0x{} from {}",
                    Long.toHexString(zxid), followerAddr);
            return;
        }
        //将leader节点sid加入包含提议结果的集合
        p.ackSet.add(sid);
        if (LOG.isDebugEnabled()) {
            LOG.debug("Count for zxid: 0x{} is {}",
                    Long.toHexString(zxid), p.ackSet.size());
        }
        //使用集群验证器进行过半验证，这里需要满足的条件是，集群中有半数以上的follower节点进行响应
        if (self.getQuorumVerifier().containsQuorum(p.ackSet)){             
            if (zxid != lastCommitted+1) {
                LOG.warn("Commiting zxid 0x{} from {} not first!",
                        Long.toHexString(zxid), followerAddr);
                LOG.warn("First is 0x{}", Long.toHexString(lastCommitted + 1));
            }
            //删掉已经处理过的zxid事务
            outstandingProposals.remove(zxid);
            if (p.request != null) {
                //将满足条件能被处理的提议加入toBeApplied
                toBeApplied.add(p);
            }

            if (p.request == null) {
                LOG.warn("Going to commmit null request for proposal: {}", p);
            }
            //提交事务，更新lastCommitted，向所有follower节点发送COMMIT包
            commit(zxid);
            //向所有observer节点发送INFORM
            inform(p);
            //提交事务，在leader节点中处理器链中，由CommitProcessor->Leader.ToBeAppliedRequestProcessor
            zk.commitProcessor.commit(p.request);
            //处理同步，向所有follower节点发送SYNC包
            if(pendingSyncs.containsKey(zxid)){
                for(LearnerSyncRequest r: pendingSyncs.remove(zxid)) {
                    sendSync(r);
                }
            }
        }
    }
    ```
3. Leader$ToBeAppliedRequestProcessor：这里没做什么处理，先转移到下个处理器FinalRequestProcessor，然后将提议从队列中删除，表示提议处理完成了。   
    ```java
        public void processRequest(Request request) throws RequestProcessorException {
            // request.addRQRec(">tobe");
            next.processRequest(request);
            Proposal p = toBeApplied.peek();
            if (p != null && p.request != null
                    && p.request.zxid == request.zxid) {
                toBeApplied.remove();
            }
        }
    ```
4. FinalRequestProcessor：这个跟单机模式一样，就是更新内存得操作，在内存中创建节点，写入数据。
    
### follower节点处理请求的过程
follower节点是通过这两条处理器链：FollowerRequestProcessor->CommitProcessor->FinalRequestProcessor和SyncRequestProcessor->SendAckRequestProcessor来处理请求的，依次看下即可。
1. FollowerRequestProcessor：在线程`run()`中找到CommitProcessor处理器来处理请求，再到`Learner#request()`方法，发送Leader.REQUEST请求包。
2. leader节点收到Leader.REQUEST请求后，调用`ZooKeeperServer#submitRequest()`方法提交请求，然后就是leader节点处理请求的流程了

observer节点也是类似的过程，就不再赘述。

**至此，zookeeper集群处理请求得过程看完了，现在看看这个过程。如果是leader节点，首先对请求进行校验，然后阻塞等待集群中的follower节点返回ack应答包，每当返回一个应答包就进行过半验证，完成验证后向集群包括observer节点提交事务，更新内存。follower和observer节点会发个Leader.REQUEST请求包给leader节点，再就是leader节点处理请求的过程了。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**