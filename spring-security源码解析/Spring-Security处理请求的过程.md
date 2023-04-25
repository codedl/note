security框架处理请求是从DelegatingFilterProxy#doFilter()方法开始的
```java
// DelegatingFilterProxy.java
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		// Lazily initialize the delegate if necessary.
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					if (wac == null) {
						throw new IllegalStateException("No WebApplicationContext found: " +
								"no ContextLoaderListener or DispatcherServlet registered?");
					}
					delegateToUse = initDelegate(wac);
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
		invokeDelegate(delegateToUse, request, response, filterChain);
	}
```
其实就是根据注入的targetBeanName("springSecurityFilterChain")获取FilterChainProxy，
```java
// DelegatingFilterProxy.java
	protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		String targetBeanName = getTargetBeanName();
		Assert.state(targetBeanName != null, "No target bean name set");
		Filter delegate = wac.getBean(targetBeanName, Filter.class);
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	}
```
每个请求只能被过滤器过滤一次，使用熟悉FILTER_APPLIED进行标识，然后在doFilterInternal()方法中对请求进行过滤。
```java
// FilterChainProxy.java
	public void doFilter(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
		boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;
		if (clearContext) {
			try {
				request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
				doFilterInternal(request, response, chain);
			}
			finally {
				SecurityContextHolder.clearContext();
				request.removeAttribute(FILTER_APPLIED);
			}
		}
		else {
			doFilterInternal(request, response, chain);
		}
	}
```
先获取过滤器，使用VirtualFilterChain进行封装，再通过doFilter()方法进行过滤。
```java
// FilterChainProxy.java
	private void doFilterInternal(ServletRequest request, ServletResponse response,
			FilterChain chain) throws IOException, ServletException {
......
		List<Filter> filters = getFilters(fwRequest);
		......
		VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
		vfc.doFilter(fwRequest, fwResponse);
	}
```
获取过滤器时就是通过属性filterChains(`private List<SecurityFilterChain> filterChains;`)获取，
```java
	private List<Filter> getFilters(HttpServletRequest request) {
		for (SecurityFilterChain chain : filterChains) {
			if (chain.matches(request)) {
				return chain.getFilters();
			}
		}

		return null;
	}
```
filterChains是在WebSecurity#performBuild()方法中赋值的，默认一共有12个过滤器，用
```java
	protected Filter performBuild() throws Exception {
......
		for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
			securityFilterChains.add(securityFilterChainBuilder.build());
		}
		FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
		......
	}
```
找到过滤器链后，就会执行每个过滤器的doFilter()方法对请求进行过滤，执行完以后就回到tomcat的过滤器链。
```java
		public void doFilter(ServletRequest request, ServletResponse response)
				throws IOException, ServletException {
                    //如果执行完了就会执行tomcat的过滤器
			if (currentPosition == size) {
				if (logger.isDebugEnabled()) {
					logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
							+ " reached end of additional filter chain; proceeding with original chain");
				}

				// Deactivate path stripping as we exit the security filter chain
				this.firewalledRequest.reset();
                //tomcat过滤器链
				originalChain.doFilter(request, response);
			}
			else {
				currentPosition++;

				Filter nextFilter = additionalFilters.get(currentPosition - 1);

				if (logger.isDebugEnabled()) {
					logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
							+ " at position " + currentPosition + " of " + size
							+ " in additional filter chain; firing Filter: '"
							+ nextFilter.getClass().getSimpleName() + "'");
				}

				nextFilter.doFilter(request, response, this);
			}
		}
```
**总结一下，security框架对请求的过滤其实就是调用FilterChainProxy封装的过滤器链中的每个过滤器的doFilter()方法，然后回到tomcat的过滤器链路。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**