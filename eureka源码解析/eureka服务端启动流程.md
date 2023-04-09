---
title: eureka服务端启动流程
tags:
notebook: eureka源码解析
---
现在开始看eureka服务端是怎么启动的。首先从主类上标记的注解@EnableEurekaServer开始分析，其实没啥，就是@Import导入了EurekaServerMarkerConfiguration组件
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(EurekaServerMarkerConfiguration.class)
public @interface EnableEurekaServer {

}
```
EurekaServerMarkerConfiguration组件创建了一个Marker对象，顾名思义就是用来标记的，其实就用标记当前应用为eureka服务端应用。
```java
@Configuration
public class EurekaServerMarkerConfiguration {

	@Bean
	public Marker eurekaServerMarkerBean() {
		return new Marker();
	}

	class Marker {
	}
}
```
不要忘了，springboot可以通过spi机制进行定制，所以看下spring.factories文件，这里我们看到定义了自动配置类EurekaServerAutoConfiguration，接下来分析这个类即可。
```java
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.cloud.netflix.eureka.server.EurekaServerAutoConfiguration
```
通过类的声明，我们可以看到导入了EurekaServerInitializerConfiguration组件，刚才创建的Marker Bean也派上用场了，这里就是判断当容器中的Marker Bean存在时，才会对EurekaServerAutoConfiguration进行解析。
```java
@Configuration
@Import(EurekaServerInitializerConfiguration.class)
@ConditionalOnBean(EurekaServerMarkerConfiguration.Marker.class)
@EnableConfigurationProperties({ EurekaDashboardProperties.class,
		InstanceRegistryProperties.class })
@PropertySource("classpath:/eureka/server.properties")
public class EurekaServerAutoConfiguration extends WebMvcConfigurerAdapter {
    ......
	@Bean
	@ConditionalOnProperty(prefix = "eureka.dashboard", name = "enabled", matchIfMissing = true)
	public EurekaController eurekaController() {
		return new EurekaController(this.applicationInfoManager);
	}

    @Bean
	public PeerAwareInstanceRegistry peerAwareInstanceRegistry(
			ServerCodecs serverCodecs) {
		this.eurekaClient.getApplications(); // force initialization
		return new InstanceRegistry(this.eurekaServerConfig, this.eurekaClientConfig,
				serverCodecs, this.eurekaClient,
				this.instanceRegistryProperties.getExpectedNumberOfClientsSendingRenews(),
				this.instanceRegistryProperties.getDefaultOpenForTrafficCount());
	}

	@Bean
	@ConditionalOnMissingBean
	public PeerEurekaNodes peerEurekaNodes(PeerAwareInstanceRegistry registry,
			ServerCodecs serverCodecs) {
		return new RefreshablePeerEurekaNodes(registry, this.eurekaServerConfig,
				this.eurekaClientConfig, serverCodecs, this.applicationInfoManager);
	}

    @Bean
	public EurekaServerContext eurekaServerContext(ServerCodecs serverCodecs,
			PeerAwareInstanceRegistry registry, PeerEurekaNodes peerEurekaNodes) {
		return new DefaultEurekaServerContext(this.eurekaServerConfig, serverCodecs,
				registry, peerEurekaNodes, this.applicationInfoManager);
	}

