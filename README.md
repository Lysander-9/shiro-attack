<h1 align="center">ShiroAttack2 Bypass</h1>
<h3 align="center">一款针对 Shiro550 反序列化漏洞的快速利用工具</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Java-8-orange" alt="Java">
  <img src="https://img.shields.io/badge/JavaFX-GUI-blue" alt="JavaFX">
  <img src="https://img.shields.io/badge/Shiro-1.2.x-9cf" alt="Shiro">
</p>

> 站在巨人的肩膀上，基于 [ShiroAttack2](https://github.com/SummerSec/ShiroAttack2) 二次开发。
> 针对 **Header 长度限制 / WAF 拦截 / 目标环境不确定** 等真实场景，增强了一键化的探测与利用能力。

---

## ✨ 功能特性

### 🧩 CommonsBeanutils 多版本支持（1.8.3 / 1.9.2）
- **修复了原版 1.8.3 支持失效的问题**：原 `_183` 系列用 javassist 改 `serialVersionUID`，但序列化后 SUID 仍为 1.9.x，目标 1.8.3 环境反序列化直接抛 `InvalidClassException`。
- 新增 `deser/util/BeanUtils183.java`，通过 `StandardExecutorClassLoader` 加载**真实的 `lib/1.8.3/commons-beanutils-1.8.3.jar`**，让 `BeanComparator` 以 1.8.3 的结构序列化，SUID 天然匹配。
- 用双参构造器 `BeanComparator(null, String.CASE_INSENSITIVE_ORDER)` 构造实例，**不引用 commons-collections**，适配无 commons-collections 的目标。
- Header 绕过窗口提供「版本(1.8.3/1.9.2) + 利用链(CommonsBeanutils1/String)」选择器。

### 🛡️ Header 绕过（WAF Bypass）
- 在 `rememberMe` Cookie 中插入非 Base64 字符（`$`），绕过 WAF 的 Base64 解码检测——Shiro 的 `discardNonBase64` 机制会自动忽略这些字符。
- 支持混淆开关、最大 Cookie 长度限制配置。

### 🔍 环境自动探测（新增）
- **探测 Cookie 限长**：二分法探测目标 `maxHttpHeaderSize` 对 Cookie 的约束，自动回填「Cookie限长」输入框。
- **探测容器环境**：通过 `Class.forName` + `Thread.sleep` 时间差，自动判定目标是否 Tomcat / Spring。

### 🕐 自动化侧信道盲注（新增）
- 当 header 限长过小、既无法回显也无法注入大 payload 时，用 **时间侧信道** 逐字符泄露目标内容。
- 支持 **文件读取** 与 **命令执行** 两种模式（默认读 `/flag`，或执行 `cat /flag`、`id` 等）。
- 参数全可调：
  - **延迟倍率**（`sleep(字符值 × N ms)`，越大越准越慢，支持**自适应**——按网络抖动自动计算）；
  - **中位数次数**（每字符测 N 次取中位数，越大越抗网络抖动）。
- **实时逐字符输出**：单趟循环，每读到一个字符立即打印累积结果，长文件也能实时确认读取进度，读到结尾自动停止。
- ⚠️ **读取范围限制**：为把载荷压进 header 限长，读取用的是 `Scanner.next()`，**只读首个 token（遇空白/换行即截断）**。读无空格的 flag、或 `id`/`whoami` 这类首段输出没问题；若要读含空格/多行内容，请用「命令执行」配合 `cat /flag | base64` 之类把内容压成单 token，或改用回显/内存马。

### 📏 Header 预算优化（限长靶场关键）
- **精简 hutool 默认请求头**：请求前把 `User-Agent`/`Accept`/`Accept-Encoding` 等默认头替换为短值，把约 200+ 字节的 header 预算让给 Cookie，有效 Cookie 上限从 ~1660 提升到 ~1900。
- **压缩序列化体积**：`_183` 系列 gadget 构造后置空 `BeanComparator.comparator` 字段（`property` 已设为 `outputProperties`，该字段不再使用），少序列化一个 JDK 内部比较器，载荷缩短约 58 字节。

### 💥 命令执行 / 内存马
- **盲执行 / 回显执行**（支持 Spring / Tomcat 回显）。
- **分块内存马注入**：大体积内存马 Base64 分块，通过系统属性 / 线程名 / 文件落地存储，Loader 拼接加载。

### 🧪 短 Payload 利用链爆破
- 基于 `Thread.sleep()` 时间差检测，替代回显验证利用链，大幅缩短爆破 payload 长度。

---

## 📦 目录结构

```
ShiroAttack2_bypass/
├── src/                       # 工具源码
├── lib/
│   ├── 1.8.3/commons-beanutils-1.8.3.jar   # CommonsBeanutils 1.8.3 依赖
│   └── 1.9.2/commons-beanutils-1.9.2.jar   # CommonsBeanutils 1.9.2 依赖
├── data/shiro_keys.txt        # Shiro 密钥字典（爆破密钥用，可选）
├── pom.xml
└── README.md
```

---

## 🔧 构建

需要 **JDK 8（含 JavaFX，如 Oracle JDK 8 / Amazon Corretto 8）** 与 Maven。

```bash
mvn -DskipTests package
```

产物：`target/shiro_attack-1.1-all.jar`（完整版，含全部依赖）。

---

## 🚀 使用

```bash
java -jar shiro_attack-1.1-all.jar
```

> 注意：`lib/` 目录需与 jar 同级（工具按相对路径 `lib/<version>/` 加载 CommonsBeanutils 依赖）；`data/shiro_keys.txt` 存放密钥字典（可选）。

### 典型流程（面对"限长/不确定环境"的靶场）

1. 填入目标地址 → **检测Shiro** → **指定密钥**（默认密钥 `kPH+bIxk5D2deZiIxcaaaA==`）或 **爆破密钥**。
2. 在「Header绕过」页：
   - **探测限长** → 自动得到 Cookie 限长并回填；
   - **探测环境** → 得知 Tomcat/Spring，决定回显方式；
   - 选择「版本=1.8.3、利用链=CommonsBeanutilsString」→ **爆破利用链**。
3. 若限长过小无法回显/注入 → **盲注读取**（文件读取 `/flag`，或命令执行 `cat /flag`），全自动逐字符泄露结果。

---

## 📖 关键技术说明

### 为什么原版 1.8.3 打不出 RCE？
`BeanComparator` 的 `serialVersionUID` 是 JVM 按类结构**计算**出来的（非显式字段），1.8.3 与 1.9.2 不同：

| 版本 | serialVersionUID |
|------|------------------|
| 1.8.3 | `-3490850999041592962` |
| 1.9.2 | `-2044202215314119608` |

原版 `_183` 用 javassist 动态"改" SUID，但序列化流里携带的仍是 1.9.x 的计算值，导致目标 1.8.3 环境报 `InvalidClassException`。本工具改为加载**真实的 1.8.3 jar**，让 SUID 天然匹配。

### `$` 混淆为什么能绕过 WAF？
- WAF 用 `java.util.Base64`（**严格模式**），遇到 `$` 直接抛异常 → 误判为"非攻击"；
- Shiro 用 `org.apache.shiro.codec.Base64`（**宽松模式**），`decode()` 先 `discardNonBase64()` 丢弃非 Base64 字符再解码。

### 侧信道盲注原理
当 header 限长过小、无法回显时，让 payload 执行 `Thread.sleep(字符ASCII值 × 延迟倍率)`，把目标内容**编码成响应延迟**，再逐字符测量延迟反推 ASCII 值。

> 💡 **限长约束与取舍**：因为要控制 payload 体积塞进 Cookie 限长，读取逻辑用最精简的 `Scanner.next()`（首 token）。这意味着「读取全部内容」与「小体积」不可兼得——若目标内容含空格/多行，建议先 `cat /flag | base64` 转成单 token 再盲注。

---

## 📝 更新日志

### v1.2 — 20260824
- 修复 1.8.3 版本支持（真实加载 `lib/1.8.3` 的 beanutils，修正 SUID）
- Header绕过 新增「版本 + 利用链」选择器
- 新增「探测限长」「探测环境」功能
- 新增「自动化侧信道盲注」（文件读取 / 命令执行，延迟倍率、中位数次数可调）
- 盲注改为实时逐字符输出，单趟循环读取，长内容可实时确认进度
- 修复盲注在限长靶场返回 400 的问题：精简 hutool 默认头 + 压缩 `_183` gadget 序列化体积

### v1.1 — 20260413
- Cookie 合并、hutool 请求方式对齐、性能优化
- 利用链类加载兼容 JDK 9+
- Header 绕过动态类改为 JDK6 字节码、回显改走响应头 `X-C`
- 绕过页 UI 可缩放等体验优化

---

## 🙏 致谢（排名不分先后）

- [ShiroAttack2](https://github.com/SummerSec/ShiroAttack2)（SummerSec）
- 凌曦安全 Syst1m
- 076w

## ⚠️ 免责声明

该工具**仅用于安全自查检测**。由于传播、利用本工具所提供的信息而造成的任何直接或间接后果及损失，均由使用者本人负责，作者不为此承担任何责任。

未经网络安全部门及相关授权，不得擅自使用本工具对任何系统进行攻击活动，不得以任何方式将其用于商业目的。请遵守《中华人民共和国网络安全法》等相关法律法规，否则后果自负。
