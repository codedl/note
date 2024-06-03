上一节说到请求url定位servlet的过程，tomcat会把请求url和容器的映射关系保存到MappingData中，org.apache.catalina.connector.Request类实现了HttpServletRequest，其中定义了属性mappingData`protected final MappingData mappingData = new MappingData();`用于保存映射关系，其是在`CoyoteAdapter.java#service()`方法中创建。定位servlet对请求进行处理的入口是`connector.getService().getContainer().getPipeline().getFirst().invoke(request, response);`，就从这里往后面看。这里的逻辑就是调用Pipeline中的每个Valve进行处理。
+ StandardEngineValve  
这是Engine的Valve，就是找到前面通过请求url定位的Host，获取Host的Valve进行处理。
```java
    public void invoke(Request request, Response response) throws IOException, ServletException {
        // Select the Host to be used for this Request
        Host host = request.getHost();
        ......
        host.getPipeline().getFirst().invoke(request, response);
    }

    public Host getHost() {
        return mappingData.host;
    }
}
```
+ StandardHostValve   
这个Valve也是跟Engine的Valve类似，沿着链表向后移动，找到下一个Valve进行处理。
+ StandardContextValve  
从请求中获取StandardWrapperValve进行请求的处理。
+ StandardWrapperValve  
前面铺垫了这么久，总算到正题了，请求的正式处理就是在StandardWrapperValve中进行的，前面的Valve做的还是对进行加工，没有进行正式的处理。怎么样，是不是看到熟悉的代码段，平时我们所说的过滤器链对请求过滤处理的代码!这块就是挨个调用每个过滤器Filter上的doFilter()方法进行处理。
```java
    public void invoke(Request request, Response response) throws IOException, ServletException {
    ......
        Servlet servlet = null;
        ......
            servlet = wrapper.allocate();
        ApplicationFilterChain filterChain = ApplicationFilterFactory.createFilterChain(request, wrapper, servlet);
        ......
                        filterChain.doFilter(request.getRequest(), response.getResponse());
                        ......        
    }
```
还是慢慢看吧，首先在Wrapper容器中取出Servlet，然后创建用来处理请求的servlet过滤器链。在调用每个过滤器的doFilter方法时，会通过HttpServlet中的模板方法`internalDoFilter()`调用内置的`service`方法，就是根据请求方式处理请求啦，也就是我们写的过滤器中处理请求的方法。
```java
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {

        String method = req.getMethod();

        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                // servlet doesn't support if-modified-since, no reason
                // to go through further expensive logic
                doGet(req, resp);
            } else {
                long ifModifiedSince;
                try {
                    ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
                } catch (IllegalArgumentException iae) {
                    // Invalid date header - proceed as if none was set
                    ifModifiedSince = -1;
                }
                if (ifModifiedSince < (lastModified / 1000 * 1000)) {
                    // If the servlet mod time is later, call doGet()
                    // Round down to the nearest second for a proper compare
                    // A ifModifiedSince of -1 will always be less
                    maybeSetLastModified(resp, lastModified);
                    doGet(req, resp);
                } else {
                    resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
                }
            }

        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);

        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);

        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);

        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);

        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req, resp);

        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req, resp);

        } else {
            //
            // Note that this means NO servlet supports whatever
            // method was requested, anywhere on this server.
            //

            String errMsg = lStrings.getString("http.method_not_implemented");
            Object[] errArgs = new Object[1];
            errArgs[0] = method;
            errMsg = MessageFormat.format(errMsg, errArgs);

            resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
        }
    }
```
