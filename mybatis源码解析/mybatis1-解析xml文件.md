在`spring.factories`中定义了要自动加载的配置类`MybatisAutoConfiguration`，在这个类上使用@Configuration注解，spring在启动时会对这个类进行解析，创建组件。
```java
@org.springframework.context.annotation.Configuration
@ConditionalOnClass({ SqlSessionFactory.class, SqlSessionFactoryBean.class })
@ConditionalOnBean(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter(DataSourceAutoConfiguration.class)
public class MybatisAutoConfiguration {...}
```
由于对方法sqlSessionFactory()使用了Bean注解，因此spring会调用此方法创建bean。
```java
  @Bean
  @ConditionalOnMissingBean
  public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    //对SqlSessionFactoryBean实例化
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    factory.setDataSource(dataSource);
    factory.setVfs(SpringBootVFS.class);
    //配置文件路径
    if (StringUtils.hasText(this.properties.getConfigLocation())) {
      factory.setConfigLocation(this.resourceLoader.getResource(this.properties.getConfigLocation()));
    }
    //mybatis的配置对象Configuration
    Configuration configuration = this.properties.getConfiguration();
    if (configuration == null && !StringUtils.hasText(this.properties.getConfigLocation())) {
      configuration = new Configuration();
    }
    if (configuration != null && !CollectionUtils.isEmpty(this.configurationCustomizers)) {
      for (ConfigurationCustomizer customizer : this.configurationCustomizers) {
        customizer.customize(configuration);
      }
    }
    factory.setConfiguration(configuration);
    if (this.properties.getConfigurationProperties() != null) {
      factory.setConfigurationProperties(this.properties.getConfigurationProperties());
    }
    //设置拦截器，待会看看怎么用
    if (!ObjectUtils.isEmpty(this.interceptors)) {
      factory.setPlugins(this.interceptors);
    }
    if (this.databaseIdProvider != null) {
      factory.setDatabaseIdProvider(this.databaseIdProvider);
    }
    if (StringUtils.hasLength(this.properties.getTypeAliasesPackage())) {
      factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
    }
    if (StringUtils.hasLength(this.properties.getTypeHandlersPackage())) {
      factory.setTypeHandlersPackage(this.properties.getTypeHandlersPackage());
    }
    //设置包含动态sql的xml文件路径
    if (!ObjectUtils.isEmpty(this.properties.resolveMapperLocations())) {
      factory.setMapperLocations(this.properties.resolveMapperLocations());
    }
   //创建SqlSessionFactory接口的实现类对象
    return factory.getObject();
  }
```
可以看到这个方法中先对SqlSessionFactoryBean类进行实例化，再调用getObject()创建SqlSessionFactory。首先在SqlSessionFactoryBean类的buildSqlSessionFactory()中配置Configuration对象，优先使用spring配置文件定义的配置，如果就看是否定义了mybatis的xml配置文件路径，再没有就默认一个。然后找到包含动态sql的xml映射文件，在XMLMapperBuilder.java#parse()中对xml文件进行解析。
```java
  public void parse() {
   //加载xml映射文件，对mapper节点进行解析
    if (!configuration.isResourceLoaded(resource)) {
      //对xml中每个节点进行解析
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }
   //对解析失败的resultMap、cache-ref、"select|insert|update|delete"标签进行重新解析
    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }
```
首先加载xml映射文件，对文件中每个标签进行解析，如果解析失败再重新尝试。接下来看下是怎么从xml中提取sql的。调用链是XMLMapperBuilder.java#configurationElement->buildStatementFromContext->XMLStatementBuilder.java#parseStatementNode->XMLLanguageDriver.java#createSqlSource。如果sql中含有"${"就是动态sql，用DynamicSqlSource表示，否则使用RawSqlSource，提取sql是在GenericTokenParser.java#parse()中完成的。
```java
  public String parse(String text) {
    if (text == null || text.isEmpty()) {
      return "";
    }
    //计算"#{"的位置
    int start = text.indexOf(openToken, 0);
    if (start == -1) {
      return text;
    }
    char[] src = text.toCharArray();
    int offset = 0;
    //将动态sql的字符保存到StringBuilder
    final StringBuilder builder = new StringBuilder();
    StringBuilder expression = null;
    while (start > -1) {
      //如果在"#{"还有个"\"，就不作处理
      if (start > 0 && src[start - 1] == '\\') {
        // this open token is escaped. remove the backslash and continue.
        builder.append(src, offset, start - offset - 1).append(openToken);
        offset = start + openToken.length();
      } else {
        // found open token. let's search close token.
        if (expression == null) {
          expression = new StringBuilder();
        } else {
          expression.setLength(0);
        }
        //将"#{"之前的字符保存，即去除"#{"
        builder.append(src, offset, start - offset);
        offset = start + openToken.length();
        //找到"}"
        int end = text.indexOf(closeToken, offset);
        while (end > -1) {
         //如果有个\则不处理
          if (end > offset && src[end - 1] == '\\') {
            // this close token is escaped. remove the backslash and continue.
            expression.append(src, offset, end - offset - 1).append(closeToken);
            offset = end + closeToken.length();
            end = text.indexOf(closeToken, offset);
          } else {
            //
            expression.append(src, offset, end - offset);
            offset = end + closeToken.length();
            break;
          }
        }
        if (end == -1) {
          // close token was not found.
          builder.append(src, start, src.length - start);
          offset = src.length;
        } else {
         //如果是"#{}"则返回"?"，如果是"${}"则不作处理
          builder.append(handler.handleToken(expression.toString()));
          offset = end + closeToken.length();
        }
      }
      //继续处理下个#{}
      start = text.indexOf(openToken, offset);
    }
    if (offset < src.length) {
      builder.append(src, offset, src.length - offset);
    }
    return builder.toString();
  }
```
处理过程也很简单，先找#{，使用?代替#{}。完成sql字符串的解析后就是用StaticSqlSource进行封装，保存动态sql字符串、参数映射。然后在MapperBuilderAssistant.java#addMappedStatement()中将</mapper>中的namespace属性指定的类路径和标签的id属性用"."连接，就能得到方法的全限定路径作为key，再把标签的内容封装为MappedStatement作为value，保存到Configuration对象的mappedStatements`protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection");`中，StrictMap重写了put方法，put时既会保存方法的全限定名，又会保存方法名。
```java
    public V put(String key, V value) {
      if (containsKey(key)) {
        throw new IllegalArgumentException(name + " already contains value for " + key);
      }
      if (key.contains(".")) {
        final String shortKey = getShortName(key);
        if (super.get(shortKey) == null) {
          super.put(shortKey, value);
        } else {
          super.put(shortKey, (V) new Ambiguity(shortKey));
        }
      }
      return super.put(key, value);
    }
```
接下来就是在SqlSessionFactoryBuilder.java#build()中使用默认的DefaultSqlSessionFactory构建SqlSessionFactory。