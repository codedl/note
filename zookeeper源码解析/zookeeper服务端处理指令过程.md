---
title: zookeeper服务端处理指令的过程
tags:
notebook: zookeeper源码解析
---
zookeeper在启动时会创建请求处理器链`setupRequestProcessors()`，可以看到构造处理器时传入了上一个处理器，这样最终会形成单向链表PrepRequestProcessor->SyncRequestProcessor->FinalRequestProcessor
```java
    protected void setupRequestProcessors() {
        RequestProcessor finalProcessor = new FinalRequestProcessor(this);
        RequestProcessor syncProcessor = new SyncRequestProcessor(this,finalProcessor);
        ((SyncRequestProcessor)syncProcessor).start();
        firstProcessor = new PrepRequestProcessor(this, syncProcessor);
        ((PrepRequestProcessor)firstProcessor).start();
    }
```
### PrepRequestProcessor
以创建节点的请求为例进行说明，收到客户端的请求后，首先会通过firstProcessor(PrepRequestProcessor)开始处理客户端请求，第一个处理器就是把请求加入到队列submittedRequests`LinkedBlockingQueue<Request> submittedRequests = new LinkedBlockingQueue<Request>();`。
```java
    public void processRequest(Request request) {
        // request.addRQRec(">prep="+zks.outstandingChanges.size());
        submittedRequests.add(request);
    }
```
然后在本地线程中将请求从队列submittedRequests(LinkedBlockingQueue<Request>)取出进行相应的处理。
```java
    public void run() {
        try {
            while (true) {
                // 取出请求
                Request request = submittedRequests.take();
                long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
                if (request.type == OpCode.ping) {
                    traceMask = ZooTrace.CLIENT_PING_TRACE_MASK;
                }
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logRequest(LOG, traceMask, 'P', request, "");
                }
                if (Request.requestOfDeath == request) {
                    break;
                }
                // 处理请求
                pRequest(request);
            }
        } catch (RequestProcessorException e) {
            if (e.getCause() instanceof XidRolloverException) {
                LOG.info(e.getCause().getMessage());
            }
            handleException(this.getName(), e);
        } catch (Exception e) {
            handleException(this.getName(), e);
        }
        LOG.info("PrepRequestProcessor exited loop!");
    }
```
这里就是对请求进行校验，判断创建的路径是否合法，找到父路径，设置父路径修改记录，并且加入到队列中addChangeRecord()，然后在`pRequest()`中`nextProcessor.processRequest(request);`交给下个处理器SyncRequestProcessor处理。
```java
    protected void pRequest2Txn(int type, long zxid, Request request, Record record, boolean deserialize)
        throws KeeperException, IOException, RequestProcessorException
    {
        request.hdr = new TxnHeader(request.sessionId, request.cxid, zxid,
                                    Time.currentWallTime(), type);

        switch (type) {
            case OpCode.create:                
                // 检查会话
                zks.sessionTracker.checkSession(request.sessionId, request.getOwner());
                CreateRequest createRequest = (CreateRequest)record;  
                // 对客户端请求进行反序列化，会检验数据是否超过1M(BinaryInputArchive.java#checkLength)
                if(deserialize)
                    ByteBufferInputStream.byteBuffer2Record(request.request, createRequest);
                String path = createRequest.getPath();
                int lastSlash = path.lastIndexOf('/');
                // 路径中必须有'/'
                if (lastSlash == -1 || path.indexOf('\0') != -1 || failCreate) {
                    LOG.info("Invalid path " + path + " with session 0x" +
                            Long.toHexString(request.sessionId));
                    throw new KeeperException.BadArgumentsException(path);
                }
                List<ACL> listACL = removeDuplicates(createRequest.getAcl());
                if (!fixupACL(request.authInfo, listACL)) {
                    throw new KeeperException.InvalidACLException(path);
                }
                String parentPath = path.substring(0, lastSlash);
                // 获取父路径请求的修改记录
                ChangeRecord parentRecord = getRecordForPath(parentPath);

                checkACL(zks, parentRecord.acl, ZooDefs.Perms.CREATE,
                        request.authInfo);
                int parentCVersion = parentRecord.stat.getCversion();
                CreateMode createMode =
                    CreateMode.fromFlag(createRequest.getFlags());
                if (createMode.isSequential()) {
                    path = path + String.format(Locale.ENGLISH, "%010d", parentCVersion);
                }
                validatePath(path, request.sessionId);
                try {
                    if (getRecordForPath(path) != null) {
                        throw new KeeperException.NodeExistsException(path);
                    }
                } catch (KeeperException.NoNodeException e) {
                    // ignore this one
                }
                boolean ephemeralParent = parentRecord.stat.getEphemeralOwner() != 0;
                if (ephemeralParent) {
                    throw new KeeperException.NoChildrenForEphemeralsException(path);
                }
                // 父路径修改newCversion+1
                int newCversion = parentRecord.stat.getCversion()+1;
                request.txn = new CreateTxn(path, createRequest.getData(),
                        listACL,
                        createMode.isEphemeral(), newCversion);
                StatPersisted s = new StatPersisted();
                if (createMode.isEphemeral()) {
                    s.setEphemeralOwner(request.sessionId);
                }
                parentRecord = parentRecord.duplicate(request.hdr.getZxid());
                parentRecord.childCount++;
                parentRecord.stat.setCversion(newCversion);
                // 将修改记录加入队列
                addChangeRecord(parentRecord);
                addChangeRecord(new ChangeRecord(request.hdr.getZxid(), path, s, 0, listACL));
                break;
            }
        }
    }
```
### SyncRequestProcessor
这个处理器的思路差不多，也是先把请求加入到队列中，然后在本地线程中取出来进行处理。
```java
    public void processRequest(Request request) {
        // request.addRQRec(">sync");
        queuedRequests.add(request);
    }
```
这里使用的是阻塞队列queuedRequests(LinkedBlockingQueue<Request>)，在run()中如果队列为空则queuedRequests.take()的时候线程会阻塞，当queuedRequests.add()的时候会signal()通知阻塞的线程取消阻塞，此时会从队列中取出请求数据。然后发现请求不为空`if (si != null)`，则将请求流放，当满足条件时进行回滚，并开个线程生成日志快照，再将处理的请求加入toFlush，当队列中的请求都处理完时就flush(toFlush);，接下来一步步看吧。
```java
    public void run() {
        try {
            int logCount = 0;

            // we do this in an attempt to ensure that not all of the servers
            // in the ensemble take a snapshot at the same time
            setRandRoll(r.nextInt(snapCount/2));
            while (true) {
                Request si = null;
                // 没有请求时阻塞等待取出请求
                if (toFlush.isEmpty()) {
                    si = queuedRequests.take();
                } else {
                    // 有请求时直接去获取请求
                    si = queuedRequests.poll();
                    // null意味着到队列尾了，即队列为空了
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
                    if (zks.getZKDatabase().append(si)) {
                        logCount++;
                        if (logCount > (snapCount / 2 + randRoll)) {
                            setRandRoll(r.nextInt(snapCount/2));
                            // roll the log
                            zks.getZKDatabase().rollLog();
                            // take a snapshot
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
1. 更新事务lastZxidSeen，当发现logStream为null说明要么刚刚启动，要么回滚了，重新生成logStream。有了logStream后，把数据缓存下来。
```java
    public synchronized boolean append(TxnHeader hdr, Record txn)
        throws IOException
    {
        if (hdr == null) {
            return false;
        }

        if (hdr.getZxid() <= lastZxidSeen) {
            LOG.warn("Current zxid " + hdr.getZxid()
                    + " is <= " + lastZxidSeen + " for "
                    + hdr.getType());
        } else {
            lastZxidSeen = hdr.getZxid();
        }

        if (logStream==null) {
           if(LOG.isInfoEnabled()){
                LOG.info("Creating new log file: " + Util.makeLogName(hdr.getZxid()));
           }

           logFileWrite = new File(logDir, Util.makeLogName(hdr.getZxid()));
           fos = new FileOutputStream(logFileWrite);
           logStream=new BufferedOutputStream(fos);
           oa = BinaryOutputArchive.getArchive(logStream);
           FileHeader fhdr = new FileHeader(TXNLOG_MAGIC,VERSION, dbId);
           fhdr.serialize(oa, "fileheader");
           // Make sure that the magic number is written before padding.
           logStream.flush();
           filePadding.setCurrentSize(fos.getChannel().position());
           streamsToFlush.add(fos);
        }
        filePadding.padFile(fos.getChannel());
        byte[] buf = Util.marshallTxnEntry(hdr, txn);
        if (buf == null || buf.length == 0) {
            throw new IOException("Faulty serialization for header " +
                    "and txn");
        }
        Checksum crc = makeChecksumAlgorithm();
        crc.update(buf, 0, buf.length);
        oa.writeLong(crc.getValue(), "txnEntryCRC");
        Util.writeTxnBytes(oa, buf);

        return true;
    }
