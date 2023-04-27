mybatis在xml文件中可以处理类型别名，根据别名获取实体类，具体是在SqlSessionFactoryBean.java#buildSqlSessionFactory()，typeAliasesPackage是mybatis.type-aliases-package配置项指定的包路径，可以指定多个，多个之间可以用",; \t\n"隔开。
```java
// SqlSessionFactoryBean
    if (hasLength(this.typeAliasesPackage)) {
      String[] typeAliasPackageArray = tokenizeToStringArray(this.typeAliasesPackage,
          ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);
      for (String packageToScan : typeAliasPackageArray) {
        //默认typeAliasesSuperType是Object
        configuration.getTypeAliasRegistry().registerAliases(packageToScan,
                typeAliasesSuperType == null ? Object.class : typeAliasesSuperType);
        if (LOGGER.isDebugEnabled()) {
          LOGGER.debug("Scanned package: '" + packageToScan + "' for aliases");
        }
      }
    }
```
处理类型别名的是TypeAliasRegistry`protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();`，即由TypeAliasRegistry#registerAliases()方法处理。
```java
  public void registerAliases(String packageName, Class<?> superType){
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<Class<?>>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for(Class<?> type : typeSet){
      // Ignore inner classes and interfaces (including package-info.java)
      // Skip also inner classes. See issue #6
      if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
        registerAlias(type);
      }
    }
  }
```
这里就是根据包路径找到所有的class，
```java
  public ResolverUtil<T> find(Test test, String packageName) {
    String path = getPackagePath(packageName);

    try {
      //找出path下所有的class路径全称  
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

  public List<String> list(String path) throws IOException {
    List<String> names = new ArrayList<String>();
    for (URL url : getResources(path)) {
      names.addAll(list(url, path));
    }
    return names;
  }

  protected List<String> list(URL url, String path) throws IOException {
    Resource[] resources = resourceResolver.getResources("classpath*:" + path + "/**/*.class");
    List<String> resourcePaths = new ArrayList<String>();
    for (Resource resource : resources) {
      resourcePaths.add(preserveSubpackageName(resource.getURI(), path));
    }
    return resourcePaths;
  }  
```
首先将返回的class路径全称最后的.class去掉，然后将'/'换成'.'，这样就能得到Java语法格式的class路径名，然后使用类加载器进行加载就能得到Class。
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

    public boolean matches(Class<?> type) {
      return type != null && parent.isAssignableFrom(type);
    }
```
最后最关键的一步是注册别名，首先通过class获取别名，一般是类名大写，如果用了@Alias，则使用@Alias注解指定的别名进行注册。
```java
  public void registerAlias(Class<?> type) {
    String alias = type.getSimpleName();
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      alias = aliasAnnotation.value();
    } 
    registerAlias(alias, type);
  }
```
注册的时候别名全小写，最后是保存到`private final Map<String, Class<?>> TYPE_ALIASES = new HashMap<String, Class<?>>();`，之后我们在xml中就可以通过别名引用了。
```java
  public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    String key = alias.toLowerCase(Locale.ENGLISH);
    if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
    }
    TYPE_ALIASES.put(key, value);
  }
```
**总结一下，mybatis会根据配置项mybatis.type-aliases-package的值获取类别名路径，然后到路径下扫描获取路径下所有的class文件，在替换成Java语法格式class路径，然后使用类加载器进行加载得到Class对象。接着获取类的别名，可以用@Alias指定，没有就是类的小写全称，最终是保存到map中。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**