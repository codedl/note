spring是在DefaultListableBeanFactory#preInstantiateSingletons()方法中对容器中所有Bean进行实例化的，就从这个方法开始分析。对Bean进行实例化的时候判断是否为FactoryBean，即实现了FactoryBean接口的Bean，如果是则对FactoryBean接口的实现类进行实例化，此时注意会在beanName之前加上"&"，没有"&"的话就是调用getObject()方法创建Bean，可以看到创建Bean都是调用getBean()，再继续看这个方法。
```java
	public void preInstantiateSingletons() throws BeansException {
		......
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					......
				}
				else {
					getBean(beanName);
				}
			}
		}
        ......
	}
```
doGetBean()中会先处理FactoryBean，然后将实例化的Bean加入Set集合，检查是否会出现@DepondOn导致的循环依赖，最后进行单例Bean的实例化。
```java
    protected <T> T doGetBean(
			String name, @Nullable Class<T> requiredType, @Nullable Object[] args, boolean typeCheckOnly)
			throws BeansException {
        //把name开头的"&"去掉
		String beanName = transformedBeanName(name);
		Object bean;

		//从缓存中获取，实例化后的Bean都会被缓存，既可以解决循环依赖，又可以防止重复创建，也是spring的单例设计
		Object sharedInstance = getSingleton(beanName);
        //如果是FactoryBean，根据"&"判断是获取getObject()还是FactoryBean对象本身
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}
        //没有获取到说明容器中没有，现在就要进行实例化
		else {
            //原型Bean不能在此时被实例化
			......
            //当前容器不能创建的话就找父容器创建
            ......

            //将正常创建的Bean的name加入alreadyCreated(Set<String>)
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 检查@DependsOn依赖组件是否出现循环依赖
                //做法是将被依赖的Bean当key，依赖这个Bean的Bean当value，value为set集合
                //如果set集合中出现了key则发生了循环依赖，循环依赖最大的特点就是最终都依赖自身
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}
                //原型Bean的创建
				......
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		......
		return (T) bean;
	}
```
看下FactoryBean的获取逻辑，如果Bean的name以"&"开头则返回FactoryBean本身，否则返回getObject()获取的Bean。
```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		//bean的name是否以"&"开头
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
            //如果获取的是FactoryBean，直接返回
			return beanInstance;
		}

		// 如果bean的name不是以"&"开头且不是FactoryBean，直接返回
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}

		Object object = null;
        //实例化阶段时需要标记isFactoryBean为true
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
            //直接从缓存中获取getObject()方法创建的Bean
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
            //没有从缓存中获取到的话就调用FactoryBean的getObject()方法生成Bean，再缓存下来
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```
Bean实例化时先加入singletonsCurrentlyInCreation(Set<String>)集合进行标记，然后进行实例化，等实例化完成就从集合中删除，添加到缓存。
```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
            //先从缓存中获取
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				....
                //创建之前先加入池子进行标记
				beforeSingletonCreation(beanName);
				......
				try {
                    //Bean的实例化阶段
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
                ......
                    //创建完成后从池子中删掉
					afterSingletonCreation(beanName);
				
				if (newSingleton) {
                    //缓存完整的属性填充后的Bean
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```
从Bean的定义中获取Class，然后在doCreateBean()中对Bean进行实例化。
```java
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		.....
		RootBeanDefinition mbdToUse = mbd;
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		......
		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			.....
			return beanInstance;
		}
		......
	}
```
doCreateBean()就是反射调用bean的构造函数进行实例化，然后缓存用来创建bean的lambda表达式，填充bean的属性，初始化bean。
```java
	protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		// 1.反射调用Bean的构造函数进行实例化
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		......
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			//2.缓存获取bean的lambda表达式，因此有可能返回bean的代理对象
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
            //填充bean的属性
			populateBean(beanName, mbd, instanceWrapper);
            //bean的初始化方法调用，@PostConstruct、@Init、afterPropertiesSet()
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		......

		return exposedObject;
	}
```
**总结下，bean的实例化的时候首先处理FactoryBean，然后将正在创建的Bean加入池子(set集合)，之后通过bean的Class反射调用构造函数进行实例化，对bean的属性进行填充。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步。**