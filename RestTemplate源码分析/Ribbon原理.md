---
title: Ribbon原理
tags:
notebook: RestTemplate源码解析
---
昨天看了@LoadBalanced调用微服务接口的过程，接下来细致的分析Ribbon负载均衡客户端的原理。调用微服务接口的时候，会创建一个新的spring应用上下文，然后从容器获取负载均衡器，即实现了ILoadBalancer接口的Bean，接下来着重分析这个Bean怎么工作的。

Ribbon相关配置都是在RibbonClientConfiguration类中实现的，负载均衡默认是通过ZoneAwareLoadBalancer.java实现，可以看到注入很多相关的Bean。
```java
	@Bean
	@ConditionalOnMissingBean
	public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
			ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
			IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
		if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
			return this.propertiesFactory.get(ILoadBalancer.class, config, name);
		}
		return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
				serverListFilter, serverListUpdater);
	}
```
有个重要的Bean先拿出来说下，就是ServerListUpdater。可以看到在构造DynamicServerListLoadBalancer实例时，会调用restOfInit()方法，此时会通过enableAndInitLearnNewServersFeature()开启异步任务，周期性刷新可用的微服务列表。
```java
//RibbonClientConfiguration.java
	@Bean
	@ConditionalOnMissingBean
	public ServerListUpdater ribbonServerListUpdater(IClientConfig config) {
		return new PollingServerListUpdater(config);
	}
//DynamicServerListLoadBalancer#<init>
	void restOfInit(IClientConfig clientConfig) {
	boolean primeConnection = this.isEnablePrimingConnections();
	// turn this off to avoid duplicated asynchronous priming done in BaseLoadBalancer.setServerList()
	this.setEnablePrimingConnections(false);
	enableAndInitLearnNewServersFeature();

	updateListOfServers();
	if (primeConnection && this.getPrimeConnections() != null) {
		this.getPrimeConnections()
				.primeConnections(getReachableServers());
	}
	this.setEnablePrimingConnections(primeConnection);
	LOGGER.info("DynamicServerListLoadBalancer for client {} initialized: {}", clientConfig.getClientName(), this.toString());
    }

	public synchronized void start(final UpdateAction updateAction) {
        if (isActive.compareAndSet(false, true)) {
            final Runnable wrapperRunnable = new Runnable() {
                @Override
                public void run() {
                    if (!isActive.get()) {
                        if (scheduledFuture != null) {
                            scheduledFuture.cancel(true);
                        }
                        return;
                    }
                    try {
						//周期性的执行
                        updateAction.doUpdate();
                        lastUpdated = System.currentTimeMillis();
                    } catch (Exception e) {
                        logger.warn("Failed one update cycle", e);
                    }
                }
            };
			//间隔时间是refreshIntervalMs
            scheduledFuture = getRefreshExecutor().scheduleWithFixedDelay(
                    wrapperRunnable,
                    initialDelayMs,
                    refreshIntervalMs,
                    TimeUnit.MILLISECONDS
            );
        } else {
            logger.info("Already active, no-op");
        }
    }

	protected final ServerListUpdater.UpdateAction updateAction = new ServerListUpdater.UpdateAction() {
        @Override
        public void doUpdate() {
			//更新服务，如果是nacos，则周期性刷新服务列表
            updateListOfServers();
        }
    };
```

