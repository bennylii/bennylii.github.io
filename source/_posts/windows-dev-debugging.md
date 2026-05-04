---
title: Windows 开发环境的四个坑：Gradle OOM、ES 内存、localhost 连不上 PostgreSQL、bat 改 ps1
date: 2026-05-05 04:30:00
tags: [windows, gradle, ipv6, postgresql, golang, powershell, cmd, debugging]
---

# Windows 开发环境连环坑：Gradle、IPv6、内存三板斧

> 一个全栈项目的 Windows 开发环境，8 个服务接连出问题。本文复盘整个排查链路，从"为什么 Gradle 随机挂"到"为什么 localhost 连不上 localhost"。

## 背景

项目 dev 环境需要启动 8 个服务：

| 层 | 服务 | 端口 |
|---|------|------|
| 基础设施 | MongoDB, Redis, Elasticsearch, PostgreSQL | 27017, 6379, 9200, 5432 |
| 后端 | Kotlin (Ktor/Gradle), Go Auth | 8081, 8080 |
| 前端 | Main (Vite), Auth (Vite) | 5173, 5174 |

用一台 Windows 10 工作站做本地开发，16GB 内存。启动脚本用 `start-all.bat`，依次拉起所有服务然后健康检查。

## 第一板斧：Kotlin 后端"起来就死"

### 症状

Kotlin 后端是 Ktor + Gradle 项目，用 `gradle.bat run --no-daemon` 启动。日志显示：

```
Application started in 1.341 seconds.
Responding at http://127.0.0.1:8081
```

几秒后进程消失，端口 8081 不再监听。没有 crash dump，没有堆栈——安静地没了。

### 排查

Gradle daemon 的日志在 `%USERPROFILE%\.gradle\daemon\8.8\daemon-*.out.log`。拉出来看最后几行：

```
Application started successfully at 01:47:08
thread 30: client disconnection detected, canceling the build
```

关键信息：**client disconnection detected**。Gradle 有两种进程模型：

```
┌──────────────┐     start      ┌──────────────────┐
│ gradle.bat   │ ────────────── │  Daemon JVM      │
│ (client)     │     fork       │  (跑编译+应用)     │
│ 64MB heap    │                │  可配 heap        │
└──────────────┘                └──────────────────┘
```

- **Daemon 模式**（默认）：client 只是个薄壳，编译在 daemon JVM 里跑。client OOM 不太影响 daemon。
- **`--no-daemon` 模式**：所有东西挤在一个 JVM 里。而 `gradle.bat` 的 `DEFAULT_JVM_OPTS` 只有 `-Xmx64m`。

64MB 跑 Kotlin 编译 + Ktor 应用？应用刚启动完，GC 一激进就 OOM，client 进程被 OS 杀掉。Daemon 检测到 client 连接断了 → "client disconnection detected, canceling the build" → 应用也被杀了。

那为什么用 `--no-daemon`？因为 daemon 模式在嵌套的 `cmd /k` 窗口里也有问题——daemon 启动后会立即返回，`cmd /k` 窗口里只剩一个空 shell，用户看不到应用输出。

### 还有一层：JAVA_HOME 斜杠

在排查过程中还撞上了另一个问题。某次重启后，Gradle 窗口报：

```
ERROR: JAVA_HOME is set to an invalid directory:
C:\Users\bennyli\auto-novel-env\jdk-17
```

但 JDK 的路径完全正确，`java.exe` 就在那。问题出在 `gradle.bat` 第 56 行：

```batch
set JAVA_EXE=%JAVA_HOME%/bin/java.exe
```

注意这是 **`/`**，不是 Windows 的 `\`。`gradle.bat` 是 Gradle 发行版自带的跨平台脚本——它在 Linux/macOS 用 `/`，但 Windows 版没改干净。

纯 CMD 直接执行时 `if exist` 通常会容错通过。但嵌套在 `start "..." cmd /k "..."` 里时，CMD 的路径解析有时拒绝这个混用写法。**这是 Gradle 8.8 发行版的 bug，不是项目代码的问题。**

### 解决

绕开 `gradle.bat`，直接调 `java.exe`：

```powershell
java.exe -Xmx512m `
  -classpath "gradle\lib\gradle-launcher-8.8.jar" `
  org.gradle.launcher.GradleMain run --no-daemon
