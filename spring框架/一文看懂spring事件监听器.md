spring基于发布订阅模式实现了事件监听器，用于处理启动中的所有事件，可以对功能按模块解耦。首先spring启动时会从spring.factories中读取配置的监听器，先看下这个过程。
```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    ...
        //获取监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
    ...
	}
```
读取所有jar包下的META-INF/spring.factories，根据ApplicationListener的全路径名获取对应的接口实现类，缓存下来后再返回ApplicationListener的定制实现类，然后以反射的方式构造实例对象。spring中大量运用了缓存，避免过多的内存分配造成的cpu无效占用。
```java
	private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {
		ClassLoader classLoader = getClassLoader();
		// 根据接口名获取实现类名
		Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        // 反射构造实例对象
		List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);
		AnnotationAwareOrderComparator.sort(instances);
		return instances;
	}   

	private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        //优先从缓存中获取
		MultiValueMap<String, String> result = cache.get(classLoader);
		if (result != null) {
			return result;
		}

		try {
            //读取所有jar包下META-INF/spring.factories配置文件
			Enumeration<URL> urls = (classLoader != null ?
					classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
					ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
			result = new LinkedMultiValueMap<>();
			while (urls.hasMoreElements()) {
				URL url = urls.nextElement();
				UrlResource resource = new UrlResource(url);
                //加载key=value配置文件
                //key为接口全路径名，value为接口的实现类
				Properties properties = PropertiesLoaderUtils.loadProperties(resource);
				for (Map.Entry<?, ?> entry : properties.entrySet()) {
					String factoryTypeName = ((String) entry.getKey()).trim();
					for (String factoryImplementationName : StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
						result.add(factoryTypeName, factoryImplementationName.trim());
					}
				}
			}
			cache.put(classLoader, result);
			return result;
		}
		catch (IOException ex) {
			throw new IllegalArgumentException("Unable to load factories from location [" +
					FACTORIES_RESOURCE_LOCATION + "]", ex);
		}
	}    
```
run方法运行容器时会发布各种事件，首先跟读取监听器的方式类似，获取事件发布器，用于发布各阶段事件，此时会将从spring.factories中读取的事件监听器加入事件发布器，让支持的事件监听器去处理，接下来我们看spring启动过程中发布的第一个事件ApplicationStartingEvent启动事件。
```java
	public ConfigurableApplicationContext run(String... args) {
...         //从spring.factories读取事件发布器SpringApplicationRunListener
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
...
	}

	public EventPublishingRunListener(SpringApplication application, String[] args) {
		this.application = application;
		this.args = args;
		this.initialMulticaster = new SimpleApplicationEventMulticaster();
        //spring的事件监听器添加到广播，即事件发布器
		for (ApplicationListener<?> listener : application.getListeners()) {
			this.initialMulticaster.addApplicationListener(listener);
		}
	}
```
获取支持处理ApplicationStartingEvent事件的监听器，然后处理，接下来重点分析这个过程。
```java
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```
spring会按事件保存到ListenerRetriever，在内置的属性`Set<ApplicationListener<?>> applicationListeners`会保存每种事件集合，同时将事件和ListenerRetriever缓存到`Map<ListenerCacheKey, ListenerRetriever> retrieverCache`中，用于快速检索，发现没获取到就会按事件过滤`retrieveApplicationListeners`。
```java
	protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

		Object source = event.getSource();
		Class<?> sourceType = (source != null ? source.getClass() : null);
		ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

		// 缓存中根据事件类型和事件对象快速检索
		ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
		if (retriever != null) {
			return retriever.getApplicationListeners();
		}

		if (this.beanClassLoader == null ||
				(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
						(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
			// Fully synchronized building and caching of a ListenerRetriever
			synchronized (this.retrievalMutex) {
				retriever = this.retrieverCache.get(cacheKey);
				if (retriever != null) {
					return retriever.getApplicationListeners();
				}
				retriever = new ListenerRetriever(true);
                //获取不到即需要手动解析，从所有监听器中找出支持事件的监听器
				Collection<ApplicationListener<?>> listeners =
						retrieveApplicationListeners(eventType, sourceType, retriever);
				this.retrieverCache.put(cacheKey, retriever);
				return listeners;
			}
		}
		else {
			// No ListenerRetriever caching -> no synchronization necessary
			return retrieveApplicationListeners(eventType, sourceType, null);
		}
	}
```
