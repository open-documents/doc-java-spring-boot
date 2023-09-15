

Springboot应用从下面的类启动。
```java
@SpringBootApplication  
public class SpringbootLearningApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(SpringbootLearningApplication.class, args);  
    }  
  
}
```
下面就介绍run()静态方法。run的实际逻辑是这样的。
```java
public static ConfigurableApplicationContext run(Class<?> primarySource, String... args) {  
    return run(new Class<?>[] { primarySource }, args);  
}
public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {  
    return new SpringApplication(primarySources).run(args);  
}
```
-- --
# 一、构造函数

```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 */
public SpringApplication(Class<?>... primarySources) {  
    this(null, primarySources);  
}
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {  
    this.resourceLoader = resourceLoader;  
    // 1. this.primarySources:
    //   - 类型为Set<Class<?>>
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    // 检查类路径下某些特定的类是否存在, 判断该应用的类型, 类型包含3种:
    // - 普通Spring应用
    // - Springmvc应用
    // - SpringReactive应用
    this.webApplicationType = WebApplicationType.deduceFromClasspath();        // 方法 1

	// 后面代码涉及到getSpringFactoriesInstances()函数, 主要功能是:
	// - 从 "META-INF/spring.factories" 中获取接口的实现类全路径, 然后依次实例化这些类

	// 
    this.bootstrapRegistryInitializers = new ArrayList<>(  
	    getSpringFactoriesInstances(BootstrapRegistryInitializer.class));
	// setInitializers(Collection<? extends ApplicationContextInitializer<?>>):
	// - 设置this.initializers()
	// - this.initializers = new ArrayList<>(initializers);
	// - 默认7个(后续使用this.initializers再列出)
    setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));  
    // setListeners(Collection<? extends ApplicationListener<?>>)
    // - 设置this.listeners
    // - this.listeners = new ArrayList<>(listeners);
    // - 默认8个(后续使用this.listeners再列出)
    setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));      
	// this.mainApplicationClass:
	// - 类型为 Class<?>
    this.mainApplicationClass = deduceMainApplicationClass();                  // 方法 2 
}
```
## 方法 1 -- deduceFromClasspath()
```java
/* ---------------------------------------- WebApplicationType ----------------------------------------
 */
static WebApplicationType deduceFromClasspath() {  
	// - WEBFLUX_INDICATOR_CLASS = "org.springframework.web.reactive.DispatcherHandler"
	// - WEBMVC_INDICATOR_CLASS  = "org.springframework.web.servlet.DispatcherServlet"
	// - JERSEY_INDICATOR_CLASS  = "org.glassfish.jersey.servlet.ServletContainer"
    if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) 
	    && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)  
        && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {  
        return WebApplicationType.REACTIVE;  
    }
    // SERVLET_INDICATOR_CLASSES = { "jakarta.servlet.Servlet",  
    //                               "org.springframework.web.context.ConfigurableWebApplicationContext" }
    for (String className : SERVLET_INDICATOR_CLASSES) {  
        if (!ClassUtils.isPresent(className, null)) {  
            return WebApplicationType.NONE;  
        }  
    }
    return WebApplicationType.SERVLET;  
}
```

## 方法 2 -- deduceMainApplicationClass()
```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 */
private Class<?> deduceMainApplicationClass() {  
    return StackWalker.getInstance(StackWalker.Option.RETAIN_CLASS_REFERENCE)  
        .walk(this::findMainClass)  
        .orElse(null);  
}
/* ---------------------------------------- SpringApplication ----------------------------------------
 */
private Optional<Class<?>> findMainClass(Stream<StackFrame> stack) {  
    return stack.filter((frame) -> Objects.equals(frame.getMethodName(), "main"))  
        .findFirst()  
        .map(StackWalker.StackFrame::getDeclaringClass);  
}
```
# 二、SpringApplication#run()

