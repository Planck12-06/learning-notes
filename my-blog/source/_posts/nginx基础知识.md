---
title: nginx基础知识
cover: /img/atri1.png
top_img: /img/atri1.png
date: 2026-09-12 11:38:14
categories:
  - 中间件
tags:
 - nginx
---

# Nginx简介

> Nginx 是一款**高性能的 HTTP 和反向代理服务器**，同时也是一个 IMAP / POP3 / SMTP 代理服务器。它以**高并发、低内存、稳定**著称，是后端部署中不可或缺的"流量入口"。

**核心特点**：

| 特点 | 说明 |
|---|---|
| 高并发 | 采用 **epoll 多路复用 + 异步非阻塞** 模型，单机可支撑数万并发连接 |
| 内存占用低 | 相比 Apache 的"一连接一进程/线程"，Nginx 更轻量 |
| 稳定性高 | 单个 worker 异常不影响整体，具备平滑重启能力 |
| 配置灵活 | 通过配置文件即可完成代理、均衡、缓存、限流等 |
| 跨平台 | Linux / Windows / macOS 均可运行，生产环境主要为 Linux |

**核心应用场景**（本笔记主线）：

```
客户端 → [ Nginx ] → 后端服务器集群
          │
          ├─ 正向代理：替【客户端】访问外网
          ├─ 反向代理：替【服务器】接收请求、转发给后端
          ├─ 负载均衡：把请求分摊到多个后端实例
          └─ 动静分离：静态资源 Nginx 直接返回，动态请求转发后端
```

---

## 正向代理

**正向代理**：代理服务器站在**客户端**一方，代替客户端去访问外部网络资源。客户端明确知道自己在使用代理，而**目标服务器不知道真正的请求者是谁**。

```
客户端 → 正向代理 → 目标服务器
        （替客户端出面）
```

**核心特点**：

- 代理的是**客户端**，隐藏的是**真实客户端**的身份
- 客户端需要**主动配置**代理地址
- 目标服务器只看到代理服务器的 IP，无法感知真实客户端

**典型用途**：

| 场景 | 说明 |
|---|---|
| 科学上网 / 代理访问 | 突破网络访问限制 |
| 企业内网出口 | 统一出口 IP、审计员工上网行为 |
| 隐藏客户端身份 | 保护访问来源 |
| 缓存加速 | 代理缓存公共资源，减少重复下载 |

> 一句话记忆：**正向代理，"我"（客户端）请人帮我办事，服务器不知道我是谁。**

---

## 反向代理

**反向代理**：代理服务器站在**服务器**一方，接收来自客户端的请求，再转发给内部的后端服务器处理。客户端以为自己访问的就是真正的服务器，**并不知道背后有代理**。

```
客户端 → 反向代理 → 后端服务器集群
        （替服务器出面）
```

**核心特点**：

- 代理的是**服务器**，隐藏的是**真实服务器**的地址和结构
- 客户端**无需任何配置**，完全无感知
- 客户端只看到代理服务器，真实后端对客户端透明

**正向代理 vs 反向代理对比**：

| 维度 | 正向代理 | 反向代理 |
|---|---|---|
| 代理对象 | 客户端 | 服务器 |
| 隐藏对象 | 真实客户端 | 真实服务器 |
| 客户端是否需配置 | 需要 | 不需要 |
| 客户端是否感知 | 感知到代理存在 | 无感知 |
| 典型场景 | 科学上网、内网出口 | 网站网关、负载均衡 |

**Nginx 反向代理配置**（`nginx.conf`）：

```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        # 转发到后端服务器
        proxy_pass http://127.0.0.1:8080;
    }
}
```

**`proxy_pass` 关键注意事项**：

- **路径带 `/`（推荐）**：`proxy_pass http://127.0.0.1:8080/;` → 会**替换**掉 `location` 匹配的前缀
- **路径不带 `/`**：`proxy_pass http://127.0.0.1:8080;` → 会**拼接**原始请求路径

```nginx
# 例：location /api/ 时
proxy_pass http://127.0.0.1:8080/;   # /api/user → /user
proxy_pass http://127.0.0.1:8080;    # /api/user → /api/user
```

**常用代理请求头**（传递真实客户端信息）：

