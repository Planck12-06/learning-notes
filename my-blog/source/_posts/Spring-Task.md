---
title: Spring Task 定时任务
date: 2026-08-09 22:00:19
categories:
  - Web后端
tags:
  - Spring Task
  - 定时任务
  - WebSocket
  - Apache POI
---

# Spring Task 定时任务

Spring Task 是 Spring 框架提供的任务调度工具，可以按照约定的时间自动执行某个代码逻辑。

## 特点

| 特点 | 说明 |
|:---|:---|
| 轻量级 | 无需引入额外依赖，Spring Boot 已内置 |
| 简单易用 | 通过注解即可配置定时任务 |
| Cron 表达式 | 支持灵活的时间调度规则 |

## 快速开始

### 第 1 步：开启定时任务功能

在启动类上添加 `@EnableScheduling` 注解：

```java
@EnableScheduling
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 第 2 步：编写定时任务方法

```java
@Component
public class MyTask {

    // 每 5 秒执行一次
    @Scheduled(cron = "0/5 * * * * ?")
    public void task1() {
        System.out.println("定时任务执行了...");
    }
}
```

---

# Cron 表达式

Cron 表达式是一个字符串，用于描述一个时间周期，用于定时任务的触发时间配置。

## 组成格式

```
秒 分 时 日 月 周 [年]
```

<table>
<thead>
<tr>
<th>字段</th>
<th>取值范围</th>
<th>允许的特殊字符</th>
</tr>
</thead>
<tbody>
<tr><td>秒（Seconds）</td><td>0 ~ 59</td><td>, - * /</td></tr>
<tr><td>分（Minutes）</td><td>0 ~ 59</td><td>, - * /</td></tr>
<tr><td>时（Hours）</td><td>0 ~ 23</td><td>, - * /</td></tr>
<tr><td>日（Day of Month）</td><td>1 ~ 31</td><td>, - * ? / L W C</td></tr>
<tr><td>月（Month）</td><td>1 ~ 12 或 JAN ~ DEC</td><td>, - * /</td></tr>
<tr><td>周（Day of Week）</td><td>1 ~ 7 或 SUN ~ SAT</td><td>, - * ? / L C #</td></tr>
<tr><td>年（Year，可选）</td><td>1970 ~ 2099</td><td>, - * /</td></tr>
</tbody>
</table>

## 特殊字符说明

<table>
<thead>
<tr>
<th>字符</th>
<th>含义</th>
<th>示例</th>
</tr>
</thead>
<tbody>
<tr><td><code>*</code></td><td>所有值（每...）</td><td><code>*</code> 在分字段表示每分钟</td></tr>
<tr><td><code>?</code></td><td>不指定值（仅用于日/周字段，必须有一个为 <code>?</code>）</td><td>-</td></tr>
<tr><td><code>-</code></td><td>范围</td><td><code>10-15</code> 表示 10 到 15</td></tr>
<tr><td><code>,</code></td><td>列举多个值</td><td><code>1,3,5</code> 表示第 1、3、5 个</td></tr>
<tr><td><code>/</code></td><td>递增步长</td><td><code>0/5</code> 从 0 开始每隔 5 执行一次</td></tr>
<tr><td><code>L</code></td><td>Last（最后一天/最后一周的某天）</td><td><code>L</code> 月最后一天</td></tr>
<tr><td><code>W</code></td><td>Workday（最近的工作日）</td><td><code>15W</code> 最近工作日到 15 号</td></tr>
<tr><td><code>#</code></td><td>第几周的星期几</td><td><code>6#3</code> 第 3 周的周五</td></tr>
</tbody>
</table>

## 常用表达式示例

<table>
<thead>
<tr>
<th>Cron 表达式</th>
<th>含义</th>
</tr>
</thead>
<tbody>
<tr><td><code>0 0 9 * * ?</code></td><td>每天上午 9 点整</td></tr>
<tr><td><code>0 0 9 * * MON-FRI</code></td><td>周一至周五上午 9 点整</td></tr>
<tr><td><code>0 0 0 1 * ?</code></td><td>每月 1 号零点</td></tr>
<tr><td><code>0/5 * * * * ?</code></td><td>每 5 秒执行一次</td></tr>
<tr><td><code>0 */10 * * * ?</code></td><td>每 10 分钟执行一次</td></tr>
<tr><td><code>0 30 8,18 * * ?</code></td><td>每天 8 点半和 18 点半</td></tr>
<tr><td><code>0 0 9 12 10 ?</code></td><td>每年 10 月 12 日上午 9 点整</td></tr>
<tr><td><code>0 0 9 L * ?</code></td><td>每月最后一天上午 9 点整</td></tr>
</tbody>
</table>

---

# WebSocket 协议

WebSocket 是基于 TCP 的一种新的网络协议，实现了浏览器与服务器全双工通信——浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并进行双向数据传输。

## WebSocket 与 HTTP 协议对比

<table>
<thead>
<tr>
<th>对比项</th>
<th>HTTP 协议</th>
<th>WebSocket 协议</th>
</tr>
</thead>
<tbody>
<tr><td>连接方式</td><td>短连接</td><td>长连接</td></tr>
<tr><td>通信模式</td><td>单向（请求-响应模式）</td><td>双向通信（基于帧的协议）</td></tr>
<tr><td>底层协议</td><td>TCP</td><td>TCP</td></tr>
<tr><td>适用场景</td><td>常规 Web 请求</td><td>实时消息推送、即时通讯</td></tr>
</tbody>
</table>

## WebSocket 使用示例

### 服务端

```java
@ServerEndpoint("/ws/{sid}")
@Component
public class WebSocketServer {

