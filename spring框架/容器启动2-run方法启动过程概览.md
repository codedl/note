---
title: 容器启动2-run方法启动过程概览
tags:
notebook: spring源码解析
---

<font size=4>\>SpringApplication.java#run  
spring中IoC容器在run方法启动的
```
public ConfigurableApplicationContext run(String... args) {
		StopWatch stopWatch = new StopWatch();
        //开始计时
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();
		configureHeadlessProperty();
        //获取spring.factories中定义的事件发布运行监听器
		SpringApplicationRunListeners listeners = getRunListeners(args);
        /*
            通过广播器发布starting启动事件，对starting事件感兴趣的监听器有:
            LoggingApplicationListener 
            BackgroundPreinitializer 
            DelegatingApplicationListener 
            LiquibaseServiceLocatorApplicationListener 
        */
		listeners.starting();
		try {
            //对命令行参数进行封装
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            //准备环境，配置属性源
			ConfigurableEnvironment environment = prepareEnvironment(listeners, applicationArguments);
			configureIgnoreBeanInfo(environment);
			//打印banner，可以用banner.txt设置banner
			Banner printedBanner = printBanner(environment);
			//创建内置容器的应用上下文
			context = createApplicationContext();
			//获取并打印Spring启动过程中的异常信息
			exceptionReporters = getSpringFactoriesInstances(SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			//为应用配置，加载主类		
			prepareContext(context, environment, listeners, applicationArguments, printedBanner);
			//通过容器的后置处理器对单例bean进行扫描，创建单例bean的实例对象
			refreshContext(context);
			//发布容器启动完成的相关事件
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
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
<font>