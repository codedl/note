mybaits在SqlSessionFactoryBean.java#buildSqlSessionFactory()中会注册用户自定义的TypeHandler，首先获取TypeHandler所在的包路径，可能是多个，然后依次注册每个路径下的TypeHandler。
```java
    if (hasLength(this.typeHandlersPackage)) {
      String[] typeHandlersPackageArray = tokenizeToStringArray(this.typeHandlersPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeHandlersPackageArray) {
        configuration.getTypeHandlerRegistry().register(packageToScan);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for type handlers");
        }
      }
    }
```
首先实例化工具类，用来扫描包路径下的TypeHandler，然后注册进来。IsA是ResolverUtil的子类，实现了Test接口中的matches方法用来进行类型匹配。
```java
  public void register(String packageName) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(TypeHandler.class), packageName);
    Set<Class<? extends Class<?>>> handlerSet = resolverUtil.getClasses();
    for (Class<?> type : handlerSet) {
      //Ignore inner classes and interfaces (including package-info.java) and abstract classes
      if (!type.isAnonymousClass() && !type.isInterface() && !Modifier.isAbstract(type.getModifiers())) {
        register(type);
      }
    }
  }
```
注册的时候先扫描包路径下所有的class。
```java
  public ResolverUtil<T> find(Test test, String packageName) {
    //将文件路径名转化成Java中包名
    String path = getPackagePath(packageName);

    try {
        //获取所有class类路径全限定字符串
      List<String> children = VFS.getInstance().list(path);
      for (String child : children) {
        if (child.endsWith(".class")) {
          addIfMatching(test, child);
        }
      }
    } catch (IOException ioe) {
      log.error("Could not read package: " + packageName, ioe);
    }

    return this;
  }
```
使用类加载器加载class，然后调用ResolverUtil.IsA中matches进行匹配，如果是TypeHandler类型则添加到matches(`private Set<Class<? extends T>> matches = new HashSet<Class<? extends T>>();`)。
```java
  protected void addIfMatching(Test test, String fqn) {
    try {
      String externalName = fqn.substring(0, fqn.indexOf('.')).replace('/', '.');
      ClassLoader loader = getClassLoader();
      if (log.isDebugEnabled()) {
        log.debug("Checking to see if class " + externalName + " matches criteria [" + test + "]");
      }

      Class<?> type = loader.loadClass(externalName);
      if (test.matches(type)) {
        matches.add((Class<T>) type);
      }
    } catch (Throwable t) {
      log.warn("Could not examine class '" + fqn + "'" + " due to a " +
          t.getClass().getName() + " with message: " + t.getMessage());
    }
  }
```
找到要注册的TypeHandler之后，先获取class上的@MappedTypes注解，取value属性，进行注册。
```java
  public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      for (Class<?> javaTypeClass : mappedTypes.value()) {
        register(javaTypeClass, typeHandlerClass);
        mappedTypeFound = true;
      }
    }
    if (!mappedTypeFound) {
      register(getInstance(null, typeHandlerClass));
    }
  }
```
获取到TypeHandler以后，反射构造TypeHandler的实例对象，然后保存到map中，`private final Map<Type, Map<JdbcType, TypeHandler<?>>> TYPE_HANDLER_MAP = new ConcurrentHashMap<Type, Map<JdbcType, TypeHandler<?>>>();`，TYPE_HANDLER_MAP的key是java类型，value是个map(保存了jdbc类型到TypeHandler的映射关系)；`private final Map<Class<?>, TypeHandler<?>> ALL_TYPE_HANDLERS_MAP = new HashMap<Class<?>, TypeHandler<?>>();`保存的是TypeHandler的Class类型到TypeHandler的实例对象的映射关系。
```java
  public void register(Class<?> javaTypeClass, JdbcType jdbcType, Class<?> typeHandlerClass) {
    register(javaTypeClass, jdbcType, getInstance(javaTypeClass, typeHandlerClass));
  }
  public <T> TypeHandler<T> getInstance(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
    if (javaTypeClass != null) {
      try {
        Constructor<?> c = typeHandlerClass.getConstructor(Class.class);
        return (TypeHandler<T>) c.newInstance(javaTypeClass);
      } catch (NoSuchMethodException ignored) {
        // ignored
      } catch (Exception e) {
        throw new TypeException("Failed invoking constructor for handler " + typeHandlerClass, e);
      }
    }
    try {
      Constructor<?> c = typeHandlerClass.getConstructor();
      return (TypeHandler<T>) c.newInstance();
    } catch (Exception e) {
      throw new TypeException("Unable to find a usable constructor for " + typeHandlerClass, e);
    }
  }
  private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    if (javaType != null) {
      Map<JdbcType, TypeHandler<?>> map = TYPE_HANDLER_MAP.get(javaType);
      if (map == null || map == NULL_TYPE_HANDLER_MAP) {
        map = new HashMap<JdbcType, TypeHandler<?>>();
        TYPE_HANDLER_MAP.put(javaType, map);
      }
      map.put(jdbcType, handler);
    }
    ALL_TYPE_HANDLERS_MAP.put(handler.getClass(), handler);
  }
```