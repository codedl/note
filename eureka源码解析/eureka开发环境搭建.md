---
title: eureka开发环境搭建
tags:
notebook: eureka源码解析
---
现在开始学另外一个微服务注册中心，虽然大部分企业都是用springcloudalibaba，但是eureka作为经典的微服务注册中心还是有很多设计值得借鉴，并且可以为我们的技术选型提供一种选择。先把eureka的开发环境搭建起来吧。
### 父工程
先创建工程，在idea里建个父工程，只要引入springcloud的相关依赖就行，以后多个子工程想添加依赖，只要在父工程添加就行。
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <packaging>pom</packaging>
    <modules>
        <module>../EurekaServer</module>
        <module>../EurekaServiceProvider</module>
        <module>../EurekaServiceConsumer</module>
    </modules>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.1.6.RELEASE</version>
    </parent>
    <groupId>org.example</groupId>
    <artifactId>parent</artifactId>
    <version>1.0-SNAPSHOT</version>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>Greenwich.RELEASE</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-commons</artifactId>
        </dependency>
    </dependencies>
</project>
```
### 服务端
eureka需要手动启动一个服务端作为微服务注册中心，其实也很简单，都是spring整合好的，建好子工程后先引入依赖
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>parent</artifactId>
        <groupId>org.example</groupId>
        <version>1.0-SNAPSHOT</version>
        <relativePath>../parent/pom.xml</relativePath>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>EurekaServer</artifactId>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
        </dependency>
    </dependencies>


</project>
```
接下来配置地址
```
server:  # 服务端口
  port: 9090
spring:
  application:  # 应用名字，eureka 会根据它作为服务id
    name: spring-cloud-eureka-server
eureka:
  instance:
    hostname: localhost
  client:
    service-url:   #  eureka server 的地址， 咱们单实例模式就写自己好了
      defaultZone:  http://localhost:9090/eureka
    register-with-eureka: false  # 不向eureka server 注册自己
    fetch-registry: false  # 不向eureka server 获取服务列表

```
然后再到主类上标记，就是使用@EnableEurekaServer注解
```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServer {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServer.class);
    }
}
```
### 客户端
服务端有了，接下来只要通过客户端向服务端注册微服务就行了，也不麻烦。先引入依赖，
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>parent</artifactId>
        <groupId>org.example</groupId>
        <version>1.0-SNAPSHOT</version>
        <relativePath>../parent/pom.xml</relativePath>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>EurekaServiceProvider</artifactId>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
        </dependency>
    </dependencies>


</project>
```
添加配置
```
server:
  port: 7070
spring:
  application:
    name: service-provider
eureka:
  client:
    service-url:
      defaultZone: http://127.0.0.1:9090/eureka
    fetch-registry: true
    register-with-eureka: true
```
在标记为eureka客户端，提供个api供其他微服务客户端发现和调用。
```java
@SpringBootApplication
@EnableEurekaClient
public class ServiceProvider {
    public static void main(String[] args) {
        SpringApplication.run(ServiceProvider.class);
    }

    @RestController
    public class OrderStatisticServiceController {
        @Value("${server.port}")
        private Integer port;

        @GetMapping("/echo/{str}")
        public String getTodayFinishOrderNum(@PathVariable("str") String str){
            return port + str;
        }
    }

}
```
再来个客户端调用其他客户端的微服务
```java
@SpringBootApplication
@EnableEurekaClient
public class ServiceConsumer {
    public static void main(String[] args) {
        SpringApplication.run(ServiceConsumer.class);
    }

    @LoadBalanced
    @Bean
    RestTemplate restTemplate(){
        return new RestTemplate();
    }

    @RestController
    public class UserCenterController {
        @Autowired
        private RestTemplate restTemplate;

        @GetMapping("/echo/{str}")
        public String echo(@PathVariable("str") String str){
            return restTemplate.getForObject("http://SERVICE-PROVIDER/echo/" + str, String.class);
        }
    }

}
```
### 集群
eureka集群的时候其他不变，首先在`C:\Windows\System32\drivers\etc\hosts`文件添加
```
127.0.0.1       eureka9090
127.0.0.1       eureka9091
127.0.0.1       eureka9092
```
再修改配置文件向其他eureka节点注册就行
```
server:  # 服务端口
  port: 9090
spring:
  application:  # 应用名字，eureka 会根据它作为服务id
    name: eureka-server-9090
eureka:
  dashboard:
    path: /
  instance:
    hostname: eureka9090
  client:
    availabilityZones:  [defaultZone, defaultZone1]
    service-url:   #  eureka server 的地址， 咱们单实例模式就写自己好了
      defaultZone:  http://eureka9091:9091/eureka,http://eureka9092:9092/eureka
    register-with-eureka: false  # 不向eureka server 注册自己
    fetch-registry: false  # 不向eureka server 获取服务列表

```
现在整个环境就搭建好了，接下来就可以随便折腾了。