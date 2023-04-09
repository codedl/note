---
title: MVC7-拦截器
tags:
notebook: spring源码解析
---
1. 在WebMvcConfigurationSupport类中对生成RequestMappingHandlerMapping处理器映射的requestMappingHandlerMapping()使用了@Bean注解，在创建Bean时会为Bean设置拦截器链。**获取应用中所有的拦截器再保存到处理器映射中。** 
    `mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));`
   1. 先生成拦截器注册工具InterceptorRegistry，在拦截器注册工具内部维护了一个用来存储拦截器的集合。  
    `private final List<InterceptorRegistration> registrations = new ArrayList<>();`
   2. 调用WebMvcConfigurer中的addInterceptors()添加拦截器到InterceptorRegistry的registrations集合中，我们可以通过addInterceptors()添加自定义的拦截器。
      1. 先对拦截器器进行封装，生成拦截器注册器，拦截器注册内部维护了拦截器、拦截路径、放行路径;再添加拦截路径和放行路径
    ```
	private final HandlerInterceptor interceptor;
	private final List<String> includePatterns = new ArrayList<>();
	private final List<String> excludePatterns = new ArrayList<>();
    ```
   3. 设置到处理器映射RequestMappingHandlerMapping的属性interceptors中
   4. 在bean的后置处理中postProcessBeforeInitialization()，对拦截器进行适配后添加到adaptedInterceptors中
2. spring在`DispatcherServlet`中的doDispatch()中处理请求时会执行拦截器中的方法
   1. 在获取能够处理请求的处理器执行器链时会生成拦截器数组，**本质是将从RequestMappingHandlerMapping获取拦截器链到执行器链**  
    `HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);`
   2. 对处理器适配器`RequestMappingHandlerMapping`中的拦截器集合`private final List<HandlerInterceptor> adaptedInterceptors = new ArrayList<>();`进行遍历
   3. 将获取到的拦截器添加到处理器执行器链中
    ```
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        if (interceptor instanceof MappedInterceptor) {
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        }
        else {
            chain.addInterceptor(interceptor);
        }
    }
    public void addInterceptor(HandlerInterceptor interceptor) {
        initInterceptorList().add(interceptor);
    }
    ```
    1. 处理请求之后遍历拦截器链，调用每个拦截器的preHandle()方法，使用interceptorIndex表示执行到的拦截器索引`private int interceptorIndex = -1;`
    2. 请求处理完成之后会执行每个拦截器的applyPostHandle()方法
    3. 当返回响应数据给客户端 || 拦截器执行失败 || 请求处理失败时，则从interceptorIndex开始逆序执行拦截器中的afterCompletion()方法，即执行参数处理的拦截器的方法
    
   