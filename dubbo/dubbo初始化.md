dubbo的自动配置类为`DubboAutoConfiguration`，默认为SingleDubboConfigConfiguration单配置源，dubbo也支持多配置源。SingleDubboConfigConfiguration存在`@EnableDubboConfig`注解，会导入DubboConfigConfiguration.Single类型Bean。DubboConfigConfiguration.Single的注解`@EnableDubboConfigBindings`会导入DubboConfigBindingsRegistrar组件，DubboConfigBindingsRegistrar组件实现了ImportBeanDefinitionRegistrar接口，spring启动时会调用这个接口向容器注册Bean定义。
```java
    @EnableDubboConfigBindings({
            @EnableDubboConfigBinding(prefix = "dubbo.application", type = ApplicationConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.module", type = ModuleConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.registry", type = RegistryConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.protocol", type = ProtocolConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.monitor", type = MonitorConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.provider", type = ProviderConfig.class),
            @EnableDubboConfigBinding(prefix = "dubbo.consumer", type = ConsumerConfig.class)
    })
    public static class Single {

    }
```
接下来会通过DubboConfigBindingRegistrar#registerDubboConfigBean注册`@EnableDubboConfigBinding`中type属性定义的配置类的BeanDefinition，并且会在registerDubboConfigBindingBeanPostProcessor()中为每个配置类注册一个DubboConfigBindingBeanPostProcessor组件，DubboConfigBindingBeanPostProcessor组件用来将配置文件中的配置项和配置类属性绑定。分析DubboConfigBindingBeanPostProcessor类知道其实现了BeanPostProcessor，在Bean实例化时会对Bean做后置处理，此时会将配置项的值注入配置类的属性中。
```java
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

        if (beanName.equals(this.beanName) && bean instanceof AbstractConfig) {

            AbstractConfig dubboConfig = (AbstractConfig) bean;

            dubboConfigBinder.bind(prefix, dubboConfig);

            if (log.isInfoEnabled()) {
                log.info("The properties of bean [name : " + beanName + "] have been binding by prefix of " +
                        "configuration properties : " + prefix);
            }
        }

        return bean;

    }
```
对主类使用@EnableDubbo注解后会导入ServiceAnnotationBeanPostProcessor组件和ReferenceAnnotationBeanPostProcessor组件，其实没加@EnableDubbo也会在DubboAutoConfiguration自动配置类中导入，这两个组件分别处理@Service和@Reference注解。