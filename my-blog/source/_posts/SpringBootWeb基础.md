---
title: SpringBootWeb基础
date: 2026-07-11 21:06:00
categories:
  - Java
  - Spring
  - SpringBoot
tags:
  - Java
  - SpringBoot
  - Web
  - Spring
---

# SpringBoot Web 基础

> 本篇笔记涵盖 HTTP 协议、请求/响应处理、三层架构、IOC/DI 核心概念等内容。

---

## 一、SpringBoot Web 入门认知

### 1.1 起步依赖

`spring-boot-starter-web` 和 `spring-boot-starter-test` 是 **起步依赖**，它们自动引入了开发 Web 应用所需的核心依赖包（如 Spring MVC、Tomcat、JUnit 等）。

### 1.2 内嵌 Tomcat

SpringBoot 项目**一个 `main` 方法即可启动**，原因是**内嵌了 Tomcat 服务器**，无需外部部署。

---

## 二、HTTP 协议基础

HTTP（HyperText Transfer Protocol）即**超文本传输协议**，是客户端与服务端之间的通信规范。

### 2.1 请求报文结构

HTTP 请求由三部分组成：

| 组成部分 | 说明 |
|:---|:---|
| **请求行** | 包含请求方法、URL、HTTP 版本 |
| **请求头** | 描述客户端信息及请求附加参数 |
| **请求体** | POST 等方法携带的请求数据 |

#### 常见请求头一览

| 请求头 | 含义 |
|:---|:---|
| `Accept` | 客户端可处理的媒体类型 |
| `Accept-Encoding` | 客户端支持的压缩编码（如 gzip） |
| `Accept-Language` | 客户端偏好的语言 |
| `Cache-Control` | 缓存控制策略 |
| `Connection` | 连接管理方式（keep-alive 等） |
| `Content-Length` | 请求体长度（字节） |
| `Content-Type` | 请求体的媒体类型 |
| `Host` | 目标服务器主机名 |
| `User-Agent` | 客户端浏览器/程序标识 |
| `Referer` | 当前请求来源页面地址 |
| `Origin` | 发起请求的源（跨域相关） |

> **注意**：GET 请求的参数位于**请求行**中，因此不需要设置请求体。

### 2.2 GET vs POST 对比

| 区分维度 | GET 请求 | POST 请求 |
|:---:|:---|:---|
| **请求参数位置** | 请求行中，拼接在 URL 后面<br/>例：`/brand/findAll?name=OPPO&status=1` | 请求体中 |
| **参数长度限制** | 有限制（因浏览器而异，一般几 KB） | 无限制 |
| **安全性** | ⚠️ 低——参数暴露在地址栏 | ✅ 相对较高——参数不可见 |
| **典型场景** | 数据查询、页面跳转 | 表单提交、文件上传、数据修改 |

---

## 三、SpringBoot 请求处理

### 3.1 获取请求信息

SpringBoot 通过 **`HttpServletRequest`** 对象可以获取请求的各个部分（参数、路径、方法、头信息等）。

```java 获取请求信息的完整示例
@RestController
public class RequestController {

    /**
     * 请求路径: http://localhost:8080/request?name=Tom&age=18
     */
    @RequestMapping("/request")
    public String request(HttpServletRequest request){
        // 1.获取请求参数 name, age
        String name = request.getParameter("name");
        String age = request.getParameter("age");
        System.out.println("name = " + name + ", age = " + age);

        // 2.获取请求路径
        String uri = request.getRequestURI();   // /request
        String url = request.getRequestURL().toString(); // http://localhost:8080/request
        System.out.println("uri = " + uri);
        System.out.println("url = " + url);

        // 3.获取请求方式 (GET/POST)
        String method = request.getMethod();
        System.out.println("method = " + method);

        // 4.获取请求头 (示例: User-Agent)
        String header = request.getHeader("User-Agent");
        System.out.println("header = " + header);

        return "request success";
    }
}
```

---

## 四、SpringBoot 响应处理

### 4.1 响应报文结构

HTTP 响应由三部分组成：