	@Bean
	public EurekaServerBootstrap eurekaServerBootstrap(PeerAwareInstanceRegistry registry,
			EurekaServerContext serverContext) {
		return new EurekaServerBootstrap(this.applicationInfoManager,
				this.eurekaClientConfig, this.eurekaServerConfig, registry,
				serverContext);
	}
    ......
}
```
看下这个配置类创建了哪些Bean吧，EurekaController是用来处理仪表盘的，PeerEurekaNodes和PeerAwareInstanceRegistry这是两个集群相关的Bean，EurekaServerBootstrap待会分析，在初始化上下文的时候会用到，现在分析EurekaServerContext，由于initialize()方法存在@PostConstruct注解，因此在实例化以后会被调用。
```java
    @PostConstruct
    @Override
    public void initialize() {
        logger.info("Initializing ...");
        peerEurekaNodes.start();
        try {
            registry.init(peerEurekaNodes);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
        logger.info("Initialized");
    }
```
### 创建集群
这里调用`peerEurekaNodes.start()`开启线程周期性地调用`updatePeerEurekaNodes(resolvePeerUrls());`。在resolvePeerUrls()方法中，首先通过内部属性applicationInfoManager的getInfo()方法获取InstanceInfo实例对象。
```java
    protected List<String> resolvePeerUrls() {
        InstanceInfo myInfo = applicationInfoManager.getInfo();
        // clientConfig(EurekaClientConfigBean)中的属性会跟"eureka.client"开头的配置项绑定
        // 这里其实是通过EurekaClientConfigBean的region到availabilityZones(HashMap)中获取对应的value，记作zone
        // zone的默认值是defaultZone
        String zone = InstanceInfo.getZone(clientConfig.getAvailabilityZones(clientConfig.getRegion()), myInfo);
        // 通过EurekaClientConfigBean的useDnsForFetchingServiceUrls判断副本来源，默认都是从配置中获取
        // 从属性serviceUrl(HashMap->service-url配置项)中根据zone的值(默认为defaultZone)获取urls
        // 将这些分区副本的url存到ArrayList<String>中
        List<String> replicaUrls = EndpointUtils
                .getDiscoveryServiceUrls(clientConfig, zone, new EndpointUtils.InstanceInfoBasedUrlRandomizer(myInfo));
        // 将本节点从现有的分区中移除
        // 做法isThisMyUrl是从url中获取hostname，再从InstanceInfo中获取本地hostname，把两者进行比较，
        // 相同返回true，进行remove
        int idx = 0;
        while (idx < replicaUrls.size()) {
            if (isThisMyUrl(replicaUrls.get(idx))) {
                replicaUrls.remove(idx);
            } else {
                idx++;
            }
        }
        return replicaUrls;
    }
```
看下InstanceInfo是怎么被实例化的，springboot的spi定制化太强大了，可以轻而易举地整合其他框架，这里也是通过这个实现的，还是EurekaClientAutoConfiguration，这里通过eurekaApplicationInfoManager()创建InstanceInfo的实例对象，然后注入到ApplicationInfoManager里面去的，再来看InstanceInfo怎么被实例化的。
```java
public class EurekaClientAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean(value = ApplicationInfoManager.class, search = SearchStrategy.CURRENT)
    public ApplicationInfoManager eurekaApplicationInfoManager(
            EurekaInstanceConfig config) {
        InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
        return new ApplicationInfoManager(config, instanceInfo);
    }
}
```
注入eurekaApplicationInfoManager()方法的EurekaInstanceConfig默认是EurekaInstanceConfigBean实现的。到这里可以把流程梳理一下。首先创建EurekaInstanceConfigBean实例Bean，而这个实例Bean中的属性跟"eureka.instance"开头的配置项绑定的。接着使用这个配置创建InstanceInfo实例对象，表示当前节点的实例信息，再注入到ApplicationInfoManager中。那我们现在就知道resolvePeerUrls()方法的InstanceInfo怎么来得了。再继续往下面看，接下来就是通过配置项获取集群中所有节点的url，作为updatePeerEurekaNodes()方法参数。接下来总算可以分析updatePeerEurekaNodes()方法，前面经过一系列的Bean注入和配置项获取，总算到updatePeerEurekaNodes()方法调用。
```java
    protected void updatePeerEurekaNodes(List<String> newPeerUrls) {
        if (newPeerUrls.isEmpty()) {
            logger.warn("The replica size seems to be empty. Check the route 53 DNS Registry");
            return;
        }

        Set<String> toShutdown = new HashSet<>(peerEurekaNodeUrls);
        toShutdown.removeAll(newPeerUrls);
        Set<String> toAdd = new HashSet<>(newPeerUrls);
        toAdd.removeAll(peerEurekaNodeUrls);

        if (toShutdown.isEmpty() && toAdd.isEmpty()) { // No change
            return;
        }

        // Remove peers no long available
        List<PeerEurekaNode> newNodeList = new ArrayList<>(peerEurekaNodes);

        if (!toShutdown.isEmpty()) {
            logger.info("Removing no longer available peer nodes {}", toShutdown);
            int i = 0;
            while (i < newNodeList.size()) {
                PeerEurekaNode eurekaNode = newNodeList.get(i);
                if (toShutdown.contains(eurekaNode.getServiceUrl())) {
                    newNodeList.remove(i);
                    eurekaNode.shutDown();
                } else {
                    i++;
                }
            }
        }

        // 依次添加集群中其他节点
        if (!toAdd.isEmpty()) {
            logger.info("Adding new peer nodes {}", toAdd);
            for (String peerUrl : toAdd) {
                newNodeList.add(createPeerEurekaNode(peerUrl));
            }
        }

        this.peerEurekaNodes = newNodeList;
        this.peerEurekaNodeUrls = new HashSet<>(newPeerUrls);
    }
