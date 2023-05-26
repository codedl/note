springmvc是支持异步处理请求，即在处理请求期间保存request和response是打开的状态，可以在其他线程中读取和写入数据，做法是创建名为`TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME`，类型为AsyncTaskExecutor的Bean即可。
```java
    @Bean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)
    public AsyncTaskExecutor asyncTaskExecutor(){
        return new SimpleAsyncTaskExecutor("mvc async:");
    }
```
其原理就是在WebMvcAutoConfiguration$WebMvcAutoConfigurationAdapter中会进行配置，逻辑就是先检查容器中是否配置了异步线程池Bean，可以根据配置项`spring.mvc.async.request-timeout`设置异步处理请求的超时时间。
```java
		@Override
		public void configureAsyncSupport(AsyncSupportConfigurer configurer) {
			if (this.beanFactory.containsBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME)) {
				Object taskExecutor = this.beanFactory
						.getBean(TaskExecutionAutoConfiguration.APPLICATION_TASK_EXECUTOR_BEAN_NAME);
				if (taskExecutor instanceof AsyncTaskExecutor) {
					configurer.setTaskExecutor(((AsyncTaskExecutor) taskExecutor));
				}
			}
			Duration timeout = this.mvcProperties.getAsync().getRequestTimeout();
			if (timeout != null) {
				configurer.setDefaultTimeout(timeout.toMillis());
			}
		}
```
还有一种方法是通过Servlet3.0规范实现的，注意要在主类添加注解`@ServletComponentScan("com.example.springsource.nonblocking")`对过滤器进行扫描。
```java
@WebServlet(name = "ServerServlet", urlPatterns = {"/server"}, asyncSupported = true)
public class ServerServlet extends HttpServlet {

    protected void processRequest(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        System.out.println(Thread.currentThread().getName());
        response.setContentType("text/html;charset=UTF-8");
        // async write
        final AsyncContext context = request.startAsync();
        final ServletOutputStream output = response.getOutputStream();
        output.setWriteListener(new WriteListenerImpl(output, context));
    }

    @Override
    protected void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }

    @Override
    protected void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }
}
```
```java
public class WriteListenerImpl implements WriteListener {

    private ServletOutputStream output;
    private AsyncContext context;

    public WriteListenerImpl(ServletOutputStream output, AsyncContext context) {
        this.context = context;
        this.output = output;
    }

    /**
     * do when the data is available to be written
     */
    @Override
    public void onWritePossible() throws IOException {

        if (output.isReady()) {
            output.println("<p>Server is sending back 5 hello...</p>");
            output.flush();
        }

        for (int i = 1; i <= 5 && output.isReady(); i++) {
            output.println("<p>Hello " + i + Thread.currentThread().getName() + ".</p>");
            output.println("<p>Sleep 3 seconds simulating data blocking.<p>");
            output.flush();

            // sleep on purpose
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                // ignore
            }
        }
        output.println("<p>Sending completes.</p>");
        output.flush();
        context.complete();
    }

    /**
     * do when error occurs.
     */
    @Override
    public void onError(Throwable t) {
        context.complete();
        t.printStackTrace();
    }

}
```
此时就可以保持与客户端的会话连接而不阻塞的进行交互，将服务端数据非阻塞地返回给客户端。