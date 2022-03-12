---
title: 'SpringBoot 自定义参数校验 '
date: 2019-01-13 15:49:19
tags: SpringBoot
categories: 服务端开发
---

### 简单校验
在后端开发的过程中，验证前端参数的合法性是一个必不可少的步骤。但是参数验证会产生大量的样板代码，导致代码可读性差。使用 `validator-api` 可以简洁优雅的验证参数。我们来看一段代码：
```java
@GetMapping
public ResponseEntity index(@RequestParam("userOrderId") String userOrderId) {
    // 对订单ID做非空校验
    if (null == userOrderId || "".equals(userOrderId))
        return ResponseEntity.badRequest().build();
    return ResponseEntity.ok().build();
}
```

<!-- more -->

虽然这样做也能起到校验的作用，但是当参数比较多，参数比较复杂的时候会写大量的校验代码，既麻烦又影响可读性。SpringBoot 可以帮我们简化校验，`spring-boot-starter-web` 中默认集成了 `hibernate-validator` 包，它提供了一系列验证各种参数的方法，所以说 SpringBoot 已经帮我们想好要怎么解决这个问题了。上面的例子，可以简化成如下代码：
```Java
@GetMapping
public ResponseEntity index(@RequestParam("userOrderId") @NotBlank String userOrderId) {
    return ResponseEntity.ok().build();
}
```
`@NotBlank` 注解用来做非空判断，这样一来，代码量少了一大截，可读性也提高了。
不仅仅可以对包装类型做校验，如果接收参数是自定义对象，我们也是可以做校验的，例如：
```java
@Data
public class User {
    @Email(message = "错误的邮箱格式")
    private String email;

    @Size(min = 6, max = 30, message = "密码长度应当在 6 ~ 30 个字符之间")
    private String password;
}
```
```java
@GetMapping
public ResponseEntity index(@Validated @RequestBody User user) {
    return ResponseEntity.ok().build();
}
```
`@Validated` 注解开启对对象的校验。还有一些其他的注解，像 **@Min @Max @Length(min = 1, max = 2)** 等等，更多的注解可以去 **[官方文档](http://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/)** 查看。

### 自定义校验
但是在真实的业务逻辑中，这些基本的校验是不够的。更多的参数需要在数据库中进行对比，才能确定参数的合法性。比如在用户查看订单的时候，客户端传递过来一个订单的ID，这时候我们不仅要做非空验证，还要从数据库去判断 **订单ID是否存在**，这时候现有的验证注解就无法满足我们的需求了。我们需要实现一个 **自定义的注解**，用于验证订单ID是否存在，代码如下：

```java
/**
 * Author: vincent
 * Date: 2019-01-12 12:35:00
 * Comment: 自定义验证注解，用于验证订单ID的合法性
 */

@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UserOrderConstraintValidator.class)
@Target(value = { ElementType.FIELD, ElementType.METHOD, ElementType.PARAMETER })
public @interface UserOrderValidation {
    String message();
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```
想让自定义验证注解生效，需要实现 `ConstraintValidator` 接口。接口的第一个参数是 * 自定义注解类型**，第二个参数是 **被注解字段的类型**。这里因为订单ID 是 String 类型，我们第二个参数定义为 String 就可以了，需要提到的一点是 `ConstraintValidator` 接口的实现类无需添加 `@Component` 它在启动的时候就已经被加载到容器中了。
```java
@Slf4j
public class UserOrderConstraintValidator
        implements ConstraintValidator<UserOrderValidation, String> {

    @Resource
    private UserOrderService userOrderService;

    /**
     * 初始化方法，我们可以做一些初始化的工作
     * @param constraintAnnotation
     */
    @Override
    public void initialize(UserOrderValidation constraintAnnotation) {
        log.info("initialize constraint");
    }

    /**
     * 判断 value 是否为空，不为空的情况下去数据库查询订单ID是否邮箱
     * @param value 被注解的字段的值，也就是 userOrderId
     * @param context
     * @return
     */
    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        return StringUtils.isNotEmpty(userOrderId) && userOrderService.exists(userOrderId);
    }
}
```

自定义校验注解和默认实现的注解的使用方法是一致的，例如我们在控制层去校验请求的订单ID是否在数据库中存在。
```java
@RestController
@RequestMapping("/user/orders")
public class UserOrderController {
    @GetMapping("/{id}")
    public ResponseEntity<?> show(@PathVariable("id")
                                  @UserOrderValidation String userOrderId) {
        return ResponseEntity.ok().build();
    }
}
```
