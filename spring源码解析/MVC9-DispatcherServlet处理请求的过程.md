---
title: MVC9-DispatcherServlet处理请求的过程
tags:
notebook: spring源码解析
---
spring调用DispatcherServlet类中的doDispatch()方法开始处理请求
1. 检查是否为文件上传的请求，即请求的内容类型"content-type"是否以"multipart/"开头；如果是就将请求封装成StandardMultipartHttpServletRequest
2. spring在启动时会初始化Controller控制器，现在则根据请求获取执行器链
   1. RequestMappingInfoHandlerMapping处理器映射实现了InitializingBean接口，在完成bean的初始化之后会调用afterPropertiesSet()方法，在此方法中会调用detectHandlerMethods(beanName)对应用中所有的控制器进行初始化和注册。
   2. 获取Controllor的Class，对Class中的所有方法遍历；判断方法上是否使用@RequestMapping注解，如果存在则创建RequestMappingInfo保存注解信息，将作用在Controllor控制器类上@RequestMapping注解与方法上的混合
   3. 将Controllor控制器方法注册到处理器映射上
      1. 生成HandlerMethod，设置类型，方法，参数
      2. 将RequestMappingInfo和HandlerMethod的映射关系缓存下来
   4. 根据请求路径获取最佳处理方法
   5. 生成执行器链，将所有的拦截器添加到执行器链
3. 获取适配器
4. 如果是获取静态资源的GET请求或HEAD请求，根据"If-Modified-Since"请求头判断是否需求继续处理
5. 执行拦截器的preHandle()方法
6. 执行请求处理方法
   1. 解析参数
   2. 反射调用目标方法
   3. 返回值处理
7. 逆向执行拦截器的postHandle()方法，如果出现异常则逆向执行拦截器的afterCompletion()方法
   