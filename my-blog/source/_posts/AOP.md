---
title: AOP 面向切面编程
date: 2026-08-01 20:54:49
categories:
  - Web后端
tags:
  - AOP
  - 面向切面编程
  - Spring
---

# AOP 概述

AOP：Aspect Oriented Programming（面向切面编程、面向方面编程），面向切面编程就是面向特定方法编程。

## 优势

| 优势 | 说明 |
|:---|:---|
| **减少重复代码** | 不需要在业务方法中定义大量重复性代码，只需将重复代码抽取到 AOP 程序中即可 |
| **代码无侵入** | 基于 AOP 实现业务功能时，对原有业务代码无任何侵入，无需修改业务代码 |
| **提高开发效率** | 通用逻辑统一管理，减少重复劳动 |
| **维护方便** | 修改通用逻辑只需调整一处 |

## 核心概念

<table>
<thead>
<tr>
<th>概念</th>
<th>英文名</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td>连接点</td><td>JoinPoint</td><td>可以被 AOP 控制的方法（暗含方法执行时的相关信息）</td></tr>
<tr><td>通知</td><td>Advice</td><td>重复的逻辑，即需要增强的功能代码</td></tr>
<tr><td>切入点</td><td>PointCut</td><td>匹配连接点的条件，通知仅会在切入点方法执行时被应用</td></tr>
<tr><td>切面</td><td>Aspect</td><td>切入点 + 通知的组合</td></tr>
<tr><td>目标对象</td><td>Target</td><td>通知所应用的对象</td></tr>
</tbody>
</table>

---

# Spring AOP 通知类型

<table>
<thead>
<tr>
<th>注解</th>
<th>名称</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>@Around</code></td><td>环绕通知</td><td>此注解标注的通知方法在目标方法前、后都被执行</td></tr>
<tr><td><code>@Before</code></td><td>前置通知</td><td>此注解标注的通知方法在目标方法前被执行</td></tr>
<tr><td><code>@After</code></td><td>后置通知</td><td>此注解标注的通知方法在目标方法后被执行，无论是否有异常都会执行</td></tr>
<tr><td><code>@AfterReturning</code></td><td>返回后通知</td><td>此注解标注的通知方法在目标方法后被执行，有异常不会执行</td></tr>
<tr><td><code>@AfterThrowing</code></td><td>异常后通知</td><td>此注解标注的通知方法发生异常后执行</td></tr>
</tbody>
</table>

## 通知示例

```java
@Slf4j
@Component
@Aspect
public class MyAspect1 {

    // 前置通知
    @Before("execution(* com.itheima.service.*.*(..))")
    public void before(JoinPoint joinPoint) {
        log.info("before ...");
    }

    // 环绕通知
    @Around("execution(* com.itheima.service.*.*(..))")
    public Object around(ProceedingJoinPoint proceedingJoinPoint) throws Throwable {
        log.info("around before ...");

        // 调用目标对象的原始方法执行
        Object result = proceedingJoinPoint.proceed();

        // 原始方法如果执行时有异常，环绕通知中的后置代码不会再执行
        log.info("around after ...");
        return result;
    }

    // 后置通知
    @After("execution(* com.itheima.service.*.*(..))")
    public void after(JoinPoint joinPoint) {
        log.info("after ...");
    }

    // 返回后通知（程序在正常执行的情况下，会执行的后置通知）
    @AfterReturning("execution(* com.itheima.service.*.*(..))")
    public void afterReturning(JoinPoint joinPoint) {
        log.info("afterReturning ...");
    }

    // 异常通知（程序在出现异常的情况下，执行的后置通知）
    @AfterThrowing("execution(* com.itheima.service.*.*(..))")
    public void afterThrowing(JoinPoint joinPoint) {
        log.info("afterThrowing ...");
    }
}
```

---

# 切入点复用

由于每一个注解都写了相同的切点表达式，可以将切点表达式抽取到一个方法中，再使用 `@PointCut` 注解引用这个切点表达式。

```java
@PointCut("execution(* com.itheima.service.*.*(..))")
private void pointCut() {}

@Before("pointCut()")
public void before(JoinPoint joinPoint) {
    log.info("before...");
}
```

---

# 通知顺序

在不同切面类中，默认按照切面类的类名字母排序：

