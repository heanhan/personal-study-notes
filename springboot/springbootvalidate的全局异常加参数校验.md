### springboot 下全局异常的处理和参数校验

代码的项目地址：

#### 一、全局异常

springboot已经做了全局异常处理，但是对于我们开发者而言不是和符合项目实际的使用，因此我们需要对这些异常进行统一的捕获并处理。Springboot 中有一个ControllerAdvice 的注解，使用该注解表示开启了全局异常的捕获，我们只需要自定义一个方法使用ExceptionHandler注解后定义捕获异常的类型即可对这些捕获的异常进行统一的处理。 

#### 二、使用spring Validation对请求的参数进行校验

##### 简单的使用

java api 规范（JSR303）定义了 Bean 校验的标准 validation-api,但没有提供实现。hibernate validation 是对这个规范的实现，并增加了校验注解@Email、@Lenth等等，Spring Valiation是对hibernate validation的二次封装，用于支持spring mvc 参数自动校验。

##### 引入依赖

如果springboot 版本小于2.3.X,spring-boot-starter-web会自动传入hibernate-valiationg的依赖。如果使用spring-boot 版本大于2.3.x 则需要手动引入依赖。

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.1.Final</version>
</dependency>
```

对于web项目来说，为了防止非法参数对业务造成影响，在controller 层一定要做参数校验，大部分请求的校验场景如下：

1. POST、PUT请求，使用@RequestBody 传递参数。
2. GET请求，使用@RequestParam/@Pathvariable传递参数

###### **@RequestBody 接受参数**

`POST`、`PUT`请求一般会使用`requestBody`传递参数，这种情况下，后端使用 对象**进行接收。**只要给对象加上`@Validated`注解就能实现自动参数校验**。比如，有一个保存`User`的接口，要求`userName`长度是`2-10`，`account`和`password`字段长度是`6-20`。如果校验失败，会抛出`MethodArgumentNotValidException`异常，`Spring`默认会将其转为`400（Bad Request）`请求。

```java
@Data
public class User {

    /**
     *用户id
     */
    @Min(value = 10000000000000000L, groups = Update.class)
    private Long userId;

    /**
     *    用户名
     */
    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 2, max = 10, groups = {Save.class, Update.class})
    private String userName;

    /**
     * 账户
     */
    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String account;

    /**
     * 密码
     */
    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String password;

    /**
     * 工作
     */
    @NotNull(groups = {Save.class, Update.class})
    @Valid
    private Job job;

    /**
     * 保存的时候校验分组
     */
    public interface Save {
    }

    /**
     * 更新的时候校验分组
     */
    public interface Update {
    }
}
```

在方法上声明注解校验

```java
@PostMapping("/save")
public ResultBody saveUser(@RequestBody @Validated User user) {
    return ResultBody.success();
}
```

**注：** 这种情况下，**使用`@Valid`和`@Validated`都可以**。



**@RequestParam、@PathVariable 接受参数**

`GET`请求一般会使用`requestParam/PathVariable`传参。如果参数比较多 (比如超过 6 个)，还是推荐使 对象接收。否则，推荐将一个个参数平铺到方法入参中。在这种情况下，**必须在`Controller`类上标注`@Validated`注解，并在入参上声明约束注解 (如`@Min`,@NotNull 等)**。如果校验失败，会抛出`ConstraintViolationException`异常，代码如下：

```java
@RequestMapping("/api/user")
@RestController
@Validated
public class UserController {

    @GetMapping("{userId}")
    public ResultBody detail(@PathVariable("userId") @Min(10000000000000000L) Long userId) {

        UserDTO userDTO = new UserDTO();
        userDTO.setUserId(userId);
        userDTO.setAccount("11111111111111111");
        userDTO.setUserName("xixi");
        userDTO.setAccount("11111111111111111");
        return ResultBody.success(userDTO);
    }

    @GetMapping("getByAccount")
    public ResultBody getByAccount(@Length(min = 6, max = 20) @NotNull String  account) {

        UserDTO userDTO = new UserDTO();
        userDTO.setUserId(10000000000000003L);
        userDTO.setAccount(account);
        userDTO.setUserName("xixi");
        userDTO.setAccount("11111111111111111");
        return ResultBody.success(userDTO);
    }
}
```

**参数校验 一般与 统一异常一起使用** 做法如下：

如果参数校验失败，会抛出MethodArgumentNotValdException 或者 constraintViolationException异常。在实际项目开发中，通常会使用统一异常处理返回来的一个更有好的提示，比如我们系统中要求发送什么异常，http 的状态码必须返回200 由业务业务吗去区分系统的异常情况。



###### 进阶使用

1、分组校验

在实际项目中，可能多个方法需要使用同一个对象来接收参数，而不同的方法的校验规则很可能是不一样的，这个时候，就爱你但的在对象上的子端上的加约束注解是无法解决这个问题的。因此spring-validation支持了分组校验的功能，专门用来解决这类问题，比如，保存User 的userId 要求为空 ，更新的时候不能为空，所以在约束注解上声明适用的groups 

```java
@Data
public class User {

