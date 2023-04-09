首先创建父工程，用于管理依赖。
```xml
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.4.5</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>org.example</groupId>
    <artifactId>DubboSpringBoot</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>1.8</java.version>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <maven.compiler.encoding>UTF-8</maven.compiler.encoding>
        <springboot-dubbo-version>1.0.0</springboot-dubbo-version>
        <zk-version>0.7</zk-version>
        <internal-version>0.0.1-SNAPSHOT</internal-version>
        <bean-version>5.3.6</bean-version>
        <spring-version>2.3.5.RELEASE</spring-version>
    </properties>

    <dependencies>
    </dependencies>
    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-web</artifactId>
                <version>${spring-version}</version>
            </dependency>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-starter-test</artifactId>
                <version>${spring-version}</version>
                <scope>test</scope>
            </dependency>

            <dependency>
                <groupId>com.alibaba.boot</groupId>
                <artifactId>dubbo-spring-boot-starter</artifactId>
                <version>0.2.0</version>
            </dependency>

            <dependency>
                <groupId>com.xiu.dubbo</groupId>
                <artifactId>demo-api</artifactId>
                <version>${internal-version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
```
再创建一个通用的模块用作其他模块的依赖，定义一些可以复用的接口。
```xml
    <parent>
        <artifactId>DubboSpringBoot</artifactId>
        <groupId>org.example</groupId>
        <version>1.0-SNAPSHOT</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>Common</artifactId>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
```
然后就是服务提供者。
+ pom.xml
```xml
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <!-- 整合dubbo -->
        <dependency>
            <groupId>com.alibaba.boot</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
        </dependency>

        <dependency>
            <groupId>org.example</groupId>
            <artifactId>Common</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
    </dependencies>
```
+ 配置文件
```
server.port=8000
dubbo.application.name=provider

dubbo.registry.address=172.29.67.170:2181
dubbo.registry.protocol=zookeeper

dubbo.protocol.name=dubbo
dubbo.protocol.port=20880
#
#dubbo.monitor.protocol=registry
dubbo.scan.base-packages=org.example
```
+ 发布接口，主类开启dubbo功能
```java
@Component
@Service
public class UserServiceImpl implements UserService{
    @Override
    public String echo(String str) {
        return "Provider " + str;
    }
}

@EnableDubbo
@SpringBootApplication
public class ProviderMain {
    public static void main(String[] args) {
        SpringApplication.run(ProviderMain.class);
    }
}
```
最后就是服务消费者，pom和配置类似提供者。
```java
@EnableDubbo
@SpringBootApplication
public class ConsumerMain {
    public static void main(String[] args) {
        SpringApplication.run(ConsumerMain.class);
    }

    @RestController
    public class test{
        @Reference
        UserService userService;

        @RequestMapping("echo/{str}")
        public String echo(@PathVariable("str") String str){
            return userService.echo(str);
        }
    }
}
```
