---
title: @Async注解源码
tags:
notebook: spring源码解析
---
spring是支持异步任务的，做法就是先用@EnableAsync标记主类，再创建一个实现了Executor接口的Bean作为线程池，最后在需要异步执行的方法上使用@Async(value = "线程池Bean名称")注解就行。现在看下spring是怎么实现的。

从@EnableAsync注解开始看起，可以看到这个注解上使用了@Import导入了AsyncConfigurationSelector组件。
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {
	Class<? extends Annotation> annotation() default Annotation.class;
	boolean proxyTargetClass() default false;
	AdviceMode mode() default AdviceMode.PROXY;
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```
AsyncConfigurationSelector的父类AdviceModeImportSelector实现了ImportSelector接口，可以通过selectImports()方法注册Bean定义，在子类AsyncConfigurationSelector中通过方法selectImports()注册了ProxyAsyncConfiguration类型的Bean。
```java
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {ProxyAsyncConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME};
			default:
				return null;
		}
	}
```
接下来再来分析ProxyAsyncConfiguration，这里使用@Bean方法创建了AsyncAnnotationBeanPostProcessor类型的Bean，顾名思义就是一个异步注解处理器，正是这个Bean对@Async注解的方法进行增强的，接下来看下这个过程。查看AsyncAnnotationBeanPostProcessor类的继承关系知道，其实现了BeanFactoryAware和BeanPostProcessor接口，这两个接口在创建Bean的过程中都会被调用的。spring会先初始化BeanPostProcessor类型的Bean，然后在其他的Bean初始化完成后进行后置处理。而在Bean初始化过程中，会调用Bean本身实现的BeanFactoryAware接口中声明的方法。接下来再看这两个接口怎么被调用的即可。setBeanFactory()会将当前容器保存下来，再创建一个增强器AsyncAnnotationAdvisor。这个增强器有两个重要的地方：
1. 由AnnotationAsyncExecutionInterceptor实现的Advice通知，这个拦截器在后来执行@Async方法时进行拦截和增强。
2. Pointcut切点，用来对@Async注解进行处理，发现类中的方法使用了@Async注解就会生成代理对象。
```java
	public void setBeanFactory(BeanFactory beanFactory) {
		super.setBeanFactory(beanFactory);

		AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
		if (this.asyncAnnotationType != null) {
			advisor.setAsyncAnnotationType(this.asyncAnnotationType);
		}
		advisor.setBeanFactory(beanFactory);
		this.advisor = advisor;
	}

    public AsyncAnnotationAdvisor(
			@Nullable Supplier<Executor> executor, @Nullable Supplier<AsyncUncaughtExceptionHandler> exceptionHandler) {

		Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet<>(2);
		asyncAnnotationTypes.add(Async.class);
		try {
			asyncAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.ejb.Asynchronous", AsyncAnnotationAdvisor.class.getClassLoader()));
		}
		catch (ClassNotFoundException ex) {
			// If EJB 3.1 API not present, simply ignore.
		}
        //通知会对@Async方法进行拦截和增强
		this.advice = buildAdvice(executor, exceptionHandler);
        // 切点@Async注解进行处理，创建@Async方法所在类的代理对象
		this.pointcut = buildPointcut(asyncAnnotationTypes);
	}
```
再来看父类AbstractAdvisingBeanPostProcessor中实现的postProcessAfterInitialization()怎么对其他的Bean进行处理的。
```java
    public Object postProcessAfterInitialization(Object bean, String beanName) {
		if (this.advisor == null || bean instanceof AopInfrastructureBean) {
			// Ignore AOP infrastructure such as scoped proxies.
			return bean;
		}
        // 如果已经被增强过，则不需要创建代理对象，直接添加增强器
		if (bean instanceof Advised) {
			Advised advised = (Advised) bean;
			if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
				// Add our local Advisor to the existing proxy's Advisor chain...
				if (this.beforeExistingAdvisors) {
					advised.addAdvisor(0, this.advisor);
				}
				else {
					advised.addAdvisor(this.advisor);
				}
				return bean;
			}
		}
        // 判断是否可以对Bean进行增强
		if (isEligible(bean, beanName)) {
			ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
			if (!proxyFactory.isProxyTargetClass()) {
				evaluateProxyInterfaces(bean.getClass(), proxyFactory);
			}
            // 添加增强器
			proxyFactory.addAdvisor(this.advisor);
			customizeProxyFactory(proxyFactory);
            // 创建代理对象
			return proxyFactory.getProxy(getProxyClassLoader());
		}

		// No proxy needed.
		return bean;
	}