    @Min(value = 10000000000000000L, groups = Update.class)
    private Long userId;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 2, max = 10, groups = {Save.class, Update.class})
    private String userName;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String account;

    @NotNull(groups = {Save.class, Update.class})
    @Length(min = 6, max = 20, groups = {Save.class, Update.class})
    private String password;

    public interface Save {
    }

    public interface Update {
    }
}
```

- **`@Validated`注解上指定校验分组**

  ```java
  @PostMapping("/save")
  public ResultBody saveUser(@RequestBody @Validated(User.Save.class) User user) {
  
      return ResultBody.success();
  }
  
  @PostMapping("/update")
  public ResultBody updateUser(@RequestBody @Validated(User.Update.class) User user) {
  
      return ResultBody.success();
  }
  ```



2、嵌套校验

前面的示例中，类里面的字段都是`基本数据类型`和`String`类型。但是实际场景中，有可能某个字段也是一个对象，这种情况先，可以使用`嵌套校验`。假如，上面保存`User`信息的时候同时还带有`另一个对象信息。需要注意的是，**此时校验的对象类的对应字段必须标记`@Valid`注解**。

3、集合校验

如果请求体直接传递了`json`数组给后台，并希望对数组中的每一项都进行参数校验。此时，如果我们直接使用`java.util.Collection`下的`list`或者`set`来接收数据，参数校验并不会生效！我们可以使用自定义`list`集合来接收参数

- 包装List 类型，并声明@Valid 注解

```java
public class ValidationList<E> implements List<E> {

    @Delegate
    @Valid
    public List<E> list = new ArrayList<>();

    @Override
    public String toString() {
        return list.toString();
    }
}
```

注：`@Delegate`注解受`lombok`版本限制，`1.18.6`以上版本可支持。如果校验不通过，会抛出`NotReadablePropertyException`，同样可以使用统一异常进行处理。

比如，我们需要一次性保存多个`User`对象，`Controller`层的方法可以这么写：

```java
@PostMapping("/saveList")
public Result saveList(@RequestBody @Validated(User.Save.class) ValidationList<User> userList) {

    return Result.ok();
}
```

4、自定义约束注解

业务需求总是比框架提供的这些简单校验要复杂的多，我们可以自定义校验来满足我们的需求。自定义`spring validation`非常简单，假设我们自定义`加密id`（由数字或者`a-f`的字母组成，`32-256`长度）校验，主要分为两步：

- 自定义约束注解

```java
@Target({METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER})
@Retention(RUNTIME)
@Documented
@Constraint(validatedBy = {EncryptIdValidator.class})
public @interface EncryptId {

    String message() default "加密id格式错误";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

-  实现`ConstraintValidator`接口编写约束校验器

  ```java
  public class EncryptIdValidator implements ConstraintValidator<EncryptId, String> {
  
      private static final Pattern PATTERN = Pattern.compile("^[a-f\\d]{32,256}$");
  
      @Override
      public boolean isValid(String value, ConstraintValidatorContext context) {
  
          if (value != null) {
              Matcher matcher = PATTERN.matcher(value);
              return matcher.find();
          }
          return true;
      }
  }
  ```

  这样我们就可以使用`@EncryptId`进行参数校验了





##### @Valid 与@Validated 区别

| **区别**     | @Valid                                          | @Validated     |
| ------------ | ----------------------------------------------- | -------------- |
| 提供者       | JSR-303规范                                     | Spring         |
| 是否支持分组 | 不支持                                          | 支持           |
| 标注位置     | METHOD, FIELD, CONSTRUCTOR, PARAMETER, TYPE_USE | ETER, TYPE_USE |
| 嵌套校验     | 支持                                            | 不支持         |



##### 实现原理

@RequestBody 参数校验实现原理

在spring-mvc中，@RequestResponseBodyMethodProcessor 用于解析@RequestBody 标注的参数以及处理@RequestBody标注方法的返回值的。鲜艳执行参数的校验的逻辑肯定就在解析参数方法resolveArgument()中。



