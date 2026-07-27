---
title: 项目部署(Docker)
date: 2026-07-24 22:37:30
categories:
  - 项目部署
tags:
  - Docker
  - 项目部署
---

# Docker 引入

在 Linux 上部署一个单体项目非常麻烦，由于每台服务器的运行环境都不同，写好的安装流程、部署脚本并不一定在每一个服务器上都能正常运行。因此需要一个工具来帮助自动化部署，Docker 就是这样一个工具。

```bash (Docker 安装 MySQL)
docker pull mysql:8  # 拉取镜像
docker run -d --name mysql \
  -p 3307:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=1234 \
  mysql:8
```

## 命令解读

| 参数 | 说明 |
|:---|:---|
| `docker run -d` | 创建并运行容器，`-d` 让容器以后台进程方式运行 |
| `--name mysql` | 将容器命名为 mysql |
| `-p 3307:3306` | 设置端口映射（宿主机端口:容器内端口）。容器是隔离环境，外界不可访问，但可以将宿主机端口映射到容器内端口。容器内端口由进程决定（如 MySQL 默认 3306），宿主机端口可任意指定 |
| `-e TZ=Asia/Shanghai` | 设置时区为上海时区 |
| `-e MYSQL_ROOT_PASSWORD=1234` | 设置 MySQL 的 root 密码 |
| `mysql:8` | 指定镜像名称及版本号，未指定版本则默认使用 latest |

执行这条命令后，Docker 会自动搜索并下载 MySQL 镜像，然后运行 MySQL 容器，程序员无需手动干预。并且这条命令可以在不同的操作系统中正常运行，非常方便。

> 这里下载的不是安装包，而是**镜像（Image）**。镜像中不仅包含了 MySQL 本身，还包含了其运行所需要的环境、配置、系统级函数库。因此它在运行时就有自己独立的环境，就可以跨系统运行，不需要手动配置环境。这套独立的隔离环境就是**容器（Container）**。

因此，Docker 安装软件的过程就是：**自动搜索下载镜像 → 创建并运行容器**。

