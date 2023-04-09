使用dubbo框架进行服务消费时，首先创建需要得配置类，然后得到关键得ReferenceConfig对象后，调用get()方法获取要调用的服务代理对象，开始对dubbo注册的服务进行消费。
```java
        /**
         * dubbo 服务消费者api形式进行服务消费
         */
        //声明应用 dubbo生态质检服务调用是基于应用的
        ApplicationConfig application = new ApplicationConfig("dubbo-refrence");
        //涉及注册中心
        RegistryConfig registryCenter = new RegistryConfig();
        registryCenter.setAddress("zookeeper://172.29.67.170:2181");

        //消费者消费
        //设置消费者全局配置
        ConsumerConfig consumerConfig = new ConsumerConfig();
        //设置默认的超时时间
        consumerConfig.setTimeout(1000*5);
        ReferenceConfig<UserService> userConfigReference = new ReferenceConfig<>();
        userConfigReference.setApplication(application);
//        List<RegistryConfig> registryConfigs = new ArrayList<>();
//        registryConfigs.add(registryCenter);
        userConfigReference.setRegistry(registryCenter);
        userConfigReference.setInterface(UserService.class);
        userConfigReference.setLoadbalance("roundrobin");
        //设置methodConfig 方法级别的dubbo参数包配置 比如方法名必填、重试次数、超时时间、负载均衡策略
        MethodConfig methodConfig = new MethodConfig();
        //方法名必填
//        methodConfig.setName("queryUserInfo");
        //超时时间
        methodConfig.setTimeout(1000 * 5);
        //重试次数
        methodConfig.setRetries(3);
        //获取服务（并非真实的对象而是代理对象）
        UserService userService = userConfigReference.get();

        Scanner scanner = new Scanner(System.in);
        String input = "";
        while (!"quit".equalsIgnoreCase(input)){
            input = scanner.next();
            System.out.println(userService.echo(input));
        }
```
首先根据dubbo配置类获取参数保存到HashMap，然后根据map中的参数创建代理对象。
```java
// ReferenceConfig.java
    private void init() {
        //dubbo相关的参数配置
        ......
        //获取要注册的ip的地址
        String hostToRegistry = ConfigUtils.getSystemProperty(Constants.DUBBO_IP_TO_REGISTRY);
        if (hostToRegistry == null || hostToRegistry.length() == 0) {
            hostToRegistry = NetUtils.getLocalHost();
        } else if (isInvalidLocalHost(hostToRegistry)) {
            throw new IllegalArgumentException("Specified invalid registry ip from property:" + Constants.DUBBO_IP_TO_REGISTRY + ", value:" + hostToRegistry);
        }
        map.put(Constants.REGISTER_IP_KEY, hostToRegistry);

        //attributes are stored by system context.
        StaticContext.getSystemContext().putAll(attributes);
        //创建代理对象
        ref = createProxy(map);
        ConsumerModel consumerModel = new ConsumerModel(getUniqueServiceName(), this, ref, interfaceClass.getMethods());
        ApplicationModel.initConsumerModel(getUniqueServiceName(), consumerModel);
    }
```
创建代理对象是一个很复杂的过程，大致分成几步
1. 从注册中心获取服务
2. 根据服务提供者的url连接服务端，创建并保存客户端
3. 生成代理对象

首先根据url的开头找到对应的Registry，比如zookeeper对应的是ZookeeperRegistry。整个订阅的过程要从RegistryDirectory#subscribe()开始看起。url为要订阅的url，this为实现了NotifyListener接口的`RegistryDirectory`实例对象。
```java
    public void subscribe(URL url) {
        setConsumerUrl(url);
        registry.subscribe(url, this);
    }
```
接下来是FailbackRegistry#subscribe()方法调用，首先调用父类中的subscribe方法，其实就是把监听器通知器即RegistryDirectory加入到url对应的Set集合中，然后再调用子类的doSubscribe()方法进行实际的订阅操作。
```java
// FailbackRegistry
    public void subscribe(URL url, NotifyListener listener) {
        super.subscribe(url, listener);
        removeFailedSubscribed(url, listener);
        try {
            // Sending a subscription request to the server side
            doSubscribe(url, listener);
        }
        ......
    }
// AbstractRegistry.java
    public void subscribe(URL url, NotifyListener listener) {
        if (url == null) {
            throw new IllegalArgumentException("subscribe url == null");
        }
        if (listener == null) {
            throw new IllegalArgumentException("subscribe listener == null");
        }
        if (logger.isInfoEnabled()) {
            logger.info("Subscribe: " + url);
        }
        //subscribed是一个ConcurrentMap<URL, Set<NotifyListener>>
        Set<NotifyListener> listeners = subscribed.get(url);
        if (listeners == null) {
            subscribed.putIfAbsent(url, new ConcurrentHashSet<NotifyListener>());
            listeners = subscribed.get(url);
        }
        listeners.add(listener);
    }
```

