# Spring 事务管理（@Transactional）详解与最佳实践

## 1. 引言

Spring 事务管理是构建企业级应用的核心技术之一，它确保了数据的一致性和完整性。`@Transactional` 注解是 Spring 声明式事务管理的核心，允许开发者以声明方式定义事务边界，而无需编写繁琐的事务管理代码。

### Spring 事务管理的优势

- **声明式事务**：通过注解配置，非侵入式实现
- **统一抽象**：支持多种事务管理器（JDBC、JPA、JTA等）
- **简化开发**：减少样板代码，提高开发效率
- **灵活配置**：支持传播行为、隔离级别等精细控制

## 2. 核心概念

### 2.1 ACID 原则

| 特性 | 描述 |
|------|------|
| **原子性 (Atomicity)** | 事务中的所有操作要么全部成功，要么全部失败 |
| **一致性 (Consistency)** | 事务将数据库从一种一致状态转变为另一种一致状态 |
| **隔离性 (Isolation)** | 并发事务之间互不干扰 |
| **持久性 (Durability)** | 事务提交后，修改将永久保存 |

### 2.2 Spring 事务抽象

```java
public interface PlatformTransactionManager {
    TransactionStatus getTransaction(TransactionDefinition definition);
    void commit(TransactionStatus status);
    void rollback(TransactionStatus status);
}
```

### 2.3 事务管理方式

- **编程式事务**：通过 `TransactionTemplate` 或 `PlatformTransactionManager` 手动管理
- **声明式事务**：通过 `@Transactional` 注解自动管理（推荐）

## 3. @Transactional 注解详解

### 3.1 基本用法

```java
@Service
public class OrderService {
    
    @Autowired
    private OrderRepository orderRepository;
    
    @Autowired
    private PaymentRepository paymentRepository;
    
    @Transactional
    public void placeOrder(Order order, Payment payment) {
        orderRepository.save(order);
        paymentRepository.processPayment(payment);
    }
}
```

### 3.2 作用位置

| 位置 | 效果 | 建议 |
|------|------|------|
| **类级别** | 应用于所有 public 方法 | 适用于类中大多数方法需要事务的场景 |
| **方法级别** | 仅应用于特定方法 | 推荐使用，更精确控制 |
| **接口级别** | 应用于接口方法 | 不推荐，可能导致代理问题 |

### 3.3 关键属性配置

#### 传播行为 (Propagation)

```java
@Transactional(propagation = Propagation.REQUIRED)
public void methodA() {
    // ...
}
```

| 传播行为 | 描述 |
|----------|------|
| **REQUIRED** (默认) | 如果当前存在事务，则加入该事务；否则创建一个新事务 |
| **REQUIRES_NEW** | 总是创建一个新事务，暂停当前事务（如果有） |
| **SUPPORTS** | 如果当前存在事务，则加入该事务；否则以非事务方式执行 |
| **NOT_SUPPORTED** | 以非事务方式执行，暂停当前事务（如果有） |
| **MANDATORY** | 必须在事务中执行，否则抛出异常 |
| **NEVER** | 必须在非事务中执行，否则抛出异常 |
| **NESTED** | 如果当前存在事务，则在嵌套事务中执行 |

#### 隔离级别 (Isolation)

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void updateData() {
    // ...
}
```

| 隔离级别 | 描述 | 可能的问题 |
|----------|------|------------|
| **DEFAULT** | 使用数据库默认级别 | - |
| **READ_UNCOMMITTED** | 允许读取未提交的变更 | 脏读、不可重复读、幻读 |
| **READ_COMMITTED** | 只允许读取已提交的变更 | 不可重复读、幻读 |
| **REPEATABLE_READ** | 确保同一事务多次读取结果一致 | 幻读 |
| **SERIALIZABLE** | 最高隔离级别，完全串行化 | 性能低 |

#### 其他重要属性

```java
@Transactional(
    timeout = 30, // 事务超时时间（秒）
    readOnly = true, // 只读事务（优化性能）
    rollbackFor = {BusinessException.class}, // 指定回滚的异常类型
    noRollbackFor = {ValidationException.class} // 指定不回滚的异常类型
)
public void readData() {
    // ...
}
```

## 4. 事务传播行为详解与示例

### 4.1 REQUIRED 与 REQUIRES_NEW 对比

```java
@Service
public class AccountService {

    @Transactional(propagation = Propagation.REQUIRED)
    public void outerMethod() {
        innerMethod();
        // 如果 innerMethod 抛出异常，整个事务回滚
    }
    
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void innerMethod() {
        // 此方法在独立事务中执行
        // 即使 outerMethod 回滚，此方法的操作也可能提交
    }
}
```

### 4.2 NESTED 传播行为

```java
@Transactional
public void parentMethod() {
    // 主事务操作
    try {
        childMethod(); // 嵌套事务
    } catch (Exception e) {
        // 处理异常，父事务可继续
    }
    // 更多操作
}

@Transactional(propagation = Propagation.NESTED)
public void childMethod() {
    // 嵌套事务：独立保存点
    // 如果失败，只回滚到此保存点，不影响父事务
}
```

## 5. 事务隔离级别实践

### 5.1 隔离级别选择建议

| 场景 | 推荐隔离级别 |
|------|--------------|
| 读多写少，数据一致性要求不高 | READ_COMMITTED |
| 财务系统，高一致性要求 | REPEATABLE_READ |
| 报表系统，只读查询 | READ_COMMITTED + 只读事务 |
| 复杂事务处理，防止幻读 | SERIALIZABLE（谨慎使用） |

### 5.2 隔离级别示例

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void transferFunds(Account from, Account to, BigDecimal amount) {
    // 1. 检查账户余额（可重复读确保多次读取一致）
    BigDecimal balance = accountRepository.getBalance(from.getId());
    if (balance.compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }
    
    // 2. 扣款
    accountRepository.debit(from.getId(), amount);
    
    // 3. 存款
    accountRepository.credit(to.getId(), amount);
    
    // 在 REPEATABLE_READ 级别下，步骤1和2之间其他事务的修改不可见
}
```

