+ 策略模式：spring对Bean进行实例化时用到了策略模式，此时默认使用Cglib对Bean进行实例化。
```java
// AbstractAutowireCapableBeanFactory.java
	protected BeanWrapper instantiateBean(String beanName, RootBeanDefinition mbd) {
		......
				beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, this);
			......
	}
    protected InstantiationStrategy getInstantiationStrategy() {
		return this.instantiationStrategy;
	}
    private InstantiationStrategy instantiationStrategy = new CglibSubclassingInstantiationStrategy();

```
+ 单例模式：spring使用ConcurrentHashMap缓存已经实例化的Bean，通过name获取时，得到的都是同一个Bean实例，因此使用了单例模式
+ 工厂模式：spring声明了用来创建Bean的接口FactoryBean，通过实现接口中声明的getObject()方法创建Bean，因此使用了工程模式。如果bean的定义是明确的，可以通过注解直接声明；但是有的Bean只有在spring运行时才能生成相关的定义，例如mybatis，在对xml标签进行解析后才能将mapper接口和xml标签关联，创建代理对象，因此可以使用FactoryBean实现
+ 适配器模式：执行aop方法时，会对目标代理对象的增强方法进行适配，得到aop的方法执行器链。
```java
// DefaultAdvisorChainFactory.java
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {

		// This is somewhat tricky... We have to process introductions first,
		// but we need to preserve order in the ultimate list.
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
		Advisor[] advisors = config.getAdvisors();
		List<Object> interceptorList = new ArrayList<>(advisors.length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		Boolean hasIntroductions = null;

		for (Advisor advisor : advisors) {
			if (advisor instanceof PointcutAdvisor) {
				// Add it conditionally.
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					boolean match;
					if (mm instanceof IntroductionAwareMethodMatcher) {
						if (hasIntroductions == null) {
							hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
						}
						match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
					}
					else {
						match = mm.matches(method, actualClass);
					}
					if (match) {
						MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
						if (mm.isRuntime()) {
							// Creating a new object instance in the getInterceptors() method
							// isn't a problem as we normally cache created chains.
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}
```
+ 代理模式：spring中的事务，aop，mybatis接口的代理对象，都是通过动态代理实现。
+ 观察者模式：spring的事件监听机制就是通过观察者模式实现的，先从spring.factories中读取事件监听器，添加到list集合中，然后通过事件广播器在运行的各个阶段发布不同的事件
+ 模板方式模式：spring的条件注解就是通过模板方法模式实现的，首先在父类SpringBootCondition中声明了通用的matches()方法调用不同子类的getMatchOutcome()，getMatchOutcome由不同的子类实现，比如OnClassCondition。