```
核心就是createPeerEurekaNode(peerUrl)，根据集群中url创建节点PeerEurekaNode，都看到这里了，死磕到底吧，看下createPeerEurekaNode(peerUrl)方法。
```java
    protected PeerEurekaNode createPeerEurekaNode(String peerEurekaNodeUrl) {
        HttpReplicationClient replicationClient = JerseyReplicationClient.createReplicationClient(serverConfig, serverCodecs, peerEurekaNodeUrl);
        String targetHost = hostFromUrl(peerEurekaNodeUrl);
        if (targetHost == null) {
            targetHost = "host";
        }
        return new PeerEurekaNode(registry, targetHost, peerEurekaNodeUrl, replicationClient, serverConfig);
    }
```
peerEurekaNodeUrl就是前面获取的集群中其他节点的url，这里先创建一个副本客户端HttpReplicationClient实例对象，再对PeerEurekaNode进行实例化，此时会创建任务，在任务里通过Jersey框架给集群中其他节点发送http请求，总算获取到集群中其他节点信息了。
### 任务处理模型
实例化之后可以通过`batchingDispatcher.process();`对任务进行处理，本质是将任务添加到队列acceptorQueue(LinkedBlockingQueue)。
```java
    void process(ID id, T task, long expiryTime) {
        acceptorQueue.add(new TaskHolder<ID, T>(id, task, expiryTime));
        acceptedTasks++;
    }
```
然后在AcceptorExecutor的AcceptorRunner线程run()中周期性处理这些任务
```java
        public void run() {
            long scheduleTime = 0;
            while (!isShutdown.get()) {
                try {
                    // 从队列中取出任务
                    drainInputQueues();

                    int totalItems = processingOrder.size();
                    // 如果没有过期就进行处理
                    long now = System.currentTimeMillis();
                    if (scheduleTime < now) {
                        scheduleTime = now + trafficShaper.transmissionDelay();
                    }
                    if (scheduleTime <= now) {
                        assignBatchWork();
                        assignSingleItemWork();
                    }

                    // If no worker is requesting data or there is a delay injected by the traffic shaper,
                    // sleep for some time to avoid tight loop.
                    if (totalItems == processingOrder.size()) {
                        Thread.sleep(10);
                    }
                } catch (InterruptedException ex) {
                    // Ignore
                } catch (Throwable e) {
                    // Safe-guard, so we never exit this loop in an uncontrolled way.
                    logger.warn("Discovery AcceptorThread error", e);
                }
            }
        }
```
其实就是从acceptorQueue中poll()出任务，添加到pendingTasks(HashMap)中，key是任务id，value是任务。
```java
        private void drainInputQueues() throws InterruptedException {
            do {
                drainReprocessQueue();
                drainAcceptorQueue();

                if (!isShutdown.get()) {
                    // 队列全空时延时等待任务加入
                    if (reprocessQueue.isEmpty() && acceptorQueue.isEmpty() && pendingTasks.isEmpty()) {
                        TaskHolder<ID, T> taskHolder = acceptorQueue.poll(10, TimeUnit.MILLISECONDS);
                        if (taskHolder != null) {
                            appendTaskHolder(taskHolder);
                        }
                    }
                }
            } while (!reprocessQueue.isEmpty() || !acceptorQueue.isEmpty() || pendingTasks.isEmpty());
        }

        private void drainAcceptorQueue() {
            while (!acceptorQueue.isEmpty()) {
                appendTaskHolder(acceptorQueue.poll());
            }
        }

        private void appendTaskHolder(TaskHolder<ID, T> taskHolder) {
            if (isFull()) {
                pendingTasks.remove(processingOrder.poll());
                queueOverflows++;
            }
            TaskHolder<ID, T> previousTask = pendingTasks.put(taskHolder.getId(), taskHolder);
            if (previousTask == null) {
                processingOrder.add(taskHolder.getId());
            } else {
                overriddenTasks++;
            }
        }
