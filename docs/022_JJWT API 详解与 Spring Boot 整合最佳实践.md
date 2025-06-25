# JJWT API 详解与 Spring Boot 整合最佳实践

## 1. JJWT 简介

### 1.1 什么是 JJWT?

JJWT (Java JSON Web Token) 是一个用于创建、解析和验证 **JSON Web Tokens (JWT)** 的 Java 库。它提供简单易用的 API，帮助开发者在 Java 应用中实现**安全认证**和**授权**机制，是构建现代认证系统的核心工具。

### 1.2 JWT 核心优势

| 特性 | 描述 | 优势 |
|------|------|------|
| **无状态认证** | 服务端无需存储会话信息 | ⚡ 高扩展性 |
| **跨域支持** | 轻松实现单点登录(SSO) | 🌐 跨平台兼容 |
| **自包含数据** | 负载(Payload)可携带用户信息 | 📦 减少数据库查询 |
| **数字签名** | 使用密钥验证令牌完整性 | 🔒 防止篡改 |
| **标准化格式** | 遵循 RFC 7519 标准 | 📜 广泛支持 |

## 2. JWT 结构解析

### 2.1 JWT 组成结构

```plain
[Header].[Payload].[Signature]
```

#### Header 示例

```json
{
  "alg": "HS256",  // 签名算法
  "typ": "JWT"     // 令牌类型
}
```

#### Payload 示例

```json
{
  "sub": "1234567890",      // 主题 (Subject)
  "name": "John Doe",       // 自定义声明
  "iat": 1516239022,        // 签发时间 (Issued At)
  "exp": 1516239022 + 3600  // 过期时间 (Expiration Time)
}
```

#### Signature 生成公式

```plain
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret
)
```

### 2.2 标准声明字段

| 字段 | 名称 | 描述 |
|------|------|------|
| `iss` | Issuer | 签发者 |
| `sub` | Subject | 主题 |
| `aud` | Audience | 接收方 |
| `exp` | Expiration Time | 过期时间 |
| `nbf` | Not Before | 生效时间 |
| `iat` | Issued At | 签发时间 |
| `jti` | JWT ID | 唯一标识 |

## 3. JJWT API 详解

### 3.1 添加依赖

```xml
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
    <scope>runtime</scope>
</dependency>
```

### 3.2 核心 API 使用

#### 创建 JWT

```java
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import java.security.Key;
import java.util.Date;

public class JwtUtil {
    
    private static final Key SECRET_KEY = Keys.secretKeyFor(SignatureAlgorithm.HS256);
    private static final long EXPIRATION_MS = 3600000; // 1小时

    public static String generateToken(String username, List<String> roles) {
        return Jwts.builder()
                .setSubject(username)
                .claim("roles", roles) // 自定义声明
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_MS))
                .signWith(SECRET_KEY)
                .compact();
    }
}
```

#### 解析验证 JWT

```java
public static Claims parseToken(String token) {
    return Jwts.parserBuilder()
            .setSigningKey(SECRET_KEY)
            .build()
            .parseClaimsJws(token)
            .getBody();
}

public static boolean validateToken(String token) {
    try {
        Jwts.parserBuilder()
            .setSigningKey(SECRET_KEY)
            .build()
            .parseClaimsJws(token);
        return true;
    } catch (JwtException | IllegalArgumentException e) {
        log.error("JWT validation failed: {}", e.getMessage());
        return false;
    }
}
```

### 3.3 高级功能

#### 自定义声明

```java
Jwts.builder()
    .claim("email", "user@example.com")
    .claim("department", "Engineering")
    .addClaims(Map.of(
        "isAdmin", true,
        "userId", 12345
    ))
    // ...其他配置
```

#### 刷新令牌

```java
public String refreshToken(String token) {
    Claims claims = parseToken(token);
    return Jwts.builder()
            .setClaims(claims)
            .setIssuedAt(new Date())
            .setExpiration(new Date(System.currentTimeMillis() + EXPIRATION_MS))
            .signWith(SECRET_KEY)
            .compact();
}
```

#### RSA 非对称加密

