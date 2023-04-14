在springboot的spring.factories配置文件中通过`org.springframework.boot.autoconfigure.EnableAutoConfiguration`配置项配置了SecurityAutoConfiguration为自动加载的配置类，可以看到通过@Import注解导入SpringBootWebSecurityConfiguration、WebSecurityEnablerConfiguration、SecurityDataConfiguration三个组件。
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(DefaultAuthenticationEventPublisher.class)
@EnableConfigurationProperties(SecurityProperties.class)
@Import({ SpringBootWebSecurityConfiguration.class, WebSecurityEnablerConfiguration.class,
		SecurityDataConfiguration.class })
public class SecurityAutoConfiguration {......}
```
SpringBootWebSecurityConfiguration组件要做的事情是当引入了Security框架(WebSecurityConfigurerAdapter类存在)且没有定制时(WebSecurityConfigurerAdapter类型Bean不存在)，创建默认的配置DefaultConfigurerAdapter。
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass(WebSecurityConfigurerAdapter.class)
@ConditionalOnMissingBean(WebSecurityConfigurerAdapter.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
public class SpringBootWebSecurityConfiguration {

	@Configuration(proxyBeanMethods = false)
	@Order(SecurityProperties.BASIC_AUTH_ORDER)
	static class DefaultConfigurerAdapter extends WebSecurityConfigurerAdapter {

	}

}
```
不要被这么多注解吓到，仔细得看其实没有什么，核心就是通过`@EnableWebSecurity`导入WebSecurityConfiguration、SpringWebMvcImportSelector、OAuth2ImportSelector组件。
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnBean(WebSecurityConfigurerAdapter.class)
@ConditionalOnMissingBean(name = BeanIds.SPRING_SECURITY_FILTER_CHAIN)
@ConditionalOnWebApplication(type = ConditionalOnWebApplication.Type.SERVLET)
@EnableWebSecurity
public class WebSecurityEnablerConfiguration {

}
......
@Import({ WebSecurityConfiguration.class,
		SpringWebMvcImportSelector.class,
		OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {

	boolean debug() default false;
}
```
setFilterChainProxySecurityConfigurer()方法上存在`@Autowired`注解，因此会被调用，会向方法注入objectPostProcessor(AutowireBeanFactoryObjectPostProcessor)和webSecurityConfigurers(SecurityConfigurer接口的实现类Bean，即我们业务中的WebSecurityConfigurerAdapter的扩展子类)。
```java
// WebSecurityConfiguration

	private WebSecurity webSecurity;
	private List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers;
	@Autowired(required = false)
	private ObjectPostProcessor<Object> objectObjectPostProcessor;

    @Autowired(required = false)
	public void setFilterChainProxySecurityConfigurer(
			ObjectPostProcessor<Object> objectPostProcessor,
			@Value("#{@autowiredWebSecurityConfigurersIgnoreParents.getWebSecurityConfigurers()}") List<SecurityConfigurer<Filter, WebSecurity>> webSecurityConfigurers)
			throws Exception {
		webSecurity = objectPostProcessor
				.postProcess(new WebSecurity(objectPostProcessor));
		if (debugEnabled != null) {
			webSecurity.debug(debugEnabled);
		}

		......
		for (SecurityConfigurer<Filter, WebSecurity> webSecurityConfigurer : webSecurityConfigurers) {
			webSecurity.apply(webSecurityConfigurer);
		}
		this.webSecurityConfigurers = webSecurityConfigurers;
	}
```
AutowireBeanFactoryObjectPostProcessor#postProcess()方法会调用Bean得初始化方法，对属性进行填充，即对WebSecurity进行实例化。
```java
	public <T> T postProcess(T object) {
		if (object == null) {
			return null;
		}
		T result = null;
		try {
			result = (T) this.autowireBeanFactory.initializeBean(object,
					object.toString());
		}
		catch (RuntimeException e) {
			Class<?> type = object.getClass();
			throw new RuntimeException(
					"Could not postProcess " + object + " of type " + type, e);
		}
		this.autowireBeanFactory.autowireBean(object);
		if (result instanceof DisposableBean) {
			this.disposableBeans.add((DisposableBean) result);
		}
		if (result instanceof SmartInitializingSingleton) {
			this.smartSingletons.add((SmartInitializingSingleton) result);
		}
		return result;
	}
```
再就是通过WebSecurity的AbstractConfiguredSecurityBuilder#apply()方法保存配置到configurers(`private final LinkedHashMap<Class<? extends SecurityConfigurer<O, B>>, List<SecurityConfigurer<O, B>>> configurers = new LinkedHashMap<>();`)，configurers的key为SecurityConfigurer或子类，value为对应的实现类对象，也就是在这里保存我们业务中实现的WebSecurityConfigurerAdapter的扩展子类。
```java
	public <C extends SecurityConfigurer<O, B>> C apply(C configurer) throws Exception {
		add(configurer);
		return configurer;
	}
	private <C extends SecurityConfigurer<O, B>> void add(C configurer) {
		Assert.notNull(configurer, "configurer cannot be null");

		Class<? extends SecurityConfigurer<O, B>> clazz = (Class<? extends SecurityConfigurer<O, B>>) configurer
				.getClass();
		synchronized (configurers) {
			if (buildState.isConfigured()) {
				throw new IllegalStateException("Cannot apply " + configurer
						+ " to already built object");
			}
            //allowConfigurersOfSameType默认为false
			List<SecurityConfigurer<O, B>> configs = allowConfigurersOfSameType ? this.configurers
					.get(clazz) : null;
			if (configs == null) {
				configs = new ArrayList<>(1);
			}
			configs.add(configurer);
			this.configurers.put(clazz, configs);
			if (buildState.isInitializing()) {
				this.configurersAddedInInitializing.add(configurer);
			}
		}
	}
```
至此spring将我们定义的WebSecurityConfigurerAdapter扩展保存下来了，接下来会通过WebSecurity的父类AbstractSecurityBuilder#build()构建对象。
```java
	@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}
```
核心在AbstractConfiguredSecurityBuilder#doBuild()方法，看下doBuild()方法的过程吧。
```java
	protected final O doBuild() throws Exception {
		synchronized (configurers) {
			buildState = BuildState.INITIALIZING;

			beforeInit();
			init();

			buildState = BuildState.CONFIGURING;

			beforeConfigure();
			configure();

			buildState = BuildState.BUILDING;

			O result = performBuild();

			buildState = BuildState.BUILT;

			return result;
		}
	}
```
1. 对AuthenticationManagerBuilder进行配置`protected void configure(AuthenticationManagerBuilder auth)`，实例化HttpSecurity，并调用钩子函数`protected void configure(HttpSecurity http)`对HttpSecurity进行定制，我们业务代码中的两个config就是在这里被调用的。
```java
	public void init(final WebSecurity web) throws Exception {
		final HttpSecurity http = getHttp();
		web.addSecurityFilterChainBuilder(http).postBuildAction(() -> {
			FilterSecurityInterceptor securityInterceptor = http
					.getSharedObject(FilterSecurityInterceptor.class);
			web.securityInterceptor(securityInterceptor);
		});
	}
    protected final HttpSecurity getHttp() throws Exception {
		if (http != null) {
			return http;
		}

		DefaultAuthenticationEventPublisher eventPublisher = objectPostProcessor
				.postProcess(new DefaultAuthenticationEventPublisher());
		localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

		AuthenticationManager authenticationManager = authenticationManager();
		authenticationBuilder.parentAuthenticationManager(authenticationManager);
		authenticationBuilder.authenticationEventPublisher(eventPublisher);
		Map<Class<?>, Object> sharedObjects = createSharedObjects();

		http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
				sharedObjects);
		if (!disableDefaults) {
			// @formatter:off
			http
				.csrf().and()
				.addFilter(new WebAsyncManagerIntegrationFilter())
				.exceptionHandling().and()
				.headers().and()
				.sessionManagement().and()
				.securityContext().and()
				.requestCache().and()
				.anonymous().and()
				.servletApi().and()
				.apply(new DefaultLoginPageConfigurer<>()).and()
				.logout();
			// @formatter:on
			ClassLoader classLoader = this.context.getClassLoader();
			List<AbstractHttpConfigurer> defaultHttpConfigurers =
					SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);

			for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
				http.apply(configurer);
			}
		}
		configure(http);
		return http;
	}
