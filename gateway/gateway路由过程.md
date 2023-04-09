gateway是在DispatcherHandler.java类的handle()方法中对请求进行处理。这个方法其实就是对handlerMappings`private List<HandlerMapping> handlerMappings;`集合进行迭代，调用每个mapping的getHandler()方法获取handler。
```java
	public Mono<Void> handle(ServerWebExchange exchange) {
		if (logger.isDebugEnabled()) {
			ServerHttpRequest request = exchange.getRequest();
			logger.debug("Processing " + request.getMethodValue() + " request for [" + request.getURI() + "]");
		}
		if (this.handlerMappings == null) {
			return Mono.error(HANDLER_NOT_FOUND_EXCEPTION);
		}
		return Flux.fromIterable(this.handlerMappings)
				.concatMap(mapping -> mapping.getHandler(exchange))
				.next()
				.switchIfEmpty(Mono.error(HANDLER_NOT_FOUND_EXCEPTION))
				.flatMap(handler -> invokeHandler(exchange, handler))
				.flatMap(result -> handleResult(exchange, result));
	}
```
gateway中会用到的HandlerMapping为RoutePredicateHandlerMapping，因此这里就是调用RoutePredicateHandlerMapping中的getHandlerInternal()方法获取FilteringWebHandler。
```java
	protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
		exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getClass().getSimpleName());

		return lookupRoute(exchange)
				// .log("route-predicate-handler-mapping", Level.FINER) //name this
				.flatMap((Function<Route, Mono<?>>) r -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isDebugEnabled()) {
						logger.debug("Mapping [" + getExchangeDesc(exchange) + "] to " + r);
					}

					exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
					return Mono.just(webHandler);
				}).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
					exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
					if (logger.isTraceEnabled()) {
						logger.trace("No RouteDefinition found for [" + getExchangeDesc(exchange) + "]");
					}
				})));
	}
```
而在getHandlerInternal()会通过lookupRoute()会获取所有路由，再使用每个路由的断言判断是否可以应用到当前的请求上，选择一个可以使用的路由。
```java
	protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
		return this.routeLocator.getRoutes()
				.filterWhen(route ->  {
					// add the current route we are testing
					exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, route.getId());
					return route.getPredicate().apply(exchange);
				})
				// .defaultIfEmpty() put a static Route not found
				// or .switchIfEmpty()
				// .switchIfEmpty(Mono.<Route>empty().log("noroute"))
				.next()
				//TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("Route matched: " + route.getId());
					}
					validateRoute(route, exchange);
					return route;
				});

		/* TODO: trace logging
			if (logger.isTraceEnabled()) {
				logger.trace("RouteDefinition did not match: " + routeDefinition.getId());
			}*/
	}
```
路由的生成是在PropertiesRouteDefinitionLocator和InMemoryRouteDefinitionRepository对象中创建的。
PropertiesRouteDefinitionLocator获取的是GatewayProperties属性源中的routes集合中所有的RouteDefinition。RouteDefinition类中的属性跟配置文件中的配置项是对应的，也就是我们在yaml文件配置的路由的各项的值，这样的话我们每添加一个路由的配置就会生成一个RouteDefinition，被保存在GatewayProperties属性源的routes集合中`private List<RouteDefinition> routes = new ArrayList<>();`;
```java
public class RouteDefinition {
	@NotEmpty
	private String id = UUID.randomUUID().toString();

	@NotEmpty
	@Valid
	private List<PredicateDefinition> predicates = new ArrayList<>();

	@Valid
	private List<FilterDefinition> filters = new ArrayList<>();

	@NotNull
	private URI uri;

	private int order = 0;
    ......
} 
``` 
获取到RouteDefinition后会在RouteDefinitionRouteLocator中的getRoutes()方法中将每个路由定义RouteDefinition转化成Route，这样我们就得到Route路由了。
```java
	public Flux<Route> getRoutes() {
		return this.routeDefinitionLocator.getRouteDefinitions()
				.map(this::convertToRoute)
				//TODO: error handling
				.map(route -> {
					if (logger.isDebugEnabled()) {
						logger.debug("RouteDefinition matched: " + route.getId());
					}
					return route;
				});


		/* TODO: trace logging
			if (logger.isTraceEnabled()) {
				logger.trace("RouteDefinition did not match: " + routeDefinition.getId());
			}*/
	}

    private Route convertToRoute(RouteDefinition routeDefinition) {
		AsyncPredicate<ServerWebExchange> predicate = combinePredicates(routeDefinition);
		List<GatewayFilter> gatewayFilters = getFilters(routeDefinition);

		return Route.async(routeDefinition)
				.asyncPredicate(predicate)
				.replaceFilters(gatewayFilters)
				.build();
	}
```
拿到所有的Route路由之后，我们就可以获取断言判断能否应用到当前的请求`route.getPredicate().apply(exchange);`，从而找出一个可用的路由，再将获取到的路由保存到请求的属性GATEWAY_ROUTE_ATTR("gatewayRoute")中。

接下来就是FilteringWebHandler.java类中的handle()方法，先获取当前路由定义的过滤器，再获取系统中的全局过滤器，然后组合排序，再依次调用每个过滤器的filter()方法对请求进行过滤处理。
```java
	public Mono<Void> handle(ServerWebExchange exchange) {
		Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
		List<GatewayFilter> gatewayFilters = route.getFilters();

		List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
		combined.addAll(gatewayFilters);
		//TODO: needed or cached?
		AnnotationAwareOrderComparator.sort(combined);

		logger.debug("Sorted gatewayFilterFactories: "+ combined);

		return new DefaultGatewayFilterChain(combined).filter(exchange);
	}
```
我们看下LoadBalancerClientFilter过滤器，这个过滤器可以实现微服务的调用，当发现调用的服务是以"lb"开头时，就会通过Ribbon负载均衡器获取相应的微服务地址进行接口调用。再往下就是http的请求调用了。
```java
	public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
		URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
		String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
		if (url == null || (!"lb".equals(url.getScheme()) && !"lb".equals(schemePrefix))) {
			return chain.filter(exchange);
		}
		//preserve the original url
		addOriginalRequestUrl(exchange, url);

		log.trace("LoadBalancerClientFilter url before: " + url);

		final ServiceInstance instance = loadBalancer.choose(url.getHost());

		if (instance == null) {
			throw new NotFoundException("Unable to find instance for " + url.getHost());
		}

		URI uri = exchange.getRequest().getURI();

		// if the `lb:<scheme>` mechanism was used, use `<scheme>` as the default,
		// if the loadbalancer doesn't provide one.
		String overrideScheme = null;
		if (schemePrefix != null) {
			overrideScheme = url.getScheme();
		}

		URI requestUrl = loadBalancer.reconstructURI(new DelegatingServiceInstance(instance, overrideScheme), uri);

		log.trace("LoadBalancerClientFilter url chosen: " + requestUrl);
		exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
		return chain.filter(exchange);
	}
```
**总结下，首先spring在启动时会根据配置文件创建路由的规则，当gateway收到请求，就会通过断言判断路由的规则是否能应用到当前的请求，然后将请求转发给路由中的uri，如果是微服务的话就会通过负载均衡动态的获取ip，之后就是过滤器对请求和响应进行处理。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**