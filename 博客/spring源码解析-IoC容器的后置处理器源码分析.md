---
title: spring源码解析-IoC容器的后置处理器源码分析
tags:
notebook: 博客
---
# 后置处理器的使用
在spring中可以使用容器的后置处理器对容器进行增强处理，常用的分别有两类为BeanDefinitionRegistryPostProcessor和BeanFactoryPostProcessor。BeanDefinitionRegistryPostProcessor中可以拿到BeanDefinitionRegistry向容器注册新的bean的定义，而BeanFactoryPostProcessor可以修改已经注册的bean的定义，比如可以设置bean的自动注入模式为AUTOWIRE_BY_TYPE，可以省区一大堆的@Autowired注解。话不多说，开始演练([也可以直接看源码](#invokebeanfactorypostprocessors方法分析))。
```
//这里定义了一个组件，并没有对属性使用注解注入，所以需要声明属性的注入方式
//常见的注入方式有按类型和按名称
@Component
public class AutowiredTypeUser {
    public String name = "AutowiredUser";
    ComponentUser componentUser;

    @Override
    public String toString() {
        return "AutowiredTypeUser{" +
                "name='" + name + '\'' +
                "componentUser=" + componentUser +
                '}';
    }

    public ComponentUser getComponentUser() {
        return componentUser;
    }

    public void setComponentUser(ComponentUser componentUser) {
        this.componentUser = componentUser;
    }

}
```
```
//这里我们实现了一个BeanDefinitionRegistryPostProcessor类型的后置处理器
//在后置处理器中我们注册了名为MyBeanFactoryPostProcessor的bean
@Component
public class MyBeanDefinitionRegistryPostProcessor implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {
        registry.registerBeanDefinition("MyBeanFactoryPostProcessor",new RootBeanDefinition(MyBeanFactoryPostProcessor.class));
        System.out.println("MyBeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        System.out.println("MyBeanDefinitionRegistryPostProcessor#postProcessBeanFactory");
    }
}
```
```
//这里我们实现了BeanFactoryPostProcessor类型的后置处理器
//可以看到这里我们可以按name获取bean的定义，此时我们可以对bean的定义进行配置，所有使用注解的配置都可以在此进行配置
//比如setDependsOn声明依赖的bean，与@DependsOn注解等效
public class MyBeanFactoryPostProcessor implements BeanFactoryPostProcessor {
    static {
        System.out.println("BeanFactoryPostProcess#static");
    }

    public MyBeanFactoryPostProcessor() {
        System.out.println("MyBeanFactoryPostProcessor#new");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        AbstractBeanDefinition beanDefinition =(AbstractBeanDefinition) beanFactory.getBeanDefinition("autowiredTypeUser");
        beanDefinition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
        System.out.println("MyBeanFactoryPostProcessor#postProcessBeanFactory");
    }
}
```
代码逻辑很简单，先创建BeanDefinitionRegistryPostProcessor后置处理器，在方法postProcessBeanDefinitionRegistry中再创建BeanFactoryPostProcessor后置处理器，最后在BeanFactoryPostProcessor的方法postProcessBeanFactory修改autowiredTypeUser的定义。可能有小伙伴觉得奇怪，为啥不直接创建MyBeanFactoryPostProcessor组件，这里卖个关，因为我们得去阅读spring中容器的后置处理器的源码，这样做的话更容易理解spring源码。看效果，走起：
```
2022-09-02 09:46:23.093  INFO 8324 --- [           main] c.e.s.SpringSourceApplicationTests       : Starting SpringSourceApplicationTests on leding2-pc with PID 8324 (started by leding2 in E:\Desktop\code\study\spring-source)
2022-09-02 09:46:23.097  INFO 8324 --- [           main] c.e.s.SpringSourceApplicationTests       : No active profile set, falling back to default profiles: default
MyBeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry
MyBeanDefinitionRegistryPostProcessor#postProcessBeanFactory
BeanFactoryPostProcess#static
MyBeanFactoryPostProcessor#new
MyBeanFactoryPostProcessor#postProcessBeanFactory
初始化...
2022-09-02 09:46:24.650  INFO 8324 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2022-09-02 09:46:25.005  INFO 8324 --- [           main] c.e.s.SpringSourceApplicationTests       : Started SpringSourceApplicationTests in 2.298 seconds (JVM running for 3.64)
AutowiredTypeUser{name='AutowiredUser'componentUser=ComponentUser{, depondOnUser=DepondOnUser, name='ComponentUser'}}
```

可以看到效果，跟前面提到的一样，AutowiredTypeUser被注入了ComponentUser组件，而并没有使用注解驱动，工作中是不是可以简化开发，省去一大堆注解。
# 源码分析
现在进入正题，开始分析后置处理器的源码。
首先容器启动时会实例化容器，在GenericApplicationContext类中：
```
this.beanFactory = new DefaultListableBeanFactory();
```
而这个类实现了BeanDefinitionRegistry
```
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable
```
接下来到容器刷新阶段，执行AbstractApplicationContext类中refresh方法，此时会调用invokeBeanFactoryPostProcessors实现后置处理器，所以接下来只要对invokeBeanFactoryPostProcessors分析即可。
```
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();

        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);

        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);

            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);

            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);

            // Initialize message source for this context.
            initMessageSource();

            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();

            // Initialize other special beans in specific context subclasses.
            onRefresh();

            // Check for listener beans and register them.
            registerListeners();

            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);

            // Last step: publish corresponding event.
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                        "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();

            // Reset 'active' flag.
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            resetCommonCaches();
        }
    }
}
```

在invokeBeanFactoryPostProcessors方法中把后置处理器分成BeanFactoryPostProcessor和BeanDefinitionRegistryPostProcessor两类，分别用两个ArrayList保存。首先处理应用启动时添加的后置处理器，按优先级排序，执行BeanDefinitionRegistryPostProcessor中的postProcessBeanDefinitionRegistry方法。然后到容器获取BeanDefinitionRegistryPostProcessor类型的处理器，执行postProcessBeanDefinitionRegistry方法。由于执行了后置处理器导致容器中可能会新增很多新的后置处理器，所以需要重复处理，先处理实现了Ordered的接口。最后执行当前所有后置处理器的postProcessBeanFactory方法。接下来再到容器获取BeanFactoryPostProcessor处理器，依次处理实现了PriorityOrdered、Ordered以及普通的后置处理器。先按优先级排序，再执行postProcessBeanFactory方法，我们平时所声明的后置处理器就是在这里被处理的。
扩展:可以在BeanDefinitionRegistryPostProcessor后置处理器中添加新的后置处理器。
```
public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // Invoke BeanDefinitionRegistryPostProcessors first, if any.
    Set<String> processedBeans = new HashSet<>();
/*
this.beanFactory = new DefaultListableBeanFactory()  
public class DefaultListableBeanFactory extends AbstractAutowireCapableBeanFactory
		implements ConfigurableListableBeanFactory, BeanDefinitionRegistry, Serializable
在容器实例化时使用的是DefaultListableBeanFactory，实现了BeanDefinitionRegistry              
*/
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
/*
对后置处理器按实现的接口分类成
BeanFactoryPostProcessor
BeanDefinitionRegistryPostProcessor
*/        
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();
/*
先对应用启动时添加的后置处理器进行处理
*/
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
/*
这里调用的是void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry);
*/                        
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans
        // uninitialized to let the bean factory post-processors apply to them!
        // Separate between BeanDefinitionRegistryPostProcessors that implement
        // PriorityOrdered, Ordered, and the rest.
/*
当前正在处理的BeanDefinitionRegistryPostProcessor类型后置处理器
*/        
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
/*
然后从容器中找到实现了BeanDefinitionRegistryPostProcessor接口的后置处理器
*/        
        String[] postProcessorNames =
                beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
/*
找到后置处理器后加到currentRegistryProcessors中，表示当前正在被处理；加到processedBeans中，表示已经处理过，防止重复处理
*/                
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
/*
按优先级排序
*/        
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
/*
执行currentRegistryProcessors包含的后置处理器方法，执行完清空currentRegistryProcessors
>ConfigurationClassPostProcessor.java#postProcessBeanDefinitionRegistry
这里会对包路径进行扫描，从主类开始依次处理配置类上的注解
*/
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();

        // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
/*
再次从容器获取实现了BeanDefinitionRegistryPostProcessor接口的处理器，因为刚刚对包路径进行了扫描，可能会有新的后置处理器加入到容器中，此时会通过processedBeans排除已经处理过得后置处理器，然后再次进行刚才同样得逻辑，先处理实现了Ordered接口的后置处理器。
*/        
        postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        registryProcessors.addAll(currentRegistryProcessors);
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
        currentRegistryProcessors.clear();
/*
在一个循环中获取后置处理器，对所有后置处理器进行处理，直到没有新的后置处理器出现为止
*/
        // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
        boolean reiterate = true;
        while (reiterate) {
            reiterate = false;
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
            }
            sortPostProcessors(currentRegistryProcessors, beanFactory);
            registryProcessors.addAll(currentRegistryProcessors);
/*
我们定义的MyBeanDefinitionRegistryPostProcessor就是在这里被处理的
*/            
            invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
            currentRegistryProcessors.clear();
        }

        // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
/*
执行所有后置处理器得postProcessBeanFactory方法，因为BeanDefinitionRegistryPostProcessor是BeanFactoryPostProcessor的子接口，所以也实现了postProcessBeanFactory方法
*/        
        invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
        invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        // Invoke factory processors registered with the context instance.
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let the bean factory post-processors apply to them!
/*
从容器中获取所有实现了BeanFactoryPostProcessor接口的后置处理器
*/    
    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
    // Ordered, and the rest.
/*
先处理实现了PriorityOrdered接口的后置处理器，再处理实现了Ordered接口的后置处理器，再处理普通无序的后置处理器；先排序再处理,处理的时候先getBean获取处理器的实例对象，再调用方法
*/    
    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    for (String ppName : postProcessorNames) {
        //如果处理过了就不做任何处理
        if (processedBeans.contains(ppName)) {
            // skip - already processed in first phase above
        }
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
    List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // Finally, invoke all other BeanFactoryPostProcessors.
/*
通常我们定义的普通的后置处理器就是在这里被处理的，MyBeanFactoryPostProcessor就是在这里被处理的
*/    
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    // Clear cached merged bean definitions since the post-processors might have
    // modified the original metadata, e.g. replacing placeholders in values...
    beanFactory.clearMetadataCache();
}
```
