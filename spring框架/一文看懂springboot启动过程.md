这是我们创建springboot应用时写的代码，是不是很熟悉，但是这行代码背后的逻辑，springboot如何启动，spring应用上下文如何创建，今天我们一探究竟。
```java
public class AApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context =SpringApplication.run(AApplication.class, "name=ld");
    }
}
```
整体分两步，先创建spring应用上下文，再运行springboot容器。
```java
	public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
		return new SpringApplication(primarySources).run(args);
	}
```    
没做特殊处理，就是记住main方法类，这一步很关键，后面扫描bean时就是根据main类所在的包进行扫描的。然后决定应用类型，默认为servlet应用，即遵循web规范。从spring.factories读取初始化器和监听器，初始化器暂时不用关注，监听器我们主要关注`ConfigFileApplicationListener`和`LoggingApplicationListener`，后面会分开细说。
```java
	public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        //设置资源加载器，默认null
		this.resourceLoader = resourceLoader;
		Assert.notNull(primarySources, "PrimarySources must not be null");
        //记录main类
		this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        //决定应用类型
		this.webApplicationType = WebApplicationType.deduceFromClasspath();
        //从spring.factories中读取初始化器
		setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        //从spring.factories中读取监听器
		setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
		this.mainApplicationClass = deduceMainApplicationClass();
	}
```
spring的源码代码量非常大，我们看的时候能看懂框架设计就行，能看到具体功能的时候再去详细讲解，主要看懂配置加载、bean实例化、aop、事务、mvc这些就够了，后面我会细讲。至于springboot总体流程是：
1. 读取用于发布事件的通知器
2. 打印banner
3. 注册部分必要bean
4. bean实例化，启动web容器
5. 调用 ApplicationRunner 和 CommandLineRunner 接口实现类
```java
	public ConfigurableApplicationContext run(String... args) {
        //打印启动时间
		StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
        //读取事件发布监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //配置环境
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
            //打印banner
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
            //注册必要bean
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
            //bean实例化，启动web容器，默认tomcat
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
            //调用 ApplicationRunner 和 CommandLineRunner 接口实现类
			callRunners(context, applicationArguments);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, listeners);
			throw new IllegalStateException(ex);
		}

		try {
			listeners.running(context);
		}
		catch (Throwable ex) {
			handleRunFailure(context, ex, exceptionReporters, null);
			throw new IllegalStateException(ex);
		}
		return context;
	}
```
**总结下，springboot启动时会读取spring.factories配置的初始化器和监听器，然后启动springboot容器，创建bean，启动内置web容器。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步，更多请关注微信公众号 葡萄开源**