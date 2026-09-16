---
title: SpringMVC
date: 2026-07-31 17:33:18
categories:
  - Java
  - SSM
tags:
  - SpringMVC
  - 表述层框架
  - RESTful
  - 参数校验
cover: /img/lika1.png
top_img: /img/lika1.png
---

# SpringMVC

> 本笔记基于尚硅谷 SSM 课程「SpringMVC实战：构建高效表述层框架」整理，系统记录 SpringMVC 从环境搭建到参数接收、数据响应、RESTFul 设计、全局异常处理、拦截器、参数校验的完整知识体系。

SpringMVC 要解决的核心问题只有一个：**把原生 Servlet 那套"手动取参、手动设属性、手动转发"的繁琐代码，变成"形参列表接收参数、return 返回结果"的极简写法。**

整章的知识主线：

| 阶段 | 要解决的问题 | 核心内容 |
|---|---|---|
| 框架认知 | SpringMVC 是什么、怎么跑起来 | DispatcherServlet、五大核心组件、调用流程 |
| 接收数据 | 前端传来的数据怎么拿 | 路径设置、param / 路径 / json / cookie / header 参数 |
| 响应数据 | 结果怎么返回给前端 | 模板视图、转发重定向、JSON、静态资源 |
| 接口设计 | 接口怎么设计才规范 | RESTFul 风格与实战 |
| 功能扩展 | 异常、拦截、校验怎么统一处理 | 全局异常处理、拦截器、JSR303 校验 |

---

## 一、SpringMVC 简介和体验

### (一) 介绍

Spring Web MVC 是**基于 Servlet API 构建的原始 Web 框架**，从一开始就包含在 Spring Framework 中。正式名称 "Spring Web MVC" 来自其源模块的名称（`spring-webmvc`），通常简称为 "Spring MVC"。

在控制层框架历经 Struts、WebWork、Struts2 等诸多产品的历代更迭之后，目前业界普遍选择了 SpringMVC 作为 Java EE 项目表述层开发的**首选方案**。之所以能做到这一点，是因为它具备如下显著优势：

- **Spring 家族原生产品**，与 IOC 容器等基础设施无缝对接
- **表述层各细分领域需要解决的问题全方位覆盖**，提供全面解决方案
- **代码清新简洁**，大幅度提升开发效率
- **内部组件化程度高**，可插拔式组件即插即用，想要什么功能配置相应组件即可
- **性能卓著**，尤其适合现代大型、超大型互联网项目要求

**与原生 Servlet 的对比**

原生 Servlet API 开发，每一个参数都要手动从 request 里取：

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response)
                                                    throws ServletException, IOException {
    String userName = request.getParameter("userName");

    System.out.println("userName=" + userName);
}
```

基于 SpringMVC 开发，参数自动绑定到形参：

```java
@RequestMapping("/user/login")
public String login(@RequestParam("userName") String userName, String password) {

    log.debug("userName=" + userName);
    // 调用业务即可

    return "result";
}
```

### (二) 主要作用

SSM 框架构建起单体项目的技术栈需求，其中 **SpringMVC 负责表述层（控制层）实现简化**。它的作用主要覆盖表述层：

- 请求映射
- 数据输入
- 视图界面
- 请求分发
- 表单回显
- 会话控制
- 过滤拦截
- 异步交互
- 文件上传 / 文件下载
- 数据校验
- 类型转换

最终总结成两点：

1. **简化前端参数接收**（形参列表）
2. **简化后端数据响应**（返回值）

### (三) 核心组件和调用流程理解

Spring MVC 与许多其他 Web 框架一样，是围绕**前端控制器模式**设计的，其中中央 Servlet `DispatcherServlet` 做整体请求处理调度。

除了 `DispatcherServlet`，SpringMVC 还会提供其他特殊组件协作完成请求处理和响应呈现。

**SpringMVC 处理请求流程**

```
请求 → DispatcherServlet（CEO，统一收口）
         ↓ 询问路径对应的 handler
       HandlerMapping（秘书，查路径→handler 映射）
         ↓ 返回 handler
       HandlerAdapter（经理，适配器，负责解析参数/处理响应）
         ↓ 间接调用
       Handler（打工人，我们写的 Controller 方法）
         ↓ 返回 ModelAndView / 数据
       ViewResolver（财务，解析视图名→页面，前后端分离时不需要）
         ↓
      响应
```

**五大组件理解**

| 组件 | 角色比喻 | 说明 |
|---|---|---|
| `DispatcherServlet` | CEO | SpringMVC 提供，需要用 `web.xml` 配置使其生效。它是整个流程处理的核心，**所有请求都经过它的处理和分发** |
| `HandlerMapping` | 秘书 | SpringMVC 提供，需要进行 IoC 配置使其加入 IoC 容器方可生效。内部缓存 handler(controller 方法) 和 handler 访问路径数据，被 DispatcherServlet 调用，用于查找路径对应的 handler |
| `HandlerAdapter` | 经理 | SpringMVC 提供，需要加入 IoC 容器。它可以处理请求参数和处理响应数据，每次 DispatcherServlet 都是通过 HandlerAdapter **间接**调用 handler，是 handler 和 DispatcherServlet 之间的适配器 |
| `Handler` | 打工人 | 又称处理器，是 Controller 类内部的方法简称，由我们自己定义，用来接收参数、向后调用业务、最终返回响应结果 |
| `ViewResolver` | 财务 | SpringMVC 提供，需要加入 IoC 容器。主要作用是简化模版视图页面查找。⚠️ **前后端分离项目后端只返回 JSON，不需要视图解析器**，所以它相对其他组件**不是必须的** |

### (四) 快速体验

**1. 配置分析**

搭建一个 SpringMVC 环境需要做三件事：

1. 配置 `DispatcherServlet`，设置处理所有请求
2. 把 `HandlerMapping`、`HandlerAdapter`、`Handler` 加入 IoC 容器，供 DispatcherServlet 调用
3. 自己声明的 Handler（Controller）需要配置到 `HandlerMapping` 中供 DispatcherServlet 查找

**2. 导入依赖**

```xml
<properties>
    <spring.version>6.0.6</spring.version>
    <servlet.api>9.1.0</servlet.api>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<dependencies>
    <!-- springioc 相关依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>${spring.version}</version>
    </dependency>

    <!-- web 相关依赖 -->
    <!--
        在 Spring Web MVC 6 中，Servlet API 迁移到了 Jakarta EE API，
        因此配置 DispatcherServlet 时需要使用 Jakarta EE 提供的类库和命名空间。
        若使用旧版 Servlet API，会报错：
        'org.springframework.web.servlet.DispatcherServlet'
        is not assignable to 'javax.servlet.Servlet,jakarta.servlet.Servlet'
    -->
    <dependency>
        <groupId>jakarta.platform</groupId>
        <artifactId>jakarta.jakartaee-web-api</artifactId>
        <version>${servlet.api}</version>
        <scope>provided</scope>
    </dependency>

    <!-- springwebmvc 相关依赖 -->
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>${spring.version}</version>
    </dependency>
</dependencies>
```

**3. Controller 声明**

```java
@Controller
public class HelloController {

    // handlers

    /**
     * handler 就是 controller 内部的具体方法
     * @RequestMapping("/springmvc/hello") 就是用来向 handlerMapping 中注册的方法注解
     * @ResponseBody 代表向浏览器直接返回数据
     */
    @RequestMapping("/springmvc/hello")
    @ResponseBody
    public String hello() {
        System.out.println("HelloController.hello");
        return "hello springmvc!!";
    }
}
```

**4. Spring MVC 核心组件配置类**

```java
// SpringMVC 对应组件的配置类 [声明 SpringMVC 需要的组件信息]

// 导入 handlerMapping 和 handlerAdapter 的三种方式：
// 1. 自动导入 handlerMapping 和 handlerAdapter [推荐]
// 2. 可以不添加，springmvc 会检查是否配置，没有配置默认加载
// 3. 使用 @Bean 方式配置 handlerMapper 和 handlerAdapter
@EnableWebMvc
@Configuration
@ComponentScan(basePackages = "com.atguigu.controller")  // 进行 controller 扫描
// WebMvcConfigurer 是 springMvc 进行组件配置的规范，配置组件，提供各种方法
public class SpringMvcConfig implements WebMvcConfigurer {

