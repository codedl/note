---
title: MVC3-获取静态资源
tags:
notebook: spring源码解析
---
1. 在springboot的spring.factories中通过`org.springframework.boot.autoconfigure.EnableAutoConfiguration`配置项配置了要自动加载的配置类
      `WebMvcAutoConfiguration`，在`WebMvcAutoConfiguration`类的内部对`EnableWebMvcConfiguration`类使用了`@Configuration`注解
2. `WebMvcConfigurationSupport`类是EnableWebMvcConfiguration的父类，spring在对使用了@Configuration注解的类进行解析时，也会对父类进行解析，
因此也会对WebMvcConfigurationSupport进行解析，解析时会提取使用了@Bean的方法，生成bean的定义。 在`WebMvcConfigurationSupport`类中对resourceHandlerMapping()方法使用@Bean注解，
在创建bean时会调用`resourceHandlerMapping()`方法。
   1. 对`ResourceHandlerRegistry`类进行实例化生成注册器，注入ApplicationContext、ServletContext、ContentNegotiationManager、UrlPathHelper
   2. 调用`WebMvcAutoConfiguration`中的`addResourceHandlers()`方法[添加资源处理器](#addresourcehandlers)
      1. 通过配置项`spring.resources.add-mappings`判断是否添加静态资源映射
      2. 通过配置项`spring.resources.cache.period`获取静态资源的缓存时间
      3. 通过配置项`spring.mvc.static-path-pattern`获取静态资源的请求路径
      4. 使用静态资源的请求路径创建一个ResourceHandlerRegistration资源处理注册器的实例
      5. 将ResourceHandlerRegistration资源处理注册器的实例保存到ArrayList`private final List<ResourceHandlerRegistration> registrations = new ArrayList<>();`
      6. 通过配置项`spring.resources.static-locations`获取静态资源在服务器上的保存位置,将这些位置保存ArrayList`private final List<String> locationValues = new ArrayList<>();`
      7. 设置静态缓存时间
   3. 获取处理器映射
      1. 生成http资源请求处理器`ResourceHttpRequestHandler`，设置静态资源的存储路径和缓存时间
      2. 设置路径解析器`UrlPathHelper`，内容协商器`ContentNegotiationManager`，通过`afterPropertiesSet()`方法作进一步的设置
      3. 将请求路径和http请求处理器的映射关系缓存到LinkedHashMap`Map<String, HttpRequestHandler> urlMap = new LinkedHashMap<>();`
      4. 通过urlMap生成处理器映射的实例对象`SimpleUrlHandlerMapping`，此时会将局部变量urlMap保存到属性urlMap中`this.urlMap.putAll(urlMap);`
   4. 配置路径匹配器PathMatcher、路径解析器UrlPathHelper、拦截器链Interceptors、CorsConfigurations
3. 处理获取静态资源的请求
   1. 将请求地址url和相应得处理器绑定
      1. spring是调用`refresh()`方法刷新容器时，会调用`prepareBeanFactory()`向容器注册一个后置处理器
`beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));`
      2. 创建资源处理器映射时，这个后置处理器起作用，处理器中得`postProcessBeforeInitialization()`会被调用
      3. 静态资源处理器`SimpleUrlHandlerMapping`实现了`ApplicationContextAware`接口，会通过应用上下文进行初始化
      4. spring会使用LinkedHashMap`private final Map<String, Object> urlMap = new LinkedHashMap<>();`来缓存请求地址url和处理器得映射关系
      5. 这里spring会调用`registerHandlers()`方法注册url和处理器到urlMap
         1. 如果url不是以"/"开头，就在url前加上"/"
         2. 处理器handler得名字字符串中得空格去掉
         3. 判断当前url得处理器是否已经存在，有就抛出异常
         4. 如果`urlPath.equals("/")`，则设置根处理器；如果`urlPath.equals("/*")`，则设置默认处理器，否则缓存urlPath和resolvedHandler得映射关系
   2. 在处理http请求时先到容器中[获取所有得处理器映射](#inithandlermappings)
      1. 到容器中找到实现了HandlerMapping接口得bean，即处理器映射，将bean缓存到ArrayList`this.handlerMappings = new ArrayList<>(matchingBeans.values());`
      2. 按照优先级对处理器映射进行排序
   3. 调用`DispatcherServlet`类中得`doDispatch()`处理请求
      1. 根据要处理得请求获取映射得处理器。`SimpleUrlHandlerMapping`为`AbstractUrlHandlerMapping`的子类，获取处理器映射时是遍历父类中的handlerMap属性， 
而子类会把处理器映射缓存到父类属性。这样通过父类对象变量调用`getHandler()`方法时，会根据动态分派调用实际子类的方法。
         1. 对所有得处理器映射进行遍历，依次调用每个处理器映射得`getHandler()`方法尝试获取能处理当前请求得处理器
         2. 对请求路径进行解析获取请求url，将请求url作为请求得属性保存下来
         3. 根据请求url到handlerMap直接获取处理器，如果能获取到则直接返回
         4. 否则对handlerMap中得key即每个请求url进行遍历，根据路径匹配器判断handlerMap中保存得请求url跟当前请求得url是否匹配，将匹配得url缓存到ArrayList
         5. 对缓存url得ArrayList进行排序，如果有多个则默认使用第一个
         6. 根据url到handlerMap获取映射得处理器
         7. 根据请求url和映射得处理器生成处理器执行器链HandlerExecutionChain
      2. 如果没有获取到当前请求url映射得处理器，则使用默认处理器
      3. 如果hander实现String，则根据handlerName到容器中获取bean，将bean当作处理器
      4. 为当前得处理器执行器链添加拦截器链
      5. 配置CorsConfiguration
   4. 根据处理器实现得接口获取处理器适配器
   5. 获取请求方式，根据时间戳进行校验，判断是否要处理当前请求
      1. 获取`If-Unmodified-Since`的值，如果为-1说明请求中没有设置此值，不需要处理。
      2. 将`If-Unmodified-Since`的值跟资源的最近修改时间戳进行比较，判断资源的修改时间是否在这个时间之后
      3. 如果资源的最近修改时间戳在`If-Unmodified-Since`字段之后，则设置响应码`PRECONDITION_FAILED(412, "Precondition Failed")`，直接返回，服务器不会对请求作进一步的处理
   6. 执行拦截器链中得每个拦截器方法
   7. 调用`handleRequest(HttpServletRequest request, HttpServletResponse response)`处理请求，对请求进行响应
      1. 获取请求得资源，即请求路径下得资源，如"/2.png"=>"2.png"
      2. 遍历每个静态资源得存储路径，结合请求获取得资源，生成Resource对象
         1. 根据静态资源的存储路径和要获取的静态资源名，拼接成完整的url
         2. 判断url路径是否为文件，如果是文件的话，只要文件可读且不是文件夹则返回
         3. 如果url是一个网络连接，则通过url获取要请求的资源
      3. 浏览器会缓存静态资源，因此服务器需要判断是否需要返回新的静态资源，即浏览器是否使用本地缓存的静态资源
         1. 对请求头`If-Unmodified-Since`进行校验
            1. 获取`If-Unmodified-Since`的值，如果为-1说明请求中没有设置此值，不需要处理。
            2. 将`If-Unmodified-Since`的值跟资源的最近修改时间戳进行比较，判断资源的修改时间是否在这个时间之后
            3. 如果资源的最近修改时间戳在`If-Unmodified-Since`字段之后，则设置响应码`PRECONDITION_FAILED(412, "Precondition Failed")`
            4. 否则返回`NOT_MODIFIED(304, "Not Modified")`
         2. 对请求头`If-Modified-Since`进行校验
            1. 获取`If-Modified-Since`的值，如果为-1说明请求中没有设置此值，不需要处理
            2. 将`If-Modified-Since`的值跟资源的最近修改时间戳进行比较，判断资源的修改时间是否在这个时间之前。如果在之前，则说明浏览器拿到的是最新的资源，
设置响应状态码为`NOT_MODIFIED(304, "Not Modified")`；否则设置响应状态码为`PRECONDITION_FAILED(412, "Precondition Failed")`
         3. 返回响应状态码`OK(200, "OK")`，返回响应头`Last-Modified`表示当前静态资源最近修改的时间戳
         + 浏览器获取到静态资源后，会将静态资源缓存在本地，同时为静态资源设置最近修改的时间戳，值为服务器返回的`NOT_MODIFIED(304, "Not Modified")`的值。之后浏览器向服务器发送请求时，
会在请求头加上`If-Modified-Since`字段表示静态资源的最近修改时间戳，服务器需要校验浏览器要获取的静态资源是否发生改动。如果有改动，需要进一步根据请求方式进行判断。如果是isHttpGetOrHead则返回
`PRECONDITION_FAILED(412, "Precondition Failed")`，其他类型的请求返回`NOT_MODIFIED(304, "Not Modified")`。之后浏览器会在请求中添加`If-Unmodified-Since`的值，服务器使用这个字
段的值校验当前静态资源是否发生改动。如果发生了改动返回`PRECONDITION_FAILED(412, "Precondition Failed")`，否则不作任何处理。
      4. 如果需要返回静态资源，则准备返回给客户端浏览器的响应数据
         1. 给当前响应设置响应头表示资源缓存时间`max-age=1002`
         2. 设置响应内容长度，设置媒体内容类型
      5. 将服务器存储的静态资源返回给客户端浏览器
4. 访问首页
   1. 在`WebMvcAutoConfiguration`类中对`welcomePageHandlerMapping()`使用@Bean注解，创建bean时会调用这个方法创建首页处理器
      1. 获取首页html文件
         1. 获取静态资源在服务器的存储路径数组，遍历每个路径
         2. 根据静态资源的路径拼接"index.html"，构成首页的完整路径，生成Resource对象
         3. 通过Resource对象判断当前的首页是否存在
      2. 如果首页存在，创建视图控制器的实例对象`ParameterizableViewController controller = new ParameterizableViewController();`
      3. 为视图控制器设置viewName为`"forward:index.html"`，设置视图控制器为首页处理器的根处理器，当请求路径为"/"时获取
      4. 为首页控制器设置拦截器链
   2. 向根路径"/"发送请求时，spring会遍历已有的处理器映射
   3. 通过调用处理器映射的`getHandler()`方法获取能够处理请求的处理器执行器链
      1. spring会通过`MediaType`类来维护数据的媒体类型，即请求和响应接受的数据类型
      2. 获取请求头`Accept`的值，判断要获取的资源的数据类型是否为`text/html`，如果是则尝试获取处理器执行器链
      3. 对请求路径进行解析获取请求路径url
      4. 到处理器映射中根据请求路径url获取映射的处理器，发现请求的url路径为"/"路径时，获取根处理器ParameterizableViewController
      5. 使用获取到的根处理器构建处理器执行器链
   4. 获取适配器SimpleControllerHandlerAdapter
   5. 创建ModelAndView，并设置视图名`forward:index.html`，返回首页index.html给浏览器


# addResourceHandlers
```
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }
    Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
    CacheControl cacheControl = this.resourceProperties.getCache().getCachecontrol().toHttpCacheControl();
    if (!registry.hasMappingForPattern("/webjars/**")) {
        customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/")
                .setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
    }
    String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    if (!registry.hasMappingForPattern(staticPathPattern)) {
        customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
                .addResourceLocations(getResourceLocations(this.resourceProperties.getStaticLocations()))
                .setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
    }
}
public ResourceHandlerRegistration addResourceHandler(String... pathPatterns) {
    ResourceHandlerRegistration registration = new ResourceHandlerRegistration(pathPatterns);
    this.registrations.add(registration);
    return registration;
}
public ResourceHandlerRegistration addResourceLocations(String... resourceLocations) {
    this.locationValues.addAll(Arrays.asList(resourceLocations));
    return this;
}
public ResourceHandlerRegistration setCachePeriod(Integer cachePeriod) {
    this.cachePeriod = cachePeriod;
    return this;
}
```
```
private void initHandlerMappings(ApplicationContext context) {
     this.handlerMappings = null;

     if (this.detectAllHandlerMappings) {
         // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
         Map<String, HandlerMapping> matchingBeans =
                 BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
         if (!matchingBeans.isEmpty()) {
             this.handlerMappings = new ArrayList<>(matchingBeans.values());
             // We keep HandlerMappings in sorted order.
             AnnotationAwareOrderComparator.sort(this.handlerMappings);
         }
     }
     .....
 }
```