```
2. 对FilterChainProxy进行实例化，FilterChainProxy封装了过滤器链
```java
	protected Filter performBuild() throws Exception {
		Assert.state(
				!securityFilterChainBuilders.isEmpty(),
				() -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
						+ "Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
						+ "More advanced users can invoke "
						+ WebSecurity.class.getSimpleName()
						+ ".addSecurityFilterChainBuilder directly");
		int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
		List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
				chainSize);
		for (RequestMatcher ignoredRequest : ignoredRequests) {
			securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
		}
		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
			securityFilterChains.add(securityFilterChainBuilder.build());
		}
		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
		if (httpFirewall != null) {
			filterChainProxy.setFirewall(httpFirewall);
		}
		filterChainProxy.afterPropertiesSet();

		Filter result = filterChainProxy;
		if (debugEnabled) {
			logger.warn("\n\n"
					+ "********************************************************************\n"
					+ "**********        Security debugging is enabled.       *************\n"
					+ "**********    This may include sensitive information.  *************\n"
					+ "**********      Do not use in a production system!     *************\n"
					+ "********************************************************************\n\n");
			result = new DebugFilter(filterChainProxy);
		}
		postBuildAction.run();
		return result;
	}
```
最终springSecurityFilterChain()方法返回的就是FilterChainProxy对象。继续看，在SecurityFilterAutoConfiguration配置类中通过Bean方法创建了DelegatingFilterProxyRegistrationBean组件。
```java
	@Bean
	@ConditionalOnBean(name = DEFAULT_FILTER_NAME)
	public DelegatingFilterProxyRegistrationBean securityFilterChainRegistration(
			SecurityProperties securityProperties) {
		DelegatingFilterProxyRegistrationBean registration = new DelegatingFilterProxyRegistrationBean(
				DEFAULT_FILTER_NAME);
		registration.setOrder(securityProperties.getFilter().getOrder());
		registration.setDispatcherTypes(getDispatcherTypes(securityProperties));
		return registration;
	}