```nginx
location / {
    proxy_pass http://backend;
    proxy_set_header Host $host;                  # 保留原始 Host
    proxy_set_header X-Real-IP $remote_addr;      # 真实客户端 IP
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  # 代理链路 IP
    proxy_set_header X-Forwarded-Proto $scheme;   # 原始协议(http/https)
}
```

> 一句话记忆：**反向代理，"服务器"请人帮忙接待，客户端不知道背后有谁。**

---

## 负载均衡

**负载均衡**：当后端有多个服务器实例时，Nginx 按照某种策略把请求**分摊**到不同实例上，从而提升整体吞吐量、可用性与扩展性。

```
               ┌─→ 服务器 A
客户端 → Nginx ┼─→ 服务器 B
               └─→ 服务器 C
```

**作用**：

- **提升并发能力**：请求分散到多台，突破单机瓶颈
- **提高可用性**：某台宕机后自动摘除，服务不中断
- **便于水平扩展**：加机器即扩容

**配置方式**：

```nginx
# 1. 定义后端服务器组（upstream）
upstream backend {
    server 127.0.0.1:8080;
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
}

server {
    listen 80;
    location / {
        proxy_pass http://backend;   # 指向 upstream 名称
    }
}
```

### 负载均衡策略

| 策略 | 指令 | 说明 |
|---|---|---|
| **轮询**（默认） | 无 | 按顺序依次分发，最常用 |
| **加权轮询** | `weight=n` | 权重越高分到的请求越多，适合性能不均的机器 |
| **IP 哈希** | `ip_hash` | 同一客户端 IP 固定落到同一台，解决 **session 一致性** |
| **最少连接** | `least_conn` | 优先分给当前连接数最少的服务器 |
| **响应时间** | `fair`（第三方） | 按后端响应时间优先分配 |
| **URL 哈希** | `url_hash`（第三方） | 相同 URL 固定落到同一台，适合缓存场景 |

```nginx
upstream backend {
    # 加权轮询：A 的权重是 B 的两倍
    server 127.0.0.1:8080 weight=2;
    server 127.0.0.1:8081 weight=1;

    # IP 哈希：同一 IP 始终访问同一台
    # ip_hash;
}
```

**常用 server 参数**：

```nginx
upstream backend {
    server 127.0.0.1:8080 weight=2 max_fails=3 fail_timeout=30s;
    # max_fails：允许失败次数，超过则认为该节点不可用
    # fail_timeout：标记不可用后的等待时间，期间不再分发请求
    # backup：备用服务器，主节点全部宕机后才启用
    server 127.0.0.1:8081 down;   # down：手动标记为不可用
}
```

> 一句话记忆：**负载均衡解决的是"一台扛不住，怎么把请求分给多台"的问题。**

---

## 动静分离

**动静分离**：把**静态资源**（图片、CSS、JS、HTML 等）与**动态请求**（接口、业务逻辑）分开处理。静态资源由 Nginx 直接返回，动态请求才转发给后端应用服务器。

**为什么这样设计？**

- 静态资源不需要后端逻辑，交给 Nginx 处理**更快、更省资源**
- 后端 Tomcat 等应用服务器专注于业务，**减少无谓压力**
- 静态资源可配合缓存策略，进一步提速

```
          ┌─ 静态资源（/img /css /js）→ Nginx 本地直接返回
请求 → Nginx
          └─ 动态请求（/api 等）      → 转发给后端 Tomcat
```

**配置方式**：

```nginx
server {
    listen 80;
    server_name localhost;

    # 1. 静态资源：由 Nginx 直接返回
    location /static/ {
        root /usr/share/nginx/html;
        expires 7d;                       # 浏览器缓存 7 天
    }

    # 或按扩展名匹配静态资源
    location ~* \.(png|jpg|jpeg|gif|css|js|ico)$ {
        root /usr/share/nginx/html;
        expires 30d;
    }

    # 2. 动态请求：转发给后端
    location /api/ {
        proxy_pass http://backend;
    }
}
```

**`root` 与 `alias` 的区别**（易错点）：

| 指令 | 行为 | 示例（location /static/） |
|---|---|---|
| `root` | **拼接**完整路径 | `root /data` → 实际路径 `/data/static/xx.png` |
| `alias` | **替换** location 路径 | `alias /data/` → 实际路径 `/data/xx.png` |

```nginx
location /static/ {
    alias /data/static/;    # 请求 /static/a.png → /data/static/a.png
    # root /data;           # 请求 /static/a.png → /data/static/a.png
}
```