第一步是从url中提取要订阅的路径，即获取Constants.CATEGORY_KEY("category")参数的值，然后拼接在接口路径后面
```java
    private String[] toCategoriesPath(URL url) {
        String[] categories;
        if (Constants.ANY_VALUE.equals(url.getParameter(Constants.CATEGORY_KEY))) {
            categories = new String[]{Constants.PROVIDERS_CATEGORY, Constants.CONSUMERS_CATEGORY,
                    Constants.ROUTERS_CATEGORY, Constants.CONFIGURATORS_CATEGORY};
        } else {
            categories = url.getParameter(Constants.CATEGORY_KEY, new String[]{Constants.DEFAULT_CATEGORY});
        }
        String[] paths = new String[categories.length];
        for (int i = 0; i < categories.length; i++) {
            //toServicePath用于获取从根路径出发的接口路径
            //例如/dubbo/com.alibaba.dubbo.demo.UserService/providers
            paths[i] = toServicePath(url) + Constants.PATH_SEPARATOR + categories[i];
        }
        return paths;
    }
```
第二步首先判断是不是对所有的接口(*)进行订阅，是的话走上面的逻辑，如果指定了要消费的接口，则走else。然后向订阅的路径的子路径注册监听器，当子路径发送改变时ChildListener的匿名实现类的childChanged()方法就会被调用，此时调用ZookeeperRegistry类的notify()方法。
```java
    protected void doSubscribe(final URL url, final NotifyListener listener) {
        try {
            //Constants.ANY_VALUE="*"
            if (Constants.ANY_VALUE.equals(url.getServiceInterface())) {
                ......
            } else {
                List<URL> urls = new ArrayList<URL>();
                for (String path : toCategoriesPath(url)) {
                    //根据url获取一个map，key是NotifyListener监听器通知器,value是节点监听器
                    //如果不存在则依次创建，先根据url生成ConcurrentHashMap
                    //再将监听器通知器跟监听器的映射关系保存下来
                    ConcurrentMap<NotifyListener, ChildListener> listeners = zkListeners.get(url);
                    if (listeners == null) {
                        zkListeners.putIfAbsent(url, new ConcurrentHashMap<NotifyListener, ChildListener>());
                        listeners = zkListeners.get(url);
                    }
                    ChildListener zkListener = listeners.get(listener);
                    if (zkListener == null) {
                        listeners.putIfAbsent(listener, new ChildListener() {
                            @Override
                            public void childChanged(String parentPath, List<String> currentChilds) {
                                ZookeeperRegistry.this.notify(url, listener, toUrlsWithEmpty(url, parentPath, currentChilds));
                            }
                        });
                        zkListener = listeners.get(listener);
                    }
                    zkClient.create(path, false);
                    //向zk节点注册监听器,节点数据改变时匿名类的childChanged()方法会被调用
                    //返回的是监听的子路径
                    List<String> children = zkClient.addChildListener(path, zkListener);
                    if (children != null) {
                        urls.addAll(toUrlsWithEmpty(url, path, children));
                    }
                }
                notify(url, listener, urls);
            }
        } catch (Throwable e) {
            throw new RpcException("Failed to subscribe " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
第三步获取子节点URL信息，保存在zookeeper上的字节的是字节流，获取到子节点字节流之后，需要以"UTF-8"对字节流进行编码，生成字符串，再转化为URL对象。然后跟消费者的要求进行比较，比如判断是否为消费者要调用的接口，如果匹配就加入到urls对应的ArrayList集合中。
```java
    private List<URL> toUrlsWithoutEmpty(URL consumer, List<String> providers) {
        List<URL> urls = new ArrayList<URL>();
        if (providers != null && !providers.isEmpty()) {
            for (String provider : providers) {
                provider = URL.decode(provider);
                if (provider.contains("://")) {
                    URL url = URL.valueOf(provider);
                    if (UrlUtils.isMatch(consumer, url)) {
                        urls.add(url);
                    }
                }
            }
        }
        return urls;
    }
```
当监听到节点发生变化时先判断是否为消费者监听的子路径，是的话再根据类别保存对应的URL。
```java
    protected void notify(URL url, NotifyListener listener, List<URL> urls) {
        .....
        Map<String, List<URL>> result = new HashMap<String, List<URL>>();
        for (URL u : urls) {
            //是否为要监听的路径url
            if (UrlUtils.isMatch(url, u)) {
                //类别，可以取的值:configurators、providers、routers
                String category = u.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
                List<URL> categoryList = result.get(category);
                if (categoryList == null) {
                    categoryList = new ArrayList<URL>();
                    result.put(category, categoryList);
                }
                categoryList.add(u);
            }
        }
        if (result.size() == 0) {
            return;
        }
        Map<String, List<URL>> categoryNotified = notified.get(url);
        if (categoryNotified == null) {
            notified.putIfAbsent(url, new ConcurrentHashMap<String, List<URL>>());
            categoryNotified = notified.get(url);
        }
        for (Map.Entry<String, List<URL>> entry : result.entrySet()) {
            String category = entry.getKey();
            List<URL> categoryList = entry.getValue();
            categoryNotified.put(category, categoryList);
            saveProperties(url);
            //调用RegistryDirectory类的notify()方法进行通知
            listener.notify(categoryList);
        }
    }
