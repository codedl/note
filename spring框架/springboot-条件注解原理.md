在ConditionEvaluator.java#shouldSkip()方法中会通过条件注解判断是否可以对当前配置类进行实例化，这里会对作用在类上的条件注解进行处理，获取条件注解上指定的Condition接口的实现类，再进行实例化得到Condition，再调用接口中的matches()进行条件判断。
```java
// ConditionEvaluator
	public boolean shouldSkip(@Nullable AnnotatedTypeMetadata metadata, @Nullable ConfigurationPhase phase) {
		if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
			return false;
		}

		if (phase == null) {
			if (metadata instanceof AnnotationMetadata &&
					ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
				return shouldSkip(metadata, ConfigurationPhase.PARSE_CONFIGURATION);
			}
			return shouldSkip(metadata, ConfigurationPhase.REGISTER_BEAN);
		}
        //获取每个Condition
		List<Condition> conditions = new ArrayList<>();
		for (String[] conditionClasses : getConditionClasses(metadata)) {
			for (String conditionClass : conditionClasses) {
				Condition condition = getCondition(conditionClass, this.context.getClassLoader());
				conditions.add(condition);
			}
		}

		AnnotationAwareOrderComparator.sort(conditions);
        //调用每个Condition的matches()进行校验
		for (Condition condition : conditions) {
			ConfigurationPhase requiredPhase = null;
			if (condition instanceof ConfigurationCondition) {
				requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
			}
			if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
				return true;
			}
		}

		return false;
	}
```
这里以`@ConditionalOnClass`注解为例进行说明，`@ConditionalOnClass`注解上通过`@Conditional`指定Condition为OnClassCondition，此时会调用OnClassCondition的父类SpringBootCondition中的matches()方法进行匹配，此时会调用模板方法getMatchOutcome()，获取条件判断的结果。
```java
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {...}

// SpringBootCondition.java
	public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		String classOrMethodName = getClassOrMethodName(metadata);
		try {
			ConditionOutcome outcome = getMatchOutcome(context, metadata);
			logOutcome(classOrMethodName, outcome);
			recordEvaluation(context, classOrMethodName, outcome);
			return outcome.isMatch();
		}
		.......
	}
```

```java
	public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
		ClassLoader classLoader = context.getClassLoader();
		ConditionMessage matchMessage = ConditionMessage.empty();
        //获取@ConditionalOnClass指定的条件装配类
		List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
		if (onClasses != null) {
            //判断这些类是否都存在，missing是个包含缺少类的集合
			List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
			if (!missing.isEmpty()) {
				return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
						.didNotFind("required class", "required classes").items(Style.QUOTE, missing));
			}
			matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
					.found("required class", "required classes")
					.items(Style.QUOTE, filter(onClasses, ClassNameFilter.PRESENT, classLoader));
		}
		......
		return ConditionOutcome.match(matchMessage);
	}
```
filetr中指定了使用ClassNameFilter.MISSING作为过滤器，因此会调用ClassNameFilter.MISSING中的matches()方法，再底层就是Class.forName()去加载类，如果能加载成功说明类存在返回true，如果不能加载成功说明类不存在，返回false，加入到missing集合。
```java
// FilteringSpringBootCondition.java
	protected final List<String> filter(Collection<String> classNames, ClassNameFilter classNameFilter,
			ClassLoader classLoader) {
		if (CollectionUtils.isEmpty(classNames)) {
			return Collections.emptyList();
		}
		List<String> matches = new ArrayList<>(classNames.size());
		for (String candidate : classNames) {
			if (classNameFilter.matches(candidate, classLoader)) {
				matches.add(candidate);
			}
		}
		return matches;
	}
// FilteringSpringBootCondition.java
    	protected enum ClassNameFilter {

		PRESENT {

			@Override
			public boolean matches(String className, ClassLoader classLoader) {
				return isPresent(className, classLoader);
			}

		},

		MISSING {

			@Override
			public boolean matches(String className, ClassLoader classLoader) {
				return !isPresent(className, classLoader);
			}

		};

		abstract boolean matches(String className, ClassLoader classLoader);

		static boolean isPresent(String className, ClassLoader classLoader) {
			if (classLoader == null) {
				classLoader = ClassUtils.getDefaultClassLoader();
			}
			try {
				resolve(className, classLoader);
				return true;
			}
			catch (Throwable ex) {
				return false;
			}
		}

	}
```