```java
// 生成密钥对
KeyPair keyPair = Keys.keyPairFor(SignatureAlgorithm.RS256);

// 创建令牌
String token = Jwts.builder()
        .setSubject("user")
        .signWith(keyPair.getPrivate())
        .compact();

// 验证令牌
Claims claims = Jwts.parserBuilder()
        .setSigningKey(keyPair.getPublic())
        .build()
        .parseClaimsJws(token)
        .getBody();
```

## 4. Spring Boot 整合实践

### 4.1 安全配置类

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private JwtTokenFilter jwtTokenFilter;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
                .antMatchers("/api/auth/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .addFilterBefore(jwtTokenFilter, UsernamePasswordAuthenticationFilter.class);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 4.2 JWT 过滤器

```java
@Component
public class JwtTokenFilter extends OncePerRequestFilter {

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                    HttpServletResponse response, 
                                    FilterChain chain)
            throws ServletException, IOException {
        
        // 1. 从请求头获取令牌
        String token = resolveToken(request);
        
        if (token != null && jwtTokenUtil.validateToken(token)) {
            // 2. 解析令牌获取认证信息
            Authentication authentication = jwtTokenUtil.getAuthentication(token);
            
            // 3. 设置安全上下文
            SecurityContextHolder.getContext().setAuthentication(authentication);
        }
        
        chain.doFilter(request, response);
    }

    private String resolveToken(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

### 4.3 JWT 工具类

```java
@Component
public class JwtTokenUtil {

    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.expiration}")
    private Long expiration;
    
    private Key getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secret));
    }

    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList()));
        
        return Jwts.builder()
                .setClaims(claims)
                .setSubject(userDetails.getUsername())
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getSigningKey())
                .compact();
    }

    public Authentication getAuthentication(String token) {
        Claims claims = extractAllClaims(token);
        String username = claims.getSubject();
        List<SimpleGrantedAuthority> authorities = ((List<?>) claims.get("roles")).stream()
                .map(role -> new SimpleGrantedAuthority(role.toString()))
                .collect(Collectors.toList());
        
        return new UsernamePasswordAuthenticationToken(username, null, authorities);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
    
    // 其他辅助方法...
}
```

### 4.4 认证控制器

```java
@RestController
@RequestMapping("/api/auth")
public class AuthController {

    @Autowired
    private AuthenticationManager authenticationManager;
    
    @Autowired
    private JwtTokenUtil jwtTokenUtil;
    
    @Autowired
    private UserDetailsService userDetailsService;

    @PostMapping("/login")
    public ResponseEntity<JwtResponse> login(@RequestBody LoginRequest request) {
        // 1. 认证用户名密码
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsername(), 
                request.getPassword()
            )
        );
        
        SecurityContextHolder.getContext().setAuthentication(authentication);
        
        // 2. 生成JWT令牌
        UserDetails userDetails = (UserDetails) authentication.getPrincipal();
        String token = jwtTokenUtil.generateToken(userDetails);
        
        // 3. 返回响应
        return ResponseEntity.ok(new JwtResponse(
            token,
            jwtTokenUtil.getExpirationFromToken(token),
            userDetails.getUsername(),
            userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList())
        ));
    }
    
    @PostMapping("/refresh")
    public ResponseEntity<JwtResponse> refreshToken(HttpServletRequest request) {
        String token = jwtTokenUtil.resolveToken(request);
        if (token != null && jwtTokenUtil.canTokenBeRefreshed(token)) {
            String refreshedToken = jwtTokenUtil.refreshToken(token);
            return ResponseEntity.ok(new JwtResponse(
                refreshedToken,
                jwtTokenUtil.getExpirationFromToken(refreshedToken),
                jwtTokenUtil.getUsernameFromToken(refreshedToken),
                jwtTokenUtil.getRolesFromToken(refreshedToken)
            );
        }
        throw new BadCredentialsException("Invalid refresh token");
    }
}
```

## 5. 安全最佳实践

### 5.1 密钥管理策略

```java
// 推荐做法：使用BASE64编码的安全密钥
@Value("${jwt.secret}")
private String base64Secret;

