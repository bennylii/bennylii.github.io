---
title: Windows 开发环境的三个坑：Gradle OOM、ES 内存、localhost 连不上 PostgreSQL
date: 2026-05-05 03:30:00
tags: [windows, gradle, ipv6, postgresql, golang, debugging]
---

事情很简单：每天早上敲一个命令把开发环境拉起来。项目结构不复杂——一个 Kotlin 写的主后端，一个 Go 写的用户认证服务，前端是两个 Vite 项目，底下垫着 MongoDB、Redis、Elasticsearch、PostgreSQL 四个基础设施。加起来 8 个进程。

我写了一个 `start-all.bat`，预期行为是：双击 → 等服务一个个亮绿灯 → 打开浏览器干活。

结果实际行为是：有的服务随机消失，有的端口不通，前台报错信息指向和根因毫无关系。以下是在 Windows 10 + 16GB 内存上调试这 8 个进程启动问题的记录。

<!-- more -->

## 坑一：Kotlin 后端启动后进程消失

### 现象

用 `gradle.bat run --no-daemon` 启动 Ktor 应用，日志显示启动成功：

```
Application started in 1.341 seconds.
Responding at http://127.0.0.1:8081
```

几秒后进程消失，端口 8081 不再监听。

### 原因

Gradle 的 daemon 日志（`%USERPROFILE%\.gradle\daemon\8.8\daemon-*.out.log`）里有这条：

```
Application started successfully at 01:47:08
thread 30: client disconnection detected, canceling the build
```

Gradle 有两种工作模式：

- **Daemon 模式**（默认）：Gradle 启动一个 daemon JVM 做编译，client 进程只转发输出。client 挂了不影响 daemon。
- **`--no-daemon` 模式**：编译和应用跑在同一个 JVM 里。`gradle.bat` 给这个 JVM 的默认堆大小是 `-Xmx64m`。

64MB 跑 Kotlin 编译 + Ktor 应用不够用。JVM OOM 后被 OS 杀掉 → daemon 检测到 client 断开 → 取消构建 → 应用进程跟着消失。

### 顺带发现的另一个问题：JAVA_HOME 路径

排查期间某次重启，Gradle 窗口报：

```
ERROR: JAVA_HOME is set to an invalid directory:
C:\Users\bennyli\auto-novel-env\jdk-17
```

JDK 路径完全正确。问题在 `gradle.bat` 第 56 行：

```batch
set JAVA_EXE=%JAVA_HOME%/bin/java.exe
```

路径用了 `/` 而非 Windows 的 `\`。这是 Gradle 8.8 发行版自带的跨平台脚本遗留下来的——在 Linux/macOS 正常，Windows 上 CMD 的 `if exist` 在嵌套 `cmd /k` 环境下不稳定。这是 Gradle 发行版的 bug，不是项目代码的问题。

### 修复

绕过 `gradle.bat`，直接用 `java.exe` 启动 Gradle：

```powershell
java.exe -Xmx512m `
  -classpath "gradle\lib\gradle-launcher-8.8.jar" `
  org.gradle.launcher.GradleMain run --no-daemon
```

`-Xmx512m` 覆盖默认的 64MB，同时不再经过 `gradle.bat`，斜杠问题也绕过去了。

## 坑二：Elasticsearch 默认占一半系统内存

### 现象

8 个服务全部启动后，任务管理器显示内存占用 7.5 GB，Elasticsearch 一个 `java.exe` 占了 7 GB 以上。

### 原因

ES 的 JVM 堆自动计算默认取系统内存的 50%。16GB 机器 → 约 7.7GB 堆。

ES 在启动顺序里排在 Kotlin 前面。ES 先占了 7GB+，后面 Kotlin Gradle 以 64MB 堆起跑，内存压力进一步加大了 Gradle OOM 的概率。

### 修复

在 `elasticsearch/config/jvm.options.d/heap.options` 里显式限制：

```
-Xms512m
-Xmx512m
```

dev 环境索引量很小，512MB 足够。

## 坑三：Go 后端用 `localhost` 连不上同一台机器的 PostgreSQL

### 现象

Auth 前端登录页面报 `[500] Internal Server Error`。

### 排查过程

直接 curl Go Auth API（端口 8080）：

```
$ curl -X POST http://localhost:8080/v1/auth/login \
  -d '{"app":"auth","username":"admin","password":"admin123"}'
查询用户失败
```

路由命中了，JSON body 解析通过了，数据库查询失败了。

Go Auth 的数据库连接参数来自环境变量：

```go
db := infra.NewSqlDb(
    env("DB_HOST", "localhost"),
    envInt("DB_PORT", 4002),   // 代码默认值
    env("DB_USER", "auth"),
    env("DB_NAME", "auth"),
)
```

查看 Go 进程的日志：

```
jet: dial tcp [::1]:4002: connectex: No connection could be made
    because the target machine actively refused it.
```

两个问题：

1. **端口是 4002 而不是 5432**。`DB_PORT` 环境变量没生效（启动脚本里用 CMD 的 `set` 语法，在 bash 下是空操作），掉到了代码默认值。
2. **走了 IPv6（`[::1]`）**。`lib/pq` 驱动用 `localhost` 时优先解析 IPv6。

把端口修成 5432 后重试，端口对了但仍然被拒：

```
jet: dial tcp [::1]:5432: connectex: ... refused.
```

用 psql 指定 `127.0.0.1`（强制 IPv4）却是通的：

```
$ psql -h 127.0.0.1 -U auth -d auth -c "SELECT username FROM auth_user;"
 username
----------
 admin
 test_admin
 ...
```

数据库和用户都在。问题就是 **`localhost` 解析到了 IPv6 `[::1]`，而 PostgreSQL 的 `pg_hba.conf` 默认只配了 IPv4 的信任规则**：

```
host    all    all    127.0.0.1/32    trust
```

### 修复

启动脚本里把 `DB_HOST` 和 `RDB_HOST` 从 `localhost` 改成 `127.0.0.1`：

```diff
- set DB_HOST=localhost
+ set DB_HOST=127.0.0.1
```

`127.0.0.1` 直接走 TCPv4，不经过 DNS 解析。同时把启动脚本整体迁移到 PowerShell——`$env:DB_HOST = "127.0.0.1"` 比 CMD 的 `set` 更可靠，也不会出现 bash/cmd 语法混淆的问题。

---

## 总结

三个问题分开看不复杂，但它们互相放大：

ES 内存过配（7GB+）→ 系统内存吃紧 → Gradle client 的 64MB 堆更容易 OOM → daemon 取消构建 → 后端消失。  

IPv6 问题则是另一条线——症状一路从 404 跳到 500 跳到"查询用户失败"，实际是网络层连不上，中间每层给出的错误信息都和根因不直接对应。

三点可以带走的经验：

1. Gradle `--no-daemon` 时，`DEFAULT_JVM_OPTS="-Xmx64m"` 不够同时编译和运行应用。要么不用 `--no-daemon`，要么显式加大堆。
2. Windows 上 `localhost` 不等于 `127.0.0.1`。`localhost` 可能解析到 `::1`（IPv6），而很多 Windows 上的服务默认只配了 IPv4。开发环境连本地服务，直接用 `127.0.0.1` 更可靠。
3. 本地多服务环境的内存要按总和算，不能逐个看。16GB 机器上，一个 ES 默认配置就能吃掉一半。