```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 */
public ConfigurableApplicationContext run(String... args) {  
	... // time 

	// 1. 使用DefaultBootstrapContext()无参构造新建一个实例
	// 2. 使用this.bootstrapRegistryInitializers(SpringApplication构造函数中赋值过)对上述实例进行初始化
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();         // 方法 1


    ConfigurableApplicationContext context = null;  
    
    // 如果系统属性java.awt.headless键没有设置过, 则使用this.headless(boolean,默认值为true,支持set())设置
    configureHeadlessProperty();                                                 // 方法 2

    SpringApplicationRunListeners listeners = getRunListeners(args);             // 方法 3
	listeners.starting(bootstrapContext, this.mainApplicationClass);  

    try {  
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);  
        ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);                                                           // 方法 4
		// 根据Environment的"spring.beaninfo.ignore"值设置系统属性"spring.beaninfo.ignore"
        configureIgnoreBeanInfo(environment);                                    // 方法 5

        // 
        Banner printedBanner = printBanner(environment);  


        context = createApplicationContext();  
        context.setApplicationStartup(this.applicationStartup);  
        prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);  
        refreshContext(context);
        // 默认的afterRefresh是protected NoOp,可以通过重写来变更默认的NoOp行为
        afterRefresh(context, applicationArguments);  
        
        ... // time
        if (this.logStartupInfo) {  
            new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), timeTakenToStartup);  
        }  
        listeners.started(context, timeTakenToStartup);  
        callRunners(context, applicationArguments);  
    }  
    catch (Throwable ex) {  
        handleRunFailure(context, ex, listeners);  
        throw new IllegalStateException(ex);  
    }  
    try {  
        Duration timeTakenToReady = Duration.ofNanos(System.nanoTime() - startTime);  
        listeners.ready(context, timeTakenToReady);  
    }  
    catch (Throwable ex) {  
        handleRunFailure(context, ex, null);  
        throw new IllegalStateException(ex);  
    }  
    return context;  
}
```

