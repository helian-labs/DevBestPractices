# Lombok 详解与最佳实践

> 简化 Java 开发，提升代码可读性与维护性

## 一、Lombok 概述

### 1.1 什么是 Lombok？

Lombok 是一个 Java 库，通过注解自动生成 Java 代码（如 getter/setter、构造方法、日志声明等），减少样板代码，提高开发效率。

**解决的问题**：

- 减少重复性代码（如 POJO 的 getter/setter）
- 简化日志记录器的初始化
- 自动生成 Builder 模式等设计模式实现
- 提高代码可读性

### 1.2 核心优势

| 优势 | 说明 |
|------|------|
| **代码简洁性** | 减少 30%-50% 样板代码 |
| **可维护性** | 自动生成代码保持一致性 |
| **开发效率** | 减少手动编写重复代码时间 |
| **可读性** | 聚焦业务逻辑而非模板代码 |

## 二、环境配置

### 2.1 项目依赖

**Maven**:

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <version>1.18.30</version>
    <scope>provided</scope>
</dependency>
```

**Gradle**:

```groovy
compileOnly 'org.projectlombok:lombok:1.18.30'
annotationProcessor 'org.projectlombok:lombok:1.18.30'
```

### 2.2 IDE 插件

- **IntelliJ IDEA**: 安装 Lombok Plugin
- **Eclipse**: 下载 lombok.jar 并执行安装

## 三、核心注解详解

### 3.1 基础注解

| 注解 | 作用 | 等效代码示例 |
|------|------|-------------|
| `@Getter` | 生成 getter 方法 | `public String getName() { return name; }` |
| `@Setter` | 生成 setter 方法 | `public void setName(String name) { this.name = name; }` |
| `@ToString` | 生成 toString() | `return "User(name=" + name + ", age=" + age + ")";` |
| `@EqualsAndHashCode` | 生成 equals() 和 hashCode() | 基于字段的相等性比较 |

**示例**：

```java
@Getter
@Setter
@ToString
@EqualsAndHashCode
public class User {
    private String name;
    private int age;
}
```

### 3.2 构造方法注解

```java
@NoArgsConstructor      // 无参构造
@AllArgsConstructor     // 全参构造
@RequiredArgsConstructor // final字段构造
public class Order {
    private final Long id;
    private String product;
    private int quantity;
}
```

### 3.3 数据对象注解

`@Data` = `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor`

**适用场景**：POJO/DTO/VO 对象

```java
@Data
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;
}
```

### 3.4 Builder 模式

```java
@Builder
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class Employee {
    private String id;
    private String name;
    private Department department;
    
    // 使用示例
    public static void main(String[] args) {
        Employee emp = Employee.builder()
            .id("E1001")
            .name("Alice")
            .department(Department.IT)
            .build();
    }
}
```

### 3.5 日志注解

```java
@Slf4j          // SLF4J
@Log4j2         // Log4j2
@CommonsLog     // Apache Commons Logging
public class OrderService {
    public void processOrder() {
        log.info("Processing order...");
        // 业务逻辑
    }
}
```

### 3.6 其他实用注解

- `@SneakyThrows`：静默抛出受检异常
- `@Synchronized`：线程安全同步方法
- `@Cleanup`：自动资源管理（类似 try-with-resources）
- `@With`：创建对象副本并修改字段

## 四、最佳实践

### 4.1 注解使用规范

1. **谨慎使用 `@Data`**
   - 避免在 JPA Entity 中使用（可能影响 equals/hashCode）
   - 替代方案：

     ```java
     @Getter
     @Setter
     @ToString(exclude = "password")
     @EqualsAndHashCode(onlyExplicitlyIncluded = true)
     public class Account {
         @EqualsAndHashCode.Include
         private Long id;
         private String username;
         private String password;
     }
     ```

2. **Builder 的高级用法**

   ```java
   @Builder(builderClassName = "InternalBuilder", builderMethodName = "internalBuilder")
   public class Configuration {
       @Builder.Default 
       private int timeout = 30;
       
       // 自定义构建逻辑
       public static class InternalBuilder {
           public Configuration build() {
               validate();
               return new Configuration(this);
           }
       }
   }
   ```

3. **避免循环引用**

   ```java
   @Data
   public class Department {
       private List<Employee> employees;
   }
   
   @Data
   public class Employee {
       private Department department; // 导致循环引用！
   }
   ```

### 4.2 与常用框架整合

**JPA/Hibernate 实体**：

```java
@Entity
@Getter
@Setter
@NoArgsConstructor
@EqualsAndHashCode(onlyExplicitlyIncluded = true)
public class Customer {
    @Id
    @EqualsAndHashCode.Include
    private Long id;
    
