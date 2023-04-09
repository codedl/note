---
title: @RefreshScope注解处理
tags:
notebook: nacos源码解析
---
spring启动时会调用ClassPathBeanDefinitionScanner.java类中的doScan()对包路径下的所有class进行扫描，获取bean的定义，同时对bean的@RefreshScope(@Scope的父类)进行处理。
1. 获取作用在类上的@RefreshScope(@Scope的父类)注解元数据，封装成ScopeMetadata对象，同时声明bean的作用域`candidate.setScope(scopeMetadata.getScopeName());`
2. 根据scopeMetadata即@Scope对bean进行代理，使用ScopedProxyFactoryBean类创建RootBeanDefinition代替原理的bean定义。将被代理的原始bean定义注册到容器中(scopedTarget.configController -> ConfigController)，再将代理的bean定义注册到容器中(configController -> ScopedProxyFactoryBean)  

处理这个使用@RefreshScope注解的bean的是RefreshScope组件，可以看到父类GenericScope实现了BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor接口，这是容器的后置处理器，当容器启动时会先调用BeanDefinitionRegistryPostProcessor接口中的postProcessBeanDefinitionRegistry()方法，再BeanFactoryPostProcessor接口中的postProcessBeanFactory()方法。
```java
public class RefreshScope extends GenericScope
		implements ApplicationContextAware, Ordered{...}
public class GenericScope implements Scope, BeanFactoryPostProcessor,
		BeanDefinitionRegistryPostProcessor, DisposableBean{...}        
```
获取容器中包含的所有的bean定义，再通过bean装饰的bean定义获取作用域，判断是否为RefreshScope能处理的作用域，在构造RefreshScope实例时设置了name为"refresh"`getName().equals(root.getDecoratedDefinition().getBeanDefinition().getScope())`。ps:这是个细节，在被装饰的bean定义的scope是"refresh"，而装饰者bean的scope是""；之后处理的时候先处理装饰者bean，在处理原始bean，就是通过这个scope来判断的。如果是的话将bean的class设置为LockedScopedProxyFactoryBean，将当前RefreshScope实例对象设置为构造函数的参数。看下LockedScopedProxyFactoryBean这个类：
```java
public static class LockedScopedProxyFactoryBean<S extends GenericScope>
			extends ScopedProxyFactoryBean implements MethodInterceptor{...}
public class ScopedProxyFactoryBean extends ProxyConfig implements FactoryBean<Object>, BeanFactoryAware{...}            
```
可以看到LockedScopedProxyFactoryBean的父类实现了BeanFactoryAware接口，因此在bean实例化的时候会调用实现的setBeanFactory()方法，这时会通过代理工厂创建代理对象，即生成LockedScopedProxyFactoryBean类的代理对象。由于LockedScopedProxyFactoryBean类实现了Advised接口，因此可以作为代理对象的拦截器，当调用代理对象的方法时拦截器就会起作用。
```java
public void setBeanFactory(BeanFactory beanFactory) {
......
    ProxyFactory pf = new ProxyFactory();
    pf.copyFrom(this);
    pf.setTargetSource(this.scopedTargetSource);
.....
    this.proxy = pf.getProxy(cbf.getBeanClassLoader());
}
public void setBeanFactory(BeanFactory beanFactory) {
    super.setBeanFactory(beanFactory);
    Object proxy = getObject();
    if (proxy instanceof Advised) {
        Advised advised = (Advised) proxy;
        advised.addAdvice(0, this);
    }
}
```
总结下@RefreshScope注解的bean实例化的过程：
1. 使用了@RefreshScope注解的bean的定义会被设置成LockedScopedProxyFactoryBean.class
2. 创建bean时先生成LockedScopedProxyFactoryBean的实例对象
3. 由于LockedScopedProxyFactoryBean实现了BeanFactoryAware接口，会调用setBeanFactory()
4. 在setBeanFactory()方法中会通过cglib创建LockedScopedProxyFactoryBean对象的代理对象，FactoryBean接口的getObject()会返回此代理对象，因此最终得到的就是LockedScopedProxyFactoryBean对象的代理对
5. 由于代理对象为LockedScopedProxyFactoryBean的子类，因此实现了addAdvice接口，可以将调用setBeanFactory()方法的LockedScopedProxyFactoryBean对象作为拦截器保存下来  

当容器启动完成刷新后会发布ContextRefreshedEvent事件，RefreshScope类中的start()使用了@EventListener注解，可以作为监听器对ContextRefreshedEvent事进行处理。找到scope是"refresh"的bean的定义，再走Scope的代码块进行实例化。
```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
......
if (mbd.isSingleton()) {
    ......
}
else if (mbd.isPrototype()) {
    ......
}
else {
    String scopeName = mbd.getScope();
    //根据name获取Scope，当前一共有四个
    //refresh -> {RefreshScope@5469}
    //request -> {RequestScope@7462} 
    //session -> {SessionScope@7464} 
    //application -> {ServletContextScope@7466} 
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) {
        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
    }
    try {
        Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            try {
                return createBean(beanName, mbd, args);
            }
            finally {
                afterPrototypeCreation(beanName);
            }
        });
        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    }
}
......
}
```
获取到RefreshScope对象后就会调用其中的get()方法进行初始化。先创建BeanLifecycleWrapper对象，包含了bean的name和创建bean的工厂方法(lamdba表达式)，再调用BeanLifecycleWrapper类中的getBean()方法获取bean的实例对象，此时调用`this.bean = this.objectFactory.getObject();`工厂方法来创建bean，则回到`createBean(beanName, mbd, args);`方法完成bean的创建。接下来看看方法是怎么调用的。
1. 获取被代理原始bean的实例对象
2. 创建拦截器链，找到之前加入的LockedScopedProxyFactoryBean拦截器，调用invoke()方法进行处理
3. 获取代理对象，即LockedScopedProxyFactoryBean的代理对象；获取读写锁的读锁，当前配置刷新时，则无法获取
4. 以反射方式调用目标方法`method.invoke(target, args)`  

再看看配置怎么刷新到bean当中的。当配置源变更时方法调用链如下：
1. NacosContextRefresher#registerNacosListener()注册监听器(receiveConfigInfo())
2. ApplicationEventPublisher.java#publishEvent()发布RefreshEvent事件
3. SimpleApplicationEventMulticaster#multicastEvent()方法广播事件
4. RefreshEventListener#handle(RefreshEvent event)处理RefreshEvent事件
5. ContextRefresher#refresh()进行处理，做了三件事
   1. 刷新配置源
   2. 发布EnvironmentChangeEvent事件，重新生成配置源Bean(@ConfigurationProperties注解Bean)
   3. 将配置写到@RefreshScope注解标注的Bean

现在开始分析怎么刷新到@RefreshScope注解标注的Bean：将保存原始被代理的Bean的BeanLifecycleWrapperCache中清空，获取写锁(此时其他线程无法获取读锁)，调用每个BeanLifecycleWrapper的destroy()方法进行销毁。缓存被清空了，如果重新获取Bean就得重新创建，此时就会进行Bean的属性填充，就可以从属性源中获取新的属性了。至此总算完成了属性源的刷新了。
