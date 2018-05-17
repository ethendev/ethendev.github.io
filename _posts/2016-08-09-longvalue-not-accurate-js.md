---
layout: post
title: Java序列化时Long型数值在JS中不准确
tags:  [Java,序列化,Long]
categories: [Java]
author: Ethen
excerpt: "DB中我用的ID是long类型的，Java代码中接口返回的数据，ID数值是对的，但当通过浏览器上返回的json串中，发现数值不对了"
---

## 现象
DB中我用的ID是long类型的，Java代码中接口返回的数据，ID数值是对的，但当通过浏览器上返回的json串中，发现数值不对了。例如：DB中ID= 4616189619433466044，浏览器上变成了 4616189619433466000。

## 原因

原来是JavaScript数值类型取值范围导致的问题，JavaScript中Number类型的能安全的表示数字的范围在 -(2^53 - 1) 到 2^53 - 1 之间之间，包含-(2^53 - 1) 和 2^53 - 1。安全的意思是指能够准确地表示整数和正确地比较整数。
 
解决这个问题的方法很简单，就是JS不用number来保存long值，而是使用string。可以在js中修改,也可以在服务端序列化的时候修改。此处给出sprign mvc 使用 jackson时的解决方案。
 
```
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
  MappingJackson2HttpMessageConverter jackson2HttpMessageConverter = new MappingJackson2HttpMessageConverter();
 
  ObjectMapper objectMapper = new ObjectMapper();
 
  /**
  * 序列换成json时,将所有的long变成string
  * 因为js中得数字类型不能包含所有的java long值
  */
  
  SimpleModule simpleModule = new SimpleModule();
  simpleModule.addSerializer(Long.class, ToStringSerializer.instance);
  simpleModule.addSerializer(Long.TYPE, ToStringSerializer.instance);
  objectMapper.registerModule(simpleModule);
 
  jackson2HttpMessageConverter.setObjectMapper(objectMapper);
  converters.add(jackson2HttpMessageConverter);
}
```