private Key getSigningKey() {
    byte[] keyBytes = Decoders.BASE64.decode(base64Secret);
    return Keys.hmacShaKeyFor(keyBytes);
}

// 生成密钥命令（Linux）：
// openssl rand -base64 64 | tr -d '\n' > jwt-secret.txt
```

### 5.2 令牌安全配置

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| 过期时间 | 15-30分钟 | 降低泄露风险 |
| 刷新令牌过期 | 7天 | 平衡用户体验与安全 |
| 签名算法 | HS256/RS256 | 避免使用none算法 |
| 密钥长度 | ≥256位 | 防止暴力破解 |
| 存储方式 | HttpOnly Cookie | 防止XSS攻击 |

### 5.3 防御常见攻击

**1. 令牌劫持防护**

```java
// 在Payload中添加客户端指纹
public String generateToken(UserDetails userDetails, String clientFingerprint) {
    return Jwts.builder()
            .claim("fp", clientFingerprint) // 客户端指纹
            // ...其他声明
}

// 验证时检查指纹
public boolean validateToken(String token, String clientFingerprint) {
    Claims claims = parseToken(token);
    String tokenFp = claims.get("fp", String.class);
    return tokenFp != null && tokenFp.equals(clientFingerprint);
}
```

**2. 令牌注销机制**

```java
// 短有效期令牌 + 令牌黑名单
@Service
public class TokenBlacklistService {
    
    private final Set<String> blacklist = Collections.newSetFromMap(
        new ConcurrentHashMap<>());
    
    public void invalidateToken(String token) {
        blacklist.add(token);
    }
    
    public boolean isTokenBlacklisted(String token) {
        return blacklist.contains(token);
    }
}

// 在过滤器中检查黑名单
if (tokenBlacklistService.isTokenBlacklisted(token)) {
    throw new JwtException("Token has been invalidated");
}
```

### 5.4 性能优化

```java
// 缓存公钥（RSA算法）
private static PublicKey publicKey;

@PostConstruct
public void init() {
    // 初始化时加载公钥
    publicKey = loadPublicKey();
}

// 使用缓存公钥验证
public Claims parseToken(String token) {
    return Jwts.parserBuilder()
            .setSigningKey(publicKey) // 使用缓存
            .build()
            .parseClaimsJws(token)
            .getBody();
}
```

## 6. Spring Boot 整合进阶

### 6.1 多环境配置

```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET:defaultBase64Secret} # 环境变量优先
  expiration: 900000 # 15分钟
  
# application-prod.yml
jwt:
  expiration: 1800000 # 30分钟
  issuer: prod-api.example.com
```

### 6.2 响应封装

```java
public class JwtResponse {
    private String token;
    private String type = "Bearer";
    private Date expiresAt;
    private String username;
    private List<String> roles;
    
    // 构造器、getters...
}

// 统一错误响应
public class AuthErrorResponse {
    private int status;
    private String message;
    private long timestamp = System.currentTimeMillis();
}
```

### 6.3 测试端点

```java
@SpringBootTest
@AutoConfigureMockMvc
public class AuthControllerTest {

    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    public void testSuccessfulAuthentication() throws Exception {
        // 准备测试用户
        User user = new User("testuser", passwordEncoder.encode("password"), "ROLE_USER");
        userRepository.save(user);
        
        // 执行登录请求
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"username\":\"testuser\",\"password\":\"password\"}"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.token").exists())
                .andExpect(jsonPath("$.username").value("testuser"));
    }
}
```

## 7. 生产环境最佳实践

### 7.1 部署配置清单

```yaml
# 推荐生产环境配置
jwt:
  secret: ${JWT_SECRET} # 必须从环境变量注入
  expiration: 1800000    # 30分钟
  refresh-expiration: 604800000 # 7天
  issuer: api.example.com
  audience: web-app
  algorithm: RS256       # 使用非对称加密