```
DelegatingFilterProxyRegistrationBean的父类RegistrationBean实现了ServletContextInitializer接口的onStartup()方法。
```java
// RegistrationBea
	public final void onStartup(ServletContext servletContext) throws ServletException {
		String description = getDescription();
		if (!isEnabled()) {
			logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
			return;
		}
		register(description, servletContext);
	}
```
首先调用DelegatingFilterProxyRegistrationBean的父类AbstractFilterRegistrationBean#getDescription()方法，其实就是将targetBeanName注入到DelegatingFilterProxy中，this.targetBeanName的默认值为"springSecurityFilterChain"，即我们调用springSecurityFilterChain()方法创建的FilterChainProxy对象的name。
```java
// AbstractFilterRegistrationBean
	protected String getDescription() {
		Filter filter = getFilter();
		Assert.notNull(filter, "Filter must not be null");
		return "filter " + getOrDeduceName(filter);
	}
// DelegatingFilterProxyRegistrationBean
    public DelegatingFilterProxy getFilter() {
		//targetBeanName="springSecurityFilterChain"
		return new DelegatingFilterProxy(this.targetBeanName, getWebApplicationContext()) {

			@Override
			protected void initFilterBean() throws ServletException {
				// Don't initialize filter bean on init()
			}

		};
	}
```
回到onStartup()方法中的register()方法调用，其实就是调用AbstractFilterRegistrationBean#addRegistration()方法注册过滤器到servlet上下文，即将DelegatingFilterProxy过滤器添加到tomcat容器中。
```java
// DynamicRegistrationBean.java
	protected final void register(String description, ServletContext servletContext) {
		D registration = addRegistration(description, servletContext);
		if (registration == null) {
			logger.info(StringUtils.capitalize(description) + " was not registered (possibly already registered?)");
			return;
		}
		configure(registration);
	}
// AbstractFilterRegistrationBean.java 
	protected Dynamic addRegistration(String description, ServletContext servletContext) {
		Filter filter = getFilter();
		return servletContext.addFilter(getOrDeduceName(filter), filter);
	}
```
**总结一下，spring通过自动配置导入WebSecurityConfigurerAdapter扩展，然后实例化HttpSecurity，使用扩展对HttpSecurity进行配置，得到 完成配置后注入到DelegatingFilterProxy中，由DelegatingFilterProxyRegistrationBean添加到tomcat容器中，这样就能通过过滤器DelegatingFilterProxy找到FilterChainProxy完成过滤器的调用。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**
