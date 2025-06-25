# Spring Boot Actuator 详解与最佳实践

## 1. Actuator 简介

### 1.1 什么是 Spring Boot Actuator?

Spring Boot Actuator 是 Spring Boot 的**生产级监控和管理模块**，提供了一系列开箱即用的端点（endpoints），用于**实时监控**应用状态、**收集运行时指标**、**管理应用配置**以及**诊断问题**。它使得应用程序的内部状态对外部系统透明，是实现系统可观察性的核心工具。

### 1.2 核心价值

| 功能 | 描述 | 业务价值 |
|------|------|----------|
| **应用监控** | 实时查看应用健康状态 | 快速发现系统异常 |
| **指标收集** | 收集性能指标和业务指标 | 性能优化和容量规划 |
| **配置管理** | 动态查看和修改配置 | 快速调整应用行为 |
| **诊断分析** | 获取线程转储、堆信息 | 快速定位问题根源 |
| **操作控制** | 优雅关闭、日志级别调整 | 减少运维停机时间 |

## 2. 快速集成

### 2.1 添加依赖

```xml
<dependencies>
    <!-- 基础Actuator依赖 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
    
    <!-- Web端点支持 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    
    <!-- 指标导出到Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>
</dependencies>
```

### 2.2 基础配置 (application.yml)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,beans # 暴露的端点
      base-path: /manage # 自定义基础路径
  endpoint:
    health:
      show-details: always # 显示健康详情
      probes:
        enabled: true # 启用K8s就绪/存活探针
    info:
      env:
        enabled: true # 显示环境信息
  server:
    port: 9001 # 独立管理端口
```

## 3. 核心端点详解

### 3.1 标准端点清单

| 端点 | 路径 | 描述 | 敏感 |
|------|------|------|------|
| **health** | `/manage/health` | 应用健康状态 | 否 |
| **info** | `/manage/info` | 应用信息 | 否 |
| **metrics** | `/manage/metrics` | 应用指标 | 否 |
| **prometheus** | `/manage/prometheus` | Prometheus格式指标 | 否 |
| **env** | `/manage/env` | 环境变量和配置 | 是 |
| **beans** | `/manage/beans` | Spring容器Bean列表 | 是 |
| **mappings** | `/manage/mappings` | URL映射关系 | 是 |
| **loggers** | `/manage/loggers` | 日志配置 | 是 |
| **threaddump** | `/manage/threaddump` | 线程转储 | 是 |
| **heapdump** | `/manage/heapdump` | 堆转储文件 | 是 |
| **shutdown** | `/manage/shutdown` | 优雅关闭应用 | 是 |

### 3.2 健康端点详解

**默认健康检查**：

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "MySQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 500107862016,
        "free": 350107862016,
        "threshold": 10485760
      }
    },
    "ping": {
      "status": "UP"
    }
  }
}
```

**自定义健康指示器**：

```java
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    private final ExternalService service;
    
    public CustomHealthIndicator(ExternalService service) {
        this.service = service;
    }

    @Override
    public Health health() {
        if (service.isAvailable()) {
            return Health.up()
                    .withDetail("version", service.getVersion())
                    .build();
        }
        return Health.down()
                .withDetail("reason", "Service unavailable")
                .build();
    }
}
```

### 3.3 指标端点实战

**自定义业务指标**：

```java
@Service
public class OrderService {
    
    // 计数器：记录订单创建数量
    private final Counter orderCounter;
    
    // 计时器：记录订单处理时间
    private final Timer orderProcessingTimer;

    public OrderService(MeterRegistry registry) {
        orderCounter = Counter.builder("orders.created")
                .description("Total created orders")
                .tag("type", "ecommerce")
                .register(registry);
        
        orderProcessingTimer = Timer.builder("orders.process.time")
                .description("Order processing time")
                .publishPercentiles(0.5, 0.95, 0.99) // 百分位数
                .register(registry);
    }
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // 计时器使用
        return orderProcessingTimer.record(() -> {
            Order order = // 订单创建逻辑
            orderCounter.increment(); // 计数器增加
            return order;
        });
    }
}
```

## 4. 安全配置

### 4.1 基础安全控制

```yaml
spring:
  security:
    user:
      name: actuator
      password: securepass
      roles: ACTUATOR_ADMIN

management:
  endpoints:
    web:
      exposure:
        include: '*'
  endpoint:
    health:
      roles: PUBLIC
    info:
      roles: PUBLIC
    shutdown:
      enabled: true
```

### 4.2 安全配置类

