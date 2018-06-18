---
layout: post
title:  SpringBoot使用Gradle构建war包
categories: [SpringBoot]
tags:  [SpringBoot, Gradle, war]
keywords: SpringBoot,Gradle,war
---

* content
{:toc}


Spring Boot默认将应用打包成可执行的jar包。有时候需要打包成war包部署在tomcat等容器。下面简单介绍下打包的步骤。




### 一、修改gradle.build文件

1.1 添加如下配置

```
apply plugin: 'war'  
```

1.2 修改依赖，将tomcat的依赖范围修改为providedCompile

```
dependencies {  
    compile('org.springframework.boot:spring-boot-starter-web')  
    providedCompile("org.springframework.boot:spring-boot-starter-tomcat")  
    testCompile('org.springframework.boot:spring-boot-starter-test')  
}  
```

### 二、主类继承SpringBootServletInitializer，重写configure方法

```
@SpringBootApplication  
public class GradletestApplication extends SpringBootServletInitializer {  
  
    public static void main(String[] args) {  
        SpringApplication.run(GradletestApplication.class, args);  
    }  
  
    @Override  
    protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {  
        return builder.sources(GradletestApplication.class);  
    }  
}  
```

### 三、构建
上述修改完后，执行如下命令，就可以在build目录下生成war包了。


```
gradle build  
```