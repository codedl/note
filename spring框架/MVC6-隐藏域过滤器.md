---
title: MVC5-隐藏域过滤器
tags:
notebook: spring源码解析
---
1. 在`WebMvcAutoConfiguration`类中对hiddenHttpMethodFilter()方法使用@Bean注解，spring会调用此方法创建OrderedHiddenHttpMethodFilter的实例对象
2. 在处理请求时会调用父类`HiddenHttpMethodFilter.java`中的方法`doFilterInternal()`对请求进行过滤
   1. 判断当前请求是否为post请求，且请求中没有异常
   2. 从请求中获取对象的属性methodParam对应的key`"_method"`的值
   3. 如果支持转换，即为PUT、DELETE、或PATCH请求，则对请求进行封装
   4. 返回封装后的请求交由后续处理

```
protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
throws ServletException, IOException {
    HttpServletRequest requestToUse = request;

    if ("POST".equals(request.getMethod()) && request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) == null) {
        String paramValue = request.getParameter(this.methodParam);
        if (StringUtils.hasLength(paramValue)) {
            String method = paramValue.toUpperCase(Locale.ENGLISH);
            if (ALLOWED_METHODS.contains(method)) {
                requestToUse = new HttpMethodRequestWrapper(request, method);
            }
        }
    }

    filterChain.doFilter(requestToUse, response);
}
public static final String DEFAULT_METHOD_PARAM = "_method";

private String methodParam = DEFAULT_METHOD_PARAM;
private static final List<String> ALLOWED_METHODS =
        Collections.unmodifiableList(Arrays.asList(HttpMethod.PUT.name(),
                HttpMethod.DELETE.name(), HttpMethod.PATCH.name()));
```