```
2. 如果`logCount > (snapCount / 2 + randRoll)`，logCount为事务计数，snapCount为快照计数，randRoll为snapCount/2以内的随机数，当条件满足时，就会回滚日志，并开启线程生成快照。
   1. 先看回滚日志，不麻烦，就是flush()，然后将logStream=null，之后便会重新生成。
    ```java
        public synchronized void rollLog() throws IOException {
            if (logStream != null) {
                this.logStream.flush();
                this.logStream = null;
                oa = null;
            }
        }
    ```
    2. 再看生成快照，就是获取当前的事务lastProcessedZxid，创建快照文件，然后把内存中的数据序列话到快照文件。
    ```java
        public void save(DataTree dataTree,
                ConcurrentHashMap<Long, Integer> sessionsWithTimeouts)
            throws IOException {
            long lastZxid = dataTree.lastProcessedZxid;
            File snapshotFile = new File(snapDir, Util.makeSnapshotName(lastZxid));
            LOG.info("Snapshotting: 0x{} to {}", Long.toHexString(lastZxid),
                    snapshotFile);
            snapLog.serialize(dataTree, sessionsWithTimeouts, snapshotFile);
        }
    ```
3. 请求处理完之后在run()中`si = queuedRequests.poll();`，此时取出的请求是null，就会flush()。首先提交，最终还是`logStream.flush();`，在找到下个处理器处理请求。
```java
    private void flush(LinkedList<Request> toFlush) throws IOException, RequestProcessorException
    {
        if (toFlush.isEmpty())
            return;

        zks.getZKDatabase().commit();
        while (!toFlush.isEmpty()) {
            Request i = toFlush.remove();
            if (nextProcessor != null) {
                nextProcessor.processRequest(i);
            }
        }
        if (nextProcessor != null && nextProcessor instanceof Flushable) {
            ((Flushable)nextProcessor).flush();
        }
    }
