



首先来看AutoConfigurationImportSelector的定义。
```java
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,  
      ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {
```
要注意的是<font color=44cf57>该类实现了DeferredImportSelector接口，而DeferredImportSelector接口的实现类需要配合@Import注解使用（通过@Import注解导入）</font>。

需要注意的是，通过@Import注解导入DeferredImportSelector接口实现类，在（Spring的ConfigurationClassPostProcessor）处理@Import注解的过程中会处理DeferredImportSelector接口实现类，但是<font color=44cf57>不会调用该selectImports()方法</font>，而是调用该类的内部类AutoConfigurationGroup的process()方法，进而直接调用该方法的主要逻辑getAutoConfigurationEntry()。

## 1、getImportGroup()

那么先来看实现的
```java
public Class<? extends Group> getImportGroup() {  
    return AutoConfigurationGroup.class;  
}
```

# 2、AutoConfigurationGroup#process()

```java
// ------------------------------- AutoConfigurationGroup --------------------------------
// Parameters:
//   - annotationMetadata: @Import(DeferredImportSelector实现类)标注的配置类的元数据
//   - deferredImportSelector: 该DeferredImportSelector实现类
public void process(AnnotationMetadata annotationMetadata, DeferredImportSelector deferredImportSelector) {  
	... // assert

	
    AutoConfigurationEntry autoConfigurationEntry = ((AutoConfigurationImportSelector) deferredImportSelector)  
         .getAutoConfigurationEntry(annotationMetadata);  
    // private final List<AutoConfigurationEntry> autoConfigurationEntries = new ArrayList<>();     
    this.autoConfigurationEntries.add(autoConfigurationEntry);  
    for (String importClassName : autoConfigurationEntry.getConfigurations()) { 
	    // private final Map<String, AnnotationMetadata> entries = new LinkedHashMap<>(); 
        this.entries.putIfAbsent(importClassName, annotationMetadata);  
    }
}
```


## getAutoConfigurationEntry()

```java
// ----------------------------- AutoConfigurationImportSelector -----------------------------
// AnnotationMetadata: 配置类的AnnotationMetadata
protected AutoConfigurationEntry getAutoConfigurationEntry(AnnotationMetadata annotationMetadata) {  
    if (!isEnabled(annotationMetadata)) {  
        return EMPTY_ENTRY;  
    }  

    // 获取指定注解的属性
	// 默认获取@EnableAutoConfiguration注解的属性
    AnnotationAttributes attributes = getAttributes(annotationMetadata);                      // 附录 -- 方法 1.1

    // 
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes); // 附录 -- 方法 1.2
    // 利用Set的性质移除重复元素, protected方法
    // return new ArrayList<>(new LinkedHashSet<>(list)); 
    configurations = removeDuplicates(configurations);                                        // 附录 -- 方法 1.3
    // 获取需要移除的元素,来源有3个:
    // - @EnableAutoConfiguration的exclude属性
    // - @EnableAutoConfiguration的excludeName属性
    // - Environment实例的spring.autoconfigure.exclude属性
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);                   // 附录 -- 方法 1.4
    // 如果exclusions中的元素不存在于configurations中,并且不能被加载,抛出异常
    checkExcludedClasses(configurations, exclusions);   // 5
    // configurations移除所有exclusions中的元素
    configurations.removeAll(exclusions);  
    configurations = getConfigurationClassFilter().filter(configurations);  // 6
    // 发布AutoConfigurationImportEvent事件
    fireAutoConfigurationImportEvents(configurations, exclusions);  // 7, 相对而言不重要
    // AutoConfigurationEntry: 对configurations和exclusions的简单封装
    // - configurations -- 移除不满足条件的配置类后所有配置类
    // - exclusions     -- 不满足条件的所有配置类
    return new AutoConfigurationEntry(configurations, exclusions);  // 附录 -- 类 1 
}
```



# 3、selectImports()