```
以服务提供者为例，如果发现服务提供者创建的节点发生改变，则将节点url添加到invokerUrls(ArrayList<URL>)中，然后refreshInvoker刷新服务提供者的信息。
```java
    public synchronized void notify(List<URL> urls) {
        List<URL> invokerUrls = new ArrayList<URL>();
        List<URL> routerUrls = new ArrayList<URL>();
        List<URL> configuratorUrls = new ArrayList<URL>();
        for (URL url : urls) {
            String protocol = url.getProtocol();
            String category = url.getParameter(Constants.CATEGORY_KEY, Constants.DEFAULT_CATEGORY);
            if (Constants.ROUTERS_CATEGORY.equals(category)
                    || Constants.ROUTE_PROTOCOL.equals(protocol)) {
                routerUrls.add(url);
            } else if (Constants.CONFIGURATORS_CATEGORY.equals(category)
                    || Constants.OVERRIDE_PROTOCOL.equals(protocol)) {
                configuratorUrls.add(url);
            } else if (Constants.PROVIDERS_CATEGORY.equals(category)) {
                invokerUrls.add(url);
            } else {
                logger.warn("Unsupported category " + category + " in notified url: " + url + " from registry " + getUrl().getAddress() + " to consumer " + NetUtils.getLocalHost());
            }
        }
        // configurators
        if (configuratorUrls != null && !configuratorUrls.isEmpty()) {
            this.configurators = toConfigurators(configuratorUrls);
        }
        // routers
        if (routerUrls != null && !routerUrls.isEmpty()) {
            List<Router> routers = toRouters(routerUrls);
            if (routers != null) { // null - do nothing
                setRouters(routers);
            }
        }
        List<Configurator> localConfigurators = this.configurators; // local reference
        // merge override parameters
        this.overrideDirectoryUrl = directoryUrl;
        if (localConfigurators != null && !localConfigurators.isEmpty()) {
            for (Configurator configurator : localConfigurators) {
                this.overrideDirectoryUrl = configurator.configure(overrideDirectoryUrl);
            }
        }
        // providers
        refreshInvoker(invokerUrls);
    }
```
至此，dubbo完成了从注册中心获取服务提供者的URL数据。

回到ReferenceConfig#init()中，接下来会通过ReferenceConfig#createProxy()创建代理对象，下一步是创建客户端，用于跟服务端进行通信，调用远程的服务端暴露的接口服务。首次订阅时也能拿到服务端暴露的接口服务，因此就会在监听器中开启连接创建客户端。RegistryDirectory#refreshInvoker()中会调用toInvokers()遍历获取到URL，依次连接，并根据每个URL创建客户端。最终到DubboProtocol类中调用getClients()根据url连接服务端，创建客户端，再底层就是netty连接服务端了，完成连接后返回了ExchangeClient对象表示已经建立连接的客户端。
```java
// DubboProtocol.java
    private ExchangeClient[] getClients(URL url) {
        // whether to share connection
        //是否为共享连接
        boolean service_share_connect = false;
        int connections = url.getParameter(Constants.CONNECTIONS_KEY, 0);
        // if not configured, connection is shared, otherwise, one connection for one service
        if (connections == 0) {
            service_share_connect = true;
            connections = 1;
        }

        ExchangeClient[] clients = new ExchangeClient[connections];
        for (int i = 0; i < clients.length; i++) {
            //如果是共享连接从缓存中获取
            if (service_share_connect) {
                clients[i] = getSharedClient(url);
            } else {
                //不是共享连接则新建一个客户端
                clients[i] = initClient(url);
            }
        }
        return clients;
    }
```
获取已经完成连接的客户端之后，使用DubboInvoker进行封装。
```java
    public <T> Invoker<T> refer(Class<T> serviceType, URL url) throws RpcException {
        optimizeSerialization(url);
        // create rpc invoker.
        DubboInvoker<T> invoker = new DubboInvoker<T>(serviceType, url, getClients(url), invokers);
        invokers.add(invoker);
        return invoker;
    }
```
然后使用JavassistProxyFactory#getProxy()方法创建动态代理对象，invoker就是刚刚返回的含有客户端的DubboInvoker对象。
```java
// JavassistProxyFactory.java
    public <T> T getProxy(Invoker<T> invoker, Class<?>[] interfaces) {
        return (T) Proxy.getProxy(interfaces).newInstance(new InvokerInvocationHandler(invoker));
    }
```
**总结下，首先创建配置类，然后根据配置类生成消费的资源URL，再到注册中心拉取服务提供者的URL，对服务提供者URL解析后通过netty连接服务端，生成客户端后创建要调用接口的代理对象。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**