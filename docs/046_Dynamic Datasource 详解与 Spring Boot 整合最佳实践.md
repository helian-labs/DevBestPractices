# dynamic-datasource-spring-boot3-starter 详解与 Spring Boot 整合最佳实践

## 1. 组件概述

**dynamic-datasource-spring-boot3-starter** 是一个专为 Spring Boot 3.x 设计的动态数据源管理工具，主要解决以下核心问题：

- **多数据源动态切换**：运行时根据策略动态切换不同数据源
- **读写分离**：自动路由读操作到从库，写操作到主库
- **数据源组管理**：支持主从数据源分组配置
- **多租户支持**：为不同租户动态分配数据源
- **数据源健康检查**：自动监控数据源连接状态

### 核心特性

- 零侵入：通过注解实现数据源切换
- 高性能：基于连接池的高效数据源管理
- 强兼容：支持主流连接池（HikariCP, Druid等）
- 易扩展：支持自定义数据源加载策略
- 完善监控：提供数据源运行状态指标

## 2. 环境要求

| 组件              | 最低要求   | 推荐版本     |
|-------------------|-----------|-------------|
| Java             | 17        | 17+         |
| Spring Boot      | 3.0.0     | 3.1.5       |
| dynamic-datasource | 4.2.0     | 4.3.0+      |
| 数据库驱动        | 根据数据库类型 | 最新稳定版   |

## 3. 快速开始

### 添加 Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>com.baomidou</groupId>
        <artifactId>dynamic-datasource-spring-boot3-starter</artifactId>
        <version>4.3.0</version>
    </dependency>
    
    <!-- 使用 HikariCP 连接池 -->
    <dependency>
        <groupId>com.zaxxer</groupId>
        <artifactId>HikariCP</artifactId>
        <version>5.0.1</version>
    </dependency>
    
    <!-- MySQL 驱动 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 基础配置 (application.yml)

```yaml
spring:
  datasource:
    dynamic:
      primary: master # 设置默认数据源
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/master_db?useSSL=false
          username: root
          password: masterpass
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.zaxxer.hikari.HikariDataSource
        slave1:
          url: jdbc:mysql://localhost:3307/slave1_db?useSSL=false
          username: readuser
          password: readpass
          driver-class-name: com.mysql.cj.jdbc.Driver
          type: com.zaxxer.hikari.HikariDataSource
      hikari: # 全局 HikariCP 配置
        connection-timeout: 30000
        maximum-pool-size: 20
        minimum-idle: 5
```

### 使用 @DS 注解切换数据源

```java
@Service
public class UserService {
    
    // 默认使用主数据源
    public void createUser(User user) {
        // 写操作使用主库
    }
    
    @DS("slave1") // 切换到从库
    public User getUserById(Long id) {
        // 读操作使用从库
    }
    
    @DS("master") // 明确指定主库
    @Transactional
    public void updateUser(User user) {
        // 事务操作必须明确指定数据源
    }
}
```

## 4. 详细配置指南

### 数据源组配置（主从架构）

```yaml
spring:
  datasource:
    dynamic:
      primary: master_group
      strict: true # 严格匹配模式，未找到数据源时抛出异常
      datasource:
        master_group:
          # 主从数据源组配置
          primary: master
          group: true
          datasource:
            master: # 主库
              url: jdbc:mysql://master-host:3306/db
              username: master_user
              password: master_pass
            slave_1: # 从库1
              url: jdbc:mysql://slave1-host:3306/db
              username: slave_user
              password: slave_pass
            slave_2: # 从库2
              url: jdbc:mysql://slave2-host:3306/db
              username: slave_user
              password: slave_pass
```

### 多租户场景配置

```yaml
spring:
  datasource:
    dynamic:
      tenant: true # 开启多租户模式
      tenant-key: tenant_id # 租户ID字段名
      datasource:
        tenant1:
          url: jdbc:mysql://tenant1-db:3306/tenant1
          username: tenant1_user
          password: tenant1_pass
        tenant2:
          url: jdbc:mysql://tenant2-db:3306/tenant2
          username: tenant2_user
          password: tenant2_pass
```

### 高级连接池配置（Druid 示例）

