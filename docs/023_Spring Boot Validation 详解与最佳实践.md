# Spring Boot Validation 详解与最佳实践

## 1. Validation 简介

### 1.1 什么是 Spring Boot Validation?

Spring Boot Validation 是基于 **Bean Validation 2.0 (JSR 380)** 规范的实现，提供了一套强大的数据验证机制。它允许开发者通过声明式注解在模型对象上定义验证规则，确保应用程序接收的数据符合业务要求和安全标准。

### 1.2 核心价值

| 功能 | 描述 | 业务价值 |
|------|------|----------|
| **数据完整性** | 确保输入数据符合预期格式 | 防止无效数据进入系统 |
| **业务规则** | 强制执行业务约束 | 保证业务流程正确性 |
| **安全性** | 防止恶意输入攻击 | 增强应用安全性 |
| **代码简洁** | 声明式验证减少代码量 | 提高开发效率 |
| **统一错误** | 标准化错误响应格式 | 改善API用户体验 |

## 2. 快速集成

### 2.1 添加依赖

```xml
<dependencies>
    <!-- Spring Boot Validation Starter -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    
    <!-- Web 支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### 2.2 启用验证

```java
@RestController
@Validated // 启用方法级验证
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<User> createUser(@Valid @RequestBody User user) {
        // 业务逻辑
        return ResponseEntity.ok(userService.save(user));
    }
}
```

## 3. 内置验证注解全解

### 3.1 标准注解 (javax.validation.constraints)

| 注解 | 描述 | 示例 |
|------|------|------|
| `@NotNull` | 值不能为null | `@NotNull String name` |
| `@NotBlank` | 字符串不能为空或空白 | `@NotBlank String username` |
| `@NotEmpty` | 集合/数组/字符串不能为空 | `@NotEmpty List<String> roles` |
| `@Size` | 长度范围限制 | `@Size(min=6, max=20) String password` |
| `@Min` | 最小值 | `@Min(18) int age` |
| `@Max` | 最大值 | `@Max(100) int discount` |
| `@DecimalMin` | 小数值最小值 | `@DecimalMin("0.0") BigDecimal price` |
| `@DecimalMax` | 小数值最大值 | `@DecimalMax("100.0") BigDecimal score` |
| `@Digits` | 数字位数限制 | `@Digits(integer=3, fraction=2) BigDecimal amount` |
| `@Pattern` | 正则表达式验证 | `@Pattern(regexp="[A-Za-z0-9]+") String code` |
| `@Email` | 邮箱格式验证 | `@Email String email` |
| `@Positive` | 正数 | `@Positive int quantity` |
| `@PositiveOrZero` | 正数或零 | `@PositiveOrZero int stock` |
| `@Negative` | 负数 | `@Negative int debt` |
| `@NegativeOrZero` | 负数或零 | `@NegativeOrZero int offset` |
| `@Future` | 未来日期 | `@Future LocalDate eventDate` |
| `@FutureOrPresent` | 未来或当前日期 | `@FutureOrPresent LocalDate startDate` |
| `@Past` | 过去日期 | `@Past LocalDate birthDate` |
| `@PastOrPresent` | 过去或当前日期 | `@PastOrPresent LocalDate reportDate` |

### 3.2 Hibernate 扩展注解

| 注解 | 描述 | 示例 |
|------|------|------|
| `@Length` | 字符串长度 | `@Length(min=2, max=50) String name` |
| `@Range` | 数字范围 | `@Range(min=1, max=10) int rating` |
| `@URL` | URL格式 | `@URL String website` |
| `@CreditCardNumber` | 信用卡号 | `@CreditCardNumber String card` |
| `@SafeHtml` | 安全HTML | `@SafeHtml String description` |
| `@UniqueElements` | 集合元素唯一 | `@UniqueElements List<String> tags` |
| `@DurationMin` | 最小持续时间 | `@DurationMin(seconds=30) Duration timeout` |
| `@DurationMax` | 最大持续时间 | `@DurationMax(minutes=10) Duration delay` |

## 4. 验证注解使用实战

### 4.1 基本模型验证

```java
public class User {
    
