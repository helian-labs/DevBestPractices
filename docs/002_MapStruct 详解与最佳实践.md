# MapStruct 详解与最佳实践

> 高效、安全、零反射的 Java Bean 映射框架

- [MapStruct 官网](https://mapstruct.org/)

## 一、MapStruct 概述

### 1.1 什么是 MapStruct？

MapStruct 是一个基于 **注解处理器** 的 Java Bean 映射框架，在编译时生成类型安全的映射代码，相比运行时反射方案性能提升 **10-100 倍**。

**核心价值**：

- 🚀 **零运行时开销**：编译时生成纯 Java 代码
- 🛡️ **类型安全**：编译时验证映射正确性
- 🔍 **显式配置**：所有映射关系清晰可见
- 📦 **无依赖**：生成的标准代码不依赖 MapStruct

### 1.2 性能对比（基于 JMH 基准测试）

| 框架 | 平均操作耗时 (ns/op) | 内存分配 (B/op) |
|------|---------------------|----------------|
| **手动映射** | 15.2 | 0 |
| **MapStruct** | 16.8 | 0 |
| **ModelMapper** | 345.7 | 240 |
| **Dozer** | 512.3 | 480 |

> 测试环境：Java 17, 映射包含10个字段的对象

## 二、环境配置

### 2.1 项目依赖

**Maven**:

```xml
<properties>
    <org.mapstruct.version>1.5.5.Final</org.mapstruct.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>${org.mapstruct.version}</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>${org.mapstruct.version}</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Gradle**:

```groovy
plugins {
    id 'com.google.devtools.ksp' version '1.8.0-1.0.9' // Kotlin项目使用
}

dependencies {
    implementation 'org.mapstruct:mapstruct:1.5.5.Final'
    annotationProcessor 'org.mapstruct:mapstruct-processor:1.5.5.Final'
    // Kotlin项目使用：
    // ksp 'org.mapstruct:mapstruct-processor:1.5.5.Final'
}
```

## 三、核心注解与基础映射

### 3.1 基本映射接口

```java
@Mapper
public interface ProductMapper {
    
    // 声明Mapper实例获取方式（推荐使用依赖注入）
    ProductMapper INSTANCE = Mappers.getMapper(ProductMapper.class);
    
    @Mapping(source = "productName", target = "name")
    @Mapping(source = "stockQty", target = "quantity")
    @Mapping(target = "category", constant = "Electronics")
    ProductDTO productToProductDTO(Product product);
    
    // 反向映射
    @InheritInverseConfiguration
    Product productDTOToProduct(ProductDTO dto);
}
```

### 3.2 自动生成的实现类（编译后）

```java
public class ProductMapperImpl implements ProductMapper {
    
    @Override
    public ProductDTO productToProductDTO(Product product) {
        if (product == null) {
            return null;
        }
        
        ProductDTO productDTO = new ProductDTO();
        productDTO.setName(product.getProductName());
        productDTO.setQuantity(product.getStockQty());
        productDTO.setCategory("Electronics"); // 常量值
        
        return productDTO;
    }
}
```

### 3.3 字段映射规则

| 场景 | 解决方案 |
|------|----------|
| **同名字段** | 自动映射 |
| **不同名字段** | `@Mapping(source="srcField", target="targetField")` |
| **常量值** | `@Mapping(target="field", constant="value")` |
| **忽略字段** | `@Mapping(target="field", ignore=true)` |
| **默认值** | `@Mapping(source="field", target="field", defaultValue="default")` |

## 四、高级映射技术

### 4.1 类型转换

**自动类型转换**：

```java
// 自动处理基本类型转换
@Mapping(source = "price", target = "price") // double -> BigDecimal
@Mapping(source = "createDate", target = "createTimestamp") // Date -> long
```

**自定义转换方法**：

```java
@Mapper
public interface OrderMapper {
    
    @Mapping(source = "order", target = "summary", qualifiedByName = "toSummary")
    OrderDTO orderToDTO(Order order);
    
    @Named("toSummary")
    default String toOrderSummary(Order order) {
        return order.getId() + " - " + order.getCustomerName();
    }
}
```

### 4.2 嵌套对象映射

```java
public class Customer {
    private Address address;
    // getters/setters
}

public class CustomerDTO {
    private AddressDTO addressDTO;
    // getters/setters
}

@Mapper(uses = AddressMapper.class)
public interface CustomerMapper {
    CustomerDTO customerToDTO(Customer customer);
}

// 子映射器
@Mapper
public interface AddressMapper {
    AddressDTO addressToDTO(Address address);
}
```

### 4.3 集合映射

```java
@Mapper
public interface ProductMapper {
    
    // List映射
    List<ProductDTO> productsToDTOs(List<Product> products);
    
    // Map映射
    @Mapping(source = "key", target = "productKey")
    @Mapping(source = "value", target = "productValue")
    ProductEntryDTO mapEntryToDTO(Map.Entry<String, Product> entry);
    
    // Stream映射
    Stream<ProductDTO> productsToDTOs(Stream<Product> productStream);
}
```

## 五、最佳实践

### 5.1 架构设计原则

1. **单一职责映射器**
   - 每个Mapper接口专注一个领域
   - 避免创建"上帝映射器"

2. **分层映射策略**

   ```mermaid
   graph TD
   A[Controller层] -->|DTO| B(Service层)
   B -->|Entity| C[Repository层]
   C -->|Entity| D[数据库]
   B -->|BO| E[外部服务]
   ```

### 5.2 性能优化技巧

1. **组件模型配置**

   ```java
   // Spring集成
   @Mapper(componentModel = MappingConstants.ComponentModel.SPRING)
   public interface UserMapper {}
   
   // CDI集成
   @Mapper(componentModel = MappingConstants.ComponentModel.CDI)
   public interface ProductMapper {}
   ```

2. **引用映射器复用**

   ```java
   @Mapper(uses = { DateMapper.class, CurrencyConverter.class })
   public interface OrderMapper {}
   ```

### 5.3 复杂场景处理

**条件映射**：

```java
@Mapper
public interface EmployeeMapper {
    
    @Mapping(target = "bonus", 
             expression = "java( employee.getRating() > 90 ? 1000 : 0 )")
    EmployeeDTO employeeToDTO(Employee employee);
    
    @Condition
    default boolean isNotEmpty(String value) {
        return value != null && !value.isEmpty();
    }
}
```

**多源对象映射**：

```java
@Mapper
public interface AccountMapper {
    
    @Mapping(source = "user.name", target = "username")
    @Mapping(source = "account.balance", target = "balance")
    AccountDTO toDto(User user, Account account);
}
```

## 六、与框架整合

### 6.1 Spring Boot 集成

```java
@Configuration
public class MapperConfig {
    
    @Bean
    public UserMapper userMapper() {
        return Mappers.getMapper(UserMapper.class);
    }
}

@Service
public class UserService {
    
    private final UserMapper userMapper;
    
    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }
    
    public UserDTO getUser(Long id) {
        User user = userRepository.findById(id);
        return userMapper.userToDTO(user);
    }
}
```

### 6.2 JPA 实体映射

```java
@Entity
public class Order {
    @Id
    private Long id;
    @ManyToOne
    private Customer customer;
}

