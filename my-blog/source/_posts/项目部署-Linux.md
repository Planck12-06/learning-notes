---
title: Linux 基础与项目部署(Linux)
date: 2026-07-24 20:55:34
categories:
  - 项目部署
tags:
  - Linux
  - 项目部署
---

# Linux 基础与项目部署

## 一、操作系统分类

### 1. 桌面操作系统

| 操作系统 | 特点 |
| :--- | :--- |
| Windows | 用户数量最多，生态最丰富 |
| macOS | 操作体验好，软硬件结合紧密 |
| Linux | 桌面端使用较少，多用于服务器 |

### 2. 服务器操作系统

| 操作系统 | 特点 |
| :--- | :--- |
| Unix | 安全稳定，商业付费 |
| **Linux** | 安全稳定、免费开源、市场占有率高 |
| Windows Server | 商业付费，市场占有率较低 |

### 3. 移动设备操作系统

| 操作系统 | 开发者 | 特点 |
| :--- | :--- | :--- |
| Android | Google | 基于 Linux 内核、开源，广泛用于手机/平板/智能电视 |
| iOS | Apple | 苹果闭源系统，仅用于 iPhone/iPad 等苹果设备 |
| HarmonyOS | 华为 | 华为自研、开源，目前主要用于华为生态产品 |

---

## 二、Linux 安装方式

### 虚拟机

> **虚拟机**：通过软件模拟的具有完整硬件系统功能、运行在完全隔离环境中的完整计算机系统。

**常见虚拟机软件**：

| 软件 | 说明 |
| :--- | :--- |
| VMware | 功能强大，企业级首选 |
| VirtualBox | 开源免费，轻量级选择 |
| QEMU | 开源虚拟化工具 |
| VMLite WorkStation | 轻量级虚拟机方案 |

---

## 三、Linux 目录结构

```
/
├── bin    # 存放二进制可执行文件（常用命令）
├── boot   # 存放系统引导时使用的各种文件
├── dev    # 存放设备文件
├── etc    # 存放系统配置文件
├── home   # 存放普通用户的家目录
├── lib    # 存放程序运行所需的共享库及内核模块
├── opt    # 额外安装的可选应用程序包
├── root   # 超级用户（root）的家目录
├── sbin   # 存放系统管理命令，仅 root 可访问
├── tmp    # 存放临时文件
├── usr    # 存放系统应用程序
└── var    # 存放运行时需要改变数据的文件（如日志）
```

> **注意**：`/` 是所有目录的顶点，整个目录结构像一棵倒挂的树。

---

## 四、常用命令

### 命令通用格式

```bash
command [-options] [parameter]
```

### 文件与目录操作

#### `ls` — 列出目录内容

```bash
ls [选项] [目录]
```

| 选项 | 说明 |
| :--- | :--- |
| `-a` | 显示所有文件及目录（含 `.` 开头的隐藏文件） |
| `-l` | 以长格式显示（权限、大小、时间等详细信息） |

> **快捷写法**：`ll` 等价于 `ls -l`

#### `cd` — 切换目录

```bash
cd [目录路径]
```

| 符号 | 含义 |
| :--- | :--- |
| `~` | 当前用户的 home 目录（如 `/home/username`） |
| `.` | 当前所在目录 |
| `..` | 上级目录 |
| `-` | 上一次所在的目录 |

#### `mkdir` — 创建目录

```bash
mkdir [-p] 目录名
```

| 选项 | 说明 |
| :--- | :--- |
| `-p` | 递归创建多层目录（如 `mkdir -p a/b/c`） |

#### `rm` — 删除文件或目录

```bash
rm [-rf] 文件或目录
```

| 选项 | 说明 |
| :--- | :--- |
| `-r` | 递归删除目录及其内容 |
| `-f` | 强制删除，不提示确认 |

> ⚠️ **慎重使用** `rm -rf`，误删无法恢复！

#### `cp` — 复制文件或目录

```bash
cp [-r] 源文件 目标位置
```

| 选项 | 说明 |
| :--- | :--- |
| `-r` | 递归复制目录及其所有子目录和文件 |

#### `mv` — 移动或重命名

```bash
mv 源文件 目标位置
```

- 目标不存在 → 重命名
- 目标存在 → 移动到目标位置

#### `pwd` — 显示当前工作目录

```bash
pwd
```

### 文件查看

#### `cat` — 查看文件全部内容

```bash
cat [-n] 文件名
# -n：显示行号
```

#### `more` — 分屏查看文件内容

```bash
more 文件名
```

| 操作键 | 功能 |
| :--- | :--- |
| `Enter` | 向下滚动一行 |
| `Space` | 向下滚动一屏 |
| `b` | 返回上一屏 |
| `q` / `Ctrl+C` | 退出 more |

#### `head` / `tail` — 查看文件头/尾

```bash
head [-n] 文件名    # 查看前 n 行（默认 10 行）
tail [-n] 文件名    # 查看后 n 行（默认 10 行）
```

### 文件查找与搜索

#### `find` — 按名称查找文件

```bash
find 目录 -name 文件名
```

#### `grep` — 在文件中搜索文本

```bash
grep [-inAB] 关键字 文件名
```

| 选项 | 说明 |
| :--- | :--- |
| `-i` | 忽略关键字大小写（ignore case） |
| `-n` | 显示匹配行的行号 |
| `-A n` | 输出匹配行及其后 n 行（After） |
| `-B n` | 输出匹配行及其前 n 行（Before） |