    @NotBlank(message = "用户名不能为空")
    @Size(min=3, max=20, message = "用户名长度需在3-20字符之间")
    private String username;
    
    @NotBlank(message = "密码不能为空")
    @Pattern(regexp = "^(?=.*[A-Za-z])(?=.*\\d)[A-Za-z\\d]{8,}$", 
             message = "密码必须至少8位，包含字母和数字")
    private String password;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Past(message = "出生日期必须是过去日期")
    private LocalDate birthDate;
    
    @Range(min=0, max=150, message = "年龄必须在0-150之间")
    private Integer age;
    
    @AssertTrue(message = "必须接受条款")
    private Boolean termsAccepted;
    
    // Getters and Setters
}
```

### 4.2 嵌套对象验证

```java
public class Order {
    
    @NotNull(message = "订单ID不能为空")
    private Long id;
    
    @Valid // 启用嵌套验证
    @NotNull(message = "客户信息不能为空")
    private Customer customer;
    
    @Valid // 验证集合内元素
    @NotEmpty(message = "订单项不能为空")
    private List<@Valid OrderItem> items;
}

public class Customer {
    @NotBlank
    private String name;
    
    @Pattern(regexp = "\\d{11}", message = "手机号格式不正确")
    private String phone;
}

public class OrderItem {
    @NotBlank
    private String productId;
    
    @Min(1)
    private Integer quantity;
    
    @DecimalMin("0.01")
    private BigDecimal price;
}
```

### 4.3 方法参数验证

```java
@RestController
@Validated
public class ProductController {
    
    @GetMapping("/products")
    public List<Product> searchProducts(
            @Size(min=2, max=50) @RequestParam String keyword,
            @Min(0) @RequestParam(defaultValue = "0") int page,
            @Range(min=1, max=100) @RequestParam(defaultValue = "20") int size) {
        // 业务逻辑
    }
    
    @PutMapping("/products/{id}/price")
    public void updatePrice(
            @PathVariable Long id,
            @DecimalMin("0.01") @RequestParam BigDecimal price) {
        // 业务逻辑
    }
}
```

## 5. 自定义验证注解

### 5.1 创建自定义注解

```java
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = StrongPasswordValidator.class)
public @interface StrongPassword {
    String message() default "密码强度不足";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    
    // 可选参数
    int minLength() default 8;
    int minUpperCase() default 1;
    int minLowerCase() default 1;
    int minDigits() default 1;
    int minSpecialChars() default 1;
}
```

### 5.2 实现验证逻辑

```java
public class StrongPasswordValidator implements ConstraintValidator<StrongPassword, String> {
    
    private int minLength;
    private int minUpperCase;
    private int minLowerCase;
    private int minDigits;
    private int minSpecialChars;
    
    @Override
    public void initialize(StrongPassword constraintAnnotation) {
        this.minLength = constraintAnnotation.minLength();
        this.minUpperCase = constraintAnnotation.minUpperCase();
        this.minLowerCase = constraintAnnotation.minLowerCase();
        this.minDigits = constraintAnnotation.minDigits();
        this.minSpecialChars = constraintAnnotation.minSpecialChars();
    }
    
    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) return false;
        
        if (password.length() < minLength) return false;
        
        int upperCaseCount = 0;
        int lowerCaseCount = 0;
        int digitCount = 0;
        int specialCharCount = 0;
        
        for (char c : password.toCharArray()) {
            if (Character.isUpperCase(c)) upperCaseCount++;
            else if (Character.isLowerCase(c)) lowerCaseCount++;
            else if (Character.isDigit(c)) digitCount++;
            else specialCharCount++;
        }
        
        return upperCaseCount >= minUpperCase &&
               lowerCaseCount >= minLowerCase &&
               digitCount >= minDigits &&
               specialCharCount >= minSpecialChars;
    }
}
```

### 5.3 使用自定义注解

```java
public class RegistrationForm {
    
