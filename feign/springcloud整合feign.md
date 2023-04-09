在springcloud框架中可以使用Feign客户端调用微服务，那么就要对Feign框架进行整合，首先在项目pom.xml中加入openfeign的依赖。
```xml
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-openfeign</artifactId>
            <version>2.0.0.RELEASE</version>
        </dependency>
```
Feign是面向接口的编程，只要对接口使用注解就能完成微服务的调用，底层是通过动态代理实现的。@FeignClient可以声明要调用的微服务名，对接口中声明的方法使用@RequestMapping来映射要调用的微服务接口路径。
```java
@FeignClient("service-provider")
public interface NacosFeignClient {
    @RequestMapping(value = "/echo/{str}",method = RequestMethod.GET)
    String echo(@PathVariable("str") String str);
}
```
调用时只要注入Feign到属性中，直接调用方法即可。
```java
        @Autowired
        NacosFeignClient nacosFeignClient;

        @RequestMapping(value = "/feign/{str}", method = RequestMethod.GET)
        public String feign(@PathVariable String str) {
            return nacosFeignClient.echo(str);
        }
```
这样就完成了Feign的整合，接下来就是Feign的底层实现了。