## 方法1 -- createBootstrapContext()
```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 * version: 3.1.0
 */
private DefaultBootstrapContext createBootstrapContext() {  
    DefaultBootstrapContext bootstrapContext = new DefaultBootstrapContext();  
    // - this.bootstrapRegistryInitializers已经在SpringApplication构造函数中赋值过
    // - 赋的值为 "META-INF/spring.factories" 中 BootstrapRegistryInitializer.class接口的实现类
    // - 默认0个
    this.bootstrapRegistryInitializers.forEach((initializer)->initializer.initialize(bootstrapContext));  
    return bootstrapContext;  
}
```
## 方法 2 -- configureHeadlessProperty()
```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 * version: 3.1.0
 */
private void configureHeadlessProperty() {  
	// 
	// 1. SYSTEM_PROPERTY_JAVA_AWT_HEADLESS = "java.awt.headless"
	// 2. this.headless默(初)值为true, 支持set()
    System.setProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS,  
         System.getProperty(SYSTEM_PROPERTY_JAVA_AWT_HEADLESS, Boolean.toString(this.headless)));  
}
```
## 方法 3 -- getRunListeners()
```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 * version: 3.1.0
 */
private SpringApplicationRunListeners getRunListeners(String[] args) {  
    ArgumentResolver argumentResolver = ArgumentResolver.of(SpringApplication.class, this);  
    argumentResolver = argumentResolver.and(String[].class, args);  
    List<SpringApplicationRunListener> listeners = getSpringFactoriesInstances(SpringApplicationRunListener.class,  
         argumentResolver);  
	// this.applicationHook:
	// - 类型为 ThreadLocal<SpringApplicationHook>
	// - 默认(初)值为 new ThreadLocal<>()
    SpringApplicationHook hook = applicationHook.get();  
    SpringApplicationRunListener hookListener = (hook != null) ? hook.getRunListener(this) : null;  
    if (hookListener != null) {  
        listeners = new ArrayList<>(listeners);  
        listeners.add(hookListener);  
    }
    return new SpringApplicationRunListeners(logger, listeners, this.applicationStartup);  
}
```
## 方法 4 -- prepareEnvironment()
```java
/* ---------------------------------------- SpringApplication ----------------------------------------
 * version: 3.1.0
 */
private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,  
DefaultBootstrapContext bootstrapContext, ApplicationArguments applicationArguments) {  
    // Create and configure the environment  
    ConfigurableEnvironment environment = getOrCreateEnvironment();           // 1
    // 1.根据addConversionService(boolean)属性,设置Environment.ConfigurablePropertyResolver.ConversionService
    configureEnvironment(environment, applicationArguments.getSourceArgs());  // 2
    ConfigurationPropertySources.attach(environment);  // 3
    listeners.environmentPrepared(bootstrapContext, environment);  
    DefaultPropertiesPropertySource.moveToEnd(environment);  // 4
   
    bindToSpringApplication(environment);  // 5
    if (!this.isCustomEnvironment) {  
        EnvironmentConverter environmentConverter = new EnvironmentConverter(getClassLoader());  
        // 将给定的Environment转换成StandardEnvironment
        environment = environmentConverter.convertEnvironmentIfNecessary(environment, deduceEnvironmentClass());  
    }  
    ConfigurationPropertySources.attach(environment);  
    return environment;  
}
```
### 相关方法--getOrCreateEnvironment() 
```java
// ----------------------------------- 1 -----------------------------
private ConfigurableEnvironment getOrCreateEnvironment() {  
    if (this.environment != null) {  
        return this.environment;  
    }  
    switch (this.webApplicationType) {  
	    case SERVLET:  
            return new ApplicationServletEnvironment();  
	    case REACTIVE:  
            return new ApplicationReactiveWebEnvironment();  
        default:  
            return new ApplicationEnvironment();  
    }  
}
```
### 相关方法--configureEnvironment()
```java
// ----------------------------------- 2 -----------------------------
protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {  
	if (this.addConversionService) {  
        environment.setConversionService(new ApplicationConversionService());  
    }  
    configurePropertySources(environment, args);  
    configureProfiles(environment, args);  
}
protected void configurePropertySources(ConfigurableEnvironment environment, String[] args) {  
   MutablePropertySources sources = environment.getPropertySources();  
   if (!CollectionUtils.isEmpty(this.defaultProperties)) {  
      DefaultPropertiesPropertySource.addOrMerge(this.defaultProperties, sources);  
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
protected void configureProfiles(ConfigurableEnvironment environment, String[] args) {  
}
```
### 相关方法--bindToSpringApplication()
```java
protected void bindToSpringApplication(ConfigurableEnvironment environment) {  
   try {  
      Binder.get(environment).bind("spring.main", Bindable.ofInstance(this));  
   }  
   catch (Exception ex) {  
      throw new IllegalStateException("Cannot bind to SpringApplication", ex);  
   }  
}
```
## 相关方法--configureIgnoreBeanInfo()
```java
private void configureIgnoreBeanInfo(ConfigurableEnvironment environment) {  
   if (System.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME) == null) {  
      Boolean ignore = environment.getProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME,  
            Boolean.class, Boolean.TRUE);  
      System.setProperty(CachedIntrospectionResults.IGNORE_BEANINFO_PROPERTY_NAME, ignore.toString());  
   }  
}
```
## 相关方法--printBanner()
```java
private Banner printBanner(ConfigurableEnvironment environment) {  
   if (this.bannerMode == Banner.Mode.OFF) {  
      return null;  
   }  
   ResourceLoader resourceLoader = (this.resourceLoader != null) ? this.resourceLoader  
         : new DefaultResourceLoader(null);  
   SpringApplicationBannerPrinter bannerPrinter = new SpringApplicationBannerPrinter(resourceLoader, this.banner);  
   if (this.bannerMode == Mode.LOG) {  
      return bannerPrinter.print(environment, this.mainApplicationClass, logger);  
   }  
   return bannerPrinter.print(environment, this.mainApplicationClass, System.out);  
}
```
## 相关方法--createApplicationContext()
```java
// ApplicationContextFactory applicationContextFactory = ApplicationContextFactory.DEFAULT;
protected ConfigurableApplicationContext createApplicationContext() {  
    return this.applicationContextFactory.create(this.webApplicationType);  
}
```
## 相关方法--prepareContext()
```java
private void prepareContext(DefaultBootstrapContext bootstrapContext, ConfigurableApplicationContext context,  
      ConfigurableEnvironment environment, SpringApplicationRunListeners listeners,  
      ApplicationArguments applicationArguments, Banner printedBanner) {  
    context.setEnvironment(environment);  
    postProcessApplicationContext(context);  
    applyInitializers(context);  
    listeners.contextPrepared(context);  
    bootstrapContext.close(context);  
    if (this.logStartupInfo) {  
      logStartupInfo(context.getParent() == null);  
      logStartupProfileInfo(context);  
   }  
   // Add boot specific singleton beans  
   ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();  
   beanFactory.registerSingleton("springApplicationArguments", applicationArguments);  
   if (printedBanner != null) {  
      beanFactory.registerSingleton("springBootBanner", printedBanner);  
   }  
   if (beanFactory instanceof AbstractAutowireCapableBeanFactory) {  
      ((AbstractAutowireCapableBeanFactory) beanFactory).setAllowCircularReferences(this.allowCircularReferences);  
      if (beanFactory instanceof DefaultListableBeanFactory) {  
         ((DefaultListableBeanFactory) beanFactory)  
               .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);  
      }  
   }  
   if (this.lazyInitialization) {  
      context.addBeanFactoryPostProcessor(new LazyInitializationBeanFactoryPostProcessor());  
   }  
   context.addBeanFactoryPostProcessor(new PropertySourceOrderingBeanFactoryPostProcessor(context));  
   // Load the sources  
   Set<Object> sources = getAllSources();  
   Assert.notEmpty(sources, "Sources must not be empty");  
   load(context, sources.toArray(new Object[0]));  
   listeners.contextLoaded(context);  
}
```
### 相关方法--postProcessApplicationContext()
```java
protected void postProcessApplicationContext(ConfigurableApplicationContext context) {  
   if (this.beanNameGenerator != null) {  
      context.getBeanFactory().registerSingleton(AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,  
            this.beanNameGenerator);  
   }  
   if (this.resourceLoader != null) {  
      if (context instanceof GenericApplicationContext) {  
         ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);  
      }  
      if (context instanceof DefaultResourceLoader) {  
         ((DefaultResourceLoader) context).setClassLoader(this.resourceLoader.getClassLoader());  
      }  
   }  
   if (this.addConversionService) {  
      context.getBeanFactory().setConversionService(context.getEnvironment().getConversionService());  
   }  
}
```
### 相关方法--applyInitializers()
```java
protected void applyInitializers(ConfigurableApplicationContext context) {  
   for (ApplicationContextInitializer initializer : getInitializers()) {  
      Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),  
            ApplicationContextInitializer.class);  
      Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");  
      initializer.initialize(context);  
   }  
}
```
### 相关方法--getAllSources()
```java
public Set<Object> getAllSources() {  
   Set<Object> allSources = new LinkedHashSet<>();  
   if (!CollectionUtils.isEmpty(this.primarySources)) {  
      allSources.addAll(this.primarySources);  
   }  
   if (!CollectionUtils.isEmpty(this.sources)) {  
      allSources.addAll(this.sources);  
   }  
   return Collections.unmodifiableSet(allSources);  
}
```
### 相关方法--load()
```java
protected void load(ApplicationContext context, Object[] sources) {  
   if (logger.isDebugEnabled()) {  
      logger.debug("Loading source " + StringUtils.arrayToCommaDelimitedString(sources));  
   }  
   BeanDefinitionLoader loader = createBeanDefinitionLoader(getBeanDefinitionRegistry(context), sources);  
   if (this.beanNameGenerator != null) {  
      loader.setBeanNameGenerator(this.beanNameGenerator);  
   }  
   if (this.resourceLoader != null) {  
      loader.setResourceLoader(this.resourceLoader);  
   }  
   if (this.environment != null) {  
      loader.setEnvironment(this.environment);  
   }  
   loader.load();  
}
protected BeanDefinitionLoader createBeanDefinitionLoader(BeanDefinitionRegistry registry, Object[] sources) {  
    return new BeanDefinitionLoader(registry, sources);  
}
```