    @StrongPassword(
        minLength = 10,
        minUpperCase = 2,
        minDigits = 2,
        minSpecialChars = 1,
        message = "密码必须至少10位，包含2个大写字母、2个数字和1个特殊字符"
    )
    private String password;
    
    // 其他字段
}
```

## 6. 分组验证与条件验证

### 6.1 验证分组定义

```java
// 定义验证组接口
public interface ValidationGroups {
    interface Create {}
    interface Update {}
    interface Patch {}
}
```

### 6.2 分组验证应用

```java
public class Product {
    @Null(groups = Create.class, message = "ID必须为空")
    @NotNull(groups = {Update.class, Patch.class}, message = "ID不能为空")
    private Long id;
    
    @NotBlank(groups = Create.class, message = "产品名称不能为空")
    @Size(min=2, max=100, groups = {Create.class, Update.class})
    private String name;
    
    @NotNull(groups = Create.class)
    @DecimalMin(value = "0.01", groups = Create.class)
    private BigDecimal price;
    
    // 其他字段
}
```

### 6.3 控制器中使用分组

```java
@RestController
@RequestMapping("/products")
public class ProductController {
    
    @PostMapping
    public Product createProduct(
            @Validated(ValidationGroups.Create.class) 
            @RequestBody Product product) {
        // 创建逻辑
    }
    
    @PutMapping("/{id}")
    public Product updateProduct(
            @PathVariable Long id,
            @Validated(ValidationGroups.Update.class) 
            @RequestBody Product product) {
        // 更新逻辑
    }
    
    @PatchMapping("/{id}")
    public Product patchProduct(
            @PathVariable Long id,
            @Validated(ValidationGroups.Patch.class) 
            @RequestBody Product product) {
        // 部分更新逻辑
    }
}
```

## 7. 验证异常处理与响应

### 7.1 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidationException(
            MethodArgumentNotValidException ex) {
        
        List<FieldError> fieldErrors = ex.getBindingResult().getFieldErrors();
        
        Map<String, String> errors = fieldErrors.stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                fieldError -> fieldError.getDefaultMessage() != null ? 
                    fieldError.getDefaultMessage() : "Invalid value"
            ));
        
        ErrorResponse response = new ErrorResponse(
            "VALIDATION_FAILED", 
            "请求参数验证失败",
            errors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
    
    // 处理其他验证相关异常
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraintViolation(
            ConstraintViolationException ex) {
        
        Map<String, String> errors = ex.getConstraintViolations().stream()
            .collect(Collectors.toMap(
                violation -> violation.getPropertyPath().toString(),
                ConstraintViolation::getMessage
            ));
        
        ErrorResponse response = new ErrorResponse(
            "INVALID_PARAMETER", 
            "方法参数验证失败",
            errors
        );
        
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
}

// 错误响应模型
public class ErrorResponse {
    private String code;
    private String message;
    private Map<String, String> errors;
    private Instant timestamp = Instant.now();
    
    // 构造函数、getters、setters
}
```

### 7.2 统一错误响应示例

```json
{
  "code": "VALIDATION_FAILED",
  "message": "请求参数验证失败",
  "timestamp": "2023-08-15T10:30:45.123Z",
  "errors": {
    "email": "邮箱格式不正确",
    "password": "密码必须至少8位，包含字母和数字",
    "birthDate": "出生日期必须是过去日期"
  }
}
```

## 8. 高级验证技巧

### 8.1 跨字段验证