```
还是在本线程中收集任务，将刚才加入队列的任务都添加batchWorkQueue中，这些所有要处理的任务都在batchWorkQueue里了。
```java
        void assignBatchWork() {
            if (hasEnoughTasksForNextBatch()) {
                if (batchWorkRequests.tryAcquire(1)) {
                    long now = System.currentTimeMillis();
                    int len = Math.min(maxBatchingSize, processingOrder.size());
                    List<TaskHolder<ID, T>> holders = new ArrayList<>(len);
                    while (holders.size() < len && !processingOrder.isEmpty()) {
                        ID id = processingOrder.poll();
                        TaskHolder<ID, T> holder = pendingTasks.remove(id);
                        if (holder.getExpiryTime() > now) {
                            holders.add(holder);
                        } else {
                            expiredTasks++;
                        }
                    }
                    if (holders.isEmpty()) {
                        batchWorkRequests.release();
                    } else {
                        batchSizeMetric.record(holders.size(), TimeUnit.MILLISECONDS);
                        batchWorkQueue.add(holders);
                    }
                }
            }
        }
```
接下是就是真正的处理这些任务`processor.process(tasks)`了
```java
        public void run() {
            try {
                while (!isShutdown.get()) {
                    // 将保存批量任务的holders从队列batchWorkQueue中取出来
                    List<TaskHolder<ID, T>> holders = getWork();
                    metrics.registerExpiryTimes(holders);

                    List<T> tasks = getTasksOf(holders);
                    // 真正的处理在这里
                    ProcessingResult result = processor.process(tasks);
                    switch (result) {
                        case Success:
                            break;
                        case Congestion:
                        case TransientError:
                            taskDispatcher.reprocess(holders, result);
                            break;
                        case PermanentError:
                            logger.warn("Discarding {} tasks of {} due to permanent error", holders.size(), workerName);
                    }
                    metrics.registerTaskResult(result, tasks.size());
                }
            } catch (InterruptedException e) {
                // Ignore
            } catch (Throwable e) {
                // Safe-guard, so we never exit this loop in an uncontrolled way.
                logger.warn("Discovery WorkerThread error", e);
            }
        }
```
到这里对eureka任务调度模型总结下：首先batchingDispatcher将要处理的任务添加到acceptorQueue队列，然后线程从acceptorQueue将要处理的任务取出来缓存到pendingTasks，最后如果可以分发任务，就将任务合并到holders(ArrayList)，然后在另外一个线程中批量处理`processor.process(tasks)`，其实这个模型设计得很好的，工作中可以参考下。

现在我们还在EurekaServerContext的initialize()方法，看看注册初始化做的事情`registry.init(peerEurekaNodes);`
```java
    public void init(PeerEurekaNodes peerEurekaNodes) throws Exception {
        // 定时刷新lastBucket值
        this.numberOfReplicationsLastMin.start();
        this.peerEurekaNodes = peerEurekaNodes;
        // 实例化响应缓存
        initializedResponseCache();
        // 开启自动续约的任务
        scheduleRenewalThresholdUpdateTask();
        initRemoteRegionRegistry();

        try {
            Monitors.registerObject(this);
        } catch (Throwable e) {
            logger.warn("Cannot register the JMX monitor for the InstanceRegistry :", e);
        }
    }
```
到这里为止EurekaServerContext的实例化彻底完成了，意味着eureka服务上下文创建好了。

再看下服务端启动的最后一步，由于EurekaServerInitializerConfiguration.java实现了SmartLifecycle接口，所在start()会被spring调用，此时EurekaServerBootstrap.java的contextInitialized()会被调用。其实主要就做了两件事，先把集群中其他节点的实例信息同步过来`this.registry.syncUp()`，再更新每分钟续约数，开个自动续约的任务。
### 服务剔除任务
服务剔除任务是在AbstractInstanceRegistry.java的postInit()方法中开启的，本质是定时执行evict()方法。
```java
    public void evict(long additionalLeaseMs) {
        logger.debug("Running the evict task");
        // 自我保护机制，如果开启了自我保护机制，并且每分钟续约数小于期望数numberOfRenewsPerMinThreshold
        // 就不会剔除服务了
        if (!isLeaseExpirationEnabled()) {
            logger.debug("DS: lease expiration is currently disabled.");
            return;
        }

        // 找到过期的实例
        List<Lease<InstanceInfo>> expiredLeases = new ArrayList<>();
        for (Entry<String, Map<String, Lease<InstanceInfo>>> groupEntry : registry.entrySet()) {
            Map<String, Lease<InstanceInfo>> leaseMap = groupEntry.getValue();
            if (leaseMap != null) {
                for (Entry<String, Lease<InstanceInfo>> leaseEntry : leaseMap.entrySet()) {
                    Lease<InstanceInfo> lease = leaseEntry.getValue();
                    if (lease.isExpired(additionalLeaseMs) && lease.getHolder() != null) {
                        expiredLeases.add(lease);
                    }
                }
            }
        }

        // To compensate for GC pauses or drifting local time, we need to use current registry size as a base for
        // triggering self-preservation. Without that we would wipe out full registry.
        int registrySize = (int) getLocalRegistrySize();
        int registrySizeThreshold = (int) (registrySize * serverConfig.getRenewalPercentThreshold());
        int evictionLimit = registrySize - registrySizeThreshold;
        // 随机清除，如果按顺序清除得话可能会清除掉整个应用
        int toEvict = Math.min(expiredLeases.size(), evictionLimit);
        if (toEvict > 0) {
            logger.info("Evicting {} items (expired={}, evictionLimit={})", toEvict, expiredLeases.size(), evictionLimit);

            Random random = new Random(System.currentTimeMillis());
            for (int i = 0; i < toEvict; i++) {
                // Pick a random item (Knuth shuffle algorithm)
                int next = i + random.nextInt(expiredLeases.size() - i);
                Collections.swap(expiredLeases, i, next);
                Lease<InstanceInfo> lease = expiredLeases.get(i);

                String appName = lease.getHolder().getAppName();
                String id = lease.getHolder().getId();
                EXPIRED.increment();
                logger.warn("DS: Registry: expired lease for {}/{}", appName, id);
                internalCancel(appName, id, false);
            }
        }
    }
