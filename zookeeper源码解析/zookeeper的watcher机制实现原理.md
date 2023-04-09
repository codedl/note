我们都知道在zookeeper中存在watcher机制，查询服务端节点数据时可以进行监听，当服务端的数据发生改变时通知客户端，客户端注册的watcher就会被调用，现在看下这个过程。
### 客户端注册Watcher
从ZooKeeper类的getData()方法开始看起，watcher为传入的监听器，这里会将watcher和监听的路径封装成WatchRegistration，WatchRegistration会被保存到发送给服务的包Packet中，在发送指令给服务器时会进行处理，将Watcher注册到内部ZKWatchManager中。
```java
    public byte[] getData(final String path, Watcher watcher, Stat stat) throws KeeperException, InterruptedException
     {
        final String clientPath = path;
        PathUtils.validatePath(clientPath);//校验路径

        // 实例化DataWatchRegistration
        WatchRegistration wcb = null;
        if (watcher != null) {
            wcb = new DataWatchRegistration(watcher, clientPath);
        }

        final String serverPath = prependChroot(clientPath);
        // 创建请求头，设置请求类型为 ZooDefs.OpCode.getData
        RequestHeader h = new RequestHeader();
        h.setType(ZooDefs.OpCode.getData);
        // 创建请求，设置获取数据的路径
        GetDataRequest request = new GetDataRequest();
        request.setPath(serverPath);
        // 是否存在监听器watcher
        request.setWatch(watcher != null);
        // 创建响应，用来接收服务端的数据
        GetDataResponse response = new GetDataResponse();
        // WatchRegistration对象保存在包Packet中,在后面进行处理
        ReplyHeader r = cnxn.submitRequest(h, request, response, wcb);
        if (r.getErr() != 0) {
            throw KeeperException.create(KeeperException.Code.get(r.getErr()),
                    clientPath);
        }
        // 将服务端的stat拷贝到客户端传的stat
        if (stat != null) {
            DataTree.copyStat(response.getStat(), stat);
        }
        // 返回从服务端获取的数据
        return response.getData();
    }
```
在`submitRequest()`中会调用`queuePacket()`对`Packet`进行实例化，用来封装发送给服务器的数据，并加入队列发送给服务端。当队列中的包被发送到服务器时，ClientCnxn.java#finishPacket()被调用，用来注册监听器，注册监听器的逻辑在方法ZooKeeper$WatchRegistration#register()中。首先获取用来保存监听器的map，key为路径，value为监听器集合，将getData()中添加的watcher保存这个map即可。
```java
        public void register(int rc) {
            if (shouldAddWatch(rc)) {
                // DataWatchRegistration#getWatches => Map<String, Set<Watcher>> dataWatches
                // key为路径，value为Watcher集合
                Map<String, Set<Watcher>> watches = getWatches(rc);
                synchronized(watches) {
                    Set<Watcher> watchers = watches.get(clientPath);
                    if (watchers == null) {
                        watchers = new HashSet<Watcher>();
                        watches.put(clientPath, watchers);
                    }
                    // watcher是调用 ZooKeeper#getData 时传的
                    watchers.add(watcher);
                }
            }
        }
```
### 服务端处理getData请求
服务器的请求处理器链(PrepRequestProcessor->SyncRequestProcessor->FinalRequestProcessor)会依次处理来自客户端的请求。PrepRequestProcessor只是做了会话的校验，没有实质的处理，SyncRequestProcessor会进行日志回滚和生成快照，那么最终应该是在FinalRequestProcessor处理器中进行处理的。首先将客户端的字节流反序列化成GetDataRequest对象，然后通过`getData()`为获取数据的节点设置Watch，其中cnxn是在NIOServerCnxn的readRequest()中传给Request的，就是NIOServerCnxn对象，而这个对象是跟客户端socket关联的。NIOServerCnxn实现了Watcher接口，所以可以被注册。NIOServerCnxn是在客户端与服务器建立连接时创建的，内部维护了客户端与服务端之间的连接信息。
```java
    public void processRequest(Request request) {
        ......
        switch (request.type) {
            case OpCode.getData: {
                lastOp = "GETD";
                //将客户端发过来的请求数据反序列化到getDataRequest
                GetDataRequest getDataRequest = new GetDataRequest();
                ByteBufferInputStream.byteBuffer2Record(request.request,
                        getDataRequest);
                DataNode n = zks.getZKDatabase().getNode(getDataRequest.getPath());
                if (n == null) {
                    throw new KeeperException.NoNodeException();
                }
                PrepRequestProcessor.checkACL(zks, zks.getZKDatabase().aclForNode(n),
                        ZooDefs.Perms.READ,
                        request.authInfo);
                Stat stat = new Stat();
                // 设置监听器watch
                byte b[] = zks.getZKDatabase().getData(getDataRequest.getPath(), stat,
                        getDataRequest.getWatch() ? cnxn : null);
                rsp = new GetDataResponse(b, stat);
                break;
            }
        }
        ......
    }
```
在getData()中将watcher跟path注册到dataWatches(WatchManager)的watchTable(HashMap<String, HashSet<Watcher>>)。watchTable的key为监听的路径，value为Watcher集合，最终将Watcher保存到这个集合。
```java
    public byte[] getData(String path, Stat stat, Watcher watcher)
            throws KeeperException.NoNodeException {
        DataNode n = nodes.get(path);
        if (n == null) {
            throw new KeeperException.NoNodeException();
        }
        synchronized (n) {
            n.copyStat(stat);
            if (watcher != null) {
                // WatchManager#addWatch
                dataWatches.addWatch(path, watcher);
            }
            return n.data;
        }
    }
    public synchronized void addWatch(String path, Watcher watcher) {
        // 将path和Watcher的映射关系保存下来
        HashSet<Watcher> list = watchTable.get(path);
        if (list == null) {
            // don't waste memory if there are few watches on a node
            // rehash when the 4th entry is added, doubling size thereafter
            // seems like a good compromise
            list = new HashSet<Watcher>(4);
            watchTable.put(path, list);
        }
        list.add(watcher);
        // 将watcher和path的映射关系保存下来
        HashSet<String> paths = watch2Paths.get(watcher);
        if (paths == null) {
            // cnxns typically have many watches, so use default cap here
            paths = new HashSet<String>();
            watch2Paths.put(watcher, paths);
        }
        paths.add(path);
    }   
```
### 服务端处理setData请求
在setData()时要把数据写道内存中，这里会更新事务。
```java
    public void processRequest(Request request) {
        if (LOG.isDebugEnabled()) {
            LOG.debug("Processing request:: " + request);
        }
        // request.addRQRec(">final");
        long traceMask = ZooTrace.CLIENT_REQUEST_TRACE_MASK;
        if (request.type == OpCode.ping) {
            traceMask = ZooTrace.SERVER_PING_TRACE_MASK;
        }
        if (LOG.isTraceEnabled()) {
            ZooTrace.logRequest(LOG, traceMask, 'E', request, "");
        }
        ProcessTxnResult rc = null;
        synchronized (zks.outstandingChanges) {
            //更新ChangeRecord，将比请求中的zxid小的ChangeRecord移除
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
            // 更新内存中节点数据
               rc = zks.processTxn(hdr, txn);
            }
            // do not add non quorum packets to the queue.
            if (Request.isQuorum(request.type)) {
                zks.getZKDatabase().addCommittedProposal(request);
            }
        }
    }
```
找到要修改的节点，将客户端发过来的数据写进去，然后触发客户端getData()时注册进来的watcher
```java
    public Stat setData(String path, byte data[], int version, long zxid,
            long time) throws KeeperException.NoNodeException {
        Stat s = new Stat();
        DataNode n = nodes.get(path);
        if (n == null) {
            throw new KeeperException.NoNodeException();
        }
        byte lastdata[] = null;
        //拷贝服务器的stat，最终发送给客户端
        synchronized (n) {
            lastdata = n.data;
            n.data = data;
            n.stat.setMtime(time);
            n.stat.setMzxid(zxid);
            n.stat.setVersion(version);
            n.copyStat(s);
        }
        // now update if the path is in a quota subtree.
        String lastPrefix;
        if((lastPrefix = getMaxPrefixWithQuota(path)) != null) {
          this.updateBytes(lastPrefix, (data == null ? 0 : data.length)
              - (lastdata == null ? 0 : lastdata.length));
        }
        //触发已经注册进来的watcher
        dataWatches.triggerWatch(path, EventType.NodeDataChanged);
        return s;
    }
```
触发Watcher时，先根据路径找到监听的Watcher，再依次遍历移除的Watcher，调用每个Watcher的process()方法。
```java
    public Set<Watcher> triggerWatch(String path, EventType type, Set<Watcher> supress) {
        WatchedEvent e = new WatchedEvent(type,
                KeeperState.SyncConnected, path);
        HashSet<Watcher> watchers;
        synchronized (this) {
            watchers = watchTable.remove(path);
            if (watchers == null || watchers.isEmpty()) {
                if (LOG.isTraceEnabled()) {
                    ZooTrace.logTraceMessage(LOG,
                            ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                            "No watchers for " + path);
                }
                return null;
            }
            for (Watcher w : watchers) {
                HashSet<String> paths = watch2Paths.get(w);
                if (paths != null) {
                    paths.remove(path);
                }
            }
        }
        for (Watcher w : watchers) {
            if (supress != null && supress.contains(w)) {
                continue;
            }
            w.process(e);
        }
        return watchers;
    }
```
客户端调用getData()时，服务器根据监听的路径注册的是NIOServerCnxn对象，因此进入这个类的process()方法调用。这里就很清楚了，创建响应头ReplyHeader，将节点的修改封装成WatcherEvent，再发送给注册Watcher的客户端。
```java
// NIOServerCnxn.java
    synchronized public void process(WatchedEvent event) {
        ReplyHeader h = new ReplyHeader(-1, -1L, 0);
        if (LOG.isTraceEnabled()) {
            ZooTrace.logTraceMessage(LOG, ZooTrace.EVENT_DELIVERY_TRACE_MASK,
                                     "Deliver event " + event + " to 0x"
                                     + Long.toHexString(this.sessionId)
                                     + " through " + this);
        }

        // Convert WatchedEvent to a type that can be sent over the wire
        WatcherEvent e = event.getWrapper();

        sendResponse(h, e, "notification");
    }
```

