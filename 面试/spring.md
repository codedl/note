+ springboot的自动装配原理  
主类上的`@SpringBootApplication`存在`@EnableAutoConfiguration`，`@EnableAutoConfiguration`会导入AutoConfigurationImportSelector组件，其`AutoConfigurationImportSelector$AutoConfigurationGroup#process()`方法会读取当前应用所有依赖jar中的自动配置类。过程是通过`SpringFactoriesLoader.loadFactoryNames()`读取spring.factories中EnableAutoConfiguration对应的配置类，然后使用三种类型的条件注解判断能否导入，分别是OnBeanCondition(判断Bean)、OnClassCondition(判断类)、OnWebApplicationCondition(判断应用类型，部分Bean只会在web中被创建)。

+ springboot容器定制的实现  
spring在创建web容器时会去获取创建容器的ServletWebServerFactory工厂，即ServletWebServerFactory接口的实现类对象，默认有TomcatServletWebServerFactory、JettyServletWebServerFactory、UndertowServletWebServerFactory，因此spring默认支持tomcat、jetty、undertow这三种web容器，具体使用哪个是由作用在ServletWebServerFactoryConfiguration的子类上的条件注解判断的，例如`@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })`，如果检测到应用中存在Servlet、Tomcat、UpgradeProtocol这三个跟tomcat相关的类时，就会使用tomcat作为web容器。

+ spring的循环依赖  
spring中循环依赖有三种可能，分别是set注入、构造器注入、@DepondOn注解声明导致的循环依赖，构造器注入、@DepondOn注解声明导致的循环依赖是无法解决的，解决set注入的过程是：spring反射调用构造函数创建bean之后，将获取bean的lambda表达式缓存下来，等bean完成了属性填充之后，将完整的bean再缓存下来。出现循环依赖时，spring先尝试获取完整的属性已经被填充的bean，没获取则获取属性没有被填充的bean，否则通过lambda表达式获取bean，此时可能会拿到代理对象，再加入二级缓存即保存没有被填充的bean。

+ @Autowired注解原理  
spring是用AutowiredAnnotationBeanPostProcessor处理@Autowired注解的，首先提取要注入的字段和注解的属性，然后根据类型到容器中找到相应的bean，如果是有多个候选bean，则优先使用@Primary标记的bean，再就是根据@Priority注解的值越大优先级越高，找到了bean之后就反射写入属性中，完成依赖注入

+ spring的设计模式  
参考[spring常见设计模式](https://blog.csdn.net/weixin_42145727/article/details/129115234)

+ Bean的初始化方法：@Bean注解的initMethod属性指定的方法、@PostConstruct注解标注的方法、实现InitializingBean接口的afterPropertiesSet()方法

+ spring容器启动的过程
    1. 根据类的加载情况判断当前应用类型，默认是servlet
    2. 从spring.factories配置文件中读取初始化器和监听器
    3. 容器启动时在各个阶段发布不同的事件，ConfigFileApplicationListener配置文件监听器会监听ApplicationEnvironmentPreparedEvent事件，此时会去读取当前应用的配置文件，添加到配置源中
    4. 容器的后置处理器发挥作用，扫描应用中的Class，处理@Configuration注解，生成Bean定义注入容器
    5. 对容器中的所有Bean进行实例化
    6. 调用实现了ApplicationRunner和CommandLineRunner接口的Bean实现的方法

+ 事务失效
   1. 构造拦截器链时会对CglibMethodInvocation实例化，此时会判断事务方法的权限，不可为非public的方法，不可为Object类中的方法
   2. 事务底层是通过动态代理实现的，需要对原方法进行重写，所有事务方法不可为final
   3. 事务声明的类必须由spring容器托管，否则不会创建代理对象；必须通过代理对象调用事务方法，才能对事务方法进行增强处理，如果调用本类中方法则会失效
   4. 方法内如果自己捕获了异常则不会发生回滚

+ dispatchservlet处理请求的过程  
根据请求的url找到能够处理请求的方法method，创建HandlerMethod用来对请求和方法进行封装，获取拦截器，生成执行器链，创建处理器适配器，对请求进行处理，生成响应。

+ @SpringBootApplication注解包含三个注解，分别是  
  1. @SpringBootConfiguration包含@Configuration注解，可以让容器解析处理
  2. @EnableAutoConfiguration是自动配置类注解，用于实现springboot的自动装配
  3. @ComponentScan用来指定类的扫描路径，默认为主类所在的包路径

+ spring注入属性时怎么进行设置

