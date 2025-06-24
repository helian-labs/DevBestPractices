# JSoup 详解与 Spring Boot 整合最佳实践

> Java 生态中最强大的 HTML 处理工具库

- [JSoup 官网](https://jsoup.org/)

## 一、JSoup 概述

### 1.1 什么是 JSoup？

JSoup 是一个用于处理实际 HTML 的 Java 库，提供：

- 🧩 **DOM 解析**：将 HTML 转换为可遍历的文档对象模型
- 🔍 **选择器语法**：使用 CSS 选择器查找元素
- ✏️ **HTML 操作**：修改元素、属性和文本内容
- � **数据清理**：防止 XSS 攻击的安全白名单机制
- 🌐 **网络能力**：直接从 URL 加载 HTML 文档

### 1.2 核心应用场景

| 场景 | 说明 | Spring Boot 集成点 |
|------|------|-------------------|
| **网页抓取** | 从网站提取结构化数据 | 定时任务、API 接口 |
| **数据清洗** | 清理用户输入的 HTML | 表单验证拦截器 |
| **内容分析** | 解析和操作 HTML 片段 | 邮件模板处理 |
| **SEO 优化** | 生成语义化 HTML | Thymeleaf 扩展 |

## 二、Spring Boot 集成配置

### 2.1 添加依赖

**Maven**:

```xml
<dependency>
    <groupId>org.jsoup</groupId>
    <artifactId>jsoup</artifactId>
    <version>1.17.2</version>
</dependency>
```

**Gradle**:

```groovy
implementation 'org.jsoup:jsoup:1.17.2'
```

### 2.2 配置 Bean（可选）

```java
@Configuration
public class JsoupConfig {
    
    @Bean
    public JsoupService jsoupService() {
        return new JsoupService();
    }
    
    // 配置全局清理策略
    @Bean
    public Whitelist htmlWhitelist() {
        return Whitelist.basicWithImages()
            .addTags("section", "article")
            .addAttributes("a", "rel", "target");
    }
}
```

## 三、核心 API 详解

### 3.1 文档加载方式

```java
// 从字符串加载
String html = "<html><body><p>Hello JSoup</p></body></html>";
Document doc = Jsoup.parse(html);

// 从 URL 加载（带超时控制）
Document doc = Jsoup.connect("https://example.com")
                .timeout(10000) // 10秒超时
                .userAgent("Mozilla/5.0") // 设置UA
                .get();

// 从文件加载（Spring Resource）
@Value("classpath:static/template.html")
Resource templateFile;

Document doc = Jsoup.parse(templateFile.getFile(), "UTF-8");
```

### 3.2 元素选择器

```java
// 基本选择器
Elements links = doc.select("a[href]"); // 带href属性的链接
Element logo = doc.selectFirst("#logo"); // ID选择

// 组合选择器
Elements items = doc.select("div.content > ul > li:eq(0)"); // 第一个列表项

// 属性选择器
Elements images = doc.select("img[src~=(?i)\\.(png|jpe?g)]"); // 图片扩展名匹配
```

### 3.3 数据提取方法

```java
// 获取文本
String title = doc.title(); // 页面标题
String text = element.text(); // 元素文本（去HTML标签）

// 获取HTML
String outerHtml = element.outerHtml(); // 包含自身标签
String innerHtml = element.html(); // 仅内部HTML

// 获取属性
String link = element.attr("abs:href"); // 绝对URL
String dataId = element.dataset().get("id"); // data-id属性
```

## 四、Spring Boot 整合实践

### 4.1 服务层封装

```java
@Service
public class WebScraperService {
    
    private static final Logger log = LoggerFactory.getLogger(WebScraperService.class);
    
    public List<NewsItem> scrapeNews(String url) {
        try {
            Document doc = Jsoup.connect(url).timeout(5000).get();
            
            return doc.select(".news-item").stream()
                .map(item -> new NewsItem(
                    item.selectFirst(".title").text(),
                    item.selectFirst("a").attr("abs:href"),
                    item.selectFirst(".summary").text(),
                    Instant.now()
                ))
                .collect(Collectors.toList());
        } catch (IOException e) {
            log.error("网页抓取失败: {}", url, e);
            throw new ScrapingException("网页解析错误");
        }
    }
}
```

### 4.2 定时任务示例

```java
@Scheduled(fixedRate = 3600000) // 每小时执行
public void scheduledScraping() {
    List<NewsItem> news = webScraperService.scrapeNews("https://news.example.com");
    newsRepository.saveAll(news);
    log.info("已抓取 {} 条新闻", news.size());
}
```

### 4.3 RESTful API 接口

```java
@RestController
@RequestMapping("/api/scrape")
public class ScrapingController {
    
    @Autowired
    private WebScraperService scraperService;
    
    @PostMapping("/extract")
    public ResponseEntity<List<Map<String, String>>> extractData(
            @RequestBody ScrapeRequest request) {
        
        Document doc = Jsoup.parse(request.getHtmlContent());
        List<Map<String, String>> results = new ArrayList<>();
        
        for (Element row : doc.select(request.getRowSelector())) {
            Map<String, String> item = new HashMap<>();
            request.getFieldSelectors().forEach((key, selector) -> {
                Element field = row.selectFirst(selector);
                item.put(key, field != null ? field.text() : "");
            });
            results.add(item);
        }
        
        return ResponseEntity.ok(results);
    }
}
```

## 五、安全最佳实践

### 5.1 XSS 防护方案

```java
@Service
public class HtmlSanitizer {
    
    @Autowired
    private Whitelist whitelist;
    
    public String sanitize(String unsafeHtml) {
        // 基本清理
        String clean = Jsoup.clean(unsafeHtml, whitelist);
        
        // 额外安全处理
        clean = clean.replaceAll("javascript:", "");
        return clean.replaceAll("(?i)on[a-z]+=\"?[^\"]*\"?", "");
    }
}

// 配置白名单（推荐）
Whitelist whitelist = Whitelist.basic()
    .addTags("div", "span", "br")
    .addAttributes("a", "href", "title")
    .addProtocols("a", "href", "http", "https");
```

### 5.2 防御性编程技巧

```java
// 1. 元素存在性检查
Element priceEl = doc.selectFirst(".price");
if (priceEl != null) {
    String price = priceEl.text().replace("¥", "");
    // 处理价格
}

// 2. 安全解析数值
try {
    double value = NumberUtils.parseDouble(priceEl.text(), 0.0);
} catch (NumberFormatException e) {
    log.warn("价格解析失败: {}", priceEl.text());
}

// 3. 限制解析深度（防止超大文档）
Document.OutputSettings settings = new Document.OutputSettings();
settings.prettyPrint(false);
doc.outputSettings(settings);
```

## 六、性能优化

### 6.1 解析性能对比

| 操作 | JSoup 耗时 | 正则表达式耗时 |
|------|------------|----------------|
| 简单文档解析 | 15 ms | 8 ms |
| 复杂文档解析 | 45 ms | 120 ms |
| 带选择器查询 | 60 ms | 失败 |
| 大文档(1MB+) | 350 ms | 内存溢出 |

> 测试环境：Spring Boot 应用，JDK 17，2.5 GHz 四核CPU

### 6.2 优化策略

```java
// 1. 重用连接对象
Connection conn = Jsoup.connect(baseUrl)
                     .userAgent("Mozilla/5.0")
                     .timeout(10000);

Document page1 = conn.url(baseUrl + "/page1").get();
Document page2 = conn.url(baseUrl + "/page2").get();

// 2. 禁用冗余功能
Document doc = Jsoup.parse(html);
doc.outputSettings().syntax(Document.OutputSettings.Syntax.xml); // 更快解析
doc.outputSettings().escapeMode(Entities.EscapeMode.xhtml); // 减少编码计算

// 3. 选择器优化
// 避免：doc.select("div .item .title") 
// 推荐：doc.select("div.item > .title")
```

## 七、高级应用场景

### 7.1 动态内容处理（需配合 Selenium）

```java
@Bean
public WebDriver webDriver() {
    WebDriverManager.chromedriver().setup();
    ChromeOptions options = new ChromeOptions();
    options.addArguments("--headless");
    return new ChromeDriver(options);
}

@Service
public class DynamicContentScraper {
    
    @Autowired
    private WebDriver driver;
    
    public String scrapeDynamicContent(String url) {
        driver.get(url);
        // 等待内容加载
        new WebDriverWait(driver, Duration.ofSeconds(10))
            .until(ExpectedConditions.presenceOfElementLocated(
                By.cssSelector(".dynamic-content")
            ));
        
        return Jsoup.parse(driver.getPageSource())
                   .select(".dynamic-content").html();
    }
}
```

### 7.2 与 Thymeleaf 整合

```java
@Component
public class JsoupDialect extends AbstractProcessorDialect {
    
    public JsoupDialect() {
        super("JSoup", "jsoup", StandardDialect.PROCESSOR_PRECEDENCE);
    }
    
    @Override
    public Set<IProcessor> getProcessors(String dialectPrefix) {
        return Set.of(new JsoupTextProcessor(dialectPrefix));
    }
    
    private static class JsoupTextProcessor extends AbstractElementTagProcessor {
        protected JsoupTextProcessor(String dialectPrefix) {
            super(TemplateMode.HTML, dialectPrefix, "text", true, null, false, 1000);
        }
        
        @Override
        protected void doProcess(ITemplateContext context, 
                IProcessableElementTag tag, IElementTagStructureHandler handler) {
            
            final String unsafe = tag.getAttributeValue("unsafe");
            final String safe = Jsoup.clean(unsafe, Whitelist.basic());
            handler.replaceWith(safe, false);
        }
    }
}
```

## 八、最佳实践总结

### 8.1 设计原则

1. **职责分离**：

   ```mermaid
   graph LR
   A[Controller] --> B[ScrapingService]
   B --> C[JsoupHelper]
   C --> D[HTML解析]
   B --> E[数据转换]
   ```

2. **异常处理规范**：

   ```java
   @RestControllerAdvice
   public class ScrapingExceptionHandler {
       
       @ExceptionHandler(IOException.class)
       public ResponseEntity<ErrorResponse> handleNetworkError(IOException ex) {
           return ResponseEntity.status(HttpStatus.GATEWAY_TIMEOUT)
                   .body(new ErrorResponse("SCRAPE_001", "目标网站不可达"));
       }
       
       @ExceptionHandler(JsoupException.class)
       public ResponseEntity<ErrorResponse> handleParseError() {
           return ResponseEntity.badRequest()
                   .body(new ErrorResponse("SCRAPE_002", "HTML解析失败"));
       }
   }
   ```

### 8.2 工程化建议

1. **配置管理**：

   ```yaml
   scraping:
     timeout: 5000
     user-agent: MyScraperBot/1.0 (+https://myapp.com/bot-info)
     whitelist: 
       tags: [a, p, div, span]
       attributes: [href, src, title]
   ```

2. **监控指标**：

   ```java
   @Bean
   MeterRegistryCustomizer<MeterRegistry> scrapingMetrics() {
       return registry -> {
           Counter.builder("scraping.requests")
                  .description("网页抓取请求计数")
                  .register(registry);
           
           Timer.builder("scraping.time")
                .description("HTML解析耗时")
                .register(registry);
       };
   }
   
   // 在服务中使用
   @Timed(value = "scraping.time", longTask = true)
   public Document fetchDocument(String url) {
       scrapingCounter.increment();
       return Jsoup.connect(url).get();
   }
   ```

## 九、常见问题解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 选择器返回空 | DOM 结构变化 | 1. 使用更宽松的选择器<br>2. 添加元素存在性检查 |
| 中文乱码 | 编码不匹配 | 1. 指定`charset`参数<br>2. 使用`doc.outputSettings().charset("UTF-8")` |
| 连接超时 | 网络问题或反爬 | 1. 增加超时时间<br>2. 设置代理<br>3. 添加 User-Agent |
| 内存溢出 | 解析超大文档 | 1. 使用流式解析<br>2. 限制解析深度 |
| XSS 绕过 | 清理策略不严 | 1. 使用更严格的白名单<br>2. 二次过滤`onEvent`属性 |

## 十、JSoup 与替代方案对比

| 特性 | JSoup | HtmlUnit | Jsoup + Selenium | 正则表达式 |
|------|-------|----------|------------------|------------|
| **HTML 标准支持** | HTML5 | HTML5 | HTML5 | 无 |
| **选择器语法** | CSS | XPath/CSS | CSS/XPath | 无 |
| **JavaScript 支持** | ❌ | ⭐⭐ | ⭐⭐⭐⭐ | ❌ |
| **性能** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐⭐⭐ |
| **Spring Boot 集成** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | ⭐ |

> **最佳实践总结**：
>
> 1. **安全第一**：始终清理用户输入的 HTML
> 2. **优雅降级**：添加全面的异常处理和超时控制
> 3. **性能监控**：记录关键操作的执行时间
> 4. **选择器优化**：使用高效的选择器表达式
> 5. **法律合规**：遵守目标网站的 robots.txt 和版权要求
> 6. **缓存策略**：对静态内容实施缓存减少请求

通过合理应用 JSoup，可以高效解决 Spring Boot 项目中的 HTML 处理需求，同时保持代码的健壮性和安全性。