## 相关方法--refreshContext()
```java
private void refreshContext(ConfigurableApplicationContext context) {  
   if (this.registerShutdownHook) {  
      shutdownHook.registerApplicationContext(context);  
   }  
   refresh(context);  
}

protected void refresh(ConfigurableApplicationContext applicationContext) {  
    applicationContext.refresh();  
}
```
## 相关方法--afterRefresh()
```java
protected void afterRefresh(ConfigurableApplicationContext context, ApplicationArguments args) {  
}
```


## 相关方法--callRunners()
```java
private void callRunners(ApplicationContext context, ApplicationArguments args) {  
   List<Object> runners = new ArrayList<>();  
   runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());  
   runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());  
   AnnotationAwareOrderComparator.sort(runners);  
   for (Object runner : new LinkedHashSet<>(runners)) {  
      if (runner instanceof ApplicationRunner) {  
         callRunner((ApplicationRunner) runner, args);  
      }  
      if (runner instanceof CommandLineRunner) {  
         callRunner((CommandLineRunner) runner, args);  
      }  
   }  
}
```




## 相关方法--loadFactoryNames()
```java
// ------------------------------------ SpringFactoriesLoader ----------------------------------------
public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {  
    ClassLoader classLoaderToUse = classLoader;  
    if (classLoaderToUse == null) {  
        classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();  
    }  
    String factoryTypeName = factoryType.getName();  
    return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());  
}  

private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {  
	// 当启动SpringApplication时没有传入类加载器,那么此处的类加载器就是AppClassLoader
    Map<String, List<String>> result = cache.get(classLoader);  
    if (result != null) {  
        return result;  
    }  
  
    result = new HashMap<>();  
    try {  
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);  
        while (urls.hasMoreElements()) {  
            URL url = urls.nextElement();  
            UrlResource resource = new UrlResource(url);  
            Properties properties = PropertiesLoaderUtils.loadProperties(resource);  
            for (Map.Entry<?, ?> entry : properties.entrySet()) {  
                String factoryTypeName = ((String) entry.getKey()).trim();  
                String[] factoryImplementationNames =  
	                StringUtils.commaDelimitedListToStringArray((String) entry.getValue());  
	            for (String factoryImplementationName : factoryImplementationNames) {  
	                result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())  
		                .add(factoryImplementationName.trim());  
                }  
            }  
        }  
	    // Replace all lists with unmodifiable lists containing unique elements  
        result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()  
            .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));  
        cache.put(classLoader, result);  
    }  
    catch (IOException ex) {  
        throw new IllegalArgumentException(...);  
    }  
    return result;  
}
```
## 相关方法--createSpringFactoriesInstances()
```java
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,  
      ClassLoader classLoader, Object[] args, Set<String> names) {  
      
    List<T> instances = new ArrayList<>(names.size());  
    for (String name : names) {  
        try {  
	        // 首先,加载类文件
            Class<?> instanceClass = ClassUtils.forName(name, classLoader);  
            Assert.isAssignable(type, instanceClass);  
            // 接着,获取构造函数
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);  
            T instance = (T) BeanUtils.instantiateClass(constructor, args);  
            instances.add(instance);  
        }  
        catch (Throwable ex) {  
            throw new IllegalArgumentException(...);  
        }  
    }  
    return instances;  
}


```





# 其它方法
## getSpringFactoriesInstances()
```java
private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes, Object... args) {  
    ClassLoader classLoader = getClassLoader();  
    // Use names and ensure unique to protect against duplicates  
    // 从META-INF/spring.factories中获取type的实现类的全限定名
    Set<String> names = new LinkedHashSet<>(SpringFactoriesLoader.loadFactoryNames(type, classLoader));  
    List<T> instances = createSpringFactoriesInstances(type, parameterTypes, classLoader, args, names);  
    AnnotationAwareOrderComparator.sort(instances);  
    return instances;  
}
private <T> List<T> createSpringFactoriesInstances(Class<T> type, Class<?>[] parameterTypes,  
      ClassLoader classLoader, Object[] args, Set<String> names) {  
    List<T> instances = new ArrayList<>(names.size());  
    for (String name : names) {  
	    try {  
			Class<?> instanceClass = ClassUtils.forName(name, classLoader);  
            ... // assert
            Constructor<?> constructor = instanceClass.getDeclaredConstructor(parameterTypes);  
            T instance = (T) BeanUtils.instantiateClass(constructor, args);  
            instances.add(instance);  
        }  
        catch (Throwable ex) {...}  
    }  
    return instances;  
}
```