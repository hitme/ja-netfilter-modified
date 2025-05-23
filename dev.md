# ja-netfilter 调试指南

本篇文档旨在为开发者提供关于如何调试 **ja-netfilter** 的详细步骤和方法。由于 ja-netfilter 是一个基于 Java Agent 技术的工具，主要用于拦截、修改 JetBrains IDE 的激活验证逻辑，因此调试其行为需要结合 Java Agent、网络拦截、日志分析等技术。

---

## 目录

- [1. 环境准备](#1-环境准备)
- [2. 日志调试](#2-日志调试)
- [3. Java Agent 启动参数调试](#3-java-agent-启动参数调试)
- [4. 源码编译与本地运行](#4-源码编译与本地运行)
- [4.1. 远程调试](#41-远程调试)
- [5. 插件模块调试（dns.conf / power.conf）](#5-插件模块调试-dnsconf--powerconf)
- [6. 网络请求拦截调试](#6-网络请求拦截调试)
- [7. 异常处理与反混淆](#7-异常处理与反混淆)

---

## 1. 环境准备

### 必要工具

- JDK 8+（推荐使用 OpenJDK）
- 反编译工具（如 JD-GUI、CFR、Bytecode Viewer）
- IDE（如 IntelliJ IDEA、Eclipse）
- 网络抓包工具（Wireshark、Charles、Fiddler）
- Linux/macOS Shell 或 Windows PowerShell

### 获取源码

```bash
git clone https://github.com/v-JiangNan/ja-netfilter-modified.git
cd ja-netfilter-modified
```


---

## 2. 日志调试

ja-netfilter 支持通过 `DebugInfo` 类输出日志信息：

### 修改日志级别

编辑 [config-jetbrains/power.conf](file:///Users/markus/dev/code/github/ja-netfilter-modified/config-jetbrains/power.conf) 或其他配置文件中日志相关字段，或在代码中查找如下类进行调整：

```java
com.janetfilter.core.commons.DebugInfo
```


### 示例：开启 DEBUG 输出

```java
DebugInfo.setLevel(DebugInfo.Level.DEBUG); // 设置日志级别
```


### 查看日志输出

在 IDE 启动时，日志会输出到控制台或文件中，具体路径取决于你的 IDE 配置。

---

## 3. Java Agent 启动参数调试

ja-netfilter 是以 Java Agent 形式加载的，通过 `-javaagent:/path/to/ja-netfilter.jar=jetbrains` 参数注入 JVM。

### 查看 agent 加载情况

在 IDE 启动脚本中查看是否正确添加了 agent 参数，例如在以下文件中：

```bash
vmoptions/idea.vmoptions
```


确保包含类似如下内容：

```
-javaagent:/path/to/ja-netfilter.jar=jetbrains
```


### 手动测试 agent

可以手动运行一个简单的 Java 程序来测试 agent 是否正常加载：

```bash
java -javaagent:ja-netfilter.jar=jetbrains -jar testapp.jar
```


---

## 4. 源码编译与本地运行

### 编译项目

确保你已导入 ja-netfilter 源码到 IDE 中，并安装依赖库。

```bash
mvn clean package
```


> 注意：如果项目未提供 `pom.xml`，可能需要手动创建 Maven 项目结构并引入依赖。

### 替换 ja-netfilter.jar

将新编译的 [ja-netfilter.jar](file:///Users/markus/dev/code/github/ja-netfilter-modified/ja-netfilter.jar) 替换到 `plugins-jetbrains/` 和 `scripts/` 目录下，然后重新执行安装脚本：

```bash
./scripts/install.sh
```
## 4.1. 远程调试
1. Edit Custom VM Options:
修改 IDEA 设置，添加如下内容：其中suspend=y表示启动后挂起，等待 debugging 进程 attach。
```declarative
-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005
```
2. Run IDEA with Remote Debugging:
在 IDEA 中，选择 Debug 按钮，选择 Remote 按钮，输入端口号 5005，点击 Connect 按钮。
3. 在 IDEA 中，直接把 jar 包添加到 classpath 中。
4. 设置端点、启动调试
---

## 5. 插件模块调试 (dns.conf / power.conf)

这些配置文件用于定义域名解析策略和激活码验证规则。

### dns.conf

```ini
# 屏蔽 JetBrains 的激活验证服务器
0.0.0.0 account.jetbrains.com
0.0.0.0 www.jetbrains.com
```


### power.conf

```ini
# 自定义激活码白名单
valid-code-12345
another-valid-code
```


### 调试方式

- 使用 Wireshark/Fiddler 抓包，确认 DNS 请求是否被重定向。
- 在 `com.janetfilter.core.rulers.KeywordRuler` 类中设置断点，观察匹配逻辑。

---

## 6. 网络请求拦截调试

JetBrains IDE 会在启动时访问远程服务器验证激活码，ja-netfilter 通过 DNS 重定向和 HTTP 响应伪造来绕过验证。

### 抓包验证

1. 启动 Wireshark/Fiddler；
2. 打开 IDE；
3. 观察是否有对 `account.jetbrains.com` 的请求；
4. 确认响应是否被本地代理伪造。

### 修改响应内容

在 `com.janetfilter.core.plugin.HttpResponseHook` 类中，可找到伪造响应的逻辑，可用于进一步定制。

---

## 7. 异常处理与反混淆

部分类如 `Initializer.class`、`PluginClassLoader.class` 等可能经过混淆处理，调试时需使用反编译工具辅助理解代码逻辑。

### 推荐反编译工具

- CFR（命令行）
- Bytecode Viewer（GUI）
- JD-GUI（GUI）

### 处理异常

- 如果出现 `ClassNotFoundException`，检查 class 文件路径是否正确；
- 如果出现 `NoClassDefFoundError`，检查依赖是否完整；
- 如果出现 `VerifyError`，可能是字节码被篡改或 agent 注入失败。

---

## 附录

### 主要类结构说明

| 类名 | 功能 |
|------|------|
| `com.janetfilter.core.Initializer` | agent 入口，负责初始化插件系统 |
| `com.janetfilter.core.commons.DebugInfo` | 日志输出管理 |
| `com.janetfilter.core.rulers.KeywordRuler` | 匹配 URL/DNS 的关键字规则 |
| `com.janetfilter.core.plugin.HttpResponseHook` | 拦截 HTTP 响应，伪造返回内容 |

### 官方资料参考

- [ja-netfilter 官方介绍](https://zhile.io/2021/11/29/ja-netfilter-javaagent-lib.html)
- [GitHub 项目地址](https://github.com/v-JiangNan/ja-netfilter-modified)

---

> ⚠️ 注意：本文档仅供技术研究交流使用，请遵守相关法律法规，尊重正版软件版权。