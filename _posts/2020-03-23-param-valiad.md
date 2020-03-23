---
layout: post
title:  "JAVA开发中，API接口如何优雅的进行参数校验？"
categories: it
tags:  [ IT,JAVA]
excerpt: 做过java web 应用开发的后端研发都应该做过API接口的参数校验，很多都是将校验逻辑放到展示层做拦截处理，但是涉及到参数过多的时候，就会显得Controller接口特别臃肿，本篇讲解的就是如何利用进行参数校验拦截。

---

我们在做与前端交互的后天应用接口的过程中，会定义很多对应的接口及参数，而有些接口参数有一定的要求，比如最大值、最小值约束，涉及到手机号、身份证等规则校验，参数是否为空等等，刚进入java领域的小伙伴们大部分第一时间都是针对每个参数进行一一判别校验、当接口参数校验，判断逻辑代码就显得极其冗杂，不美观也不便于阅读。其实我们可以使用*validation*结合spring进行参数的校验，并且通过Controller增强器@ControllerAdvice来定义全局异常处理类，配合@ExceptionHandler可以更加优雅的处理参数的校验问题。

话不多述，看代码：

## 1.新增全局异常处理

```java
@ControllerAdvice
@ResponseBody
public class CommonExceptionAdvice {
    private static Logger logger = LoggerFactory.getLogger(CommonExceptionAdvice.class);
    @Autowired
    private MessageSource messageSource;
    
   @ResponseStatus(HttpStatus.OK)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public BaseResponse handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        logger.error("参数验证失败:{}", e.getMessage());
        BindingResult result = e.getBindingResult();
        List<FieldError> errorList = e.getBindingResult().getFieldErrors();
        List<String> errorMessages = errorList.stream().map(x->{
            String itemMessage= messageSource.getMessage(x.getDefaultMessage(), null, x.getDefaultMessage(), LocaleContextHolder.getLocale());
            return String.format("%s", itemMessage);
        }).collect(Collectors.toList());
        return BaseResponse.errorResponse(errorMessages.toString());
    }
 }
```



## 2.定义参数

###基础请求参数：

```java
/*
 * Copyright (C) 2020 Baidu, Inc. All Rights Reserved.
 */
package com.itstabber.blog.example.request;

import io.swagger.annotations.ApiModel;
import lombok.Data;

import javax.validation.constraints.Max;
import javax.validation.constraints.Min;
import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import java.util.List;

/**
 * TODO: 请添加描述
 *
 * @author Stabber
 * @date 2020/3/24 00:33
 * @since 1.0.0
 */
@Data
public class BaseRequest {
    @NotNull(message = "参数i不能为空")
    private Integer i;
    @NotBlank(message="参数s不能为空")
    private String s;

    @Min(value =0,message = "最小值不能小于0")
    private int min;

    @Max(value=100,message = "最大值不能大约100")
    private int max;


}

```

@NotNull 用于约束请求参数不能为Null

@NotBlank针对于String类型，不能为空串或者全部空格的参数

@Min @Max 可以限制参数的最小、最大值

###复杂请求参数

```java
@ApiModel("订单申请信息")
public class OrderRequest extends BaseRequest{

    @ApiModelProperty(name="订单编号")
    @Length( max= 10,message = "订单编号最大10位")
    private String orderNo;

    @Valid
    @ApiModelProperty(name="商品列表信息")
    private List<ProductRequest> productRequestList;

}

@Data
@ApiModel("商品申请信息")
public class ProductRequest {

    @NotBlank
    @ApiModelProperty(name = "商品名称")
    private String productName;
    @NotBlank
    @ApiModelProperty(name = "商品编码")
    private String productCode;
}
```

@Length 可以定义min,max属性约束字段的最小、最大长度

@Valid 放在对象属性上，可以是对象内的参数约束生效

## 3.定义controller

```java
@RequestMapping("/hello")
@RestController
@Api(tags = "HelloWorld 入口")
public class HelloWorld {

    @GetMapping("/world")
    @ApiOperation(value = "helloWorld")
    public String helloWorld() {
        return "hello world!";
    }


    @PostMapping("/test/param")
    @ApiOperation(value = "testParam")
    public BaseResponse<String> testParam(
            @Valid @RequestBody BaseRequest baseRequest
    ) {
        return BaseResponse.successResponse("参数校验成功");
    }

    @ApiOperation(value = "testOrderParam")
    @PostMapping("/test/order")
    public BaseResponse<String> testOrderParam(
            @Validated @RequestBody OrderRequest baseRequest
    ) {
        return BaseResponse.successResponse("参数校验成功");
    }
}
```

上面源码可以看到，testParam接口和testOrderParam两个接口使用的注解不一样，一个是@Valid、另一个是@Validated，两个稍有区别，前者使得当前对象内属性校验生效，如果包含对象属性，则对象属性自身的属性校验无法生效，而后者注释的对象，对象内部配合@Valid可以使得对象内部的对象属性的属性校验规则生效。

## 请求效果

项目引入swagger2，便于演示结果：

### 普通参数请求

#### 请求参数

![bP36OP](http://image.itstabber.com/uPic/bP36OP.png)

#### 响应结果：

![hF20hi](http://image.itstabber.com/uPic/hF20hi.png)

请求参数我全部设置的不满足规则，响应结果输出了所有的校验不通过的信息。

### 复杂参数请求

#### 请求参数

![HQeEJS](http://image.itstabber.com/uPic/HQeEJS.png)

#### 响应结果

![kQHHDy](http://image.itstabber.com/uPic/kQHHDy.png)

如上可以看到，商品编号属性是OrderRequest中的productRequestList的属性，依然能够成功校验，另外，productCode和productName，传递参数的时候都是给到的空串，由于两者的使用的注解不一样，其中商品名称被拦截，商品编码规则通过，这也是针对String类型参数时，@NotBlank与@NotNull的不同之处。

本文就简单的介绍在这里，有兴趣的小伙伴可以访问我的github地址，[本文的源码请点击](https://github.com/gdspw/blog-demo );

源码的全局异常处理类还有很多其他的异常拦截处理，可供参考。

有什么问题需要问的或者可供讨论的，欢迎在文章下方评论区留言！



