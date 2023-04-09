---
title: MVC8-文件上传解析器
tags:
notebook: spring源码解析
---
1. 当DispatcherServlet第一次收到请求时，会调用类中initStrategies()方法进行初始化，此时会创建文件上传解析器initMultipartResolver()。
   1. 从容器中获取名称为`"multipartResolver"`，类型为`MultipartResolver.class`的bean，写到属性multipartResolver中。
      1. 在spring.factories中声明了`MultipartAutoConfiguration`为要自动加载的配置类
      2. 在`MultipartAutoConfiguration`类中对`multipartResolver()`使用了@Bean注解，因此spring在启动时会调用multipartResolver()创建文件上传解析器
      3. 对类MultipartProperties使用了@ConfigurationProperties(prefix = "spring.servlet.multipart", ignoreUnknownFields = false)，因此会与"spring.servlet.multipart"开头的配置项绑定
      4. 使用MultipartProperties属性创建MultipartProperties文件上传解析器的配置MultipartConfigElement，比如文件大小。
2. 当spring收到上传文件的请求时，会调用Request.java中的parseParameters()方法对请求参数进行解析，发现请求内容类型为"multipart/form-data"时，会进行文件解析。
   1. 获取tomcat存储文件的临时路径，用来缓存文件
   2. 根据MultipartConfigElement获取文件解析的配置
   3. 从请求中获取输入流，生成part保存下来
3. 在DispatcherServlet中对上传文件的请求进行封装
   1. 判断是否为文件上传的请求，即请求的内容类型是否以"multipart/"开头
   2. 从请求中获取所有的Part，封装成MultiValueMap，写到multipartFiles属性中；key为文件参数名，value为StandardMultipartFile对象
4. 对方法中的参数进行处理
   1. 根据supportsParameter()方法找到能够处理的参数解析器RequestPartMethodArgumentResolver，即判断方法上的参数是否使用了@RequestPart注解
   2. 获取方法参数上的注解@RequestPart，根据参数名到multipartFiles属性获取StandardMultipartFile文件对象，注入到参数中

