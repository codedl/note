对接口使用@FeignClient后声明feign客户端后，可以使用属性fallback指定异常处理类，这个类必须实现@FeignClient作用的接口，且被注入到容器中。
```java
@FeignClient(name = "service-provider1",fallback = NacosFeignImpl.class)
public interface NacosFeignClient {
    @RequestMapping(value = "/echo/{str}",method = RequestMethod.GET)
    String echo(@PathVariable("str") String str);
}
@Component
public class NacosFeignImpl implements NacosFeignClient{
    @Override
    public String echo(String str) {
        System.out.println("NacosFeignImpl#echo called");
        return "echo error";
    }
}
```
添加配置项`feign.hystrix.enabled=true`，当配置了HystrixFeign时，即会创建HystrixFeign.Builder。
```java
// FeignClientsConfiguration.java
	@Configuration
	@ConditionalOnClass({ HystrixCommand.class, HystrixFeign.class })
	protected static class HystrixFeignConfiguration {
		@Bean
		@Scope("prototype")
		@ConditionalOnMissingBean
		@ConditionalOnProperty(name = "feign.hystrix.enabled")
		public Feign.Builder feignHystrixBuilder() {
			return HystrixFeign.builder();
		}
	}
```
在HystrixTargeter#target()发现使用了HystrixFeign.Builder并且属性fallback存在时，就会通过targetWithFallback()创建feign客户端。这里会到容器中获取实现了@FeignClient接口的Bean。
```java
	public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign, FeignContext context,
						Target.HardCodedTarget<T> target) {
		if (!(feign instanceof feign.hystrix.HystrixFeign.Builder)) {
			return feign.target(target);
		}
		feign.hystrix.HystrixFeign.Builder builder = (feign.hystrix.HystrixFeign.Builder) feign;
		SetterFactory setterFactory = getOptional(factory.getName(), context,
			SetterFactory.class);
		if (setterFactory != null) {
			builder.setterFactory(setterFactory);
		}
		Class<?> fallback = factory.getFallback();
		if (fallback != void.class) {
			return targetWithFallback(factory.getName(), context, target, builder, fallback);
		}
		Class<?> fallbackFactory = factory.getFallbackFactory();
		if (fallbackFactory != void.class) {
			return targetWithFallbackFactory(factory.getName(), context, target, builder, fallbackFactory);
		}

		return feign.target(target);
	}
```
此时创建InvocationHandlerFactory，注入到属性invocationHandlerFactory，invocationHandlerFactory默认是`InvocationHandlerFactory.Default`，这里重写create()方法，创建了HystrixInvocationHandler的实例对象。
```java
    Feign build(final FallbackFactory<?> nullableFallbackFactory) {
      super.invocationHandlerFactory(new InvocationHandlerFactory() {
        @Override public InvocationHandler create(Target target,
            Map<Method, MethodHandler> dispatch) {
          return new HystrixInvocationHandler(target, dispatch, setterFactory, nullableFallbackFactory);
        }
      });
      super.contract(new HystrixDelegatingContract(contract));
      return super.build();
    }
```
生成代理对象时，就会调用create()对HystrixInvocationHandler实例化，这样就会调用HystrixInvocationHandler#invoke()。
```java
  public <T> T newInstance(Target<T> target) {
    ......
    InvocationHandler handler = factory.create(target, methodToHandler);
    T proxy = (T) Proxy.newProxyInstance(target.type().getClassLoader(), new Class<?>[]{target.type()}, handler);

    for(DefaultMethodHandler defaultMethodHandler : defaultMethodHandlers) {
      defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
  }
```
invoke()中run()的逻辑还是调用SynchronousMethodHandler#invoke()处理请求，发生异常时就会到getFallback()调用接口的实现类对象的方法。
```java
  public Object invoke(final Object proxy, final Method method, final Object[] args)
      throws Throwable {
    // early exit if the invoked method is from java.lang.Object
    // code is the same as ReflectiveFeign.FeignInvocationHandler
    if ("equals".equals(method.getName())) {
      try {
        Object otherHandler =
            args.length > 0 && args[0] != null ? Proxy.getInvocationHandler(args[0]) : null;
        return equals(otherHandler);
      } catch (IllegalArgumentException e) {
        return false;
      }
    } else if ("hashCode".equals(method.getName())) {
      return hashCode();
    } else if ("toString".equals(method.getName())) {
      return toString();
    }

    HystrixCommand<Object> hystrixCommand = new HystrixCommand<Object>(setterMethodMap.get(method)) {
      @Override
      protected Object run() throws Exception {
        try {
          return HystrixInvocationHandler.this.dispatch.get(method).invoke(args);
        } catch (Exception e) {
          throw e;
        } catch (Throwable t) {
          throw (Error) t;
        }
      }

      @Override
      protected Object getFallback() {
        if (fallbackFactory == null) {
          return super.getFallback();
        }
        try {
          Object fallback = fallbackFactory.create(getExecutionException());
          Object result = fallbackMethodMap.get(method).invoke(fallback, args);
          if (isReturnsHystrixCommand(method)) {
            return ((HystrixCommand) result).execute();
          } else if (isReturnsObservable(method)) {
            // Create a cold Observable
            return ((Observable) result).toBlocking().first();
          } else if (isReturnsSingle(method)) {
            // Create a cold Observable as a Single
            return ((Single) result).toObservable().toBlocking().first();
          } else if (isReturnsCompletable(method)) {
            ((Completable) result).await();
            return null;
          } else {
            return result;
          }
        } catch (IllegalAccessException e) {
          // shouldn't happen as method is public due to being an interface
          throw new AssertionError(e);
        } catch (InvocationTargetException e) {
          // Exceptions on fallback are tossed by Hystrix
          throw new AssertionError(e.getCause());
        }
      }
    };

    if (isReturnsHystrixCommand(method)) {
      return hystrixCommand;
    } else if (isReturnsObservable(method)) {
      // Create a cold Observable
      return hystrixCommand.toObservable();
    } else if (isReturnsSingle(method)) {
      // Create a cold Observable as a Single
      return hystrixCommand.toObservable().toSingle();
    } else if (isReturnsCompletable(method)) {
      return hystrixCommand.toObservable().toCompletable();
    }
    return hystrixCommand.execute();
  }
```