# Mockito 详解与 Spring Boot 整合最佳实践

## 1. 引言：为什么需要 Mockito？

在软件开发中，**单元测试**是保证代码质量的关键环节。然而，当测试对象依赖外部服务、数据库或其他复杂组件时，直接进行测试会面临诸多挑战：

- **测试速度慢**：真实数据库/网络调用导致测试执行缓慢
- **环境依赖性**：需要配置复杂的外部环境
- **不可控因素**：难以模拟错误场景和边界条件
- **测试隔离性差**：一个组件的失败可能导致整个测试链失败

**Mockito** 应运而生 - 它是一个流行的 Java 模拟框架，允许你创建和配置模拟对象（mocks），从而解决上述问题。通过与 Spring Boot 整合，我们可以构建高效、可靠的测试套件。

## 2. Mockito 核心概念

### 2.1 核心术语

| 术语        | 说明                                                                 |
|-------------|----------------------------------------------------------------------|
| **Mock**    | 模拟对象，完全替代真实依赖，所有方法默认返回空值或默认值              |
| **Spy**     | 部分模拟对象，默认调用真实方法，可选择性地模拟特定方法                |
| **Stubbing**| 定义模拟对象的行为（当调用X方法时返回Y）                             |
| **Verification** | 验证模拟对象的交互行为（如：方法是否被调用、调用次数等）          |

### 2.2 Mockito 核心能力

- 创建模拟对象（mock）和部分模拟对象（spy）
- 定义方法调用的返回值或异常
- 验证方法调用行为
- 捕获调用参数进行分析
- 使用参数匹配器进行灵活验证

## 3. Mockito 基础用法

### 3.1 添加依赖

```xml
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.12.0</version>
    <scope>test</scope>
</dependency>
```

### 3.2 基本 Mock 操作

```java
import static org.mockito.Mockito.*;

// 创建模拟对象
List<String> mockedList = mock(List.class);

// Stubbing - 定义模拟行为
when(mockedList.get(0)).thenReturn("first");
when(mockedList.get(1)).thenThrow(new RuntimeException());

// 使用模拟对象
System.out.println(mockedList.get(0)); // 输出 "first"
System.out.println(mockedList.get(1)); // 抛出 RuntimeException

// 验证交互
verify(mockedList).get(0);     // 验证get(0)被调用一次
verify(mockedList, times(1)).get(0); // 同上
verify(mockedList, never()).get(2);  // 验证get(2)从未被调用
```

### 3.3 参数匹配器

```java
// 使用内置匹配器
when(mockedList.get(anyInt())).thenReturn("element");

// 自定义参数匹配器
when(mockedList.add(argThat(s -> s.length() > 5))).thenReturn(true);

// 混合使用匹配器和具体值
when(mockedList.set(anyInt(), eq("newValue"))).thenReturn("oldValue");
```

## 4. Spring Boot 整合 Mockito

### 4.1 添加测试依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 4.2 测试类结构

```java
@ExtendWith(MockitoExtension.class)
@SpringBootTest
class UserServiceTest {
    
    @Mock
    private UserRepository userRepository;
    
    @Spy
    private EmailService emailService;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void testGetUserById() {
        // 准备测试数据
        User mockUser = new User(1L, "john@example.com");
        
        // Stubbing 模拟行为
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));
        
        // 执行测试方法
        User result = userService.getUserById(1L);
        
        // 断言验证
        assertNotNull(result);
        assertEquals("john@example.com", result.getEmail());
        
        // 验证交互
        verify(userRepository).findById(1L);
        verify(emailService, never()).sendWelcomeEmail(any());
    }
}
```

### 4.3 测试分层策略

| 测试类型       | 特点                                                                 | Mockito 使用场景                          |
|----------------|----------------------------------------------------------------------|-------------------------------------------|
| **单元测试**   | 测试单个类，隔离所有依赖                                             | 使用 `@Mock` 模拟所有外部依赖             |
| **集成测试**   | 测试多个组件的交互，使用真实数据库和部分服务                         | 混合使用真实对象和模拟对象 (`@Spy`)       |
| **切片测试**   | 测试特定层（如Web层、数据层）                                        | 使用 `@WebMvcTest` 或 `@DataJpaTest` 注解 |

## 5. 最佳实践与高级技巧

### 5.1 测试服务层（Service Layer）