```java
@Configuration
@EnableWebSecurity
public class ActuatorSecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/manage/health", "/manage/info").permitAll()
                .antMatchers("/manage/**").hasRole("ACTUATOR_ADMIN")
                .anyRequest().authenticated()
            .and()
            .httpBasic()
            .and()
            .csrf().ignoringAntMatchers("/manage/shutdown"); // 关闭端点禁用CSRF
    }
}
```

### 4.3 敏感信息脱敏

```java
@Configuration
public class SensitiveDataConfig {

    @Bean
    public SanitizingFunction envSanitizer() {
        return data -> {
            // 敏感key脱敏
            if (data.getKey().contains("password") || 
                data.getKey().contains("secret")) {
                return data.withValue("******");
            }
            return data;
        };
    }
}
```

## 5. 自定义端点开发

### 5.1 自定义信息端点

```java
@Component
public class AppInfoContributor implements InfoContributor {

    @Value("${app.version}")
    private String appVersion;
    
    @Override
    public void contribute(Info.Builder builder) {
        Map<String, Object> details = new HashMap<>();
        details.put("version", appVersion);
        details.put("author", "Tech Team");
        details.put("deployTime", Instant.now().toString());
        
        builder.withDetail("app", details)
               .withDetail("system", System.getProperties());
    }
}
```

### 5.2 自定义管理端点

```java
@Endpoint(id = "features")
@Component
public class FeatureToggleEndpoint {
    
    private final Map<String, Boolean> features = new ConcurrentHashMap<>();

    @ReadOperation
    public Map<String, Boolean> features() {
        return features;
    }

    @WriteOperation
    public void configureFeature(@Selector String feature, boolean enabled) {
        features.put(feature, enabled);
    }
}
```

### 5.3 自定义健康端点

```java
@Endpoint(id = "readiness")
@Component
public class ReadinessProbeEndpoint {
    
    private final HealthIndicator healthIndicator;
    
    public ReadinessProbeEndpoint(HealthEndpoint healthEndpoint) {
        this.healthIndicator = healthEndpoint::health;
    }
    
    @ReadOperation
    public HealthComponent readiness() {
        return healthIndicator.health();
    }
}
```

## 6. 生产环境最佳实践

### 6.1 安全加固配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus # 仅暴露必要端点
      base-path: /internal # 隐藏路径
  endpoint:
    health:
      show-components: never # 生产环境隐藏详情
    shutdown:
      enabled: false # 生产环境禁用关闭
  server:
    port: 9001 # 独立管理端口
    address: 127.0.0.1 # 仅监听本地
    ssl:
      enabled: true # 启用HTTPS
```

### 6.2 监控系统集成

**Prometheus + Grafana 配置**：

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-apps'
    metrics_path: '/manage/prometheus'
    static_configs:
      - targets: ['app-host:9001']
    scheme: https
    tls_config:
      insecure_skip_verify: true
    basic_auth:
      username: 'prometheus'
      password: 'securepass'
```

**Grafana 仪表板示例**：

```
+-----------------------------------+
| Spring Boot 应用监控               |
+-----------------+-----------------+
| CPU使用率        | 内存使用        |
| [图表]          | [图表]          |
+-----------------+-----------------+
| HTTP请求量       | 数据库连接池    |
| [图表]          | [图表]          |
+-----------------+-----------------+
| JVM GC暂停       | 线程状态        |
| [图表]          | [图表]          |
+-----------------------------------+
```

### 6.3 K8s 健康探针配置

```yaml
# Kubernetes Deployment配置
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: app
        livenessProbe:
          httpGet:
            path: /manage/health/liveness
            port: 9001
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /manage/health/readiness
            port: 9001
          initialDelaySeconds: 20
          periodSeconds: 5
```

## 7. 高级功能与技巧

### 7.1 指标标签与维度

```java
@RestController
public class OrderController {
    
    private final MeterRegistry registry;
    private final Counter errorCounter;
    
    public OrderController(MeterRegistry registry) {
        this.registry = registry;
        this.errorCounter = Counter.builder("api.errors")
                .tag("controller", "OrderController")
                .register(registry);
    }
    
    @GetMapping("/orders")
    public ResponseEntity<?> getOrders() {
        try {
            // 业务逻辑
            return ResponseEntity.ok(orders);
        } catch (Exception e) {
            // 记录错误并添加标签
            errorCounter.increment();
            registry.counter("api.errors", 
                    "controller", "OrderController",
                    "exception", e.getClass().getSimpleName())
                .increment();
            return ResponseEntity.internalServerError().build();
        }
    }
}
```

### 7.2 分布式追踪集成

