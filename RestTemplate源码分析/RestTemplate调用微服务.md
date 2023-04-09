在spring-cloud-netflix-ribbon-2.0.0.RELEASE.jar依赖的spring.factories中，通过`org.springframework.boot.autoconfigure.EnableAutoConfiguration`配置项定义了要加载的自动配置类`org.springframework.cloud.netflix.ribbon.RibbonAutoConfiguration`，而这个类又导入了`LoadBalancerAutoConfiguration`配置类，这是程序的入口点，现在开始分析LoadBalancerAutoConfiguration类是怎么起作用的。
1. 类中有个属性restTemplates，因此会注入当前项目中所有的RestTemplate
2. 调用类中的restTemplateCustomizer()方法创建Bean，可以看到注入了LoadBalancerInterceptor负载均衡拦截器。而这个Bean要做的事情就是为所有的RestTemplate注入LoadBalancerInterceptor，这是通过lamban表达式实现的
3. 调用loadBalancedRestTemplateInitializerDeprecated()方法创建SmartInitializingSingleton Bean，这是一个函数式接口，也是通过lambda表达式实现的。这个Bean要做的事情是先获取所有的restTemplateCustomizer，再使用每个RestTemplateCustomizer的customize()方法，即restTemplateCustomizer的lambda表达式添加LoadBalancerInterceptor。
4. 在preInstantiateSingletons()方法中会先对容器中所有的bean进行实例化，然后找到实现了SmartInitializingSingleton接口的Bean，调用afterSingletonsInstantiated()。此时，loadBalancedRestTemplateInitializerDeprecated()方法创建的SmartInitializingSingleton Bean的lambda表达式就会起作用。
```java
@Configuration
@ConditionalOnClass(RestTemplate.class)
@ConditionalOnBean(LoadBalancerClient.class)
@EnableConfigurationProperties(LoadBalancerRetryProperties.class)
public class LoadBalancerAutoConfiguration {

	@LoadBalanced
	@Autowired(required = false)
	private List<RestTemplate> restTemplates = Collections.emptyList();

	@Bean
	public SmartInitializingSingleton loadBalancedRestTemplateInitializerDeprecated(
			final ObjectProvider<List<RestTemplateCustomizer>> restTemplateCustomizers) {
		return () -> restTemplateCustomizers.ifAvailable(customizers -> {
            for (RestTemplate restTemplate : LoadBalancerAutoConfiguration.this.restTemplates) {
                for (RestTemplateCustomizer customizer : customizers) {
                    customizer.customize(restTemplate);
                }
            }
        });
	}

	@Autowired(required = false)
	private List<LoadBalancerRequestTransformer> transformers = Collections.emptyList();

	@Bean
	@ConditionalOnMissingBean
	public LoadBalancerRequestFactory loadBalancerRequestFactory(
			LoadBalancerClient loadBalancerClient) {
		return new LoadBalancerRequestFactory(loadBalancerClient, transformers);
	}

	@Configuration
	@ConditionalOnMissingClass("org.springframework.retry.support.RetryTemplate")
	static class LoadBalancerInterceptorConfig {
		@Bean
		public LoadBalancerInterceptor ribbonInterceptor(
				LoadBalancerClient loadBalancerClient,
				LoadBalancerRequestFactory requestFactory) {
			return new LoadBalancerInterceptor(loadBalancerClient, requestFactory);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestTemplateCustomizer restTemplateCustomizer(
				final LoadBalancerInterceptor loadBalancerInterceptor) {
			return restTemplate -> {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(loadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            };
		}
	}
	
}
```
**总结下来，其实这个自动配置类要做的事情就是给容器中所有的RestTemplate添加LoadBalancerInterceptor拦截器。** 所以接下来只要关注拦截器怎么起作用的就行了。

