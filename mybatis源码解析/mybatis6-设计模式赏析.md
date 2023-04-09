mybatis源码中用到了很多设计模式，接下来就对这些设计模式分析，希望以后可以运用到自己的工作中。
+ 单例模式

mybatis有个全局唯一的Configuration对象，生成Configuration对象之后就是全局唯一存在的。解析xml文件，创建mapper接口的代理对象，执行sql，用的都是这个对象。

+ 工厂模式

mybatis声明了SqlSessionFactory用来创建SqlSession，不同的厂商可以有不同的实现，像MySQL、Oracle等，只要将封装了厂商驱动的DataSource传给Configuration的Environment对象，就可以使用包含Configuration的SqlSessionFactory的实现类对象创建SqlSession，这样的话对不同的厂商可以创建统一的SqlSession，而不用关心过程。

+ 建造者模式

mybatis在声明了抽象父类建造者BaseBuilder，子类继承了抽象父类，对具体的建造过程给出了不同的实现。XMLStatementBuilder负责对包含具体的要执行的sql的标签进行解析，创建MappedStatement对象，用来对标签进行封装，包含了标签中定义的属性；XMLScriptBuilder用来生成要执行的sql字符串，对字符串中的参数进行处理，最终用SqlSource对象表示；XMLConfigBuilder用来创建最重要的Configuration对象，是mybatis用来解析xml配置文件的工具。可以看出不同的建造者功能单一，只会创建一种对象，这样在代码上就实现了边界隔离。如果发现哪块需要优化，只需要修改这个产物对应的模块就行，便于维护。

+ 适配器模式

mybatis正是通过适配器模式实现日志打印的，现在的日志框架很多，但是mybatis把这些框架巧妙的整合起来了。首先声明了org.apache.ibatis.logging.Log接口，在org.apache.ibatis.logging.Log接口的实现类中注入其他框架的入口对象，比如Slf4j框架，使用Slf4jLoggerImpl进行适配，Slf4jLoggerImpl类包含org.slf4j.Logger接口，这样就可以调用org.slf4j.Logger接口的方法实现日志功能；而Slf4jLoggerImpl实现了org.apache.ibatis.logging.Log接口，实现这个方法时正是通过调用org.slf4j.Logger接口的方法，这样就可以通过调用mybatis声明的org.apache.ibatis.logging.Log接口进行日志打印；Log4j2LoggerImpl适配器也是类似，最终可以通过统一的Log接口打印日志。使用的时候只要在配置文件中声明的适配器类，再调用工厂方法LogFactory#getLog就能构造日志适配器对象进行日志输出了。这种设计真的很棒，使用统一的接口适配不同的厂商，这样就可以实现良好的扩展性，在不修改代码的情况对原有的功能进行扩展。
```java
class Slf4jLoggerImpl implements Log {

  private final Logger log;

  public Slf4jLoggerImpl(Logger logger) {
    log = logger;
  }

  @Override
  public boolean isDebugEnabled() {
    return log.isDebugEnabled();
  }
......  

}
```
+ 代理模式

mybatis中可以对接口使用@Mapper注解，此时便会为接口动态代理，之后通过接口调用方法时，其实调用的就是代理对象的方法。看下整个过程吧。在解析xml文件时会调用MapperRegistry#addMapper方法会设置接口的代理工厂为MapperProxyFactory。
```java
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      ......
      try {
        knownMappers.put(type, new MapperProxyFactory<T>(type));
        ......
      } ......
  }
```
调用接口的方法时通过`SqlSession#<T> T getMapper(Class<T> type);`获取接口的代理对象，此时会通过jdk动态代理创建接口的代理对象，注意代理对象实现了要代理的接口，并且包含实现了拦截接口的InvocationHandler的MapperProxy，调用代理对象的方法时就会进行拦截，即MapperProxy代理对象的invoke()方法调用。
```java
  public <T> T getMapper(Class<T> type, SqlSession sqlSession) {
    final MapperProxyFactory<T> mapperProxyFactory = (MapperProxyFactory<T>) knownMappers.get(type);
    ......
  }
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<T>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
  protected T newInstance(MapperProxy<T> mapperProxy) {
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
```
可见代理模式使用之后并不是直接访问原始对象，而是先进行一定的预处理，再调用原始对象的方法，这样就可以控制对原始对象的访问和引用。