```

### 7.2 监控与日志

```java
// 添加JWT相关指标
@Bean
public MeterRegistryCustomizer<MeterRegistry> jwtMetrics() {
    return registry -> {
        Counter.builder("jwt.tokens.issued")
            .description("Total issued JWT tokens")
            .register(registry);
        
        Counter.builder("jwt.tokens.validated")
            .description("Total validated JWT tokens")
            .register(registry);
        
        Counter.builder("jwt.tokens.invalid")
            .description("Total invalid JWT tokens")
            .tag("reason", "expired") // expired|signature|malformed
            .register(registry);
    };
}

// 在TokenUtil中记录指标
public boolean validateToken(String token) {
    try {
        // ...验证逻辑
        Metrics.counter("jwt.tokens.validated").increment();
        return true;
    } catch (ExpiredJwtException ex) {
        Metrics.counter("jwt.tokens.invalid", "reason", "expired").increment();
        log.warn("JWT expired: {}", ex.getMessage());
    } catch (JwtException | IllegalArgumentException ex) {
        Metrics.counter("jwt.tokens.invalid", "reason", "invalid").increment();
        log.warn("Invalid JWT: {}", ex.getMessage());
    }
    return false;
}
```

### 7.3 密钥轮换策略

```java
public class KeyRotationService {
    
    private final Map<String, Key> keys = new ConcurrentHashMap<>();
    private String currentKeyId;
    
    @PostConstruct
    public void init() {
        // 初始化当前密钥
        currentKeyId = "key-" + System.currentTimeMillis();
        keys.put(currentKeyId, generateNewKey());
    }
    
    public void rotateKey() {
        String newKeyId = "key-" + System.currentTimeMillis();
        keys.put(newKeyId, generateNewKey());
        currentKeyId = newKeyId;
        // 保留旧密钥一段时间用于令牌验证
    }
    
    public Key getCurrentKey() {
        return keys.get(currentKeyId);
    }
    
    public Key getKey(String keyId) {
        return keys.get(keyId);
    }
    
    private Key generateNewKey() {
        return Keys.secretKeyFor(SignatureAlgorithm.HS256);
    }
}

// 在创建令牌时添加keyId
public String generateToken(UserDetails userDetails) {
    return Jwts.builder()
            .setHeaderParam("kid", keyRotationService.getCurrentKeyId())
            // ...其他声明
            .signWith(keyRotationService.getCurrentKey())
            .compact();
}

// 在验证时使用kid查找密钥
public Claims parseToken(String token) {
    String keyId = getKeyIdFromToken(token); // 从头部分析kid
    Key key = keyRotationService.getKey(keyId);
    
    return Jwts.parserBuilder()
            .setSigningKey(key)
            .build()
            .parseClaimsJws(token)
            .getBody();
}
```

## 8. 常见问题与解决方案

### 8.1 令牌过期处理

**前端方案**：

```javascript
// 使用Axios拦截器自动刷新令牌
axios.interceptors.response.use(response => response, error => {
  if (error.response.status === 401) {
    return refreshToken().then(() => {
      return axios(error.config);
    });
  }
  return Promise.reject(error);
});
```

**后端方案**：

```java
// 提供刷新令牌端点
@PostMapping("/refresh")
public ResponseEntity<?> refreshToken(HttpServletRequest request) {
    String refreshToken = jwtTokenUtil.resolveToken(request);
    if (jwtTokenUtil.validateRefreshToken(refreshToken)) {
        String newToken = jwtTokenUtil.refreshAccessToken(refreshToken);
        return ResponseEntity.ok(new JwtResponse(newToken));
    }
    throw new AccessDeniedException("Invalid refresh token");
}
```

### 8.2 跨域问题

```java
// 安全配置中添加CORS
@Bean
public CorsFilter corsFilter() {
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowCredentials(true);
    config.addAllowedOrigin("https://yourdomain.com");
    config.addAllowedHeader("*");
    config.addAllowedMethod("*");
    source.registerCorsConfiguration("/**", config);
    return new CorsFilter(source);
}
```

### 8.3 性能问题

**优化建议**：

1. 使用非对称加密（RS256）减少密钥分发开销
2. 缓存公钥验证结果
3. 避免在令牌中存储过多数据
4. 使用高效的JSON处理器（如Jackson）
5. 启用HTTP/2减少连接开销

## 9. 最佳实践总结

### 9.1 核心原则

1. **最小权限原则**：令牌只包含必要声明
2. **短时效令牌**：访问令牌≤30分钟
3. **安全传输**：始终使用HTTPS
4. **密钥保护**：使用强密钥并定期轮换
5. **深度防御**：结合其他安全措施（CORS、CSRF防护）

### 9.2 实施清单

- [ ] 使用强密钥（≥256位）
- [ ] 设置合理的令牌过期时间
- [ ] 实现令牌刷新机制
- [ ] 添加客户端指纹验证
- [ ] 实施令牌黑名单
- [ ] 配置非对称加密（生产环境）
- [ ] 集成监控和日志
- [ ] 建立密钥轮换策略

> **安全专家提示**：定期进行安全审计，包括：
>
> 1. 检查JWT实现是否符合最新安全标准
> 2. 验证密钥存储和轮换策略
> 3. 模拟令牌劫持和重放攻击
> 4. 审查令牌声明是否泄露敏感信息

## 附录：JWT 工具类完整示例

```java
@Component
public class JwtTokenUtil {
    private static final Logger log = LoggerFactory.getLogger(JwtTokenUtil.class);
    