- 目标方法前的通知方法：字母排名靠前的先执行
- 目标方法后的通知方法：字母排名靠前的后执行

想控制通知的执行顺序有两种方式：

1. **修改切面类的类名**（这种方式非常繁琐、而且不便管理）
2. **使用 Spring 提供的 `@Order` 注解**（推荐）

```java
@Order(2)  // 数字越小优先级越高
@Component
@Aspect
public class MyAspect1 { ... }

@Order(1)
@Component
@Aspect
public class MyAspect2 { ... }
```

---

# 切点表达式

## 作用

主要用来决定项目中的哪些方法需要加入通知。

## 常见形式

| 形式 | 说明 |
|:---|:---|
| `execution(...)` | 根据方法的签名来匹配 |
| `@annotation(...)` | 根据注解匹配 |

## execution 切入点表达式

`execution` 主要根据方法的返回值、包名、类名、方法名、方法参数等信息来匹配。

### 语法格式

```
execution(访问修饰符? 返回值 包名.类名.?方法名(方法参数) throws 异常?)
```

其中带 `?` 的表示可以省略的部分。

### 通配符说明

<table>
<thead>
<tr>
<th>通配符</th>
<th>含义</th>
<th>示例</th>
</tr>
</thead>
<tbody>
<tr><td><code>*</code></td><td>单个独立的任意符号，可通配任意返回值、包名、类名、方法名、一个参数，或包/类/方法名的一部分</td><td><code>* com.itheima.service.*Service.*(..)</code></td></tr>
<tr><td><code>..</code></td><td>多个连续的任意符号，可通配任意层级的包，或任意类型、任意个数的参数</td><td><code>com..*Service.*(..)</code></td></tr>
</tbody>
</table>

### 语法规则

| 序号 | 规则 | 说明 |
|:---|:---|:---|
| 1 | 访问修饰符可省略 | 如 `public` 可省略不写 |
| 2 | 返回值可用 `*` 代替 | 表示任意返回值类型 |
| 3 | 包名可用 `*` 代替 | 代表任意包（一层包使用一个 `*`） |
| 4 | 使用 `..` 配置包名 | 表示此包以及此包下的所有子包 |
| 5 | 类名可用 `*` 代替 | 表示任意类 |
| 6 | 方法名可用 `*` 代替 | 表示任意方法 |
| 7 | 使用 `*` 配置参数 | 表示一个任意类型的参数 |
| 8 | 使用 `..` 配置参数 | 表示任意个任意类型的参数 |

### 示例对照表

<table>
<thead>
<tr>
<th>描述</th>
<th>表达式</th>
</tr>
</thead>
<tbody>
<tr><td>省略方法的修饰符号</td><td><code>execution(void com.itheima.service.impl.DeptServiceImpl.delete(java.lang.Integer))</code></td></tr>
<tr><td>使用 * 代替返回值类型</td><td><code>execution(* com.itheima.service.impl.DeptServiceImpl.delete(java.lang.Integer))</code></td></tr>
<tr><td>使用 * 代替包名（一层包使用一个 *）</td><td><code>execution(* com.itheima.*.*.DeptServiceImpl.delete(java.lang.Integer))</code></td></tr>
<tr><td>使用 .. 省略包名</td><td><code>execution(* com..DeptServiceImpl.delete(java.lang.Integer))</code></td></tr>
<tr><td>使用 * 代替类名</td><td><code>execution(* com..*.delete(java.lang.Integer))</code></td></tr>
<tr><td>使用 * 代替方法名</td><td><code>execution(* com..*.*(java.lang.Integer))</code></td></tr>
<tr><td>使用 * 代替参数</td><td><code>execution(* com.itheima.service.impl.DeptServiceImpl.delete(*))</code></td></tr>
<tr><td>使用 .. 代替参数（最常用）</td><td><code>execution(* com..*.*(..))</code></td></tr>
</tbody>
</table>

### 组合表达式

根据业务需要，可以使用 **且（&&）、或（\|\|）、非（!）** 来组合比较复杂的切入点表达式：

```java
// 匹配 list 或 delete 方法
execution(* com.itheima.service.DeptService.list(..)) || execution(* com.itheima.service.DeptService.delete(..))
```

### 书写建议