```
这个处理器做的事情就是先做事务提交，到一定数量就生成快照，再看下个处理器吧。
### FinalRequestProcessor
在FinalRequestProcessor中首先移除比请求中的事务zxid小的修改记录，然后在内存中添加数据，最后生成响应返回给客户端。
```java
    public void processRequest(Request request) {
        synchronized (zks.outstandingChanges) {
            // 队列不为空时，将比当前请求的事务zxid小的修改删除
            while (!zks.outstandingChanges.isEmpty()
                    && zks.outstandingChanges.get(0).zxid <= request.zxid) {
                ChangeRecord cr = zks.outstandingChanges.remove(0);
                if (cr.zxid < request.zxid) {
                    LOG.warn("Zxid outstanding "
                            + cr.zxid
                            + " is less than current " + request.zxid);
                }
                if (zks.outstandingChangesForPath.get(cr.path) == cr) {
                    zks.outstandingChangesForPath.remove(cr.path);
                }
            }
            if (request.hdr != null) {
               TxnHeader hdr = request.hdr;
               Record txn = request.txn;
            // 处理请求
               rc = zks.processTxn(hdr, txn);
            }
            // do not add non quorum packets to the queue.
            if (Request.isQuorum(request.type)) {
                zks.getZKDatabase().addCommittedProposal(request);
            }
        }
        // 根据请求类型创建响应返回给客户端
        switch (request.type) {
            ......
            case OpCode.create: {
                lastOp = "CREA";
                rsp = new CreateResponse(rc.path);
                err = Code.get(rc.err);
                break;
            }
            ......
        }
        ......
        cnxn.sendResponse(hdr, rsp, "response");
        ......
    }