    @Value("${jwt.secret}")
    private String secret;
    
    @Value("${jwt.access-token-expiration}")
    private Long accessTokenExpiration;
    
    @Value("${jwt.refresh-token-expiration}")
    private Long refreshTokenExpiration;
    
    private Key getAccessTokenSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(secret);
        return Keys.hmacShaKeyFor(keyBytes);
    }
    
    public String generateAccessToken(UserDetails userDetails) {
        return buildToken(userDetails, accessTokenExpiration);
    }
    
    public String generateRefreshToken(UserDetails userDetails) {
        return buildToken(userDetails, refreshTokenExpiration);
    }
    
    private String buildToken(UserDetails userDetails, long expiration) {
        List<String> roles = userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList());
        
        return Jwts.builder()
                .setSubject(userDetails.getUsername())
                .claim("roles", roles)
                .setIssuedAt(new Date())
                .setExpiration(new Date(System.currentTimeMillis() + expiration))
                .signWith(getAccessTokenSigningKey())
                .compact();
    }
    
    public boolean validateToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(getAccessTokenSigningKey())
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (MalformedJwtException ex) {
            log.error("Invalid JWT token: {}", ex.getMessage());
        } catch (ExpiredJwtException ex) {
            log.error("JWT token is expired: {}", ex.getMessage());
        } catch (UnsupportedJwtException ex) {
            log.error("JWT token is unsupported: {}", ex.getMessage());
        } catch (IllegalArgumentException ex) {
            log.error("JWT claims string is empty: {}", ex.getMessage());
        }
        return false;
    }
    
    public String getUsernameFromToken(String token) {
        return extractClaim(token, Claims::getSubject);
    }
    
    public List<String> getRolesFromToken(String token) {
        return extractClaim(token, claims -> claims.get("roles", List.class));
    }
    
    public Date getExpirationFromToken(String token) {
        return extractClaim(token, Claims::getExpiration);
    }
    
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }
    
    private Claims extractAllClaims(String token) {
        return Jwts.parserBuilder()
                .setSigningKey(getAccessTokenSigningKey())
                .build()
                .parseClaimsJws(token)
                .getBody();
    }
    
    public boolean canTokenBeRefreshed(String token) {
        return !isTokenExpired(token);
    }
    
    public String refreshToken(String token) {
        Claims claims = extractAllClaims(token);
        claims.setIssuedAt(new Date());
        claims.setExpiration(new Date(System.currentTimeMillis() + accessTokenExpiration));
        
        return Jwts.builder()
                .setClaims(claims)
                .signWith(getAccessTokenSigningKey())
                .compact();
    }
    
    private boolean isTokenExpired(String token) {
        final Date expiration = getExpirationFromToken(token);
        return expiration.before(new Date());
    }
}
```
