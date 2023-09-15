
ImportCandidates对外只提供一个public static方法，从classpath下的META-INF/spring/%s.imports中获取资源URLs

# load()
```java
public static ImportCandidates load(Class<?> annotation, ClassLoader classLoader) {  
    ClassLoader classLoaderToUse = decideClassloader(classLoader);  
    // LOCATION = "META-INF/spring/%s.imports"
    String location = String.format(LOCATION, annotation.getName()); 
    // return classLoader.getResources(location);
    Enumeration<URL> urls = findUrlsInClasspath(classLoaderToUse, location);  
    List<String> autoConfigurations = new ArrayList<>();  
    while (urls.hasMoreElements()) {  
		// 正常情况只有一个元素
        URL url = urls.nextElement();  
        autoConfigurations.addAll(readAutoConfigurations(url));  
    }  
    return new ImportCandidates(autoConfigurations);  
}
```
## 涉及的方法--decideClassloader()
```java
private static ClassLoader decideClassloader(ClassLoader classLoader) {  
    if (classLoader == null) {  
        return ImportCandidates.class.getClassLoader();  
    }   
    return classLoader;  
} 
```
## findUrlsInClasspath()
```java
private static Enumeration<URL> findUrlsInClasspath(ClassLoader classLoader, String location) {  
    try {  
        return classLoader.getResources(location);  
    }  
    catch (IOException ex) {...}  
}
```
## readAutoConfigurations()
```java
// jar:file:/D:/Software/apache-maven-3.8.1/repository/org/springframework/boot/spring-boot-autoconfigure/2.7.2/spring-boot-autoconfigure-2.7.2.jar!/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
private static List<String> readAutoConfigurations(URL url) {  
    try (BufferedReader reader = new BufferedReader(  
	        new InputStreamReader(new UrlResource(url).getInputStream(), StandardCharsets.UTF_8))) {  
        List<String> autoConfigurations = new ArrayList<>();  
        String line;  
        while ((line = reader.readLine()) != null) {  
            line = stripComment(line);  
            line = line.trim();  
            if (line.isEmpty()) {  
                continue;  
            }  
            autoConfigurations.add(line);  
        }  
        return autoConfigurations;  
    }  
    catch (IOException ex) {...}  
}
```
## stripComment()
```java
private static String stripComment(String line) {  
	// COMMENT_START = "#"
    int commentStart = line.indexOf(COMMENT_START);  
    if (commentStart == -1) {  
        return line;  
    }  
    return line.substring(0, commentStart);  
}
```