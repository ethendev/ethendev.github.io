---
layout: post
title: SpringBoot之使用Hibernate Validator验证参数
tags:  [SpringBoot,Hibernate Validator,参数验证]
categories: [SpringBoot]
keywords: SpringBoot,Hibernate Validator,参数验证
---

开发 WEB 应用时参数校验必不可少。前端通过 js 校验参数合法性，后端也需要对参数进行校验。常见的做法是在 Controller 或者 Service 中通过 if 或者 assert 判断参数是否合法。这样的方式虽然简单，但是代码冗余、耦合度高。其实可以通过 Hibernate Validator 优雅的进行参数校验。




[Hibernate Validator](http://hibernate.org/validator/) 是 Bean Validation 的参考实现 . Hibernate Validator 提供了 [JSR 303](http://jcp.org/en/jsr/detail?id=303) 规范中所有内置 constraint 的实现，除此之外还有一些附加的 constraint。


## 添加依赖
使用 SPringBoot 开发程序时，用户不需要添加额外的依赖来使用 Hibernate Validator ，因为 spring-boot-starter-web 会自动引入它。

如果没用 SpringBoot ，则需要添加依赖
```
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.8.Final</version>
</dependency>
```


## 内置注解
### Bean 校验注解
Hibernate Validator 包含一组基本的常用约束， 其中最重要的是 Bean Validation 规范定义的约束，所有这些约束都适用于字段/属性级别，Bean验证规范中没有定义类级别约束。

<style>
table th:first-of-type {
    width: 30%;
}
</style>

|限制         |说明   |
|-            |-      |
|@AssertFalse |限制必须为false|
|@AssertTrue  |限制必须为true|
|@DecimalMax  |限制必须为一个不大于指定值的数字|
|@DecimalMin  |限制必须为一个不小于指定值的数字|
|@Digits      |限制必须为一个小数，且整数部分的位数不能超过integer，小数部分的位数不能超过fraction
|@Null        |限制只能为null|
|@NotNull     |限制必须不为null|
|@Email       |验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式|
|@Future      |限制必须是一个将来的日期|
|@Max         |限制必须为一个不大于指定值的数字|
|@Min         |限制必须为一个不小于指定值的数字|
|@NotBlank    |验证注解的元素值不为null且去除空格后长度不为0，@NotBlank只用于字符串|
|@NotEmpty    |验证注解的元素值不为null且不为空，支持字符串、集合、Map和数组类型|
|@Past        |限制必须是一个过去的日期|
|@Pattern     |限制必须符合指定的正则表达式|
|@Range	      |限制必须在合适的范围内|
|@Size        |限制字符长度必须在min到max之间|

### 附加注解 
Hibernate Validator 还提供了几个有用的自定义约束，如下所示。除了一个例外，这些约束也适用于字段/属性级别，只有@ScriptAssert类级别约束。

|限制         |说明   |
|-            |-      |
|@CreditCardNumber|检查带注释的字符序列是否通过了Luhn校验和测试|
|@Currency    |检查带注释的货币单位javax.money.MonetaryAmount是否为指定货币单位的一部分|
|@EAN |检查带注释的字符序列是否为有效的EAN条形码。type确定条形码的类型。默认值为EAN-13|
|@ISBN        |检查带注释的字符序列是否为有效的ISBN。type确定ISBN的类型。默认值为ISBN-13|
|@Length      |验证该注释字符序列是间min和max包含|
|@Range       |检查带注释的值是否在（包括）指定的最小值和最大值之间|
|@Length      |限制必须为true|
|@Range       |限制必须为一个不大于指定值的数字|
|@SafeHtml    |检查带注释的值是否包含潜在的恶意片段|
|@UniqueElements|检查带注释的集合是否仅包含唯一元素|
|@URL         |根据RFC2396检查带注释的字符序列是否为有效URL |

## 示例
### 请求参数校验
下面来看一个例子，使用 Validator 来验证 bean 参数。

校验 name 属性不能为空且长度不能超过 20 ，description 属性不能为空且长度不能超过 200， 若没有通过校验则使用自定义的 message 返回报错信息。 
```
public class User {
    
    @Length(max = 20, message = "The length of name can not exceed 20")
    @NotEmpty(message = "name can not be null")
    private String name;
    
    @Length(max = 200, message = "The length of description can not exceed 200")
    @NotEmpty(message = "description can not be null")
    private String description;

    // get set 省略
}
```

Controller 方法中在需要校验的 bean 前面添加 @Validated 注解，如果参数不满足条件，将抛出异常。
```
@RestController 方法中在需要校验的 bean 前面添加 
public class TestController {

    @PostMapping("/addUser")
    public void addUser(@Validated @RequestBody User user) throws Exception {
        System.out.println(user);
    }
}
```

### 单个方法参数校验

Controller 类上添加 @Validated 注解， 方法上在需要校验的参数前添加对应的注解， 比如使用 @URL 校验 url 字符串是否合法，代码如下：

```
@RestController
@Validated
public class TestController {

    @GetMapping(value = "/test")
    public void test(@URL @RequestParam String url) throws Exception {
        System.out.println(url);
    }
}
```

浏览器上访问 http://127.0.0.1:8080/test?url=1234 ，报错信息如下：
```
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Mon May 16 20:17:16 CST 2018
There was an unexpected error (type=Internal Server Error, status=500).
test.url: must be a valid URL
```

### 方法返回值校验
还可以用来校验方法的返回值
```
@RestController
@Validated
public class TestController {

    @GetMapping(value = "/testReturn")
    public @NotEmpty String testReturn(String param) {
        return param;
    }
}
```

### 分组校验
有时候对同一个对象，不同的场景下有不同的验证要求。比如，添加新用户时不需要验证 userId ，但是更新时需要验证。可以使用分组校验来实现这个需求。

首先创建分组
```
public interface Add {
}
```

```
public interface Update {
}
```

```
public class UserVo {
    // 只在更新时验证
    @NotNull(groups = {Update.class}, message = "userId can notbe null")
    private Long userId;

    @NotBlank
    @Length(min = 4, max = 20, message = "Length must be between 4 and 20")
    private String nickName;

    // 省略 set get
}
```

@Validated 注解上需要指定组，可以从 BindingResult 对象中获取校验失败的详细信息
```
public class TestController {
    @RequestMapping(value = "/add")
    public void add(@Validated UserVo user, BindingResult result) {
        System.out.println(user);
        if(result.hasErrors()){
            List<ObjectError> allErrors = result.getAllErrors();
            for (ObjectError error : allErrors) {
                System.out.println(error.getDefaultMessage());
            }
        }
    }

    @RequestMapping(value = "/update")
    public void update(@Validated({Update.class}) UserVo user, BindingResult result) {
        System.out.println(user);
        if(result.hasErrors()){
            List<ObjectError> allErrors = result.getAllErrors();
            for (ObjectError error : allErrors) {
                System.out.println(error.getDefaultMessage());
            }
        }
    }
}
```

## 自定义校验
虽然 Hibernate Validator 已经提供了很多常用的校验注解，但对于一些复杂的参数的校验，原有的校验不满足要求的情况下，用户可以实现自己的校验注解。

所有的验证都需要实现 ConstraintValidator 接口，它包含一个初始化事件方法，和一个判断是否合法的方法。
```
public interface ConstraintValidator<A extends Annotation, T> {
    default void initialize(A constraintAnnotation) {
    }

    boolean isValid(T var1, ConstraintValidatorContext var2);
}
```

下面的例子实现一个自定义注解，要求参数不能与设置的值相等

注解定义如下：
```
@Documented
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = NotEqualValidator.class)
public @interface NotEqual {
    String message() default "不能与给定的值相等";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    String value();
}
```

注解实现类如下：
```
public class NotEqualValidator implements ConstraintValidator<NotEqual, String> {

    private String value;

    @Override
    public void initialize(NotEqual constraintAnnotation) {
        this.value = constraintAnnotation.value();
    }

    @Override
    public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        return !s.equals(value);
    }
}
```

Controller 代码：
```
@RestController
@Validated
public class TestController {

    @GetMapping(value = "/test")
    public void test(@NotEqual("1111") @RequestParam String url) throws Exception {
        System.out.println(url);
    }
}
```

浏览器上访问 http://127.0.0.1:8080/test?url=1111 ，结果如下：
```
Whitelabel Error Page
This application has no explicit mapping for /error, so you are seeing this as a fallback.

Mon May 16 20:54:22 CST 2018
There was an unexpected error (type=Internal Server Error, status=500).
test.url: 不能与给定的值相等
```


## 自定义异常处理
当参数校验没有通过时，Hibernate Validator 会抛出异常。但是 SpringBoot 默认的异常返回结果可能并不是用户所希望的，就像上面的例子一样。我只想获取报错信息，不希望返回异常的页面。这个时候我们可以通过 @ControllerAdvice 来自定义异常处理，返回自己希望的结果。

下面是一个自定义异常的例子。

返回结果封装类：
```
public class ServiceResult {

    private Integer code;
    private String message;
    private Object result;

    public ServiceResult () {
    }

    public ServiceResult(Integer code, String message) {
        this.code = code;
        this.message = message;
    }
    
    // get set 省略
}
```

全局异常处理类，主要用来处理参数校验没有通过时的异常信息
```
@ControllerAdvice
@RestController
public class GlobalExceptionHandler {

    // 方法参数校验异常
    @ExceptionHandler(value = ConstraintViolationException.class)
    public ServiceResult constraintViolationException(Exception e) {
        if (e.getMessage() != null) {
            int index = e.getMessage().indexOf(":");
            return new ServiceResult(HttpStatus.BAD_REQUEST.value(), index != -1 ?
                    e.getMessage().substring(index + 1).trim() : e.getMessage());
        }
        return new ServiceResult(HttpStatus.BAD_REQUEST.value(), null);
    }


    // Bean 校验异常
    @ExceptionHandler(value = MethodArgumentNotValidException.class)
    public ServiceResult notValidExceptionHandler(MethodArgumentNotValidException e) throws Exception {
        ServiceResult ret = new ServiceResult();
        ret.setCode(HttpStatus.BAD_REQUEST.value());
        if (e.getBindingResult() != null && !CollectionUtils.isEmpty(e.getBindingResult().getAllErrors())) {
            ret.setMessage(e.getBindingResult().getAllErrors().get(0).getDefaultMessage());
        } else {
            ret.setMessage(e.getMessage());
        }
        return ret;
    }

}
```

自定义返回结果：
```
{
  "code": 400,
  "message": "不能与给定的值相等",
  "result": null
}
```