    @Bean
    public HandlerMapping handlerMapping() {
        return new RequestMappingHandlerMapping();
    }

    @Bean
    public HandlerAdapter handlerAdapter() {
        return new RequestMappingHandlerAdapter();
    }
}
```

**5. SpringMVC 环境搭建（替代 web.xml）**

```java
// SpringMVC 提供的接口，是替代 web.xml 的方案，更方便实现完全注解方式 ssm 处理
// Springmvc 框架会自动检查当前类的实现类，会自动加载
// getRootConfigClasses / getServletConfigClasses 提供的配置类
// getServletMappings 返回的地址设置 DispatcherServlet 对应处理的地址
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    /**
     * 指定 service / mapper 层的配置类
     */
    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    /**
     * 指定 springmvc 的配置类
     */
    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { SpringMvcConfig.class };
    }

    /**
     * 设置 dispatcherServlet 的处理路径
     * 一般情况下为 / 代表处理所有请求
     */
    @Override
    protected String[] getServletMappings() {
        return new String[] { "/" };
    }
}
```

**6. 启动测试**

> ⚠️ 注意：**Tomcat 应该是 10+ 版本**，方可支持 Jakarta EE API！

---

## 二、SpringMVC 接收数据

### (一) 访问路径设置

`@RequestMapping` 注解的作用就是**将请求的 URL 地址和处理请求的方式（handler 方法）关联起来，建立映射关系**。SpringMVC 接收到指定的请求，就会来找到在映射关系中对应的方法来处理这个请求。

**1. 精准路径匹配**

在 `@RequestMapping` 注解指定 URL 地址时，不使用任何通配符，按照请求地址进行精确匹配。

```java
@Controller
public class UserController {

    /**
     * 精准设置访问地址 /user/login
     */
    @RequestMapping(value = {"/user/login"})
    @ResponseBody
    public String login() {
        System.out.println("UserController.login");
        return "login success!!";
    }

    /**
     * 精准设置访问地址 /user/register
     */
    @RequestMapping(value = {"/user/register"})
    @ResponseBody
    public String register() {
        System.out.println("UserController.register");
        return "register success!!";
    }
}
```

**2. 模糊路径匹配**

通过使用通配符，匹配多个类似的地址。

```java
@Controller
public class ProductController {

    /**
     *  路径设置为 /product/*
     *    /*  为单层任意字符串：/product/a、/product/aaa 可以访问；/product/a/a 不可以
     *  路径设置为 /product/**
     *    /** 为任意层任意字符串：/product/a、/product/aaa 可以访问；/product/a/a 也可以
     */
    @RequestMapping("/product/*")
    @ResponseBody
    public String show() {
        System.out.println("ProductController.show");
        return "product show!";
    }
}
```

单层匹配与多层匹配：

- `/*`：只能匹配 URL 地址中的**一层**，想准确匹配两层就写 `/*/*`，以此类推
- `/**`：可以匹配 URL 地址中的**多层**

> 所谓"一层或多层"，是指一个 URL 地址字符串被 `/` 划分出来的各个层次。这个知识点虽然对 `@RequestMapping` 来说实用性不大，但**将来配置拦截器时也遵循这个规则**。

**3. 类级别和方法级别的区别**

| 位置 | 作用 |
|---|---|
| **类级别** | 映射整个控制器的通用请求路径。如果控制器中的多个方法都需要映射同一请求路径，就不需要在每个方法上都添加映射路径 |
| **方法级别** | 更细粒度地映射请求路径和处理方法。当多个方法处理同一个路径的不同操作时，使用方法级别注解做更精细的映射 |

```java
// 1. 标记到 handler 方法
@RequestMapping("/user/login")
@RequestMapping("/user/register")
@RequestMapping("/user/logout")

// 2. 优化：标记类 + handler 方法
// 类上
@RequestMapping("/user")
// handler 方法上
@RequestMapping("/login")
@RequestMapping("/register")
@RequestMapping("/logout")
```

**4. 附带请求方式限制**

HTTP 协议定义了八种请求方式，在 SpringMVC 中封装到了下面这个枚举类：

```java
public enum RequestMethod {
  GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE
}
```

默认情况下 `@RequestMapping("/logout")` **任何请求方式都可以访问**。如果需要特定指定：

```java
@Controller
public class UserController {

    /**
     * 精准设置访问地址 /user/login
     * method = RequestMethod.POST 可以指定单个或者多个请求方式
     * 注意：违背请求方式会出现 405 异常！
     */
    @RequestMapping(value = {"/user/login"}, method = RequestMethod.POST)
    @ResponseBody
    public String login() {
        System.out.println("UserController.login");
        return "login success!!";
    }

    /**
     * 精准设置访问地址 /user/register
     */
    @RequestMapping(value = {"/user/register"}, method = {RequestMethod.POST, RequestMethod.GET})
    @ResponseBody
    public String register() {
        System.out.println("UserController.register");
        return "register success!!";
    }
}
```

> ⚠️ **违背请求方式，会出现 405 异常！**

**5. 进阶注解**

还有 `@RequestMapping` 的 HTTP 方法特定快捷方式变体：

- `@GetMapping`
- `@PostMapping`
- `@PutMapping`
- `@DeleteMapping`
- `@PatchMapping`

```java
@RequestMapping(value = "/login", method = RequestMethod.GET)
        ||
@GetMapping(value = "/login")
```

> ⚠️ 注意：**进阶注解只能添加到 handler 方法上，无法添加到类上！**

**6. 常见配置问题**

出现原因：**多个 handler 方法映射了同一个地址**，导致 SpringMVC 在接收到这个地址的请求时不知道该找哪个 handler 方法处理。

```
There is already 'demo03MappingMethodHandler' bean method
com.atguigu.mvc.handler.Demo03MappingMethodHandler#empGet() mapped.
```

### (二) 接收参数（重点）

#### param 和 json 参数比较

在 HTTP 请求中，我们可以选择不同的参数类型，如 param 类型和 JSON 类型。二者的区别：

| 对比维度 | param 类型 | JSON 类型 |
|---|---|---|
| **参数编码** | 编码为 ASCII 码。如 `name=john doe` 会被编码为 `name=john%20doe` | 编码为 UTF-8 |
| **参数顺序** | 没有顺序限制 | 有序，键值对有序排列 |
| **数据类型** | 仅支持字符串、数值、布尔等简单类型 | 支持数组、对象等复杂类型 |
| **嵌套性** | 不支持嵌套 | 支持嵌套，可传递复杂数据结构 |
| **可读性** | 格式更简单、易读 | 传递嵌套数据结构时更清晰 |

> 总结：param 适用于**单一的数据传递**，JSON 适用于**更复杂的数据结构传递**。实际开发中常见做法是：**GET 请求用 param，POST 请求用 JSON**。

#### param 参数接收

**1. 直接接值**

只要**形参数名和类型与传递参数相同**，即可自动接收。

```java
@Controller
@RequestMapping("param")
public class ParamController {

    /**
     * 前端请求: http://localhost:8080/param/value?name=xx&age=18
     *
     * 可以利用形参列表，直接接收前端传递的 param 参数
     *     要求：参数名 = 形参名
     *          类型相同
     * 出现乱码正常，json 接收具体解决
     */
    @GetMapping(value = "/value")
    @ResponseBody
    public String setupForm(String name, int age) {
        System.out.println("name = " + name + ", age = " + age);
        return name + age;
    }
}
```

**2. `@RequestParam` 注解**

可以使用 `@RequestParam` 注释将 Servlet 请求参数（即查询参数或表单数据）绑定到控制器中的方法参数。

**使用场景**：

- 指定绑定的请求参数名
- 要求请求参数必须传递
- 为请求参数提供默认值

```java
/**
 * 前端请求: http://localhost:8080/param/data?name=xx&stuAge=18
 *
 *  使用 @RequestParam 注解标记 handler 方法的形参
 *  指定形参对应的请求参数 @RequestParam(请求参数名称)
 */