- 所有业务方法名在命名时尽量规范，方便切入点表达式快速匹配
- 例如：查询类方法都是 `find` 开头，更新类方法都是 `update` 开头

---

# @annotation 切入点

## 实现步骤

1. 编写自定义注解
2. 在业务类要作为连接点的方法上添加自定义注解

## 自定义注解 LogOperation

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogOperation {
}
```

## 业务类 DeptServiceImpl

```java
@Slf4j
@Service
public class DeptServiceImpl implements DeptService {

    @Autowired
    private DeptMapper deptMapper;

    @Override
    @LogOperation  // 自定义注解（表示：当前方法属于目标方法）
    public List<Dept> list() {
        List<Dept> deptList = deptMapper.list();
        // 模拟异常
        // int num = 10 / 0;
        return deptList;
    }

    @Override
    @LogOperation  // 自定义注解（表示：当前方法属于目标方法）
    public void delete(Integer id) {
        // 1. 删除部门
        deptMapper.delete(id);
    }

    @Override
    public void save(Dept dept) {
        dept.setCreateTime(LocalDateTime.now());
        dept.setUpdateTime(LocalDateTime.now());
        deptMapper.save(dept);
    }

    @Override
    public Dept getById(Integer id) {
        return deptMapper.getById(id);
    }

    @Override
    public void update(Dept dept) {
        dept.setUpdateTime(LocalDateTime.now());
        deptMapper.update(dept);
    }
}
```

## 切面类 MyAspect6

```java
@Slf4j
@Component
@Aspect
public class MyAspect6 {

    // 针对 list 方法、delete 方法进行前置通知和后置通知

    // 前置通知
    @Before("@annotation(com.itheima.anno.LogOperation)")
    public void before() {
        log.info("MyAspect6 -> before ...");
    }

    // 后置通知
    @After("@annotation(com.itheima.anno.LogOperation)")
    public void after() {
        log.info("MyAspect6 -> after ...");
    }
}
```

---

## execution 与 @annotation 对比

<table>
<thead>
<tr>
<th>对比项</th>
<th>execution</th>
<th>@annotation</th>
</tr>
</thead>
<tbody>
<tr><td>匹配方式</td><td>根据方法签名（返回值、包名、类名、方法名、参数）匹配</td><td>根据方法上的注解匹配</td></tr>
<tr><td>灵活性</td><td>表达式复杂，修改时需调整切面类</td><td>只需在目标方法上添加/移除注解</td></tr>
<tr><td>可读性</td><td>表达式较长，不易理解</td><td>语义清晰，一目了然</td></tr>
<tr><td>维护性</td><td>业务代码与切面耦合度高</td><td>业务代码与切面解耦</td></tr>
<tr><td>适用场景</td><td>批量匹配固定模式的方法</td><td>灵活选择需要增强的方法</td></tr>
<tr><td>推荐程度</td><td>适合简单场景</td><td>**推荐使用**（企业开发主流）</td></tr>
</tbody>
</table>

---

# 连接点 JoinPoint

在 Spring 中用 `JoinPoint` 抽象了连接点，通过它可以获得方法执行时的相关信息，如目标类名、方法名、方法参数等。

| 通知类型 | 连接点类型 | 说明 |
|:---|:---|:---|
| `@Around` | `ProceedingJoinPoint` | 必须使用此类型，才能调用 `proceed()` 执行原始方法 |
| 其他四种通知 | `JoinPoint` | 使用 `JoinPoint` 类型即可（`ProceedingJoinPoint` 是其子接口） |

---

# ThreadLocal

## 简介

ThreadLocal 并不是一个 Thread，而是 Thread 的局部变量。它会为每个线程提供一份单独的存储空间，具有线程隔离的效果，不同线程之间互不干扰。

## 常见方法

<table>
<thead>
<tr>
<th>方法</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>public void set(T value)</code></td><td>设置当前线程的线程局部变量的值</td></tr>
<tr><td><code>public T get()</code></td><td>返回当前线程所对应的线程局部变量的值</td></tr>
<tr><td><code>public void remove()</code></td><td>移除当前线程的线程局部变量</td></tr>
</tbody>
</table>

## 应用场景

>在同一个线程/同一个请求中进行数据共享时，就可以使用 ThreadLocal。