```java
// ------------------------------- AutoConfigurationGroup --------------------------------
public Iterable<Entry> selectImports() {  
	// List<AutoConfigurationEntry> autoConfigurationEntries = new ArrayList<>();     
	// process()已经向this.autoConfigurationEntries添加了元素
    if (this.autoConfigurationEntries.isEmpty()) {  
        return Collections.emptyList();  
    }

	// 
    Set<String> allExclusions = this.autoConfigurationEntries.stream()   // Stream<AutoConfigurationEntry>
	    .map(AutoConfigurationEntry::getExclusions)                      // Stream<Set<String>>
        .flatMap(Collection::stream)                                     // Stream<String> 
    .collect(Collectors.toSet());                                        // Set<String>

	// 获取
    Set<String> processedConfigurations = this.autoConfigurationEntries.stream()  
        .map(AutoConfigurationEntry::getConfigurations)
        .flatMap(Collection::stream)  
    .collect(Collectors.toCollection(LinkedHashSet::new));  

	processedConfigurations.removeAll(allExclusions);  

    return sortAutoConfigurations(
	    processedConfigurations, 
	    getAutoConfigurationMetadata())
	.stream()  
    .map((importClassName) -> new Entry(this.entries.get(importClassName), importClassName))  
    .collect(Collectors.toList());  // List<Entry>
}

private List<String> sortAutoConfigurations(Set<String> configurations, AutoConfigurationMetadata autoConfigurationMetadata) {  
    return new AutoConfigurationSorter(getMetadataReaderFactory(), autoConfigurationMetadata)  
         .getInPriorityOrder(configurations);  
}
```



# 附录 -- 方法

## 1、getAttributes()
```java
protected AnnotationAttributes getAttributes(AnnotationMetadata metadata) {  
	// getAnnotationClass(): 默认返回 EnableAutoConfiguration.class
    String name = getAnnotationClass().getName();  
    AnnotationAttributes attributes = AnnotationAttributes.fromMap(metadata.getAnnotationAttributes(name, true));  
    ... // Assert
    return attributes;  
}
```

## 2、（重要）getCandidateConfigurations()
```java
/* ------------------------------ AutoConfigurationImportSelector ---------------------------------------- 
 * 
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {  

    List<String> configurations = ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())  
        .getCandidates();  

	... // Assert
    return configurations;
}

/* ------------------------------ AutoConfigurationImportSelector ---------------------------------------- 
 * 
 */
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, AnnotationAttributes attributes) {  
	// getSpringFactoriesLoaderFactoryClass(): 返回EnableAutoConfiguration.class,protected方法
	// 从META-INF/spring.factories文件中找到
    List<String> configurations = new ArrayList<>(  
		SpringFactoriesLoader.loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));
	// 从 META-INF/spring/full-qualified-annotation-name.imports 文件中获取 import candidates
	// - 文件中的每一行都是全限定名
	// - # 表示注释
    ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader()).forEach(configurations::add);  
    ...
    return configurations;  
}
```
## 3、removeDuplicates()
```java
protected final <T> List<T> removeDuplicates(List<T> list) {  
    return new ArrayList<>(new LinkedHashSet<>(list));  
}
```
## 4、getExclusions()
```java
// AnnotationMetadata: 默认Springboot配置类的AnnotationMetadata
// AnnotationAttributes: 默认@EnableAutoConfiguration的属性
protected Set<String> getExclusions(AnnotationMetadata metadata, AnnotationAttributes attributes) { 
   Set<String> excluded = new LinkedHashSet<>();  
   // 获取@EnableAutoConfiguration的exclude属性
   excluded.addAll(asList(attributes, "exclude"));  
   // 获取@EnableAutoConfiguration的excludeName属性
   excluded.addAll(Arrays.asList(attributes.getStringArray("excludeName")));  
   // 获取Environment中spring.autoconfigure.exclude的值
   excluded.addAll(getExcludeAutoConfigurationsProperty());  
   return excluded;  
}


```
## 4.1、asList()
```java
protected final List<String> asList(AnnotationAttributes attributes, String name) {  
    String[] value = attributes.getStringArray(name);  
    return Arrays.asList(value);  
}
```
## 4.2、getExcludeAutoConfigurationsProperty()
```java
protected List<String> getExcludeAutoConfigurationsProperty() {  
   Environment environment = getEnvironment();  
   if (environment == null) {  
      return Collections.emptyList();  
   }  
   if (environment instanceof ConfigurableEnvironment) {  
      Binder binder = Binder.get(environment);  
      return binder.bind(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class)  
         .map(Arrays::asList)  
         .orElse(Collections.emptyList());  
   }  
   String[] excludes = environment.getProperty(PROPERTY_NAME_AUTOCONFIGURE_EXCLUDE, String[].class);  
   return (excludes != null) ? Arrays.asList(excludes) : Collections.emptyList();  
}
```