@GetMapping(value = "/data")
@ResponseBody
public Object paramForm(@RequestParam("name") String name,
                        @RequestParam("stuAge") int age) {
    System.out.println("name = " + name + ", age = " + age);
    return name + age;
}
```

默认情况下，使用此批注的方法参数是**必需的**。如果没设置非必须、也没有传递参数，会直接报错。将参数设置非必须，并设置默认值：

```java
@GetMapping(value = "/data")
@ResponseBody
public Object paramForm(@RequestParam("name") String name,
                        @RequestParam(value = "stuAge", required = false, defaultValue = "18") int age) {
    System.out.println("name = " + name + ", age = " + age);
    return name + age;
}
```

**3. 特殊场景接值**

**(1) 一名多值**

多选框提交数据时一个 key 对应多个值，可以使用集合进行接收。

```java
/**
 * 前端请求: http://localhost:8080/param/mul?hbs=吃&hbs=喝
 *
 *  一名多值，可以使用集合接收即可！但是需要使用 @RequestParam 注解指定
 */
@GetMapping(value = "/mul")
@ResponseBody
public Object mulForm(@RequestParam List<String> hbs) {
    System.out.println("hbs = " + hbs);
    return hbs;
}
```

**(2) 实体接收**

Spring MVC 允许开发者使用实体对象来接收 HTTP 请求中的参数。通过这种方式，可以在方法内部直接使用对象的属性来访问请求参数，而不需要每个参数都写一遍。

定义一个用于接收参数的实体类：

```java
public class User {

  private String name;

  private int age = 18;

  // getter 和 setter 略
}
```

在控制器中，使用实体对象接收：

```java
@Controller
@RequestMapping("param")
public class ParamController {

    @RequestMapping(value = "/user", method = RequestMethod.POST)
    @ResponseBody
    public String addUser(User user) {
        // 在这里可以使用 user 对象的属性来接收请求参数
        System.out.println("user = " + user);
        return "success";
    }
}
```

> ⚠️ 将请求参数 `name` 和 `age` 映射到实体类属性上，**要求属性名必须等于参数名**，否则无法映射！

#### 路径参数接收

路径传递参数是一种在 URL 路径中传递参数的方式。在 RESTful 的 Web 应用程序中，经常使用路径传递参数来表示资源的唯一标识符或更复杂的表示方式。Spring MVC 提供了 `@PathVariable` 注解来处理路径传递参数。

`@PathVariable` 注解允许将 URL 中的占位符映射到控制器方法中的参数。

```java
/**
 * 动态路径设计: /user/{动态部分}/{动态部分}   动态部分使用 {} 包含即可！{} 内部动态标识
 * 形参列表取值: @PathVariable Long id  如果形参名 = {动态标识} 自动赋值
 *              @PathVariable("动态标识") Long id  如果形参名 != {动态标识} 可以通过指定动态标识赋值
 *
 * 访问测试:  /param/user/1/root  -> id = 1  uname = root
 */
@GetMapping("/user/{id}/{name}")
@ResponseBody
public String getUser(@PathVariable Long id,
                      @PathVariable("name") String uname) {
    System.out.println("id = " + id + ", uname = " + uname);
    return "user_detail";
}
```

#### json 参数接收

前端传递 JSON 数据时，Spring MVC 可以使用 `@RequestBody` 注解将 JSON 数据转换为 Java 对象。`@RequestBody` 注解表示当前方法参数的值应该从请求体中获取，并且需要指定 value 属性来指示请求体应该映射到哪个参数上。

**1. 前端发送 JSON 数据的示例**

```json
{
  "name": "张三",
  "age": 18,
  "gender": "男"
}
```

**2. 定义一个用于接收 JSON 数据的 Java 类**

```java
public class Person {
  private String name;
  private int age;
  private String gender;
  // getter 和 setter 略
}
```

**3. 在控制器中使用 `@RequestBody` 接收**

```java
@PostMapping("/person")
@ResponseBody
public String addPerson(@RequestBody Person person) {

  // 在这里可以使用 person 对象来操作 JSON 数据中包含的属性
  return "success";
}
```

**4. 完善配置**

测试时可能遇到问题：

```
org.springframework.web.HttpMediaTypeNotSupportedException:
Content-Type 'application/json;charset=UTF-8' is not supported
```

**原因**：

- 不支持 json 数据类型处理
- 没有 json 类型处理的工具（jackson）

**解决**：两步缺一不可。

第一步，配置类必须加 `@EnableWebMvc`：

```java
@EnableWebMvc  // json 数据处理，必须使用此注解，因为它会加入 json 处理器
@Configuration
@ComponentScan(basePackages = "com.atguigu.controller")
public class SpringMvcConfig implements WebMvcConfigurer {

}
```

第二步，`pom.xml` 加入 jackson 依赖：

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.0</version>
</dependency>
```

**5. `@EnableWebMvc` 注解说明**

`@EnableWebMvc` 注解效果等同于在 XML 配置中的 `<mvc:annotation-driven>` 元素。查看 `<mvc:annotation-driven>` 对应的解析工作：

打开源码 `org.springframework.web.servlet.config.MvcNamespaceHandler` → `AnnotationDrivenBeanDefinitionParser`：

```java
class AnnotationDrivenBeanDefinitionParser implements BeanDefinitionParser {

  public static final String HANDLER_MAPPING_BEAN_NAME = RequestMappingHandlerMapping.class.getName();

  public static final String HANDLER_ADAPTER_BEAN_NAME = RequestMappingHandlerAdapter.class.getName();

  static {
    ClassLoader classLoader = AnnotationDrivenBeanDefinitionParser.class.getClassLoader();
    javaxValidationPresent = ClassUtils.isPresent("jakarta.validation.Validator", classLoader);
    // ...
    jackson2Present = ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", classLoader) &&
            ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", classLoader);
    // ...
  }

  @Override
  @Nullable
  public BeanDefinition parse(Element element, ParserContext context) {
    // handlerMapping 加入到 ioc 容器
    readerContext.getRegistry().registerBeanDefinition(HANDLER_MAPPING_BEAN_NAME, handlerMappingDef);

    // 添加 jackson 转化器
    addRequestBodyAdvice(handlerAdapterDef);
    addResponseBodyAdvice(handlerAdapterDef);

    // handlerAdapter 加入到 ioc 容器
    readerContext.getRegistry().registerBeanDefinition(HANDLER_ADAPTER_BEAN_NAME, handlerAdapterDef);
    return null;
  }

  // 具体添加 jackson 转化对象方法
  protected void addRequestBodyAdvice(RootBeanDefinition beanDef) {
    if (jackson2Present) {
      beanDef.getPropertyValues().add("requestBodyAdvice",
          new RootBeanDefinition(JsonViewRequestBodyAdvice.class));
    }
  }

  protected void addResponseBodyAdvice(RootBeanDefinition beanDef) {
    if (jackson2Present) {
      beanDef.getPropertyValues().add("responseBodyAdvice",
          new RootBeanDefinition(JsonViewResponseBodyAdvice.class));
    }
  }
}
```

> 结论：`@EnableWebMvc` 做了三件事——注册 `RequestMappingHandlerMapping`、注册 `RequestMappingHandlerAdapter`、**检测 classpath 有 jackson 则挂上 json 转换器**。

### (三) 接收 Cookie 数据

可以使用 `@CookieValue` 注释将 HTTP Cookie 的值绑定到控制器中的方法参数。

考虑使用以下 cookie 的请求：

```
JSESSIONID=415A4AC178C59DACE0B2C9CA727CDD84
```

下面的示例演示如何获取 cookie 值：

```java
@GetMapping("/demo")
public void handle(@CookieValue("JSESSIONID") String cookie) {
  //...
}
```

### (四) 接收请求头数据

可以使用 `@RequestHeader` 批注将请求标头绑定到控制器中的方法参数。

请考虑以下带有标头的请求：

```
Host                    localhost:8080
Accept                  text/html,application/xhtml+xml,application/xml;q=0.9
Accept-Language         fr,en-gb;q=0.7,en;q=0.3
Accept-Encoding         gzip,deflate
Accept-Charset          ISO-8859-1,utf-8;q=0.7,*;q=0.7
Keep-Alive              300
```

下面的示例获取 `Accept-Encoding` 和 `Keep-Alive` 标头的值：

```java
@GetMapping("/demo")
public void handle(
    @RequestHeader("Accept-Encoding") String encoding,
    @RequestHeader("Keep-Alive") long keepAlive) {
  //...
}
```

