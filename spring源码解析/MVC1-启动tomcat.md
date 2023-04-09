---
title: MVC1-启动tomcat
tags:
notebook: spring源码解析
---
+ 注册相关组件定义
1. 在springboot jar包中的spring.factories配置文件中通过`org.springframework.boot.autoconfigure.EnableAutoConfiguration`配置项，配置了要自动加载的组件 ServletWebServerFactoryAutoConfiguration，
在 ServletWebServerFactoryAutoConfiguration 中会通过@Import ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar 组件和 ServletWebServerFactoryConfiguration.EmbeddedTomcat,
默认情况下spring会导入EmbeddedTomcat组件。
2. ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar 组件会向容器注册注册WebServerFactoryCustomizerBeanPostProcessor和ErrorPageRegistrarBeanPostProcessor两个处理器
3. EmbeddedTomcat组件内部使用@Bean注解声明了创建TomcatServletWebServerFactory组件的工厂方法tomcatServletWebServerFactory()
+ 创建tomcat实例 
在通过run方法运行spring应用时，会在ioc容器刷新完成后调用onRefresh方法通知子类，此时会调用createWebServer创建web服务器
1. 通过 TomcatServletWebServerFactory 组件的工厂方法生成bean
2. 通过 getWebServer() 方法创建web服务器
   1. 生成 Tomcat 实例
   2. 为 Tomcat 实例设置各种属性
   3. 生成`TomcatWebServer`实例
+ 启动tomcat服务器 
+ 通过`TomcatWebServer.java`类中的`start()`方法启动tomcat服务器
1. 通过`NioEndpoint.java`类中的`initServerSocket()`方法来初始化socket
   1. 通过`ServerSocketChannel.open()`获取ServerSocketChannel
   2. 可以为ServerSocket设置属性
   3. 获取ip和端口，生成`InetSocketAddress`
   4. 将socket和ip端口绑定`serverSock.socket().bind(addr,getAcceptCount());`
   5. 之后的Acceptor会通过serverSock.accept()监听新的连接
+ 通过NioEndpoint.java类中的startInternal方法来启动线程监听连接和socket
2. 通过`AbstractEndpoint.java`类中的`startAcceptorThread()`方法监听连接
   1. 启动线程时调用Acceptor类中run()方法
   2. 在run方法中会调用`serverSock.accept();`阻塞地等待新连接的到来
   3. 当有新的连接进来时会调用`endpoint.setSocketOptions(socket))`进行处理
      1. 先到stack中去获取预先分配的NioChannel`private SynchronizedStack<NioChannel> nioChannels;`，如果没有说明已经用完，则新建一个NioChannel，NioChannel会包含读缓冲区和写缓冲区
      2. 将NioChannel和NioEndpoint封装成NioSocketWrapper
      3. 对NioChannel进行复位reset，将SocketChannel和NioSocketWrapper设置到NioChannel中，将NioChannel中的读、写缓冲区清掉
      4. 将socket设置成非阻塞
      5. 设置NioSocketWrapper的读超时和写超时
      6. 注册到 Poller
         1. 给通道设置读事件监听SelectionKey.OP_READ
         2. 从缓存中获取PollerEvent，如果没有说明已经用完，则新建一个PollerEvent，并且标记当前的PollerEvent为OP_REGISTER注册事件
         3. 将PollerEvent添加到同步队列中`private final SynchronizedQueue<PollerEvent> events = new SynchronizedQueue<>();`
