---
id: "20210927"
date: "2021-09-27 15:42:00"
title: "SpringBoot异常处理看这篇就够了"
tags: ["java", "SpringBoot","exception"]
categories:
  - "java"
  - "spring boot学习"
---

**本文用到全部源码，见文末**

## 为什么要有全局异常处理

一个软件系统总是会遇到各种异常,包括人为抛出的业务异常，以及bug导致的异常。那怎么对这些异常进行处理呢？

最简单的做法是在`controller`层用`try/catch`捕获这些异常，然后进行对应的处理.显而易见这种处理方法存在很大的问题，主要是：

1. **代码冗余**：`controller`中会存在大量的`try/catch`代码，这些代码可能内容基本都是一样的，属于垃圾代码

2. **不便于修改**：假设需要对某种错误类型进行特殊处理，那么需要修改所有的接口，很麻烦,也容易出错

## Spring统一异常处理

那么有没有一种方法能够统一处理呢？当然是有的，spring中的AOP就是用来做这样的统一处理的。当然不需要我们来实现这个AOP逻辑。spring已经帮我们实现了。通过`@ControllerAdvice`（从英文名称就能看出来这个注解用于处理controller的各种事件通知）注解，可以配合`@ExceptionHandler`、`@InitBinder`、`@ModelAttribute`等注解配套使用.既然是异常处理，那么我们需要用到的就是`@ExceptionHandler`.

<!-- more -->

最简单的用法如下：

建立一个ExceptionHandle类即可（**`@RestControllerAdvice`就是`@ControllerAdvice`和`@ResponseBody`的组合注解**）：
```java
/**
 * @author fanxb
 * @date 2021-09-24-下午4:37
 */
@RestControllerAdvice
@Slf4j
public class ExceptionHandle {
    @ExceptionHandler(Exception.class)
    public Result handleException(Exception e) {
        log.error("捕获到错误：{}", e.getMessage(), e);
        return new Result(0, e.getMessage(), null);
    }
}
```

建立上述类后,**controller**层抛出的异常全部会进入到`handleException`方法中进行统一处理。

**注意，只有controller抛出的异常会到这里来，过滤器、拦截器的异常是不行的，因为这里的AOP切点是controller**

## 如何优雅的进行错误处理

上一节说明了如何对异常进行捕获，那么在真实的项目中是如何进行处理的呢？这里介绍一种比较优雅的处理方式。

### 定义统一返回类

统一接口返回的数据格式，便于前后端交互，同时也方便进行一些统一的操作。

```java
/**
 * 下面三个注解是lombok的减负注解，减少一些结构性的编码 
 */
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Result implements Serializable {
    private static final long serialVersionUID = 451834802206432869L;
    /**
     * 1：成功,其他失败
     */
    private int code;
    /**
     * message提示内容，当接口异常时提示异常信息
     */
    private String message;
    /**
     * 携带真正需要返回的数据
     */
    private Object data;

    public static Result success(Object obj) {
        return new Result(1, null, obj);
    }
}
```

### 定义异常类

首先定义一个基础异常类，继承`RuntimeException`

```java
@Getter
public class BaseException extends RuntimeException {
    private static final long serialVersionUID = -3149747381632839680L;
    /**
     * 基本错误code为0
     */
    private static final int BASE_CODE = 0;
    private final int code;
    private final String message;

    public BaseException() {
        this(null, null, null);
    }

    public BaseException(String message) {
        this(message, null, null);
    }

    public BaseException(Exception e) {
        this(null, null, e);
    }

    public BaseException(String message, Exception e) {
        this(message, null, e);
    }

    protected BaseException(String message, Integer code, Exception e) {
        super(e);
        this.message = message == null ? "" : message;
        this.code = code == null ? BASE_CODE : code;
    }

    @Override
    public String getMessage() {
        if (this.message != null && this.message.length() > 0) {
            return this.message;
        }
        return super.getMessage();
    }
}
```

可以看到其中有个构造方法是`protect`,是为了避免开发图省事直接传入错误码code（强制建立新的业务异常类）

然后如果有自定义的异常，需要建立新的异常类继承`BaseException`,比如定义一个`CustomBusinessException`
```java
public class CustomBusinessException extends BaseException {
    private static final long serialVersionUID = 1564935267302330109L;

    /**
     * 自定义业务异常错误码
     */
    private static final int ERROR_CODE = -1;

    public CustomBusinessException() {
        super("自定义义务异常", ERROR_CODE, null);
    }

    public CustomBusinessException(String message, Exception e) {
        super(message, ERROR_CODE, e);
    }
}

```

之所以用这种新建业务异常类的方式来定义错误码，一方面是为了提高代码可读性，另一方面也是为了方便对异常进行分类处理。

另外也可采用另外一种方式，将错误码定义为枚举类型，构造函数中传入枚举。

### 编写统一异常处理类

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
        } else {
            //其它异常，非自动抛出的,无需给前端返回具体错误内容（用户不需要看见空指针之类的异常信息）
            log.error("other exception:{}", e.getMessage(), e);
            be = new BaseException("系统异常，请稍后重试", e);
        }
        return new Result(be.getCode(), be.getMessage(), null);
    }
}

```

可以将各种统一处理逻辑定义在这里。本类中是使用一个方法来对所有的异常进行处理，在这个方法中再对异常类型进行区分。还有另外一种写法是定义多个handle方法，每个handle处理一种异常。

**另外在此还可以统一处理参数校验的异常，无需在接口代码中手动判断，下一篇中专门说明**

### 如何使用

通常代码会分为controller,service,dao三层，业务代码会写在service层中，因此一般是在service层中抛出异常。当然这三层中抛出的未捕获异常都能被`Exceptionhandle`捕获。

**本文所用到代码见：[github](https://github.com/FleyX/demo-project/tree/master/spring-boot/exceptionTest)**