### (五) 原生 API 对象操作

SpringMVC 支持在 handler 方法的形参列表中直接声明原生对象，官方文档见 [Method Arguments](https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-controller/ann-methods/arguments.html)。

下表描述了支持的控制器方法参数：

| Controller method argument | Description |
|---|---|
| `jakarta.servlet.ServletRequest`, `jakarta.servlet.ServletResponse` | 请求/响应对象 |
| `jakarta.servlet.http.HttpSession` | 强制存在会话。因此，这样的参数永远不会为 `null` |
| `java.io.InputStream`, `java.io.Reader` | 用于访问由 Servlet API 公开的原始请求正文 |
| `java.io.OutputStream`, `java.io.Writer` | 用于访问由 Servlet API 公开的原始响应正文 |
| `@PathVariable` | 接收路径参数注解 |
| `@RequestParam` | 用于访问 Servlet 请求参数，包括多部分文件。参数值将转换为声明的方法参数类型 |
| `@RequestHeader` | 用于访问请求标头。标头值将转换为声明的方法参数类型 |
| `@CookieValue` | 用于访问 Cookie。Cookie 值将转换为声明的方法参数类型 |
| `@RequestBody` | 用于访问 HTTP 请求正文。正文内容通过使用 `HttpMessageConverter` 实现转换为声明的方法参数类型 |
| `java.util.Map`, `org.springframework.ui.Model`, `org.springframework.ui.ModelMap` | 共享域对象，并在视图呈现过程中向模板公开 |
| `Errors`, `BindingResult` | 验证和数据绑定中的错误信息获取对象 |

获取原生对象示例：

```java
/**
 * 如果想要获取请求或者响应对象，或者会话等，可以直接在形参列表传入，并且不分先后顺序！
 * 注意：接收原生对象，并不影响参数接收！
 */
@GetMapping("api")
@ResponseBody
public String api(HttpSession session, HttpServletRequest request,
                  HttpServletResponse response) {
    String method = request.getMethod();
    System.out.println("method = " + method);
    return "api";
}
```

### (六) 共享域对象操作

#### 属性（共享）域作用回顾

在 JavaWeb 中，共享域指的是在 Servlet 中存储数据，以便在同一 Web 应用程序的多个组件中进行共享和访问。常见的共享域有四种：

| 共享域 | 作用范围 | 典型用途 |
|---|---|---|
| **ServletContext** | 整个 Web 应用程序，是最大的共享域 | 保存全局配置信息、所有用户共享的数据。在其中保存的数据是线程安全的 |
| **HttpSession** | 同一用户发出的多个请求之间，只能在同一个会话中使用 | 保存用户登录状态，让用户在多个页面间保持登录 |
| **HttpServletRequest** | 同一个请求的多个处理器方法之间 | 存储请求的参数和属性，让处理器方法之间可以访问 |
| **PageContext** | JSP 页面 Servlet 创建时自动创建 | 在 JSP 的各个作用域中共享数据，包括 pageScope、requestScope、sessionScope、applicationScope 等 |

#### Request 级别属性（共享）域

SpringMVC 提供了五种方式向请求域存数据，效果等价。

**1. 使用 `Model` 类型的形参**

```java
@RequestMapping("/attr/request/model")
@ResponseBody
public String testAttrRequestModel(

        // 在形参位置声明 Model 类型变量，用于存储模型数据
        Model model) {

    // 我们将数据存入模型，SpringMVC 会帮我们把模型数据存入请求域
    // 存入请求域这个动作也被称为暴露到请求域
    model.addAttribute("requestScopeMessageModel", "i am very happy[model]");

    return "target";
}
```

**2. 使用 `ModelMap` 类型的形参**

```java
@RequestMapping("/attr/request/model/map")
@ResponseBody
public String testAttrRequestModelMap(ModelMap modelMap) {

    modelMap.addAttribute("requestScopeMessageModelMap", "i am very happy[model map]");

    return "target";
}
```

**3. 使用 `Map` 类型的形参**

```java
@RequestMapping("/attr/request/map")
@ResponseBody
public String testAttrRequestMap(Map<String, Object> map) {

    map.put("requestScopeMessageMap", "i am very happy[map]");

    return "target";
}
```

**4. 使用原生 request 对象**

```java
@RequestMapping("/attr/request/original")
@ResponseBody
public String testAttrOriginalRequest(HttpServletRequest request) {

    request.setAttribute("requestScopeMessageOriginal", "i am very happy[original]");

    return "target";
}
```

**5. 使用 `ModelAndView` 对象**

```java
@RequestMapping("/attr/request/mav")
public ModelAndView testAttrByModelAndView() {

    // 1. 创建 ModelAndView 对象
    ModelAndView modelAndView = new ModelAndView();
    // 2. 存入模型数据
    modelAndView.addObject("requestScopeMessageMAV", "i am very happy[mav]");
    // 3. 设置视图名称
    modelAndView.setViewName("target");

    return modelAndView;
}
```

> `ModelAndView` 的优势在于：**既能存数据，又能同时指定视图名**，一步到位。

#### Session 级别属性（共享）域

```java
@RequestMapping("/attr/session")
@ResponseBody
public String testAttrSession(HttpSession session) {
    // 直接对 session 对象操作，即对会话范围操作！
    return "target";
}
```

#### Application 级别属性（共享）域

> 解释：SpringMVC 会在初始化容器的时候，将 `ServletContext` 对象存储到 IOC 容器中！

```java
@Autowired
private ServletContext servletContext;

@RequestMapping("/attr/application")
@ResponseBody
public String attrApplication() {

    servletContext.setAttribute("appScopeMsg", "i am hungry...");

    return "target";
}
```

---

## 三、SpringMVC 响应数据

### (一) handler 方法分析

理解 handler 方法的作用和组成：

```java
/**
 * TODO: 一个 controller 的方法是控制层的一个处理器，我们称为 handler
 * TODO: handler 需要使用 @RequestMapping/@GetMapping 系列，声明路径，
 *       在 HandlerMapping 中注册，供 DispatcherServlet 查找！
 * TODO: handler 作用总结：
 *       1. 接收请求参数(param, json, pathVariable, 共享域等)
 *       2. 调用业务逻辑
 *       3. 响应前端数据(页面、json、转发和重定向等)
 * TODO: handler 如何处理呢
 *       1. 接收参数：handler(形参列表: 主要的作用就是用来接收参数)
 *       2. 调用业务：{ 方法体 可以向后调用业务方法 service.xx() }
 *       3. 响应数据：return 返回结果，可以快速响应前端数据
 */
@GetMapping
public Object handler(简化请求参数接收) {
    调用业务方法
    return 简化响应前端数据;
}
```

> 总结：**请求数据接收，我们都是通过 handler 的形参列表；前端数据响应，我们都是通过 handler 的 return 关键字快速处理！** SpringMVC 简化了参数接收和响应。

### (二) 页面跳转控制

#### 开发模式回顾

在 Web 开发中，有两种主要的开发模式：

| 模式 | 说明 |
|---|---|
| **前后端分离模式（重点）** | 将前端界面和后端业务逻辑通过接口分离开发。前后端使用不同技术栈，前端负责页面呈现与用户交互，后端负责业务逻辑和数据存储。通信通过 API 接口完成，数据格式一般用 JSON 或 XML。可提高开发效率，有助于代码重用和维护 |
| **混合开发模式** | 将前端和后端的代码集成在同一个项目中，共享相同的技术栈和框架。在小型项目中比较常见，可减少学习成本和部署难度。但在大型项目中会导致代码耦合性很高，维护和升级难度较大 |

> 对于**混合开发**，我们就需要使用动态页面技术，动态展示 Java 的共享域数据！

#### jsp 技术了解

JSP（JavaServer Pages）是一种动态网页开发技术，由 Sun 公司提出，可以在 HTML 文件中嵌入 Java 代码，使得生成动态内容的编写更加简单。

**主要特点**：

1. **简单**：通过将 Java 代码嵌入 HTML 页面，使动态内容编写更简单
2. **高效**：首次运行时会被转换为 Servlet，然后编译为字节码，可启用 JIT 编译器
3. **多样化**：支持 JSTL、EL 等多种标准标签库

