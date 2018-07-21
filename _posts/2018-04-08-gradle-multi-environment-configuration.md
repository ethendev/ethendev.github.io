---
layout: post
title: Gradle 多环境配置教程
tags:  [Gradle]
categories: [Gradle]
keywords: Gradle,多环境
---

通常我们的项目会存在多个环境，比如，开发环境，测试环境，生产环境等。不同的环境配置不同，发布的时候需要根据发目标环境来选择打包对应的配置文件。使用 Gradle 作为构建工具，可以很方便的实现。




### 目录结构

首先 新建一个 SpringBoot 应用，然后根据环境创建对应的配置文件，目录结构如下：
resources：
-- application.properties
resources-dev
-- application.properties
resources-prod
-- application.properties
resources-test
-- application.properties


### 配置
首先在每一个 application.properties 配置文中添加 phase 属性，值为对应的环境名称。比如，dev 环境的配置如下：
```
phase=dev
```


build.gradle中添加下面的配置，设置对应环境变量

```
// set environment
if (!project.hasProperty('profile') || !profile) {
	ext.profile = 'dev'
}

// set Configuration file path
sourceSets {
	main {
		java {
			srcDir "src/main/java"
		}
		resources {
			srcDirs = ["src/main/resources", "src/main/resources-${profile}", "src/main/java"]
		}
	}
}
```

编写一个测试类进行测试
```
@RestController
public class Test {
 
    @Autowired
    private Environment env;
 
    @RequestMapping("/testEnv")
    String testEnv()  {
        String name = env.getProperty("phase");
        return name;
    }

}
```

将对应的环境名称赋值给 -Pprofile ，比如， dev 环境 -Pprofile=dev。运行打包命令
```
gradle -Pprofile=dev build
```

访问 http://127.0.0.1:8080/testEnv ，
开发环境下输出dev，测试环境下输出test，多环境配置这样就完成了。

