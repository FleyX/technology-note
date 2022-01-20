---
id: "20211009"
date: "2021-10-09 15:42:00"
title: "SpringBoot参数校验看这篇就够了"
tags: ["java", "SpringBoot","exception",“validation”]
categories:
  - "java"
  - "spring boot学习"
---


**本文所用到的全部代码见文末**

## 前言

任何一个项目都需要对接口做参数校验，最简单粗暴的校验方式就是在代码中硬编码来一个个检查参数，这种方式显然是很不优雅的，Spring 已经为我们设计了一套比较优雅的校验方式，本篇文章将进行详细说明。

## 两个注解`@Valid`和`@Validated`

标题中的两个注解，想必是大家经常看到的，但是大部分人应该都不清楚这两个注解有什么区别（比如我），本节将解开你的疑惑。

首先来看看下这两个注解的来源：

`@Valid`:`javax.validation.Valid`,来自于 javax,属于标准 JSR-380 规范

`@Validated`：`org.springframework.validation.annotation.Validated`,来自于 spring validation,属于 Spring 的 JSR-380 规范，是标准 JSR-380 的一个变种,有一些增强功能。

<!-- more -->

由此可以看出这两个注解大部分功能相似，但是存在一些区别。

**网上大部分文章说的是 JSR-303,JSR-303 是很老的标准了（Bean Validation 1.0），JSR-380 是目前最新的标准（Bean Validation 2.0）,具体可在[此网站](https://beanvalidation.org/)中查看相关文档**

### 作用域区别

这两个注解的作用域定义如下：

```java
//@Valid
@Target({ METHOD, FIELD, CONSTRUCTOR, PARAMETER, TYPE_USE })
//@Validated
@Target({TYPE, METHOD, PARAMETER})
```

`@Valid`可作用于方法、**成员属性**、构造函数、方法参数、使用类型的任意语句中（这个是 Java8 中新增的）

`@Validated`可作用于类型、方法、方法参数上

可以看到只有`@Valid`能够作用在成员属性上,也就是只能通过它能够实现嵌套验证（什么是嵌套验证见后文）,单单`@Validated`不能实现。

### 功能区别

对比两个注解的定义可以发现，@Validated 多了一个参数`Class<?>[] value() default {}`,可以传入 Class 参数，此参数用于分组校验逻辑的实现。`@Valid`目前为止还不支持分组。

## 校验内容

**本节只说明如何进行校验，校验异常全局处理见下一节**

当我们引入校验依赖后，可以发现有很多的校验注解可以使用，比如@NotNull,@Range,@Email 等等,我们仔细查看这些注解的来源，可以发现部分注解是来自于`javax.validation`,部分注解是来自于`org.hibernate.validator`,为啥呢

javax 是 JSR-380 的标准实现，hibernate 是这个规范的参考实现并扩充了一些功能。如果标准实现将 hibernate 中扩充的某些校验加入到标准后，hibernate 就会将自身的校验实现标记为过期（比如 6.2 版本中的@Email 注解）

具体有哪些校验注解可以[看看这篇文章](https://juejin.cn/post/6844903976270299149)

## 使用

### 依赖引入

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
    <version>2.5.5</version>
</dependency>
```

引入这个依赖会一起引入`javax.validation`和`org.hibernate.validator`

每个校验注解中都有一个 message 参数，此参数可传入国际化的 key,或者自定义的字符串，用于自定义错误提示。

### url 参数校验

首先需要在类上加一个`@Validated`注解,否则无法对 url 参数进行校验

```java
@RestController
@Validated
public class TestController {
}
```

然后在对具体的方法参数进行校验，比如：

```java
/**
 * url参数校验
 */
@GetMapping("/test1")
public Result test1(@NotBlank @RequestParam String param1, @Range(max = 100, min = 10) @RequestParam int param2) {
    return null;
}
```

### post 表单校验

首先定义入参类 TestBody.java

```java
@Data
public class TestBody {
    @NotBlank
    @Email(message = "请输入一个邮箱")
    private String param1;
    @NotNull
    private String param2;
}
```

然后在入口方法中对要校验的参数加上@Valid 或者@Validated

```java
@PostMapping("/test2")
public Result test2(@Validated @RequestBody TestBody body) {
    return null;
}
```

### 嵌套校验

何谓嵌套？就是一个对象中包含另外的对象。默认情况下是不会对潜逃对象中的属性进行校验的。比如新建一个 TestBody2 类，其中包含 TestBody

```java
@Data
public class TestBody2 {
    @NotBlank
    @Email(message = "请输入一个邮箱")
    private String param1;
    @NotNull
    private String param2;
    @NotNull
    private TestBody testBody;
}
```

上述情况在校验时只会校验 testBody 是否为 null,并不会校验其中的属性。必须要用@Valid（@Validated 不能修饰属性，所以只能用@valid）来注解 testBody 属性，才会对其属性进行校验。

入口方法如下：

```java
/**
 * 嵌套校验
 */
@PostMapping("/test3")
public Result test3(@Valid @RequestBody TestBody2 testBody2) {
    throw new RuntimeException("我是test2");
}
```

### 分组校验

在实际的业务需求中，可能一个入参类，会被多个接口使用，每个接口有不同的校验逻辑，这是就需要对校验逻辑进行分组，以在不同的接口中调用不同的校验组和。例子如下：

建立 TestBody3

```java
@Data
public class TestBody3 {
    @NotBlank(groups = {Insert.class})
    private String param1;
    @NotBlank(groups = {Update.class})
    private String param2;
}
```

通过**groups**属性来说明本注解是在哪些情况下生效，如 param1 只在 Insert 模式下才会进行校验，param2 只在 Update 模式下生效，默认在入口函数中定义，如下：

```java
/**
 * 分组校验1
 */
@PostMapping("/test4")
public Result test4(@Validated({Update.class}) @RequestBody TestBody3 testBody) {
    return null;
}

/**
 * 分组校验2
 */
@PostMapping("/test5")
public Result test5(@Validated({Insert.class}) @RequestBody TestBody3 testBody) {
    return null;
}
```

**注意分组校验必须使用`@validated`注解**

### 集合类校验

集合类校验分两种

一种是对象内集合的校验，直接使用@valid 注解该属性即可，例如：

```java
@Data
public class TestBody5 {
    @Valid
    private List<TestBody4> list;
}
```

使用示例见：`com.fanxb.exceptiontest.controller.TestController#test7`

另外一直直接接收集合入参，如：

```java
/**
     * 集合校验2（对象集合）
     */
    @PostMapping("/test8")
    public Result test8(@RequestBody List<@Valid TestBody4> list) {
        return null;
    }
```

这种需要使用`@Valid`来注解泛型类,使用示例见`com.fanxb.exceptiontest.controller.TestController#test8`

### 自定义校验

最后，校验依赖提供的校验逻辑是有限的，有时我们需要进行一些特殊的校验，这就需要实现自定义校验注解，实现流程如下：

首先定义注解

```java
@Target({ElementType.FIELD}) //作用于字段
@Retention(RetentionPolicy.RUNTIME)//生命周期
@Constraint(validatedBy = CustomCheckValidator.class)//校验逻辑实现类
public @interface CustomCheck {

    String message() default "自定义校验默认错误提示";

    /**
     * 自定义参数，可传递到校验实现类CustomCheckValidator中
     */
    String param1() default "";

    Class<?>[] groups() default {}; //用于分组校验

    Class<? extends Payload>[] payload() default {};

}
```

然后编写校验逻辑类

```java
public class CustomCheckValidator implements ConstraintValidator<CustomCheck, String> {
    private String param1;

    @Override
    public void initialize(CustomCheck constraintAnnotation) {
       //在此获取校验参数
       param1 = constraintAnnotation.param1();
        ConstraintValidator.super.initialize(constraintAnnotation);
    }

    @Override
    public boolean isValid(String s, ConstraintValidatorContext constraintValidatorContext) {
        return param1 != null && param1.equals(s);
    }
}
```

最后即可向正常校验注解一样使用。详情见代码`TestController#test6`

## 校验异常处理

### `BindingResult`处理

前一节讲了如何进行参数校验,那么参数校验的结果怎么处理呢？一种方式是将校验结果放在`BindingResult`中，如下：

```java
/**
     * 自定义校验
     */
    @PostMapping("/test6")
    public Result test6(@Validated @RequestBody TestBody4 testBody,BindingResult result) {
        if(result.hasErrors()){
            log.info("asdf");
        }
        return null;
    }
```
显而易见这种处理方式很繁琐，需要对每个接口都做一样的处理，不推荐使用。

**推荐使用全局异常处理来对校验结果进行统一处理**

### 全局异常处理

[上一篇:blog.fleyx.com/blog/detail/20210927/](https://blog.fleyx.com/blog/detail/20210927/)中详细说明了springboot的全局异常处理,不了解的可以去看看。

在全局异常处理类`Exceptionhandle`中能对全部的异常进行捕获处理，那么我们只需要找到参数校验抛出的异常，然后针对这些异常进行处理就可以了。代码如下：

```java
@RestControllerAdvice
@Slf4j
public class ExceptionHandle {
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        BaseException be;
        if (e instanceof BaseException) {
            be = (BaseException) e;
            //手动抛出的异常，仅记录message
            log.info("baseException:{}", be.getMessage());
            if (be instanceof CustomBusinessException) {
                //可在这里进行一些针对特定异常的处理逻辑
                log.info("customBusinessException:{}", be.getMessage());
            }
        } else if (e instanceof ConstraintViolationException) {
            //url参数、数组类参数校验报错类
            ConstraintViolationException ce = (ConstraintViolationException) e;
            //针对参数校验异常，建立了一个异常类
            be = new CustomValidException(ce.getMessage());
        } else if (e instanceof MethodArgumentNotValidException) {
            //json对象类参数校验报错类
            MethodArgumentNotValidException ce = (MethodArgumentNotValidException) e;
            be = new CustomValidException(Objects.requireNonNull(ce.getFieldError()).getDefaultMessage());
        } else {
            //其它异常，非自动抛出的,无需给前端返回具体错误内容（用户不需要看见空指针之类的异常信息）
            log.error("other exception:{}", e.getMessage(), e);
            be = new BaseException("系统异常，请稍后重试", e);
        }
        return new Result(be.getCode(), be.getMessage(), null);
    }
}
```
核心是对`ConstraintViolationException`和`MethodArgumentNotValidException`两种异常的处理。

## 结尾

本篇收集了大量的材料，对参数校验的相关内容覆盖应该比较全面了，码字不已，望点赞收藏。

**本篇用到的全部代码见：**[github](https://github.com/FleyX/demo-project/tree/master/spring-boot/paramsCheck)

