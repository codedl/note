可以对主类使用@MapperScan注解来指定扫描@Mapper接口的类路径，在@MapperScan注解通过@Import(MapperScannerRegistrar.class)导入了MapperScannerRegistrar组件。这个类实现了ImportBeanDefinitionRegistrar接口的registerBeanDefinitions方法，在方法中会扫描使用了@Mapper注解的接口，创建代理对象。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)
public @interface MapperScan {

  String[] value() default {};

  String[] basePackages() default {};

  Class<?>[] basePackageClasses() default {};

  Class<? extends BeanNameGenerator> nameGenerator() default BeanNameGenerator.class;

  Class<? extends Annotation> annotationClass() default Annotation.class;

  Class<?> markerInterface() default Class.class;

  String sqlSessionTemplateRef() default "";

  String sqlSessionFactoryRef() default "";

  Class<? extends MapperFactoryBean> factoryBean() default MapperFactoryBean.class;

}
```
在registerBeanDefinitions中先获取@MapperScan注解的元数据，定义扫描的规则，设置扫描接口的包路径。
```java
  public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    AnnotationAttributes annoAttrs = AnnotationAttributes.fromMap(importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName()));
    ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

    // this check is needed in Spring 3.1
    if (resourceLoader != null) {
      scanner.setResourceLoader(resourceLoader);
    }

    Class<? extends Annotation> annotationClass = annoAttrs.getClass("annotationClass");
    if (!Annotation.class.equals(annotationClass)) {
      scanner.setAnnotationClass(annotationClass);
    }

    Class<?> markerInterface = annoAttrs.getClass("markerInterface");
    if (!Class.class.equals(markerInterface)) {
      scanner.setMarkerInterface(markerInterface);
    }

    Class<? extends BeanNameGenerator> generatorClass = annoAttrs.getClass("nameGenerator");
    if (!BeanNameGenerator.class.equals(generatorClass)) {
      scanner.setBeanNameGenerator(BeanUtils.instantiateClass(generatorClass));
    }

    Class<? extends MapperFactoryBean> mapperFactoryBeanClass = annoAttrs.getClass("factoryBean");
    if (!MapperFactoryBean.class.equals(mapperFactoryBeanClass)) {
      scanner.setMapperFactoryBean(BeanUtils.instantiateClass(mapperFactoryBeanClass));
    }

    scanner.setSqlSessionTemplateBeanName(annoAttrs.getString("sqlSessionTemplateRef"));
    scanner.setSqlSessionFactoryBeanName(annoAttrs.getString("sqlSessionFactoryRef"));

    List<String> basePackages = new ArrayList<String>();
    for (String pkg : annoAttrs.getStringArray("value")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (String pkg : annoAttrs.getStringArray("basePackages")) {
      if (StringUtils.hasText(pkg)) {
        basePackages.add(pkg);
      }
    }
    for (Class<?> clazz : annoAttrs.getClassArray("basePackageClasses")) {
      basePackages.add(ClassUtils.getPackageName(clazz));
    }
    scanner.registerFilters();
    scanner.doScan(StringUtils.toStringArray(basePackages));
  }
```
为扫描器设置过滤器，可以看到在不声明的情况下会扫描类路径下所有接口，如果通过属性annotationClass声明指定的注解，则会扫描使用了指定的注解的接口，注意声明注解时要加`@Retention(RUNTIME)`，否则运行时没法获取到。
```java
  public void registerFilters() {
    boolean acceptAllInterfaces = true;

    // if specified, use the given annotation and / or marker interface
    if (this.annotationClass != null) {
      addIncludeFilter(new AnnotationTypeFilter(this.annotationClass));
      acceptAllInterfaces = false;
    }

    // override AssignableTypeFilter to ignore matches on the actual marker interface
    if (this.markerInterface != null) {
      addIncludeFilter(new AssignableTypeFilter(this.markerInterface) {
        @Override
        protected boolean matchClassName(String className) {
          return false;
        }
      });
      acceptAllInterfaces = false;
    }
    //默认返回true，即扫描路径下所有接口
    if (acceptAllInterfaces) {
      // default include filter that accepts all classes
      addIncludeFilter(new TypeFilter() {
        @Override
        public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
          return true;
        }
      });
    }

    // exclude package-info.java
    addExcludeFilter(new TypeFilter() {
      @Override
      public boolean match(MetadataReader metadataReader, MetadataReaderFactory metadataReaderFactory) throws IOException {
        String className = metadataReader.getClassMetadata().getClassName();
        return className.endsWith("package-info");
      }
    });
  }
```
然后在ClassPathMapperScanner.java#processBeanDefinitions()方法中设置代理类构造函数的参数为接口Class`definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName());`，设置Bean的Class为MapperFactoryBean`definition.setBeanClass(this.mapperFactoryBean.getClass());`，设置按照类型自动注入`definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);`。MapperFactoryBean类实现了FactoryBean接口，因此对Bean实例化的时候会调用getObject()创建Bean，由于指定了构造函数的参数为接口的Class，所以会使用以Class为参数的构造函数，将接口Class保存下来，之后在注入到其他Bean的属性时调用getObjectType()返回接口Class来匹配类型。
```java
  public MapperFactoryBean(Class<T> mapperInterface) {
    this.mapperInterface = mapperInterface;
  }
  public Class<T> getObjectType() {
    return this.mapperInterface;
  }
```
接下来看getObject()怎么创建代理对象的。
```java
  public T getObject() throws Exception {
    return getSqlSession().getMapper(this.mapperInterface);
  }
```
首先获取SqlSession，SqlSession在MapperFactoryBean中按类型注入的，spring会到容器中寻找SqlSession类型的Bean，即sqlSessionTemplate。
```java
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
    ExecutorType executorType = this.properties.getExecutorType();
    if (executorType != null) {
      return new SqlSessionTemplate(sqlSessionFactory, executorType);
    } else {
      return new SqlSessionTemplate(sqlSessionFactory);
    }
  }
```
接下来会调用SqlSessionTemplate#getMapper()方法，先获取mybaits的Configuration单例对象，再调用getMapper()创建代理对象。
```java
  public <T> T getMapper(Class<T> type) {
    return getConfiguration().getMapper(type, this);
  }
```
Configuration#getMapper()中会根据Bean的类型从Map中获取MapperProxyFactory对象，MapperProxyFactory封装了代理的接口类型Class。生成代理对象时就是先创建实现了InvocationHandler接口的MapperProxy对象，再通过jdk动态代理生成接口的代理对象了。
```java
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }

  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```
**总结一下，spring会到@MapperScan注解指定的路径扫描使用@Mapper注解的接口，为接口生成GenericBeanDefinition，不过得设置Class为MapperFactoryBean。注入的时候根据getObjectType()匹配接口的类型，类型匹配通过getObject()生成接口的代理对象。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**