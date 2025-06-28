# SDKMAN 详解与最佳实践

> 一个强大的工具链版本管理工具，让开发环境管理变得简单高效

## 1. 什么是 SDKMAN？

**SDKMAN**（Software Development Kit Manager）是一个命令行工具，用于管理多个软件开发工具包的并行版本。它专为基于 JVM 的环境设计，但已扩展支持多种技术栈。

### 主要特性

- 📦 **多版本管理**：轻松安装、切换和移除 SDK 版本
- 🔄 **跨平台支持**：在 Linux、macOS、WSL 和 Windows 上无缝运行
- 🌐 **丰富的生态系统**：支持 Java、Groovy、Scala、Kotlin 等 30+ SDK
- ⚡ **自动化配置**：自动设置环境变量和路径
- 🔒 **安全可靠**：所有安装包均经过加密校验

## 2. 安装 SDKMAN

```bash
# 基本安装（类Unix系统）
curl -s "https://get.sdkman.io" | bash

# 安装完成后初始化
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 验证安装
sdk version
```

**Windows 用户**：

1. 安装 [Windows Subsystem for Linux (WSL)](https://learn.microsoft.com/en-us/windows/wsl/install)
2. 在 WSL 环境中执行上述 Unix 安装命令

## 3. 核心命令详解

### 3.1 列出可用 SDK

```bash
# 列出所有可用候选 SDK
sdk list

# 过滤特定 SDK（如 Java）
sdk list java
```

### 3.2 安装 SDK

```bash
# 安装最新稳定版 Java
sdk install java

# 安装特定版本
sdk install java 17.0.3-oracle

# 安装本地版本（离线安装）
sdk install scala 2.13.6 /path/to/scala-2.13.6/
```

### 3.3 版本管理

```bash
# 查看当前使用的版本
sdk current java

# 切换版本
sdk use java 11.0.15-zulu

# 设置默认版本
sdk default java 17.0.3-oracle

# 查看已安装版本
sdk list java | grep installed
```

### 3.4 卸载与升级

```bash
# 卸载特定版本
sdk uninstall java 11.0.15-zulu

# 升级 SDK 版本
sdk upgrade java

# 升级 SDKMAN 自身
sdk selfupdate
```

## 4. 支持的主要技术栈

| 技术        | 命令示例                     | 说明                      |
|-------------|-----------------------------|--------------------------|
| Java        | `sdk install java`          | 支持多种发行版 (Zulu, OpenJDK) |
| Maven       | `sdk install maven`         | 管理 Maven 构建工具        |
| Gradle      | `sdk install gradle`        | 管理 Gradle 构建工具       |
| Spring Boot | `sdk install springboot`    | Spring Boot CLI           |
| Kotlin      | `sdk install kotlin`        | Kotlin 语言支持           |
| Scala       | `sdk install scala`         | Scala 语言支持            |
| SBT         | `sdk install sbt`           | Scala 构建工具            |
| Micronaut   | `sdk install micronaut`     | Micronaut 框架 CLI        |
| Flink       | `sdk install flink`         | Apache Flink             |

## 5. 高级配置技巧

### 5.1 自定义安装路径

```bash
# 设置自定义 SDK 安装目录
export SDKMAN_DIR="/opt/sdkman"
curl -s "https://get.sdkman.io" | bash
```

### 5.2 代理配置

```bash
# 设置 HTTP 代理
export SDKMAN_HTTP_PROXY=http://proxy.example.com:8080
export SDKMAN_HTTPS_PROXY=http://proxy.example.com:8080

# 安装时自动配置
sdk config
# 然后在配置文件中设置：
# proxy=proxy.example.com
# proxy_port=8080
```

### 5.3 离线模式

```bash
# 启用离线模式（禁止网络访问）
sdk offline enable

# 禁用离线模式
sdk offline disable
```

### 5.4 自定义版本别名

```bash
# 创建自定义别名
sdk alias create jdk17 17.0.3-oracle

# 使用别名
sdk use jdk17

# 查看所有别名
sdk alias
```

## 6. 最佳实践指南

### 6.1 项目级版本管理

创建 `.sdkmanrc` 文件自动切换版本：

```bash
# 生成 .sdkmanrc 文件
sdk env init

# 示例 .sdkmanrc 内容
# Enable environment auto-configuration
# This file must be used with sdk env
auto_env=true
# Configure Java version
java=17.0.3-oracle
# Configure Maven version
maven=3.8.6
```

**自动激活环境**：

```bash
# 在项目目录自动切换版本
sdk env

# 自动启用（添加到 shell 配置文件）
echo 'sdk env' >> ~/.bashrc
```

### 6.2 CI/CD 集成

```bash
# 在 CI 脚本中安装 SDKMAN
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"

# 安装所需版本
sdk install java 17.0.3-oracle
sdk install maven 3.8.6

# 使用 .sdkmanrc 配置
sdk env
```

### 6.3 多版本并行开发

```bash
# 为不同项目设置不同默认版本
# 项目A使用Java 11
cd ~/projects/projectA
sdk default java 11.0.15-zulu

# 项目B使用Java 17
cd ~/projects/projectB
sdk default java 17.0.3-oracle
```

### 6.4 安全实践

```bash
# 验证安装包的完整性（自动启用）
sdk install java 17.0.3-oracle

# 手动验证候选版本
sdk list java | grep -A 5 17.0.3-oracle
# 输出应包含校验和信息
```

## 7. 常见问题解决

### 7.1 版本冲突问题

**症状**：`command not found` 或执行错误版本  
**解决方案**：

```bash
# 查看当前激活版本
sdk current

# 重置当前会话
sdk reset

# 显式指定版本
sdk use java 17.0.3-oracle
```

### 7.2 安装速度慢

**优化方法**：

```bash
# 1. 更换下载镜像
export SDKMAN_CANDIDATES_API="https://china-mirror.sdkman.io/candidates"

# 2. 使用离线安装模式
sdk offline enable
sdk install java 17.0.3-oracle --no-verbose
sdk offline disable
```

### 7.3 环境变量问题

```bash
# 修复环境变量配置
sdk home java 17.0.3-oracle

# 检查 JAVA_HOME 设置
export JAVA_HOME=$(sdk home java current)
echo $JAVA_HOME
```

## 8. 性能优化技巧

1. **精简安装**：只安装必需版本，定期清理旧版本

   ```bash
   # 删除30天未使用的版本
   sdk prune
   ```

2. **禁用自动更新**：

   ```bash
   sdk config
   # 设置:
   sdkman_auto_answer=false
   sdkman_auto_selfupdate=false
   ```

3. **使用内存缓存**：

   ```bash
   # 在 .bashrc 或 .zshrc 中添加
   export SDKMAN_DIR="/dev/shm/sdkman"
   ```

## 9. 生态系统集成

### 9.1 IDE 集成

**IntelliJ IDEA**：

1. 打开项目设置
2. 选择 SDKMAN 安装的 JDK 路径：
   `~/.sdkman/candidates/java/current`

**VS Code**：

```json
// .vscode/settings.json
{
  "java.home": "/home/user/.sdkman/candidates/java/current",
  "java.configuration.runtimes": [
    {
      "name": "JavaSE-17",
      "path": "/home/user/.sdkman/candidates/java/17.0.3-oracle"
    }
  ]
}
```

### 9.2 Docker 集成

```dockerfile
FROM ubuntu:22.04

# 安装 SDKMAN
RUN apt-get update && apt-get install -y curl zip unzip
RUN curl -s "https://get.sdkman.io" | bash
RUN bash -c "source /root/.sdkman/bin/sdkman-init.sh && \
             sdk install java 17.0.3-oracle && \
             sdk install maven 3.8.6"

# 设置环境变量
ENV JAVA_HOME="/root/.sdkman/candidates/java/current"
ENV PATH="${JAVA_HOME}/bin:${PATH}:/root/.sdkman/candidates/maven/current/bin"
```

## 10. 替代方案比较

| 工具         | 优点                      | 局限性                 |
|--------------|--------------------------|-----------------------|
| **SDKMAN**   | 多语言支持，简单易用      | 主要面向 JVM 生态系统 |
| asdf         | 插件架构，支持任意工具    | 配置复杂              |
| Jabba        | 专注 Java，兼容 SDKMAN    | 生态系统较小          |
| Homebrew     | macOS 生态完善            | Linux 支持有限        |
| apt/yum/dnf  | 系统级集成                | 版本更新滞后          |

## 结论

SDKMAN 是现代开发环境中不可或缺的工具，特别适合需要管理多版本 Java 和其他 JVM 语言的开发者。通过遵循本文的最佳实践：

✅ 实现项目级别的精确版本控制  
✅ 简化团队协作和环境复现  
✅ 提高开发效率和环境一致性  
✅ 降低不同项目间的版本冲突风险  

**开始使用**：

```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java
```

> 保持更新：定期执行 `sdk update` 获取最新功能和候选版本更新