+ 组合模式

组合模式的三个要素，第一个是接口为SqlNode，第二个为叶子TextSqlNode或StaticTextSqlNode，第三个为容器MixedSqlNode，其中TextSqlNode、StaticTextSqlNode和MixedSqlNode都实现了SqlNode接口。
```java
public interface SqlNode {
  boolean apply(DynamicContext context);
}
```
mybatis对xml文件解析时就是使用组合模式实现的，先得到容器，再通过容器实现的apply()生成SqlSource。
```java
  public SqlSource parseScriptNode() {
    MixedSqlNode rootSqlNode = parseDynamicTags(context);
    SqlSource sqlSource = null;
    if (isDynamic) {
      sqlSource = new DynamicSqlSource(configuration, rootSqlNode);
    } else {
      sqlSource = new RawSqlSource(configuration, rootSqlNode, parameterType);
    }
    return sqlSource;
  }
```
根据节点是否为动态节点(是否包含"$()")创建TextSqlNode或StaticTextSqlNode，再添加到容器MixedSqlNode。
```java
  protected MixedSqlNode parseDynamicTags(XNode node) {
    List<SqlNode> contents = new ArrayList<SqlNode>();
    NodeList children = node.getNode().getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
      XNode child = node.newXNode(children.item(i));
      //是否对sql标签进行解析
      if (child.getNode().getNodeType() == Node.CDATA_SECTION_NODE || child.getNode().getNodeType() == Node.TEXT_NODE) {
        String data = child.getStringBody("");
        TextSqlNode textSqlNode = new TextSqlNode(data);
        if (textSqlNode.isDynamic()) {
          contents.add(textSqlNode);
          isDynamic = true;
        } else {
          contents.add(new StaticTextSqlNode(data));
        }
        //是否对sql中包含的标签进行解析
      } else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
        String nodeName = child.getNode().getNodeName();
        NodeHandler handler = nodeHandlerMap.get(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        //处理这些标签然后添加到contents
        handler.handleNode(child, contents);
        isDynamic = true;
      }
    }
    return new MixedSqlNode(contents);
  }
```
最后把容器当成普通的叶子节点调用apply()方法，此时就会调用容器对象的apply()，而容器对象又会遍历内部的List<SqlNode>依次调用每个SqlNode的apply()方法。
```java
  private static String getSql(Configuration configuration, SqlNode rootSqlNode) {
    DynamicContext context = new DynamicContext(configuration, null);
    rootSqlNode.apply(context);
    return context.getSql();
  }
```
以上就是mybatis对组合模式的使用，组合模式又名部分整体模式，用于树形结构的对象，可以把容器和叶子当成普通的对象进行处理，即可以使用更普通的方式处理复杂对象，按我理解就是把复杂对象解耦成一些简单对象。

+ 装饰器模式
在创建Executor执行器对象执行sql时，mybatis正是使用装饰器模式：如果需要使用全局缓存，则使用CachingExecutor对SimpleExecutor进行装饰。
```java
  public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```
在CachingExecutor#query()方法中，先获取Cache缓存对象，如果使用缓存的话就到全局缓存根据key获取查询结果，没有的话就使用原有的SimpleExecutor对象进行查询，即将从缓存获取查询结果加入到sql的执行过程中。
```java
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          list = delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.<E> query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```
如果有新的需求，还可以在原有的基础上继续装饰，便于扩展了。这里也能看到装饰器模式与代理模式的区别，装饰器模式会装饰对象原有的行为，新增的代码和对象原有的行为有关，属于功能上的增强；代理模式则用于控制对象的引用和访问，会在执行方法前后增加代码，但是这些代码不会改变对象原有的行为。比如，某个租客通过中介找到房主租房子，中介就是房主的代理，租客正是通过代理租的房子，用的就是代理模式；如果中介找了间商铺给租客了，对租客而言既能住又能做生意了，就相当于增强了租客租房子的行为了，就是装饰器模式。

+ 模板模式

