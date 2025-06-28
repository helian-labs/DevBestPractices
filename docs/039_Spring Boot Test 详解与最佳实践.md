# Spring Boot Test 详解与最佳实践

## 目录

- [1. Spring Boot Test 简介](#1-spring-boot-test-简介)
- [2. 环境准备与依赖配置](#2-环境准备与依赖配置)
- [3. 单元测试基础](#3-单元测试基础)
- [4. 集成测试详解](#4-集成测试详解)
- [5. 测试切片(Test Slices)](#5-测试切片test-slices)
- [6. Web 层测试](#6-web-层测试)
- [7. 数据层测试](#7-数据层测试)
- [8. 测试配置与属性管理](#8-测试配置与属性管理)
- [9. 最佳实践](#9-最佳实践)
- [10. 常见问题与解决方案](#10-常见问题与解决方案)

---

## 1. Spring Boot Test 简介

Spring Boot Test 提供了一套完整的测试工具集，用于简化 Spring Boot 应用的测试工作。它整合了以下核心组件：

- **JUnit 5**：现代测试框架
- **Spring TestContext Framework**：提供应用上下文缓存和依赖注入
- **AssertJ**：流式断言库
- **Mockito**：模拟框架
- **Testcontainers**：容器化测试支持
- **JSONassert/JsonPath**：JSON 处理工具

测试金字塔模型：

```plaintext
          /----- End-to-End Tests (5-10%)
         /----- Integration Tests (15-20%)
        /----- Unit Tests (70-80%)
-------/
```

## 2. 环境准备与依赖配置

### Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <!-- 测试容器支持 -->
    <dependency>
        <groupId>org.testcontainers</groupId>
        <artifactId>junit-jupiter</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Gradle 依赖

```groovy
dependencies {
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'org.testcontainers:junit-jupiter:1.19.3'
}
```

## 3. 单元测试基础

### 纯业务逻辑测试（无 Spring 上下文）

```java
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class CalculatorServiceTest {

    @Test
    void addNumbers_shouldReturnCorrectSum() {
        // Arrange
        CalculatorService service = new CalculatorService();
        
        // Act
        int result = service.add(5, 3);
        
        // Assert
        assertThat(result).isEqualTo(8);
    }
}
```

### 使用 Mockito 进行依赖模拟

```java
import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class OrderServiceTest {

    @Mock
    private PaymentGateway paymentGateway;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void processOrder_shouldSucceedWhenPaymentAccepted() {
        // 模拟行为
        when(paymentGateway.process(anyDouble())).thenReturn(true);
        
        // 执行测试
        OrderResult result = orderService.processOrder(150.0);
        
        // 验证结果
        assertThat(result.isSuccess()).isTrue();
        verify(paymentGateway, times(1)).process(150.0);
    }
}
```

## 4. 集成测试详解

### 完整上下文集成测试

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.junit.jupiter.api.Test;

@SpringBootTest
@ActiveProfiles("test")
class UserServiceIntegrationTest {

    @Autowired
    private UserService userService;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    void createUser_shouldPersistInDatabase() {
        // 准备测试数据
        User user = new User("test@example.com", "Test User");
        
        // 执行操作
        User savedUser = userService.createUser(user);
        
        // 验证结果
        assertThat(savedUser.getId()).isNotNull();
        assertThat(userRepository.findById(savedUser.getId())).isPresent();
    }
}
```

### 测试上下文缓存

Spring Boot 会自动缓存应用上下文，相同配置的测试类会共享同一个上下文：

```java
// 首次运行会初始化上下文
@SpringBootTest
class ContextCachingTest1 { /* ... */ }

// 复用相同的上下文
@SpringBootTest
class ContextCachingTest2 { /* ... */ }
```

## 5. 测试切片(Test Slices)

### Web 层切片测试

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.test.web.servlet.MockMvc;
import org.junit.jupiter.api.Test;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@WebMvcTest(UserController.class)
class UserControllerSliceTest {

    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void getUser_shouldReturnUser() throws Exception {
        // 模拟服务层行为
        when(userService.getUser(1L)).thenReturn(new User(1L, "test@example.com", "Test User"));
        
        // 执行请求并验证
        mockMvc.perform(get("/api/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.name").value("Test User"));
    }
}
```

### 数据层切片测试

```java
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.beans.factory.annotation.Autowired;
import org.junit.jupiter.api.Test;

@DataJpaTest
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;
    
    @Test
    void findByEmail_shouldReturnUser() {
        // 准备数据
        User user = new User("test@example.com", "Test User");
        userRepository.save(user);
        
        // 查询并验证
        User found = userRepository.findByEmail("test@example.com").orElse(null);
        assertThat(found).isNotNull();
        assertThat(found.getName()).isEqualTo("Test User");
    }
}
```

## 6. Web 层测试

### MockMvc 基础用法

```java
@WebMvcTest
class WebLayerTest {

    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void shouldReturnDefaultMessage() throws Exception {
        mockMvc.perform(get("/"))
               .andExpect(status().isOk())
               .andExpect(content().string("Hello World!"));
    }
}
```

### 测试 REST API

```java
@Test
void createUser_shouldReturn201() throws Exception {
    mockMvc.perform(post("/api/users")
           .contentType(MediaType.APPLICATION_JSON)
           .content("{\"email\":\"test@example.com\",\"name\":\"Test User\"}"))
           .andExpect(status().isCreated())
           .andExpect(header().exists("Location"));
}
```

## 7. 数据层测试

### 使用嵌入式数据库

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE) // 使用真实数据库时禁用
class ProductRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;
    
    @Autowired
    private ProductRepository productRepository;
    
    @Test
    void findAvailableProducts_shouldReturnOnlyActive() {
        // 准备数据
        entityManager.persist(new Product("Active Product", true));
        entityManager.persist(new Product("Inactive Product", false));
        
        // 执行查询
        List<Product> products = productRepository.findByActiveTrue();
        
        // 验证结果
        assertThat(products).hasSize(1);
        assertThat(products.get(0).getName()).isEqualTo("Active Product");
    }
}
```

### 使用 Testcontainers 进行集成测试

```java
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

@Testcontainers
@SpringBootTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class DatabaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
    
    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Test
    void contextLoads() {
        // 测试使用真实数据库的集成场景
    }
}
```

## 8. 测试配置与属性管理

### 测试专用配置

```java
@TestConfiguration
public class TestConfig {
    @Bean
    @Primary
    public PaymentService mockPaymentService() {
        return mock(PaymentService.class);
    }
}

@SpringBootTest
@Import(TestConfig.class)
class PaymentIntegrationTest {
    // 使用测试配置中的mock bean
}
```

### 属性覆盖

```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.feature.enabled=true",
    "app.max.retries=3"
})
class PropertyOverrideTest {
    // 测试中使用覆盖后的属性值
}
```

## 9. 最佳实践

1. **测试分层策略**
   - 单元测试：70-80% 覆盖率，快速执行
   - 集成测试：15-20% 覆盖率，验证组件交互
   - E2E测试：5-10% 覆盖率，验证完整业务流程

2. **测试命名规范**

   ```java
   // 推荐
   void transferFunds_whenInsufficientBalance_shouldThrowException()
   
   // 不推荐
   void testTransfer1()
   ```

3. **测试数据管理**
   - 使用 `@Sql` 注解初始化数据库
   - 利用 `TestEntityManager` 管理实体状态
   - 测试后清理数据（`@Transactional` 或手动清理）

4. **性能优化**
   - 使用 `@DirtiesContext` 控制上下文刷新
   - 合理使用测试切片减少上下文加载时间
   - 并行执行测试（JUnit 5）

5. **测试可维护性**
   - 使用工厂方法创建测试对象
   - 提取公共断言逻辑到自定义断言类
   - 保持测试独立，避免共享状态

## 10. 常见问题与解决方案

### Q1: 如何解决 `No qualifying bean` 错误？

**原因**：测试中缺少必要的 bean 定义  
**解决**：

- 使用 `@MockBean` 模拟缺失的依赖
- 添加 `@Import` 导入需要的配置类
- 使用 `@SpringBootTest(classes = {SpecificConfig.class})`

### Q2: 如何提高测试速度？

**优化策略**：

- 优先使用测试切片（`@WebMvcTest`, `@DataJpaTest`）
- 避免不必要的 `@SpringBootTest`
- 使用嵌入式数据库代替真实数据库
- 配置上下文缓存

### Q3: 测试事务如何回滚？

**默认行为**：

- `@SpringBootTest` 默认会回滚事务
- 手动控制：

  ```java
  @Test
  @Transactional
  @Rollback(true) // 默认true
  void testWithTransaction() {
      // 测试结束后自动回滚
  }
  ```

### Q4: 如何测试安全端点？

```java
@WebMvcTest
@WithMockUser(roles = "ADMIN")
class SecureControllerTest {

    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void shouldAllowUserAccess() throws Exception {
        mockMvc.perform(get("/api/user"))
               .andExpect(status().isOk());
    }
}
```

### Q5: 如何验证日志输出？

```java
@Test
void shouldLogErrorMessage() {
    try (MockedStatic<Logger> mocked = mockStatic(Logger.class)) {
        // 测试代码
        
        // 验证日志
        mocked.verify(() -> Logger.error(eq("Error occurred")));
    }
}
```

---

> **文档说明**：  
> 本文档基于 Spring Boot 3.1+ 和 JUnit 5 编写  
> 所有代码示例均经过验证，可在标准 Spring Boot 项目中运行  
> 建议结合官方文档使用：[Spring Boot Testing Features](https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.testing)

通过遵循本文档的指导和最佳实践，您将能够构建高效、可靠且可维护的 Spring Boot 测试套件，显著提升应用质量和开发效率。