```yaml
spring:
  datasource:
    dynamic:
      druid: # 全局 Druid 配置
        initial-size: 5
        min-idle: 5
        max-active: 20
        test-on-borrow: true
        filters: stat,wall
      datasource:
        master:
          url: jdbc:mysql://localhost:3306/master
          type: com.alibaba.druid.pool.DruidDataSource
          druid: # 数据源独有配置
            connection-properties: druid.stat.mergeSql=true
        slave:
          url: jdbc:mysql://localhost:3307/slave
          type: com.alibaba.druid.pool.DruidDataSource
```

## 5. 核心注解 @DS 详解

### 注解使用位置

```java
// 类级别注解：整个类的方法默认使用 slave 数据源
@DS("slave")
@Service
public class UserReadService {
    
    // 方法级别注解：优先级高于类注解
    @DS("master")
    public User getAdminUser() {
        // ...
    }
    
    // 使用类注解指定的 slave 数据源
    public List<User> listUsers() {
        // ...
    }
}
```

### 事务中的特殊处理

```java
@Service
public class OrderService {
    
    @DS("order_db") // 整个类使用order_db数据源
    @Transactional
    public void createOrder(Order order) {
        // 主业务逻辑
        orderMapper.insert(order);
        
        // 需要操作其他数据源时
        updateInventory(order.getProductId(), order.getQuantity());
    }
    
    // REQUIRES_NEW 事务传播级别
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    @DS("inventory_db") // 切换数据源
    public void updateInventory(Long productId, int quantity) {
        inventoryMapper.reduceStock(productId, quantity);
    }
}
```

### 注解使用注意事项

1. **事务边界**：`@DS` 注解应在 `@Transactional` 之前执行
2. **方法可见性**：仅对 public 方法有效
3. **继承问题**：父类注解不被子类继承
4. **优先级**：方法注解 > 类注解 > 全局配置

## 6. 最佳实践

### 数据源配置建议

```yaml
spring:
  datasource:
    dynamic:
      health: true # 启用健康检查
      seata: true # 开启Seata分布式事务支持
      p6spy: true # 开启SQL日志分析
      metrics: true # 启用监控指标
      
      # 多环境配置示例
      datasource:
        ${spring.profiles.active}:
          primary: master
          datasource:
            master: 
              url: ${DB_MASTER_URL}
            slave1:
              url: ${DB_SLAVE1_URL}
```

### 动态增减数据源

```java
@Autowired
private DynamicRoutingDataSource routingDataSource;

// 添加新数据源
public void addDataSource(String dsName, DataSource dataSource) {
    routingDataSource.addDataSource(dsName, dataSource);
}

// 移除数据源
public void removeDataSource(String dsName) {
    routingDataSource.removeDataSource(dsName);
}
```

### 读写分离策略定制

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DynamicDataSourceStrategy loadBalanceStrategy() {
        // 自定义负载均衡策略（轮询）
        return new RandomDynamicDataSourceStrategy();
        
        // 或使用权重策略
        // return new WeightRandomDynamicDataSourceStrategy();
    }
}
```

### 多租户解决方案

```java
public class TenantDataSourceSelector {

    // 通过拦截器自动设置租户数据源
    @Interceptor
    public class TenantInterceptor implements HandlerInterceptor {
        
        @Override
        public boolean preHandle(HttpServletRequest request, 
                                HttpServletResponse response, 
                                Object handler) {
            String tenantId = request.getHeader("X-Tenant-ID");
            if (StringUtils.isNotBlank(tenantId)) {
                DynamicDataSourceContextHolder.push(tenantId);
            }
            return true;
        }
        
        @Override
        public void afterCompletion(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   Object handler, Exception ex) {
            DynamicDataSourceContextHolder.clear();
        }
    }
}
```

## 7. 高级特性

### 数据源健康检查

```yaml
management:
  endpoint:
    health:
      show-details: always
  health:
    db:
      enabled: true
      query: "SELECT 1" # 自定义健康检查SQL
```

访问健康检查端点：`http://localhost:8080/actuator/health`

### SQL 性能监控（集成 P6Spy）

```yaml
spring:
  datasource:
    dynamic:
      p6spy: true # 开启SQL分析
```

配置 spy.properties:

```properties
module.log=com.p6spy.engine.logging.P6LogFactory
excludecategories=info,debug,result,batch
deregisterdrivers=true
outagedetection=true
outagedetectioninterval=30
```

### 整合 Spring Cloud Alibaba Seata

```yaml
spring:
  datasource:
    dynamic:
      seata: true # 开启分布式事务支持
      seata-mode: AT # 使用AT模式
```

