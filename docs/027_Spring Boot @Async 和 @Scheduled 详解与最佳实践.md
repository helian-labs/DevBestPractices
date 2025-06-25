# Spring Boot @Async 和 @Scheduled 详解与最佳实践

## 1. 引言

在构建现代Java应用时，**异步任务**和**定时任务**是常见的需求。Spring Boot提供了`@Async`和`@Scheduled`两个强大的注解来简化这些功能的实现。然而，**默认线程池配置存在严重缺陷**，可能导致应用性能下降甚至崩溃。本文将深入探讨这两个注解的用法、线程池配置及最佳实践。

## 2. @Async 详解

### 2.1 基本用法

`@Async`用于标记异步执行的方法：

```java
@Service
public class NotificationService {

    @Async
    public void sendEmail(String recipient, String content) {
        // 模拟耗时操作
        try {
            Thread.sleep(2000);
            System.out.println("Email sent to: " + recipient);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

启用异步支持：

```java
@SpringBootApplication
@EnableAsync
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 2.2 默认线程池的缺陷

Spring Boot默认使用`SimpleAsyncTaskExecutor`，其存在以下问题：

- **每次调用创建新线程**，无线程复用
- **无队列容量限制**，容易导致OOM
- **无最大线程数限制**，可能耗尽系统资源
- **无拒绝策略**，线程爆炸时不提供保护

### 2.3 自定义线程池实现