```
看下创建代理对象的过程吧。
1. 判断当前Bean是否可以进行增强，最终通过AopUtils.java类中的canApply()方法判断是否能够应用增强器。
```java
    public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
            // 获取切点表达式，判断是否能增强
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			return true;
		}
	}

    public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		MethodMatcher methodMatcher = pc.getMethodMatcher();
		if (methodMatcher == MethodMatcher.TRUE) {
			// No need to iterate the methods if we're matching any method anyway...
			return true;
		}

		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
            // 对类中的方法进行遍历，判断是否有方法使用了@Async
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}
```
2. 再通过代理工厂生成Bean的代理对象，就是cglib动态代理。
```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    if (logger.isTraceEnabled()) {
        logger.trace("Creating CGLIB proxy: " + this.advised.getTargetSource());
    }

    try {
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

        Class<?> proxySuperClass = rootClass;
        if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
            proxySuperClass = rootClass.getSuperclass();
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }

        // Validate the class, writing log messages as necessary.
        validateClassIfNecessary(proxySuperClass, classLoader);

        // Configure CGLIB Enhancer...
        Enhancer enhancer = createEnhancer();
        if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                    ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                enhancer.setUseCache(false);
            }
        }
        enhancer.setSuperclass(proxySuperClass);
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        // fixedInterceptorMap only populated at this point, after getCallbacks call above
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
                this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        // Generate the proxy class and create a proxy instance.
        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (CodeGenerationException | IllegalArgumentException ex) {
        throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                ": Common causes of this problem include using a final class or a non-visible class",
                ex);
    }
    catch (Throwable ex) {
        // TargetSource.getTarget() failed
        throw new AopConfigException("Unexpected AOP exception", ex);
    }
}
```
得到代理对象后，就是通过代理对象调用方法，可以对这个过程进行逻辑增强，这个时候就感受到动态代理的意义所在了。
```java
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
		final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
        // 通过Method获取对应的执行这个方法的线程池
		AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
		if (executor == null) {
			throw new IllegalStateException(
					"No executor specified and no default executor set on AsyncExecutionInterceptor either");
		}
        // 创建执行异步方法的任务，由Callable接口的lambda表达式实现
		Callable<Object> task = () -> {
			try {
                // 调用目标方法
				Object result = invocation.proceed();
				if (result instanceof Future) {
					return ((Future<?>) result).get();
				}
			}
			catch (ExecutionException ex) {
				handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
			}
			catch (Throwable ex) {
				handleError(ex, userDeclaredMethod, invocation.getArguments());
			}
			return null;
		};
        // 提交任务到线程池，接下来就是线程池的事情啦
		return doSubmit(task, executor, invocation.getMethod().getReturnType());
	}
```
整个过程就是通过Method获取对应的执行这个方法的线程池，创建执行异步方法的任务。就是使用lambda表达式实现Callable接口，这个是动态的，可以调用不用的方法，又是一个通用的实现，非常巧妙由Callable接口的lambda表达式实现。然后再提交任务。看下怎么找到能够执行异步方法的线程池吧。
```java
	protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
        // 先从缓存中获取，获取不到再去找，找到后再缓存，spring就喜欢这么干
		AsyncTaskExecutor executor = this.executors.get(method);
		if (executor == null) {
			Executor targetExecutor;
            // 获取方法上@Async注解的属性value字符串值
			String qualifier = getExecutorQualifier(method);
			if (StringUtils.hasLength(qualifier)) {
                // 到容器中根据value的值找到实现了Executor接口的Bean作为线程池
				targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
			}
			else {
				targetExecutor = this.defaultExecutor.get();
			}
			if (targetExecutor == null) {
				return null;
			}
            // 这里使用适配器模式封装成TaskExecutorAdapter对象
			executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
					(AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
            // 缓存
			this.executors.put(method, executor);
		}
		return executor;
	}
```
总结一下，首先创建一个Bean后置处理器，对含有@Async方法的Bean增强生成代理对象。然后执行方法时进行拦截和增强，创建异步任务提交到线程池。