
下面是通过maven构建springboot应用时的pom文件。
```xml
<?xml version="1.0" encoding="UTF-8"?> 
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"   xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">   
	<modelVersion>4.0.0</modelVersion>   
	<groupId>com.example</groupId>   
	<artifactId>myproject</artifactId>   
	<version>0.0.1-SNAPSHOT</version>   
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.0.6</version>
	</parent>
	<!-- Additional lines to be added here... -->
</project>
```

spring-boot-starter-parent 是一个特殊的starter，其父pom spring-boot-dependencies 提供了<properties/>和<dependencyManagement/>，即提供了依赖管理和版本控制。

因为maven默认编译来自src/main/java目录下的源文件，因此需要构建此目录结构。

创建下面的类。
```java
@SpringBootApplication  
public class SpringbootLearningApplication {  
  
    public static void main(String[] args) {  
        SpringApplication.run(SpringbootLearningApplication.class, args);  
    }  
  
}
```