## 6. 最佳实践

### 6.1 正确使用姿势

1. **服务层使用**：在 Service 层添加 `@Transactional`，而不是 DAO 层
2. **异常处理**：

   ```java
   @Transactional
   public void processOrder() {
       try {
           // 业务逻辑
       } catch (DataIntegrityViolationException e) {
           // 转换为业务异常触发回滚
           throw new BusinessException("Data error", e);
       }
   }
   ```

3. **只读事务优化**：

   ```java
   @Transactional(readOnly = true)
   public List<Order> findOrdersByUser(Long userId) {
       return orderRepository.findByUserId(userId);
   }
   ```

### 6.2 常见陷阱及解决方案

#### 自调用问题

```java
@Service
public class OrderService {
    
    public void placeOrder(Order order) {
        validateOrder(order);
        // 自调用不会触发事务
        processPayment(order); 
    }
    
    @Transactional
    public void processPayment(Order order) {
        // 事务不会生效
    }
    
    // 解决方案1：注入自身代理
    @Autowired
    private OrderService selfProxy;
    
    // 解决方案2：重构代码，将方法拆分到不同类
}
```

#### 异常回滚问题

```java
@Transactional
public void updateData() {
    try {
        // 可能抛出SQLException
        jdbcTemplate.update("...");
    } catch (DataAccessException e) {
        // 默认只回滚RuntimeException和Error
        // 解决方案：添加rollbackFor
        throw new CustomException(e);
    }
}

// 正确配置
@Transactional(rollbackFor = {SQLException.class, CustomException.class})
```

### 6.3 性能优化建议

1. **设置合理超时**：防止长时间事务占用资源
2. **只读事务优先**：为查询方法设置 `readOnly=true`
3. **避免大事务**：拆分长事务为多个小事务
4. **合理选择隔离级别**：在一致性和性能间取得平衡

## 7. 高级主题

### 7.1 多数据源事务管理

使用 JTA 实现分布式事务：

```java
@Configuration
@EnableTransactionManagement
@EnableJta
public class TransactionConfig {
    // 配置多个数据源和JTA事务管理器
}

@Service
public class DistributedService {
    
    @Transactional
    public void distributedOperation() {
        // 操作多个数据源
        dataSource1Repository.save(...);
        dataSource2Repository.update(...);
        // 全部成功或全部回滚
    }
}
```

### 7.2 事务事件监听

```java
@Component
public class TransactionListener {
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handleAfterCommit(OrderCreatedEvent event) {
        // 事务提交后发送通知
        notificationService.send(event.getOrderId());
    }
    
    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleAfterRollback(OrderFailedEvent event) {
        // 事务回滚后记录日志
        logger.error("Order failed: " + event.getOrderId());
    }
}
```

## 8. 测试事务

### 8.1 Spring Boot 测试示例

```java
@SpringBootTest
@Transactional // 测试结束后自动回滚
public class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @Test
    public void testPlaceOrderSuccess() {
        Order order = createTestOrder();
        orderService.placeOrder(order);
        assertNotNull(order.getId());
    }

    @Test
    @Rollback(false) // 禁用自动回滚
    public void testCommitBehavior() {
        // 测试实际提交行为
    }

    @Test
    public void testTransactionRollback() {
        Order invalidOrder = createInvalidOrder();
        assertThrows(BusinessException.class, () -> {
            orderService.placeOrder(invalidOrder);
        });
        
        // 验证数据未持久化
        assertNull(orderRepository.findById(invalidOrder.getId()));
    }
}
```

## 9. 总结

### 关键要点回顾

1. **正确使用位置**：在服务层方法上使用 `@Transactional`
2. **传播行为选择**：理解 REQUIRED、REQUIRES_NEW 和 NESTED 的区别
3. **隔离级别权衡**：根据业务需求选择合适级别
4. **异常处理**：明确配置 `rollbackFor` 和 `noRollbackFor`
5. **性能优化**：使用只读事务、设置超时时间

### 推荐实践

- **保持事务精简**：事务中只包含必要的数据库操作
- **避免远程调用**：事务中不进行 HTTP 调用或消息队列操作
- **事务边界清晰**：一个事务对应一个业务用例
- **结合重试机制**：对于乐观锁冲突使用重试策略

> **提示**：Spring 事务管理虽然强大，但并非万能。对于复杂分布式事务场景，建议考虑 Saga 模式或使用 Seata 等分布式事务解决方案。

## 附录：常见问题解答

**Q：为什么我的 @Transactional 注解不生效？**
A：常见原因：

1. 方法不是 public
2. 自调用问题（类内部方法调用）
3. 异常类型不是 RuntimeException
4. 数据库引擎不支持事务（如 MyISAM）

**Q：如何选择编程式事务和声明式事务？**
A：

- 声明式事务：适用于大多数场景（推荐）
- 编程式事务：需要精细控制事务边界时使用

**Q：事务超时如何计算？**
A：超时时间从事务开始时计算，包括所有嵌套操作。超过指定时间未完成将自动回滚。

**Q：@Transactional 和 synchronized 能一起用吗？**
A：不能保证线程安全！事务在数据库层控制，synchronized 在 JVM 层控制。对于高并发场景，使用乐观锁或悲观锁机制。