```
处理create指令时就是通过DataTree.java中的`processTxn()`方法根据请求类型创建节点`createNode()`。
```java
    public String createNode(String path, byte data[], List<ACL> acl,
            long ephemeralOwner, int parentCVersion, long zxid, long time)
            throws KeeperException.NoNodeException,
            KeeperException.NodeExistsException {
        int lastSlash = path.lastIndexOf('/');
        String parentName = path.substring(0, lastSlash);
        String childName = path.substring(lastSlash + 1);
        // 记录节点信息
        StatPersisted stat = new StatPersisted();
        stat.setCtime(time);
        stat.setMtime(time);
        stat.setCzxid(zxid);
        stat.setMzxid(zxid);
        stat.setPzxid(zxid);
        stat.setVersion(0);
        stat.setAversion(0);
        stat.setEphemeralOwner(ephemeralOwner);
        // 获取父节点
        DataNode parent = nodes.get(parentName);
        if (parent == null) {
            throw new KeeperException.NoNodeException();
        }
        synchronized (parent) {
            Set<String> children = parent.getChildren();
            if (children.contains(childName)) {
                throw new KeeperException.NodeExistsException();
            }
            
            if (parentCVersion == -1) {
                parentCVersion = parent.stat.getCversion();
                parentCVersion++;
            }    
            parent.stat.setCversion(parentCVersion);
            parent.stat.setPzxid(zxid);
            Long longval = aclCache.convertAcls(acl);
            // 创建节点，可见每个节点其实都是DataNode实例对象
            DataNode child = new DataNode(parent, data, longval, stat);
            parent.addChild(childName);
            // 保存下来
            nodes.put(path, child);
            if (ephemeralOwner != 0) {
                HashSet<String> list = ephemerals.get(ephemeralOwner);
                if (list == null) {
                    list = new HashSet<String>();
                    ephemerals.put(ephemeralOwner, list);
                }
                synchronized (list) {
                    list.add(path);
                }
            }
        }
        // now check if its one of the zookeeper node child
        if (parentName.startsWith(quotaZookeeper)) {
            // now check if its the limit node
            if (Quotas.limitNode.equals(childName)) {
                // this is the limit node
                // get the parent and add it to the trie
                pTrie.addPath(parentName.substring(quotaZookeeper.length()));
            }
            if (Quotas.statNode.equals(childName)) {
                updateQuotaForPath(parentName
                        .substring(quotaZookeeper.length()));
            }
        }
        // also check to update the quotas for this node
        String lastPrefix;
        if((lastPrefix = getMaxPrefixWithQuota(path)) != null) {
            // ok we have some match and need to update
            updateCount(lastPrefix, 1);
            updateBytes(lastPrefix, data == null ? 0 : data.length);
        }
        // 触发Watcher，返回NodeCreated事件
        dataWatches.triggerWatch(path, Event.EventType.NodeCreated);
        childWatches.triggerWatch(parentName.equals("") ? "/" : parentName,
                Event.EventType.NodeChildrenChanged);
        return path;
    }
```
**总结一下，zookeeper收到客户端的请求指令后会先对请求校验，然后更新事务，可能会生成快照，最后写入内存中。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**