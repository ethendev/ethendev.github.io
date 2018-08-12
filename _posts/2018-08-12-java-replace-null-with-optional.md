---
layout: post
title: 用Optional取代null
tags:  [Java,Optional]
categories: [Java]
keywords: Java,Optional
---


作为Java程序员我们经常会碰到 NullPointerException 异常。不管是初级，还是经验丰富的专家，NullPointerException 是一个很让人头疼的问题，一不小心程序就会出错。




1965年，Tony Hoare  用一个面向对象语言( ALGOL W )设计了第一个全面的引用类型系统。他的目的是确保所有引用的使用都是绝对安全的，编译器会自动进行检查。他加入了 null 引用，仅仅是因为实现起来非常容易。但是 null 引用导致了数不清的错误、漏洞和系统崩溃，可能在之后40年中造成了十亿美元的损失。所以他把它称作价值十亿美元的错误。近十年出现的大多数现代程序设计语言，包括Java，都采用了同样的设计方式，其原因是为了与更老的语言保持兼容，或者就像 Hoare 曾经陈述的那样，“仅仅是因为这样实现起来更加容易”。  


## null 引发的问题
这是一个拥有汽车及汽车保险的客户, Person、Car、Insurance的数据模型如下：
```
public class Person { 
    private Car car;
    public Car getCar() { return car; } 
}

public class Car { 
    private Insurance insurance; 
    public Insurance getInsurance() { return insurance; } 
}

public class Insurance { 
    private String name;
    public String getName() { return name; } 
}
```

获取汽车保险公司的名称：
```
public String getCarInsuranceName(Person person) { 
    return person.getCar().getInsurance().getName(); 
}
```

getCarInsuranceName 方法看起来相当正常，但是现实生活中很多人没有车。所以调用 getCar 方法的结果会怎样呢？在实践中，一种比较常见的做法是返回一个 null 引用，表示该值的缺失，即用户没有车。而接下来，对getInsurance的调用会返回null引用的insurance，这会导致运行时出现 一个 NullPointerException ，终止程序的运行。但这还不是全部。如果返回的 person 值为 null 会怎样？如果 getInsurance 的返回值也是null，结果又会怎样？ 

比较安全的做法是对每一个引用做 null 检查，如果引用链上的任何一个遍历的解变量值为null，它就返回一个值为 "Unknown" 的字符串。
```
public String getCarInsuranceName(Person person) {
    if (person != null) {
        Car car = person.getCar();
        if (car != null) {
            Insurance insurance = car.getInsurance();
            if (insurance != null) {
                return insurance.getName();
            }
        }
    }
    return "Unknown";
}
```
虽然这样做比较安全，但是嵌套了多层 if 块，可读性差。

## Optional
Java 8 中引入了一个新的类 java.util.Optional<T>。使用 Optional 类对上面的代码进行重构。

```
public class Person {
    private Optional<Car> car;
    public Optional<Car> getCar() { return car; } 
}

public class Car {
    private Optional<Insurance> insurance; 
    public Optional<Insurance> getInsurance() { return insurance; } 
}

public class Insurance {
    private String name;
    public String getName() { return name; } 
} 
```

使用Optional获取汽车保险公司的名称
```
public String getCarInsuranceName(Optional<Person> person) {
    return person.flatMap(Person::getCar)
            .flatMap(Car::getInsurance)
            .map(Insurance::getName)
            .orElse("Unknown");
}
```

可以看到，处理潜在可能缺失的值时，使用 Optional 具有明显的优势。不再需要使用那么多的条件分支，也不会增加代码的复杂性。

> 注意：Optional 类型的属性不能序列化。

Optional的设计初衷仅仅是要支持能返回Optional对象的语法。由于Optional类设计时就没特别考虑将其作为类的字段使用，所以它也并未实现 Serializable接口。由于这个原因，如果你的应用使用了某些要求序列化的库或者框架，在域模型中使用Optional，有可能引发应用程序故障。如果你一定要实现序列化的域模型，作为替代方案，可以提供一个能访问声明为 Optional、变量值可能缺失的方法:

```
public class Person { 
    private Car car; 
    public Optional<Car> getCarAsOptional() { 
        return Optional.ofNullable(car); 
    } 
} 
```

### Optional类的方法

<style>
table th:first-of-type {
    width: 15%;
}
</style>

|方法       |描述               |
|:-         |:-                    |
|empty      |返回一个空的 Optional 实例       |
|filter     |如果值存在并且满足提供的谓词，就返回包含该值的 Optional 对象；否则返回一个空的 Optional 对象|
|flatMap    |如果值存在，就对该值执行提供的 mapping 函数调用，返回一个 Optional 类型的值，否则就返回一个空的 Optional 对象 |
|get|如果该值存在，将该值用 Optional 封装返回，否则抛出一个 NoSuchElementException 异常|
|ifPresent  |如果值存在，就执行使用该值的方法调用，否则什么也不做|
|isPresent  |如果值存在就返回 true，否则返回 false|
|map        |如果值存在，就对该值执行提供的 mapping 函数调用|
|of         |将指定值用 Optional 封装之后返回，如果该值为 null，则抛出一个 NullPointerException 异常|
|ofNullable |将指定值用 Optional 封装之后返回，如果该值为 null，则返回一个空的 Optional 对象|
|orElse     |如果有值则将其返回，否则返回一个默认值| 
|orElseGet  |如果有值则将其返回，否则返回一个由指定的 Supplier 接口生成的值 |
|orElseThrow|如果有值则将其返回，否则抛出一个由指定的 Supplier 接口生成的异常 |


### 应用 Optional 的几种模式

#### 创建 Optional 对象
使用Optional之前，你首先需要学习的是如何创建Optional对象。完成这一任务有多种方法。 

声明一个空的 Optional
```
Optional<Car> optCar = Optional.empty(); 
```

依据一个非空值创建 Optional, 如果值为 null，这段代码会立即抛出一个NullPointerException
```
Optional<Car> optCar = Optional.of(car); 
```
  
创建一个允许null值的 Optional 对象
```
Optional<Car> optCar = Optional.ofNullable(car); 
```

#### 使用 map 从 Optional 对象中提取和转换值
调用方法前需要检查对象是否为null，为了支持这种模式，Optional提供了一个map方法。
```
Optional<Insurance> optInsurance = Optional.ofNullable(insurance); 
Optional<String> name = optInsurance.map(Insurance::getName); 
```

#### 两个 Optional 对象的组合
假设有这样一个方法，它接受一个Person和一个Car对象，并以此为条件对外部提供的服务进行查询，通过一些复杂的业务逻辑，试图找到满足该组合的最便宜的保险公司：
```
public Optional<Insurance> nullSafeFindCheapestInsurance(Optional<Person> person, Optional<Car> car) { 
    return person.flatMap(p -> car.map(c -> findCheapestInsurance(p, c))); 
} 
```

#### 使用 filter 剔除特定的值
检查保险公司的名称是否为 "AAA"，如果是的话输出 "ok"
```
Optional<Insurance> optInsurance = ...; 
optInsurance.filter(insurance -> "AAA".equals(insurance.getName())) 
            .ifPresent(x -> System.out.println("ok")); 
```