```yaml
# 添加依赖
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>

# 配置
spring:
  sleuth:
    sampler:
      probability: 1.0 # 采样率
  zipkin:
    base-url: http://zipkin-server:9411
```

### 7.3 审计事件

```java
@Configuration
@EnableAuditEvents
public class AuditConfig {
    // 自动启用审计事件端点
}

// 发布审计事件
@Component
public class OrderService {
    
    private final AuditEventRepository auditEventRepository;
    
    public void createOrder(Order order) {
        // 业务逻辑
        
        // 记录审计事件
        AuditEvent event = new AuditEvent(
            "admin", 
            "ORDER_CREATED",
            Map.of("orderId", order.getId(), "amount", order.getAmount())
        );
        auditEventRepository.add(event);
    }
}
```

## 8. 常见问题与解决方案

### 8.1 端点访问404问题

**原因分析**：

1. 端点未暴露 (`management.endpoints.web.exposure.include`)
2. 安全配置拦截
3. 基础路径配置错误

**解决方案**：

```yaml
# 确保端点暴露
management:
  endpoints:
    web:
      exposure:
        include: health,info
      base-path: /manage
```

### 8.2 健康检查失败

**常见原因**：

- 数据库连接失败
- 磁盘空间不足
- 第三方服务不可用
- 自定义健康指示器返回DOWN

**诊断步骤**：

1. 查看 `/manage/health` 返回的详细状态
2. 检查相关组件的日志
3. 使用 `/manage/env` 检查配置
4. 验证外部服务连通性

### 8.3 性能影响优化

**优化策略**：

```yaml
management:
  metrics:
    export:
      prometheus:
        step: 60s # 拉取间隔
    enable:
      jvm: true
      system: false # 禁用不必要指标
      logback: false
  endpoint:
    metrics:
      enabled: true
    prometheus:
      enabled: true
```

## 9. 生产环境检查清单

### 9.1 安全配置

- [ ] 使用独立管理端口
- [ ] 启用HTTPS访问
- [ ] 配置RBAC权限控制
- [ ] 敏感端点脱敏处理
- [ ] 禁用shutdown端点

### 9.2 监控配置

- [ ] 集成Prometheus指标导出
- [ ] 配置Grafana监控仪表板
- [ ] 设置关键指标告警（CPU、内存、错误率）
- [ ] 实现日志集中收集（ELK/Splunk）

### 9.3 健康检查

- [ ] 配置K8s存活/就绪探针
- [ ] 添加自定义健康指示器
- [ ] 设置健康检查超时时间
- [ ] 定期验证健康端点

### 9.4 性能优化

- [ ] 调整指标收集频率
- [ ] 禁用不必要指标
- [ ] 使用缓存响应频繁请求
- [ ] 限制历史数据存储量

## 10. 最佳实践总结

### 10.1 核心原则

1. **最小暴露原则**：仅暴露必要端点
2. **纵深防御**：多层安全防护（网络层、应用层、数据层）
3. **持续监控**：集成到统一监控平台
4. **自动化运维**：与CI/CD和编排系统集成
5. **文档化**：维护端点使用文档

### 10.2 进阶技巧

- **动态日志调整**：通过`/manage/loggers`端点实时修改日志级别
- **配置热更新**：结合Spring Cloud Config实现配置动态刷新
- **自定义指标**：使用Micrometer收集业务关键指标
- **审计追踪**：记录关键操作审计日志
- **跨服务追踪**：集成Sleuth+Zipkin实现分布式追踪

> **架构师提示**：将Actuator端点视为应用的管理API，设计时考虑版本兼容性、访问控制和监控分析，构建完整的可观测性体系。

## 附录：常用端点响应示例

### health 端点

```json
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": { "database": "MySQL" }
    },
    "diskSpace": {
      "status": "UP",
      "details": { "free": 250GB, "threshold": 10GB }
    },
    "customService": {
      "status": "UP"
    }
  }
}
```

### metrics 端点

```json
{
  "names": [
    "jvm.memory.used",
    "jvm.threads.live",
    "http.server.requests",
    "orders.created"
  ]
}

# 具体指标
GET /manage/metrics/jvm.memory.used
{
  "name": "jvm.memory.used",
  "measurements": [
    { "statistic": "VALUE", "value": 450.5 }
  ],
  "availableTags": [
    { "tag": "area", "values": ["heap", "nonheap"] }
  ]
}
```

### env 端点

```json
{
  "activeProfiles": ["production"],
  "propertySources": [
    {
      "name": "applicationConfig",
      "properties": {
        "server.port": { "value": 8080 },
        "spring.datasource.url": { "value": "******" }
      }
    }
  ]
}
```