RestTemplate调用InterceptingClientHttpRequest#executeInternal()处理请求时，会构造InterceptingRequestExecution实例对象，对包含的所有拦截器进行遍历，执行每个拦截器的intercept()方法，显然这里使用了责任链模式。而在创建LoadBalancerInterceptor拦截器对象时注入了LoadBalancerClient(RibbonLoadBalancerClient)，这样就进入这个拦截器的intercept()方法调用，这里使用ServiceRequestWrapper对请求进行封装`requestFactory.createRequest(request, body, execution)`，接下来就是RibbonLoadBalancerClient#execute()的处理逻辑了。
```java
	public ClientHttpResponse intercept(final HttpRequest request, final byte[] body,
			final ClientHttpRequestExecution execution) throws IOException {
		final URI originalUri = request.getURI();
		String serviceName = originalUri.getHost();
		Assert.state(serviceName != null, "Request URI does not contain a valid hostname: " + originalUri);
		return this.loadBalancer.execute(serviceName, requestFactory.createRequest(request, body, execution));
	}
	public LoadBalancerRequest<ClientHttpResponse> createRequest(final HttpRequest request,
			final byte[] body, final ClientHttpRequestExecution execution) {
		return instance -> {
            HttpRequest serviceRequest = new ServiceRequestWrapper(request, instance, loadBalancer);
            if (transformers != null) {
                for (LoadBalancerRequestTransformer transformer : transformers) {
                    serviceRequest = transformer.transformRequest(serviceRequest, instance);
                }
            }
            return execution.execute(serviceRequest, body);
        };
	}
```
每个微服务都有一个与之对应的负载均衡器对象ILoadBalancer，在RibbonLoadBalancerClient#execute()方法中会通过当前要调用的微服务找到相应的负载均衡器。构造RibbonLoadBalancerClient实例对象时会注入SpringClientFactory，这时会通过SpringClientFactory创建应用上下文，再从应用上下文获取负载均衡器。先到contexts`private Map<String, AnnotationConfigApplicationContext> contexts = new ConcurrentHashMap<>();`根据微服务名获取对应的上下文对象，不存在则通过createContext()创建。
```java
	protected AnnotationConfigApplicationContext getContext(String name) {
		if (!this.contexts.containsKey(name)) {
			synchronized (this.contexts) {
				if (!this.contexts.containsKey(name)) {
					this.contexts.put(name, createContext(name));
				}
			}
		}
		return this.contexts.get(name);
	}
```
configurations`private Map<String, C> configurations = new ConcurrentHashMap<>();`为NamedContextFactory的属性，key为组件名，value为RibbonClientSpecification。对类使用了@RibbonClients注解后，会导入RibbonClientConfigurationRegistrar组件。
```java
@Configuration
@Import(RibbonClientConfigurationRegistrar.class)
public @interface RibbonClients {
```
RibbonClientConfigurationRegistrar组件实现了ImportBeanDefinitionRegistrar接口，可以向容器注册Bean的定义。可以看到注册的是RibbonClientSpecification类型的Bean定义，而后跟的是构造函数的参数，即使用了@RibbonClients的类Class。当前使用了这个注解的类有RibbonNacosAutoConfiguration和RibbonAutoConfiguration，因而configurations中的RibbonClientSpecification分别包含RibbonNacosAutoConfiguration和RibbonAutoConfiguration，在RibbonAutoConfiguration#springClientFactory()生成SpringClientFactory会传进来。
```java
	private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name,
			Object configuration) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder
				.genericBeanDefinition(RibbonClientSpecification.class);
		builder.addConstructorArgValue(name);
		//configuration为Class，用来定义类的
		builder.addConstructorArgValue(configuration);
		registry.registerBeanDefinition(name + ".RibbonClientSpecification",
				builder.getBeanDefinition());
	}
```
通过configurations获取到负载均衡客户端的类Class后会添加到容器中，再将RibbonClientConfiguration添加到容器，然后refresh()对容器中的Bean定义进行实例化。
```java
	protected AnnotationConfigApplicationContext createContext(String name) {
		//当前的应用上下文实例就是AnnotationConfigApplicationContext类的实例对象
		AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
		if (this.configurations.containsKey(name)) {
			for (Class<?> configuration : this.configurations.get(name)
					.getConfiguration()) {
				context.register(configuration);
			}
		}
		//configurations的value为RibbonClientSpecification，getConfiguration()返回的是配置类Class
		for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
			if (entry.getKey().startsWith("default.")) {
				for (Class<?> configuration : entry.getValue().getConfiguration()) {
					context.register(configuration);
				}
			}
		}
		//this.defaultConfigType=RibbonClientConfiguration
		//构造SpringClientFactory固定成RibbonClientConfiguration了
		context.register(PropertyPlaceholderAutoConfiguration.class,
				this.defaultConfigType);
		context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
				this.propertySourceName,
				Collections.<String, Object> singletonMap(this.propertyName, name)));
		if (this.parent != null) {
			// Uses Environment from parent as well as beans
			context.setParent(this.parent);
		}
		context.setDisplayName(generateDisplayName(name));
		context.refresh();
		return context;
	}
```
给微服务创建应用上下文后，就可以到上下文容器中获取实现了ILoadBalancer接口的Bean，即负载均衡器对象了。
```java
// RibbonClientConfiguration
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
获取到负载均衡器对象后，就要获取能够被调用的微服务了，这是由ZoneAwareLoadBalancer#chooseServer()实现的，rule为IRule接口的实现类对象，用来制定选择微服务的规则，默认为RoundRobinRule。
```java
//BaseLoadBalancer.java
    public Server chooseServer(Object key) {
        if (counter == null) {
            counter = createCounter();
        }
        counter.increment();
        if (rule == null) {
            return null;
        } else {
            try {
                return rule.choose(key);
            } catch (Exception e) {
                logger.warn("LoadBalancer [{}]:  Error choosing server for key {}", name, key, e);
                return null;
            }
        }
    }
