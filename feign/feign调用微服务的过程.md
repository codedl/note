使用Feign客户端调用微服务时，就是通过接口的代理对象调用微服务，逻辑都是在SynchronousMethodHandler#invoke()方法调用。
```java
  public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = buildTemplateFromArgs.create(argv);
    Retryer retryer = this.retryer.clone();
    while (true) {
      try {
        return executeAndDecode(template);
      } catch (RetryableException e) {
        retryer.continueOrPropagate(e);
        if (logLevel != Logger.Level.NONE) {
          logger.logRetry(metadata.configKey(), logLevel);
        }
        continue;
      }
    }
  }
```
首先创建请求模板RequestTemplate，先获取解析过的参数索引和参数名的映射关系，比如0->str，然后根据索引获取真实的参数值，即传给方法的参数值，这样就能获取映射的参数值，比如str->hello，然后在RequestTemplate#expand()方法完成变量的替换，这样就能知道真正的请求路径，然后在请求路径前拼接以http开头的微服务名
```java
    public RequestTemplate create(Object[] argv) {
      RequestTemplate mutable = new RequestTemplate(metadata.template());
      if (metadata.urlIndex() != null) {
        int urlIndex = metadata.urlIndex();
        checkArgument(argv[urlIndex] != null, "URI parameter %s was null", urlIndex);
        mutable.insert(0, String.valueOf(argv[urlIndex]));
      }
      Map<String, Object> varBuilder = new LinkedHashMap<String, Object>();
      for (Entry<Integer, Collection<String>> entry : metadata.indexToName().entrySet()) {
        int i = entry.getKey();
        Object value = argv[entry.getKey()];
        if (value != null) { // Null values are skipped.
          if (indexToExpander.containsKey(i)) {
            value = expandElements(indexToExpander.get(i), value);
          }
          for (String name : entry.getValue()) {
            varBuilder.put(name, value);
          }
        }
      }

      RequestTemplate template = resolve(argv, mutable, varBuilder);
      if (metadata.queryMapIndex() != null) {
        // add query map parameters after initial resolve so that they take
        // precedence over any predefined values
        template = addQueryMapQueryParameters((Map<String, Object>) argv[metadata.queryMapIndex()], template);
      }

      if (metadata.headerMapIndex() != null) {
        template = addHeaderMapHeaders((Map<String, Object>) argv[metadata.headerMapIndex()], template);
      }

      return template;
    }
```
然后是请求的执行了，在LoadBalancerFeignClient#getClientConfig()通过SpringClientFactory为调用的微服务创建上下文容器，再从容器中获取IClientConfig(LoadBalancerFeignClient$FeignOptionsClientConfig)，再继续获取负载均衡器ILoadBalancer(ZoneAwareLoadBalancer)。
```java
	public Response execute(Request request, Request.Options options) throws IOException {
		try {
			URI asUri = URI.create(request.url());
			String clientName = asUri.getHost();
			URI uriWithoutHost = cleanUrl(request.url(), clientName);
			FeignLoadBalancer.RibbonRequest ribbonRequest = new FeignLoadBalancer.RibbonRequest(
					this.delegate, request, uriWithoutHost);

			IClientConfig requestConfig = getClientConfig(options, clientName);
			return lbClient(clientName).executeWithLoadBalancer(ribbonRequest,
					requestConfig).toResponse();
		}
		catch (ClientException e) {
			IOException io = findIOException(e);
			if (io != null) {
				throw io;
			}
			throw new RuntimeException(e);
		}
	}
```
最后就是请求的执行了，在LoadBalancerCommand#submit()实现，这里会通过selectServer()找到负载均衡器，再使用负载均衡器ILoadBalancer#chooseServer()获取要调用的微服务；然后在LoadBalancerCommand#submit()的参数lamdba表达式通过LoadBalancerContext#reconstructURIWithServer()获取ip地址，再替换调用的url的微服务名，往下就是Client.Default#execute()发送请求了，本质还是HttpURLConnection。
```java
    public Response execute(Request request, Options options) throws IOException {
      HttpURLConnection connection = convertAndSend(request, options);
      return convertResponse(connection).toBuilder().request(request).build();
    }
```
**总结下，使用feign作为客户端调用微服务时，首先创建请求模板，即封装请求的路径，参数等信息的RequestTemplate对象，再为微服务创建应用上下文，从应用上下文中获取负载均衡器，通过负载均衡器找到要调用的微服务，完成url中ip地址的替换，使用HttpURLConnection进行http接口的调用。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**