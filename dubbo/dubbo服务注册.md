dubbo框架可以将服务注册到zookeeper上，接下来看下dubbo框架是怎么注册服务的。
```java
        /**
         * 使用api编码的形式进行dubbo服务暴露
         */
            //模拟spring服务实现（此处不使用spring环境 读者可以自行使用）
            UserService demoService = new UserServiceImpl();

            //1、创建应用信息（服务提供者和服务消费者均需要，以便用于计算应用之间的依赖关系）
            ApplicationConfig appliction = new ApplicationConfig("demo-core");

            //2、创建注册中心（服务提供者和服务消费者均需要，以便用于服务注册和服务发现）
            //dubbo支持的注册中心：①simple、②Multicast、③zookeeper、④ redis
            //这里采用zookeeper（常用也是dubbo推荐使用的）
            RegistryConfig registry = new RegistryConfig();
            registry.setAddress("zookeeper://localhost:2181");

            //3、创建服务暴露后对应的协议信息，该设置约定了消费者使用哪种协议来请求服务提供者
            //dubbo消费协议如下：   ①dubbo://、②hessian://、③rmi://、 ④http://、⑤webservice://、⑦memcached://、⑧redis://
            ProtocolConfig protocol = new ProtocolConfig();
            //使用dubbo协议
            protocol.setName("dubbo");
            protocol.setPort(28830);

            //4、该类为服务消费者的全局配置（比如是否延迟暴露，是否暴露服务等）
            ProviderConfig provider = new ProviderConfig();
            //是否暴露服务
            provider.setExport(true);
            //延迟一秒发布服务
            provider.setDelay(1000);

            //5、服务提供者 会配置应用、注册、协议、提供者等信息
            ServiceConfig<UserService> serviceService = new ServiceConfig<>();
            //设置应用信息
            serviceService.setApplication(appliction);
            //设置注册信息
            serviceService.setRegistry(registry);
            //设置相关协议
            serviceService.setProtocol(protocol);
            //设置全局配置信息
            serviceService.setProvider(provider);
            //设置接口信息
            serviceService.setInterface(UserService.class);
            serviceService.setRef(demoService);

            //6、服务发布 （最终上面设置的相关信息转换为Dubbo URL的形式暴露出去）
            serviceService.export();
            while (true){
            }
```
在ServiceConfig#doExport()方法中会为dubbo用到的配置类的属性赋值，值取自环境变量，再以setXXX()方法进行注入。
```java
// ServiceConfig.java
    protected synchronized void doExport() {
        ......
        checkApplication();
        checkRegistry();
        checkProtocol();
        appendProperties(this);
        checkStubAndMock(interfaceClass);
        if (path == null || path.length() == 0) {
            path = interfaceName;
        }
        doExportUrls();
        ProviderModel providerModel = new ProviderModel(getUniqueServiceName(), this, ref);
        ApplicationModel.initProviderModel(getUniqueServiceName(), providerModel);
    }
```
接下来是通过配置类RegistryConfig创建URL
```java
// AbstractInterfaceConfig.java
    protected List<URL> loadRegistries(boolean provider) {
        checkRegistry();
        List<URL> registryList = new ArrayList<URL>();
        if (registries != null && !registries.isEmpty()) {
            for (RegistryConfig config : registries) {
                String address = config.getAddress();
                if (address == null || address.length() == 0) {
                    address = Constants.ANYHOST_VALUE;
                }
                String sysaddress = System.getProperty("dubbo.registry.address");
                if (sysaddress != null && sysaddress.length() > 0) {
                    address = sysaddress;
                }
                if (address != null && address.length() > 0
                        && !RegistryConfig.NO_AVAILABLE.equalsIgnoreCase(address)) {
                    Map<String, String> map = new HashMap<String, String>();
                    appendParameters(map, application);
                    appendParameters(map, config);
                    map.put("path", RegistryService.class.getName());
                    map.put("dubbo", Version.getVersion());
                    map.put(Constants.TIMESTAMP_KEY, String.valueOf(System.currentTimeMillis()));
                    if (ConfigUtils.getPid() > 0) {
                        map.put(Constants.PID_KEY, String.valueOf(ConfigUtils.getPid()));
                    }
                    if (!map.containsKey("protocol")) {
                        if (ExtensionLoader.getExtensionLoader(RegistryFactory.class).hasExtension("remote")) {
                            map.put("protocol", "remote");
                        } else {
                            map.put("protocol", "dubbo");
                        }
                    }
                    List<URL> urls = UrlUtils.parseURLs(address, map);
                    for (URL url : urls) {
                        url = url.addParameter(Constants.REGISTRY_KEY, url.getProtocol());
                        url = url.setProtocol(Constants.REGISTRY_PROTOCOL);
                        if ((provider && url.getParameter(Constants.REGISTER_KEY, true))
                                || (!provider && url.getParameter(Constants.SUBSCRIBE_KEY, true))) {
                            registryList.add(url);
                        }
                    }
                }
            }
        }
        return registryList;
    }
```
获取到URL后，先创建代理类对象，然后暴露服务。
```java
// ServiceConfig.java#doExportUrlsFor1Protocol
                    for (URL registryURL : registryURLs) {
                        url = url.addParameterIfAbsent(Constants.DYNAMIC_KEY, registryURL.getParameter(Constants.DYNAMIC_KEY));
                        URL monitorUrl = loadMonitor(registryURL);
                        if (monitorUrl != null) {
                            url = url.addParameterAndEncoded(Constants.MONITOR_KEY, monitorUrl.toFullString());
                        }
                        if (logger.isInfoEnabled()) {
                            logger.info("Register dubbo service " + interfaceClass.getName() + " url " + url + " to registry " + registryURL);
                        }
                        Invoker<?> invoker = proxyFactory.getInvoker(ref, (Class) interfaceClass, registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString()));
                        DelegateProviderMetaDataInvoker wrapperInvoker = new DelegateProviderMetaDataInvoker(invoker, this);

                        Exporter<?> exporter = protocol.export(wrapperInvoker);
                        exporters.add(exporter);
                    }
```
先到DubboProtocol调用export()方法进行服务暴露，openServer()中会创建netty服务器，监听的是ProtocolConfig配置类中设置的端口号，因此不同的服务需要设置不同的端口号。
```java
    public <T> Exporter<T> export(Invoker<T> invoker) throws RpcException {
        URL url = invoker.getUrl();

        // export service.
        String key = serviceKey(url);
        DubboExporter<T> exporter = new DubboExporter<T>(invoker, key, exporterMap);
        exporterMap.put(key, exporter);

        //export an stub service for dispatching event
        Boolean isStubSupportEvent = url.getParameter(Constants.STUB_EVENT_KEY, Constants.DEFAULT_STUB_EVENT);
        Boolean isCallbackservice = url.getParameter(Constants.IS_CALLBACK_SERVICE, false);
        if (isStubSupportEvent && !isCallbackservice) {
            String stubServiceMethods = url.getParameter(Constants.STUB_EVENT_METHODS_KEY);
            if (stubServiceMethods == null || stubServiceMethods.length() == 0) {
                if (logger.isWarnEnabled()) {
                    logger.warn(new IllegalStateException("consumer [" + url.getParameter(Constants.INTERFACE_KEY) +
                            "], has set stubproxy support event ,but no stub methods founded."));
                }
            } else {
                stubServiceMethodsMap.put(url.getServiceKey(), stubServiceMethods);
            }
        }

        openServer(url);
        optimizeSerialization(url);
        return exporter;
    }
```
真正触发暴露服务的操作是在RegistryProtocol#export()方法中。
```java
    public <T> Exporter<T> export(final Invoker<T> originInvoker) throws RpcException {
        //export invoker
        final ExporterChangeableWrapper<T> exporter = doLocalExport(originInvoker);
        //获取注册接口的url
        URL registryUrl = getRegistryUrl(originInvoker);

        //registry provider
        //解析url获取相应的获取注册器
        final Registry registry = getRegistry(originInvoker);
        //获取要注册的接口url
        final URL registedProviderUrl = getRegistedProviderUrl(originInvoker);

        //to judge to delay publish whether or not
        boolean register = registedProviderUrl.getParameter("register", true);

        ProviderConsumerRegTable.registerProvider(originInvoker, registryUrl, registedProviderUrl);

        if (register) {
            //真正的注册在这里
            register(registryUrl, registedProviderUrl);
            ProviderConsumerRegTable.getProviderWrapper(originInvoker).setReg(true);
        }

        // Subscribe the override data
        // FIXME When the provider subscribes, it will affect the scene : a certain JVM exposes the service and call the same service. Because the subscribed is cached key with the name of the service, it causes the subscription information to cover.
        final URL overrideSubscribeUrl = getSubscribedOverrideUrl(registedProviderUrl);
        final OverrideListener overrideSubscribeListener = new OverrideListener(overrideSubscribeUrl, originInvoker);
        overrideListeners.put(overrideSubscribeUrl, overrideSubscribeListener);
        registry.subscribe(overrideSubscribeUrl, overrideSubscribeListener);
        //Ensure that a new exporter instance is returned every time export
        return new DestroyableExporter<T>(exporter, originInvoker, overrideSubscribeUrl, registedProviderUrl);
    }
```
以ZookeeperRegistry#doRegister()看下，其实就是根据url设置节点路径。
```java
    protected void doRegister(URL url) {
        try {
            zkClient.create(toUrlPath(url), url.getParameter(Constants.DYNAMIC_KEY, true));
        } catch (Throwable e) {
            throw new RpcException("Failed to register " + url + " to zookeeper " + getUrl() + ", cause: " + e.getMessage(), e);
        }
    }
```
**总结下，先创建dubbo的配置类，再根据配置类创建URL，然后根据设置的协议就是url的开头选择注册器，启动netty服务器，到zk上创建节点。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**