> 一句话记忆：**动静分离让"能直接给的不绕路"，静态直出、动态转发。**

---

# 常用命令

> 以下命令基于 Linux 环境，需在 Nginx 安装目录（如 `/usr/local/nginx/sbin`）下执行，或已配置好环境变量。

| 命令 | 说明 |
|---|---|
| `nginx` | 启动 Nginx |
| `nginx -s stop` | **强制停止**（快速关闭，不处理完当前连接） |
| `nginx -s quit` | **优雅停止**（处理完当前请求再退出） |
| `nginx -s reload` | 重新加载配置（**平滑重启**，服务不中断） |
| `nginx -s reopen` | 重新打开日志文件（日志切割后使用） |
| `nginx -v` | 查看 Nginx 版本 |
| `nginx -V` | 查看版本及编译参数（含模块信息） |
| `nginx -t` | **测试配置文件语法是否正确** |
| `nginx -c /path/nginx.conf` | 指定配置文件启动 |
| `nginx -h` / `-?` | 查看帮助 |

**使用流程建议**：

```bash
nginx -t          # 1. 先检查配置语法
nginx -s reload   # 2. 无误后再平滑重载
nginx -s quit     # 3. 需要停止时优雅退出
```

**关于信号量**：`nginx -s xxx` 本质上就是给 Nginx 主进程发信号，等价操作如下：

| 命令 | 等价信号 | 含义 |
|---|---|---|
| `nginx -s stop` | `TERM` | 快速关闭 |
| `nginx -s quit` | `QUIT` | 优雅关闭 |
| `nginx -s reload` | `HUP` | 重载配置 |
| `nginx -s reopen` | `USR1` | 重开日志 |

```bash
# 也可以直接用 kill 发信号（pid 为 nginx master 进程号）
kill -QUIT $(cat /usr/local/nginx/logs/nginx.pid)
```

**在 Windows 下**（若需要）：

```powershell
# 需在 nginx 解压目录下执行
start nginx            # 启动
nginx -s stop          # 停止
nginx -s reload        # 重载
tasklist | findstr nginx   # 查看进程
```

---

# 配置文件结构（补充）

Nginx 主配置文件为 `conf/nginx.conf`，整体分为三块：

```nginx
# 1. 全局块：影响 Nginx 全局运行的配置
worker_processes  1;                 # worker 进程数，通常设为 CPU 核心数
events {
    # 2. events 块：影响 Nginx 与用户的网络连接
    worker_connections  1024;        # 单个 worker 最大连接数
}
http {
    # 3. http 块：配置 HTTP 服务，可嵌套多个 server
    include       mime.types;
    default_type  application/octet-stream;
    sendfile      on;                # 开启高效文件传输

    upstream backend {               # 后端服务器组（负载均衡）
        server 127.0.0.1:8080;
    }

    server {
        # 4. server 块：配置一个虚拟主机
        listen       80;             # 监听端口
        server_name  localhost;      # 域名/主机名

        location / {
            # 5. location 块：匹配 URI，配置具体处理规则
            root   html;
            index  index.html;
            proxy_pass http://backend;
        }
    }
}
```

**层级关系**：`全局块 → events 块 → http 块 → server 块 → location 块`

**location 匹配优先级**（常考易错）：

| 写法 | 类型 | 优先级 |
|---|---|---|
| `location = /a` | 精确匹配 | 最高（1） |
| `location ^~ /a` | 前缀匹配，命中后不再匹配正则 | 2 |
| `location ~ /a` | 正则匹配（区分大小写） | 3 |
| `location ~* /a` | 正则匹配（不区分大小写） | 3 |
| `location /a` | 普通前缀匹配 | 最低（4） |

> 匹配顺序总结：**精确 `=` → 前缀 `^~` → 正则 `~` / `~*`（按出现顺序取第一个命中的）→ 普通前缀（最长匹配）**。

---

# 小结

| 知识点 | 一句话理解 |
|---|---|
| 正向代理 | 代理客户端，服务器不知道真实客户是谁 |
| 反向代理 | 代理服务器，客户端不知道真实服务器是谁 |
| 负载均衡 | 多台实例分摊请求，提升并发与可用性 |
| 动静分离 | 静态 Nginx 直出，动态转发后端，各司其职 |
| 常用命令 | `-t` 查语法、`-s reload` 平滑重载、`-s quit` 优雅停止 |
