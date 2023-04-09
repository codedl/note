---
title: 组件实例化14-Bean的后置处理器
tags:
notebook: spring源码解析
---
1. 注册Bean的后置处理器
   1. 在PostProcessorRegistrationDelegate.java的registerBeanPostProcessors方法找到实现了BeanPostProcessor接口的bean name
   2. 通过bean name的获取bean的定义，创建bean
   3. 注册到CopyOnWriteArrayList中
2. BeanPostProcessor后置处理器
   1. 生成bean的实例并且完成了属性填充之后调用postProcessBeforeInitialization初始化前方法
   2. 初始化以后调用postProcessAfterInitialization初始化后方法
3. InstantiationAwareBeanPostProcessor后置处理器
   1. 被AbstractAutowireCapableBeanFactory.java的resolveBeforeInstantiation方法调用，在创建bean之前尝试生成bean的代理对象
   2. 创建bean完成以后调用postProcessProperties对bean的属性进行填充，注入依赖
4. SmartInstantiationAwareBeanPostProcessor后置处理器
   1. AbstractAutowireCapableBeanFactory.java类的getEarlyBeanReference方法中被调用，通过getEarlyBeanReference方法获取bean的早期引用对象，即通过三级缓存获取早期实例对象