通过chooseServer()方法获取可用的服务，最终到父类BaseLoadBalancer.java的chooseServer()方法调用。
1. 通过AtomicLong维护调用微服务接口的次数。
2. 获取负载均衡的规则IRule。在父类BaseLoadBalancer中通过属性rule(IRule)设置负载均衡策略，默认为RoundRobinRule。在setRule()方法设置IRule时，会先到容器中获取实现了IRule接口的Bean，而在RibbonClientConfiguration.java类中创建了ZoneAvoidanceRule Bean，因此会被注入到BaseLoadBalancer中，此时获取到的就是ZoneAvoidanceRule。
```java
public class BaseLoadBalancer extends AbstractLoadBalancer implements
        PrimeConnections.PrimeConnectionListener, IClientConfigAware {

    private final static IRule DEFAULT_RULE = new RoundRobinRule();
    protected IRule rule = DEFAULT_RULE;
	public void setRule(IRule rule) {
		if (rule != null) {
			this.rule = rule;
		} else {
			/* default rule */
			this.rule = new RoundRobinRule();
		}
		if (this.rule.getLoadBalancer() != this) {
			this.rule.setLoadBalancer(this);
		}
	}
	......
}
	@Bean
	@ConditionalOnMissingBean
	public IRule ribbonRule(IClientConfig config) {
		if (this.propertiesFactory.isSet(IRule.class, name)) {
			return this.propertiesFactory.get(IRule.class, config, name);
		}
		ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
		rule.initWithNiwsConfig(config);
		return rule;
	}
```
找到负载均衡策略后就会调用choose()选择要调用的服务接口。在setRule()时同时设置设置了负载均衡器`this.rule.setLoadBalancer(this);`，this就是负载均衡器ZoneAwareLoadBalancer的实例对象。接下来就是Ribbon的核心了，一步步来看吧。
1. 先通过负载均衡器获取所有可调用的微服务接口allServerList，看下这个怎么来的。先梳理方法调用栈，在构造负载均衡器ZoneAwareLoadBalancer的实例对象时，会调用updateListOfServers()方法更新微服务接口列表，这是通过ServerList接口的实现类对象getUpdatedListOfServers()方法调用实现的。构造负载均衡器的时候注入了ServerList接口Bean，而在NacosRibbonClientConfiguration创建了这样的Bean，因此这里的serverListImpl就是Nacos的ribbonServerList，最终通过Nacos获取到所有的微服务接口，Nacos这里怎么实现的我们后面再看。
```java
    @VisibleForTesting
    public void updateListOfServers() {
        List<T> servers = new ArrayList<T>();
        if (serverListImpl != null) {
            servers = serverListImpl.getUpdatedListOfServers();
            LOGGER.debug("List of Servers for {} obtained from Discovery client: {}",
                    getIdentifier(), servers);

            if (filter != null) {
                servers = filter.getFilteredListOfServers(servers);
                LOGGER.debug("Filtered List of Servers for {} obtained from Discovery client: {}",
                        getIdentifier(), servers);
            }
        }
        updateAllServerList(servers);
    }
// DynamicServerListLoadBalancer
    public DynamicServerListLoadBalancer(IClientConfig clientConfig, IRule rule, IPing ping,
                                         ServerList<T> serverList, ServerListFilter<T> filter,
                                         ServerListUpdater serverListUpdater) {
        super(clientConfig, rule, ping);
        this.serverListImpl = serverList;
        this.filter = filter;
        this.serverListUpdater = serverListUpdater;
        if (filter instanceof AbstractServerListFilter) {
            ((AbstractServerListFilter) filter).setLoadBalancerStats(getLoadBalancerStats());
        }
        restOfInit(clientConfig);
    }	
// NacosRibbonClientConfiguration
	@Bean
	@ConditionalOnMissingBean
	public ServerList<?> ribbonServerList(IClientConfig config, NacosDiscoveryProperties nacosDiscoveryProperties) {
		NacosServerList serverList = new NacosServerList(nacosDiscoveryProperties);
		serverList.initWithNiwsConfig(config);
		return serverList;
	}
```
2. 获取到所有的Server后，还得判断是不是每个Server都是可用的。可以看到构造rule时生成了ZoneAvoidancePredicate和AvailabilityPredicate，现在其实就是调用apply()方法判断当前的Server是不是可用的。
```java
    public ZoneAvoidanceRule() {
        super();
        ZoneAvoidancePredicate zonePredicate = new ZoneAvoidancePredicate(this);
        AvailabilityPredicate availabilityPredicate = new AvailabilityPredicate(this);
        compositePredicate = createCompositePredicate(zonePredicate, availabilityPredicate);
    }
```
3. 获取所有可用的服务之后得选择一个具体可用的对吧。这里其实就是轮询算法，挨个调用的。