```java
@Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = FieldMatchValidator.class)
public @interface FieldMatch {
    String message() default "字段值不匹配";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
    
    String first();
    String second();
    
    @Target({ElementType.TYPE, ElementType.ANNOTATION_TYPE})
    @Retention(RetentionPolicy.RUNTIME)
    @interface List {
        FieldMatch[] value();
    }
}

public class FieldMatchValidator implements ConstraintValidator<FieldMatch, Object> {
    
    private String firstFieldName;
    private String secondFieldName;
    
    @Override
    public void initialize(FieldMatch constraintAnnotation) {
        firstFieldName = constraintAnnotation.first();
        secondFieldName = constraintAnnotation.second();
    }
    
    @Override
    public boolean isValid(Object value, ConstraintValidatorContext context) {
        try {
            BeanWrapper beanWrapper = new BeanWrapperImpl(value);
            Object firstValue = beanWrapper.getPropertyValue(firstFieldName);
            Object secondValue = beanWrapper.getPropertyValue(secondFieldName);
            
            return Objects.equals(firstValue, secondValue);
        } catch (Exception e) {
            return false;
        }
    }
}
```

### 8.2 使用示例

```java
@FieldMatch(first = "password", second = "confirmPassword", message = "两次密码输入不一致")
public class PasswordResetForm {
    
    @NotBlank
    private String password;
    
    @NotBlank
    private String confirmPassword;
    
    // Getters and Setters
}
```

### 8.3 条件验证

```java
public class PaymentRequest {
    
    @NotNull
    private PaymentMethod paymentMethod;
    
    @NotBlank
    private String cardNumber;
    
    @NotNull
    private Month expiryMonth;
    
    @NotNull
    private Year expiryYear;
    
    @NotBlank
    @Size(min=3, max=4)
    private String cvv;
    
    @NotBlank(groups = PaypalGroup.class)
    @Email(groups = PaypalGroup.class)
    private String paypalEmail;
    
    // 条件验证组
    public interface PaypalGroup {}
    public interface CreditCardGroup {}
    
    // 自定义验证器
    @AssertTrue(message = "信用卡信息必须完整")
    public boolean isCreditCardInfoValid() {
        if (paymentMethod == PaymentMethod.CREDIT_CARD) {
            return cardNumber != null && 
                   expiryMonth != null && 
                   expiryYear != null && 
                   cvv != null;
        }
        return true;
    }
    
    @AssertTrue(message = "PayPal邮箱必须提供")
    public boolean isPaypalEmailValid() {
        if (paymentMethod == PaymentMethod.PAYPAL) {
            return paypalEmail != null && !paypalEmail.isBlank();
        }
        return true;
    }
}
```

## 9. 最佳实践

### 9.1 验证策略

1. **分层验证**：
   - 控制器层：验证API输入
   - 服务层：验证业务规则
   - 持久层：验证数据库约束

2. **验证粒度**：
   - 简单验证：使用内置注解
   - 复杂规则：使用自定义验证器
   - 业务规则：在服务层实现

3. **错误处理**：
   - 统一错误响应格式
   - 本地化错误消息
   - 记录验证失败详情

### 9.2 性能优化

```java
@Configuration
public class ValidationConfig {
    
    // 自定义验证器自动注册
    @Bean
    public LocalValidatorFactoryBean validator() {
        LocalValidatorFactoryBean factory = new LocalValidatorFactoryBean();
        factory.setValidationMessageSource(messageSource());
        return factory;
    }
    
    // 国际化消息源
    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource messageSource = 
            new ReloadableResourceBundleMessageSource();
        messageSource.setBasename("classpath:messages");
        messageSource.setDefaultEncoding("UTF-8");
        return messageSource;
    }
    
    // 方法验证后处理器
    @Bean
    public MethodValidationPostProcessor methodValidationPostProcessor() {
        MethodValidationPostProcessor processor = new MethodValidationPostProcessor();
        processor.setValidator(validator());
        return processor;
    }
}
```

### 9.3 安全验证

```java
// 防止批量赋值攻击
public class UserDTO {
    
    @NotBlank
    private String username;
    
    @NotBlank
    private String password;
    
    // 管理员字段，普通用户不能设置
    @NotNull(groups = AdminGroup.class)
    private Boolean admin;
    
    public interface AdminGroup {}
}

// 控制器
@PostMapping("/users")
public User createUser(
        @Valid @RequestBody UserDTO userDTO,
        @AuthenticationPrincipal UserPrincipal principal) {
    
    // 根据权限应用分组验证
    if (principal.isAdmin()) {
        validator.validate(userDTO, UserDTO.AdminGroup.class);
    }
    
    return userService.createUser(userDTO);
}
```

