feign客户端的初始化是从作用在主类的@EnableFeignClients注解开始的，可以看到通过@Import导入了FeignClientsRegistrar组件。
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
```
FeignClientsRegistrar实现了ImportBeanDefinitionRegistrar接口，可以通过接口中声明的钩子函数registerBeanDefinitions()向容器注册Bean的定义。首先通过registerDefaultConfiguration注册客户配置，本质是为feign客户端创建FeignClientSpecification类型的Bean定义添加到容器中。
```java
	public void registerBeanDefinitions(AnnotationMetadata metadata,
			BeanDefinitionRegistry registry) {
		registerDefaultConfiguration(metadata, registry);
		registerFeignClients(metadata, registry);
	}
```
registerFeignClients()方法用来注册每个feign客户端，首先获取@EnableFeignClients注解的属性，根据属性clients判断作用在客户端上的注解，默认为@FeignClient；然后根据basePackages属性获取要扫描的包路径，默认为@EnableFeignClients注解类所在的包；接下来是到包在扫描使用了@FeignClient注解的接口，并为每个接口生成Bean定义。设置接口的Bean定义类型为FeignClientFactoryBean.class，同时根据注解属性为FeignClientFactoryBean对象的属性赋值。FeignClientFactoryBean是实现了FactoryBean接口的工厂Bean，可以通过getObjectType判断注入的类型，通过getObject创建要注入的对象。
```java
	private void registerFeignClient(BeanDefinitionRegistry registry,
			AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
		String className = annotationMetadata.getClassName();
		BeanDefinitionBuilder definition = BeanDefinitionBuilder
				.genericBeanDefinition(FeignClientFactoryBean.class);
		validate(attributes);
		definition.addPropertyValue("url", getUrl(attributes));
		definition.addPropertyValue("path", getPath(attributes));
		String name = getName(attributes);
		definition.addPropertyValue("name", name);
		definition.addPropertyValue("type", className);
		definition.addPropertyValue("decode404", attributes.get("decode404"));
		definition.addPropertyValue("fallback", attributes.get("fallback"));
		definition.addPropertyValue("fallbackFactory", attributes.get("fallbackFactory"));
		definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);

		String alias = name + "FeignClient";
		AbstractBeanDefinition beanDefinition = definition.getBeanDefinition();

		boolean primary = (Boolean)attributes.get("primary"); // has a default, won't be null

		beanDefinition.setPrimary(primary);

		String qualifier = getQualifier(attributes);
		if (StringUtils.hasText(qualifier)) {
			alias = qualifier;
		}

		BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition, className,
				new String[] { alias });
		BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
	}
```
通过getObject创建对象时分三步：
1. 获取FeignContext，即feign的上下文，spring会调用FeignAutoConfiguration类的Bean方法feignContext创建FeignContext
```java
	public Object getObject() throws Exception {
		FeignContext context = applicationContext.getBean(FeignContext.class);
		Feign.Builder builder = feign(context);

		if (!StringUtils.hasText(this.url)) {
			String url;
			if (!this.name.startsWith("http")) {
				url = "http://" + this.name;
			}
			else {
				url = this.name;
			}
			url += cleanPath();
			return loadBalance(builder, context, new HardCodedTarget<>(this.type,
					this.name, url));
		}
		......
	}
```
2. 创建Feign.Builder，可以看到默认是创建Feign.Builder，如果存在Hystrix限流熔断，并且配置项feign.hystrix.enabled存在，就会创建HystrixFeign.Builder
```java
// FeignClientsConfiguration.java
	@Configuration
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {
		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}
	}
    @Bean
	@Scope("prototype")
	@ConditionalOnMissingBean
	public Feign.Builder feignBuilder(Retryer retryer) {
		return Feign.builder().retryer(retryer);
	}
```
3. 根据@FeignClient的属性url的值判断是否调用微服务，默认没有值时会调用属性name的值表示的微服务，此时会用loadBalance()方法创建接口的代理对象。

loadBalance()方法就是用来创建代理对象的方法，创建代理对象也是有个过程的。
```java
	protected <T> T loadBalance(Feign.Builder builder, FeignContext context,
			HardCodedTarget<T> target) {
		Client client = getOptional(context, Client.class);
		if (client != null) {
			builder.client(client);
			Targeter targeter = get(context, Targeter.class);
			return targeter.target(this, builder, context, target);
		}

		throw new IllegalStateException(
				"No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
	}
```
1. 从容器中获取实现了Client接口的Bean，这个Bean默认是在DefaultFeignLoadBalancedConfiguration类中通过仅有的方法feignClient()创建的，为LoadBalancerFeignClient对象。
```java
@Configuration
class DefaultFeignLoadBalancedConfiguration {
	@Bean
	@ConditionalOnMissingBean
	public Client feignClient(CachingSpringLoadBalancerFactory cachingFactory,
							  SpringClientFactory clientFactory) {
		return new LoadBalancerFeignClient(new Client.Default(null, null),
				cachingFactory, clientFactory);
	}
}
```
2. 从容器中获取实现了Targeter接口的Bean，可以看到当HystrixFeign类存在时使用的是HystrixTargeter，默认是DefaultTargeter，spring集成了Hystrix，因此这里我们使用的HystrixTargeter，因此接下来就是HystrixTargeter#target()创建动态的代理对象。
```java
// FeignAutoConfiguration.java
	@Configuration
	@ConditionalOnClass(name = "feign.hystrix.HystrixFeign")
	protected static class HystrixFeignTargeterConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public Targeter feignTargeter() {
			return new HystrixTargeter();
		}
	}

	@Configuration
	@ConditionalOnMissingClass("feign.hystrix.HystrixFeign")
	protected static class DefaultFeignTargeterConfiguration {
		@Bean
		@ConditionalOnMissingBean
		public Targeter feignTargeter() {
			return new DefaultTargeter();
		}
	} 
