对接口使用@Mapper注解后，mybatis会为接口创建代理对象。通过接口调用方法时，本质就是调用InvocationHandler接口的实现类MapperProxy中的invoke()方法，因此执行sql的逻辑都是在这个方法中。
```java
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //判断是否为Object对象的方法调用
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
        //是否为接口中声明的默认方法
      } else if (isDefaultMethod(method)) {
        return invokeDefaultMethod(proxy, method, args);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
    //缓存调用方法
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
```
缓存方法其实就是将调用的方法Method对象和MapperMethod对象的映射保存到ConcurrentHashMap中，那么就需要对MapperMethod实例化，对MapperMethod实例化的时候会在构造方法中实例化SqlCommand和MethodSignature。SqlCommand的name表示方法的全限定路径名，type为执行的sql类型。首先是从Configuration的mappedStatements(StrictMap)中根据方法的全限定名获取MappedStatement对象，MappedStatement对象是对xml中"select|insert|update|delete"标签内容的封装，MappedStatement对象的属性对应的就是标签的属性。
```java
public enum SqlCommandType {
  UNKNOWN, INSERT, UPDATE, DELETE, SELECT, FLUSH;
}
```
接下来对MethodSignature实例化，这个类其实就是对方法签名进行处理，过程就是依次处理方法上的@MapKey注解，RowBounds和ResultHandler类型的参数，参数上的@Param注解。@Param注解被处理时就是获取参数的索引和对应的参数映射名(@Param的值)保存到TreeMap中，而TreeMap会按照参数的索引值从小到大排序。
```java
    public MethodSignature(Configuration configuration, Class<?> mapperInterface, Method method) {
      Type resolvedReturnType = TypeParameterResolver.resolveReturnType(method, mapperInterface);
      if (resolvedReturnType instanceof Class<?>) {
        this.returnType = (Class<?>) resolvedReturnType;
      } else if (resolvedReturnType instanceof ParameterizedType) {
        this.returnType = (Class<?>) ((ParameterizedType) resolvedReturnType).getRawType();
      } else {
        this.returnType = method.getReturnType();
      }
      //是否返回void
      this.returnsVoid = void.class.equals(this.returnType);
      //是否返回集合或数组
      this.returnsMany = configuration.getObjectFactory().isCollection(this.returnType) || this.returnType.isArray();
      this.returnsCursor = Cursor.class.equals(this.returnType);
      //mapKey是作用在方法上的注解@MapKey的值，表示返回的对象的属性名；
      //这样就能根据属性名从对象中获取值作为map的key
      //@MapKey("operateTime") Map<Object, CallPhone> getAll(@Param("id") Integer id);
      //就会从CallPhone中获取属性operateTime的值作为返回的Map中的key
      this.mapKey = getMapKey(method);
      this.returnsMap = this.mapKey != null;
      //RowBounds可以实现分页查询
      this.rowBoundsIndex = getUniqueParamIndex(method, RowBounds.class);
      //ResultHandler可以用来对查询结果进行处理
      this.resultHandlerIndex = getUniqueParamIndex(method, ResultHandler.class);
      //处理@Param
      this.paramNameResolver = new ParamNameResolver(configuration, method);
    }
```
至此，方法的签名有了，xml标签也有了，接下来就是sql的执行逻辑，以select为例去看吧，如果是普通的查询就是先转换参数，再执行sql`executeForMany`。
```java
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    switch (command.getType()) {
      ......
      case SELECT:
        if (method.returnsVoid() && method.hasResultHandler()) {
          executeWithResultHandler(sqlSession, args);
          result = null;
        } else if (method.returnsMany()) {
          result = executeForMany(sqlSession, args);
        } else if (method.returnsMap()) {
          result = executeForMap(sqlSession, args);
        } else if (method.returnsCursor()) {
          result = executeForCursor(sqlSession, args);
        } else {
          Object param = method.convertArgsToSqlCommandParam(args);
          result = sqlSession.selectOne(command.getName(), param);
        }
        break;
      ......
    return result;
  }
```
如果只有一个参数，就直接返回这个参数；有多个参数时，将参数映射名(@Param的值)及参数的值(根据索引获取)保存到ParamMap(HashMap的子类)，并且添加通用的参数名(param1, param2, ...)。
执行sql时是用通过`SqlSessionTemplate.sqlSessionProxy#selectList()`，sqlSessionProxy为SqlSession接口的代理对象，因此执行selectList()时实际调用传入的InvocationHandler的实现类SqlSessionInterceptor中的invoke()方法。
1. 创建会话:在DefaultSqlSessionFactory.java#openSessionFromDataSource()中，首先创建事务，根据事务创建执行器，默认由SimpleExecutor实现，再为执行器对象SimpleExecutor安装拦截器，再用DefaultSqlSession进行封装。如果开启了缓存(mybatis.configuration.cache-enabled=true)则使用CachingExecutor作为执行器。
2. 会话中执行查询方法，也就是到数据库中查询数据
3. 事务提交，关闭会话
```java
  private class SqlSessionInterceptor implements InvocationHandler {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
      SqlSession sqlSession = getSqlSession(
          SqlSessionTemplate.this.sqlSessionFactory,
          SqlSessionTemplate.this.executorType,
          SqlSessionTemplate.this.exceptionTranslator);
      try {
        Object result = method.invoke(sqlSession, args);
        if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
          // force commit even on non-dirty sessions because some databases require
          // a commit/rollback before calling close()
          sqlSession.commit(true);
        }
        return result;
      } catch (Throwable t) {
        Throwable unwrapped = unwrapThrowable(t);
        if (SqlSessionTemplate.this.exceptionTranslator != null && unwrapped instanceof PersistenceException) {
          // release the connection to avoid a deadlock if the translator is no loaded. See issue #22
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
          sqlSession = null;
          Throwable translated = SqlSessionTemplate.this.exceptionTranslator.translateExceptionIfPossible((PersistenceException) unwrapped);
          if (translated != null) {
            unwrapped = translated;
          }
        }
        throw unwrapped;
      } finally {
        if (sqlSession != null) {
          closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
        }
      }
    }
  }
```
看下是怎么添加拦截器的，因为后面执行方法的时候会调用拦截器。
1. 处理@Intercepts注解：获取注解中的@Signature[]，@Signature的type表示拦截的接口，method表示方法，args是方法的参数，这样就可以将type作为key，method对象作为value保存到signatureMap(Map<Class<?>, Set<Method>>)
2. 获取拦截的接口：获取当前类及其父类实现的接口，作为要代理的接口
3. 为拦截的接口创建代理对象：不用说Plugin肯定实现了InvocationHandler接口，这样就会先执行拦截器了
```java
  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      //target为要代理的对象，因为在for中，所以每次都会为上个代理对象生成新的代理对象
      target = interceptor.plugin(target);
    }
    return target;
  }
  public static Object wrap(Object target, Interceptor interceptor) {
    //将拦截的接口和方法保存下来
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    //获取拦截的接口
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    //为接口生成代理对象
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
```
为执行器安装插件后最终得到的是SimpleExecutor的代理对象，而代理对象是为SimpleExecutor实现的接口代理的，因此通过接口调用方法时会被增强，调用Plugin#invoke，最终到拦截器的intercept方法调用。
```java
//proxy为包裹的代理对象；method是接口中声明的方法
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      //判断@Intercepts是否声明了要为调用的方法进行拦截
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      //进入代理对象中包含的对象，如果是代理对象就继续调用Plugin#invoke；否则到执行器对象的方法调用
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
```
拦截器的方法执行完之后，就到数据库查询方法的执行了，是这个`Object result = method.invoke(sqlSession, args);`，这里就是以反射方式调用方法，即调用sqlSession(DefaultSqlSession)中的method(selectList)，DefaultSqlSession#selectList。
+ 不带全局缓存的查询SimpleExecutor
此时会调用执行器对象SimpleExecutor的父类方法BaseExecutor#query。首先对参数进行处理，即sql中的#{}中的属性进行处理；然后创建CacheKey对象，这个对象在后面处理缓存的时候会用到，接下来就是sql的执行逻辑了。
```java
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds,   ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      //从会话级缓存中获取
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        //到数据库中查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```
在查询前会到localCache中根据CacheKey获取查询数据，而localCache是在构造方法中生成的，构造方法只有在创建SqlSession的时候才会被调用，因此这就是个会话级缓存。获取到以后直接处理即可，没有获取就要到数据库查询BaseExecutor#queryFromDatabase，查询到结果后就会将CacheKey和对应的查询结果List缓存到localCache中，之后就是jdbc数据库驱动查询的逻辑，先设置参数，再查询，然后处理查询结果。
+ 带全局缓存的查询CachingExecutor
此时会调用CachingExecutor#query方法进行查询，先尝试获取缓存`List<E> list = (List<E>) tcm.getObject(cache, key);`，没有的话就到数据库查询，就是上面的SimpleExecutor的查询逻辑，这里也能看出mybatis在查询时优先到全局缓存获取，再到会话缓存中获取。查询到数据后再缓存下来`tcm.putObject(cache, key, list);`。
```java
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      //根据标签的属性flushCache判断是否刷新缓存
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
至此，sql已经执行完成，接下来就是清除会话级缓存，提交事务。
**总结一下，mybatis在调用查询方法时，先对参数进行处理，再添加拦截器。查询时，先执行拦截器方法，再到全局缓存中获取，然后到会话级缓存中获取，没有获取缓存结果就到数据库查询，完成查询拿到结果后就是提交事务了。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**