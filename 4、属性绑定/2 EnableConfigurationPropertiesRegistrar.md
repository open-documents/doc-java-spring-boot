
首先看该类的定义。
```java
class EnableConfigurationPropertiesRegistrar implements ImportBeanDefinitionRegistrar
```

# registerBeanDefinitions()
```java
/* ---------------------- EnableConfigurationPropertiesRegistrar ----------------------
 */
public void registerBeanDefinitions(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {  
	// 注册两个特别重要的类到BeanDefinitionRegistry中
	//   - ConfigurationPropertiesBindingPostProcessor
	//   - BoundConfigurationProperties
    registerInfrastructureBeans(registry);  
    registerMethodValidationExcludeFilter(registry);  
    ConfigurationPropertiesBeanRegistrar beanRegistrar = new ConfigurationPropertiesBeanRegistrar(registry);  
    getTypes(metadata).forEach(beanRegistrar::register);  
}
```

## 相关方法--registerInfrastructureBeans()
```java
/* ---------------------- EnableConfigurationPropertiesRegistrar ----------------------
 
 */
static void registerInfrastructureBeans(BeanDefinitionRegistry registry) {  
	// 向BeanDefinitionRegistry中注册一个ConfigurationPropertiesBindingPostProcessor,如果没有的话
    ConfigurationPropertiesBindingPostProcessor.register(registry);  
    // 向BeanDefinitionRegistry中注册一个BoundConfigurationProperties,如果没有的话
    BoundConfigurationProperties.register(registry);  
}
```
相关方法-- 
```java
/* ---------------------- EnableConfigurationPropertiesRegistrar ----------------------
 */
static void registerMethodValidationExcludeFilter(BeanDefinitionRegistry registry) {  
   if (!registry.containsBeanDefinition(METHOD_VALIDATION_EXCLUDE_FILTER_BEAN_NAME)) {  
      BeanDefinition definition = BeanDefinitionBuilder  
         .rootBeanDefinition(MethodValidationExcludeFilter.class, "byAnnotation")  
         .addConstructorArgValue(ConfigurationProperties.class)  
         .setRole(BeanDefinition.ROLE_INFRASTRUCTURE)  
         .getBeanDefinition();  
      registry.registerBeanDefinition(METHOD_VALIDATION_EXCLUDE_FILTER_BEAN_NAME, definition);  
   }  
}
```

```java
private Set<Class<?>> getTypes(AnnotationMetadata metadata) {  
   return metadata.getAnnotations()  
      .stream(EnableConfigurationProperties.class)  
      .flatMap((annotation) -> Arrays.stream(annotation.getClassArray(MergedAnnotation.VALUE)))  
      .filter((type) -> void.class != type)  
      .collect(Collectors.toSet());  
}
```