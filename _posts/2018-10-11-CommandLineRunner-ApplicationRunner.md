---
layout: post
title: CommandLineRunner 和 ApplicationRunner
tags:  [Java]
categories: [Java]
keywords: CommandLineRunner,ApplicationRunner
---


如果想在 SpringApplication 启动后需要运行某些特定代码，可以通过实现 ApplicationRunner 或 CommandLineRunner 接口。 这两个接口以的工作方式相同，并提供单个 run 方法，该方法在 SpringApplication.run（...）方法结束之前被调用。




### CommandLineRunner

CommandLineRunner 接口提供了访问应用程序参数的方式，即通过字符串数组访问。

接口源码
```
package org.springframework.boot;

@FunctionalInterface
public interface CommandLineRunner {
    void run(String... args) throws Exception;
}
```

例子： 程序启动后输出 CommandLineRunner 
```
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Component;

@Component
public class CommandLineRunnerTest implements CommandLineRunner {

    public void run(String... args) throws Exception {
        System.out.println("CommandLineRunner test");
    }

}
```

输出结果：
```
...
Tomcat started on port(s): 8080 (http) with context path '/lip' 
Started FrontierApplication in 8.536 seconds (JVM running for 9.711) 
ApplicationRunner test
...
```

### ApplicationRunner

ApplicationRunner 通过 ApplicationArguments 接口访问程序参数。


接口源码
```
package org.springframework.boot;

@FunctionalInterface
public interface ApplicationRunner {
    void run(ApplicationArguments args) throws Exception;
}
```

例子： 程序启动后输出 ApplicationRunner 
```
import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.stereotype.Component;

@Component
public class ApplicationRunnerTest implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("ApplicationRunner test");
    }

}
```

### 指定运行顺序

如果程序中定义了必须以特定顺序调用的多个 CommandLineRunner 或 ApplicationRunner bean，则还可以实现 org.springframework.core.Ordered 接口或使用 org.springframework.core.annotation.Order 注解。


```
@Order(1)
@Component
public class ApplicationRunnerTestA implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("ApplicationRunner 1");
    }

}
```

```
@Order(2)
@Component
public class ApplicationRunnerTestB implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) throws Exception {
        System.out.println("ApplicationRunner 2");
    }

}
```

输出结果：
```
ApplicationRunner 1
ApplicationRunner 2
```