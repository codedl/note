springboot在启动可以通过自动装配导入相关配置类组件，然后进一步实现相关功能，因此可以整合多数框架，现在我们来分析下springboot自动装配的原理。可以看到对主类使用的`@SpringBootApplication`上存在`@EnableAutoConfiguration`，而这个注解导入AutoConfigurationImportSelector组件，因此接下来需要分析AutoConfigurationImportSelector组件。
```java
@EnableAutoConfiguration
public @interface SpringBootApplication {}

@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {}
```
我们都知道spring在启动时会先获取Bean的定义再对Bean进行实例化，而扫描类路径创建Bean定义是由`PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors()`方法实现的，此时会调用`BeanDefinitionRegistryPostProcessor`的postProcessBeanDefinitionRegistry()方法。创建spring应用上下文时会对`AnnotationConfigServletWebServerApplicationContext`类实例化，进而调用`AnnotationConfigUtils#registerAnnotationConfigProcessors`向容器注册ConfigurationClassPostProcessor的Bean定义。
```java
		if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
			RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
			def.setSource(source);
			beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
		}
```
ConfigurationClassPostProcessor实现了postProcessBeanDefinitionRegistry方法，此时会被调用。中间经过一堆方法调用，最终到`AutoConfigurationImportSelector$AutoConfigurationGroup#process()`，此时会调用当前类的getAutoConfigurationEntry()，这时就会到spring.factories中读取EnableAutoConfiguration配置的自动装配类了，其实此时已经不是从spring.factories文件中获取了，spring在构造SpringApplication时会创建初始化器，此时会调用getSpringFactoriesInstances()向setInitializers()传参，就会去spring.factories读取配置类并缓存下来，这样可以减少文件的读写次数。
```java
// AutoConfigurationImportSelector.java
	protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {
		if (!isEnabled(annotationMetadata)) {
			return EMPTY_ENTRY;
		}
		AnnotationAttributes attributes = getAttributes(annotationMetadata);
        //从spring.factories文件中获取
		List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
		configurations = removeDuplicates(configurations);
		Set<String> exclusions = getExclusions(annotationMetadata, attributes);
		checkExcludedClasses(configurations, exclusions);
		configurations.removeAll(exclusions);
        //根据OnBeanCondition、OnClassCondition、OnWebApplicationCondition过滤组件
		configurations = getConfigurationClassFilter().filter(configurations);
		fireAutoConfigurationImportEvents(configurations, exclusions);
		return new AutoConfigurationEntry(configurations, exclusions);
	}
    protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {
		List<String> configurations = SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(),
				getBeanClassLoader());
		Assert.notEmpty(configurations, "No auto configuration classes found in META-INF/spring.factories. If you "
				+ "are using a custom packaging, make sure that file is correct.");
		return configurations;
	}
```
**总结下，springboot的自动装配其实就是通过触发钩子函数，到spring.factories中读取配置项的值获取自动配置类。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**