```
这就是eureka的自我保护机制，如果开启了自我保护机制，判断最近一分钟的心跳数是否大于期望收到的心跳数，如果小于期望值得话，说明有服务不在线，可能是重启或者宕机就不会剔除了，等他上线再说。
```java
    @Override
    public boolean isLeaseExpirationEnabled() {
        if (!isSelfPreservationModeEnabled()) {
            // The self preservation mode is disabled, hence allowing the instances to expire.
            return true;
        }
        return numberOfRenewsPerMinThreshold > 0 && getNumOfRenewsInLastMin() > numberOfRenewsPerMinThreshold;
    }
```
evictionTimestamp是服务过期得时间戳，如果有值说明服务过期；lastUpdateTimestamp是最近一次得续约时间(心跳包发送得时间)，duration为续约时间间隔，如果`System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs))`，说明到当前时间为止还没有收到续约心跳，则任务过期。
```java
    public boolean isExpired(long additionalLeaseMs) {
        return (evictionTimestamp > 0 || System.currentTimeMillis() > (lastUpdateTimestamp + duration + additionalLeaseMs));
    }
```
剔除服务是在internalCancel()方法中，将要剔除得服务从registry的Map中移除remove(id)掉，然后清除缓存。
```java
    protected boolean internalCancel(String appName, String id, boolean isReplication) {
        try {
            read.lock();
            CANCEL.increment(isReplication);
            Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
            Lease<InstanceInfo> leaseToCancel = null;
            if (gMap != null) {
                leaseToCancel = gMap.remove(id);
            }
            synchronized (recentCanceledQueue) {
                recentCanceledQueue.add(new Pair<Long, String>(System.currentTimeMillis(), appName + "(" + id + ")"));
            }
            InstanceStatus instanceStatus = overriddenInstanceStatusMap.remove(id);
            if (instanceStatus != null) {
                logger.debug("Removed instance id {} from the overridden map which has value {}", id, instanceStatus.name());
            }
            if (leaseToCancel == null) {
                CANCEL_NOT_FOUND.increment(isReplication);
                logger.warn("DS: Registry: cancel failed because Lease is not registered for: {}/{}", appName, id);
                return false;
            } else {
                leaseToCancel.cancel();
                InstanceInfo instanceInfo = leaseToCancel.getHolder();
                String vip = null;
                String svip = null;
                if (instanceInfo != null) {
                    instanceInfo.setActionType(ActionType.DELETED);
                    recentlyChangedQueue.add(new RecentlyChangedItem(leaseToCancel));
                    instanceInfo.setLastUpdatedTimestamp();
                    vip = instanceInfo.getVIPAddress();
                    svip = instanceInfo.getSecureVipAddress();
                }
                invalidateCache(appName, vip, svip);
                logger.info("Cancelled instance {}/{} (replication={})", appName, id, isReplication);
                return true;
            }
        } finally {
            read.unlock();
        }
    }
```
现在总结下eureka服务端启动的过程：首先从配置文件中获取集群中每个节点的信息，然后创建定时任务周期性发心跳包给其他节点，然后从其他节点将实例信息同步过来。
