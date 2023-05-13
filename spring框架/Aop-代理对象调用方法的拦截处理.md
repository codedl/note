通过cglib动态代理创建代理对象后调用目标方法时，就会进行拦截处理，先获取拦截器链，没有获取到则直接调用目标方法，获取到之后就会依次调用拦截器链上的方法。
```java
// DynamicAdvisedInterceptor
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			.....
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;

				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// We need to create a method invocation...
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				......
```
看下怎么获取拦截器链的，，就是获取每个通知关联的切点表达式，也就是切面类中使用了@PointCut的方法，然后判断要调用的类中的方法是否满足切点表达式，如果满足则将通知转换成MethodInterceptor。
```java
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {
......
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
			......
		}

		return interceptorList;
	}
```
接下来就是拦截器链的处理，使用CglibMethodInvocation类对拦截器链、目标方法、代理类进行封装，可以看到目标方法必须时public且不可以为Object中的方法。
```java
		public CglibMethodInvocation(Object proxy, @Nullable Object target, Method method,
				Object[] arguments, @Nullable Class<?> targetClass,
				List<Object> interceptorsAndDynamicMethodMatchers, MethodProxy methodProxy) {

			super(proxy, target, method, arguments, targetClass, interceptorsAndDynamicMethodMatchers);

			// Only use method proxy for public methods not derived from java.lang.Object
			this.methodProxy = (Modifier.isPublic(method.getModifiers()) &&
					method.getDeclaringClass() != Object.class && !AopUtils.isEqualsMethod(method) &&
					!AopUtils.isHashCodeMethod(method) && !AopUtils.isToStringMethod(method) ?
					methodProxy : null);
		}
```
此时会以递归的方式调用proceed()方法，依次执行拦截器链上的每个拦截器对应的通知方法，没有异常时执行顺序是：`@Around->@Before->@AfterReturning->@After->@Around`，发生异常时：`@Around->@Before->@AfterThrowing->@After`
```java
	public Object proceed() throws Throwable {
		// We start with an index of -1 and increment early.
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			return invokeJoinpoint();
		}

		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			// Evaluate dynamic method matcher here: static part will already have
			// been evaluated and found to match.
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
			if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// Dynamic matching failed.
				// Skip this interceptor and invoke the next in the chain.
				return proceed();
			}
		}
		else {
			// It's an interceptor, so we just invoke it: The pointcut will have
			// been evaluated statically before this object was constructed.
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
```