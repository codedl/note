---
title: MVC2-DispatcherServlet初始化
tags:
notebook: spring源码解析
---
1. 在springboot的spring.factory配置文件中配置了要自动加载的配置类`DispatcherServletAutoConfiguration`,在类上使用了`@Configuration`注解，spring对类解析时，提取使用了@Bean注解的方法
`dispatcherServlet()`，之后spring会调用这个方法创建bean，创建bean时设置了spring.mvc相关的属性。在生成DispatcherServletRegistrationBean组件时，会注入dispatcherServlet组件，
同时添加映射的请求路径为"/"，这样dispatcherServlet组件就会对"/"路径下的请求进行处理。
2. 当dispatcherServlet第一次处理请求时，会调用init方法进行对Servlet进行初始化。
   1. 调用`initWebApplicationContext()`初始化web应用上下文
   2. 调用`initStrategies()`方法对dispatcherServlet组件进行初始化
   3. 记录已经完成初始化的应用上下文。
------
initStrategies()方法解析
1. 获取上传文件解析器`initMultipartResolver(context)`，到容器中获取名为`multipartResolver`，类型为`MultipartResolver`的bean当作文件上传解析器
2. 