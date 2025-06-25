# Servlet 5.0 详解与最佳实践

## 目录

- [Servlet 5.0 详解与最佳实践](#servlet-50-详解与最佳实践)
  - [目录](#目录)
  - [Servlet 简介与演进](#servlet-简介与演进)
  - [Servlet 5.0 核心特性](#servlet-50-核心特性)
  - [环境搭建与配置](#环境搭建与配置)
    - [依赖配置 (Maven)](#依赖配置-maven)
    - [web.xml 配置 (传统方式)](#webxml-配置-传统方式)
    - [注解配置 (Servlet 3.0+)](#注解配置-servlet-30)
  - [Servlet 生命周期详解](#servlet-生命周期详解)
  - [请求处理与响应生成](#请求处理与响应生成)
    - [处理 GET 请求](#处理-get-请求)
    - [处理 POST 请求与表单数据](#处理-post-请求与表单数据)
    - [处理路径参数](#处理路径参数)
  - [会话管理与会话跟踪](#会话管理与会话跟踪)
    - [创建和访问会话](#创建和访问会话)
    - [会话监听器](#会话监听器)
  - [过滤器与监听器](#过滤器与监听器)
    - [创建过滤器](#创建过滤器)
    - [应用场景](#应用场景)
  - [异步处理](#异步处理)
    - [基本异步处理](#基本异步处理)
    - [非阻塞I/O](#非阻塞io)
  - [Servlet 安全](#servlet-安全)
    - [声明式安全 (web.xml)](#声明式安全-webxml)
    - [编程式安全](#编程式安全)
    - [安全最佳实践](#安全最佳实践)
  - [HTTP/2 支持](#http2-支持)
    - [服务器推送](#服务器推送)
    - [HTTP/2 最佳实践](#http2-最佳实践)
  - [Servlet 最佳实践](#servlet-最佳实践)
    - [架构设计](#架构设计)
    - [性能优化](#性能优化)
    - [可维护性实践](#可维护性实践)
  - [常见问题与解决方案](#常见问题与解决方案)
    - [1. 内存泄漏问题](#1-内存泄漏问题)
    - [2. 并发问题](#2-并发问题)
    - [3. 文件上传限制](#3-文件上传限制)
    - [4. 中文乱码问题](#4-中文乱码问题)
    - [5. HTTP 方法不支持](#5-http-方法不支持)
  - [结语](#结语)

---

## Servlet 简介与演进

**Servlet** 是运行在 Web 服务器上的 Java 程序，用于处理客户端请求并生成动态内容。自 1997 年首次推出以来，Servlet API 已成为 Java EE/Jakarta EE 平台的核心组件。

**版本演进**：

- Servlet 1.0 (1997)：初始版本
- Servlet 2.5 (2005)：注解支持
- Servlet 3.0 (2009)：异步处理、注解配置
- Servlet 4.0 (2017)：HTTP/2 支持
- **Servlet 5.0 (2020)**：Jakarta EE 命名空间、增强的 HTTP/2 支持

> **关键变化**：Servlet 5.0 将包名从 `javax.servlet` 改为 `jakarta.servlet`，这是 Java EE 转移到 Eclipse 基金会后的重要变化。

---

## Servlet 5.0 核心特性

1. **全面支持 HTTP/2**
   - 服务器推送（Server Push）
   - 头部压缩（HPACK）
   - 多路复用（Multiplexing）

2. **增强的异步处理**
   - 更简洁的异步上下文管理
   - 支持 Reactive 编程模型

3. **安全性增强**
   - 改进的认证机制
   - 增强的 URL 模式匹配安全策略

4. **API 改进**
   - 新的 `HttpServletMapping` 接口
   - 增强的 `Part` 接口（文件上传）

---

## 环境搭建与配置

### 依赖配置 (Maven)

```xml
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
```

### web.xml 配置 (传统方式)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="https://jakarta.ee/xml/ns/jakartaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="https://jakarta.ee/xml/ns/jakartaee
                             https://jakarta.ee/xml/ns/jakartaee/web-app_5_0.xsd"
         version="5.0">
    
    <servlet>
        <servlet-name>UserServlet</servlet-name>
        <servlet-class>com.example.UserServlet</servlet-class>
    </servlet>
    
    <servlet-mapping>
        <servlet-name>UserServlet</servlet-name>
        <url-pattern>/users/*</url-pattern>
    </servlet-mapping>
</web-app>
```

### 注解配置 (Servlet 3.0+)

```java
@WebServlet(
    name = "ProductServlet",
    urlPatterns = {"/products", "/catalog"},
    initParams = {
        @WebInitParam(name = "maxItems", value = "100")
    },
    loadOnStartup = 1
)
public class ProductServlet extends HttpServlet {
    // Servlet 实现
}
```

---

## Servlet 生命周期详解

Servlet 生命周期由 Web 容器管理，包含三个阶段：

1. **初始化** - `init()` 方法

   ```java
   @Override
   public void init(ServletConfig config) throws ServletException {
       super.init(config);
       // 初始化资源：数据库连接、配置文件等
       log("Servlet initialized at " + new Date());
   }
   ```

2. **请求处理** - `service()` 方法

   ```java
   @Override
   protected void service(HttpServletRequest req, HttpServletResponse resp) 
       throws ServletException, IOException {
       
       // 前置处理（日志、验证等）
       logRequestDetails(req);
       
       // 调用 doGet/doPost 等具体方法
       super.service(req, resp);
       
       // 后置处理
   }
   ```

3. **销毁** - `destroy()` 方法

   ```java
   @Override
   public void destroy() {
       // 释放资源：关闭数据库连接、清理线程等
       log("Servlet destroyed at " + new Date());
   }
   ```

> **最佳实践**：避免在 `init()` 中执行耗时操作，使用异步初始化模式。

---

## 请求处理与响应生成

### 处理 GET 请求

```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
    throws ServletException, IOException {
    
    // 获取请求参数
    String id = req.getParameter("id");
    
    // 设置响应类型
    resp.setContentType("application/json");
    resp.setCharacterEncoding("UTF-8");
    
    // 生成 JSON 响应
    try (PrintWriter out = resp.getWriter()) {
        out.print("{\"id\": \"" + id + "\", \"name\": \"Product A\"}");
    }
}
```

### 处理 POST 请求与表单数据

```java
@Override
protected void doPost(HttpServletRequest req, HttpServletResponse resp) 
    throws ServletException, IOException {
    
    // 读取表单数据
    String username = req.getParameter("username");
    String password = req.getParameter("password");
    
    // 处理文件上传 (Servlet 5.0 增强)
    Part filePart = req.getPart("profilePic");
    String fileName = Paths.get(filePart.getSubmittedFileName()).getFileName().toString();
    filePart.write("/uploads/" + fileName);
    
    // 业务逻辑处理...
    
    // 重定向
    resp.sendRedirect(req.getContextPath() + "/success");
}
```

### 处理路径参数

```java
@WebServlet("/api/users/*")
public class UserServlet extends HttpServlet {
    
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
        throws ServletException, IOException {
        
        String pathInfo = req.getPathInfo(); // /123/profile
        if (pathInfo == null || pathInfo.equals("/")) {
            // 返回所有用户
            return;
        }
        
        String[] pathParts = pathInfo.split("/");
        if (pathParts.length >= 2) {
            String userId = pathParts[1]; // 123
            // 获取用户信息...
        }
    }
}
```

---

## 会话管理与会话跟踪

### 创建和访问会话

```java
HttpSession session = req.getSession(true); // 创建新会话（如果不存在）
session.setAttribute("user", authenticatedUser);
session.setMaxInactiveInterval(30 * 60); // 30分钟超时
```

### 会话监听器

```java
@WebListener
public class SessionLogger implements HttpSessionListener {
    
    @Override
    public void sessionCreated(HttpSessionEvent se) {
        System.out.println("Session created: " + se.getSession().getId());
    }
    
    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        System.out.println("Session destroyed: " + se.getSession().getId());
    }
}
```

> **安全最佳实践**：
>
> 1. 敏感数据不要存储在会话中
> 2. 会话ID定期更换
> 3. 使用HTTPS传输会话Cookie

---

## 过滤器与监听器

### 创建过滤器

```java
@WebFilter(urlPatterns = "/*")
public class LoggingFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        
        HttpServletRequest req = (HttpServletRequest) request;
        long startTime = System.currentTimeMillis();
        
        // 前置处理
        System.out.println("Request received for: " + req.getRequestURI());
        
        chain.doFilter(request, response); // 继续处理链
        
        // 后置处理
        long duration = System.currentTimeMillis() - startTime;
        System.out.println("Request processed in " + duration + "ms");
    }
}
```

### 应用场景

1. 认证/授权检查
2. 请求日志记录
3. 数据压缩
4. 字符编码转换
5. XSS 防护

---

## 异步处理

### 基本异步处理

```java
@WebServlet(urlPatterns = "/async", asyncSupported = true)
public class AsyncServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
        throws ServletException, IOException {
        
        AsyncContext asyncCtx = req.startAsync();
        asyncCtx.setTimeout(30000); // 30秒超时
        
        // 使用线程池执行长时间任务
        ExecutorService executor = (ExecutorService) req.getServletContext()
            .getAttribute("executor");
        
        executor.execute(() -> {
            try {
                // 模拟长时间任务
                Thread.sleep(5000);
                
                // 生成响应
                ServletResponse response = asyncCtx.getResponse();
                response.setContentType("text/plain");
                PrintWriter out = response.getWriter();
                out.print("Async task completed!");
            } catch (Exception e) {
                // 错误处理
            } finally {
                asyncCtx.complete(); // 完成异步处理
            }
        });
    }
}
```

### 非阻塞I/O

```java
@WebServlet(urlPatterns = "/nio", asyncSupported = true)
public class NonBlockingServlet extends HttpServlet {
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
        throws ServletException, IOException {
        
        AsyncContext asyncCtx = req.startAsync();
        ServletInputStream input = req.getInputStream();
        
        input.setReadListener(new ReadListener() {
            @Override
            public void onDataAvailable() throws IOException {
                // 数据可用时读取
            }
            
            @Override
            public void onAllDataRead() throws IOException {
                // 处理完成
                asyncCtx.complete();
            }
            
            @Override
            public void onError(Throwable t) {
                // 错误处理
                asyncCtx.complete();
            }
        });
    }
}
```

---

## Servlet 安全

### 声明式安全 (web.xml)

```xml
<security-constraint>
    <web-resource-collection>
        <web-resource-name>Admin Area</web-resource-name>
        <url-pattern>/admin/*</url-pattern>
    </web-resource-collection>
    <auth-constraint>
        <role-name>admin</role-name>
    </auth-constraint>
</security-constraint>

<login-config>
    <auth-method>FORM</auth-method>
    <form-login-config>
        <form-login-page>/login.html</form-login-page>
        <form-error-page>/login-error.html</form-error-page>
    </form-login-config>
</login-config>
```

### 编程式安全

```java
protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
    throws ServletException, IOException {
    
    if (req.isUserInRole("admin")) {
        // 显示管理功能
    } else {
        resp.sendError(HttpServletResponse.SC_FORBIDDEN);
    }
    
    // 获取用户信息
    String username = req.getRemoteUser();
    Principal principal = req.getUserPrincipal();
}
```

### 安全最佳实践

1. 使用HTTPS传输敏感数据
2. 实施CSRF保护
3. 对用户输入进行严格验证
4. 使用安全头（如CSP、HSTS）
5. 避免在URL中暴露会话ID

---

## HTTP/2 支持

### 服务器推送

```java
protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
    throws ServletException, IOException {
    
    // 检查是否支持HTTP/2
    if (req.isPushSupported()) {
        // 推送相关资源
        PushBuilder pushBuilder = req.newPushBuilder();
        if (pushBuilder != null) {
            pushBuilder.path("/css/styles.css")
                      .addHeader("content-type", "text/css")
                      .push();
            
            pushBuilder.path("/js/app.js")
                      .addHeader("content-type", "application/javascript")
                      .push();
        }
    }
    
    // 生成主响应
    resp.setContentType("text/html");
    PrintWriter out = resp.getWriter();
    out.println("<html><head>");
    out.println("<link rel=\"stylesheet\" href=\"/css/styles.css\">");
    out.println("</head><body>...</body></html>");
}
```

### HTTP/2 最佳实践

1. 合理使用服务器推送（避免过度推送）
2. 利用多路复用优化资源加载
3. 使用头部压缩减少传输量
4. 使用HPACK编码的Cookie
5. 避免域分片（HTTP/2下不再需要）

---

## Servlet 最佳实践

### 架构设计

1. **遵循单一职责原则**：每个Servlet专注于特定功能
2. **使用前端控制器模式**：统一处理请求分发
3. **避免在Servlet中嵌入业务逻辑**：使用Service层
4. **使用DTO传输数据**：避免直接暴露领域模型

### 性能优化

```java
// 1. 使用连接池管理数据库连接
@WebListener
public class AppContextListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        // 初始化连接池
        DataSource dataSource = createDataSource();
        sce.getServletContext().setAttribute("dataSource", dataSource);
    }
}

// 2. 启用输出压缩
@WebFilter("/*")
public class CompressionFilter implements Filter {
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) 
        throws IOException, ServletException {
        
        HttpServletRequest request = (HttpServletRequest) req;
        HttpServletResponse response = (HttpServletResponse) res;
        
        String acceptEncoding = request.getHeader("Accept-Encoding");
        if (acceptEncoding != null && acceptEncoding.contains("gzip")) {
            response.setHeader("Content-Encoding", "gzip");
            chain.doFilter(request, new GZIPResponseWrapper(response));
        } else {
            chain.doFilter(request, response);
        }
    }
}

// 3. 缓存静态内容
@WebServlet("/static/*")
public class StaticResourceServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) 
        throws ServletException, IOException {
        
        String resourcePath = req.getPathInfo();
        if (resourcePath == null) return;
        
        // 设置缓存头
        resp.setHeader("Cache-Control", "public, max-age=86400"); // 24小时
        resp.setHeader("ETag", generateETagForResource(resourcePath));
        
        // 提供静态资源...
    }
}
```

### 可维护性实践

1. **集中错误处理**：

   ```java
   @WebServlet("/app/*")
   public class MainServlet extends HttpServlet {
       protected void service(HttpServletRequest req, HttpServletResponse resp) 
           throws ServletException, IOException {
           
           try {
               super.service(req, resp);
           } catch (AppException e) {
               req.setAttribute("error", e.getMessage());
               req.getRequestDispatcher("/error").forward(req, resp);
           }
       }
   }
   ```

2. **配置外部化**：

   ```java
   // config.properties 文件
   // max.users=100
   // timeout.ms=5000
   
   Properties config = new Properties();
   try (InputStream input = getServletContext().getResourceAsStream("/WEB-INF/config.properties")) {
       config.load(input);
       int maxUsers = Integer.parseInt(config.getProperty("max.users"));
   } catch (IOException e) {
       // 处理异常
   }
   ```

3. **使用日志框架**：

   ```java
   public class UserServlet extends HttpServlet {
       private static final Logger logger = LoggerFactory.getLogger(UserServlet.class);
       
       protected void doPost(HttpServletRequest req, HttpServletResponse resp) {
           try {
               // 业务逻辑
           } catch (Exception e) {
               logger.error("User creation failed", e);
               resp.sendError(HttpServletResponse.SC_INTERNAL_SERVER_ERROR);
           }
       }
   }
   ```

---

## 常见问题与解决方案

### 1. 内存泄漏问题

**症状**：应用重新部署后出现 PermGen 空间错误
**解决方案**：

- 清理静态集合中的对象
- 使用 `@WebListener` 正确注销监听器
- 避免在 Servlet 中创建线程，使用容器管理的线程池

### 2. 并发问题

**症状**：Servlet 实例变量被多个线程同时修改
**解决方案**：

- 避免使用实例变量（Servlet 是单例）
- 使用线程安全的数据结构
- 使用同步块（谨慎使用）

### 3. 文件上传限制

**问题**：大文件上传导致内存溢出
**解决方案**：

```java
@MultipartConfig(
    location = "/tmp",          // 临时存储目录
    fileSizeThreshold = 1024 * 1024, // 1MB 后写入磁盘
    maxFileSize = 1024 * 1024 * 10, // 10MB 最大文件
    maxRequestSize = 1024 * 1024 * 50 // 50MB 最大请求
)
public class FileUploadServlet extends HttpServlet {
    // ...
}
```

### 4. 中文乱码问题

**解决方案**：

```java
// 统一设置编码
request.setCharacterEncoding("UTF-8");
response.setCharacterEncoding("UTF-8");
response.setContentType("text/html; charset=UTF-8");

// JSP页面添加
<%@ page contentType="text/html;charset=UTF-8" %>
```

### 5. HTTP 方法不支持

**解决方案**：

```java
protected void service(HttpServletRequest req, HttpServletResponse resp) 
    throws ServletException, IOException {
    
    if (!"GET".equals(req.getMethod()) && !"POST".equals(req.getMethod())) {
        resp.sendError(HttpServletResponse.SC_METHOD_NOT_ALLOWED);
        return;
    }
    super.service(req, resp);
}
```

---

## 结语

Servlet 5.0 作为 Java Web 开发的核心技术，提供了强大的功能来处理 HTTP 请求、管理会话、实现安全控制等。通过本文档，您应该已经掌握了 Servlet 的核心概念、高级特性和最佳实践。

**关键要点总结**：

1. 理解 Servlet 生命周期和线程模型
2. 合理使用过滤器和监听器实现横切关注点
3. 利用异步处理提高应用吞吐量
4. 实施全面的安全策略
5. 优化性能（连接池、缓存、压缩）
6. 充分利用 HTTP/2 的新特性

随着 Jakarta EE 的持续发展，Servlet 技术将继续演进，建议关注：

- 与响应式编程的集成
- 更深入的云原生支持
- 增强的安全特性

> 本教程的所有代码示例均已在 Tomcat 10.x（支持 Servlet 5.0）环境中测试通过