```

三个要点：
- `-Xmx512m` 显式覆盖默认的 64MB
- `--no-daemon` 一个 JVM 扛全部，512MB 足够
- 不经过 `gradle.bat`，JAVA_HOME 斜杠问题不存在


## 第二板斧：Elasticsearch 吃掉一半内存

### 症状

所有服务按 bat 脚本顺序启动后，任务管理器显示内存占用 7.5 GB，光 Elasticsearch 一个 `java.exe` 就占了 7 GB+。

### 排查

Elasticsearch 自带的 JVM 堆自动计算逻辑：取系统内存的 50%。16GB 内存 → 自动分配约 7.7GB。而这台机器同时还要跑 MongoDB、PostgreSQL、另一个 JDK、两个 Node 进程、Go 编译……

更糟的是，ES 在 Kotlin 之前启动。ES 的 JVM 刚吞掉 7GB，Kotlin 的 Gradle 又以 64MB 堆起跑——内存压力挤压了 Gradle client 的生存空间，让它更容易 OOM。

### 解决

在 `elasticsearch/config/jvm.options.d/` 下新建 `heap.options`：

```
-Xms512m
-Xmx512m
```

ES 用于 dev 环境绰绰有余，512MB 能跑所有索引操作。


## 第三板斧：Go 连不上同一台机器上的 PostgreSQL

### 症状

Kotlin 后端稳定后，打开 Auth 前端 `http://localhost:5174` 登录。输入 admin / admin123，页面提示：

```
登录失败: [404] Not Found
```

排查后 404 是前端 body 缺少 `app` 字段。补上之后再试，变成：

```
登录失败: [500] Internal Server Error
```

### 排查

直接 curl Go Auth API（端口 8080）：

```
$ curl -X POST http://localhost:8080/v1/auth/login \
  -d '{"app":"auth","username":"admin","password":"admin123"}'
查询用户失败
```

500 + "查询用户失败" → 说明路由命中、JSON 解析通过，但数据库查询挂了。

Go Auth 的 `main.go` 读取环境变量：

```go
db := infra.NewSqlDb(
    env("DB_HOST", "localhost"),
    envInt("DB_PORT", 4002),     // 默认 4002
    env("DB_USER", "auth"),
    env("DB_PASSWORD", ""),
    env("DB_NAME", "auth"),
)
```

查看 Go 进程的 stderr 日志，真正错误浮出水面：

```
jet: dial tcp [::1]:4002: connectex: No connection could be made
    because the target machine actively refused it.
```

两个问题：
1. **端口错了**：连的是 4002（代码默认值），不是 5432。`DB_PORT` 环境变量没传进去——bash 的 `set` 和 Windows CMD 的 `set` 是两回事。
2. **走了 IPv6**：`[::1]` 是 IPv6 的 localhost。

修复端口后重试：

```
jet: dial tcp [::1]:5432: connectex: ... refused.
```

端口对了，但 IPv6 (`[::1]`) 仍然被拒。用 psql 明确指定 `127.0.0.1` 测试：

```
$ psql -h 127.0.0.1 -U auth -d auth -c "SELECT username FROM auth_user;"
 username
----------
 admin
 test_admin
 ...
(9 rows)
```

数据库好好的，`admin` 用户也在。问题窄化为：**Go 的 `lib/pq` 驱动在 Windows 上用 `localhost` 解析时优先尝试 IPv6，被 pg_hba.conf 拒绝。**

为什么 pg_hba.conf 拒绝 IPv6？PostgreSQL 默认的 `pg_hba.conf` 可能只配了 IPv4 的信任规则：

```
# IPv4 local connections:
host    all    all    127.0.0.1/32    trust
# IPv6 可能没配，或被注释
```

### 解决

两处修改：

**bat 脚本**（CMD 语法）：
```batch
:: 之前
set DB_HOST=localhost && set RDB_HOST=localhost && ...

:: 之后
set DB_HOST=127.0.0.1 && set RDB_HOST=127.0.0.1 && ...
```

**新建 ps1 脚本**（PowerShell），彻底告别 CMD 的环境变量坑：
```powershell
$env:DB_HOST = "127.0.0.1"
$env:DB_PORT = "5432"
$env:RDB_HOST = "127.0.0.1"
...
Start-Process -FilePath "go" -ArgumentList "run", "." -WorkingDirectory "$AUTH_DIR\api"
```

`127.0.0.1` 强制 IPv4，Go 驱动直接走 TCPv4，无需 DNS 解析。


## 总结：三板斧的根因关系

这三件事不是孤立的——它们互相放大：