#### 准备 jsp 页面和依赖

`pom.xml` 依赖：

```xml
<!-- jsp 需要依赖! jstl -->
<dependency>
    <groupId>jakarta.servlet.jsp.jstl</groupId>
    <artifactId>jakarta.servlet.jsp.jstl-api</artifactId>
    <version>3.0.0</version>
</dependency>
```

创建 jsp 页面，**建议位置 `/WEB-INF/` 下，避免外部直接访问**：

位置：`/WEB-INF/views/home.jsp`

```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
  <head>
    <title>Title</title>
  </head>
  <body>
        <!-- 可以获取共享域的数据，动态展示！jsp == 后台的 vue -->
        ${msg}
  </body>
</html>
```

#### 快速响应模版页面

**1. 配置 jsp 视图解析器**

```java
@EnableWebMvc
@Configuration
@ComponentScan(basePackages = "com.atguigu.controller")
public class SpringMvcConfig implements WebMvcConfigurer {

    // 配置 jsp 对应的视图解析器
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        // 快速配置 jsp 模板语言对应的
        registry.jsp("/WEB-INF/views/", ".jsp");
    }
}
```

**2. handler 返回视图**

```java
/**
 *  跳转到提交文件页面  /save/jump
 *
 *  如果要返回 jsp 页面！
 *     1. 方法返回值改成字符串类型
 *     2. 返回逻辑视图名即可
 *         prefix(/WEB-INF/views/) + 逻辑视图名 + suffix(.jsp)
 */
@GetMapping("jump")
public String jumpJsp(Model model) {
    System.out.println("FileController.jumpJsp");
    model.addAttribute("msg", "request data!!");
    return "home";
}
```

### (三) 转发和重定向

在 Spring MVC 中，Handler 方法返回值来实现快速转发，可以使用 `redirect` 或者 `forward` 关键字来实现重定向。

```java
@RequestMapping("/redirect-demo")
public String redirectDemo() {
    // 重定向到 /demo 路径
    return "redirect:/demo";
}

@RequestMapping("/forward-demo")
public String forwardDemo() {
    // 转发到 /demo 路径
    return "forward:/demo";
}
```

> ⚠️ 注意：**转发和重定向到项目下资源路径都是相同的，都不需要添加项目根路径**！填写项目下路径即可。

总结：

- 将方法的返回值设置为 `String` 类型
- 转发使用 `forward` 关键字，重定向使用 `redirect` 关键字
- 关键字写法：`关键字: /路径`
- 如果是项目下的资源，转发和重定向都一样，都是项目下路径，都不需要添加项目根路径

### (四) 返回 JSON 数据（重点）

#### 前置准备

**1. 导入 jackson 依赖**

```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.15.0</version>
</dependency>
```

**2. 添加 json 数据转化器**

```java
@EnableWebMvc  // json 数据处理，必须使用此注解，因为它会加入 json 处理器
@Configuration
@ComponentScan(basePackages = "com.atguigu.controller")
public class SpringMvcConfig implements WebMvcConfigurer {

}
```

#### `@ResponseBody`

可以在方法上使用 `@ResponseBody` 注解，用于将方法返回的对象序列化为 JSON 或 XML 格式的数据，并发送给客户端。**在前后端分离的项目中使用！**

```java
@GetMapping("/accounts/{id}")
@ResponseBody
public Object handle() {
  // ...
  return obj;
}
```

具体来说，`@ResponseBody` 注解可以用来标识方法或者方法返回值，表示**方法的返回值是要直接返回给客户端的数据，而不是由视图解析器来解析并渲染生成响应体**（`ViewResolver` 没用了）。

```java
@RequestMapping(value = "/user/detail", method = RequestMethod.POST)
@ResponseBody
public User getUser(@RequestBody User userParam) {
    System.out.println("userParam = " + userParam);
    User user = new User();
    user.setAge(18);
    user.setName("John");
    // 返回的对象，会使用 jackson 的序列化工具，转成 json 返回给前端！
    return user;
}
```

如果类中每个方法上都标记了 `@ResponseBody` 注解，那么这些注解就可以提取到类上：

```java
@ResponseBody  // responseBody 可以添加到类上，代表默认类中的所有方法都生效！
@Controller
@RequestMapping("param")
public class ParamController {

}
```

#### `@RestController`

类上的 `@ResponseBody` 注解可以和 `@Controller` 注解合并为 `@RestController` 注解。所以使用了 `@RestController` 注解就相当于给类中的每个方法都加了 `@ResponseBody` 注解。

`RestController` 源码：

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Controller
@ResponseBody
public @interface RestController {

  /**
   * The value may indicate a suggestion for a logical component name,
   * to be turned into a Spring bean in case of an autodetected component.
   */
  @AliasFor(annotation = Controller.class)
  String value() default "";
}
```

### (五) 返回静态资源处理

**1. 静态资源概念**

资源本身已经是可以直接拿到浏览器上使用的程度了，**不需要在服务器端做任何运算、处理**。典型的静态资源包括：

- 纯 HTML 文件
- 图片
- CSS 文件
- JavaScript 文件

**2. 静态资源访问和问题解决**

**问题分析**：

- `DispatcherServlet` 的 `url-pattern` 配置的是 `/`
- `url-pattern` 配置 `/` 表示整个 Web 应用范围内所有请求都由 SpringMVC 来处理
- 对 SpringMVC 来说，必须有对应的 `@RequestMapping` 才能找到处理请求的方法
- 现在 `images/mi.jpg` 请求没有对应的 `@RequestMapping`，所以返回 404

**问题解决**：在 SpringMVC 配置类中开启默认 Servlet 处理。

```java
@EnableWebMvc
@Configuration
@ComponentScan(basePackages = "com.atguigu.controller")
public class SpringMvcConfig implements WebMvcConfigurer {