| 组成部分 | 说明 |
|:---|:---|
| **响应行** | 包含 HTTP 版本、状态码、状态描述 |
| **响应头** | 描述服务端信息及响应附加参数 |
| **响应体** | 返回给客户端的实际数据 |

#### HTTP 状态码分类

| 类别 | 含义 | 常见示例 |
|:---:|:---|:---|
| **1xx** | 响应中（ informational ） | 100 Continue |
| **2xx** | ✅ 成功 | 200 OK、201 Created |
| **3xx** | 重定向 | 301 Moved、302 Found |
| **4xx** | ❌ 客户端错误 | 400 Bad Request、404 Not Found |
| **5xx** | ❌ 服务端错误 | 500 Internal Server Error |

### 4.2 设置响应的方式

SpringBoot 中有两种主要方式来设置响应数据：

| 方式 | 说明 | 适用场景 |
|:---|:---|:---|
| **HttpServletResponse** | 原生 Servlet API，手动设置状态码/头/体 | 需要精细控制响应细节 |
| **ResponseEntity\<T\>** | Spring 封装的对象，链式调用更优雅 | 推荐日常使用 |

#### 方式一：HttpServletResponse

```java HttpServletResponse 设置响应
@RequestMapping("/response")
public void response(HttpServletResponse response) throws IOException {
    // 1.设置响应状态码
    response.setStatus(401);
    // 2.设置响应头
    response.setHeader("name", "Planck");
    // 3.设置响应体
    response.setContentType("text/html;charset=utf-8");
    response.setCharacterEncoding("utf-8");
    response.getWriter().write("<h1>hello response</h1>");
}
```

#### 方式二：ResponseEntity（推荐）

```java ResponseEntity 设置响应（推荐写法）
@RequestMapping("/response2")
public ResponseEntity<String> response2() {
    return ResponseEntity
            .status(401)              // 设置状态码
            .header("name", "Planck")  // 设置响应头
            .body("<h1>hello response</h1>"); // 设置响应体
}
```

> 💡 **提示**：如果没有特殊需求，通常**无需手动设置**响应状态码和响应头，Spring 会根据业务逻辑自动处理。

### 4.3 从 classpath 读取资源文件

```java 从 classpath 根目录读取文件的通用写法
// this.getClass()           → 获取当前运行类的 Class 对象
// .getClassLoader()         → 获取类加载器 ClassLoader
// .getResourceAsStream(...) → 从 classpath 根目录查找文件并返回输入流
//
// 搜索路径：
//   - 编译后的 target/classes 目录
//   - src/main/resources 目录下的文件
//
InputStream in = this.getClass()
        .getClassLoader()
        .getResourceAsStream("user.txt");

// ⚠️ 注意：如果文件不存在，in 为 null，后续读取会抛出 NullPointerException
```

---

## 五、三层架构

### 5.1 架构分层

| 层次 | 注解 | 职责 |
|:---:|:---:|:---|
| **Controller（控制层）** | `@Controller` / `@RestController` | 接收请求、调用 Service、返回响应 |
| **Service（业务层）** | `@Service` | 业务逻辑处理、事务控制 |
| **Dao / Repository（持久层）** | `@Repository` / `@Mapper` | 数据库 CRUD 操作 |

### 5.2 三大优势

1. **分层明确** — 各层职责单一，便于分工协作
2. **解耦合** — 层与层之间通过接口交互，降低依赖
3. **可维护性高** — 修改某一层不影响其他层

---

## 六、IOC 与 DI

### 6.1 核心概念

| 概念 | 全称 | 含义 |
|:---:|:---|:---|
| **IOC** | Inversion of Control（控制反转） | 将对象的**创建权**交给 Spring 容器管理，而非手动 new |
| **DI** | Dependency Injection（依赖注入） | Spring 容器在运行时，将依赖对象**自动注入**到目标 Bean 中 |

简单来说：**IOC 是思想，DI 是实现手段**。

### 6.2 声明 Bean 的注解

| 注解 | 类型 | 标注位置 | 说明 |
|:---:|:---:|:---|:---|
| `@Component` | 基础注解 | 任意组件 | 不属于下面三类时使用 |
| `@Controller` |衍生注解 | 控制层 | `@Component` 的特化，标注 Web 控制器 |
| `@Service` | 衍生注解 | 业务层 | `@Component` 的特化，标注业务逻辑类 |
| `@Repository` | 衍生注解 | 持久层 | `@Component` 的特化，标注数据访问类（Mybatis 整合后用得较少） |