3. 通过`NioEndpoint.java`类中的[`startInternal()`](#startinternal)方法来开启线程对socket进行监听
   1. Poller实例化，Poller为Runnable接口的实现类，重写了run方法
   2. 创建线程，运行run方法
   3. 对socket进行监听
      1. 通过events()方法获取socket监听事件
         1. 首先对同步队列进行遍历，获取队列中保存的PollerEvent实例对象
         2. [从PollerEvent获取NioChannel](#SocketChannel到PollerEvent的过程)
         3. 从NioChannel获取NioSocketWrapper
         4. 处理PollerEvent事件
            1. 如果是OP_REGISTER注册事件
               1. 将保存在NioChannel中的 SocketChannel 以及读事件SelectionKey.OP_READ注册到selector，同时绑定NioSocketWrapper`channel.getIOChannel().register(getSelector(), SelectionKey.OP_READ, socketWrapper);`
            2. 如果是其他事件
               1. 先从selector中获取监听的事件SelectionKey`final SelectionKey key = channel.getIOChannel().keyFor(getSelector());`
               2. 获取绑定的NioSocketWrapper
               3. 将获取的通道监听事件和PollerEvent事件合并`int ops = key.interestOps() | interestOps;`，然后设置到监听事件中`key.interestOps(ops);`
            3. PollerEvent事件处理完成，就将PollerEvent复位，缓存到同步栈，留作下个连接使用
      2. 在每次添加PollerEvent事件到同步队列中，AtomicLong型变量wakeupCounter的值都会被incrementAndGet，现在wakeupCounter.getAndSet(-1)获取并清空这个值，通过获取的值判断是否存在PollerEvent事件
      3. 如果有立即调用selectNow()获取，否则调用select()阻塞1s判断是否存在监听事件
      4. 通过`selector.selectedKeys().iterator()`获取监听到的事件
      5. 通过processKey处理监听的事件`processKey(sk, socketWrapper);`

```java
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
		ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
		ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
		ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration{}

@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
static class EmbeddedTomcat {}
```
```java
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
@ConditionalOnMissingBean(value = ServletWebServerFactory.class, search = SearchStrategy.CURRENT)
static class EmbeddedTomcat {
    @Bean
    TomcatServletWebServerFactory tomcatServletWebServerFactory(
            ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
            ObjectProvider<TomcatContextCustomizer> contextCustomizers,
            ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
        TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
        factory.getTomcatConnectorCustomizers()
                .addAll(connectorCustomizers.orderedStream().collect(Collectors.toList()));
        factory.getTomcatContextCustomizers()
                .addAll(contextCustomizers.orderedStream().collect(Collectors.toList()));
        factory.getTomcatProtocolHandlerCustomizers()
                .addAll(protocolHandlerCustomizers.orderedStream().collect(Collectors.toList()));
        return factory;
    }

}
```
```
protected void initServerSocket() throws Exception {
......
      serverSock = ServerSocketChannel.open();
      socketProperties.setProperties(serverSock.socket());
      InetSocketAddress addr = new InetSocketAddress(getAddress(), getPortWithOffset());
      serverSock.socket().bind(addr,getAcceptCount());
......
  serverSock.configureBlocking(true); //mimic APR behavior
}
```
# startInternal
```
public void startInternal() throws Exception {
......
      // Start poller thread
      //public class Poller implements Runnable { void run{}}
      poller = new Poller();
      Thread pollerThread = new Thread(poller, getName() + "-ClientPoller");
      pollerThread.setPriority(threadPriority);
      pollerThread.setDaemon(true);
      pollerThread.start();
......
}
```
# SocketChannel到PollerEvent的过程
```
1.在Acceptor中会同accpet监听新的连接，当有新的连接到来时会获取 SocketChannel
socket = endpoint.serverSocketAccept();
protected SocketChannel serverSocketAccept() throws Exception {
  return serverSock.accept();
}
2.为SocketChannel设置相关属性，先创建NioChannel，在将获取到的SocketChannel保存到NioChannel的sc属性中
endpoint.setSocketOptions(socket)
protected boolean setSocketOptions(SocketChannel socket) {
...... 
      channel = new NioChannel(bufhandler); 
      channel.reset(socket, newWrapper);
...... 
      poller.register(channel, socketWrapper);     
}
public void reset(SocketChannel channel, NioSocketWrapper socketWrapper) throws IOException {
  this.sc = channel;
  this.socketWrapper = socketWrapper;
  bufHandler.reset();
}
3. 实例化一个PollerEvent，将含有SocketChannel的NioChannel保存到PollerEvent的socket属性中
poller.register(channel, socketWrapper);
public void register(final NioChannel socket, final NioSocketWrapper socketWrapper) {
   socketWrapper.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
   PollerEvent event = null;
   if (eventCache != null) {
       event = eventCache.pop();
   }
   if (event == null) {
       event = new PollerEvent(socket, OP_REGISTER);
   } else {
       event.reset(socket, OP_REGISTER);
   }
   addEvent(event);
}
public void reset(NioChannel ch, int intOps) {
   socket = ch;
   interestOps = intOps;
}
4. 将封装好的PollerEvent添加到同步队列events中
private void addEvent(PollerEvent event) {
   events.offer(event);
   if (wakeupCounter.incrementAndGet() == 0) {
       selector.wakeup();
   }
}
```