### 打包与压缩（tar）

```bash
tar [-zcxvf] 包文件名 [文件...]
```

| 选项 | 全称 | 说明 |
| :--- | :--- | :--- |
| `-z` | gzip | 通过 gzip 进行压缩/解压 |
| `-c` | create | 创建新的打包文件 |
| `-x` | extract | 从包文件中还原文件 |
| `-v` | verbose | 显示详细执行过程 |
| `-f` | file | 指定包文件名称（必须放在最后） |

> **后缀区别**：`.tar` 仅打包；`.tar.gz` 打包 + 压缩

**常用示例**：

```bash
# 打包（不压缩）
tar -cvf hello.tar ./*

# 打包并压缩
tar -zcvf hello.tar.gz ./*

# 解包（不解压）
tar -xvf hello.tar

# 解压 .tar.gz 到当前目录
tar -zxvf hello.tar.gz

# 解压 .tar.gz 到指定目录
tar -zxvf hello.tar.gz -C /usr/local
```

### 文本编辑器（vi/vim）

> **vi** 是 Linux 系统内置的文本编辑器；**vim** 是 vi 的增强版，支持语法高亮，实际工作中更常用。

#### 三种模式切换

```
┌─────────────┐     i / a / o      ┌─────────────┐     ESC        ┌─────────────┐     : / /      ┌─────────────┐
│             │ ──────────────────→│             │ ─────────────→ │             │ ─────────────→ │             │
│  命令模式    │                    │  插入模式   │                │  命令模式    │                │  底行模式    │
│ (默认)      │ ←──────────────────│ (编辑文本)   │                │             │ ←───────────── │ (保存退出)   │
└─────────────┘        ESC         └─────────────┘                └─────────────┘                └─────────────┘
```

#### 命令模式常用操作

| 操作 | 功能 |
| :--- | :--- |
| `gg` | 跳转到文件第一行 |
| `G` | 跳转到文件最后一行 |
| `dd` | 删除光标所在行 |
| `ndd` | 删除光标所在行及之后的 n 行 |
| `u` | 撤销上一步操作 |
| `i` | 在光标前插入，进入插入模式 |
| `a` | 在光标后插入，进入插入模式 |
| `o` | 在下方新建一行并插入，进入插入模式 |

#### 底行模式常用命令

| 命令 | 功能 |
| :--- | :--- |
| `:wq` | 保存并退出 |
| `:q!` | 不保存强制退出 |
| `:set nu` | 显示行号 |
| `:set nonu` | 取消行号显示 |
| `:n` | 跳转到第 n 行（如 `:10`） |
| `/关键词` | 向下搜索关键词 |

---

## 五、软件安装

### 安装方式对比

| 方式 | 说明 | 适用场景 |
| :--- | :--- | :--- |
| 二进制发布包 | 解压即可使用 | JDK、Tomcat 等 Java 应用 |
| RPM 包 | RedHat 系包管理器 | CentOS/RHEL 系统 |
| YUM | 自动解决依赖，需联网 | 推荐日常使用 |
| 源码编译 | 自由度高，可定制 | 特殊需求场景 |

### JDK 环境配置

编辑 `/etc/profile`，在文件末尾添加：

```bash
export JAVA_HOME=/usr/local/jdk-17.0.10
export PATH=$JAVA_HOME/bin:$PATH
```

使配置生效：

```bash
source /etc/profile
```

验证安装：

```bash
java -version
```

### MySQL 数据库部署

#### 1. 创建用户和组

```sql
-- 创建 mysql 用户组
groupadd mysql

-- 创建系统用户 mysql，归属 mysql 组，不允许登录
useradd -r -g mysql -s /bin/false mysql
```

#### 2. 注册为系统服务

```bash
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysql
chkconfig --add mysql
```

#### 3. 初始化数据库（生成临时密码）

```bash
mysqld --initialize --user=mysql --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data
```

> 初始化完成后会输出临时 root 密码，记录下来。

#### 4. 启动 MySQL 并修改密码

```sql
-- 使用临时密码登录后，修改 root 密码
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY '你的新密码';
```

#### 5. 配置远程访问

```sql
-- 创建允许远程连接的 root 用户
CREATE USER 'root'@'%' IDENTIFIED BY '你的密码';

-- 授予所有权限
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%';

-- 刷新权限使立即生效
FLUSH PRIVILEGES;
```

---

## 六、防火墙操作（firewalld）

| 操作 | 命令 | 备注 |
| :--- | :--- | :--- |
| 查看状态 | `systemctl status firewalld` 或 `firewall-cmd --state` | — |
| 暂时关闭 | `systemctl stop firewalld` | 重启后恢复 |
| 永久关闭 | `systemctl disable firewalld` | 禁止开机自启，下次启动生效 |
| 暂时开启 | `systemctl start firewalld` | 立即生效 |
| 永久开启 | `systemctl enable firewalld` | 设置开机自启，下次启动生效 |
| 开放端口 | `firewall-cmd --zone=public --add-port=8080/tcp --permanent` | 需 reload 生效 |
| 关闭端口 | `firewall-cmd --zone=public --remove-port=8080/tcp --permanent` | 需 reload 生效 |
| 重新加载 | `firewall-cmd --reload` | 使端口规则立即生效 |
| 查看已开放端口 | `firewall-cmd --zone=public --list-ports` | — |

> **提示**：`--permanent` 表示永久生效，修改后需执行 `firewall-cmd --reload` 才能立即生效。