> 📌 **组件扫描**：以上注解要生效，必须被 `@ComponentScan` 扫描到。<br/>
> 实际上 `@SpringBootApplication` 内部已包含该注解，**默认扫描启动类所在包及其子包**。

### 6.3 依赖注入：@Autowired 的三种方式

#### ① 属性注入（Field Injection）

```java 属性注入示例
@RestController
public class UserController {
    @Autowired
    private UserService userService;
}
```

| 优点 | 缺点 |
|:---|:---|
| ✅ 代码简洁，开发效率高 | ❌ 可能破坏封装性（直接操作私有字段） |
| | ❌ 不利于单元测试（无法在构造时传入 Mock 对象） |
| | ❌ IDEA 会给出警告（不推荐） |

#### ② Setter 注入

```java Setter 注入示例
@RestController
public class UserController {
    private UserService userService;

    @Autowired
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
}
```

| 优点 | 缺点 |
|:---|:---|
| ✅ 保持封装性，依赖关系清晰 | ❌ 需额外编写 setter 方法 |
| | ❌ 对象构造完成后仍可被修改，非线程安全 |

#### ③ 构造器注入（⭐ 推荐）

```java 构造器注入示例（推荐）
@RestController
public class UserController {
    private final UserService userService;

    @Autowired
    public UserController(UserService userService) {
        this.userService = userService;
    }
}
```

| 优点 | 缺点 |
|:---|:---|
| ✅ 依赖关系一目了然，强制必要依赖 | ❌ 代码量略多 |
| ✅ 保证对象构建完成即处于可用状态 | |
| ✅ 配合 `final` 保证不可变性，线程安全 | |
| ✅ 方便单元测试（可直接传参构造） | |

> 💡 **省略技巧**：如果类中**只有一个构造函数**，`@Autowired` 可以省略，Spring 4.3+ 自动识别。

### 6.4 解决多个同类型 Bean 冲突

当容器中存在**多个相同类型**的 Bean 时，`@Autowired` 按类型装配会失败，有以下三种解决方案：

#### 方案一：@Qualifier + @Autowired

通过 `@Qualifier` 指定 Bean 名称进行精确匹配：

```java @Qualifier 按名称指定注入
@Autowired
@Qualifier("userServiceImpl")
private UserService userService;
```

#### 方案二：@Primary 标记优先级

在某个实现类上加 `@Primary`，标记为首选 Bean：

```java @Primary 标记优先 Bean
@Primary
@Service
public class UserServiceImpl implements UserService {
    // ...
}
```

#### 方案三：@Resource（JDK 原生注解）

按**名称**注入（默认字段名作为 beanName），不属于 Spring：

```java @Resource 按名称注入
@Resource(name = "userServiceImpl")
private UserService userService;
```

---

## 七、知识速查表

| 主题 | 关键点 |
|:---|:---|
| 起步依赖 | `spring-boot-starter-web`、`spring-boot-starter-test` |
| 内嵌 Tomcat | 一个 main 方法即可启动 Web 服务 |
| HTTP 请求组成 | 请求行 + 请求头 + 请求体 |
| GET vs POST | GET 参数在 URL、有长度限制；POST 参数在体中、无限制 |
| 请求处理核心 API | `HttpServletRequest`（原生）、`@RequestMapping` 及其变体（简化） |
| 响应处理核心 API | `HttpServletResponse`（原生）、`ResponseEntity`（推荐链式写法） |
| 三层架构 | Controller → Service → Dao（Repository/Mapper） |
| IOC | 控制反转，将对象创建权交予 Spring 容器 |
| DI | 依赖注入，运行时自动装配 Bean |
| Bean 声明注解 | `@Component` → `@Controller` / `@Service` / `@Repository` |
| DI 推荐方式 | **构造器注入**（配合 `final` + 单构造可省略 `@Autowired`） |
| 多 Bean 冲突解决 | `@Qualifier`、`@Primary`、`@Resource` |
