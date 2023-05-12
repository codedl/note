spring创建bean时默认都是单例的，通过DefaultSingletonBeanRegistry.java#getSingleton方法添加到map中缓存，之后我们获取到的都是单例bean，但是这样是存在线程问题的。比如单例bean存在某个和线程相关的属性，在不同线程中取值不同的，这些值都是根据用户传参决定的。例如，商城用户是个男性，则喜欢看电子类产品；如果是个女性则喜欢化妆品。这时可能会出现一种情况，单例bean发现登录商城的是个男性，而同一时间有个女性也登陆了商城，单例bean于是把男性改成女性，向男性推荐化妆品。结合代码看下吧，
```java
@RestController
@RequestMapping("/person")
public class SingletonBeanController {
    private String iSex;

    private Map<String, String> map = new HashMap(){{
        put("man", "short hair, 180");
        put("woman", "long hair, big eye");
    }
    };

    @GetMapping("{sex}")
    String test(@PathVariable String sex){
        this.iSex = sex;
        return map.get(iSex);
    }
}
```
最初这是没有问题的，但是随着业务量增加，test()方法变成这样，sleep()表示中间进行了很多业务操作。
```java
    @GetMapping("{sex}")
    String test(@PathVariable String sex){
        this.iSex = sex;
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return map.get(iSex);
    }
```
此时，如果有两个用户同时调用接口，第一个男性用户时iSex为"man"，第二个女性用户时iSex为"woman"，但是第二个女性用户会把第一个男性用户的iSex改为"woman"，因为这是单例bean，在高并发时很容易出现这种情况，结果就是男性用户看到的全是女性商品。解决办法也很简单，可以通过@Scope定义当前bean为原型bean，`@Scope(value = BeanDefinition.SCOPE_PROTOTYPE)`，但不是很建议，因为这样每次都创建，也可以通过ThreadLocal线程变量来解决。
```java
    private ThreadLocal<String> iSex = new ThreadLocal<>();
    @GetMapping("{sex}")
    String test(@PathVariable String sex){
        this.iSex.set(sex);
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return map.get(iSex.get());
    }
```
如果这个属性类型是简单得类型，不会占用太多内存，很容易生成得话，大可以定义个方法内局部变量。
```java
    @GetMapping("/shop/{sex}")
    String sex(@PathVariable String sex){
        String iSex = sex;
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return map.get(iSex);
    }
```