```java
@DS("order_db")
@GlobalTransactional // 分布式事务注解
public void createOrder(Order order) {
    orderMapper.insert(order);
    inventoryService.reduceStock(order.getProductId(), order.getQuantity());
    accountService.deductBalance(order.getUserId(), order.getAmount());
}
```

## 8. 常见问题解决

### Q1: 事务中数据源切换失效

**原因**：事务中数据源在事务开始时确定  
**解决方案**：

```java
// 方案1：将需要切换数据源的方法拆分到不同类中
@DS("slave")
@Service
public class ReportService {
    public Report generateReport() { ... }
}

// 方案2：使用编程式事务
@Autowired
private TransactionTemplate transactionTemplate;

public void transactionalOperation() {
    transactionTemplate.execute(status -> {
        // 操作主数据源
        masterOperation();
        
        // 切换数据源操作
        DynamicDataSourceContextHolder.push("slave");
        try {
            slaveOperation();
        } finally {
            DynamicDataSourceContextHolder.poll();
        }
        return null;
    });
}
```

### Q2: 动态数据源无法注册到 Spring 容器

**解决方案**：手动注册数据源 Bean

```java
@Bean
@ConfigurationProperties(prefix = "spring.datasource.dynamic.datasource.custom")
public DataSource customDataSource() {
    return DataSourceBuilder.create().build();
}

@PostConstruct
public void registerDynamicDataSource() {
    DynamicRoutingDataSource ds = 
        (DynamicRoutingDataSource) dataSource;
    ds.addDataSource("custom", customDataSource());
}
```

### Q3: 多数据源下 MyBatis-Plus 分页插件失效

**解决方案**：为每个数据源单独配置

```java
@Configuration
public class MyBatisConfig {
    
    @Bean
    @DS("master")
    public MybatisPlusInterceptor masterInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
    
    @Bean
    @DS("slave")
    public MybatisPlusInterceptor slaveInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor());
        return interceptor;
    }
}
```

## 9. 性能优化建议

1. **连接池配置优化**：

   ```yaml
   spring:
     datasource:
       dynamic:
         hikari:
           maximum-pool-size: ${DB_MAX_POOL_SIZE:20}
           minimum-idle: ${DB_MIN_IDLE:5}
           idle-timeout: 30000
           max-lifetime: 1800000
   ```

2. **禁用不必要的数据源健康检查**：

   ```yaml
   management:
     health:
       db:
         enabled: false # 生产环境可关闭
   ```

3. **使用数据源组负载均衡**：

   ```java
   @DS("slave_group") // 使用组名而非单个数据源
   public List<User> listUsers() {
       // 请求会自动在 slave1 和 slave2 间负载均衡
   }
   ```

4. **精简动态数据源数量**：避免创建过多未使用数据源

## 10. 完整示例项目结构

```
src/main/java
├── com
│   └── example
│       └── demo
│           ├── config
│           │   ├── DataSourceConfig.java       # 数据源配置
│           │   ├── MyBatisConfig.java          # MyBatis配置
│           │   └── TenantConfig.java           # 多租户配置
│           ├── controller
│           │   ├── UserController.java
│           │   └── OrderController.java
│           ├── service
│           │   ├── UserService.java
│           │   └── OrderService.java
│           ├── mapper
│           │   ├── UserMapper.java
│           │   └── OrderMapper.java
│           └── Application.java                # 主启动类
└── resources
    ├── application.yml                         # 主配置文件
    ├── application-dev.yml                     # 开发环境配置
    ├── application-prod.yml                    # 生产环境配置
    └── mapper                                  # MyBatis XML文件
        ├── UserMapper.xml
        └── OrderMapper.xml
```

> 提示：完整示例代码可在官方 GitHub 仓库获取：  
> [https://github.com/baomidou/dynamic-datasource-spring-boot-starter](https://github.com/baomidou/dynamic-datasource-spring-boot-starter)

## 总结

**dynamic-datasource-spring-boot3-starter** 为 Spring Boot 3 应用提供了强大的多数据源管理能力。通过本文的详解和最佳实践，您可以：

1. 实现多数据源动态切换和读写分离
2. 构建健壮的多租户架构
3. 优化数据源性能配置
4. 解决复杂事务场景下的数据源管理问题
5. 集成分布式事务和监控能力

遵循最佳实践并根据实际业务需求灵活配置，可显著提升应用的数据库访问能力和架构灵活性。
