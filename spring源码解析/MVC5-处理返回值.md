---
title: MVC5-处理返回值
tags:
notebook: spring源码解析
---
spring会调用`HandlerMethodReturnValueHandlerComposite`类中的`handleReturnValue()`方法对返回值进行处理
1. 获取能够进行处理的返回值处理器
   1. 对现有的返回值处理器进行遍历
      1. 创建RequestMappingHandlerAdapter组件进行初始化时，会调用afterPropertiesSet()方法，此时会给适配器设置返回值处理器。
      2. spring会添加默认的返回值处理器以及自定义的返回值处理器。spring会先到容器中找到实现了WebMvcConfigurer接口的bean，保存到configurers中，然后调用每个config中的`addReturnValueHandlers()`来添加自定义的返回值处理器。
      3. 找到所有的返回值处理器后，添加到内部的属性对象returnValueHandlers的returnValueHandlers属性中。  
`this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);`  
`private final List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<>();`
    1. 调用每个返回值处理器的`supportsReturnType()`判断能否对当前的返回值进行处理，其实就是根据类型或注解进行判断。如果在类或方法的返回值上使用了@ResponseBody注解，则使用`RequestResponseBodyMethodProcessor`处理器
2. 使用找到的处理器对返回值进行处理
   1. 设置响应内容的类型，即内容协商
      1. 从http请求中获取请求头Accept指定的内容类型，即浏览器能够接受的内容类型
      2. 获取服务器能生成的内容类型：对所有的消息转换器进行遍历，先调用canWrite()判断能否进行处理返回值类型，再调用getSupportedMediaTypes()获取支持的媒体类型
      3. 将服务器能支持的媒体类型与浏览器的内容类型进行比较，确定要返回的内容类型
   2. 对所有的消息转换器进行遍历，调用`canWrite()`方法找到能够转换返回值类型的消息转换器
   3. 使用消息转换器对返回值进行处理
      1. 获取响应的内容类型和编码方式
      2. 从响应中获取OutputStream，通过OutputStream写入数据到响应
      3. 对返回值进行序列化，转成json格式字符串