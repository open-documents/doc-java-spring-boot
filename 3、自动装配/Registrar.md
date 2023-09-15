
前置知识：
- Spring如何处理@Import注解

Registrar是AutoConfigurationPackages类的内部类，先来看AutoConfigurationPackages$Registrar的定义。
```java
static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {}
```
可以看到Registrar实现了ImportBeanDefinitionRegistrar接口，ImportBeanDefinitionRegistrar接口（和ImportSelector接口、DeferredImportSelector接口）通过@Import导入时会在处理@Import的过程中得到处理，也就是说这个类需要通过 @Import 导入。

# 1、registerBeanDefinitions()

```java
// ------------------ Registrar(AutoConfigurationPackages内部类) ---------------------
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {  
    register(registry, new PackageImports(metadata).getPackageNames().toArray(new String[0]));  
}

// ------------------------ AutoConfigurationPackages ------------------------
// 参数:
//   - registry: 
//   - packageNames: 
public static void register(BeanDefinitionRegistry registry, String... packageNames) {  
	// BEAN: String "org.springframework.boot.autoconfigure.AutoConfigurationPackages"
    if (registry.containsBeanDefinition(BEAN)) {  
        BasePackagesBeanDefinition beanDefinition = (BasePackagesBeanDefinition) registry.getBeanDefinition(BEAN);  
        beanDefinition.addBasePackages(packageNames);
    }
    else {  // 默认走这
        registry.registerBeanDefinition(BEAN, new BasePackagesBeanDefinition(packageNames));  
    }  
}
```
## 1.1、涉及的类--PackageImports

PackageImports只有一个属性。
```java
private final List<String> packageNames;  // 支持get()
```

### 构造函数（重要）

PackageImports只提供一个构造方法。
```java
// ------------------------------ PackageImports -----------------------------
PackageImports(AnnotationMetadata metadata) {  
	// 获取@AutoConfigurationPackage注解属性
    AnnotationAttributes attributes = AnnotationAttributes      .fromMap(metadata.getAnnotationAttributes(AutoConfigurationPackage.class.getName(), false));  
    // 获取@AutoConfigurationPackage注解basePackages属性,默认为空
    List<String> packageNames = new ArrayList<>(Arrays.asList(attributes.getStringArray("basePackages")));  
    // 遍历@AutoConfigurationPackage注解basePackageClasses属性,默认为空
    for (Class<?> basePackageClass : attributes.getClassArray("basePackageClasses")) {  
        packageNames.add(basePackageClass.getPackage().getName());  
    }  
    if (packageNames.isEmpty()) {  // 默认走这
	    // 获取Springboot应用类所在的包
        packageNames.add(ClassUtils.getPackageName(metadata.getClassName()));  
    }  
    this.packageNames = Collections.unmodifiableList(packageNames);  
}
```