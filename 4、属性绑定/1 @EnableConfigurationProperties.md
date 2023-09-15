
首先看该注解的定义。
```java
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Import(EnableConfigurationPropertiesRegistrar.class)  
public @interface EnableConfigurationProperties{...}
```
其中需要注意的是，通过@Import注解导入了EnableConfigurationPropertiesRegistrar类。

