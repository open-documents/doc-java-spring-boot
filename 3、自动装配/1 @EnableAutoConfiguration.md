
下面是@EnableAutoConfiguration注解的定义。
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@AutoConfigurationPackage  
@Import(AutoConfigurationImportSelector.class)  
public @interface EnableAutoConfiguration {
	String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
	Class<?>[] exclude() default {};
	String[] excludeName() default {};
}
```

# 其它方法--getExcludeAutoConfigurationsProperty()
```java
protected List<String> getExcludeAutoConfigurationsProperty() {  
   Environment environment = getEnvironment();  
   if (environment == null) {  
      return Collections.emptyList();  
   }  
   if (environment instanceof ConfigurableEnvironment) {  
      Binder binder = Binder.get(environment);  
      // PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE = spring.autoconfigure.exclude
      return binder.bind(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class).map(Arrays::asList)  
            .orElse(Collections.emptyList());  
   }  
   String[] excludes = environment.getProperty(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class);  
   return (excludes != null) ? Arrays.asList(excludes) : Collections.emptyList();  
}
```
# 其它方法--invokeAwareMethods()
```java
private void invokeAwareMethods(Object instance) {  
    if (instance instanceof Aware) {  
		if (instance instanceof BeanClassLoaderAware) {  
            ((BeanClassLoaderAware) instance).setBeanClassLoader(this.beanClassLoader);  
        }  
        if (instance instanceof BeanFactoryAware) {  
            ((BeanFactoryAware) instance).setBeanFactory(this.beanFactory);  
        }  
        if (instance instanceof EnvironmentAware) {  
            ((EnvironmentAware) instance).setEnvironment(this.environment);  
        }  
        if (instance instanceof ResourceLoaderAware) {  
            ((ResourceLoaderAware) instance).setResourceLoader(this.resourceLoader);  
        }  
    }  
}
```



# 涉及的类--ConfigurationClassFilter

```java
private final AutoConfigurationMetadata autoConfigurationMetadata;  
private final List<AutoConfigurationImportFilter> filters;  

// new ConfigurationClassFilter(this.beanClassLoader, filters)
ConfigurationClassFilter(ClassLoader classLoader, List<AutoConfigurationImportFilter> filters) {  
    this.autoConfigurationMetadata = AutoConfigurationMetadataLoader.loadMetadata(classLoader);  
    this.filters = filters;  
}
```
这个类只有下面一个方法。
## filter()
```java
List<String> filter(List<String> configurations) {  
    ...
    String[] candidates = StringUtils.toStringArray(configurations);  
    boolean skipped = false;  
    // 对每一个将要实现自动装配的类进行条件检测
    // 默认的this.filters包含3个:,,OnWebApplicationCondition
    // - OnBeanCondition: 根据将要自动装配的类上的@ConditionalOnBean,检测是否存在指定的bean
    // - OnClassCondition: 根据将要自动装配的类上的@ConditionalOnClass,检测是否存在指定的类
    // - OnWebApplicationCondition: 根据将要自动装配的类上的@ConditionalOnWebApplication,检测是否存在指定的WebApplication
    for (AutoConfigurationImportFilter filter : this.filters) {  
	    boolean[] match = filter.match(candidates, this.autoConfigurationMetadata);  
        for (int i = 0; i < match.length; i++) {  
            if (!match[i]) {  // 
                candidates[i] = null;  
                skipped = true;  
            }  
        }  
    }  
    if (!skipped) {  
        return configurations;  
    }
    List<String> result = new ArrayList<>(candidates.length);  
    for (String candidate : candidates) {  
        if (candidate != null) {  
            result.add(candidate);  
        }  
    }  
    ...
    return result;  
}
```


# 涉及的类--ConditionEvaluationReportAutoConfigurationImportListener

只有下面一个方法。
## onAutoConfigurationImportEvent()
```java
public void onAutoConfigurationImportEvent(AutoConfigurationImportEvent event) {  
    if (this.beanFactory != null) {  
        ConditionEvaluationReport report = ConditionEvaluationReport.get(this.beanFactory);  
        report.recordEvaluationCandidates(event.getCandidateConfigurations());  
        report.recordExclusions(event.getExclusions());  
    }  
}
```


## checkExcludedClasses()
```java
private void checkExcludedClasses(List<String> configurations, Set<String> exclusions) {  
    List<String> invalidExcludes = new ArrayList<>(exclusions.size());  
    for (String exclusion : exclusions) {  
        if (ClassUtils.isPresent(exclusion, getClass().getClassLoader()) && !configurations.contains(exclusion)){ 
	        invalidExcludes.add(exclusion);  
        }  
    }  
    if (!invalidExcludes.isEmpty()) {  // 如果不为空,抛出异常
        handleInvalidExcludes(invalidExcludes);  
    }  
}

