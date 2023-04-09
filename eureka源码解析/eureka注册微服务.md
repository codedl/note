---
title: eureka注册微服务
tags:
notebook: eureka源码解析
---
在自动配置类EurekaClientAutoConfiguration.java中会实例化CloudEurekaClient，此时会创建三个执行器，用来调度任务。先看向eureka注册微服务的任务。
```java
    public void run() {
        try {
            discoveryClient.refreshInstanceInfo();

            Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
            if (dirtyTimestamp != null) {
                discoveryClient.register();
                instanceInfo.unsetIsDirty(dirtyTimestamp);
            }
        } catch (Throwable t) {
            logger.warn("There was a problem with the instance info replicator", t);
        } finally {
            Future next = scheduler.schedule(this, replicationIntervalSeconds, TimeUnit.SECONDS);
            scheduledPeriodicRef.set(next);
        }
    }
```
首先在启动任务的时候`instanceInfo.setIsDirty()`，这样才会发起注册请求。
1. 判断微服务实例是否发生变化，做法是先获取注册的微服务的ip地址，再从配置中获取新的ip地址，将两者比较判断是否发生变化。
```java
    public void refreshDataCenterInfoIfRequired() {
        String existingAddress = instanceInfo.getHostName();

        String existingSpotInstanceAction = null;
        if (instanceInfo.getDataCenterInfo() instanceof AmazonInfo) {
            existingSpotInstanceAction = ((AmazonInfo) instanceInfo.getDataCenterInfo()).get(AmazonInfo.MetaDataKey.spotInstanceAction);
        }

        String newAddress;
        if (config instanceof RefreshableInstanceConfig) {
            // Refresh data center info, and return up to date address
            newAddress = ((RefreshableInstanceConfig) config).resolveDefaultAddress(true);
        } else {
            newAddress = config.getHostName(true);
        }
        String newIp = config.getIpAddress();

        if (newAddress != null && !newAddress.equals(existingAddress)) {
            logger.warn("The address changed from : {} => {}", existingAddress, newAddress);
            updateInstanceInfo(newAddress, newIp);
        }

        if (config.getDataCenterInfo() instanceof AmazonInfo) {
            String newSpotInstanceAction = ((AmazonInfo) config.getDataCenterInfo()).get(AmazonInfo.MetaDataKey.spotInstanceAction);
            if (newSpotInstanceAction != null && !newSpotInstanceAction.equals(existingSpotInstanceAction)) {
                logger.info(String.format("The spot instance termination action changed from: %s => %s",
                        existingSpotInstanceAction,
                        newSpotInstanceAction));
                updateInstanceInfo(null , null );
            }
        }        
    }
```
2. 更新续约信息：先拿到初始化的时候生成的续约信息LeaseInfo，再从EurekaInstanceConfigBean配置中中获取leaseExpirationDurationInSeconds和leaseRenewalIntervalInSeconds，比较配置中的值和初始化的值是否相同，不同则更新
```java
    public void refreshLeaseInfoIfRequired() {
        LeaseInfo leaseInfo = instanceInfo.getLeaseInfo();
        if (leaseInfo == null) {
            return;
        }
        int currentLeaseDuration = config.getLeaseExpirationDurationInSeconds();
        int currentLeaseRenewal = config.getLeaseRenewalIntervalInSeconds();
        if (leaseInfo.getDurationInSecs() != currentLeaseDuration || leaseInfo.getRenewalIntervalInSecs() != currentLeaseRenewal) {
            LeaseInfo newLeaseInfo = LeaseInfo.Builder.newBuilder()
                    .setRenewalIntervalInSecs(currentLeaseRenewal)
                    .setDurationInSecs(currentLeaseDuration)
                    .build();
            instanceInfo.setLeaseInfo(newLeaseInfo);
            instanceInfo.setIsDirty();
        }
    }
```
3. 判断实例的状态是否发生改变，如果发生改变则通知监听器
```java
    public synchronized void setInstanceStatus(InstanceStatus status) {
        InstanceStatus next = instanceStatusMapper.map(status);
        if (next == null) {
            return;
        }

        InstanceStatus prev = instanceInfo.setStatus(next);
        if (prev != null) {
            for (StatusChangeListener listener : listeners.values()) {
                try {
                    listener.notify(new StatusChangeEvent(prev, next));
                } catch (Exception e) {
                    logger.warn("failed to notify listener: {}", listener.getId(), e);
                }
            }
        }
    }
```
以上三个有任意一个发生改变则需要向eureka重新注册微服务，现在开始看怎么向eureka注册微服务的。其实也不麻烦，就是先生成要调用的url，再发送post请求，传的参数是InstanceInfo(InstanceInfo是在实例化ApplicationInfoManager的时候生成的)也就是把InstanceInfo发给服务端。
```java
    public EurekaHttpResponse<Void> register(InstanceInfo info) {
        String urlPath = "apps/" + info.getAppName();
        ClientResponse response = null;
        try {
            Builder resourceBuilder = jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
            addExtraHeaders(resourceBuilder);
            response = resourceBuilder
                    .header("Accept-Encoding", "gzip")
                    .type(MediaType.APPLICATION_JSON_TYPE)
                    .accept(MediaType.APPLICATION_JSON)
                    .post(ClientResponse.class, info);
            return anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
        } finally {
            if (logger.isDebugEnabled()) {
                logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                        response == null ? "N/A" : response.getStatus());
            }
            if (response != null) {
                response.close();
            }
        }
    }
```
注册完成之后清除dirty信息`instanceInfo.unsetIsDirty(dirtyTimestamp)`，下次在微服务实例没有变化时就不会注册了。