    // 配置 jsp 对应的视图解析器
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
    }

    // 开启静态资源处理 <mvc:default-servlet-handler/>
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }
}
```

> ⚠️ **新的问题**：开启后其他原本正常的 handler 请求访问不了了。解决方案是补上 `@EnableWebMvc` 注解。

---

## 四、RESTFul 风格设计和实战

### (一) RESTFul 风格概述

#### 简介

RESTful（Representational State Transfer）是一种**软件架构风格**，用于设计网络应用程序和服务之间的通信。它是一种基于标准 HTTP 方法的简单和轻量级的通信协议，广泛应用于现代的 Web 服务开发。

通过遵循 RESTful 架构的设计原则，可以构建出**易于理解、可扩展、松耦合和可重用**的 Web 服务。RESTful API 的特点是简单、清晰、易于使用和理解，它们使用标准的 HTTP 方法和状态码进行通信，**不需要额外的协议和中间件**。

> 总而言之：RESTful 是一种基于 HTTP 和标准化设计原则的软件架构风格，用于设计和实现可靠、可扩展和易于集成的 Web 服务和应用程序。学习 RESTful 设计原则可以帮助我们更好地设计 HTTP 协议的 API 接口！

#### 特点

1. **每一个 URI 代表 1 种资源**（URI 是名词）
2. 客户端使用 **GET、POST、PUT、DELETE** 4 个表示操作方式的动词对服务端资源进行操作：GET 用来获取资源，POST 用来新建资源（也可以用于更新资源），PUT 用来更新资源，DELETE 用来删除资源
3. 资源的表现形式是 **XML 或者 JSON**
4. 客户端与服务端之间的交互在请求之间是**无状态的**，从客户端到服务端的每个请求都必须包含理解请求所必需的信息

#### 设计规范

**1. HTTP 协议请求方式要求**

REST 风格主张在项目设计、开发过程中，具体的操作符合 HTTP 协议定义的请求方式的语义。

| 操作 | 请求方式 |
|---|---|
| 查询操作 | GET |
| 保存操作 | POST |
| 删除操作 | DELETE |
| 更新操作 | PUT |

**2. URL 路径风格要求**

REST 风格下每个资源都应该有一个唯一的标识符，例如一个 URI 或 URL。资源的标识符应该能明确地说明该资源的信息，同时也应该是可被理解和解释的。

使用 **URL + 请求方式**确定具体的动作，它也是一种标准的 HTTP 协议请求：

| 操作 | 传统风格 | REST 风格 |
|---|---|---|
| 保存 | `/CRUD/saveEmp` | URL：`/CRUD/emp` 请求方式：POST |
| 删除 | `/CRUD/removeEmp?empId=2` | URL：`/CRUD/emp/2` 请求方式：DELETE |
| 更新 | `/CRUD/updateEmp` | URL：`/CRUD/emp` 请求方式：PUT |
| 查询 | `/CRUD/editEmp?empId=2` | URL：`/CRUD/emp/2` 请求方式：GET |

总结：

- 根据接口的具体动作，选择具体的 HTTP 协议请求方式
- 路径设计从原来携带动标识（saveEmp、removeEmp），改成名词（emp），对应资源的唯一标识即可

#### 好处

1. **含蓄，安全**：使用问号键值对的方式给服务器传递数据太明显，容易被人利用来对系统进行破坏。使用 REST 风格携带数据不再需要明显地暴露数据的名称
2. **风格统一**：URL 地址整体格式统一，从前到后始终都使用斜杠划分各个单词，用简单一致的格式表达语义
3. **无状态**：在调用一个接口（访问、操作资源）的时候，可以不用考虑上下文，不用考虑当前状态，极大地降低了系统设计的复杂度
4. **严谨，规范**：严格按照 HTTP1.1 协议中定义的请求方式本身的语义进行操作
5. **简洁，优雅**：过去做增删改查操作需要设计 4 个不同的 URL，现在一个就够了
6. **丰富的语义**：通过 URL 地址就可以知道资源之间的关系。它能够把一句话中的很多单词用斜杠连起来，反过来说就是可以在 URL 地址中用一句话来充分表达语义

```
http://localhost:8080/shop
http://localhost:8080/shop/product
http://localhost:8080/shop/product/cellPhone
http://localhost:8080/shop/product/cellPhone/iPhone
```

### (二) RESTFul 风格实战

#### 需求分析

- **数据结构**：`User { id 唯一标识, name 用户名, age 用户年龄 }`
- **功能分析**：
  - 用户数据分页展示功能（条件：page 页数 默认 1，size 每页数量 默认 10）
  - 保存用户功能
  - 根据用户 id 查询用户详情功能
  - 根据用户 id 更新用户数据功能
  - 根据用户 id 删除用户数据功能
  - 多条件模糊查询用户功能（条件：keyword 模糊关键字，page 页数 默认 1，size 每页数量 默认 10）

#### 接口设计

| 功能 | 接口和请求方式 | 请求参数 | 返回值 |
|---|---|---|---|
| 分页查询 | GET `/user` | `page=1&size=10` | { 响应数据 } |
| 用户添加 | POST `/user` | { user 数据 } | { 响应数据 } |
| 用户详情 | GET `/user/1` | 路径参数 | { 响应数据 } |
| 用户更新 | PUT `/user` | { user 更新数据 } | { 响应数据 } |
| 用户删除 | DELETE `/user/1` | 路径参数 | { 响应数据 } |
| 条件模糊 | GET `/user/search` | `page=1&size=10&keyword=关键字` | { 响应数据 } |

#### 问题讨论：路径参数还是请求参数？

> **误区**：RESTful 风格下，不是所有请求参数都是路径传递！可以使用其他方式传递！

在 RESTful API 的设计中，路径、请求参数和请求体都是用来向服务器传递信息的方式。

| 场景 | 用哪种 | 原因 |
|---|---|---|
| **查询用户详情** | **路径参数** `/user/1` | 这是一个单一资源的查询，即查询一条用户记录。使用路径参数可以明确指定所请求的资源，便于服务器定位并返回对应的资源，也符合 RESTful 风格的要求 |
| **多条件模糊查询** | **请求参数** `/user/search?keyword=x` | 这是一个资源集合的查询，即查询多条用户记录。使用请求参数可以通过组合不同参数来限制查询结果，路径参数的组合和排列可能会很多，不如请求参数灵活简洁 |

此外，还有一些通用的原则可以遵循：

- **路径参数应该用于指定资源的唯一标识或者 ID，请求参数应该用于指定查询条件或者操作参数**
- 请求参数应该限制在 10 个以内，过多的请求参数可能导致接口难以维护和使用
- 对于敏感信息，最好使用 POST 和请求体来传递参数

#### 后台接口实现

**准备用户实体类**

```java
package com.atguigu.pojo;

/**
 * 用户实体类
 */
public class User {

    private Integer id;
    private String name;

    private Integer age;

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}
```

**准备用户 Controller**

```java
/**
 * 用户模块的控制器
 */
@RequestMapping("user")
@RestController
public class UserController {

    /**
     * 模拟分页查询业务接口
     */
    @GetMapping
    public Object queryPage(@RequestParam(name = "page", required = false, defaultValue = "1") int page,
                            @RequestParam(name = "size", required = false, defaultValue = "10") int size) {
        System.out.println("page = " + page + ", size = " + size);
        System.out.println("分页查询业务!");
        return "{'status':'ok'}";
    }

    /**
     * 模拟用户保存业务接口
     */
    @PostMapping
    public Object saveUser(@RequestBody User user) {
        System.out.println("user = " + user);
        System.out.println("用户保存业务!");
        return "{'status':'ok'}";
    }

    /**
     * 模拟用户详情业务接口
     */
    @PostMapping("/{id}")
    public Object detailUser(@PathVariable Integer id) {
        System.out.println("id = " + id);
        System.out.println("用户详情业务!");
        return "{'status':'ok'}";
    }

    /**
     * 模拟用户更新业务接口
     */
    @PutMapping
    public Object updateUser(@RequestBody User user) {
        System.out.println("user = " + user);
        System.out.println("用户更新业务!");
        return "{'status':'ok'}";
    }

    /**
     * 模拟条件分页查询业务接口
     */
    @GetMapping("search")
    public Object queryPage(@RequestParam(name = "page", required = false, defaultValue = "1") int page,
                            @RequestParam(name = "size", required = false, defaultValue = "10") int size,
                            @RequestParam(name = "keyword", required = false) String keyword) {
        System.out.println("page = " + page + ", size = " + size + ", keyword = " + keyword);
        System.out.println("条件分页查询业务!");
        return "{'status':'ok'}";
    }
}
```

---

## 五、SpringMVC 其他扩展

### (一) 全局异常处理机制

开发过程中不可避免会出现各种异常情况，例如网络连接异常、数据格式异常、空指针异常等等。异常的出现可能导致程序的运行出现问题，甚至直接导致程序崩溃。因此，合理处理异常、避免异常产生、以及对异常进行有效的调试是非常重要的。

#### 异常处理两种方式

| 方式 | 特点 |
|---|---|
| **编程式异常处理** | 在代码中显式地编写处理异常的逻辑，通常涉及对异常类型的检测及其处理，例如使用 `try-catch` 块捕获异常，在 `catch` 块中编写特定的处理代码，或者在 `finally` 块中执行清理操作。**异常处理代码混杂在业务代码中，导致代码可读性较差** |
| **声明式异常处理** | 将异常处理的逻辑从具体的业务逻辑中分离出来，通过配置等方式进行统一的管理和处理。开发人员只需要为方法或类标注相应的注解（如 `@ExceptionHandler`）即可。**使代码更加简洁、易于维护和扩展** |

**站在宏观角度来看待声明式异常处理**：

整个项目从架构这个层面设计的异常处理的统一机制和规范。

一个项目中会包含很多个模块，各个模块需要分工完成。如果张三负责的模块按照 A 方案处理异常，李四负责的模块按照 B 方案处理异常……各个模块处理异常的思路、代码、命名细节都不一样，那么就会让整个项目非常混乱。

**使用声明式异常处理，可以统一项目处理异常思路，项目更加清晰明了！**

#### 基于注解异常声明异常处理

**1. 声明异常处理控制器类**

```java
/**
 * 全局异常处理器，内部可以定义异常处理 Handler
 *
 * @RestControllerAdvice = @ControllerAdvice + @ResponseBody
 * @ControllerAdvice 代表当前类的异常处理 controller
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

}
```

**2. 声明异常处理 handler 方法**

> 异常处理 handler 方法和普通的 handler 方法参数接收和响应都一致！
> 只不过**异常处理 handler 方法要映射异常**，发生对应的异常会调用；
> 普通的 handler 方法要使用 `@RequestMapping` 注解映射路径，发生对应的路径调用。

```java
/**
 * 异常处理 handler
 * @ExceptionHandler(HttpMessageNotReadableException.class)
 * 该注解标记异常处理 Handler，并且指定发生异常调用该方法
 *
 * @param e 获取异常对象
 * @return 返回 handler 处理结果
 */
