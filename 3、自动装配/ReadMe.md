

下面是@SpringBootApplication注解的定义。
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@SpringBootConfiguration  
@EnableAutoConfiguration  
@ComponentScan(
	excludeFilters = { 
		@Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),  
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
	}
)  
public @interface SpringBootApplication {}
```

下面对它的元注解依次展开介绍。
# 1、@SpringBootConfiguration

下面是@SpringBootConfiguration注解的定义。
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Configuration  
@Indexed  
public @interface SpringBootConfiguration {
	@AliasFor(annotation = Configuration.class)  
	boolean proxyBeanMethods() default true;
}
```

# 2、@EnableAutoConfiguration


## 相关注解--@AutoConfigurationPackage

下面是@AutoConfigurationPackage注解的定义。
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@Import(AutoConfigurationPackages.Registrar.class)  
public @interface AutoConfigurationPackage {
	String[] basePackages() default {};
	Class<?>[] basePackageClasses() default {};
}
```