客户端注册微服务的过程看完，再看下服务端怎么处理注册请求的，服务端是在ApplicationResource类的addInstance()方法中处理注册请求。首先就是对客户端发送的微服务实例信息进行校验，判断ip地址，应用名等是不是都是存在的，然后就是AbstractInstanceRegistry.java类中register()方法处理注册。
```java
// AbstractInstanceRegistry.java
// registrant:要注册的实例信息
// leaseDuration:eureka.instance.lease-expiration-duration-in-seconds配置项的值
// isReplication:是否在集群的节点中同步
    public void register(InstanceInfo registrant, int leaseDuration, boolean isReplication) {
        try {
            read.lock();
            // 从ConcurrentHashMap中获取应用名对应的实例信息
            // 一个应用可能对应多个微服务实例
            Map<String, Lease<InstanceInfo>> gMap = registry.get(registrant.getAppName());
            REGISTER.increment(isReplication);
            // 没有的话就新建一个
            if (gMap == null) {
                // 使用ConcurrentHashMap保存实例，key为实例id，value为对应的续约信息
                final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new ConcurrentHashMap<String, Lease<InstanceInfo>>();
                gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
                if (gMap == null) {
                    gMap = gNewMap;
                }
            }
            Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
            // Retain the last dirty timestamp without overwriting it, if there is already a lease
            if (existingLease != null && (existingLease.getHolder() != null)) {
                // getLastDirtyTimestamp()返回的是实例变化时的时间
                // existingLastDirtyTimestamp是服务端保存的时间
                Long existingLastDirtyTimestamp = existingLease.getHolder().getLastDirtyTimestamp();
                // registrationLastDirtyTimestamp是客户端发过来的时间
                Long registrationLastDirtyTimestamp = registrant.getLastDirtyTimestamp();
                logger.debug("Existing lease found (existing={}, provided={}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);

                // this is a > instead of a >= because if the timestamps are equal, we still take the remote transmitted
                // InstanceInfo instead of the server local copy.
                // 服务端保存的实例比客户端新
                if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                    logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}", existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                    logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                    // 保留最新的服务实例
                    registrant = existingLease.getHolder();
                }
            } else {
                // The lease does not exist and hence it is a new registration
                synchronized (lock) {
                    if (this.expectedNumberOfClientsSendingRenews > 0) {
                        // Since the client wants to register it, increase the number of clients sending renews
                        this.expectedNumberOfClientsSendingRenews = this.expectedNumberOfClientsSendingRenews + 1;
                        updateRenewsPerMinThreshold();
                    }
                }
                logger.debug("No previous lease information found; it is new registration");
            }
            // 保存注册的实例信息及续约信息
            Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant, leaseDuration);
            if (existingLease != null) {
                lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
            }
            gMap.put(registrant.getId(), lease);
            synchronized (recentRegisteredQueue) {
                recentRegisteredQueue.add(new Pair<Long, String>(
                        System.currentTimeMillis(),
                        registrant.getAppName() + "(" + registrant.getId() + ")"));
            }
            // This is where the initial state transfer of overridden status happens
            if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
                logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                                + "overrides", registrant.getOverriddenStatus(), registrant.getId());
                if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                    logger.info("Not found overridden id {} and hence adding it", registrant.getId());
                    overriddenInstanceStatusMap.put(registrant.getId(), registrant.getOverriddenStatus());
                }
            }
            InstanceStatus overriddenStatusFromMap = overriddenInstanceStatusMap.get(registrant.getId());
            if (overriddenStatusFromMap != null) {
                logger.info("Storing overridden status {} from map", overriddenStatusFromMap);
                registrant.setOverriddenStatus(overriddenStatusFromMap);
            }

            // Set the status based on the overridden status rules
            InstanceStatus overriddenInstanceStatus = getOverriddenInstanceStatus(registrant, existingLease, isReplication);
            registrant.setStatusWithoutDirty(overriddenInstanceStatus);

            // If the lease is registered with UP status, set lease service up timestamp
            if (InstanceStatus.UP.equals(registrant.getStatus())) {
                lease.serviceUp();
            }
            // 新增操作
            registrant.setActionType(ActionType.ADDED);
            recentlyChangedQueue.add(new RecentlyChangedItem(lease));
            registrant.setLastUpdatedTimestamp();
            // 清除缓存
            invalidateCache(registrant.getAppName(), registrant.getVIPAddress(), registrant.getSecureVipAddress());
            logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(), registrant.getStatus(), isReplication);
        } finally {
            read.unlock();
        }
    }
```
完成微服务的注册后，如果是客户端还需要在集群中同步注册信息。当isReplication为true时表示这是节点的同步信息，不用再次同步了。遍历所有节点，剔除本身节点，向其他节点进行注册。
```java
// PeerAwareInstanceRegistryImpl.java
    private void replicateToPeers(Action action, String appName, String id,
                                  InstanceInfo info /* optional */,
                                  InstanceStatus newStatus /* optional */, boolean isReplication) {
        Stopwatch tracer = action.getTimer().start();
        try {
            if (isReplication) {
                numberOfReplicationsLastMin.increment();
            }
            // If it is a replication already, do not replicate again as this will create a poison replication
            if (peerEurekaNodes == Collections.EMPTY_LIST || isReplication) {
                return;
            }

            for (final PeerEurekaNode node : peerEurekaNodes.getPeerEurekaNodes()) {
                // If the url represents this host, do not replicate to yourself.
                if (peerEurekaNodes.isThisMyUrl(node.getServiceUrl())) {
                    continue;
                }
                replicateInstanceActionsToPeers(action, appName, id, info, newStatus, node);
            }
        } finally {
            tracer.stop();
        }
    }
```
其实就是通过batchingDispatcher进行处理的，本质就是向队列中添加任务，让其他线程从队列中将任务取出再进行处理。
```java
    public void register(final InstanceInfo info) throws Exception {
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        batchingDispatcher.process(
                taskId("register", info),
                new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                    public EurekaHttpResponse<Void> execute() {
                        return replicationClient.register(info);
                    }
                },
                expiryTime
        );
    }
```
**总结一下，首先客户端生成实例信息InstanceInfo，向服务端post过去，服务端保存下来并创建续约信息，然后定时更新**