@ExceptionHandler(HttpMessageNotReadableException.class)
public Object handlerJsonDateException(HttpMessageNotReadableException e) {
    return null;
}

/**
 * 当发生空指针异常会触发此方法
 */
@ExceptionHandler(NullPointerException.class)
public Object handlerNullException(NullPointerException e) {
    return null;
}

/**
 * 所有异常都会触发此方法！但是如果有具体的异常处理 Handler！
 * 具体异常处理 Handler 优先级更高！
 * 例如：发生 NullPointerException 异常，
 *       会触发 handlerNullException 方法，不会触发 handlerException 方法！
 */
@ExceptionHandler(Exception.class)
public Object handlerException(Exception e) {
    return null;
}
```

> ⚠️ **优先级**：具体异常 > 兜底 `Exception`。

**3. 配置文件扫描控制器类配置**

确保异常处理控制类被扫描：

```java
// 扫描 controller 对应的包，将 handler 加入到 ioc
@ComponentScan(basePackages = {"com.atguigu.controller",
                               "com.atguigu.exceptionhandler"})
```

### (二) 拦截器使用

#### 拦截器概念

**拦截器和过滤器解决的问题**

- **生活中**：为了提高乘车效率，在乘客进入站台前统一检票
- **程序中**：使用拦截器在请求到达具体 handler 方法前，统一执行检测

**拦截器 SpringMVC vs 过滤器 javaWeb**

| 对比维度 | 说明 |
|---|---|
| **相似点** | |
| 拦截 | 必须先把请求拦住，才能执行后续操作 |
| 过滤 | 拦截器或过滤器存在的意义就是对请求进行统一处理 |
| 放行 | 对请求执行了必要操作后，放请求过去，让它访问原本想要访问的资源 |
| **不同点** | |
| 工作平台不同 | 过滤器工作在 **Servlet 容器**中；拦截器工作在 **SpringMVC** 的基础上 |
| 拦截的范围 | 过滤器：能够拦截到的最大范围是**整个 Web 应用**；拦截器：能够拦截到的最大范围是**整个 SpringMVC 负责的请求** |
| IOC 容器支持 | 过滤器：想得到 IOC 容器需要调用专门的工具方法，是**间接**的；拦截器：它自己就在 IOC 容器中，可以直接从 IOC 容器中装配组件，也就是**可以直接得到 IOC 容器的支持** |

> **选择**：功能需要如果用 SpringMVC 的拦截器能够实现，就不使用过滤器。

#### 拦截器使用

**1. 创建拦截器类**

```java
public class Process01Interceptor implements HandlerInterceptor {

    // if( ! preHandler()){return;}
    // 在处理请求的目标 handler 方法前执行
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("request = " + request + ", response = " + response + ", handler = " + handler);
        System.out.println("Process01Interceptor.preHandle");

        // 返回 true：放行
        // 返回 false：不放行
        return true;
    }

    // 在目标 handler 方法之后，handler 报错不执行！
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
        System.out.println("Process01Interceptor.postHandle");
    }

    // 渲染视图之后执行(最后)，一定执行！
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) throws Exception {
        System.out.println("Process01Interceptor.afterCompletion");
    }
}
```

**三个方法的执行时机对比**

| 方法 | 执行时机 | 返回值 |
|---|---|---|
| `preHandle` | handler 方法**之前** | `true` 放行 / `false` 不放行 |
| `postHandle` | handler 方法**之后**，**handler 报错则不执行** | 无 |
| `afterCompletion` | 视图渲染之后（最后），**一定执行** | 无 |

**2. 修改配置类添加拦截器**

```java
@EnableWebMvc
@Configuration
@ComponentScan(basePackages = {"com.atguigu.controller", "com.atguigu.exceptionhandler"})
public class SpringMvcConfig implements WebMvcConfigurer {

    // 配置 jsp 对应的视图解析器
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        registry.jsp("/WEB-INF/views/", ".jsp");
    }

    // 开启静态资源处理 <mvc:default-servlet-handler/>
    @Override
    public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
        configurer.enable();
    }

    // 添加拦截器
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        // 将拦截器添加到 Springmvc 环境，默认拦截所有 Springmvc 分发的请求
        registry.addInterceptor(new Process01Interceptor());
    }
}
```

**3. 配置详解**

**(1) 默认拦截全部**

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    // 将拦截器添加到 Springmvc 环境，默认拦截所有 Springmvc 分发的请求
    registry.addInterceptor(new Process01Interceptor());
}
```

**(2) 精准配置**

```java
@Override
public void addInterceptors(InterceptorRegistry registry) {
    // 精准匹配，设置拦截器处理指定请求，路径可以设置一个或者多个，为项目下路径即可
    // addPathPatterns("/common/request/one") 添加拦截路径
    // 也支持 /* 和 /** 模糊路径：* 任意一层字符串，** 任意层任意字符串
    registry.addInterceptor(new Process01Interceptor())
            .addPathPatterns("/common/request/one", "/common/request/tow");
}
```

**(3) 排除配置**

```java
// 添加拦截器
@Override
public void addInterceptors(InterceptorRegistry registry) {
    // 排除匹配，排除应该在匹配的范围内排除
    // addPathPatterns("/common/request/one") 添加拦截路径
    // excludePathPatterns("/common/request/tow") 排除路径，排除应该在拦截的范围内
    registry.addInterceptor(new Process01Interceptor())
            .addPathPatterns("/common/request/one", "/common/request/tow")
            .excludePathPatterns("/common/request/tow");
}
```

> ⚠️ **排除路径必须在拦截路径的范围内**，否则排除不生效。

**4. 多个拦截器执行顺序**

| 方法 | 执行顺序 |
|---|---|
| `preHandle()` | 按配置顺序**正序**调用 |
| `postHandle()` | 按配置顺序**逆序**调用 |
| `afterCompletion()` | 按配置顺序**逆序**调用 |

### (三) 参数校验

> 在 Web 应用三层架构体系中，表述层负责接收浏览器提交的数据，业务逻辑层负责数据的处理。为了能够让业务逻辑层基于正确的数据进行处理，我们需要在表述层对数据进行检查，**将错误的数据隔绝在业务逻辑层之外**。

#### 校验概述

**JSR 303** 是 Java 为 Bean 数据合法性校验提供的标准框架，它已经包含在 JavaEE 6.0 标准中。JSR 303 通过在 Bean 属性上标注类似于 `@NotNull`、`@Max` 等标准的注解指定校验规则，并通过标准的验证接口对 Bean 进行验证。

**JSR 303 标准注解**

| 注解 | 规则 |
|---|---|
| `@Null` | 标注值必须为 null |
| `@NotNull` | 标注值不可为 null |
| `@AssertTrue` | 标注值必须为 true |
| `@AssertFalse` | 标注值必须为 false |
| `@Min(value)` | 标注值必须大于或等于 value |
| `@Max(value)` | 标注值必须小于或等于 value |
| `@DecimalMin(value)` | 标注值必须大于或等于 value |
| `@DecimalMax(value)` | 标注值必须小于或等于 value |
| `@Size(max, min)` | 标注值大小必须在 max 和 min 限定的范围内 |
| `@Digits(integer, fraction)` | 标注值必须是一个数字，且必须在可接受的范围内 |
| `@Past` | 标注值只能用于日期型，且必须是过去的日期 |
| `@Future` | 标注值只能用于日期型，且必须是将来的日期 |
| `@Pattern(value)` | 标注值必须符合指定的正则表达式 |

JSR 303 只是一套标准，需要提供其实现才可以使用。**Hibernate Validator** 是 JSR 303 的一个参考实现，除支持所有标准的校验注解外，它还支持以下的扩展注解：

| 注解 | 规则 |
|---|---|
| `@Email` | 标注值必须是格式正确的 Email 地址 |
| `@Length` | 标注值字符串大小必须在指定的范围内 |
| `@NotEmpty` | 标注值字符串不能是空字符串 |
| `@Range` | 标注值必须在指定的范围内 |

