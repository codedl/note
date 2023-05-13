引入tkmybatis后还是通过声明接口的扫描路径，对接口进行扫描，这个跟[mybatis-@MapperScan扫描接口](https://blog.csdn.net/weixin_42145727/article/details/127247558)是一样的，只是在处理bean定义时用的是自定义的processBeanDefinitions()。
```java
// ClassPathMapperScanner.java
    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
        GenericBeanDefinition definition;
        for (BeanDefinitionHolder holder : beanDefinitions) {
            ...
            // 通过构造函数注入扫描到的接口
            definition.getConstructorArgumentValues().addGenericArgumentValue(definition.getBeanClassName()); // issue #59
            // 实例化时构造MapperFactoryBean对象
            definition.setBeanClass(this.mapperFactoryBean.getClass());
            //设置通用 Mapper 为MapperHelper
            if(StringUtils.hasText(this.mapperHelperBeanName)){
                definition.getPropertyValues().add("mapperHelper", new RuntimeBeanReference(this.mapperHelperBeanName));
            } else {
                //不做任何配置的时候使用默认方式
                if(this.mapperHelper == null){
                    this.mapperHelper = new MapperHelper();
                }
                definition.getPropertyValues().add("mapperHelper", this.mapperHelper);
            }

            ...

            if (!explicitFactoryUsed) {
                // 默认按类型注入
                definition.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_BY_TYPE);
            }
        }
    }   
```
MapperFactoryBean实现了FactoryBean接口，会通过getObject()实例化Bean；同时实现了SqlSessionDaoSupport，其钩子函数checkDaoConfig()在实例化以后会被调用。Configuration是mybatis的全局单例配置对象，不多赘述。
```java
// MapperFactoryBean.java
    protected void checkDaoConfig() {
        super.checkDaoConfig();

        notNull(this.mapperInterface, "Property 'mapperInterface' is required");

        Configuration configuration = getSqlSession().getConfiguration();
        if (this.addToConfig && !configuration.hasMapper(this.mapperInterface)) {
            try {
                configuration.addMapper(this.mapperInterface);
            } catch (Exception e) {
                logger.error("Error while adding the mapper '" + this.mapperInterface + "' to configuration.", e);
                throw new IllegalArgumentException(e);
            } finally {
                ErrorContext.instance().reset();
            }
        }
        //直接针对接口处理通用接口方法对应的 MappedStatement 是安全的，通用方法不会出现 IncompleteElementException 的情况
        if (configuration.hasMapper(this.mapperInterface) && mapperHelper != null && mapperHelper.isExtendCommonMapper(this.mapperInterface)) {
            mapperHelper.processConfiguration(getSqlSession().getConfiguration(), this.mapperInterface);
        }
    }
```
registerClass(`private List<Class<?>> registerClass = new ArrayList<Class<?>>();`)是保存通用接口的list集合，这里就是在判断扫描的接口是否继承了这些通用接口，第一次会通过hasRegisterMapper()内部注册这些接口。
```java
    public boolean isExtendCommonMapper(Class<?> mapperInterface) {
        for (Class<?> mapperClass : registerClass) {
            if (mapperClass.isAssignableFrom(mapperInterface)) {
                return true;
            }
        }
        //通过 @RegisterMapper 注解自动注册的功能
        return hasRegisterMapper(mapperInterface);
    }
```

```java
    private boolean hasRegisterMapper(Class<?> mapperInterface){
        //获取扫描接口继承的接口
        Class<?>[] interfaces = mapperInterface.getInterfaces();
        boolean hasRegisterMapper = false;
        if (interfaces != null && interfaces.length > 0) {
            for (Class<?> anInterface : interfaces) {
                //自动注册标记了 @RegisterMapper 的接口
                if(anInterface.isAnnotationPresent(RegisterMapper.class)){
                    hasRegisterMapper = true;
                    //如果已经注册过，就避免在反复调用下面会迭代的方法
                    if (!registerMapper.containsKey(anInterface)) {
                        //注册
                        registerMapper(anInterface);
                    }
                }
                //递归地注册被扫描的接口继承的所有接口
                else if(hasRegisterMapper(anInterface)){
                    hasRegisterMapper = true;
                }
            }
        }
        return hasRegisterMapper;
    }
```
注册接口是没什么讲究的，就是保存到集合，关键是通过fromMapperClass()方法获取MapperTemplate
```java
    public void registerMapper(Class<?> mapperClass) {
        if (!registerMapper.containsKey(mapperClass)) {
            registerClass.add(mapperClass);
            registerMapper.put(mapperClass, fromMapperClass(mapperClass));
        }
        //自动注册继承的接口
        Class<?>[] interfaces = mapperClass.getInterfaces();
        if (interfaces != null && interfaces.length > 0) {
            for (Class<?> anInterface : interfaces) {
                registerMapper(anInterface);
            }
        }
    }
```
这里直接讲可能不好讲，以UpdateByPrimaryKeyMapper接口为例进行说明，其声明的updateByPrimaryKey()方法存在`@UpdateProvider(type = BaseUpdateProvider.class, method = "dynamicSQL")`注解，这里就是获取BaseUpdateProvider类，再以反射方式构造BaseUpdateProvider的实例对象，最后在UpdateByPrimaryKeyMapper接口中的方法名updateByPrimaryKey与BaseUpdateProvider类中的updateByPrimaryKey方法Method对象建立映射关系，记住这个映射，后面会用到。
```java
    private MapperTemplate fromMapperClass(Class<?> mapperClass) {
        Method[] methods = mapperClass.getDeclaredMethods();
        Class<?> templateClass = null;
        Class<?> tempClass = null;
        Set<String> methodSet = new HashSet<String>();
        for (Method method : methods) {
            if (method.isAnnotationPresent(SelectProvider.class)) {
                SelectProvider provider = method.getAnnotation(SelectProvider.class);
                tempClass = provider.type();
                methodSet.add(method.getName());
            } else if (method.isAnnotationPresent(InsertProvider.class)) {
                InsertProvider provider = method.getAnnotation(InsertProvider.class);
                tempClass = provider.type();
                methodSet.add(method.getName());
            } else if (method.isAnnotationPresent(DeleteProvider.class)) {
                DeleteProvider provider = method.getAnnotation(DeleteProvider.class);
                tempClass = provider.type();
                methodSet.add(method.getName());
            } else if (method.isAnnotationPresent(UpdateProvider.class)) {
                UpdateProvider provider = method.getAnnotation(UpdateProvider.class);
                tempClass = provider.type();
                methodSet.add(method.getName());
            }
            if (templateClass == null) {
                templateClass = tempClass;
            } else if (templateClass != tempClass) {
                throw new MapperException("一个通用Mapper中只允许存在一个MapperTemplate子类!");
            }
        }
        if (templateClass == null || !MapperTemplate.class.isAssignableFrom(templateClass)) {
            templateClass = EmptyProvider.class;
        }
        MapperTemplate mapperTemplate = null;
        try {
            mapperTemplate = (MapperTemplate) templateClass.getConstructor(Class.class, MapperHelper.class).newInstance(mapperClass, this);
        } catch (Exception e) {
            throw new MapperException("实例化MapperTemplate对象失败:" + e.getMessage());
        }
        //注册方法
        for (String methodName : methodSet) {
            try {
                //将接口中声明的方法名与注解中声明的模板类的方法Method对象的映射关系保存下来
                mapperTemplate.addMethodMap(methodName, templateClass.getMethod(methodName, MappedStatement.class));
            } catch (NoSuchMethodException e) {
                throw new MapperException(templateClass.getCanonicalName() + "中缺少" + methodName + "方法!");
            }
        }
        return mapperTemplate;
    }
```
最后一路调用到setSqlSource()，这个方法创建了sql的动态xml标签，然后交由mybatis创建动态sql。对于UpdateByPrimaryKeyMapper接口而言，其获取到的是BaseUpdateProvider类中的updateByPrimaryKey方法Method对象，`public String updateByPrimaryKey(MappedStatement ms) `看下方法签名就知道其返回了String，所以会返回xml形式的sql字符串。
```java
// MapperTemplate.java
    public void setSqlSource(MappedStatement ms) throws Exception {
        if (this.mapperClass == getMapperClass(ms.getId())) {
            throw new MapperException("请不要配置或扫描通用Mapper接口类：" + this.mapperClass);
        }
        Method method = methodMap.get(getMethodName(ms));
        try {
            //第一种，直接操作ms，不需要返回值
            if (method.getReturnType() == Void.TYPE) {
                method.invoke(this, ms);
            }
            //第二种，返回SqlNode
            else if (SqlNode.class.isAssignableFrom(method.getReturnType())) {
                SqlNode sqlNode = (SqlNode) method.invoke(this, ms);
                DynamicSqlSource dynamicSqlSource = new DynamicSqlSource(ms.getConfiguration(), sqlNode);
                setSqlSource(ms, dynamicSqlSource);
            }
            //第三种，返回xml形式的sql字符串
            else if (String.class.equals(method.getReturnType())) {
                String xmlSql = (String) method.invoke(this, ms);
                SqlSource sqlSource = createSqlSource(ms, xmlSql);
                //替换原有的SqlSource
                setSqlSource(ms, sqlSource);
            } else {
                throw new MapperException("自定义Mapper方法返回类型错误,可选的返回类型为void,SqlNode,String三种!");
            }
        } catch (IllegalAccessException e) {
            throw new MapperException(e);
        } catch (InvocationTargetException e) {
            throw new MapperException(e.getTargetException() != null ? e.getTargetException() : e);
        }
    }
```
这里就好看多了，就是生成update的动态xml。
```java
    public String updateByPrimaryKey(MappedStatement ms) {
        Class<?> entityClass = getEntityClass(ms);
        StringBuilder sql = new StringBuilder();
        sql.append(SqlHelper.updateTable(entityClass, tableName(entityClass)));
        sql.append(SqlHelper.updateSetColumns(entityClass, null, false, false));
        sql.append(SqlHelper.wherePKColumns(entityClass, true));
        return sql.toString();
    }
```
首先第一步获取实体类，其实就是获取泛型类，
```java
    public Class<?> getEntityClass(MappedStatement ms) {
        String msId = ms.getId();
        if (entityClassMap.containsKey(msId)) {
            return entityClassMap.get(msId);
        } else {
            Class<?> mapperClass = getMapperClass(msId);
            Type[] types = mapperClass.getGenericInterfaces();
            for (Type type : types) {
                if (type instanceof ParameterizedType) {
                    ParameterizedType t = (ParameterizedType) type;
                    if (t.getRawType() == this.mapperClass || this.mapperClass.isAssignableFrom((Class<?>) t.getRawType())) {
                        Class<?> returnType = (Class<?>) t.getActualTypeArguments()[0];
                        //获取该类型后，第一次对该类型进行初始化
                        EntityHelper.initEntityNameMap(returnType, mapperHelper.getConfig());
                        entityClassMap.put(msId, returnType);
                        return returnType;
                    }
                }
            }
        }
        throw new MapperException("无法获取 " + msId + " 方法的泛型信息!");
    }
```
再往下就是对实体类信息的封装了，通过@Table注解获取映射的表，
```java
//DefaultEntityResolve.java
    public EntityTable resolveEntity(Class<?> entityClass, Config config) {
        Style style = config.getStyle();
        //style，该注解优先于全局配置
        if (entityClass.isAnnotationPresent(NameStyle.class)) {
            NameStyle nameStyle = entityClass.getAnnotation(NameStyle.class);
            style = nameStyle.value();
        }

        //创建并缓存EntityTable
        EntityTable entityTable = null;
        if (entityClass.isAnnotationPresent(Table.class)) {
            Table table = entityClass.getAnnotation(Table.class);
            if (!"".equals(table.name())) {
                entityTable = new EntityTable(entityClass);
                entityTable.setTable(table);
            }
        }
        if (entityTable == null) {
            entityTable = new EntityTable(entityClass);
            //可以通过stye控制
            String tableName = StringUtil.convertByStyle(entityClass.getSimpleName(), style);
            //自动处理关键字
            if (StringUtil.isNotEmpty(config.getWrapKeyword()) && SqlReservedWords.containsWord(tableName)) {
                tableName = MessageFormat.format(config.getWrapKeyword(), tableName);
            }
            entityTable.setName(tableName);
        }
        entityTable.setEntityClassColumns(new LinkedHashSet<EntityColumn>());
        entityTable.setEntityClassPKColumns(new LinkedHashSet<EntityColumn>());
        //处理所有列
        List<EntityField> fields = null;
        if (config.isEnableMethodAnnotation()) {
            fields = FieldHelper.getAll(entityClass);
        } else {
            fields = FieldHelper.getFields(entityClass);
        }
        for (EntityField field : fields) {
            //如果启用了简单类型，就做简单类型校验，如果不是简单类型，直接跳过
            //3.5.0 如果启用了枚举作为简单类型，就不会自动忽略枚举类型
            //4.0 如果标记了 Column 或 ColumnType 注解，也不忽略
            if (config.isUseSimpleType()
                    && !field.isAnnotationPresent(Column.class)
                    && !field.isAnnotationPresent(ColumnType.class)
                    && !(SimpleTypeUtil.isSimpleType(field.getJavaType())
                    ||
                    (config.isEnumAsSimpleType() && Enum.class.isAssignableFrom(field.getJavaType())))) {
                continue;
            }
            processField(entityTable, field, config, style);
        }
        //当pk.size=0的时候使用所有列作为主键
        if (entityTable.getEntityClassPKColumns().size() == 0) {
            entityTable.setEntityClassPKColumns(entityTable.getEntityClassColumns());
        }
        entityTable.initPropertyMap();
        return entityTable;
    }
```
接着就是字段处理，使用了@Transient注解则不关联sql，@Id注解标记注解，@Column注解映射字段，@ColumnType注解处理复杂类型，@OrderBy指定用来排序的字段，没有@Column注解则自动转化，还是比较智能的哈。
```java
    protected void processField(EntityTable entityTable, EntityField field, Config config, Style style) {
        //排除字段
        if (field.isAnnotationPresent(Transient.class)) {
            return;
        }
        //Id
        EntityColumn entityColumn = new EntityColumn(entityTable);
        //是否使用 {xx, javaType=xxx}
        entityColumn.setUseJavaType(config.isUseJavaType());
        //记录 field 信息，方便后续扩展使用
        entityColumn.setEntityField(field);
        if (field.isAnnotationPresent(Id.class)) {
            entityColumn.setId(true);
        }
        //Column
        String columnName = null;
        if (field.isAnnotationPresent(Column.class)) {
            Column column = field.getAnnotation(Column.class);
            columnName = column.name();
            entityColumn.setUpdatable(column.updatable());
            entityColumn.setInsertable(column.insertable());
        }
        //ColumnType
        if (field.isAnnotationPresent(ColumnType.class)) {
            ColumnType columnType = field.getAnnotation(ColumnType.class);
            //是否为 blob 字段
            entityColumn.setBlob(columnType.isBlob());
            //column可以起到别名的作用
            if (StringUtil.isEmpty(columnName) && StringUtil.isNotEmpty(columnType.column())) {
                columnName = columnType.column();
            }
            if (columnType.jdbcType() != JdbcType.UNDEFINED) {
                entityColumn.setJdbcType(columnType.jdbcType());
            }
            if (columnType.typeHandler() != UnknownTypeHandler.class) {
                entityColumn.setTypeHandler(columnType.typeHandler());
            }
        }
        //列名
        if (StringUtil.isEmpty(columnName)) {
            columnName = StringUtil.convertByStyle(field.getName(), style);
        }
        //自动处理关键字
        if (StringUtil.isNotEmpty(config.getWrapKeyword()) && SqlReservedWords.containsWord(columnName)) {
            columnName = MessageFormat.format(config.getWrapKeyword(), columnName);
        }
        entityColumn.setProperty(field.getName());
        entityColumn.setColumn(columnName);
        entityColumn.setJavaType(field.getJavaType());
        if (field.getJavaType().isPrimitive()) {
            log.warn("通用 Mapper 警告信息: <[" + entityColumn + "]> 使用了基本类型，基本类型在动态 SQL 中由于存在默认值，因此任何时候都不等于 null，建议修改基本类型为对应的包装类型!");
        }
        //OrderBy
        processOrderBy(entityTable, field, entityColumn);
        //处理主键策略
        processKeyGenerator(entityTable, field, entityColumn);
        entityTable.getEntityClassColumns().add(entityColumn);
        if (entityColumn.isId()) {
            entityTable.getEntityClassPKColumns().add(entityColumn);
        }
    }
```
这样就得到了实体类与数据库表的关联信息，然后就是生成xml标签了，就是字符串处理，没什么逻辑。
**总结一下，tkmybatis先获取实体类，然后生成映射的xml标签，再交由mybatis处理。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**