---
title: MVC4-处理请求参数
tags:
notebook: spring源码解析
---
# 处理请求参数  
`DispatcherServlet`收到请求后会调用`InvocableHandlerMethod`类中的[`getMethodArgumentValues()`](#getdefaultargumentresolvers)处理请求参数
1. 获取方法上声明的参数数组，定义个`Object[] args`用来对处理的参数封装，对参数数组进行遍历依次处理每个参数
2. 判断当前参数是否支持解析，即是否有参数解析器能够解析当前参数，并获取当前参数的解析器
   1. 先从缓存中获取参数解析器，没有则进行解析
   2. 从当前所有的参数解析器中获取合适的解析器
      1. spring会维护一个ArrayList来保存参数解析器`private final List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>();`
      2. 在创建RequestMappingHandlerAdapter组件时，会调用实现的InitializingBean接口的`afterPropertiesSet()`方法，向argumentResolvers中添加参数解析器
      3. 对ArrayList进行遍历，调用每个参数解析器的`supportsParameter()`方法判断是否支持解析当前的参数，`supportsParameter()`中会判断是否使用了相应的注解
      4. 将找到的参数和对应的参数解析器缓存到ConcurrentHashMap`private final Map<MethodParameter, HandlerMethodArgumentResolver> argumentResolverCache = new ConcurrentHashMap<>(256);`
3. 使用找到的参数解析器对参数进行解析，从请求中获取对应的参数
# 普通pojo类型参数
1. 使用`ServletModelAttributeMethodProcessor`中的`supportsParameter()`判断是否能对当前参数进行解析
    1. 参数上不需要有任何注解
    2. 根据参数类型判断是否为无法解析的基本类型
       1. 获取参数类型
       2. 不能进行处理的参数类型有：Enum、CharSequence、Number、Date、URI、URL、Locale、Class
2. 调用`resolveArgument()`方法对参数进行解析
   1. 从请求中根据参数名获取属性的值，如果能获取就转换成参数类型的对象
   2. 反射调用参数的构造函数创建对象
      1. 创建web数据绑定器，设置转换服务`conversionService`
      2. 对请求中包含的所有属性进行遍历
         1. 获取每个请求属性名，通过请求属性名获取请求属性值，现在要将此请求属性值赋给参数对象中的同名属性
         2. 获取参数对象中目标属性的类型，以及请求中属性类型，进行转换
            1. 先调用`canConvert()`方法判断是否可以进行转换
               1. 通过请求属性类型和对象属性类型创建key，根据key到缓存中获取对应的转换器，没有获取到则进行获取
               2. 到LinkedHashMap集合中获取转换器`private final Map<ConvertiblePair, ConvertersForPair> converters = new LinkedHashMap<>(36);`
               3. 将获取的转换器缓存下来
            2. 转换器最终通过`Integer.valueOf(trimmed)`将字符串转成Integer类型
         3. 获取到对象属性值后，获取到对象属性的set方法，以发射方式进行赋值
# 参数类型转换器 
## [添加自定义转换器](#添加转换器)
1. 在`WebMvcAutoConfiguration`类中对mvcConversionService()使用了@Bean注解，spring在启动时会调用这个方法创建WebConversionService对象。而WebMvcAutoConfiguration类是GenericConversionService类的子类，因此会先对GenericConversionService父类进行实例化，此时会创建Converters实例对象赋给属性converters。
2. 在mvcConversionService()中会调用父类DelegatingWebMvcConfiguration中的addFormatters(FormatterRegistry registry) 添加转换器，参数为mvcConversionService()创建的WebMvcAutoConfiguration对象，因为WebConversionService实现了FormatterRegistry接口所以可以转成addFormatters()的参数
   1. addFormatters()内部会调用configurers的addFormatters()方法
   2. configurers为DelegatingWebMvcConfiguration类中的对象属性，其set方法使用了@Autowired注解，因此会到容器中找到实现了WebMvcConfigurer的bean集合，添加到configurers中。
   3. 对添加进来的WebMvcConfigurer的实现类对象进行遍历，依次调用每个对象的addFormatters()方法。如果我们定义了实现了WebMvcConfigurer接口的bean对象，并重写了addFormatters()方法就会被调用
      1. `public void addFormatters(FormatterRegistry registry)`方法的参数为FormatterRegistry接口，WebConversionService类的父类GenericConversionService实现了FormatterRegistry接口继承的父接口ConverterRegistry中的addConverter()方法。因此调用的是GenericConversionService类中实现的addConverter()方法
      2. 在addConverter()方法中会调用对象属性converters中的add()方法
         1. add()方法中会到内部维护的converters`private final Map<ConvertiblePair, ConvertersForPair> converters = new LinkedHashMap<>(36);`中获取相应的value，value类型为ConvertersForPair对象。如果没获取到新建一个用于添加
         2. 调用ConvertersForPair对象中的add()方法将转换器添加到内部的LinkedList中
## 获取转换器
1. 调用GenericConversionService对象中的getConverter()
2. 在getConverter()方法中调用对象属性converters中的find()方法获取转换器`private final Converters converters = new Converters();`
3. 根据请求属性类型和对象属性类型创建key，通过key到Converters类内部的LinkedHashMap中获取转换器。`private final Map<ConvertiblePair, ConvertersForPair> converters = new LinkedHashMap<>(36);`。在converters中，key为ConvertiblePair类型，由请求属性类型和对象属性类型构造；value是ConvertersForPair类型，内部维护了一个转换器数组`private final LinkedList<GenericConverter> converters = new LinkedList<>();`，保存转换器。
# @PathVariable
  + 注解的value属性不为空，则获取某个具体的路径变量
    1. 方法参数必须使用@PathVariable注解
    2. 参数类型为Map：方法参数必须使用@PathVariable注解，且@PathVariable注解的value属性不为空
  + 注解的value属性为空 
    1. 参数类型为Map类型；注解的value属性为空
    2. 解析参数：从请求中获取`org.springframework.web.servlet.HandlerMapping.uriTemplateVariables`属性的值

# 自定义参数解析器
收到请求时，spring会调用HandlerMethodArgumentResolverComposite类中的getArgumentResolver()获取参数解析器，先对argumentResolvers`private final List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>();`集合进行遍历，调用集合中每个解析器的supportsParameter方法判断是否支持当前参数的解析  
在EnableWebMvcConfiguration类中对requestMappingHandlerAdapter()使用@Bean注解，spring在启动时会调用这个方法创建`RequestMappingHandlerAdapter`对象，中间会为bean对象设置参数解析器。
   1. 获取自定义的参数解析器
      1. 先创建ArrayList集合用于保存参数解析器
      2. 调用addArgumentResolvers()获取已有的参数解析器添加到集合
      3. 在WebMvcConfigurationSupport类中定义了addArgumentResolvers()的空方法，其子类DelegatingWebMvcConfiguration.java重写了addArgumentResolvers()。
      4. EnableWebMvcConfiguration类为DelegatingWebMvcConfiguration的子类，由动态分派的方式决定要调用addArgumentResolvers()时，会从子类开始向上搜索addArgumentResolvers()的实现，因此调用这个方法优先级是EnableWebMvcConfiguration>DelegatingWebMvcConfiguration>WebMvcConfigurationSupport，所以会优先调用DelegatingWebMvcConfiguration类中实现的addArgumentResolvers()方法，即模板模式。
      5. 此时会调用所有WebMvcConfigurer接口的实现类对象中的addArgumentResolvers()方法添加参数解析器，我们实现的对象中的方法也会被调用
   2. 最终将参数解析器返回给RequestMappingHandlerAdapter适配器的setCustomArgumentResolvers()，通过这个方法将参数解析器传给customArgumentResolvers属性。
   3. RequestMappingHandlerAdapter实现了InitializingBean接口，因此实例化后会调用afterPropertiesSet()方法进行后置处理
      1. 先添加spring应用定义的参数解析器，再获取自定义的参数解析器，即customArgumentResolvers
      2. 将适配器中所有的参数解析器保存到HandlerMethodArgumentResolverComposite对象中的argumentResolvers`private final List<HandlerMethodArgumentResolver> argumentResolvers = new ArrayList<>();`。
   4. 之后进行参数解析时会对argumentResolvers进行遍历获取能够解析当前参数的解析器。



--------------------
# getDefaultArgumentResolvers
```
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>(30);

    // Annotation-based argument resolution
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    resolvers.add(new PathVariableMethodArgumentResolver());
    resolvers.add(new PathVariableMapMethodArgumentResolver());
    resolvers.add(new MatrixVariableMethodArgumentResolver());
    resolvers.add(new MatrixVariableMapMethodArgumentResolver());
    resolvers.add(new ServletModelAttributeMethodProcessor(false));
    resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new RequestHeaderMapMethodArgumentResolver());
    resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
    resolvers.add(new SessionAttributeMethodArgumentResolver());
    resolvers.add(new RequestAttributeMethodArgumentResolver());

    // Type-based argument resolution
    resolvers.add(new ServletRequestMethodArgumentResolver());
    resolvers.add(new ServletResponseMethodArgumentResolver());
    resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    resolvers.add(new RedirectAttributesMethodArgumentResolver());
    resolvers.add(new ModelMethodProcessor());
    resolvers.add(new MapMethodProcessor());
    resolvers.add(new ErrorsMethodArgumentResolver());
    resolvers.add(new SessionStatusMethodArgumentResolver());
    resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

    // Custom arguments
    if (getCustomArgumentResolvers() != null) {
        resolvers.addAll(getCustomArgumentResolvers());
    }

    // Catch-all
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
    resolvers.add(new ServletModelAttributeMethodProcessor(true));

    return resolvers;
}
public void afterPropertiesSet() {
......
    if (this.argumentResolvers == null) {
        List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
        this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
    }
......
}
```
# 添加转换器
```
public FormattingConversionService mvcConversionService() {
    Format format = this.mvcProperties.getFormat();
    WebConversionService conversionService = new WebConversionService(new DateTimeFormatters()
            .dateFormat(format.getDate()).timeFormat(format.getTime()).dateTimeFormat(format.getDateTime()));
    addFormatters(conversionService);
    return conversionService;
}
//WebConversionService实现了FormatterRegistry接口，可以转成FormatterRegistry调用addFormatters
//在DelegatingWebMvcConfiguration类中定义了configurers属性，对属性的set方法使用了@Autowired注解
//在创建bean时，会从容器找到WebMvcConfigurer接口的实现类对象，注入到configurers
//在addFormatters方法中会调用这些对象实现的addFormatters方法
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
    protected void addFormatters(FormatterRegistry registry) {
        this.configurers.addFormatters(registry);
    }
    private final WebMvcConfigurerComposite configurers = new WebMvcConfigurerComposite();
    @Autowired(required = false)
    public void setConfigurers(List<WebMvcConfigurer> configurers) {
        if (!CollectionUtils.isEmpty(configurers)) {
            this.configurers.addWebMvcConfigurers(configurers);
        }
    }
}
//我们自定义的WebConfig类实现了WebMvcConfigurer接口，因此会被发现添加到configurers中
//在构建WebConversionService对象时会调用我们重写的addFormatters方法
//在addFormatters方法中我们会调用registry接口的addConverter方法
//registry是由WebConversionService转的
//WebConversionService类的父类GenericConversionService实现了addConverter方法
//因此会调用GenericConversionService类中的addConverter方法
public class WebConfig  implements WebMvcConfigurer {
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new Converter<String, User.Pet>() {
            @Override
            public User.Pet convert(String source) {
                User.Pet pet = new User.Pet();
                pet.setName(source.split(",")[0]);
                pet.setColor(source.split(",")[1]);
                return pet;
            }
        });
    }
}
//最终转到GenericConversionService类中的addConverter添加转换器
public void addConverter(GenericConverter converter) {
    this.converters.add(converter);
    invalidateCache();
}
//此时会调用类中的对象属性converters中的add方法
/根据key获取converters中的ConvertersForPair对象，如果不存在new一个
//然后convertersForPair.add(converter);添加到key对应的value中
//最终添加到ConvertersForPair对象中的converters指向的LinkedList中。
private static class Converters {
    private final Map<ConvertiblePair, ConvertersForPair> converters = new LinkedHashMap<>(36);
    public void add(GenericConverter converter) {
        Set<ConvertiblePair> convertibleTypes = converter.getConvertibleTypes();
        if (convertibleTypes == null) {
            Assert.state(converter instanceof ConditionalConverter,
                    "Only conditional converters may return null convertible types");
            this.globalConverters.add(converter);
        }
        else {
            for (ConvertiblePair convertiblePair : convertibleTypes) {
                ConvertersForPair convertersForPair = getMatchableConverters(convertiblePair);
                //添加转换器
                convertersForPair.add(converter);
            }
        }
    }
    private ConvertersForPair getMatchableConverters(ConvertiblePair convertiblePair) {
        return this.converters.computeIfAbsent(convertiblePair, k -> new ConvertersForPair());
    }
}
private static class ConvertersForPair {
    private final LinkedList<GenericConverter> converters = new LinkedList<>();
    public void add(GenericConverter converter) {
        this.converters.addFirst(converter);
    }
}
```

# 消息转换器
1. 在`WebMvcConfigurationSupport`类中定义了静态变量jackson2XmlPresent`private static final boolean jackson2XmlPresent;`，在加载WebMvcConfigurationSupport类时会在静态代码块中加载`com.fasterxml.jackson.dataformat.xml.XmlMapper`类，返回加载的结果，如果为true表明当前应用存在这个类
2. 在对`RequestMappingHandlerAdapter`进行实例化时，会设置获取到的消息转换器。先将应用中预先设置的消息转换器添加进来，然后判断消息转换器的类是否存在，如果存在则进行实例化，生成消息转换器的实例对象保存下来，并写到messageConverters`private List<HttpMessageConverter<?>> messageConverters;`属性中
```java
if (jackson2XmlPresent) {
    Jackson2ObjectMapperBuilder builder = Jackson2ObjectMapperBuilder.xml();
    if (this.applicationContext != null) {
        builder.applicationContext(this.applicationContext);
    }
    messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
}
```
3. 完成RequestMappingHandlerAdapter的实例化后，会执行afterPropertiesSet()方法进行初始化，设置适配器的返回值处理器，此时会将消息转换器与返回值处理器绑定。即会对``RequestResponseBodyMethodProcessor``类进行实例化，AbstractMessageConverterMethodArgumentResolver类是RequestResponseBodyMethodProcessor类的父类，也会被初始化，将获取到的消息转换器作为构造函数的参数保存到messageConverters属性中。
4. 在对返回值进行处理时，获取返回值处理器后，找到绑定的消息转换器，使用转换器对返回值类型进行转换。
# 内容协商
1. 在`WebMvcAutoConfiguration`中对mvcContentNegotiationManager()使用了@Bean注解，spring在启动时会调用这个方法创建bean，最终会调用ContentNegotiationManagerFactoryBean类中的build()方法生成内容协商策略。
   1. 对`WebMvcProperties`类使用了`@ConfigurationProperties(prefix = "spring.mvc")`注解，可以将配置文件中以"spring.mvc"开头的配置和属性绑定，最终会将`spring.mvc.contentnegotiation.favor-parameter`配置项，和WebMvcProperties类中的内部类Contentnegotiation中的`favorParameter`绑定。
   2. 在生成内容协商管理器的实例时，会调用mvcContentNegotiationManager()中的configureContentNegotiation()方法配置内容协商管理器，此时会将`spring.mvc.contentnegotiation.favor-parameter`配置项的值写到属性`favorParameter`中。
2. 根据favorParameter的值判断是否可以由参数进行内容协商，如果为true则添加参数内容协商策略ParameterContentNegotiationStrategy
```
if (this.favorParameter) {
    ParameterContentNegotiationStrategy strategy = new ParameterContentNegotiationStrategy(this.mediaTypes);
    strategy.setParameterName(this.parameterName);
    if (this.useRegisteredExtensionsOnly != null) {
        strategy.setUseRegisteredExtensionsOnly(this.useRegisteredExtensionsOnly);
    }
    else {
        strategy.setUseRegisteredExtensionsOnly(true);  // backwards compatibility
    }
    strategies.add(strategy);
}
```
3. 在对适配器进行实例化时将创建的内容协商管理器注入进来
4. 在适配器的afterPropertiesSet()方法中将内容协商器与返回值处理器绑定
```
//AbstractMessageConverterMethodProcessor为RequestResponseBodyMethodProcessor的父类
handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),
        this.contentNegotiationManager, this.requestResponseBodyAdvice));
protected AbstractMessageConverterMethodProcessor(List<HttpMessageConverter<?>> converters,
        @Nullable ContentNegotiationManager manager, @Nullable List<Object> requestResponseBodyAdvice) {

    super(converters, requestResponseBodyAdvice);

    this.contentNegotiationManager = (manager != null ? manager : new ContentNegotiationManager());
    this.safeExtensions.addAll(this.contentNegotiationManager.getAllFileExtensions());
    this.safeExtensions.addAll(SAFE_EXTENSIONS);
}        
```
5. 协商返回给浏览器的内容类型时，会调用resolveMediaTypes()实现，可以看到通过从请求中获取参数"format"的值决定内容类型：xml -> {MediaType@7466} "application/xml"；json -> {MediaType@7468} "application/json"
```
public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
    for (ContentNegotiationStrategy strategy : this.strategies) {
        List<MediaType> mediaTypes = strategy.resolveMediaTypes(request);
        if (mediaTypes.equals(MEDIA_TYPE_ALL_LIST)) {
            continue;
        }
        return mediaTypes;
    }
    return MEDIA_TYPE_ALL_LIST;
}
public List<MediaType> resolveMediaTypes(NativeWebRequest webRequest)
        throws HttpMediaTypeNotAcceptableException {

    return resolveMediaTypeKey(webRequest, getMediaTypeKey(webRequest));
}
protected String getMediaTypeKey(NativeWebRequest request) {
    return request.getParameter(getParameterName());
}
private String parameterName = "format";
xml -> {MediaType@7466} "application/xml"
json -> {MediaType@7468} "application/json"
```
   