模板模式也是一种常用的设计模式，工作中常常用到。对于一个通用的流程，大部分都一样，而只有某个步骤需要分类，就可以交给子类实现，而父类实现的是个通用的流程。Java中模板模式的原理是，jdk调用方法时总是从最底层的类开始根据方法签名匹配方法，如果找不到就到父类中寻找。mybatis首先在BaseExecutor#queryFromDatabase()方法中定义标准的查询过程，而把doQuery()交由子类比如SimpleExecutor实现，这样不同的子类就可以有不同的处理，而查询过程都是相同的，先查询再缓存，注意这里是缓存到会话缓存，因为SimpleExecutor只有在创建SqlSession时才存在。
```java
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);
    }
    localCache.putObject(key, list);
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
  protected abstract <E> List<E> doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql)
      throws SQLException;
```

+ 策略模式

策略模式就是说同一个对象在不同的场景下具有不同的行为，说得直白点就是同一个接口在不同的条件下对应不同的实现类对象，例如mybaits中解析xml标签时的使用，对不同的标签进行解析时使用不同的NodeHandler，实现不同的策略。
```java
// 第一步声明一个用于存储节点对应NodeHandler的HashMap
  private final Map<String, NodeHandler> nodeHandlerMap = new HashMap<String, NodeHandler>();
// 第二步将多个标签对应的NodeHandler保存下来
  private void initNodeHandlerMap() {
    nodeHandlerMap.put("trim", new TrimHandler());
    nodeHandlerMap.put("where", new WhereHandler());
    nodeHandlerMap.put("set", new SetHandler());
    nodeHandlerMap.put("foreach", new ForEachHandler());
    nodeHandlerMap.put("if", new IfHandler());
    nodeHandlerMap.put("choose", new ChooseHandler());
    nodeHandlerMap.put("when", new IfHandler());
    nodeHandlerMap.put("otherwise", new OtherwiseHandler());
    nodeHandlerMap.put("bind", new BindHandler());
  }
// 第三步根据节点获取不同的NodeHandler，进行不同的处理  
  protected MixedSqlNode parseDynamicTags(XNode node) {
    ...... else if (child.getNode().getNodeType() == Node.ELEMENT_NODE) { // issue #628
        String nodeName = child.getNode().getNodeName();
        NodeHandler handler = nodeHandlerMap.get(nodeName);
        if (handler == null) {
          throw new BuilderException("Unknown element <" + nodeName + "> in SQL statement.");
        }
        handler.handleNode(child, contents);
        isDynamic = true;
      }
    }
    return new MixedSqlNode(contents);
  }
```

+ 迭代器模式

mybatis在进行参数处理时使用了迭代器模式，首先声明实现了Iterator接口的类PropertyTokenizer，对sql中的动态参数如#{phone.callPhone}处理时，会对PropertyTokenizer进行实例化。
```java
public class PropertyTokenizer implements Iterator<PropertyTokenizer> {
  private String name;
  private final String indexedName;
  private String index;
  private final String children;

  public PropertyTokenizer(String fullname) {
    int delim = fullname.indexOf('.');
    if (delim > -1) {
      name = fullname.substring(0, delim);
      children = fullname.substring(delim + 1);
    } else {
      name = fullname;
      children = null;
    }
    indexedName = name;
    delim = name.indexOf('[');
    if (delim > -1) {
      index = name.substring(delim + 1, name.length() - 1);
      name = name.substring(0, delim);
    }
  }
  @Override
  public boolean hasNext() {
    return children != null;
  }
}
```  
参数处理的逻辑是在MetaObject#getValue()方法中，本质是对"#{phone.callPhone}"中的字符串进行迭代，即先获取phone对应的Java对象CallPhone，再获取对象中的callPhone属性的值，不过这里的迭代是以递归的形式实现的，可能不同于我们平时的`while(Iterator.hasNext()){...}`。从这里可能看到迭代器模式处理聚合对象时，是以一种更加通用的方式处理聚合对象包含的对象数据，做到对数据的处理与对象数据的类型无关，这样就能处理任意一种对象了。
```java
  public Object getValue(String name) {
    PropertyTokenizer prop = new PropertyTokenizer(name);
    if (prop.hasNext()) {
      MetaObject metaValue = metaObjectForProperty(prop.getIndexedName());
      if (metaValue == SystemMetaObject.NULL_META_OBJECT) {
        return null;
      } else {
        return metaValue.getValue(prop.getChildren());
      }
    } else {
      return objectWrapper.get(prop);
    }
  }
```
学废了学废了，mybatis使用了众多的设计模式，还需要时间慢慢消化。