Docker 官方提供了一个专门管理、存储镜像的网站 [Docker Hub](https://hub.docker.com)，对外开放了镜像上传、下载的权限。Docker 官方提供了一些基础镜像，各大软件公司又在基础镜像基础上制作了自家软件的镜像，全部都存放在这个网站。

像这样提供存储、管理 Docker 镜像的服务器被称为 **Docker Registry（镜像仓库）**。

## 总结

Docker 本身包含一个后台服务，利用 Docker 命令告诉 Docker 服务去搜索并下载应用对应的镜像，然后根据镜像创建并运行容器，应用就部署完成了。

---

# 基础知识

## 常见命令

<table>
<thead>
<tr>
<th>命令</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>docker pull</code></td><td>拉取镜像</td></tr>
<tr><td><code>docker push</code></td><td>推送镜像到 Docker Registry</td></tr>
<tr><td><code>docker images</code></td><td>查看本地镜像</td></tr>
<tr><td><code>docker rmi</code></td><td>删除本地镜像</td></tr>
<tr><td><code>docker run</code></td><td>创建并运行容器（不能重复创建同名容器）</td></tr>
<tr><td><code>docker stop</code></td><td>停止指定容器</td></tr>
<tr><td><code>docker start</code></td><td>启动指定容器</td></tr>
<tr><td><code>docker restart</code></td><td>重新启动容器</td></tr>
<tr><td><code>docker rm</code></td><td>删除指定容器</td></tr>
<tr><td><code>docker ps</code></td><td>查看容器（加 <code>-a</code> 查看所有容器）</td></tr>
<tr><td><code>docker logs</code></td><td>查看容器运行日志</td></tr>
<tr><td><code>docker exec</code></td><td>进入容器执行命令</td></tr>
<tr><td><code>docker save</code></td><td>保存镜像到本地压缩文件</td></tr>
<tr><td><code>docker load</code></td><td>加载本地压缩文件为镜像</td></tr>
<tr><td><code>docker inspect</code></td><td>查看容器/镜像详细信息</td></tr>
</tbody>
</table>

### 开机自启配置

默认情况下，每次重启服务器都需要手动启动 Docker 和其中的容器。通过以下命令可以实现开机自启：

```bash
# Docker 服务开机自启
systemctl enable docker

# Docker 容器开机自启
docker update --restart=always [容器名/容器ID]
```

## 演示

```bash
# 第1步：去 DockerHub 查看 nginx 镜像仓库及相关信息
# https://hub.docker.com/_/nginx

# 第2步：拉取 Nginx 镜像（比较耗时）
docker pull nginx:1.20.2

# 第3步：查看镜像
docker images

# 第4步：创建并运行 Nginx 容器
docker run -d --name nginx -p 80:80 nginx

# 第5步：查看运行中的容器
docker ps

# 也可以加格式化方式访问，显示更加清爽
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"

# 第6步：访问网页测试，地址：http://虚拟机IP地址

# 第7步：停止容器
docker stop nginx

# 第8步：查看所有容器（包括已停止的）
docker ps -a --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"

# 第9步：再次启动 nginx 容器
docker start nginx

# 第10步：再次查看容器
docker ps --format "table {{.ID}}\t{{.Image}}\t{{.Ports}}\t{{.Status}}\t{{.Names}}"

# 第11步：查看容器详细信息
docker inspect nginx

# 第12步：进入容器，查看容器内目录结构
docker exec -it nginx bash

# 或者进入 MySQL 容器
docker exec -it mysql mysql -uroot -p

# 第13步：删除容器
docker rm nginx

# 发现无法删除，因为容器仍在运行中，强制删除容器
docker rm -f nginx
```

---

# 数据卷

## 基础概念

容器是隔离环境，容器内的程序文件、配置、运行时产生的数据都在容器内部。要读写容器内的文件非常不方便，因此程序产生的数据与程序运行依赖的配置都应该与容器解耦。

> **数据卷（Volume）** 是一个虚拟目录，是容器内目录与宿主机目录之间映射的桥梁。

> 容器内目录与宿主机目录关联起来的操作称为**挂载（Mount）**。

`/var/lib/docker/volumes` 这个目录就是默认存放所有容器数据卷的目录，其下再根据数据卷名称创建新目录，格式为 `/数据卷名/_data`。

### 为什么不让容器目录直接指向宿主机目录？

- 直接指向宿主机目录会与宿主机**强耦合**。如果切换了环境，宿主机目录就可能发生改变。而容器一旦创建，目录挂载就无法修改，这样容器就无法正常工作了。
- 但是容器指向**数据卷**（一个逻辑名称），数据卷再指向宿主机目录，就不存在强耦合问题。如果宿主机目录发生改变，只要改变数据卷与宿主机目录之间的映射关系即可。

## 数据卷常用命令

<table>
<thead>
<tr>
<th>命令</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>docker volume create</code></td><td>创建数据卷</td></tr>
<tr><td><code>docker volume ls</code></td><td>查看所有数据卷</td></tr>
<tr><td><code>docker volume inspect</code></td><td>查看数据卷详细信息</td></tr>
<tr><td><code>docker volume rm</code></td><td>删除指定数据卷</td></tr>
<tr><td><code>docker volume prune</code></td><td>删除所有未使用的数据卷</td></tr>
</tbody>
</table>

> **注意**：容器与数据卷的挂载要在**创建容器时**配置，对于已创建的容器无法再设置数据卷。而且在创建容器的过程中，如果数据卷不存在会自动创建。

## 演示：Nginx 的 HTML 目录挂载

```bash
# 1. 创建容器并指定数据卷，通过 -v 参数来指定数据卷
docker run -d --name nginx -p 80:80 -v html:/usr/share/nginx/html nginx:1.20.2

# 2. 查看数据卷
docker volume ls

# 3. 查看数据卷详情
docker volume inspect html

# 4. 查看 /var/lib/docker/volumes/html/_data 目录
ll /var/lib/docker/volumes/html/_data

# 5. 进入该目录，修改 index.html 内容
cd /var/lib/docker/volumes/html/_data
vi index.html

# 6. 打开页面查看效果

# 7. 进入容器内部，查看 /usr/share/nginx/html 目录内的文件是否同步变化
docker exec -it nginx bash
```

## 本地目录挂载

可以发现，数据卷的默认目录结构较深（`/var/lib/docker/volumes/...`），操作起来不太方便。在很多情况下，可以直接将容器目录与宿主机的指定目录进行挂载。语法如下：

```bash
# 挂载本地目录
-v 本地目录:容器内目录

# 挂载本地文件
-v 本地文件:容器内文件
```

> **注意**：本地目录或文件必须以 `/` 或 `./` 开头，如果直接以名字开头，会被识别为**数据卷名**而非本地目录名。

**示例对比：**

```bash
-v mysql:/var/lib/mysql           # 识别为数据卷名为 mysql，运行时会自动创建
-v ./mysql:/var/lib/mysql         # 识别为当前目录下的 mysql 目录，不存在则自动创建
-v /root/mysql:/var/lib/mysql     # 识别为宿主机绝对路径 /root/mysql 目录
```

---

# 自定义镜像

## 镜像的结构

之前说过，镜像之所以能实现快速跨操作系统部署应用而忽略其运行环境、配置，就是因为镜像中包含了程序运行需要的系统函数库、环境、配置、依赖。

自定义镜像本质就是依次准备好程序运行的**基础环境 → 依赖 → 应用本身 → 运行配置**等文件，并且打包而成。

上述步骤中的每一次操作其实都是在生产一些文件（系统运行环境、函数库、配置最终都是磁盘文件），所以镜像就是一堆文件的集合。但需要注意的是，镜像文件不是随意堆放的，而是按照操作的步骤**分层叠加**而成，每一层形成的文件都会单独打包并标记一个唯一 ID，称为 **Layer（层）**。

这样做的好处是：如果构建时用到的某些层其他人已经制作过，就可以直接拷贝使用这些层，而不用重复制作（这就是分层复用的思想）。

## Dockerfile

由于制作镜像的过程中需要逐层处理和打包，比较复杂，所以 Docker 提供了自动打包镜像的功能。只需要将打包的过程、每一层要做的事情用固定的语法写下来，交给 Docker 去执行即可。而这种记录镜像构建过程的文件就称为 **Dockerfile**。

### Dockerfile 常用指令

<table>
<thead>
<tr>
<th>指令</th>
<th>说明</th>
<th>示例</th>
</tr>
</thead>
<tbody>
<tr><td><code>FROM</code></td><td>指定基础镜像</td><td><code>FROM centos:7</code></td></tr>
<tr><td><code>ENV</code></td><td>设置环境变量，可在后续指令中使用</td><td><code>ENV key value</code></td></tr>
<tr><td><code>COPY</code></td><td>拷贝本地文件到镜像的指定目录</td><td><code>COPY ./xx.jar /tmp/app/</code></td></tr>
<tr><td><code>RUN</code></td><td>执行 Shell 命令，一般是安装过程的命令</td><td><code>RUN yum install -y java</code></td></tr>
<tr><td><code>EXPOSE</code></td><td>声明容器运行时监听的端口（仅声明作用）</td><td><code>EXPOSE 8080</code></td></tr>
<tr><td><code>WORKDIR</code></td><td>设置工作目录</td><td><code>WORKDIR /app</code></td></tr>
<tr><td><code>ENTRYPOINT</code></td><td>镜像中应用的启动命令（容器运行时调用，不可被覆盖）</td><td><code>ENTRYPOINT ["java", "-jar", "xx.jar"]</code></td></tr>
<tr><td><code>CMD</code></td><td>容器运行时默认执行的命令（可被覆盖）</td><td><code>CMD ["java", "-jar", "xx.jar"]</code></td></tr>
</tbody>
</table>

### Dockerfile 示例

```dockerfile
# 使用 CentOS 7 作为基础镜像
FROM centos:7

# 添加 JDK 到镜像中
COPY jdk17.tar.gz /usr/local/
RUN tar -xzf /usr/local/jdk17.tar.gz -C /usr/local/ && rm /usr/local/jdk17.tar.gz

# 设置环境变量
ENV JAVA_HOME=/usr/local/jdk-17.0.10
ENV PATH=$JAVA_HOME/bin:$PATH

# 创建应用目录并设置为工作目录
RUN mkdir -p /app
WORKDIR /app

# 复制应用 JAR 文件到容器
COPY app.jar app.jar

# 暴露端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/app/app.jar"]
```

### 构建镜像

Dockerfile 文件编写好之后，可以使用以下命令来构建镜像：

```bash
docker build -t 镜像名:标签 .
```

| 参数 | 说明 |
|:---|:---|
| `-t` | 给镜像起名，格式为 `repository:tag`，不指定 tag 时默认为 `latest` |
| `.` | 指定 Dockerfile 所在目录，`.` 表示当前目录 |

---

# Docker 网络

## 网络常用命令

<table>
<thead>
<tr>
<th>命令</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>docker network create</code></td><td>创建一个自定义网络</td></tr>
<tr><td><code>docker network ls</code></td><td>查看所有网络</td></tr>
<tr><td><code>docker network rm</code></td><td>删除指定网络</td></tr>
<tr><td><code>docker network prune</code></td><td>清除所有未使用的网络</td></tr>
<tr><td><code>docker network connect</code></td><td>使指定容器连接加入某网络</td></tr>
<tr><td><code>docker network disconnect</code></td><td>使指定容器离开某网络</td></tr>
<tr><td><code>docker network inspect</code></td><td>查看网络详细信息</td></tr>
</tbody>
</table>

## 演示：自定义网络

```bash
# 1. 通过命令创建一个自定义网络
docker network create mynet

# 2. 查看网络列表
docker network ls

# 输出示例：
# NETWORK ID     NAME      DRIVER    SCOPE
# 639bc44d0a87   bridge    bridge    local
# 403f16ec62a2   mynet     bridge    local    ← 这是新建的
# 0dc0f72a0fbb   host      host      local
# cd8d3e8df47b   none      null      local
# 其中除了 mynet 以外，其它都是 Docker 默认的网络

# 3.让 myapp 和 mysql 都加入该网络
# 3.1. mysql 容器加入 mynet 网络
docker network connect mynet mysql

# 3.2. myapp 容器（Java 项目）加入 mynet 网络
docker network connect mynet myapp

# 4. 进入 myapp 容器，尝试利用容器名访问 mysql
# 4.1. 进入容器
docker exec -it myapp bash

# 4.2. 用容器名作为 hostname 进行 ping 测试
ping mysql

# 输出示例：
# PING mysql (172.18.0.2) 56(84) bytes of data.
# 64 bytes from mysql.mynet (172.18.0.2): icmp_seq=1 ttl=64 time=0.044 ms
# 64 bytes from mysql.mynet (172.18.0.2): icmp_seq=2 ttl=64 time=0.054 ms
```

---

# Docker Compose

部署一个简单的 Java 项目，其中就包含 3 个容器：
- MySQL
- Nginx
- Java 项目

而稍微复杂的项目，其中还会有各种各样的其它中间件，需要部署的东西远不止 3 个。如果还像之前那样手动的逐一部署，就太麻烦了。

**Docker Compose** 可以实现多个相互关联的 Docker 容器的快速部署。它允许通过一个单独的 `docker-compose.yml` 模板文件（YAML 格式）来定义一组相关联的应用容器。

## 基本语法

`docker-compose.yml` 文件中可以定义多个相互关联的应用容器，每一个应用容器被称为一个**服务（service）**。由于 service 就是在定义某个应用的运行时参数，因此与 `docker run` 参数非常相似。

### docker-compose.yml 示例

```yaml
services:
  mysql:
    image: mysql:8
    container_name: mysql
    environment:
      MYSQL_ROOT_PASSWORD: 1234
      TZ: Asia/Shanghai
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - mynet

  nginx:
    image: nginx:1.20.2
    container_name: nginx-tlias
    ports:
      - "80:80"
    volumes:
      - "/usr/local/app/html:/usr/share/nginx/html"
      - "/usr/local/app/conf/nginx.conf:/etc/nginx/nginx.conf"
    networks:
      - mynet

volumes:
  mysql_data:

networks:
  mynet:
    name: mynet
```

### docker run 与 docker compose 参数对照表

<table>
<thead>
<tr>
<th>docker run 参数</th>
<th>docker compose 指令</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr><td><code>--name</code></td><td><code>container_name</code></td><td>容器名称</td></tr>
<tr><td><code>-p</code></td><td><code>ports</code></td><td>端口映射</td></tr>
<tr><td><code>-e</code></td><td><code>environment</code></td><td>环境变量</td></tr>
<tr><td><code>-v</code></td><td><code>volumes</code></td><td>数据卷/目录挂载</td></tr>
<tr><td><code>--network</code></td><td><code>networks</code></td><td>网络配置</td></tr>
</tbody>
</table>

## 基础命令

编写好 `docker-compose.yml` 文件后，就可以部署项目了。基本语法如下：

```bash
docker compose [OPTIONS] [COMMAND]
```

### 参数与指令说明

<table>
<thead>
<tr>
<th>类型</th>
<th>参数/指令</th>
<th>说明</th>
</tr>
</thead>
<tbody>
<tr>
<td rowspan="2">Options</td>
<td><code>-f</code></td>
<td>指定 compose 文件的路径和名称</td>
</tr>
<tr>
<td><code>-p</code></td>
<td>指定 project 名称。project 就是当前 compose 文件中设置的多个 service 的集合，是逻辑概念</td>
</tr>
<tr>
<td rowspan="9">Commands</td>
<td><code>up</code></td>
<td>创建并启动所有 service 容器（加 <code>-d</code> 后台运行）</td>
</tr>
<tr>
<td><code>down</code></td>
<td>停止并移除所有容器、网络</td>
</tr>
<tr>
<td><code>ps</code></td>
<td>列出所有启动的容器</td>
</tr>
<tr>
<td><code>logs</code></td>
<td>查看指定容器的日志</td>
</tr>
<tr>
<td><code>stop</code></td>
<td>停止容器</td>
</tr>
<tr>
<td><code>start</code></td>
<td>启动容器</td>
</tr>
<tr>
<td><code>restart</code></td>
<td>重启容器</td>
</tr>
<tr>
<td><code>top</code></td>
<td>查看运行的进程</td>
</tr>
<tr>
<td><code>exec</code></td>
<td>在指定的运行中容器中执行命令</td>
</tr>
</tbody>
</table>

### 部署示例

```bash
# 后台启动所有服务
docker compose up -d

# 查看服务状态
docker compose ps

# 查看日志
docker compose logs -f

# 停止所有服务
docker compose down
```