## 10. 生产环境检查清单

### 10.1 验证配置

- [ ] 启用全局 `@Validated`
- [ ] 配置统一异常处理器
- [ ] 实现本地化错误消息
- [ ] 设置合理的验证组

### 10.2 安全防护

- [ ] 防止过度暴露验证细节
- [ ] 限制批量赋值攻击
- [ ] 验证敏感字段格式（密码、支付信息）
- [ ] 实现跨字段验证关键操作

### 10.3 性能监控

- [ ] 监控验证失败率
- [ ] 记录复杂验证执行时间
- [ ] 优化正则表达式性能
- [ ] 避免嵌套对象深度验证

### 10.4 测试覆盖

- [ ] 单元测试所有验证规则
- [ ] 集成测试API验证行为
- [ ] 边界值测试验证逻辑
- [ ] 测试自定义验证器

## 总结与进阶

### 核心原则

1. **声明优于命令**：使用注解声明验证规则
2. **关注点分离**：验证逻辑与业务逻辑分离
3. **深度防御**：多层验证确保数据安全
4. **用户体验**：提供清晰友好的错误信息

### 进阶主题

- **动态验证**：根据运行时条件调整验证规则
- **自定义约束组合**：创建可复用验证注解组
- **验证器缓存**：优化自定义验证器性能
- **异步验证**：并行执行复杂验证逻辑
- **分布式验证**：跨服务边界共享验证规则

> **架构师提示**：将核心验证规则抽象为独立模块，在微服务架构中通过共享库实现验证逻辑的统一管理，确保系统范围内的数据一致性。

## 附录：完整注解速查表

### 标准注解速查

| 注解 | 适用类型 | 描述 |
|------|----------|------|
| `@NotNull` | 任意 | 值不能为null |
| `@NotBlank` | String | 字符串非空且长度>0 |
| `@NotEmpty` | String, Collection, Map, Array | 非空 |
| `@Size` | String, Collection, Map, Array | 大小在min和max之间 |
| `@Min` | 数值类型 | 最小值 |
| `@Max` | 数值类型 | 最大值 |
| `@DecimalMin` | BigDecimal, BigInteger, String | 小数值最小值 |
| `@DecimalMax` | BigDecimal, BigInteger, String | 小数值最大值 |
| `@Digits` | 数值类型 | 整数位和小数位限制 |
| `@Pattern` | String | 正则表达式匹配 |
| `@Email` | String | 邮箱格式验证 |
| `@Positive` | 数值类型 | 正数 |
| `@PositiveOrZero` | 数值类型 | 正数或零 |
| `@Negative` | 数值类型 | 负数 |
| `@NegativeOrZero` | 数值类型 | 负数或零 |
| `@Future` | 日期时间 | 未来日期时间 |
| `@FutureOrPresent` | 日期时间 | 未来或当前日期时间 |
| `@Past` | 日期时间 | 过去日期时间 |
| `@PastOrPresent` | 日期时间 | 过去或当前日期时间 |

### Hibernate 注解速查

| 注解 | 适用类型 | 描述 |
|------|----------|------|
| `@Length` | String | 字符串长度限制 |
| `@Range` | 数值类型 | 数值范围限制 |
| `@URL` | String | URL格式验证 |
| `@CreditCardNumber` | String | 信用卡号验证 |
| `@SafeHtml` | String | 安全HTML验证 |
| `@UniqueElements` | Collection | 集合元素唯一 |
| `@DurationMin` | Duration | 最小持续时间 |
| `@DurationMax` | Duration | 最大持续时间 |
| `@EAN` | String | EAN/UPC编码验证 |
| `@ISBN` | String | ISBN号验证 |
| `@Currency` | MonetaryAmount | 货币单位验证 |