```
ES 堆 7.4GB (50% 内存)
        │
        ▼
   系统内存吃紧
        │
        ▼
Gradle client 64MB 堆 ──── OOM 被 OS 杀 ──── daemon 检测断开 ──── 取消构建
        │
        ▼
   应用进程消失（无 crash 日志）
```

而 IPv6 问题是另一条线——和内存无关，但同样"症状不指向根因"。404 → 500 → "查询用户失败" → 实际是网络层连不上，几层中间件都没有自己的日志能说明问题。

三点教训：
1. **`--no-daemon` 时检查 JVM 堆大小**。Gradle 默认 `-Xmx64m` 只够转发输出，不够同时编译和运行。
2. **Windows 上 `localhost` 不等于 `127.0.0.1`**。localhost 可能解析成 IPv6 `::1`，而很多 Windows 服务默认只配了 IPv4。
3. **内存问题不单点看**。单个服务的内存配置看起来都没问题，但叠加起来（ES 7GB + Kotlin 编译 512MB + MongoDB + PG + Redis + 两个 Node + Go）就把一台 16GB 机器吃满了。


## 第四板斧：CMD 环境变量不可靠，bat 改 ps1

### 问题已修好，为什么还要折腾？

前面三板斧砍完后 bat 脚本能工作了——所有 8 个服务都能启动。但重跑了几次发现：

- **偶发性 `DB_PORT` 不生效**。代码里读到的是 4002（默认值），不是 5432。
- **`Read-Host` 在后台/非交互式 PowerShell 里抛异常**，脚本最后一行挂了。
- **PS 5.1 的 `Start-Process` 找不到 `.cmd` 文件**（`npx`、`pnpm`），前端 dev server 启不来。
- **`2>$null` 对原生命令可能阻塞管道**。PowerShell 给原生 exe 做 stderr 重定向时会创建一个隐藏的后台管道——如果 exe 输出时序不对，脚本就会在 `&` 调用后卡住。
- **CMD 和 bash 的 `set` 不互通**。排查时用 bash 设 `set DB_PORT=5432` 是空操作，浪费了一次排查回合。

### 关键差异

| 问题 | bat (CMD) | ps1 (PowerShell) |
|---|---|---|
| 环境变量 | `set VAR=val && cmd` — cmd /c 嵌套层级不确定时可能丢 | `$env:VAR = "val"` — 进程级设置，`Start-Process` 子进程继承 |
| 原生 exe 管道 | `2>nul` 直连 console，不阻塞 | `2>$null` 创建隐藏管道，某些 exe 会卡住 |
| 找到 .cmd | CMD 自动找同名 `.cmd`/`.bat` | PS 5.1 的 `Start-Process` 只认 `.exe`，必须写完整路径 |
| 路径分隔符 | `\` 和 `/` 混用（gradle.bat 遗留） | 统一 `\`，不混用 |
| 后台/非交互 | `pause` 没问题 | `Read-Host` 在 NonInteractive 模式抛异常 |

### 最终 ps1 方案

```powershell
# 全部用完整路径，消除 PATH 歧义
Start-Process -FilePath "C:\Program Files\Go\bin\go.exe" ...
Start-Process -FilePath "C:\Program Files\nodejs\npx.cmd" ...
Start-Process -FilePath "C:\Users\bennyli\AppData\Roaming\npm\pnpm.cmd" ...

# 环境变量在 Start-Process 前设置，子进程自动继承
$env:DB_HOST = "127.0.0.1"    # 强制 IPv4
$env:DB_PORT = "5432"

# 原生命令用 $ErrorActionPreference 替代 2>$null
$ErrorActionPreference = "SilentlyContinue"
& pg_ctl.exe start
$ErrorActionPreference = "Continue"

# 交互式等待加 try/catch
try { Read-Host } catch {}
```

### 教训

4. **Windows 上启动脚本用 PowerShell，不用 CMD**。环境变量传递明确、管道行为可预测、错误处理更灵活。CMD 的嵌套 `start`/`cmd /k`/`cmd /c` 组合在复杂启动场景下不够可靠。
5. **用完整路径而不是依赖 PATH**。`Start-Process` 在 PS 5.1 里不自动补全扩展名，`npx.cmd` 和 `npx` 是两回事。
6. **`2>$null` 不是 `2>nul` 的等价物**。PowerShell 对原生 exe 的 stderr 重定向是通过后台管道实现的，和 CMD 的直接 console 重定向机制不同，可能引入阻塞。
