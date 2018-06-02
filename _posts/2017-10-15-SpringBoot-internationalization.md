---
layout: post
title: SpringBoot国际化教程
tags:  [SpringBoot, 国际化]
categories: [SpringBoot]
keywords: SpringBoot,国际化
---

* content
{:toc}

在这个教程中，我们将看看如何将国际化添加到Spring Boot应用程序。




### LocaleResolver
为了让我们的应用程序能够确定当前正在使用的语言环境，我们需要在自己的WebMvcConfig类中添加一个LocaleResolver bean：
```java
@Bean
public LocaleResolver localeResolver() {
    CookieLocaleResolver cookieLocaleResolver = new CookieLocaleResolver();
    cookieLocaleResolver.setDefaultLocale(Locale.US);
    return cookieLocaleResolver;
}
```

LocaleResolver接口能根据会话，Cookie，Accept-Language头或固定值确定当前语言环境。
在我们的例子中，我们使用了基于Cookie的解析器CookieLocaleResolver并设置了一个值为US的默认语言环境。

### LocaleChangeInterceptor
接下来，我们需要添加一个拦截器bean，该bean将根据Cookie中的lang的值切换到新的区域设置：
```java
@Bean
public LocaleChangeInterceptor localeChangeInterceptor() {
    LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
    lci.setParamName("lang");
    return lci;
}
```

这个bean需要被添加到应用程序的拦截器注册表中。
```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    registry.addInterceptor(localeChangeInterceptor());
}
```

### 定义MessageSources
默认情况下，Spring Boot应用程序将在src/main/resources文件夹中查找包含国际化键和值的消息文件。
缺省语言环境的文件名称为messages.properties，并且每个语言环境的文件将被命名为messages_XX.properties，其中XX是语言环境代码。
要进行本地化的值的键必须在每个文件中都是相同的，其值应与其对应的语言相适应。

如果某个键在某个请求的语言环境中不存在，则应用程序将回退到默认语言环境值。
我们定义一个名为messages.properties的英文语言默认消息文件：
```
greeting=Hello! Welcome to our website!
```
接下来，我们使用相同的键为法语创建一个名为messages_zh-CN.properties的文件：
```
greeting=你好，欢迎访问我们的网站！
```

### ReloadableResourceBundleMessageSource
使用MessageSource来管理国际资源文件
```java
@Bean
public ReloadableResourceBundleMessageSource messageSource() {
    ReloadableResourceBundleMessageSource messageSource = newReloadableResourceBundleMessageSource();
    messageSource.setBasename("classpath:messages");
    messageSource.setDefaultEncoding("UTF-8");
    return messageSource;
}
```

### 测试
写一个controller和一个简单的HTML页面，来测试结果：
```Java
@Controller
public class TestController {

    @Resource(name = "messageSource")
    MessageSource messageSource;
 
    @GetMapping("/test")
    public String test() {
        String msg = messageSource.getMessage("greeting", null,LocaleContextHolder.getLocale());
        model.addAttribute("msg", msg);
        return "index";
    }
}
```

使用thymeleaf写一个简单网页
```html
<h1 th:text="#{msg}"></h1>
```

将Cookie中lang设置为zh-CN，访问 http://localhost:8080/test ，页面上将显示

你好，欢迎访问我们的网站！