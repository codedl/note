---
title: AOP4-执行通知方法
tags:
notebook: spring源码解析
---
在通过cglib动态代理创建代理对象时设置了DynamicAdvisedInterceptor作为回调，DynamicAdvisedInterceptor实现了接口MethodInterceptor中声明的[intercept](#intercept)方法，通知方法就是在这里执行的。\
获取拦截器链：  
1. 获取被代理的对象及Class
2. 通过目标方法Method和目标类Class[获取拦截器链](#getinterceptorsanddynamicinterceptionadvice)
   1. 首先到缓存Map中获取目标方法method的拦截器链，获取到的话就直接返回
      1. 使用[MethodCacheKey](#methodcachekey)对Method进行封装，MethodCacheKey类重写了hashCode方法
      2. 使用ConcurrentHashMap保存MethodCacheKey到拦截器链的映射关系
      3. 保存数据时，会先构造MethodCacheKey的实例对象作为key，构造时会计算这个对象的hashCode值，然后构造拦截器链作为value，之后再通过hashCode确定作为value的拦截器链在数组链表中的位置，进而保存数据
      4. 查询数据时，通过计算作为key的MethodCacheKey实例对象的hashCode值，确定value的位置后获取value
   2. [没有获取到则创建拦截器链](#getinterceptorsanddynamicinterceptionadvice)
      1. [获取适配器registry](#获取适配器注册器registry的单实例对象)
         1. 在DefaultAdvisorAdapterRegistry中定义一个存储适配器的ArrayList集合
         2. 构造DefaultAdvisorAdapterRegistry实例向集合添加三个适配器实例
         3. 将DefaultAdvisorAdapterRegistry的实例保存到GlobalAdvisorAdapterRegistry的静态私有变量instance
      2. 创建保存拦截器MethodIntercepquedtor的集合interceptorList，也就是拦截器链
      3. 遍历已有的增强器，如果是普通增强器PointcutAdvisor
         1. 判断是否可以对当前类进行增强
         2. 通过切点表达式获取方法匹配器，判断目标方法是否可以被增强
         3. 如果能够被增强，[通过增强器生成拦截器](#getinterceptors)
            1. 从增强器中获取保存的通知
            2. 如果是实现了MethodInterceptor接口的通知，直接转型成MethodInterceptor接口
            3. 通过适配器使用通知创建拦截器
               1. 先判断是否支持适配，即适配器内部构造实例时，类中的实例字段的类型和通知的类型是否一致，如果是一致或存在继承关系，则可以将通知对象赋值给实例字段
               2. 对实现了MethodInterceptor接口的类实例化，将通知对象赋值给实例字段
      4. 如果是对类增强的引介增强器IntroductionAdvisor
      5. 如果是没有实现接口的增强器
   3. 将Method到拦截器链的映射关系缓存到ConcurrentHashMap
------
原理：先调用拦截器的通知方法，在递归地调用proceed方法，proceed方法中会依次沿拦截器链向后移动直到目标方法被调用；目标方法调用完后再反向地调用通知方法。
执行通知方法的过程：
1. 判断拦截器链是否为空，如果为空则说明没有拦截器，以反射方式调用目标方法
2. [调用拦截器链中的拦截器方法](#调用拦截器方法)
   1. 创建CglibMethodInvocation方法调用对象，被增强的方法必须是public的，不可以为Object对象中的方法增强，不可以为equals方法、hashCode方法、toString方法增强proce
   2. **执行CglibMethodInvocation方法调用对象中的proceed方法**
   3. 起始索引currentInterceptorIndex为-1，如果currentInterceptorIndex值为拦截器链中最后一个拦截器,则表示拦截器方法调用完了，执行目标方法
   4. 索引currentInterceptorIndex+1=0，[获取第一个拦截器](#拦截器链中第一个拦截器的由来)
   5. 调用*第一个拦截器中的[invoke](#exposeinvocationinterceptor-invoke)方法*
      1. 记录ThreadLocal线程变量invocation的值
      2. 设置invocation的值为当前的CglibMethodInvocation方法调用对象
      3. **执行CglibMethodInvocation方法调用对象中的proceed方法**
         1. 索引currentInterceptorIndex+1=1，获取第二个拦截器AspectJAroundAdvice
         2. 调用*第二个拦截器的[invoke](#aspectjaroundadvice-invoke)方法*
            1. 生成目标方法的连接点
               1. 对实现了ProceedingJoinPoint接口的MethodInvocationProceedingJoinPoint类进行实例化
               2. 用属性methodInvocation记录当前的CglibMethodInvocation方法调用对象
            2. 获取切点表达式
            3. 处理参数，如果@Around方法的第一个参数为JoinPoint类型，则注入连接点对象MethodInvocationProceedingJoinPoint
               1. 使用传给方法的参数构造CglibMethodInvocation方法调用对象
               2. 对参数进行处理，生成Object[]表示每个参数构成的数组
               3. 通过属性arguments保存参数数组
               4. 通过getArgs获取参数
            4. [反射调用@Around注解标注的方法](#invokeadvicemethodwithgivenargs)，方法参数为连接点
            5. 调用连接点中的proceed方法，即MethodInvocationProceedingJoinPoint中的proceed方法，proceed方法中通过methodInvocation属性找到CglibMethodInvocation方法调用对象，**执行CglibMethodInvocation方法调用对象中的proceed方法**
               1. 索引currentInterceptorIndex+1=2，获取第三个拦截器AspectJMethodBeforeAdvice
               2. 调用*第三个拦截器的invoke方法*，方法内调用before方法
                  1. 获取被增强方法，被增强方法的参数，代理对象，作为before方法的参数
                  2. 利用第一个拦截器从ThreadLocad中获取CglibMethodInvocation方法调用对象
                  3. 处理参数，如果@Before方法的第一个参数为JoinPoint类型，则注入连接点对象
                  4. 反射调用@Before注解标注的方法，方法参数为连接点
                  5. @Before方法调用完成后，**执行CglibMethodInvocation方法调用对象中的proceed方法**
                     1. 索引currentInterceptorIndex+1=3，获取第四个拦截器AspectJAfterAdvice
                     2. 调用*第四个拦截器的invoke方法*， invoke方法中**执行CglibMethodInvocation方法调用对象中的proceed方法**
                        1. 索引currentInterceptorIndex+1=4，获取第五个拦截器AspectJAfterReturningAdvice
                        2. 执行*第五个拦截器的invoke方法*
                           1. **执行CglibMethodInvocation方法调用对象中的proceed方法**
                           2. 索引currentInterceptorIndex+1=4，获取第六个拦截器AspectJAfterThrowingAdvice
                           3. 调用*第六个拦截器的invoke方法*
                              1. **执行CglibMethodInvocation方法调用对象中的proceed方法**
                              2. 此时currentInterceptorIndex索引值为5，拦截器链所有拦截器方法调用完成，调用invokeJoinpoint方法以反射方式调用目标方法，返回目标方法返回值；如果发生异常，则抛出异常
                              3. 目标方法返回值返回给invokeJoinpoint方法；抛出异常到invokeJoinpoint方法
                              4. invokeJoinpoint方法返回值返回给CglibMethodInvocation方法调用对象中的proceed方法；抛出异常到proceed方法
                              5. proceed方法调用完成
                              6. proceed方法的返回值返回给第六个拦截器的invoke方法；抛出异常到invoke方法
                           4. 第六个拦截器的invoke方法调用完成；catch块中处理异常，执行异常通知方法，即@AfterThrowing注解方法
                              1. 使用throwingName属性记录注解属性throwing的值
                              2. 在注解方法中声明一个和属性throwing的值同名的参数
                              3. 将参数名和参数的索引推入一个Map
                              4. 根据索引获取参数的类型
                              5. 判断参数的类型与抛出的异常类型是否相同
                           5. 第六个拦截器的invoke方法的返回值返回给proceed方法，proceed方法调用完成；抛出异常给proceed方法
                           6. 回到第五个拦截器的invoke方法；
                        3. 这里没有捕获异常，不会进行异常处理，如果发生了异常不作处理；
                        4. [没有异常时判断目标方法的返回值类型和参数类型是否相同](#shouldinvokeonreturnvalueof)
                           1. 获取@AfterReturning注解中属性returning的值
                           2. @AfterReturning注解方法必须声明returning的值同名的参数
                           3. 获取此参数的类型
                           4. 将参数类型与返回值类型判断，如果相同则进行处理，不同则跳过，直接回到第四个拦截器的invoke方法
                        5. 否则注入连接点，反射调用通知方法，即@AfterReturning注解方法
                        6. 第五个拦截器的invoke方法调用完成，invoke方法的方法值返回给proceed方法，proceed方法调用完成
                        7. 回到第四个拦截器的invoke方法
                     3. invoke方法执行finally代码块，即便发生了异常也会执行；注入连接点，反射调用通知方法，即@After注解方法
                     4. finally代码块执行完成，第四个拦截器的invoke方法调用完成，invoke方法的方法值返回给proceed方法，proceed方法调用完成
                     5. 回到第三个拦截器的invoke方法
                  6. proceed方法调用完成，将返回值返回给invoke方法
                  7. 第三个拦截器的invoke方法调用完成，将返回值返回给proceed方法
                  8. 回到第二个拦截器的invoke方法，即@Around方法
            6. 获取连接点的proceed方法的返回值，proceed方法调用完成
            7. 继续执行@Around方法剩下来的代码
            8. @Around方法调用完成，将返回值返回给invoke方法
         3. 第二个拦截器的invoke方法调用完成，将返回值返回值给proceed方法
      4. 将记录ThreadLocal线程变量invocation的值还原
      5. 第一个拦截器的invoke方法调用完成
   6. proceed方法调用完成，拦截器链中所有拦截器方法调用完成


# intercept
```
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        Object oldProxy = null;
        boolean setProxyContext = false;
        Object target = null;
        TargetSource targetSource = this.advised.getTargetSource();
        try {
            if (this.advised.exposeProxy) {
                // Make invocation available if necessary.
                oldProxy = AopContext.setCurrentProxy(proxy);
                setProxyContext = true;
            }
            // Get as late as possible to minimize the time we "own" the target, in case it comes from a pool...
            target = targetSource.getTarget();
            Class<?> targetClass = (target != null ? target.getClass() : null);
            //生成拦截器链
            List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
            Object retVal;
            // Check whether we only have one InvokerInterceptor: that is,
            // no real advice, but just reflective invocation of the target.
            if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
                // We can skip creating a MethodInvocation: just invoke the target directly.
                // Note that the final invoker must be an InvokerInterceptor, so we know
                // it does nothing but a reflective operation on the target, and no hot
                // swapping or fancy proxying.
                Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
                retVal = methodProxy.invoke(target, argsToUse);
            }
            else {
                // We need to create a method invocation...
                retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
            }
            retVal = processReturnType(proxy, target, method, retVal);
            return retVal;
        }
        finally {
            if (target != null && !targetSource.isStatic()) {
                targetSource.releaseTarget(target);
            }
            if (setProxyContext) {
                // Restore old proxy.
                AopContext.setCurrentProxy(oldProxy);
            }
        }
    }
```
# MethodCacheKey
```
private static final class MethodCacheKey implements Comparable<MethodCacheKey> {
    private final int hashCode;
    public MethodCacheKey(Method method) {
        this.method = method;
        this.hashCode = method.hashCode();
    }
    @Override
    public int hashCode() {
        return this.hashCode;
    }
}
```

# getInterceptorsAndDynamicInterceptionAdvice
```
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```
# getInterceptorsAndDynamicInterceptionAdvice
```
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, @Nullable Class<?> targetClass) {

    // This is somewhat tricky... We have to process introductions first,
    // but we need to preserve order in the ultimate list.
    //创建适配器
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
# getInterceptors
生成拦截器
```
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<>(3);
    Advice advice = advisor.getAdvice();
    //如果增强器实现了MethodInterceptor接口，直接转型添加到集合
    if (advice instanceof MethodInterceptor) {
        interceptors.add((MethodInterceptor) advice);
    }
    //通过适配器生成增强器
    for (AdvisorAdapter adapter : this.adapters) {
        if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
        }
    }
    if (interceptors.isEmpty()) {
        throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[0]);
}
```
# 获取适配器注册器registry的单实例对象
```
AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
public static AdvisorAdapterRegistry getInstance() {
    return instance;
}
private static AdvisorAdapterRegistry instance = new DefaultAdvisorAdapterRegistry();
public DefaultAdvisorAdapterRegistry() {
    registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
    registerAdvisorAdapter(new AfterReturningAdviceAdapter());
    registerAdvisorAdapter(new ThrowsAdviceAdapter());
}
public void registerAdvisorAdapter(AdvisorAdapter adapter) {
    this.adapters.add(adapter);
}
private final List<AdvisorAdapter> adapters = new ArrayList<>(3);
```
# 调用拦截器方法
```
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
# 拦截器链中第一个拦截器的由来
```
>AbstractAdvisorAutoProxyCreator.java#findEligibleAdvisors
>AspectJAwareAdvisorAutoProxyCreator.java#extendAdvisors
>AspectJProxyUtils.java#makeAdvisorChainAspectJCapableIfNecessary
public static boolean makeAdvisorChainAspectJCapableIfNecessary(List<Advisor> advisors) {
... 
    advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
...
}
public static final Advisor ADVISOR = new DefaultPointcutAdvisor(INSTANCE)
public static final ExposeInvocationInterceptor INSTANCE = new ExposeInvocationInterceptor();
public final class ExposeInvocationInterceptor implements MethodInterceptor, PriorityOrdered, Serializable {...}
```
# ExposeInvocationInterceptor-invoke
```
public Object invoke(MethodInvocation mi) throws Throwable {
    MethodInvocation oldInvocation = invocation.get();
    invocation.set(mi);
    try {
        return mi.proceed();
    }
    finally {
        invocation.set(oldInvocation);
    }
}
```
# AspectJAroundAdvice-invoke
```
public Object invoke(MethodInvocation mi) throws Throwable {
    if (!(mi instanceof ProxyMethodInvocation)) {
        throw new IllegalStateException("MethodInvocation is not a Spring ProxyMethodInvocation: " + mi);
    }
    ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
    ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
    JoinPointMatch jpm = getJoinPointMatch(pmi);
    return invokeAdviceMethod(pjp, jpm, null, null);
}
```
# invokeAdviceMethodWithGivenArgs
```
protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
...
    return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
...
@Around("pointcut()")
//参数为连接点对象
    Object around(ProceedingJoinPoint joinPoint) throws Throwable {
    System.out.println("@Around前..."+joinPoint.getSignature().getName());
    Object result = joinPoint.proceed();
    System.out.println("@Around后..."+joinPoint.getSignature().getName());
    return result;
}
}
```
# MethodInvocationProceedingJoinPoint
```
public class MethodInvocationProceedingJoinPoint implements ProceedingJoinPoint, JoinPoint.StaticPart{}
//methodInvocation指向方法调用对象的
private final ProxyMethodInvocation methodInvocation;
public MethodInvocationProceedingJoinPoint(ProxyMethodInvocation methodInvocation) {
    this.methodInvocation = methodInvocation;
}
//本质是执行方法调用对象的proceed方法
public Object proceed() throws Throwable {
    return this.methodInvocation.invocableClone().proceed();
}
```
# MethodBeforeAdviceInterceptor-invoke
```
//方法第一个参数为被增强方法
//第二个参数为被增强方法参数
//第三个参数为代理对象
public Object invoke(MethodInvocation mi) throws Throwable {
    this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
    return mi.proceed();
}
```
# AspectJAfterAdvice-invoke
```
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        return mi.proceed();
    }
    finally {
        invokeAdviceMethod(getJoinPointMatch(), null, null);
    }
}
```
# shouldInvokeOnReturnValueOf
```
//returnValue为返回值
if (shouldInvokeOnReturnValueOf(method, returnValue)) {
    invokeAdviceMethod(getJoinPointMatch(), returnValue, null);
}
private boolean shouldInvokeOnReturnValueOf(Method method, @Nullable Object returnValue) {
    Class<?> type = getDiscoveredReturningType();
    Type genericType = getDiscoveredReturningGenericType();
    // If we aren't dealing with a raw type, check if generic parameters are assignable.
    //将返回值类型与参数类型进行比较，如果相同则返回true
    return (matchesReturnValue(type, method, returnValue) &&
            (genericType == null || genericType == type ||
                    TypeUtils.isAssignable(genericType, method.getGenericReturnType())));
}
//获取@AfterReturning注解方法声明的参数类型
protected Class<?> getDiscoveredReturningType() {
    return this.discoveredReturningType;
}
private void bindExplicitArguments(int numArgumentsLeftToBind) {
......		
    Integer index = this.argumentBindings.get(this.returningName);
    this.discoveredReturningType = this.aspectJAdviceMethod.getParameterTypes()[index];
    this.discoveredReturningGenericType = this.aspectJAdviceMethod.getGenericParameterTypes()[index];
......
}	
```
# shouldInvokeOnThrowing
```
>ReflectiveAspectJAdvisorFactory.java#getAdvice
//生成异常通知，使用throwingName属性记录注解属性throwing的值		
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
			MetadataAwareAspectInstanceFactory aspectInstanceFactory, int declarationOrder, String aspectName) {
    case AtAfterThrowing:
        springAdvice = new AspectJAfterThrowingAdvice(
                candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
        AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
        if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
            springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
        }
    break;
}
>AbstractAspectJAdvice.java#bindArgumentsByName
//获取注解方法声明的参数
private void bindArgumentsByName(int numArgumentsExpectingToBind) {
......
    this.argumentNames = createParameterNameDiscoverer().getParameterNames(this.aspectJAdviceMethod);
......
}
>AbstractAspectJAdvice.java#bindExplicitArguments
{
......
//将参数名和参数在方法头中的索引推入一个Map
//private Map<String, Integer> argumentBindings;
    for (int i = argumentIndexOffset; i < this.argumentNames.length; i++) {
        this.argumentBindings.put(this.argumentNames[i], i);
    }
......
    if (this.throwingName != null) {
......
        //根据参数名获取参数在方法中的索引
        Integer index = this.argumentBindings.get(this.throwingName);
        //由参数的索引获取此参数的类型，保存到discoveredThrowingType属性
        this.discoveredThrowingType = this.aspectJAdviceMethod.getParameterTypes()[index];
......
    }
}
//判断discoveredThrowingType属性指定的类型与异常的类型是否相同，如果相同则进行处理
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        return mi.proceed();
    }
    catch (Throwable ex) {
        if (shouldInvokeOnThrowing(ex)) {
            invokeAdviceMethod(getJoinPointMatch(), null, ex);
        }
        throw ex;
    }
}
private boolean shouldInvokeOnThrowing(Throwable ex) {
    return getDiscoveredThrowingType().isAssignableFrom(ex.getClass());
}
protected Class<?> getDiscoveredThrowingType() {
    return this.discoveredThrowingType;
}
```
```
//获取arguments属性的值即保存的参数
Object[] args = joinPoint.getArgs();
public Object[] getArgs() {
    if (this.args == null) {
        this.args = this.methodInvocation.getArguments().clone();
    }
    return this.args;
}
protected ReflectiveMethodInvocation(
        Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments,
        @Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

    this.proxy = proxy;
    this.target = target;
    this.targetClass = targetClass;
    this.method = BridgeMethodResolver.findBridgedMethod(method);
    //将方法的参数保存到arguments属性
    this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments);
    this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
}
//生成Object[] arguments保存传给方法的参数
static Object[] adaptArgumentsIfNecessary(Method method, @Nullable Object[] arguments) {
    if (ObjectUtils.isEmpty(arguments)) {
        return new Object[0];
    }
    if (method.isVarArgs()) {
        if (method.getParameterCount() == arguments.length) {
            Class<?>[] paramTypes = method.getParameterTypes();
            int varargIndex = paramTypes.length - 1;
            Class<?> varargType = paramTypes[varargIndex];
            if (varargType.isArray()) {
                Object varargArray = arguments[varargIndex];
                if (varargArray instanceof Object[] && !varargType.isInstance(varargArray)) {
                    Object[] newArguments = new Object[arguments.length];
                    System.arraycopy(arguments, 0, newArguments, 0, varargIndex);
                    Class<?> targetElementType = varargType.getComponentType();
                    int varargLength = Array.getLength(varargArray);
                    Object newVarargArray = Array.newInstance(targetElementType, varargLength);
                    System.arraycopy(varargArray, 0, newVarargArray, 0, varargLength);
                    newArguments[varargIndex] = newVarargArray;
                    return newArguments;
                }
            }
        }
    }
    return arguments;
}
```