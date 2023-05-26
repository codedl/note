### 基于配置文件
首先是web.xml的方式，这种方式就是在xml中进行配置，不想多讲，感兴趣的可以找资料看下。

### 基于注解
对主类标记注解`@ServletComponentScan("com.example.springsource.nonblocking")`，然后在过滤器上使用`@WebServlet(name = "ServerServlet", urlPatterns = {"/server"}, asyncSupported = true)`，注意过滤器一定要实现servlet接口，下面看下这个的原理。
@ServletComponentScan会导入ServletComponentScanRegistrar组件，其实现了ImportBeanDefinitionRegistrar，registerBeanDefinitions()会被调用。这个方法就是确定Servlet的扫描路径，并且添加BeanFactory的后置处理器。
```java
	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
		if (registry.containsBeanDefinition(BEAN_NAME)) {
			updatePostProcessor(registry, packagesToScan);
		}
		else {
			addPostProcessor(registry, packagesToScan);
		}
	}
```
添加的后置处理器为ServletComponentRegisteringPostProcessor。
```java
	private void addPostProcessor(BeanDefinitionRegistry registry, Set<String> packagesToScan) {
		GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
		beanDefinition.setBeanClass(ServletComponentRegisteringPostProcessor.class);
		beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(packagesToScan);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(BEAN_NAME, beanDefinition);
	}
```
容器启动时会调用postProcessBeanFactory，对包路径进行扫描，然后使用ServletComponentHandler进行处理。
```java
	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		if (isRunningInEmbeddedWebServer()) {
			ClassPathScanningCandidateComponentProvider componentProvider = createComponentProvider();
			for (String packageToScan : this.packagesToScan) {
				scanPackage(componentProvider, packageToScan);
			}
		}
	}

	private void scanPackage(ClassPathScanningCandidateComponentProvider componentProvider, String packageToScan) {
		for (BeanDefinition candidate : componentProvider.findCandidateComponents(packageToScan)) {
			if (candidate instanceof AnnotatedBeanDefinition) {
				for (ServletComponentHandler handler : HANDLERS) {
					handler.handle(((AnnotatedBeanDefinition) candidate),
							(BeanDefinitionRegistry) this.applicationContext);
				}
			}
		}
	}
```
一共有三种ServletComponentHandler,分别对应Servlet、Filter、Listener，我们先看Servlet。
```java
	private static final List<ServletComponentHandler> HANDLERS;

	static {
		List<ServletComponentHandler> servletComponentHandlers = new ArrayList<>();
		servletComponentHandlers.add(new WebServletHandler());
		servletComponentHandlers.add(new WebFilterHandler());
		servletComponentHandlers.add(new WebListenerHandler());
		HANDLERS = Collections.unmodifiableList(servletComponentHandlers);
	}
```
好了，看到这里就明白了，其实就是添加一个ServletRegistrationBean，通过这个Bean去配置过滤器，进行注册。
```java
	@Override
	public void doHandle(Map<String, Object> attributes, AnnotatedBeanDefinition beanDefinition,
			BeanDefinitionRegistry registry) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.rootBeanDefinition(ServletRegistrationBean.class);
		builder.addPropertyValue("asyncSupported", attributes.get("asyncSupported"));
		builder.addPropertyValue("initParameters", extractInitParameters(attributes));
		builder.addPropertyValue("loadOnStartup", attributes.get("loadOnStartup"));
		String name = determineName(attributes, beanDefinition);
		builder.addPropertyValue("name", name);
		builder.addPropertyValue("servlet", beanDefinition);
		builder.addPropertyValue("urlMappings", extractUrlPatterns(attributes));
		builder.addPropertyValue("multipartConfig", determineMultipartConfig(beanDefinition));
		registry.registerBeanDefinition(name, builder.getBeanDefinition());
	}
```

### 手动注册
只要手动创建ServletRegistrationBean就行。
```java
    @Bean
    public ServletRegistrationBean serverServlet() {
        ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean();
        servletRegistrationBean.setServlet(new ServerServlet());
        servletRegistrationBean.addUrlMappings("/server");
        servletRegistrationBean.setAsyncSupported(true);
        return servletRegistrationBean;
    }
```
本质是调用抽象父类RegistrationBean的onStartup()方法，向web容器注册Servlet。
```java
	@Override
	public final void onStartup(ServletContext servletContext) throws ServletException {
		String description = getDescription();
		if (!isEnabled()) {
			logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
			return;
		}
		register(description, servletContext);
	}
```