```
返回到InterceptingClientHttpRequest#execute()方法执行，接下来需要获取请求真正的ip地址`request.getURI()`，注意这里的请求在InterceptingClientHttpRequest#executeInternal()方法被封装到ServiceRequestWrapper，因此将通过ServiceRequestWrapper#getURI()获取请求的uri，最终是在LoadBalancerContext#reconstructURIWithServer()完成微服务名到ip地址的替换。
```java
	public URI reconstructURIWithServer(Server server, URI original) {
        String host = server.getHost();
        int port = server.getPort();
        String scheme = server.getScheme();
        
        if (host.equals(original.getHost()) 
                && port == original.getPort()
                && scheme == original.getScheme()) {
            return original;
        }
        if (scheme == null) {
            scheme = original.getScheme();
        }
        if (scheme == null) {
            scheme = deriveSchemeAndPortFromPartialUri(original).first();
        }

        try {
            StringBuilder sb = new StringBuilder();
            sb.append(scheme).append("://");
            if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
                sb.append(original.getRawUserInfo()).append("@");
            }
            sb.append(host);
            if (port >= 0) {
                sb.append(":").append(port);
            }
            sb.append(original.getRawPath());
            if (!Strings.isNullOrEmpty(original.getRawQuery())) {
                sb.append("?").append(original.getRawQuery());
            }
            if (!Strings.isNullOrEmpty(original.getRawFragment())) {
                sb.append("#").append(original.getRawFragment());
            }
            URI newURI = new URI(sb.toString());
            return newURI;            
        } catch (URISyntaxException e) {
            throw new RuntimeException(e);
        }
    }
```
**总结下，首先为RestTemplate注入LoadBalancerInterceptor拦截器，调用微服务时会通过拦截器为微服务创建应用上下文，从上下文容器中获取负载均衡器对象ILoadBalancer(ZoneAwareLoadBalancer)，接下来就可以通过负载均衡器获取调用的微服务，然后完成ip地址的替换就可以发送请求了。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**