    private String name;
    
    @ToString.Exclude
    @OneToMany(mappedBy = "customer")
    private List<Order> orders;
}
```

**Spring Boot 配置**：

```java
@Configuration
@Slf4j
public class AppConfig {
    
    @Bean
    @SneakyThrows
    public DataSource dataSource() {
        return DataSourceBuilder.create()
                .url("jdbc:mysql://localhost/db")
                .username("admin")
                .password("secret")
                .build();
    }
}
```

### 4.3 调试与兼容性

1. **IDE 问题排查**：
   - 确保启用注解处理
   - 检查 Lombok 插件版本兼容性

2. **Jackson 序列化**：

   ```java
   @Value // 创建不可变对象
   @Jacksonized // 支持 Jackson 反序列化
   @Builder
   public class ApiResponse<T> {
       int status;
       T data;
   }
   ```

### 4.4 团队协作规范

1. **项目统一配置**（`lombok.config`）：

   ```properties
   # 强制字段注解才生成Getter/Setter
   lombok.accessors.fluent = false
   lombok.getter.noIsPrefix = true
   
   # 日志字段名称统一
   lombok.log.fieldName = LOGGER
   ```

2. **代码审查重点**：
   - 检查 `@EqualsAndHashCode` 排除动态字段
   - 验证 `@Builder` 对象的完整性
   - 避免在继承层次中使用 `@Data`

## 五、性能与陷阱

### 5.1 编译时 vs 运行时

- **编译时增强**：Lombok 在编译阶段修改 AST，不影响运行时性能
- **无运行时依赖**：生成的代码是标准 Java 字节码

### 5.2 常见陷阱与规避

| 陷阱 | 解决方案 |
|------|----------|
| **JPA 代理对象问题** | 避免在实体中使用 `@ToString` 或排除关联集合 |
| **继承失效** | 手动重写 equals/hashCode 方法 |
| **Builder 默认值** | 使用 `@Builder.Default` 标注字段 |
| **线程安全** | 不可变对象使用 `@Value` 代替 `@Data` |

## 六、替代方案比较

| 方案 | 优点 | 缺点 |
|------|------|------|
| **Lombok** | 零运行时开销，注解简洁 | IDE 依赖插件，学习曲线 |
| **Kotlin** | 语言级支持，功能强大 | 需切换语言，生态迁移成本 |
| **Record（Java 14+）** | 官方标准，类型安全 | 功能有限（不可变对象专用） |
| **MapStruct** | 专业字段映射 | 仅解决 DTO 转换问题 |

## 七、升级与迁移

### 7.1 向 Java Record 迁移

```java
// Lombok 方式
@Value
public class Point {
    int x;
    int y;
}

// Java 17 Record
public record Point(int x, int y) {}
```

### 7.2 版本升级策略

1. 测试覆盖率需 > 80%
2. 逐步替换废弃注解（如 `@NonNull` -> `@lombok.NonNull`）
3. 验证新版本与 IDE 的兼容性

## 附录：常用注解速查表

| 注解 | 作用域 | 描述 |
|------|--------|------|
| `@Getter` | 类/字段 | 生成 getter 方法 |
| `@Setter` | 类/字段 | 生成 setter 方法 |
| `@ToString` | 类 | 生成 toString() |
| `@EqualsAndHashCode` | 类 | 生成 equals() 和 hashCode() |
| `@NoArgsConstructor` | 类 | 无参构造器 |
| `@AllArgsConstructor` | 类 | 全参构造器 |
| `@RequiredArgsConstructor` | 类 | final/@NonNull 字段构造器 |
| `@Data` | 类 | 组合注解（Getter/Setter/ToString/EqualsAndHashCode/RequiredArgsConstructor） |
| `@Value` | 类 | 不可变对象组合注解 |
| `@Builder` | 类/方法 | 生成 Builder 模式 API |
| `@Slf4j` | 类 | 生成 SLF4J 日志实例 |
| `@SneakyThrows` | 方法 | 静默抛出受检异常 |
| `@Synchronized` | 方法 | 线程安全同步 |
| `@Cleanup` | 局部变量 | 自动资源管理 |
| `@With` | 字段/类 | 创建不可变对象的修改副本 |

> **最佳实践总结**：Lombok 是提升 Java 开发效率的利器，但需团队规范使用，避免过度依赖。推荐在 DTO、配置类、工具类中积极使用，而在实体类、核心业务对象中谨慎选择注解组合。