protected void handleInvalidExcludes(List<String> invalidExcludes) {  
    StringBuilder message = new StringBuilder();  
    for (String exclude : invalidExcludes) {  
        message.append("\t- ").append(exclude).append(String.format("%n"));  
    }  
    throw new IllegalStateException(String.format(  
         "The following classes could not be excluded because they are not auto-configuration classes:%n%s",  
         message));  
}
```
## getConfigurationClassFilter
```java
private ConfigurationClassFilter getConfigurationClassFilter() {  
	// 默认为null
    if (this.configurationClassFilter == null) {  
        // return SpringFactoriesLoader.loadFactories(AutoConfigurationImportFilter.class, this.beanClassLoader);
        // 默认获取到下面3个:
        // - org.springframework.boot.autoconfigure.condition.OnBeanCondition
        // - org.springframework.boot.autoconfigure.condition.OnClassCondition
        // - org.springframework.boot.autoconfigure.condition.OnWebApplicationCondition
        List<AutoConfigurationImportFilter> filters = getAutoConfigurationImportFilters();  
        for (AutoConfigurationImportFilter filter : filters) {
	        // 设置AutoConfigurationImportFilter实例中的各种Aware属性
	        // - BeanClassLoaderAware
	        // - BeanFactoryAware
	        // - EnvironmentAware
	        // - ResourceLoaderAware
            invokeAwareMethods(filter);  
        }  
        this.configurationClassFilter = new ConfigurationClassFilter(this.beanClassLoader, filters);  
    }  
    return this.configurationClassFilter;  
}
```
## fireAutoConfigurationImportEvents()
```java
private void fireAutoConfigurationImportEvents(List<String> configurations, Set<String> exclusions) {  
	// return SpringFactoriesLoader.loadFactories(AutoConfigurationImportListener.class, this.beanClassLoader);
	// 默认ConditionEvaluationReportAutoConfigurationImportListener
    List<AutoConfigurationImportListener> listeners = getAutoConfigurationImportListeners();  
    if (!listeners.isEmpty()) {  
        AutoConfigurationImportEvent event = new AutoConfigurationImportEvent(this, configurations, exclusions);  
        for (AutoConfigurationImportListener listener : listeners) {  
            invokeAwareMethods(listener);  
            listener.onAutoConfigurationImportEvent(event);  
        }  
    }  
}
```

# 附录 -- 类

## 1、AutoConfigurationEntry

该类只是对需要自动装配的类的列表和不满足条件的欲自动装配的列表进行极简单的封装。

下面是它的定义。
```java
private final List<String> configurations;    // 支持get()
private final Set<String> exclusions;         // 支持get()

private AutoConfigurationEntry() {  
    this.configurations = Collections.emptyList();  
    this.exclusions = Collections.emptySet();  
}
AutoConfigurationEntry(Collection<String> configurations, Collection<String> exclusions) {  
    this.configurations = new ArrayList<>(configurations);  
    this.exclusions = new HashSet<>(exclusions);  
}
```

## AutoConfigurationSorter

```java
List<String> getInPriorityOrder(Collection<String> classNames) {  
   AutoConfigurationClasses classes = new AutoConfigurationClasses(this.metadataReaderFactory,  
         this.autoConfigurationMetadata, classNames);  
   List<String> orderedClassNames = new ArrayList<>(classNames);  
   // Initially sort alphabetically  
   Collections.sort(orderedClassNames);  
   // Then sort by order  
   orderedClassNames.sort((o1, o2) -> {  
      int i1 = classes.get(o1).getOrder();  
      int i2 = classes.get(o2).getOrder();  
      return Integer.compare(i1, i2);  
   });  
   // Then respect @AutoConfigureBefore @AutoConfigureAfter  
   orderedClassNames = sortByAnnotation(classes, orderedClassNames);  
   return orderedClassNames;  
}
```
