众所周知，spring配置属性源有四个：命令行参数、Java系统属性、操作系统环境变量、配置文件，现在看下spring如何获取到这四个属性源。
```java
	private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
			ApplicationArguments applicationArguments) {
		//创建环境接口
		ConfigurableEnvironment environment = getOrCreateEnvironment();
        //封装命令行参数
		configureEnvironment(environment, applicationArguments.getSourceArgs());
		ConfigurationPropertySources.attach(environment);
        //发布环境准备好事件
		listeners.environmentPrepared(environment);
		bindToSpringApplication(environment);
		if (!this.isCustomEnvironment) {
			environment = new EnvironmentConverter(getClassLoader()).convertEnvironmentIfNecessary(environment,
					deduceEnvironmentClass());
		}
		ConfigurationPropertySources.attach(environment);
		return environment;
	}
```
创建的ConfigurableEnvironment接口默认为`StandardServletEnvironment`,所有配置均保存在`private final MutablePropertySources propertySources = new MutablePropertySources();`，`StandardServletEnvironment`继承的父类`AbstractEnvironment`构造方法中会添加Java系统属性、操作系统环境变量到内置属性源，此时会将Java系统属性和操作系统环境变量添加到属性源`MutablePropertySources`。
```java
	public AbstractEnvironment() {
		customizePropertySources(this.propertySources);
	}
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(
				new PropertiesPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME, getSystemProperties()));
		propertySources.addLast(
				new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME, getSystemEnvironment()));
	}
```
再就是封装命令行参数到属性源，使用`SimpleCommandLinePropertySource`进行封装后添加进来，参数格式为`--`开头则被解析成key、value格式保存下来，不然直接保存。
```java
	protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {
		MutablePropertySources sources = environment.getPropertySources();
		if (this.defaultProperties != null && !this.defaultProperties.isEmpty()) {
			sources.addLast(new MapPropertySource("defaultProperties", this.defaultProperties));
		}
		if (this.addCommandLineProperties && args.length > 0) {
			String name = CommandLinePropertySource.COMMAND_LINE_PROPERTY_SOURCE_NAME;
			if (sources.contains(name)) {
				PropertySource<?> source = sources.get(name);
				CompositePropertySource composite = new CompositePropertySource(name);
				composite.addPropertySource(
						new SimpleCommandLinePropertySource("springApplicationCommandLineArgs", args));
				composite.addPropertySource(source);
				sources.replace(name, composite);
			}
			else {
				sources.addFirst(new SimpleCommandLinePropertySource(args));
			}
		}
	}

	public CommandLineArgs parse(String... args) {
		CommandLineArgs commandLineArgs = new CommandLineArgs();
		for (String arg : args) {
			if (arg.startsWith("--")) {
				String optionText = arg.substring(2);
				String optionName;
				String optionValue = null;
				int indexOfEqualsSign = optionText.indexOf('=');
				if (indexOfEqualsSign > -1) {
					optionName = optionText.substring(0, indexOfEqualsSign);
					optionValue = optionText.substring(indexOfEqualsSign + 1);
				}
				else {
					optionName = optionText;
				}
				if (optionName.isEmpty()) {
					throw new IllegalArgumentException("Invalid argument syntax: " + arg);
				}
				commandLineArgs.addOptionArg(optionName, optionValue);
			}
			else {
				commandLineArgs.addNonOptionArg(arg);
			}
		}
		return commandLineArgs;
	}    
```
之后就是读取配置文件，这一步有点复杂，之前已经讲过spring会读取事件监听器，现在发布`ApplicationEnvironmentPreparedEvent`后，会找到支持的`ConfigFileApplicationListener`读取配置文件，现在看下这个过程。`ConfigFileApplicationListener`会从spring.factories找到所有`EnvironmentPostProcessor`处理配置，我们只关心读取配置文件的`EnvironmentPostProcessor`即`ConfigFileApplicationListener`自身即可。
```java
	private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
		List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
		postProcessors.add(this);
		AnnotationAwareOrderComparator.sort(postProcessors);
		for (EnvironmentPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessEnvironment(event.getEnvironment(), event.getSpringApplication());
		}
	}
        //加载配置文件
		void load() {
			FilteredPropertySource.apply(this.environment, DEFAULT_PROPERTIES, LOAD_FILTERED_PROPERTY,
					(defaultProperties) -> {
						this.profiles = new LinkedList<>();
						this.processedProfiles = new LinkedList<>();
						this.activatedProfiles = false;
						this.loaded = new LinkedHashMap<>();
						initializeProfiles();
						while (!this.profiles.isEmpty()) {
							Profile profile = this.profiles.poll();
							if (isDefaultProfile(profile)) {
								addProfileToEnvironment(profile.getName());
							}
							load(profile, this::getPositiveProfileFilter,
									addToLoaded(MutablePropertySources::addLast, false));
							this.processedProfiles.add(profile);
						}
						load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
						addLoadedPropertySources();
						applyActiveProfiles(defaultProperties);
					});
		}
```
可以看到共分成两部，首先读取环境配置，再根据环境配置。读取环境配置时，首先添加null，意思是没有任何环境后缀的配置文件，然后从已有的属性源即Java属性和环境变量中获取`spring.profiles.active`和`spring.profiles.include`的配置，如果没有的话就是default。
```java
		private void initializeProfiles() {
			// The default profile for these purposes is represented as null. We add it
			// first so that it is processed first and has lowest priority.
			this.profiles.add(null);
			Binder binder = Binder.get(this.environment);
			Set<Profile> activatedViaProperty = getProfiles(binder, ACTIVE_PROFILES_PROPERTY);
			Set<Profile> includedViaProperty = getProfiles(binder, INCLUDE_PROFILES_PROPERTY);
			List<Profile> otherActiveProfiles = getOtherActiveProfiles(activatedViaProperty, includedViaProperty);
			this.profiles.addAll(otherActiveProfiles);
			// Any pre-existing active profiles set via property sources (e.g.
			// System properties) take precedence over those added in config files.
			this.profiles.addAll(includedViaProperty);
			addActiveProfiles(activatedViaProperty);
			if (this.profiles.size() == 1) { // only has null profile
				for (String defaultProfileName : this.environment.getDefaultProfiles()) {
					Profile defaultProfile = new Profile(defaultProfileName, true);
					this.profiles.add(defaultProfile);
				}
			}
		}
```
读取到环境配置后，要加载配置文件了，此时从保存环境Profile的队列中取出来，再进行加载，可以看到null的环境配置加载了两次，由于存在`spring.config.activate.on-profile=dev`的配置项，所以第一次加载时无法处理，需要在加载完读取到环境配置时再进行处理。
```java
		void load() {
			FilteredPropertySource.apply(this.environment, DEFAULT_PROPERTIES, LOAD_FILTERED_PROPERTY,
					(defaultProperties) -> {
                        //待处理的环境配置
						this.profiles = new LinkedList<>();
                        //标记处理过的环境配置
						this.processedProfiles = new LinkedList<>();
						this.activatedProfiles = false;
                        //已完成加载的配置
						this.loaded = new LinkedHashMap<>();
						initializeProfiles();
						while (!this.profiles.isEmpty()) {
							Profile profile = this.profiles.poll();
							if (isDefaultProfile(profile)) {
								addProfileToEnvironment(profile.getName());
							}
							load(profile, this::getPositiveProfileFilter,
									addToLoaded(MutablePropertySources::addLast, false));
							this.processedProfiles.add(profile);
						}
						load(null, this::getNegativeProfileFilter, addToLoaded(MutablePropertySources::addFirst, true));
						addLoadedPropertySources();
						applyActiveProfiles(defaultProperties);
					});
		}
```
默认读取配置文件的路径为`private static final String DEFAULT_SEARCH_LOCATIONS = "classpath:/,classpath:/config/,file:./,file:./config/*/,file:./config/";`，现在spring会到这些路径下根据环境读取配置文件。
```java
		private void load(Profile profile, DocumentFilterFactory filterFactory, DocumentConsumer consumer) {
			getSearchLocations().forEach((location) -> {
				boolean isDirectory = location.endsWith("/");
				Set<String> names = isDirectory ? getSearchNames() : NO_SEARCH_NAMES;
				names.forEach((name) -> load(location, name, profile, filterFactory, consumer));
			});
		}
```
PropertySourceLoader读取器是由spring.factories文件配置的，默认有两个：`PropertiesPropertySourceLoader`、`YamlPropertySourceLoader`。
```java
		private void load(String location, String name, Profile profile, DocumentFilterFactory filterFactory,
				DocumentConsumer consumer) {
			if (!StringUtils.hasText(name)) {
				for (PropertySourceLoader loader : this.propertySourceLoaders) {
					if (canLoadFileExtension(loader, location)) {
						load(loader, location, profile, filterFactory.getDocumentFilter(profile), consumer);
						return;
					}
				}
				throw new IllegalStateException("File extension of config file location '" + location
						+ "' is not known to any PropertySourceLoader. If the location is meant to reference "
						+ "a directory, it must end in '/'");
			}
			Set<String> processed = new HashSet<>();
			for (PropertySourceLoader loader : this.propertySourceLoaders) {
				for (String fileExtension : loader.getFileExtensions()) {
					if (processed.add(fileExtension)) {
						loadForFileExtension(loader, location + name, "." + fileExtension, profile, filterFactory,
								consumer);
					}
				}
			}
		}
```
现在根据配置的环境拼接成配置文件路径后，就开始加载配置文件，过程不复杂，检查文件是否存在，存在的话就`loadDocuments`。
```java
		private void load(PropertySourceLoader loader, String location, Profile profile, DocumentFilter filter,
				DocumentConsumer consumer) {
			Resource[] resources = getResources(location);
			for (Resource resource : resources) {
				try {
					if (resource == null || !resource.exists()) {
						if (this.logger.isTraceEnabled()) {
							StringBuilder description = getDescription("Skipped missing config ", location, resource,
									profile);
							this.logger.trace(description);
						}
						continue;
					}
					if (!StringUtils.hasText(StringUtils.getFilenameExtension(resource.getFilename()))) {
						if (this.logger.isTraceEnabled()) {
							StringBuilder description = getDescription("Skipped empty config extension ", location,
									resource, profile);
							this.logger.trace(description);
						}
						continue;
					}
					String name = "applicationConfig: [" + getLocationName(location, resource) + "]";
					List<Document> documents = loadDocuments(loader, name, resource);
					if (CollectionUtils.isEmpty(documents)) {
						if (this.logger.isTraceEnabled()) {
							StringBuilder description = getDescription("Skipped unloaded config ", location, resource,
									profile);
							this.logger.trace(description);
						}
						continue;
					}
					List<Document> loaded = new ArrayList<>();
					for (Document document : documents) {
						if (filter.match(document)) {
							addActiveProfiles(document.getActiveProfiles());
							addIncludedProfiles(document.getIncludeProfiles());
							loaded.add(document);
						}
					}
					Collections.reverse(loaded);
					if (!loaded.isEmpty()) {
						loaded.forEach((document) -> consumer.accept(profile, document));
						if (this.logger.isDebugEnabled()) {
							StringBuilder description = getDescription("Loaded config file ", location, resource,
									profile);
							this.logger.debug(description);
						}
					}
				}
				catch (Exception ex) {
					StringBuilder description = getDescription("Failed to load property source from ", location,
							resource, profile);
					throw new IllegalStateException(description.toString(), ex);
				}
			}
		}
```
还是缓存的思想，优化从缓存中获取，没有再load()。
```java
		private List<Document> loadDocuments(PropertySourceLoader loader, String name, Resource resource)
				throws IOException {
			DocumentsCacheKey cacheKey = new DocumentsCacheKey(loader, resource);
			List<Document> documents = this.loadDocumentsCache.get(cacheKey);
			if (documents == null) {
				List<PropertySource<?>> loaded = loader.load(name, resource);
				documents = asDocuments(loaded);
				this.loadDocumentsCache.put(cacheKey, documents);
			}
			return documents;
		}
```
一路看到底吧，以properties文件为例，读取时其实就是字符串处理，读取key时读取到'='或':'为止，读取value时读取到空格符、制表符、换行符为止，然后读取完配置文件中所有配置，至此读取完配置文件，读取完用`OriginTrackedMapPropertySource`封装。
```java
	Map<String, OriginTrackedValue> load(boolean expandLists) throws IOException {
		try (CharacterReader reader = new CharacterReader(this.resource)) {
			Map<String, OriginTrackedValue> result = new LinkedHashMap<>();
			StringBuilder buffer = new StringBuilder();
			while (reader.read()) {
				String key = loadKey(buffer, reader).trim();
				if (expandLists && key.endsWith("[]")) {
					key = key.substring(0, key.length() - 2);
					int index = 0;
					do {
						OriginTrackedValue value = loadValue(buffer, reader, true);
						put(result, key + "[" + (index++) + "]", value);
						if (!reader.isEndOfLine()) {
							reader.read();
						}
					}
					while (!reader.isEndOfLine());
				}
				else {
					OriginTrackedValue value = loadValue(buffer, reader, false);
					put(result, key, value);
				}
			}
			return result;
		}
	}
    //读取key
    private String loadKey(StringBuilder buffer, CharacterReader reader) throws IOException {
		buffer.setLength(0);
		boolean previousWhitespace = false;
		while (!reader.isEndOfLine()) {
			if (reader.isPropertyDelimiter()) {
				reader.read();
				return buffer.toString();
			}
			if (!reader.isWhiteSpace() && previousWhitespace) {
				return buffer.toString();
			}
			previousWhitespace = reader.isWhiteSpace();
			buffer.append(reader.getCharacter());
			reader.read();
		}
		return buffer.toString();
	}
    //读取value
	private OriginTrackedValue loadValue(StringBuilder buffer, CharacterReader reader, boolean splitLists)
			throws IOException {
		buffer.setLength(0);
		while (reader.isWhiteSpace() && !reader.isEndOfLine()) {
			reader.read();
		}
		Location location = reader.getLocation();
		while (!reader.isEndOfLine() && !(splitLists && reader.isListDelimiter())) {
			buffer.append(reader.getCharacter());
			reader.read();
		}
		Origin origin = new TextResourceOrigin(this.resource, location);
		return OriginTrackedValue.of(buffer.toString(), origin);
	}    
```
接下来要将加载的配置文件和已有的属性源都保存到MutablePropertySources中,可以看到配置文件是放在最后的，所以配置文件的配置优先级最低。
```java
		private void addLoadedPropertySources() {
			MutablePropertySources destination = this.environment.getPropertySources();
			List<MutablePropertySources> loaded = new ArrayList<>(this.loaded.values());
			Collections.reverse(loaded);
			String lastAdded = null;
			Set<String> added = new HashSet<>();
			for (MutablePropertySources sources : loaded) {
				for (PropertySource<?> source : sources) {
                    //每个配置文件只添加一次
					if (added.add(source.getName())) {
						addLoadedPropertySource(destination, lastAdded, source);
                        //添加到上个配置文件之后
						lastAdded = source.getName();
					}
				}
			}
		}
		private void addLoadedPropertySource(MutablePropertySources destination, String lastAdded,
				PropertySource<?> source) {
			if (lastAdded == null) {
				if (destination.contains(DEFAULT_PROPERTIES)) {
					destination.addBefore(DEFAULT_PROPERTIES, source);
				}
				else {
					destination.addLast(source);
				}
			}
			else {
				destination.addAfter(lastAdded, source);
			}
		}        
```
**总结下，springboot加载配置时会先加载Java属性和操作系统环境变量，再封装命令行参数添加到属性源开头，最后根据环境读取配置文件，优先级依次为命令行参数->Java属性->操作系统环境变量->配置文件。有不对的地方请大神指出，欢迎大家一起讨论交流，共同进步，更多请关注微信公众号 葡萄开源**