    // 存放会话对象
    private static Map<String, Session> sessionMap = new ConcurrentHashMap<>();

    // 连接建立成功调用的方法
    @OnOpen
    public void onOpen(Session session, @PathParam("sid") String sid) {
        sessionMap.put(sid, session);
    }

    // 收到客户端消息后调用的方法
    @OnMessage
    public void onMessage(String message, @PathParam("sid") String sid) {
        System.out.println("收到来自 " + sid + " 的消息：" + message);
    }

    // 连接关闭调用的方法
    @OnClose
    public void onClose(@PathParam("sid") String sid) {
        sessionMap.remove(sid);
    }
}
```

### 客户端

```javascript
let socket = new WebSocket("ws://localhost:8080/ws/" + sid);

// 连接建立事件
socket.onopen = function() {
    console.log("连接建立");
};

// 接收消息事件
socket.onmessage = function(event) {
    console.log("收到消息：" + event.data);
};

// 发送消息
socket.send("Hello Server");

// 关闭连接
socket.close();
```

---

# Apache POI

Apache POI 是一个开源 Java 库，用于在 Java 程序中对 Microsoft Office 各种文件进行读写操作。

## 核心组件

| 组件 | 说明 | 适用格式 |
|:---|:---|:---|
| HSSF | 操作 Excel 97-2003 格式 | `.xls` |
| XSSF | 操作 Excel 2007+ 格式 | `.xlsx` |
| HWPF | 操作 Word 文档 | `.doc` |
| XWPF | 操作 Word 2007+ 文档 | `.docx` |

## Maven 依赖

```xml
<dependency>
    <groupId>org.apache.poi</groupId>
    <artifactId>poi-ooxml</artifactId>
    <version>5.2.3</version>
</dependency>
```

## 读取 Excel 示例

```java
public List<User> importExcel(MultipartFile file) throws IOException {
    InputStream is = file.getInputStream();

    // 根据输入流创建工作簿对象
    Workbook workbook = new XSSFWorkbook(is);

    // 获取第一个 Sheet
    Sheet sheet = workbook.getSheetAt(0);

    List<User> userList = new ArrayList<>();

    // 遍历每一行数据
    for (int i = 1; i <= sheet.getLastRowNum(); i++) { // 跳过表头
        Row row = sheet.getRow(i);

        User user = new User();
        user.setName(row.getCell(0).getStringCellValue());
        user.setAge((int) row.getCell(1).getNumericCellValue());
        user.setPhone(row.getCell(2).getStringCellValue());

        userList.add(user);
    }

    workbook.close();
    return userList;
}
```

## 写入 Excel 示例

```java
public void exportExcel(HttpServletResponse response, List<User> list) throws IOException {
    // 创建工作簿
    Workbook workbook = new XSSFWorkbook();

    // 创建 Sheet
    Sheet sheet = workbook.createSheet("用户列表");

    // 创建表头行
    Row headerRow = sheet.createRow(0);
    headerRow.createCell(0).setCellValue("姓名");
    headerRow.createCell(1).setCellValue("年龄");
    headerRow.createCell(2).setCellValue("电话");

    // 填充数据
    for (int i = 0; i < list.size(); i++) {
        User user = list.get(i);
        Row dataRow = sheet.createRow(i + 1);
        dataRow.createCell(0).setCellValue(user.getName());
        dataRow.createCell(1).setCellValue(user.getAge());
        dataRow.createCell(2).setCellValue(user.getPhone());
    }

    // 设置响应头
    response.setContentType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet");
    response.setHeader("Content-Disposition", "attachment;filename=user.xlsx");

    // 输出流
    workbook.write(response.getOutputStream());
    workbook.close();
}
```