@Mapper(uses = CustomerMapper.class)
public interface OrderMapper {
    
    // 避免循环引用
    @Mapping(target = "customer.orders", ignore = true)
    OrderDTO orderToDTO(Order order);
}
```

## 七、测试策略

### 7.1 单元测试示例

```java
public class ProductMapperTest {
    
    private final ProductMapper mapper = ProductMapper.INSTANCE;
    
    @Test
    void testProductToDTO() {
        Product product = new Product("P100", "Laptop", 1299.99);
        
        ProductDTO dto = mapper.productToProductDTO(product);
        
        assertThat(dto.getName()).isEqualTo("Laptop");
        assertThat(dto.getPrice()).isEqualByComparingTo("1299.99");
        assertThat(dto.getCategory()).isEqualTo("Electronics");
    }
}
```

### 7.2 集成测试建议

1. 验证所有字段映射正确性
2. 测试null值处理逻辑
3. 验证集合映射的边界情况
4. 检查深度嵌套对象的映射

## 八、常见问题解决方案

| 问题 | 解决方案 |
|------|----------|
| **编译错误：未实现方法** | 检查@Mapper注解是否正确定义 |
| **嵌套对象未映射** | 添加子映射器到uses参数 |
| **字段忽略无效** | 确认target名称正确，检查继承层次 |
| **循环依赖** | 使用@Context参数或忽略反向引用 |
| **Lombok兼容问题** | 确保Lombok在MapStruct之前处理 |

## 九、替代方案比较

| 框架 | 类型安全 | 性能 | 学习曲线 | 适用场景 |
|------|----------|------|----------|----------|
| **MapStruct** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | 高性能关键系统 |
| **ModelMapper** | ⭐⭐ | ⭐⭐ | ⭐ | 快速原型开发 |
| **Orika** | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐ | 复杂映射需求 |
| **手动映射** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐ | 小型项目 |

## 十、进阶技巧

### 10.1 自定义映射器

```java
public class CustomDateMapper {
    
    public String asString(Date date) {
        return date != null ? new SimpleDateFormat("yyyy-MM-dd").format(date) : null;
    }
    
    public Date asDate(String date) {
        try {
            return date != null ? new SimpleDateFormat("yyyy-MM-dd").parse(date) : null;
        } catch (ParseException e) {
            throw new RuntimeException(e);
        }
    }
}

// 使用自定义映射器
@Mapper(uses = CustomDateMapper.class)
public interface EventMapper {}
```

### 10.2 装饰器模式

```java
@Mapper
@DecoratedWith(UserMapperDecorator.class)
public interface UserMapper {
    UserDTO userToDTO(User user);
}

// 装饰器实现
public abstract class UserMapperDecorator implements UserMapper {
    
    private final UserMapper delegate;
    
    public UserMapperDecorator(UserMapper delegate) {
        this.delegate = delegate;
    }
    
    @Override
    public UserDTO userToDTO(User user) {
        UserDTO dto = delegate.userToDTO(user);
        // 扩展逻辑
        dto.setFullName(user.getFirstName() + " " + user.getLastName());
        return dto;
    }
}
```

## 附录：核心注解参考

| 注解 | 作用 | 参数示例 |
|------|------|----------|
| `@Mapper` | 声明映射接口 | `componentModel = "spring"` |
| `@Mapping` | 字段映射配置 | `source="name", target="fullName"` |
| `@Mappings` | 多字段映射组合 | `{@Mapping(...), @Mapping(...)}` |
| `@InheritConfiguration` | 继承映射配置 | |
| `@InheritInverseConfiguration` | 反向映射配置 | |
| `@ValueMapping` | 枚举值映射 | `source="EXTRA", target="PREMIUM"` |
| `@BeanMapping` | Bean级别配置 | `ignoreByDefault=true` |
| `@Named` | 命名自定义方法 | `value="dateToString"` |

> **最佳实践总结**：MapStruct 是 Java Bean 映射的工业级解决方案，适用于高性能要求的系统。关键成功要素：
>
> 1. 严格遵循"编译时验证"原则
> 2. 建立分层映射体系
> 3. 结合CI/CD实施映射测试
> 4. 避免过度复杂映射逻辑
> 5. 团队统一配置规范
