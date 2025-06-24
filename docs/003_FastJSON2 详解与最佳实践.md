# FastJSON2 详解与最佳实践

> 阿里巴巴新一代高性能 JSON 处理库

- [FastJSON2 官网](https://github.com/alibaba/fastjson2)

## 一、FastJSON2 概述

### 1.1 什么是 FastJSON2？

FastJSON2 是 FastJSON 项目的全新架构版本，针对 JSON 处理进行了**全面重构**，解决了 FastJSON1 的安全漏洞和设计缺陷，提供：

- ⚡ **极致性能**：解析速度提升 2-10 倍
- 🔒 **增强安全性**：消除反序列化漏洞
- 📐 **标准化兼容**：严格遵循 JSON RFC 标准
- 📦 **模块化设计**：按需引入功能组件

### 1.2 与 FastJSON1 关键差异

| 特性 | FastJSON1 | FastJSON2 |
|------|-----------|-----------|
| **架构** | 单体设计 | 模块化设计 |
| **性能** | 高 | **极高**（提升2-10倍） |
| **安全性** | 多个高危漏洞 | 安全加固设计 |
| **内存占用** | 较高 | 降低30%-50% |
| **API设计** | 历史包袱 | 全新合理化API |
| **Java版本** | Java 6+ | **Java 8+** |

### 1.3 性能基准（基于 JMH）

| 操作 | FastJSON2 (ops/s) | Jackson | Gson |
|------|------------------|---------|------|
| **序列化** | 1,850,000 | 1,120,000 | 980,000 |
| **反序列化** | 1,420,000 | 950,000 | 760,000 |
| **大文本解析** | 3.2 GB/s | 2.1 GB/s | 1.8 GB/s |

> 测试环境：JDK 17, 8核CPU, 32GB RAM

## 二、环境配置

### 2.1 基础依赖

**Maven**:

```xml
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.43</version>
</dependency>
```

**Gradle**:

```groovy
implementation 'com.alibaba.fastjson2:fastjson2:2.0.43'
```

### 2.2 可选模块

```xml
<!-- 支持JSONPath -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2-extension</artifactId>
    <version>2.0.43</version>
</dependency>

<!-- Kotlin支持 -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2-kotlin</artifactId>
    <version>2.0.43</version>
</dependency>
```

## 三、核心 API 使用

### 3.1 基础序列化/反序列化

```java
// 序列化
User user = new User(1, "Alice", 30);
String json = JSON.toJSONString(user); 
// 结果: {"id":1,"name":"Alice","age":30}

// 反序列化
User parsedUser = JSON.parseObject(json, User.class);

// 泛型反序列化
List<User> users = JSON.parseArray(jsonArrayStr, User.class);
```

### 3.2 配置选项

```java
// 创建配置实例
JSONWriter.Context writeContext = new JSONWriter.Context(
    JSONWriter.Feature.WriteEnumsUsingName,
    JSONWriter.Feature.PrettyFormat
);

JSONReader.Context readContext = new JSONReader.Context(
    JSONReader.Feature.SupportSmartMatch,
    JSONReader.Feature.UseNativeObject
);

// 应用配置
String json = JSON.toJSONString(user, writeContext);
User user = JSON.parseObject(json, User.class, readContext);
```

### 3.3 常用配置特性

| 特性 | 类型 | 描述 |
|------|------|------|
| `PrettyFormat` | Writer | 美化输出格式 |
| `WriteEnumsUsingName` | Writer | 枚举输出为名称 |
| `SupportSmartMatch` | Reader | 智能字段匹配（忽略大小写/下划线） |
| `UseNativeObject` | Reader | 使用原生对象代替Map |
| `IgnoreNoneSerializable` | Both | 忽略不可序列化字段 |

## 四、高级特性

### 4.1 注解使用

```java
public class Product {
    @JSONField(name = "product_id", ordinal = 1)
    private Long id;
    
    @JSONField(serialize = false) // 不序列化
    private String internalCode;
    
    @JSONField(format = "yyyy-MM-dd HH:mm:ss")
    private Date createTime;
    
    @JSONField(deserializeFeatures = JSONReader.Feature.ErrorOnEnumNotMatch)
    private ProductType type;
}
```

### 4.2 自定义序列化

```java
public class MoneySerializer implements ObjectSerializer<Money> {
    @Override
    public void write(
        JSONWriter writer, 
        Money money, 
        Object fieldName, 
        Type fieldType, 
        long features
    ) {
        writer.writeString(money.getAmount() + " " + money.getCurrency());
    }
}

// 注册自定义序列化器
JSON.register(Money.class, new MoneySerializer());
```

### 4.3 JSONPath 查询

```java
String json = "{\"store\":{\"books\":[{\"title\":\"Java\",\"price\":39.99}]}}";

// 提取所有书名
List<String> titles = JSONPath.extract(json, "$.store.books[*].title");

// 修改价格
JSONPath.set(json, "$.store.books[0].price", 35.99);
```

### 4.4 流式 API

```java
// 流式写入
JSONWriter writer = JSONWriter.of();
writer.startObject();
writer.writeName("id").writeInt(101);
writer.writeName("items");
writer.startArray();
writer.writeString("item1");
writer.writeString("item2");
writer.endArray();
writer.endObject();
String json = writer.toString();

// 流式读取
JSONReader reader = JSONReader.of(json);
reader.startObject();
while (reader.hasNext()) {
    String key = reader.readFieldName();
    if ("items".equals(key)) {
        reader.startArray();
        while (reader.hasNext()) {
            String item = reader.readString();
            // 处理item
        }
        reader.endArray();
    } else {
        reader.skipValue();
    }
}
reader.endObject();
```

## 五、安全最佳实践

### 5.1 反序列化安全防护

```java
// 启用安全模式（推荐）
JSONReader.Context context = new JSONReader.Context();
context.config(JSONReader.Feature.SupportAutoType, false); // 关闭AutoType
context.setAutoTypeBeforeHandler((typeName) -> {
    // 白名单校验
    if (!typeName.startsWith("com.safe.")) {
        throw new JSONException("Forbidden type: " + typeName);
    }
});

User user = JSON.parseObject(json, User.class, context);
```

### 5.2 关键安全措施

1. **禁用 AutoType**：

   ```java
   // 全局禁用（推荐）
   JSON.config(JSONReader.Feature.SupportAutoType, false);
   ```

2. **使用最新版本**：及时修复安全漏洞
3. **输入长度限制**：

   ```java
   JSONReader reader = JSONReader.of(inputStream, StandardCharsets.UTF_8);
   reader.setMaxSize(1024 * 1024); // 限制1MB
   ```

4. **类型白名单**：

   ```java
   ParserConfig config = new ParserConfig();
   config.addAccept("com.yourcompany.model.");
   ```

## 六、性能优化

### 6.1 高性能编码技巧

```java
// 1. 重用JSONWriter实例（线程不安全，需ThreadLocal封装）
ThreadLocal<JSONWriter> writerCache = ThreadLocal.withInitial(JSONWriter::of);

// 2. 使用原生缓冲区
JSONWriter writer = JSONWriter.ofUTF8();
writer.writeAny(object);
byte[] bytes = writer.getBytes(); // 零拷贝获取字节

// 3. 预编译序列化器
ObjectWriter<User> userWriter = JSONFactory.getDefaultObjectWriterProvider().getObjectWriter(User.class);
String json = userWriter.write(JSONWriter.of(), user).toString();
```

### 6.2 配置调优参数

```java
// 创建高性能配置
JSONWriter.Context writeContext = new JSONWriter.Context(
    JSONWriter.Feature.WriteNonStringValueAsString,
    JSONWriter.Feature.OptimizedForAscii
);

JSONReader.Context readContext = new JSONReader.Context(
    JSONReader.Feature.UseNativeObject,
    JSONReader.Feature.SupportArrayToBean
);
```

## 七、Spring Boot 集成

### 7.1 配置 HttpMessageConverter

```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
        FastJsonHttpMessageConverter converter = new FastJsonHttpMessageConverter();
        converter.setSupportedMediaTypes(Collections.singletonList(MediaType.APPLICATION_JSON));
        
        // 配置序列化特性
        JSONWriter.Context context = new JSONWriter.Context(
            JSONWriter.Feature.PrettyFormat,
            JSONWriter.Feature.WriteMapNullValue
        );
        converter.setWriterContext(context);
        
        converters.add(0, converter);
    }
}
```

### 7.2 自定义序列化规则

```java
@Bean
public ObjectMapper fastjsonObjectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    
    // 配置全局日期格式
    mapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));
    
    // 配置模块
    SimpleModule module = new SimpleModule();
    module.addSerializer(BigDecimal.class, new BigDecimalSerializer());
    mapper.registerModule(module);
    
    return mapper;
}
```

## 八、最佳实践总结

### 8.1 使用准则

1. **序列化**：

   ```java
   // 推荐方式
   String json = JSON.toJSONString(object, 
       JSONWriter.Feature.PrettyFormat,
       JSONWriter.Feature.WriteNulls
   );
   
   // 避免（性能差）
   String json = JSONObject.toJSONString(object);
   ```

2. **反序列化**：

   ```java
   // 安全方式
   User user = JSON.parseObject(json, User.class, 
       JSONReader.Feature.SupportSmartMatch
   );
   
   // 危险方式（避免使用）
   User user = JSON.parseObject(json, User.class, Feature.SupportAutoType);
   ```

### 8.2 架构建议

1. **DTO 设计原则**：
   - 显式定义所有字段
   - 避免嵌套超过3层
   - 使用不可变对象

2. **服务边界**：

   ```mermaid
   graph LR
   A[Controller] -->|FastJSON2| B(Service)
   B -->|领域对象| C[Repository]
   D[外部服务] -->|Protobuf/XML| B
   ```

### 8.3 错误处理

```java
try {
    return JSON.parseObject(json, type, features);
} catch (JSONException e) {
    // 处理格式错误
    log.error("JSON parse error: {}", json, e);
    throw new BusinessException("INVALID_JSON");
} catch (Exception e) {
    // 处理其他异常
    throw new SystemException("JSON_PROCESS_FAILED");
}
```

## 九、常见问题解决方案

| 问题 | 解决方案 |
|------|----------|
| **日期格式不一致** | 使用 `@JSONField(format)` 或全局配置 |
| **字段名不匹配** | 启用 `SupportSmartMatch` 或使用 `@JSONField(name)` |
| **循环引用** | 使用 `@JSONField(serialize = false)` 忽略反向引用 |
| **超大JSON处理** | 使用流式API (`JSONReader`/`JSONWriter`) |
| **多态类型处理** | 使用 `@JSONType(typeKey="type")` 注解 |
| **Kotlin兼容问题** | 添加 `fastjson2-kotlin` 模块 |

## 十、迁移指南（FastJSON1 → FastJSON2）

### 10.1 兼容性处理

```java
// FastJSON1 代码
JSONObject obj = JSON.parseObject(json);

// 修改为
JSONObject obj = JSON.parseObject(json, JSONObject.class);
```

### 10.2 不兼容变更处理

| FastJSON1 用法 | FastJSON2 替代方案 |
|----------------|---------------------|
| `SerializerFeature.WriteClassName` | **禁止使用**（安全风险） |
| `Feature.AutoCloseSource` | 使用 `JSONReader.Feature.AutoCloseSource` |
| `TypeUtils.cast()` | 使用标准类型转换 |
| `JSONPath.eval()` | 使用 `JSONPath.extract()` |

### 10.3 迁移步骤

1. 替换依赖为 `fastjson2`
2. 全局搜索替换包名：`com.alibaba.fastjson` → `com.alibaba.fastjson2`
3. 修复编译错误（主要涉及API变更）
4. 运行测试用例，重点验证：
   - 日期格式
   - 枚举处理
   - 空值处理
   - 泛型类型
5. 性能基准测试

## 附录：核心 API 速查表

### 序列化 API

| 方法 | 描述 |
|------|------|
| `JSON.toJSONString(Object)` | 对象→JSON字符串 |
| `JSON.toJSONBytes(Object)` | 对象→UTF8字节 |
| `JSON.writeTo(OutputStream, Object)` | 流式写入 |
| `JSONWriter.of()` | 创建写入上下文 |

### 反序列化 API

| 方法 | 描述 |
|------|------|
| `JSON.parseObject(String, Class)` | JSON→Java对象 |
| `JSON.parseArray(String, Class)` | JSON→对象列表 |
| `JSON.parseObject(byte[], Class)` | 字节→Java对象 |
| `JSONReader.of(InputStream)` | 流式读取 |

### 注解参考

| 注解 | 描述 |
|------|------|
| `@JSONField` | 字段级配置 |
| `@JSONType` | 类级配置 |
| `@JSONCompiler` | 启用ASM字节码增强 |

> **最佳实践总结**：FastJSON2 在性能和安全方面实现了质的飞跃，建议：
>
> 1. 新项目直接使用 FastJSON2
> 2. 存量项目制定迁移计划
> 3. 严格遵守安全规范（尤其禁用AutoType）
> 4. 关键服务使用流式API处理大JSON
> 5. 结合性能测试持续优化配置