Spring 4.0 版本已经拥有自己独立的数据校验框架，同时支持 JSR 303 标准的校验框架。Spring 在进行数据绑定时，可同时调用校验框架完成数据校验工作。在 SpringMVC 中，可直接通过注解驱动 `@EnableWebMvc` 的方式进行数据校验。

Spring 的 `LocalValidatorFactoryBean` 既实现了 Spring 的 `Validator` 接口，也实现了 JSR 303 的 `Validator` 接口。只要在 Spring 容器中定义了一个 `LocalValidatorFactoryBean`，即可将其注入到需要数据校验的 Bean 中。**Spring 本身并没有提供 JSR 303 的实现，所以必须将 JSR 303 实现者的 jar 包放到类路径下。**

配置 `@EnableWebMvc` 后，SpringMVC 会默认装配好一个 `LocalValidatorFactoryBean`，通过在处理方法的入参上标注 `@Validated` 注解即可让 SpringMVC 在完成数据绑定后执行数据校验的工作。

#### 操作演示

**1. 导入依赖**

```xml
<!-- 校验注解 -->
<dependency>
    <groupId>jakarta.platform</groupId>
    <artifactId>jakarta.jakartaee-web-api</artifactId>
    <version>9.1.0</version>
    <scope>provided</scope>
</dependency>

<!-- 校验注解实现 -->
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>8.0.0.Final</version>
</dependency>
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator-annotation-processor</artifactId>
    <version>8.0.0.Final</version>
</dependency>
```

**2. 应用校验注解**

```java
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Min;
import org.hibernate.validator.constraints.Length;

public class User {
    // age   1 <= age <= 150
    @Min(10)
    private int age;

    // name 3 <= name.length <= 6
    @Length(min = 3, max = 10)
    private String name;

    // email 邮箱格式
    @Email
    private String email;

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }
}
```

**3. handler 标记和绑定错误收集**

```java
@RestController
@RequestMapping("user")
public class UserController {

    /**
     * @Validated 代表应用校验注解！必须添加！
     */
    @PostMapping("save")
    public Object save(@Validated @RequestBody User user,
                       // 在实体类参数和 BindingResult 之间不能有任何其他参数，
                       // BindingResult 可以接受错误信息，避免信息抛出！
                       BindingResult result) {
        // 判断是否有信息绑定错误！有可以自行处理！
        if (result.hasErrors()) {
            System.out.println("错误");
            String errorMsg = result.getFieldError().toString();
            return errorMsg;
        }
        // 没有，正常处理业务即可
        System.out.println("正常");
        return user;
    }
}
```

> ⚠️ **`BindingResult` 必须紧跟着被校验的实体参数**，中间不能有任何其他参数。

#### 易混总结：三个"非空"注解

`@NotNull`、`@NotEmpty`、`@NotBlank` 都是用于在数据校验中检查字段值是否为空的注解，但是它们的用法和校验规则有所不同。

**1. `@NotNull`（包装类型不为 null）**

`@NotNull` 注解是 JSR 303 规范中定义的注解，当被标注的字段值为 null 时，会认为校验失败而抛出异常。

> ⚠️ **该注解不能用于字符串类型的校验**，若要对字符串进行校验，应该使用 `@NotBlank` 或 `@NotEmpty` 注解。

**2. `@NotEmpty`（集合类型长度大于 0）**

`@NotEmpty` 注解同样是 JSR 303 规范中定义的注解，对于 `CharSequence`、`Collection`、`Map` 或者数组对象类型的属性进行校验，校验时会检查该属性是否为 null 或者 `size() == 0`，如果是的话就会校验失败。

> ⚠️ 对于其他类型的属性，该注解无效。需要注意的是**只校验空格前后的字符串，如果该字符串中间只有空格，不会被认为是空字符串，校验不会失败**。

**3. `@NotBlank`（字符串，不为 null，且不为 `" "` 字符串）**

`@NotBlank` 注解是 Hibernate Validator 附加的注解，对于字符串类型的属性进行校验，校验时会检查该属性是否为 null 或 `""` 或者只包含空格，如果是的话就会校验失败。

> ⚠️ `@NotBlank` 注解**只能用于字符串类型的校验**。

**三者对比表**

| 注解 | 来源 | 适用类型 | 校验规则 |
|---|---|---|---|
| `@NotNull` | JSR 303 | 任意类型 | 仅判断 `!= null`，不校验字符串内容 |
| `@NotEmpty` | JSR 303 | CharSequence / Collection / Map / 数组 | `!= null` 且 `size() > 0`，纯空格字符串能通过 |
| `@NotBlank` | Hibernate Validator | **仅字符串** | `!= null`、`!= ""`、不能只包含空格 |

---

## 六、SpringMVC 总结

### (一) 核心知识点

| 核心点 | 掌握目标 |
|---|---|
| SpringMVC 框架 | 主要作用、核心组件、调用流程 |
| 简化参数接收 | 路径设计、参数接收、请求头接收、cookie 接收 |
| 简化数据响应 | 模板页面、转发和重定向、JSON 数据、静态资源 |
| RESTFul 风格设计 | 主要作用、具体规范、请求方式和请求参数选择 |
| 功能扩展 | 全局异常处理、拦截器、参数校验注解 |

### (二) 参数接收注解速查

| 注解 | 作用 | 对应内容 |
|---|---|---|
| `@RequestMapping` | 建立 URL 与 handler 的映射 | 路径 |
| `@RequestParam` | 绑定请求参数 | param 参数（未在本章详述，见上文 2.2） |
| `@PathVariable` | 绑定 URL 占位符 | 路径参数 |
| `@RequestBody` | 读取请求体并反序列化 | JSON 参数 |
| `@CookieValue` | 绑定 Cookie 值 | Cookie |
| `@RequestHeader` | 绑定请求头值 | Header |
| `@Validated` | 开启参数校验 | JSR 303 校验 |

### (三) 关键避坑清单

| 坑 | 现象 | 解决 |
|---|---|---|
| 用了旧版 Servlet API | `DispatcherServlet is not assignable to javax.servlet.Servlet` | Spring 6 迁移到 Jakarta EE，依赖换 `jakarta.jakartaee-web-api`，Tomcat 用 **10+** |
| 缺 `@EnableWebMvc` 或 jackson | `HttpMediaTypeNotSupportedException: Content-Type 'application/json' is not supported` | 配置类加 `@EnableWebMvc`，引入 `jackson-databind` |
| 违背请求方式 | **405 异常** | 检查 handler 上的 `method` 限制 |
| 多个 handler 映射同一地址 | `There is already 'xxx' bean method ... mapped` | 保证路径 + 请求方式唯一 |
| `DispatcherServlet` 映射 `/` 后静态资源 404 | 图片/CSS 访问不到 | `configureDefaultServletHandling(configurer -> configurer.enable())` |
| 开启静态资源后 handler 全失效 | 原本正常的接口访问不了 | 补上 `@EnableWebMvc` |
| 进阶注解加到类上 | 编译/映射异常 | `@GetMapping` 等只能加在**方法**上 |
| 实体接收参数绑定不上 | 属性值为 null | 实体**属性名必须等于参数名** |
| `BindingResult` 位置不对 | 校验异常直接抛出 400 | `BindingResult` 必须**紧跟着**被校验的实体参数 |
| 全局异常处理器不生效 | 异常没被捕获 | 把异常处理类所在包加进 `@ComponentScan` |
| 拦截器排除路径不生效 | 排除的路径仍被拦截 | 排除路径必须在 `addPathPatterns` 的范围内 |
| 用了 `@NotNull` 校验字符串 | 空字符串能通过校验 | 字符串要用 `@NotBlank` |

### (四) 一句话总结

> SpringMVC 的本质是**用一个 `DispatcherServlet` 统一收口所有请求，再通过 `HandlerMapping` 找方法、`HandlerAdapter` 解析参数，最终把"取参数"和"返数据"两件事简化成"形参列表"和"return"**。RESTFul 决定了接口长什么样，拦截器负责统一拦截，全局异常处理器负责统一兜底，JSR 303 负责把脏数据挡在业务层之外。