#### 2.3.1 配置自定义线程池

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);        // 核心线程数
        executor.setMaxPoolSize(10);         // 最大线程数
        executor.setQueueCapacity(100);      // 队列容量
        executor.setThreadNamePrefix("Async-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}
```

#### 2.3.2 指定特定线程池

```java
@Async("customTaskExecutor")
public void processOrder(Order order) {
    // 订单处理逻辑
}
```

### 2.4 最佳实践

1. **始终配置自定义线程池**，避免使用默认实现
2. **合理设置线程池参数**：
   - CPU密集型任务：核心线程数 = CPU核心数 + 1
   - IO密集型任务：核心线程数 = CPU核心数 * 2
3. **设置合理的拒绝策略**，推荐：
   - `CallerRunsPolicy`：由调用线程执行任务
   - `CustomPolicy`：记录日志并降级处理
4. **使用线程池监控**（如Micrometer）
5. **处理异步异常**：

   ```java
   @Async
   public Future<String> asyncMethod() {
       try {
           // 业务逻辑
           return new AsyncResult<>("Success");
       } catch (Exception e) {
           return new AsyncResult<>("Failed");
       }
   }
   ```

## 3. @Scheduled 详解

### 3.1 基本用法

```java
@Service
public class ReportService {

    @Scheduled(fixedRate = 300000) // 每5分钟执行一次
    public void generateDailyReport() {
        System.out.println("Generating report: " + new Date());
    }
    
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void cleanupDatabase() {
        System.out.println("Cleaning up database...");
    }
}
```

启用定时任务：

```java
@SpringBootApplication
@EnableScheduling
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 3.2 默认线程池问题

- **单线程执行所有定时任务**（`ScheduledAnnotationBeanPostProcessor`）
- 任务执行时间长会阻塞后续任务
- 无任务隔离，一个任务异常影响所有任务

### 3.3 自定义调度线程池

#### 3.3.1 配置自定义调度器

```java
@Configuration
public class SchedulerConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(taskExecutor());
    }

    @Bean(destroyMethod = "shutdown")
    public Executor taskExecutor() {
        return Executors.newScheduledThreadPool(5); // 5个线程
    }
}
```

#### 3.3.2 使用特定线程池执行

```java
@Scheduled(fixedDelay = 10000, scheduler = "reportScheduler")
public void generateReport() {
    // 报表生成逻辑
}
```

### 3.4 最佳实践

1. **根据任务类型分离线程池**：
   - 关键任务：独立线程池
   - 非关键任务：共享线程池
2. **设置合理的执行策略**：
   - `fixedDelay`：上次结束到下次开始的间隔
   - `fixedRate`：固定频率执行（注意重叠问题）
   - `cron`：复杂时间表达式
3. **添加分布式锁**（如Redis锁），防止多实例重复执行
4. **任务执行监控**，记录开始/结束时间
5. **优雅处理异常**，避免任务终止

## 4. Spring Boot 线程池清单

### 4.1 内置线程池类型

Spring Boot本身不直接提供线程池实现，但支持以下Java标准线程池：

| 线程池类型                | 实现类                     | 特点                                                                 |
|--------------------------|---------------------------|----------------------------------------------------------------------|
| FixedThreadPool          | `Executors.newFixedThreadPool` | 固定大小线程池                                                       |
| CachedThreadPool         | `Executors.newCachedThreadPool`| 弹性线程池，空闲线程60秒回收                                        |
| SingleThreadExecutor     | `Executors.newSingleThreadExecutor`| 单线程执行，保证任务顺序                                           |
| ScheduledThreadPool      | `Executors.newScheduledThreadPool`| 支持定时/周期性任务                                                |
| WorkStealingPool         | `Executors.newWorkStealingPool` | Java 8+，基于ForkJoinPool，适合并行计算                             |

### 4.2 Spring提供的增强实现

| 实现类                  | 特点                                                                 |
|------------------------|----------------------------------------------------------------------|
| `ThreadPoolTaskExecutor` | Spring包装的ThreadPoolExecutor，更易配置和集成                      |
| `ConcurrentTaskExecutor`| 适配java.util.concurrent.Executor的包装类                          |
| `SimpleAsyncTaskExecutor`| **不推荐生产使用**，每次请求新建线程                               |

## 5. 高级线程池配置

### 5.1 线程池参数调优公式

```java
// 最优线程数 = CPU核心数 * (1 + 等待时间/计算时间)
int optimalThreads = Runtime.getRuntime().availableProcessors() * 
                    (1 + (avgWaitTime / avgComputeTime));
```

### 5.2 动态线程池配置

使用Nacos/Apollo动态调整参数：

```java
@RefreshScope
@Bean
public ThreadPoolTaskExecutor taskExecutor(
    @Value("${thread-pool.core-size:5}") int coreSize,
    @Value("${thread-pool.max-size:10}") int maxSize) {
    
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(coreSize);
    executor.setMaxPoolSize(maxSize);
    // ...其他配置
    return executor;
}
```

### 5.3 线程池监控

集成Micrometer监控：

```java
@Bean
public ExecutorServiceMetrics executorServiceMetrics(
    MeterRegistry registry, 
    ThreadPoolTaskExecutor executor) {
    
    return new ExecutorServiceMetrics(
        executor.getThreadPoolExecutor(),
        "custom_thread_pool",
        Collections.emptyList()
    );
}
```

监控指标：

- `executor.pool.size`：当前线程数
- `executor.queue.size`：队列大小
- `executor.active.count`：活动线程数
- `executor.completed.count`：完成任务数

## 6. 常见陷阱与解决方案

| 陷阱 | 现象 | 解决方案 |
|------|------|----------|
| **OOM（内存溢出）** | 队列任务堆积导致内存溢出 | 设置合理队列大小，使用拒绝策略 |
| **任务饥饿** | 低优先级任务长期得不到执行 | 使用优先级队列或分离线程池 |
| **线程泄漏** | 线程数持续增长不释放 | 检查任务是否正常结束，使用线程池监控 |
| **上下文丢失** | 异步任务中无法获取SecurityContext | 使用`DelegatingSecurityContextAsyncTaskExecutor` |
| **定时任务堆积** | 任务执行时间 > 间隔时间 | 使用`fixedDelay`代替`fixedRate` |

## 7. 总结

正确使用`@Async`和`@Scheduled`能显著提升应用性能，但必须注意：

1. **避免使用默认线程池**，始终配置自定义线程池
2. **根据任务特性选择合适线程池类型**
3. **关键参数需监控和动态调整**（核心线程数、队列容量等）
4. **定时任务与异步任务使用不同线程池隔离**
5. **生产环境必须配置拒绝策略**

通过遵循本文的最佳实践，您可以构建高效、可靠的任务执行系统，充分利用系统资源同时避免常见并发问题。

> **注**：本文所有代码示例均基于Spring Boot 3.x版本。实际使用时请根据您的Spring Boot版本进行适当调整。
