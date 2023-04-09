用来开启事务的注解@EnableTransactionManagement上通过@Import导入了TransactionManagementConfigurationSelector组件，TransactionManagementConfigurationSelector类的父类AdviceModeImportSelector实现了ImportSelector接口，因此会调用`public final String[] selectImports(AnnotationMetadata importingClassMetadata)`方法注册Bean定义，此时会调用子类TransactionManagementConfigurationSelector中的selectImports()方法导入AutoProxyRegistrar和ProxyTransactionManagementConfiguration组件。
```java
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
```
属性mode的默认值是AdviceMode.PROXY，因此回导入AutoProxyRegistrar组件和ProxyTransactionManagementConfiguration组件。
```java
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}
```
AutoProxyRegistrar实现了ImportBeanDefinitionRegistrar接口中声明的registerBeanDefinitions()方法，此时会向容器中注册`InfrastructureAdvisorAutoProxyCreator`组件，这个组件的作用就是找到当前容器中的通知，再使用通知创建目标类的代理对象。  
ProxyTransactionManagementConfiguration类是个使用了@Configuration注解的配置，通过@Bean注解创建了
+ BeanFactoryTransactionAttributeSourceAdvisor
+ TransactionAttributeSource
+ TransactionInterceptor。
```java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
			TransactionAttributeSource transactionAttributeSource, TransactionInterceptor transactionInterceptor) {

		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		......
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor(TransactionAttributeSource transactionAttributeSource) {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		......
	}

}
```
InfrastructureAdvisorAutoProxyCreator类的父类AbstractAutoProxyCreator实现了SmartInstantiationAwareBeanPostProcessor接口的postProcessBeforeInstantiation()，因此在Bean实例化的时候会被调用。shouldSkip()中会调用BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans()方法从容器找到实现了Advisor接口的Bean，即`BeanFactoryTransactionAttributeSourceAdvisor`。
```java
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
		Object cacheKey = getCacheKey(beanClass, beanName);

		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}
.....
	}
```
读过aop的源码后我们知道，每个通知都包含切点表达式和拦截器，BeanFactoryTransactionAttributeSourceAdvisor的拦截器就是TransactionInterceptor，此拦截器会在调用目标方法时生效，而切点表达式Pointcut为
```java
	private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
		@Override
		@Nullable
		protected TransactionAttributeSource getTransactionAttributeSource() {
			return transactionAttributeSource;
		}
	};
```
InfrastructureAdvisorAutoProxyCreator类的父类AbstractAutoProxyCreator实现了BeanPostProcessor接口的postProcessAfterInitialization()方法，创建代理对象的过程与aop类似，获取能作用在当前Bean上面的通知，再使用通知生成代理对象。
```java
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		......

		// Create proxy if we have advice.
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		if (specificInterceptors != DO_NOT_PROXY) {
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			Object proxy = createProxy(
					bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}
```
首先看通知的获取，首先获取容器内所有的通知，然后通过AopUtils#canApply()找到能应用在当前方法上的通知。
```java
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
事务对应的切点为TransactionAttributeSourcePointcut，TransactionAttributeSourcePointcut的父类StaticMethodMatcherPointcut#getMethodMatcher()返回的就是TransactionAttributeSourcePointcut实例对象本身。
```java
// StaticMethodMatcherPointcut
	@Override
	public final MethodMatcher getMethodMatcher() {
		return this;
	}
```
然后调用自身的TransactionAttributeSourcePointcut#matches()方法对目标类和目标方法进行判断，是否可以应用通知。getTransactionAttributeSource()返回的是注入到BeanFactoryTransactionAttributeSourceAdvisor组件中的TransactionAttributeSource属性，默认是AnnotationTransactionAttributeSource实现的。
```java
	public boolean matches(Method method, Class<?> targetClass) {
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}
```
然后调用AnnotationTransactionAttributeSource的父类AbstractFallbackTransactionAttributeSource#getTransactionAttribute()获取作用在方法上的`@Transactional`注解属性。
```java
	public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		......
		else {
			// We need to work it out.
			TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
			// Put it in the cache.
			if (txAttr == null) {
				this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
			}
			else {
				String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
				if (txAttr instanceof DefaultTransactionAttribute) {
					((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
				}
				this.attributeCache.put(cacheKey, txAttr);
			}
			return txAttr;
		}
	}
```
首先尝试获取作用在方法上的`@Transactional`，没有获取到则获取作用在类上的`@Transactional`注解，
```java
	protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// Don't allow no-public methods as required.
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// The method may be on an interface, but we need attributes from the target class.
		// If the target class is null, the method will be unchanged.
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// First try is the method in the target class.
		TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
		if (txAttr != null) {
			return txAttr;
		}

		// Second try is the transaction attribute on the target class.
		txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
		if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
			return txAttr;
		}

		if (specificMethod != method) {
			// Fallback is to look at the original method.
			txAttr = findTransactionAttribute(method);
			if (txAttr != null) {
				return txAttr;
			}
			// Last fallback is the class of the original method.
			txAttr = findTransactionAttribute(method.getDeclaringClass());
			if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
				return txAttr;
			}
		}

		return null;
	}
```
AnnotationTransactionAttributeSource内部存在SpringTransactionAnnotationParser用来解析`@Transactional`注解，使用RuleBasedTransactionAttribute封装`@Transactional`注解属性。
```java
// SpringTransactionAnnotationParser.java
	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();

		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());
		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		rbta.setQualifier(attributes.getString("value"));

		List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
		for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		rbta.setRollbackRules(rollbackRules);

		return rbta;
	}
```
至此我们找到了可以应用的通知，接下来就是代理对象的创建了，设置拦截器为DynamicAdvisedInterceptor，当通过代理对象调用方法时就会调用intercept()进行拦截。
```java
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}

		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);

		if (!proxyFactory.isProxyTargetClass()) {
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}

		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		proxyFactory.addAdvisors(advisors);
		proxyFactory.setTargetSource(targetSource);
		customizeProxyFactory(proxyFactory);

		proxyFactory.setFrozen(this.freezeProxy);
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}

		return proxyFactory.getProxy(getProxyClassLoader());
	}
```
