---
layout: post
title: Java Long型数值序列化后在JS中丢失精度
tags:  [Java, 序列化, Long]
categories: [Java]
keywords: Java,序列化,Long
---


Java对象中的主键Id是long类型，API返回的数据里面Id数值是正确的，但是序列化后返回给浏览器的json串中，发现Id的值不对了。例如：Java中Id = 4616189619433466044，浏览器上变成了 4616189619433466000。




### 原因

原来是JavaScript的 Number  类型取值范围导致的问题。JavaScript中Number类型能安全表示数字的范围是 -(2^53 - 1) 到 (2^53 - 1)。

知道原因后解决这个问题的方法就很简单了，JS不用Number类型来保存long值，而是使用String类型。可以在JS中修改类型，也可以在后端代码中修改。此处给出使用jackson时的解决方案。

 
#### 方案一、全局配置
```
@Configuration
public class WebConfig extends WebMvcConfigurerAdapter {

    ......
    
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();

        ObjectMapper objectMapper = new ObjectMapper();

        /**
         * 将所有的long变成string
         */
        SimpleModule simpleModule = new SimpleModule();
        simpleModule.addSerializer(Long.class, ToStringSerializer.instance);
        simpleModule.addSerializer(Long.TYPE, ToStringSerializer.instance);
        objectMapper.registerModule(simpleModule);

        converter.setObjectMapper(objectMapper);
        converters.add(converter);
    }


}
```


#### 方案二、bean中配置
```
public class UserVo {

    @JsonSerialize(using = ToStringSerializer.class)
    private Long userId;

    ......
}
```