```java
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    
    @Mock
    private InventoryService inventoryService;
    
    @Mock
    private PaymentService paymentService;
    
    @InjectMocks
    private OrderService orderService;
    
    @Test
    void placeOrder_WhenInventorySufficient_ShouldProcessPayment() {
        // Arrange
        Order order = new Order("order-123", 5);
        when(inventoryService.checkStock("item-001", 5)).thenReturn(true);
        when(paymentService.processPayment(anyDouble())).thenReturn("PAY-123");
        
        // Act
        OrderResult result = orderService.placeOrder(order);
        
        // Assert
        assertEquals(OrderStatus.SUCCESS, result.getStatus());
        assertNotNull(result.getPaymentId());
        
        // Verify interactions
        verify(inventoryService).checkStock("item-001", 5);
        verify(paymentService).processPayment(anyDouble());
        verify(inventoryService).deductStock("item-001", 5);
    }
    
    @Test
    void placeOrder_WhenInventoryInsufficient_ShouldFail() {
        // Arrange
        Order order = new Order("order-124", 10);
        when(inventoryService.checkStock("item-001", 10)).thenReturn(false);
        
        // Act
        OrderResult result = orderService.placeOrder(order);
        
        // Assert
        assertEquals(OrderStatus.FAILED, result.getStatus());
        assertEquals("Insufficient stock", result.getErrorMessage());
        
        // Verify payment was NOT processed
        verify(paymentService, never()).processPayment(anyDouble());
    }
}
```

### 5.2 测试控制器层（Controller Layer）

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void getUserById_ShouldReturnUser() throws Exception {
        // Arrange
        User mockUser = new User(1L, "john@example.com");
        when(userService.getUserById(1L)).thenReturn(mockUser);
        
        // Act & Assert
        mockMvc.perform(get("/users/1"))
               .andExpect(status().isOk())
               .andExpect(jsonPath("$.id").value(1))
               .andExpect(jsonPath("$.email").value("john@example.com"));
        
        verify(userService).getUserById(1L);
    }
}
```

### 5.3 使用 ArgumentCaptor 捕获参数

```java
@Test
void registerUser_ShouldSendWelcomeEmail() {
    // Arrange
    User newUser = new User(null, "new@example.com", "password");
    User savedUser = new User(1L, "new@example.com", "encodedPassword");
    
    when(userRepository.save(any(User.class))).thenReturn(savedUser);
    
    // 创建参数捕获器
    ArgumentCaptor<User> userCaptor = ArgumentCaptor.forClass(User.class);
    
    // Act
    userService.registerUser(newUser);
    
    // Assert
    // 捕获传递给emailService.sendWelcomeEmail()方法的参数
    verify(emailService).sendWelcomeEmail(userCaptor.capture());
    User capturedUser = userCaptor.getValue();
    assertEquals(1L, capturedUser.getId());
    assertEquals("new@example.com", capturedUser.getEmail());
}
```

## 6. 常见陷阱与解决方案

### 6.1 过度模拟问题

**问题**：模拟了过多的对象，导致测试与实现细节过度耦合  
**解决方案**：

- 仅模拟真正的外部依赖（数据库、API客户端等）
- 对值对象和领域模型使用真实实例
- 遵循"不要模拟你不拥有的类型"原则

### 6.2 脆弱的测试

**问题**：测试因内部实现变化而失败，即使功能正常  
**解决方案**：

- 验证行为而非实现细节
- 避免过度使用 `verify()` 检查每个方法调用
- 使用 `ArgumentCaptor` 代替具体值验证

### 6.3 静态方法模拟

**问题**：Mockito 默认无法模拟静态方法  
**解决方案**：

```java
// 添加依赖
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>5.2.0</version>
    <scope>test</scope>
</dependency>

// 测试中使用
try (MockedStatic<UtilityClass> utilities = mockStatic(UtilityClass.class)) {
    utilities.when(UtilityClass::staticMethod).thenReturn("mockValue");
    // 测试代码
}
```

## 7. 性能优化策略

### 7.1 测试分层金字塔

```mermaid
pie
    title 测试类型分布
    “单元测试” ： 70
    “集成测试” ： 20
    “端到端测试” ： 10
```

### 7.2 优化建议

1. **优先编写单元测试**：快速反馈，覆盖核心逻辑
2. **合理使用内存数据库**：在集成测试中使用 H2 替代真实数据库
3. **并行执行测试**：配置 Maven Surefire 并行运行测试
4. **避免重复初始化**：使用 `@BeforeAll` 替代 `@BeforeEach` 初始化共享资源

```xml
<!-- 在pom.xml中配置并行测试 -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.2.0</version>
    <configuration>
        <parallel>classes</parallel>
        <threadCount>4</threadCount>
    </configuration>
</plugin>
```

## 8. 结论

Mockito 与 Spring Boot 的结合为 Java 应用提供了强大的测试能力：

- **快速反馈**：通过模拟依赖实现毫秒级测试执行
- **可靠隔离**：精确控制测试环境，排除外部干扰
- **全面覆盖**：轻松模拟各种成功/失败场景
- **持续集成友好**：无外部依赖的测试适合 CI/CD 流水线

**最后建议**：

1. 遵循 Given-When-Then 结构组织测试
2. 保持测试代码与生产代码同等质量标准
3. 定期审查测试覆盖率（推荐目标：核心逻辑 ≥80%）
4. 将测试作为设计工具，驱动出更好的架构

> "好的测试不是证明代码能工作，而是揭示代码如何工作。" - 测试驱动开发箴言

通过本文介绍的最佳实践，您可以构建健壮、可维护的测试套件，为 Spring Boot 应用的质量保驾护航。