```
在HystrixTargeter#target()中发现使用的feign为Feign.Builder，则到Feign$Builder#target()中创建代理对象。首先封装各种对象到ReflectiveFeign里，然后ReflectiveFeign#newInstance()创建代理对象。首先获取接口中的所有方法，封装成MethodHandler(SynchronousMethodHandler)，将接口中声明的方法和MethodHandler保存到methodToHandler(LinkedHashMap)，然后创建ReflectiveFeign.FeignInvocationHandler实例对象，ReflectiveFeign.FeignInvocationHandler实现了InvocationHandler接口，有了这个接口就可以Proxy.newProxyInstance生成代理对象了。
```java
  public <T> T newInstance(Target<T> target) {
    Map<String, MethodHandler> nameToHandler = targetToHandlersByName.apply(target);
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap<Method, MethodHandler>();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList<DefaultMethodHandler>();

    for (Method method : target.type().getMethods()) {
      if (method.getDeclaringClass() == Object.class) {
        continue;
      } else if(Util.isDefault(method)) {
        DefaultMethodHandler handler = new DefaultMethodHandler(method);
        defaultMethodHandlers.add(handler);
        methodToHandler.put(method, handler);
      } else {
        methodToHandler.put(method, nameToHandler.get(Feign.configKey(target.type(), method)));
      }
    }
    InvocationHandler handler = factory.create(target, methodToHandler);
    //@1-类加载器
    //@2-代理的接口，即使用了@FeignClient的接口
    //@3-InvocationHandler接口，通过代理对象调用方法时其invoke()方法会被调用
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```
看下怎么处理接口中声明的方法吧，默认是在Contract.BaseContract#parseAndValidateMetadata进行解析处理的。默认的contract是SpringMvcContract，而在父类BaseContract提供一组模板方法，实际上是由SpringMvcContract实现的，获取到注解的数据后会保存到MethodMetadata中，再用MethodHandler(SynchronousMethodHandler)进行封装。

```java
    protected MethodMetadata parseAndValidateMetadata(Class<?> targetType, Method method) {
      MethodMetadata data = new MethodMetadata();
      data.returnType(Types.resolve(targetType, targetType, method.getGenericReturnType()));
      data.configKey(Feign.configKey(targetType, method));

      if(targetType.getInterfaces().length == 1) {
        processAnnotationOnClass(data, targetType.getInterfaces()[0]);
      }
      //处理作用在接口上的@RequestMapping
      processAnnotationOnClass(data, targetType);

      //处理作用在方法上的@RequestMapping
      for (Annotation methodAnnotation : method.getAnnotations()) {
        processAnnotationOnMethod(data, methodAnnotation, method);
      }
      checkState(data.template().method() != null,
                 "Method %s not annotated with HTTP method type (ex. GET, POST)",
                 method.getName());
      Class<?>[] parameterTypes = method.getParameterTypes();
      Type[] genericParameterTypes = method.getGenericParameterTypes();

      Annotation[][] parameterAnnotations = method.getParameterAnnotations();
      int count = parameterAnnotations.length;
      for (int i = 0; i < count; i++) {
        boolean isHttpAnnotation = false;
        if (parameterAnnotations[i] != null) {
            //处理作用在方法参数上的注解:@PathVariable、@RequestParam、@RequestHeader
          isHttpAnnotation = processAnnotationsOnParameter(data, parameterAnnotations[i], i);
        }
        if (parameterTypes[i] == URI.class) {
          data.urlIndex(i);
        } else if (!isHttpAnnotation) {
          checkState(data.formParams().isEmpty(),
                     "Body parameters cannot be used with form parameters.");
          checkState(data.bodyIndex() == null, "Method has too many Body parameters: %s", method);
          data.bodyIndex(i);
          data.bodyType(Types.resolve(targetType, targetType, genericParameterTypes[i]));
        }
      }

      if (data.headerMapIndex() != null) {
        checkMapString("HeaderMap", parameterTypes[data.headerMapIndex()], genericParameterTypes[data.headerMapIndex()]);
      }

      if (data.queryMapIndex() != null) {
        checkMapString("QueryMap", parameterTypes[data.queryMapIndex()], genericParameterTypes[data.queryMapIndex()]);
      }

      return data;
    }
```


**总结下，首先通过@EnableFeignClients注解导入可以注册Bean定义的组件FeignClientsRegistrar，这个组件会扫描所有的feign客户端，向容器注册Bean定义，设置Class为FeignClientFactoryBean，然后在getObject中创建代理对象。首先获取接口中声明的所有方法，封装成MethodHandler，再生成InvocationHandler，继而就是jdk动态代理了。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**