### 客户端触发Watcher
当服务器检测到节点的数据发生变化时就会通知客户端，客户端就会去调用watcher，看下这个调用过程吧。先用WatcherEvent表示客户端与服务端传输的事件数据，收到服务端响应后会反序列化创建WatcherEvent对象，再封装成客户端要使用的WatchedEvent对象，然后加入到事件处理线程的队列。
```java
void readResponse(ByteBuffer incomingBuffer) throws IOException {
    ......
    if (replyHdr.getXid() == -1) {
        // -1 means notification
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got notification sessionid:0x"
                + Long.toHexString(sessionId));
        }
        // 使用WatcherEvent对象接收服务端返回的事件
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
        // 接收到的事件封装成客户端可以使用的WatchedEvent对象
        WatchedEvent we = new WatchedEvent(event);
        if (LOG.isDebugEnabled()) {
            LOG.debug("Got " + we + " for sessionid 0x"
                    + Long.toHexString(sessionId));
        }
        // 加入到事件处理队列 waitingEvents(LinkedBlockingQueue<Object>)
        eventThread.queueEvent( we );
        return;
    }
}
```
首先对事件进行具体化materialize()，再封装成WatcherSetEventPair，然后加入到waitingEvents(LinkedBlockingQueue<Object>)。
```java
        public void queueEvent(WatchedEvent event) {
            if (event.getType() == EventType.None
                    && sessionState == event.getState()) {
                return;
            }
            sessionState = event.getState();

            // materialize the watchers based on the event
            // watcher由ZooKeeper$ZKWatchManager实现的
            WatcherSetEventPair pair = new WatcherSetEventPair(
                    watcher.materialize(event.getState(),
                                        event.getType(),
                                        event.getPath()),
                                        event);
            // queue the pair (watch set & event) for later processing
            // 将封装的事件和watcher加到队列中
            waitingEvents.add(pair);
        }
```
看下怎么具体化事件的，刚才我们把监听器注册到map，现在从map中移除监听器，再加入到result中，表示此次事件需要触发这些监听器。因此每次都会移除，所以我们每次都需要重新注册。
```java
        public Set<Watcher> materialize(Watcher.Event.KeeperState state,
                                        Watcher.Event.EventType type,
                                        String clientPath)
        {
            Set<Watcher> result = new HashSet<Watcher>();

            switch (type) {
            ......
            case NodeDataChanged:
            case NodeCreated:
                //移除第一次注册的watcher，添加到这次的result中
                //所以每次watcher触发后就要重新注册
                synchronized (dataWatches) {
                    addTo(dataWatches.remove(clientPath), result);
                }
                synchronized (existWatches) {
                    addTo(existWatches.remove(clientPath), result);
                }
                break;
            ......

            return result;
        }
    }
```
在事件处理线程中会调用processEvent()处理加入的事件，这就很明白了，对刚刚加入的监听器进行遍历，依次调用每个监听器的process()方法。
```java
       private void processEvent(Object event) {
          try {
              if (event instanceof WatcherSetEventPair) {
                  // each watcher will process the event
                  WatcherSetEventPair pair = (WatcherSetEventPair) event;
                  // 调用每个watcher的process方法
                  for (Watcher watcher : pair.watchers) {
                      try {
                          watcher.process(pair.event);
                      } catch (Throwable t) {
                          LOG.error("Error while calling watcher ", t);
                      }
                  }
              
              }
        }
```
**总结下，首先在获取节点数据时会客户端会保存Watcher到一个map，服务端会将自身维护的NIOServerCnxn(Watcher)保存到本地map，当setData时根据路径从map取出NIOServerCnxn，将修改信息返回给客户端，客户端收到服务端的事件通知时，从map中取出Watcher依次调用即可。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**