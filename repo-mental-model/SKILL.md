---
name: repo-mental-model
description: Analyze a software repository and generate 9-dimension mental models for software architects. Produces visual and textual artifacts that mirror how expert programmers build cognitive representations of code — architecture, concepts, flows, spatial structure, decisions, evolution, concerns, features, and patterns.
---

# Repo Mental Model

为软件架构师系统地分析一个代码仓库，沿 9 个认知维度产出心理表征（Mental Representations）。每个维度对应大脑理解代码时真正构建的一类认知结构：视觉形象、文字影像、时空关系、抽象概念。

支持语言：**C、C++、Rust、Java、Kotlin、Go、Python、TypeScript/JavaScript**，覆盖大型系统项目（Android、Linux kernel、AOSP、嵌入式系统等）。

## 调用方式

```
/repo-mental-model [path]
```

分析 `[path]` 路径下的仓库，省略则分析当前目录。所有产出写入 `<repo-root>/mental-model/`。

---

## 9 个心理表征维度

| # | 维度 | 认知类型 | 思考方式 | 输出形式 |
|---|------|----------|----------|----------|
| 1 | **Arch 架构** | 视觉形象 | 可视化思考 | HTML 架构图 |
| 2 | **Concept 概念** | 文字影像 + 视觉形象 | 抽象思考 | Markdown 词汇表 |
| 3 | **Flow 时间流** | 视觉形象 | 时间线思考 | Excalidraw 流程图 |
| 4 | **Spatial 空间结构** | 视觉形象 | 归纳思考 | 标注树 + 关系图 |
| 5 | **Decision 决策知识** | 文字 + 视觉 | 演绎思考 | 思维导图 / 决策树 |
| 6 | **Evolution 历史演化** | 文字 + 视觉 | 时间导航 | 演化时间线 |
| 7 | **Concern 关注点** | 文字 + 视觉 | 发散思考（多角度） | 关注点矩阵 |
| 8 | **Feature/Function 功能** | 文字 / 视觉 | 归纳 + 抽象思考 | 功能目录 |
| 9 | **Pattern 模式** | 文字 + 视觉 | 抽象 + 归纳思考 | 模式目录 |

---

## 大型代码库策略

当面对 Android、Linux kernel 等超大型仓库（>10 万文件）时：
- **分层采样**：优先分析顶层 2-3 层目录，再深入 2-3 个核心子系统
- **入口导向**：从 main()、module_init()、AndroidManifest.xml、init.rc 等入口向外扩展
- **热点优先**：git log 的高频变更文件是认知优先区域
- **子系统隔离**：把 kernel/drivers/net、frameworks/base/core 等子系统作为独立"模块"建模
- 每条 find/grep 命令都加 `-maxdepth` 和 `| head -N` 限制，避免输出爆炸

---

## 执行计划

### Phase 0：仓库概貌调查

在分析任何维度之前，先建立仓库的基本画像，包括**语言家族判断**——这决定了后续所有维度使用哪套分析策略。

```bash
# ── 语言 / 技术栈检测 ──────────────────────────────────────────

# Web / JVM / Rust 构建文件
find . \( -name "package.json" -o -name "Cargo.toml" -o -name "pom.xml" \
  -o -name "build.gradle" -o -name "build.gradle.kts" \) \
  -maxdepth 3 -not -path "*/.git/*" -not -path "*/node_modules/*" | head -10

# C/C++ 构建系统
find . \( -name "CMakeLists.txt" -o -name "configure.ac" -o -name "meson.build" \
  -o -name "BUILD.gn" -o -name "Android.bp" -o -name "Android.mk" \) \
  -maxdepth 4 -not -path "*/.git/*" | head -10

# Linux kernel 标志
find . \( -name "Kconfig" -o -name "Makefile" \) -maxdepth 2 | head -5
ls vmlinux 2>/dev/null; ls arch/ 2>/dev/null | head -5

# Android 标志
find . \( -name "AndroidManifest.xml" -o -name "OWNERS" -o -name "init.rc" \) \
  -maxdepth 4 -not -path "*/.git/*" | head -10

# ── 规模统计 ──────────────────────────────────────────────────

# 系统语言文件数
find . -type f \( -name "*.c" -o -name "*.cpp" -o -name "*.cc" -o -name "*.cxx" \
  -o -name "*.h" -o -name "*.hpp" -o -name "*.hh" \) \
  -not -path "*/.git/*" | wc -l

# 其他语言文件数
find . -type f \( -name "*.rs" -o -name "*.java" -o -name "*.kt" \
  -o -name "*.py" -o -name "*.go" -o -name "*.ts" \) \
  -not -path "*/.git/*" -not -path "*/node_modules/*" | wc -l

# ── 顶层结构 ──────────────────────────────────────────────────

find . -maxdepth 2 -type d -not -path "*/.*" \
  -not -path "*/node_modules/*" -not -path "*/target/*" \
  -not -path "*/build/*" -not -path "*/out/*" | sort | head -50

# ── 入口点检测 ────────────────────────────────────────────────

# C/C++ main
find . \( -name "main.c" -o -name "main.cpp" -o -name "main.cc" \) \
  -not -path "*/.git/*" -maxdepth 6 | head -10

# Rust main / lib
find . \( -name "main.rs" -o -name "lib.rs" \) \
  -not -path "*/.git/*" -maxdepth 6 | head -10

# Java/Kotlin 入口
grep -rn "public static void main\|fun main(" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git -l | head -10

# Android 入口
find . -name "AndroidManifest.xml" -maxdepth 6 | head -5

# Linux kernel module 入口
grep -rn "module_init\|core_initcall\|subsys_initcall" \
  --include="*.c" --exclude-dir=.git -l | head -10

# README
head -120 README.md 2>/dev/null || head -120 readme.md 2>/dev/null
```

产出 `mental-model/00-overview.md`：
- 项目名称、**语言家族**（系统语言 C/C++/Rust / JVM / 脚本 / 混合）、主框架 / 构建系统
- 项目类型（OS 内核 / Android 系统 / 嵌入式固件 / 服务端 / 工具链 / 库 / 应用）
- 规模（文件数、大致行数）
- 入口点列表
- 一段话说明：这个软件是什么、给谁用、解决什么问题

---

### Dimension 1：Arch — 架构

**认知目标**：一张展示软件基本元素及其关系的视觉形象。大脑在这里构建的是：我眼前的这个软件由什么组成？它们怎么连在一起？

**分析步骤：**

1. **模块识别**：找出顶层模块 / 包 / 子系统 / 库
2. **接口边界**：定位公开 API（头文件、AIDL、IDL、HTTP 路由、RPC 接口）
3. **数据层识别**：数据库、缓存、消息队列、共享内存、文件系统接入点
4. **外部集成**：第三方服务、HAL、驱动、IPC 机制
5. **数据 / 控制流**：主要组件之间的调用关系

```bash
# ── 通用：顶层模块结构 ────────────────────────────────────────

find . -maxdepth 3 -type d -not -path "*/.*" \
  -not -path "*/node_modules/*" -not -path "*/target/*" \
  -not -path "*/build/*" -not -path "*/out/*" -not -path "*/dist/*"

# ── C/C++ API 边界（头文件是 C/C++ 的接口层）──────────────────

# 公开头文件目录
find . -type d \( -name "include" -o -name "public" -o -name "api" \) \
  -not -path "*/.git/*" | head -20

# 导出宏（库对外接口）
grep -rn "EXPORT_API\|DLL_EXPORT\|__declspec(dllexport)\|__attribute__((visibility" \
  --include="*.h" --include="*.hpp" --include="*.c" --include="*.cpp" \
  --exclude-dir=.git | head -30

# Linux kernel：导出符号 & 子系统入口
grep -rn "EXPORT_SYMBOL\|EXPORT_SYMBOL_GPL\|module_init\|subsys_initcall\|arch_initcall" \
  --include="*.c" --include="*.h" --exclude-dir=.git | head -30

# C++ 命名空间 / 类层次（API 顶层）
grep -rn "^namespace \|^class \|^struct " \
  --include="*.h" --include="*.hpp" \
  --exclude-dir=.git | grep -v "test\|Test\|mock\|Mock" | head -50

# ── Android 架构边界 ──────────────────────────────────────────

# AIDL 接口（Binder IPC 契约）
find . -name "*.aidl" -not -path "*/.git/*" | head -20
find . -name "*.aidl" -not -path "*/.git/*" -exec head -5 {} \; 2>/dev/null | head -60

# Android 组件（系统架构节点）
grep -rn "Activity\|Service\|BroadcastReceiver\|ContentProvider\|Fragment" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | grep "extends\|implements\|class " | head -40

# HAL 接口
find . \( -name "*.hal" -o -path "*/hardware/interfaces/*" \) \
  -not -path "*/.git/*" | head -20

# JNI 边界
grep -rn "JNIEXPORT\|JNI_OnLoad\|RegisterNatives\|native " \
  --include="*.c" --include="*.cpp" --include="*.java" --include="*.kt" \
  --exclude-dir=.git | head -30

# ── Rust crate 边界 ────────────────────────────────────────────

grep -rn "^pub mod \|^pub use \|^pub fn \|^pub struct \|^pub trait " \
  --include="*.rs" --exclude-dir=.git | head -50

# ── Web / JVM 路由（如果有）────────────────────────────────────

grep -rn "@Get\|@Post\|@Put\|@Delete\|app\.get\|app\.post\|@RequestMapping\|@RestController" \
  --include="*.ts" --include="*.py" --include="*.go" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=node_modules --exclude-dir=.git | head -40

# ── IPC / 通信机制（跨所有语言）──────────────────────────────

grep -rn "socket\|pipe\|mmap\|shm\|ioctl\|netlink\|dbus\|binder\|grpc\|thrift" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --include="*.java" --include="*.kt" --include="*.rs" \
  --include="*.go" --include="*.py" \
  --exclude-dir=.git | grep -v "test\|Test" | head -30
```

**架构风格识别**（在绘制架构图之前先判断风格——风格决定图的形状和布局）

每组命令的命中数量 = 信号强度，多组命中 = 风格确立。

```bash
# ══════════════════════════════════════════════════════════════
# 架构风格识别：逐风格扫描，统计信号命中数
# ══════════════════════════════════════════════════════════════

# ── 1. 分层架构（Layered Architecture）───────────────────────
# 信号：目录名暗示职责层 + 类型名后缀 Controller/Service/Repository

find . -type d \( -name "controller*" -o -name "service*" -o -name "repository" \
  -o -name "repositories" -o -name "dao" -o -name "domain" \
  -o -name "infrastructure" -o -name "infra" -o -name "presentation" \
  -o -name "application" -o -name "usecase*" -o -name "use_case*" \) \
  -not -path "*/.git/*" -not -path "*/test*" | sort | head -20

# 各层类型命中文件数（三层都有 → 分层确认）
grep -rl "Controller\b\|@Controller\|@RestController" \
  --include="*.java" --include="*.kt" --include="*.ts" --include="*.go" \
  --exclude-dir=.git 2>/dev/null | wc -l
grep -rl "Service\b\|@Service\b" \
  --include="*.java" --include="*.kt" --include="*.ts" \
  --exclude-dir=.git 2>/dev/null | wc -l
grep -rl "Repository\b\|@Repository\|DAO\b\|@DAO" \
  --include="*.java" --include="*.kt" --include="*.ts" \
  --exclude-dir=.git 2>/dev/null | wc -l

# ── 2. 管道-过滤器（Pipeline / Filter）──────────────────────
# 信号：Stage/Filter/Transform/Processor 命名 + 链式处理结构

find . -type f \( -name "*pipeline*" -o -name "*stage*" -o -name "*filter*" \
  -o -name "*transform*" -o -name "*processor*" \) \
  -not -path "*/.git/*" -not -path "*/test*" | head -20

grep -rn "class.*Pipeline\|class.*Stage\|class.*Filter\|class.*Transform\|class.*Processor\|struct.*Pipeline\|struct.*Stage" \
  --include="*.cpp" --include="*.h" --include="*.rs" \
  --include="*.java" --include="*.kt" --include="*.ts" --include="*.go" \
  --exclude-dir=.git | grep -v "test\|Test" | head -20

# 编译器 / LLVM 风格 Pass 管道
grep -rn "PassManager\|addPass\|runOnFunction\|runOnModule\|Pass\b" \
  --include="*.cpp" --include="*.h" --exclude-dir=.git | head -10

# Unix 工具链 pipe 语义
grep -rn "\bpipe\b\|\bpopen\b\|stdin\b\|stdout\b" \
  --include="*.c" --include="*.cpp" --include="*.go" \
  --exclude-dir=.git | grep -v "test" | head -10

# ── 3. 事件驱动（Event-Driven Architecture）─────────────────
# 信号：EventBus / emit / subscribe + 事件名词解耦生产者与消费者

grep -rn "EventBus\|EventEmitter\|EventDispatcher\|MessageBus\|DomainEvent\|ApplicationEvent" \
  --include="*.cpp" --include="*.h" --include="*.rs" \
  --include="*.java" --include="*.kt" --include="*.ts" --include="*.go" \
  --exclude-dir=.git | grep -v "test\|Test" | head -20

grep -rn "\.emit(\|\.publish(\|\.subscribe(\|\.on(\|addEventListener\|\.dispatch(" \
  --include="*.ts" --include="*.java" --include="*.kt" --include="*.cpp" --include="*.rs" \
  --exclude-dir=.git | grep -v "test\|Test" | head -15

# 内核事件驱动：inotify / epoll / io_uring
grep -rn "inotify_\|epoll_\|io_uring\|kqueue\|eventfd\|signalfd" \
  --include="*.c" --include="*.h" --exclude-dir=.git | head -10

# ── 4. 发布订阅（Pub-Sub）────────────────────────────────────
# 信号：topic/consumer/producer + 消息中间件依赖

grep -rn "\bTopic\b\|Consumer\b\|Producer\b\|Subscriber\b\|Publisher\b" \
  --include="*.java" --include="*.kt" --include="*.rs" \
  --include="*.go" --include="*.ts" --include="*.py" \
  --exclude-dir=.git \
  | grep "class \|struct \|trait \|interface " | grep -v "test\|Test" | head -20

grep -rn "kafka\|rabbitmq\|nats\|mqtt\|amqp\|zmq\|zeromq\|pulsar\|redis.*pub\|redis.*sub" \
  --include="*.toml" --include="*.gradle" --include="*.xml" --include="*.yaml" \
  --include="*.java" --include="*.kt" --include="*.go" --include="*.rs" \
  --exclude-dir=.git | head -10

# ── 5. 微服务（Microservices）────────────────────────────────
# 信号：多个独立部署单元 + 服务发现 + API 网关

echo "=== Dockerfile 数量 ==="
find . -name "Dockerfile" -not -path "*/.git/*" | wc -l
find . -name "docker-compose*" -not -path "*/.git/*" | head -5
find . \( -name "*.yaml" -o -name "*.yml" \) \
  | xargs grep -l "kind: Service\|kind: Deployment\|kind: Ingress" 2>/dev/null | head -5

grep -rn "eureka\|consul\|nacos\|etcd\|api.gateway\|ApiGateway\|circuit.breaker\|@FeignClient\|service.discovery" \
  --include="*.java" --include="*.kt" --include="*.yaml" --include="*.yml" \
  --exclude-dir=.git | head -10

# ── 6. 六边形架构 / Ports & Adapters ─────────────────────────
# 信号：port/adapter 目录命名 + 应用核心不依赖外部框架

find . -type d \( -name "ports" -o -name "adapters" -o -name "driven" \
  -o -name "driving" -o -name "primary" -o -name "secondary" \
  -o -name "inbound" -o -name "outbound" \) \
  -not -path "*/.git/*" | head -10

grep -rn "\bPort\b\|\bAdapter\b\|\bInboundPort\|\bOutboundPort\|\bDrivenAdapter\|\bDrivingAdapter\b" \
  --include="*.java" --include="*.kt" --include="*.ts" --include="*.rs" \
  --exclude-dir=.git \
  | grep "class \|interface \|trait \|struct " | head -20

# ── 7. CQRS（命令查询职责分离）───────────────────────────────
# 信号：Command/Query 分离目录 + CommandHandler/QueryHandler

find . -type d \( -name "commands" -o -name "queries" \
  -o -name "command" -o -name "query" \) \
  -not -path "*/.git/*" | head -10

grep -rn "CommandBus\|QueryBus\|CommandHandler\|QueryHandler\|ICommand\b\|IQuery\b" \
  --include="*.ts" --include="*.java" --include="*.kt" --include="*.rs" \
  --exclude-dir=.git | grep -v "test\|Test" | head -20

# ── 8. 插件架构（Plugin / Extension）────────────────────────
# 信号：动态加载 + hook/register/plugin 命名

grep -rn "dlopen\|dlsym\|LoadLibrary\|GetProcAddress\|PluginManager\|ExtensionPoint\|plugin_register\|hook_register" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --include="*.java" --include="*.kt" --include="*.ts" \
  --exclude-dir=.git | grep -v "test\|Test" | head -20

# Linux kernel module（插件架构的系统级实现）
grep -rn "module_init\|module_exit\|MODULE_LICENSE\|struct [a-z_]*_driver\b" \
  --include="*.c" --include="*.h" --exclude-dir=.git | head -10

# ── 9. 状态机（State Machine / FSM）─────────────────────────
# 信号：state 枚举 + 转换表/函数 + 状态名词密集

grep -rn "enum.*[Ss]tate\b\|STATE_[A-Z]\|StateMachine\|state_machine\|FSM\b\|Transition\b\|transition_table" \
  --include="*.c" --include="*.h" --include="*.cpp" \
  --include="*.rs" --include="*.java" --include="*.kt" \
  --include="*.ts" --include="*.go" \
  --exclude-dir=.git | grep -v "test\|Test" | head -30

# switch(state) 模式（C/C++ 状态机惯用法）
grep -rn "switch.*state\|case.*STATE_\|case.*State::" \
  --include="*.c" --include="*.cpp" \
  --exclude-dir=.git | head -15

# ── 10. 响应式（Reactive / Stream-based）───────────────────
# 信号：Observable/Flow/Flux + 背压 / 调度器

grep -rn "Observable\|Flowable\|Mono\b\|Flux\b\|\bFlow\b\|Coroutine\|StateFlow\|SharedFlow\|RxJava\|reactor\." \
  --include="*.java" --include="*.kt" --include="*.ts" \
  --exclude-dir=.git | grep -v "test\|Test\|java\.util\.stream" | head -20

# ── 11. 整洁架构（Clean Architecture）──────────────────────
# 信号：entities/use_cases/interface_adapters + 依赖规则（内层不知外层）

find . -type d \( -name "entities" -o -name "entity" \
  -o -name "use_cases" -o -name "usecases" \
  -o -name "interface_adapters" -o -name "frameworks" \) \
  -not -path "*/.git/*" | head -10

# ── 12. 系统级架构风格（OS / 驱动 / 嵌入式）───────────────

# 宏内核（Linux Monolithic Kernel 特征目录）
ls -d arch drivers kernel mm fs net ipc security 2>/dev/null

# 分层驱动模型（Linux Device Model）
grep -rn "struct bus_type\|struct device_driver\b\|platform_driver_register\|i2c_add_driver\|usb_register" \
  --include="*.h" --include="*.c" --exclude-dir=.git | head -10

# Android HAL 架构
find . \( -path "*/hardware/interfaces/*" -o -name "*.hal" \) \
  -not -path "*/.git/*" | head -10
grep -rn "hw_module_t\|hw_device_t\|HAL_MODULE_INFO_SYM" \
  --include="*.h" --include="*.c" --exclude-dir=.git | head -10
```

根据扫描结果按下表判断主要风格：

| 风格 | 强信号（≥5 处命中） | 弱信号（1-4 处命中） |
|------|---------------------|----------------------|
| 分层架构 | Controller+Service+Repository 三层目录均有 | 只有部分层 |
| 管道-过滤器 | Stage/Filter/Processor 类型 + 链式调用 | 有 Pipeline 命名无链式结构 |
| 事件驱动 | EventBus + 多处 emit/subscribe | 只有少量事件 |
| Pub-Sub | 消息中间件依赖 + Topic/Consumer 类型 | 只有命名无实际中间件 |
| 微服务 | 多个 Dockerfile + 服务发现客户端 | 单 Dockerfile 但有跨进程通信 |
| 六边形 | ports/ + adapters/ 目录 + Port/Adapter 类型 | 只有命名无结构分离 |
| CQRS | CommandBus+QueryBus + commands/queries 双目录 | 只有 Command/Query 命名 |
| 插件架构 | dlopen/module_init + PluginManager | 只有 Plugin 命名 |
| 状态机 | enum State + switch(state) + 转换表 | 只有 state 变量散布各处 |
| 响应式 | Observable/Flux + 调度器 + 背压处理 | 只有少量 Stream |
| 整洁架构 | entities+use_cases+adapters 三目录齐全 | 部分层次 |
| 宏内核 | arch/+drivers/+kernel/+mm/ 齐全 | 部分子系统 |
| 分层驱动 | bus_type+device_driver+platform_driver | 只有 platform_driver |

**产出**：

1. 调用 `architecture-diagram` skill，产出 `mental-model/01-arch.html`，图的布局方向跟随风格：
   - 分层架构 → 垂直叠层，每层一个泳道
   - 管道-过滤器 → 水平线性，箭头单向流动
   - 事件驱动 → 中心 EventBus/总线 + 辐射状组件
   - 微服务 → 各服务独立框 + API 网关入口
   - 插件架构 → 核心宿主框 + 扩展点 + 插件实例
   - 状态机 → 椭圆状态节点 + 有向转换箭头

2. 产出 `mental-model/01-arch-style.md`：

```markdown
## 架构风格识别报告

### 主要风格
**[风格名称]**（证据强度：强 / 中 / 弱）

证据：
- [具体目录或文件路径]
- [代码证据，含文件:行号]

### 次要风格特征（混合元素）
- [风格A]：[证据] — 局限于 [模块/子系统]
- [风格B]：[证据]

### 风格含义
**优势：** [这种风格带来什么好处]
**限制：** [这种风格带来什么约束]
**对导航代码的影响：** [新人应该从哪里开始读、沿哪个方向追踪]
**架构违规信号：** [是否有破坏该风格一致性的地方]
```

架构图另外**必须**展示：
- 所有主要模块 / 服务 / 子系统（带标签的框）
- API 边界（头文件层、AIDL、IDL、HTTP、gRPC）
- IPC / 通信机制（Binder、Socket、共享内存、消息队列）
- 数据存储（DB、缓存、文件系统、内核数据结构）
- 外部集成（HAL、驱动、第三方 SDK）
- 最重要的依赖 / 调用箭头

---

### Dimension 2：Concept — 领域概念

**认知目标**：提取代码库的词汇表和核心抽象。这是软件推理所依赖的"名词"世界。大脑在这里捕捉名词、寻找共性，形成文字影像和视觉形象。

**分析步骤：**

1. 扫描领域模型文件（entity、model、domain、schema、dto、struct 定义头文件）
2. 提取在类名 / 类型名 / 函数名中反复出现的名词
3. 对相关概念进行聚类
4. 找出概念层次（继承、组合、trait 实现、类型联合）
5. 识别跨模块普遍存在的核心抽象

```bash
# ── C：struct / typedef / enum 定义 ───────────────────────────

grep -rn "^typedef struct\|^struct [A-Z]\|^typedef enum\|^enum [A-Z]" \
  --include="*.h" --include="*.c" \
  --exclude-dir=.git | grep -v "test\|Test" | head -60

# C 函数指针类型（接口抽象）
grep -rn "typedef.*(\*[a-zA-Z_]*)(.*)" \
  --include="*.h" \
  --exclude-dir=.git | head -30

# ── C++：class / namespace / template ────────────────────────

grep -rn "^class \|^struct \|^namespace \|^template\s*<\|^using [A-Z]" \
  --include="*.h" --include="*.hpp" --include="*.cpp" --include="*.cc" \
  --exclude-dir=.git | grep -v "test\|Test\|mock\|Mock" | head -80

# 继承关系（概念层次）
grep -rn "class [A-Za-z].*: " \
  --include="*.h" --include="*.hpp" --include="*.cpp" \
  --exclude-dir=.git | grep -v "test\|Test" | head -40

# ── Rust：struct / enum / trait / impl ────────────────────────

grep -rn "^pub struct \|^pub enum \|^pub trait \|^struct \|^enum \|^trait " \
  --include="*.rs" --exclude-dir=.git | grep -v "test\|Test" | head -80

# trait 实现（类型关系）
grep -rn "^impl [A-Za-z].*for " \
  --include="*.rs" --exclude-dir=.git | head -40

# ── Java / Kotlin：class / interface / annotation ─────────────

grep -rn "^class \|^interface \|^enum \|^@interface \|^abstract class \|^data class \|^sealed class " \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | grep -v "test\|Test\|spec\|Spec\|mock\|Mock" | head -80

# ── 通用：高频名词（从文件名推断领域词汇）────────────────────

find . -type f \( -name "*.h" -o -name "*.hpp" -o -name "*.rs" \
  -o -name "*.java" -o -name "*.kt" -o -name "*.go" \) \
  -not -path "*/.git/*" -not -path "*/test*" \
  | xargs -I{} basename {} | sed 's/\.[^.]*$//' \
  | grep -v "^[a-z]" | sort | uniq -c | sort -rn | head -40

# ── 内核概念（如果是 Linux kernel）──────────────────────────

grep -rn "struct [a-z_]*_ops\b\|struct [a-z_]*_class\b" \
  --include="*.h" --exclude-dir=.git | head -30
```

**产出**：`mental-model/02-concept.md`

```markdown
# 领域概念图

## 核心概念（按字母序）

### ConceptName
**定义：** 这是什么东西？
**系统角色：** 这个概念在系统中承担什么工作？
**语言表示：** struct / class / trait / interface / typedef
**相关概念：** [[ConceptA]]（has-a），[[ConceptB]]（extends / impl）
**定义位置：** `path/to/file.h:42`

---

## 概念聚类

### [聚类名称]
- ConceptA → ConceptB（has-a 关系）
- ConceptB extends / impl ConceptC
- ConceptA 与 ConceptB 共有：[共同属性]

## 统一语言（Ubiquitous Language）
（在整个代码库中一致使用的核心词汇）
```

---

### Dimension 3：Flow — 时间流

**认知目标**：展示事物在时间轴上如何发生。一个请求、一个事件、一笔交易、一个中断、一个系统调用从开始到结束的故事。大脑在这里做时间线上的思考：How to do？

**分析步骤：**

1. 识别系统中 2-3 条最重要的"流"（根据语言/项目类型选择：syscall 路径、启动序列、请求生命周期、中断处理、Activity 生命周期、Binder 调用链等）
2. 对每条流，追踪函数调用、消息传递、状态转变的序列
3. 标记分支点（成功路径 / 错误路径）
4. 识别异步步骤（队列、回调、async/await、goroutine、workqueue、epoll）

```bash
# ── C/C++ 控制流：回调链 / 事件循环 ─────────────────────────

# 函数指针 / 回调注册（流的连接点）
grep -rn "\->.*=\s*&\?\s*[a-zA-Z_]\+;\|register_\|set_handler\|install_\|add_hook\|callback" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --exclude-dir=.git | grep -v "test\|Test" | head -40

# 事件循环 / 消息泵
grep -rn "poll\|epoll\|select\|event_loop\|run_loop\|dispatch\|libuv\|libevent" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --exclude-dir=.git | grep -v "test" | head -20

# ── Linux kernel 流：syscall / 中断 / 驱动 ───────────────────

# syscall 入口
grep -rn "SYSCALL_DEFINE\|asmlinkage" \
  --include="*.c" --exclude-dir=.git | head -20

# 中断处理链
grep -rn "request_irq\|irq_handler\|threaded_irq\|IRQF_\|ISR\|irqreturn_t" \
  --include="*.c" --include="*.h" --exclude-dir=.git | head -20

# 内核 workqueue / tasklet / softirq
grep -rn "INIT_WORK\|schedule_work\|tasklet_\|softirq\|INIT_DELAYED_WORK" \
  --include="*.c" --exclude-dir=.git | head -20

# ── Android 流：Activity 生命周期 / Binder / Intent ──────────

# Activity/Service 生命周期方法
grep -rn "onCreate\|onStart\|onResume\|onPause\|onStop\|onDestroy\|onBind\|onReceive" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | head -30

# Binder 调用链
grep -rn "Binder\|IBinder\|IInterface\|transact\|onTransact\|AIDL" \
  --include="*.java" --include="*.kt" --include="*.cpp" \
  --exclude-dir=.git | head -30

# Intent / Message / Handler 异步
grep -rn "sendMessage\|postMessage\|Handler\|Looper\|Intent\|startActivity\|sendBroadcast" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | grep -v "test\|Test" | head -30

# ── Rust 异步流 ────────────────────────────────────────────

grep -rn "async fn\|\.await\|tokio::\|futures::\|mpsc::\|channel\|Arc<Mutex" \
  --include="*.rs" --exclude-dir=.git | grep -v "test\|Test" | head -30

# ── 通用：中间件链 / 错误处理路径 ───────────────────────────

grep -rn "middleware\|handler\|controller\|usecase\|interactor" \
  --include="*.ts" --include="*.py" --include="*.go" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=node_modules --exclude-dir=.git | grep -v "test\|spec" | head -30

# 错误 / 恢复路径
grep -rn "catch\|except\|error\|recover\|rollback\|goto err\|return -E" \
  --include="*.c" --include="*.cpp" --include="*.rs" \
  --include="*.java" --include="*.kt" --include="*.go" \
  --exclude-dir=.git | grep -v "test" | head -30
```

**产出**：调用 `excalidraw-diagram` skill，产出 `mental-model/03-flow.excalidraw`（或多个文件）。

根据项目类型选择 2-3 条最有代表性的流：
- **Linux kernel**：syscall 调用链、驱动中断处理流、模块初始化序列
- **Android**：Binder IPC 调用链、Activity 启动流、系统服务请求路径
- **嵌入式 C**：main loop 事件分发、中断→回调→状态机
- **Rust 服务**：async 请求处理流、channel 消息流
- **Web 服务**：HTTP 请求生命周期、认证流程

每张流程图使用**时间线模式**：
- 主脊线 = 正常路径（happy path）
- 错误分支 = 从主脊线分叉出去的支线
- 异步边界 = 虚线分隔符
- 起点：发起者（用户 / 系统 / 中断 / 定时器）
- 终点：结果（响应 / 状态变化 / 副作用）

---

### Dimension 4：Spatial — 空间结构

**认知目标**：映射代码库的物理和逻辑组织——什么包含什么、什么与什么相邻、什么在上面、什么在下面。大脑在这里做空间关系的归纳思考：上下关系、兄弟关系、父子关系、协同关系。

**分析步骤：**

1. 将目录结构映射为树
2. 识别分层边界（UI → API → Service → Repository → DB；User space → Kernel space → Hardware）
3. 发现跨层依赖（架构违规）
4. 绘制模块耦合图（哪些模块引用哪些模块）
5. 识别中心节点（被最多模块依赖的模块）

```bash
# ── 目录树（2-3 层，排除构建产物）──────────────────────────

find . -type d -not -path "*/.*" \
  -not -path "*/node_modules/*" -not -path "*/target/*" \
  -not -path "*/build/*" -not -path "*/out/*" \
  -not -path "*/__pycache__/*" -not -path "*/dist/*" \
  | sort | head -80

# ── C/C++ 依赖：#include 分析 ─────────────────────────────

# 跨目录 include（揭示耦合）
grep -rn "^#include\s*\"" \
  --include="*.c" --include="*.cpp" --include="*.cc" \
  --include="*.h" --include="*.hpp" \
  --exclude-dir=.git | grep "\.\./\|src/\|lib/\|include/" | head -60

# 被最多文件 include 的头文件（中心节点）
grep -rh "^#include\s*\"" \
  --include="*.c" --include="*.cpp" --include="*.cc" \
  --exclude-dir=.git \
  | sed 's/#include\s*"\(.*\)"/\1/' \
  | sort | uniq -c | sort -rn | head -20

# ── Rust crate/module 依赖 ────────────────────────────────

grep -rn "^use \|^mod \|^extern crate " \
  --include="*.rs" --exclude-dir=.git \
  | grep -v "test\|spec" | head -60

# Cargo workspace 成员
find . -name "Cargo.toml" -maxdepth 4 | xargs grep -l "\[workspace\]\|\[package\]" 2>/dev/null | head -10

# ── Java/Android 包依赖 ───────────────────────────────────

grep -rn "^import " \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | grep -v "test\|Test\|java.util\|java.lang" | head -60

# Android Gradle 模块依赖
find . -name "build.gradle" -o -name "build.gradle.kts" \
  | xargs grep "implementation\|api\|compileOnly" 2>/dev/null | head -30

# ── 通用导入分析 ──────────────────────────────────────────

grep -rn "^import\|^from\|^require\|^use " \
  --include="*.ts" --include="*.py" --include="*.go" \
  --exclude-dir=node_modules --exclude-dir=.git | grep -v "test\|spec" | head -60
```

**产出**：`mental-model/04-spatial.md`

```markdown
# 空间结构

## 目录树（带注释）
```
kernel/                    ← Linux kernel 顶层
├── arch/                  ← [架构相关] x86/arm/riscv 等
├── drivers/               ← [驱动层] 设备驱动
│   ├── net/               ← 网络驱动
│   └── block/             ← 块设备驱动
├── fs/                    ← [文件系统层]
├── net/                   ← [网络子系统]
├── mm/                    ← [内存管理]
└── include/               ← [公共头文件] API 边界
```

## 分层模型

```
[ 用户空间 / UI / 应用层 ]
         ↓ syscall / Binder / JNI
[ 系统服务 / Framework / 中间件 ]
         ↓ HAL / 驱动接口
[ 内核 / 驱动层 ]
         ↓ 寄存器 / 总线
[ 硬件 ]
```

## 耦合分析

被依赖最多的模块（中心节点）：
1. `include/linux/sched.h` — 被 N 个文件 include
2. ...

## 空间异常
- 架构违规（跨层依赖）：[列举]
- 热点（高度耦合）：[列举]
- 孤立区域（低耦合孤岛）：[列举]
- 兄弟关系（同层协同）：[列举]
```

---

### Dimension 5：Decision — 决策知识

**认知目标**：浮现代码中隐含和显式的决策：为什么选择 X 而不是 Y？系统遵循什么规则？关键权衡点在哪里？大脑在这里做演绎推理。

**分析步骤：**

1. 阅读注释（尤其是 `// TODO`、`// FIXME`、`// WHY`、`// HACK`、`// BECAUSE`、`// NOTE`、`/* WORKAROUND */`）
2. 寻找 ADR（Architecture Decision Records）文件
3. 读取配置文件和 Kconfig，理解它们编码了什么选择
4. 找出 feature flag、环境变量、编译期开关（`#ifdef`、`cfg!`、`@Conditional`）
5. 识别显式对比了备选方案的地方

```bash
# ── ADR / 设计文档 ────────────────────────────────────────

find . \( -path "*/adr*" -o -path "*/decisions*" -o -path "*/rfcs*" \
  -o -path "*/docs/design*" -o -path "*/Documentation/*" \) \
  -not -path "*/.git/*" | head -20

# ── 有意义的注释（跨所有语言）────────────────────────────

grep -rn "TODO\|FIXME\|HACK\|NOTE\|WHY\|BECAUSE\|WORKAROUND\|TRADEOFF\|WARN\|DANGER\|SAFETY\|UNSAFE" \
  --include="*.c" --include="*.cpp" --include="*.h" --include="*.hpp" \
  --include="*.rs" --include="*.java" --include="*.kt" \
  --include="*.go" --include="*.py" --include="*.ts" \
  --exclude-dir=node_modules --exclude-dir=.git | head -60

# ── C/C++ 编译期决策 ──────────────────────────────────────

# 条件编译开关（重大决策点）
grep -rn "^#ifdef \|^#if defined\|^#ifndef " \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --exclude-dir=.git | grep -v "test\|Test" | head -40

# ── Linux kernel Kconfig（功能选项决策）────────────────────

find . -name "Kconfig" -maxdepth 5 | head -10
# 读取核心子系统的 Kconfig
find . -name "Kconfig" -maxdepth 3 | xargs grep "^config \|^menu \|depends on\|select " 2>/dev/null | head -50

# ── Rust 特性开关 ──────────────────────────────────────────

grep -rn "#\[cfg(\|cfg!\|#\[feature\|Cargo.toml" \
  --include="*.rs" --include="Cargo.toml" \
  --exclude-dir=.git | head -30

# ── 配置 / 环境变量 ───────────────────────────────────────

find . \( -name "*.env*" -o -name "config.*" -o -name "settings.*" \
  -o -name "constants.*" -o -name "defaults.*" \) \
  -not -path "*/node_modules/*" -not -path "*/.git/*" | head -15
```

**产出**：`mental-model/05-decision.md`

```markdown
# 决策知识

## 关键架构决策

### 决策：[名称]
**选择：** [做了什么选择]
**放弃的备选方案：** [考虑过但放弃的]
**理由：** [为什么做这个选择]
**后果：** [这个选择带来什么约束 / 好处]
**代码证据：** `path/to/file.c:42`

## 编译期决策（#ifdef / cfg! / Kconfig）

| 开关 | 默认值 | 控制的行为 | 影响范围 |
|------|--------|------------|----------|

## 技术债务登记

| 区域 | 债务 | 形成原因 | 影响 |
|------|------|----------|------|

## 配置知识

| 参数 | 默认值 | 作用 | 何时需要调整 |
|------|--------|------|-------------|

## 规则与不变量

- [规则1]：总是 [做 X]，因为 [原因]
- [不变量1]：[Y] 总是 [Z]，因为 [原因]
```

---

### Dimension 6：Evolution — 历史演化

**认知目标**：理解代码库如何随时间演变——什么成长了、什么被替换了、创始愿景与当前现实有什么差距。

**分析步骤：**

1. 分析 git log 中的重大里程碑
2. 找出最老和最新的文件
3. 识别变更最频繁的文件（热点）
4. 追踪核心抽象的演变
5. 检测被移除的功能或重大重构

```bash
# 全局提交历史（里程碑视图）
git log --oneline --graph --all | head -60

# 变更最频繁的文件
git log --name-only --format="" -- | sort | uniq -c | sort -rn | head -25

# 最近活动（3个月）
git log --since="3 months ago" --oneline | head -40

# 创始提交
git log --reverse --oneline | head -5

# 贡献者分布
git shortlog -sn | head -15

# 最近新增的文件
git log --diff-filter=A --name-only --format="" --since="6 months ago" | sort -u | head -20

# Android/Linux 特有：看 MAINTAINERS 了解历史维护者分工
head -100 MAINTAINERS 2>/dev/null
```

**产出**：`mental-model/06-evolution.md`

```markdown
# 历史演化

## 演化时间线

### 阶段 1：[名称]（[日期范围]）
[系统当时的面貌 / 聚焦点]
代表性提交：`[hash]` [消息]

## 变更热图（最近 6 个月）

变更最频繁的文件：
1. `path/to/file.c` — N 次变更 — [解读]

## 演化趋势

- **增长中：** [正在被添加的内容]
- **稳定中：** [基本没有变化的内容]
- **缩减 / 废弃中：** [正在被移除或替换的内容]

## 团队焦点

- 主要贡献者：[姓名 / 负责区域]
- 当前投入区域：[近期提交聚集在哪里]

## 架构演化观察

[从代码历史中能看到的架构层面的转变]
```

---

### Dimension 7：Concern — 关注点

**认知目标**：识别系统如何处理横跨多个模块的关注点。这个维度对系统语言项目尤其重要——**内存安全**、**并发安全**、**资源管理**是 C/C++/Rust 独有的第一类关注点。

核心概念：**Tangling（混杂）** 和 **Scattering（散布）**——业务逻辑与关注点代码是否交织在一起？

**每个关注点的分析框架：**
- **覆盖度**：系统是否处理了这个关注点？
- **集中度（Scattering）**：集中在一处，还是散落各处？
- **混杂度（Tangling）**：是否与业务逻辑纠缠在一起？
- **空白**：缺少什么？

```bash
# ── 内存管理（C/C++ 特有关注点）──────────────────────────

# 手动内存管理（潜在泄漏点）
grep -rn "\bmalloc\b\|\bcalloc\b\|\brealloc\b\|\bfree\b\|\bnew\b\|\bdelete\b\|\bkmalloc\b\|\bkfree\b\|\bvmalloc\b" \
  --include="*.c" --include="*.cpp" --include="*.cc" --include="*.h" \
  --exclude-dir=.git | grep -v "test\|Test" | head -40

# RAII / 智能指针（C++ 内存安全机制）
grep -rn "unique_ptr\|shared_ptr\|weak_ptr\|make_unique\|make_shared\|RAII\|Scoped\|Guard\|Lock\b" \
  --include="*.cpp" --include="*.cc" --include="*.h" --include="*.hpp" \
  --exclude-dir=.git | head -30

# Rust：unsafe 块（安全边界）
grep -rn "unsafe\b\|raw pointer\|*mut \|*const \|std::mem::" \
  --include="*.rs" --exclude-dir=.git | head -30

# ── 并发安全 ──────────────────────────────────────────────

# Linux kernel：锁原语
grep -rn "mutex_lock\|spin_lock\|rcu_read_lock\|down_read\|atomic_\|barrier\(\)\|smp_mb\|DEFINE_MUTEX\|DEFINE_SPINLOCK" \
  --include="*.c" --include="*.h" \
  --exclude-dir=.git | grep -v "test" | head -40

# C++ 并发
grep -rn "std::mutex\|std::lock_guard\|std::atomic\|std::thread\|pthread_\|std::unique_lock\|memory_order" \
  --include="*.cpp" --include="*.cc" --include="*.h" --include="*.hpp" \
  --exclude-dir=.git | head -30

# Rust 并发
grep -rn "Mutex\|RwLock\|Arc\|Atomic\|tokio::sync\|std::sync" \
  --include="*.rs" --exclude-dir=.git | grep -v "test" | head -30

# Java/Android 并发
grep -rn "synchronized\|volatile\|AtomicInteger\|ReentrantLock\|CountDownLatch\|Executor\|coroutines" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | grep -v "test\|Test" | head -30

# ── 安全（Security）────────────────────────────────────────

grep -rn "auth\|jwt\|token\|password\|encrypt\|hash\|sanitize\|escape\|csrf\|cors\|permission\|role\|acl\|rbac\|selinux\|capabilities\|seccomp" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --include="*.java" --include="*.kt" \
  --include="*.ts" --include="*.py" --include="*.go" \
  --exclude-dir=node_modules --exclude-dir=.git | grep -v "test" | head -40

# Android 权限
grep -rn "uses-permission\|checkPermission\|enforcePermission\|Manifest.permission" \
  --include="*.xml" --include="*.java" --include="*.kt" \
  --exclude-dir=.git | head -20

# ── 性能（Performance）────────────────────────────────────

grep -rn "cache\|memoize\|pool\|batch\|lazy\|prefetch\|zero-copy\|DMA\|SIMD\|vectorize\|O(n\|perf_event\|ftrace" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --include="*.java" --include="*.kt" --include="*.rs" \
  --exclude-dir=.git | grep -v "test\|Test" | head -30

# ── 可靠性（Reliability）──────────────────────────────────

grep -rn "retry\|circuit\|fallback\|timeout\|watchdog\|heartbeat\|recovery\|idempotent\|panic\|BUG_ON\|WARN_ON\|assert" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --include="*.rs" --include="*.java" --include="*.go" \
  --exclude-dir=.git | grep -v "test" | head -30

# ── 可观测性（Observability）──────────────────────────────

grep -rn "printk\|pr_info\|pr_err\|dev_dbg\|log\.\|logger\.\|metric\.\|trace\.\|tracepoint\|ftrace\|otel\|telemetry" \
  --include="*.c" --include="*.h" --include="*.cpp" \
  --include="*.java" --include="*.kt" --include="*.rs" \
  --include="*.ts" --include="*.py" \
  --exclude-dir=.git | grep -v "test" | head -30

# ── 测试覆盖 ──────────────────────────────────────────────

# 测试文件数量
find . \( -name "*_test.c" -o -name "*_test.cpp" -o -name "*test*.rs" \
  -o -name "*.test.ts" -o -name "*.spec.*" -o -name "*Test.java" \
  -o -name "*_unittest.*" -o -name "test_*.py" \) \
  -not -path "*/.git/*" | wc -l
```

**产出**：`mental-model/07-concern.md`

```markdown
# 关注点分析

## 关注点矩阵

| 关注点 | 覆盖度 | Scattering | Tangling | 缺口 |
|--------|--------|------------|----------|------|
| 内存安全 | 高/中/低 | 集中/散布 | 有/无 | [缺什么] |
| 并发安全 | ... | ... | ... | ... |
| 安全 | ... | ... | ... | ... |
| 性能 | ... | ... | ... | ... |
| 可靠性 | ... | ... | ... | ... |
| 可观测性 | ... | ... | ... | ... |
| 测试 | ... | ... | ... | ... |

## 深度分析

### 内存安全（Memory Safety）
[仅适用于 C/C++/Rust]
**代码位置：** [文件 / 模块]
**实现方式：** [RAII / 智能指针 / Rust 所有权 / 手动管理]
**Scattering 分析：** [malloc/free 是否配对？RAII 封装程度如何？]
**Tangling 分析：** [内存管理代码是否与业务逻辑混杂？]
**缺口：** [潜在内存泄漏点、use-after-free 风险区域]

### 并发安全（Concurrency Safety）
[仅适用于多线程/异步项目]
**锁原语：** [使用了哪些锁？]
**无锁机制：** [RCU / atomic / lock-free 数据结构]
**潜在竞争条件：** [共享状态、缺少保护的区域]

### 安全（Security）
...

### 性能（Performance）
...

### 可靠性（Reliability）
...

### 可观测性（Observability）
...
```

---

### Dimension 8：Feature/Function — 功能

**认知目标**：从功能视角识别和总结软件能做什么。这是在最高抽象层次上的"what"——核心功能的归纳和抽象。

**分析步骤：**

1. 阅读 README、文档、产品描述、内核 Documentation/
2. 枚举所有面向用户 / 操作者 / 上层软件的功能
3. 按用户角色 / 领域区域 / 子系统分组
4. 识别"杀手级功能"——这个软件独特擅长的事
5. 明确系统边界：有意不做什么

```bash
# 文档目录
find . -name "*.md" -o -name "*.rst" -o -name "*.txt" \
  | grep -v ".git\|node_modules\|target" \
  | grep -i "readme\|doc\|guide\|manual\|changelog\|overview" \
  | head -20

# Linux kernel Documentation
find . -path "*/Documentation/*.txt" -o -path "*/Documentation/*.rst" \
  | head -20

# CLI 命令 / API 入口
grep -rn "add_command\|add_parser\|Cmd\|cobra\.\|click\.\|@app\.command\|DEFINE_COMMAND" \
  --include="*.c" --include="*.go" --include="*.py" \
  --exclude-dir=.git | head -20

# Android Feature Flags
grep -rn "feature\|flag\|toggle\|experiment\|FeatureFlag\|isEnabled" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | grep -v "test\|Test" | head -20

# Kconfig 功能选项（Linux 内核功能目录）
find . -name "Kconfig" -maxdepth 4 \
  | xargs grep "^config \|tristate\|bool\|help" 2>/dev/null | head -60
```

**产出**：`mental-model/08-feature-function.md`

```markdown
# 功能与特性

## 核心价值主张

[一句话：这个软件是什么？给谁用？解决什么问题？]

## 用户画像 / 使用者

- **[画像名]：** [角色、需求、主要操作]（例：内核开发者 / App 开发者 / 终端用户）

## 功能目录

### [功能名称]
**描述：** 做什么
**入口点：** `path/to/file.c:func_name`
**核心组件：** [列表]
**成熟度：** 核心功能 / 实验性 / 废弃中

## 功能全图（按子系统 / 领域分组）

### [子系统 / 领域 1]
- ✓ [功能] — [一行描述]
- ○ [功能] — [部分实现或计划中]

## 系统边界

**有意不做的事：** [这个软件明确不处理什么]
```

---

### Dimension 9：Pattern — 模式

**认知目标**：识别代码库中使用的架构风格、设计模式、惯用法和算法。模式是抽象思考和归纳思考的产物——一种"见过就认得出"的预先存在的知识结构。

**分析步骤：**

1. 识别整体架构风格（宏内核 / 微内核 / 事件驱动 / 分层 / 组件化等）
2. 发现语言惯用模式（RAII、CRTP、Pimpl、Newtype、Typestate 等）
3. 识别系统级模式（Device Model、kobject、RCU、Workqueue、Binder 等）
4. 找出领域特有算法（系统如何解决它最难的问题？）
5. 识别编码惯用法和约定

```bash
# ── C 设计模式 ────────────────────────────────────────────

# Opaque pointer / 句柄模式（C 的封装惯用法）
grep -rn "typedef struct [a-zA-Z_]* \* [a-zA-Z_]*;\|void \*priv\|void \*data\|void \*ctx" \
  --include="*.h" --exclude-dir=.git | head -20

# C vtable（多态实现）
grep -rn "struct.*_ops\b\|struct.*_vtable\|struct.*_interface\|function pointer table" \
  --include="*.h" --include="*.c" --exclude-dir=.git | head -20

# 状态机（嵌入式 / 驱动常见）
grep -rn "state\|STATE\|switch.*state\|enum.*state\|state_machine\|fsm" \
  --include="*.c" --include="*.h" --exclude-dir=.git \
  | grep -v "test" | head -30

# ── C++ 模式 ──────────────────────────────────────────────

# RAII（资源获取即初始化）
grep -rn "class.*Guard\|class.*Scoped\|class.*Lock\|class.*Handle\|~[A-Z].*free\|~[A-Z].*close\|~[A-Z].*release" \
  --include="*.h" --include="*.hpp" --include="*.cpp" \
  --exclude-dir=.git | head -20

# CRTP（Curiously Recurring Template Pattern）
grep -rn "template.*class.*:.*public.*<.*>" \
  --include="*.h" --include="*.hpp" \
  --exclude-dir=.git | head -15

# Pimpl（pointer to implementation）
grep -rn "class.*Impl\b\|struct.*Impl\b\|unique_ptr.*Impl\|d_ptr\|pImpl" \
  --include="*.h" --include="*.hpp" --include="*.cpp" \
  --exclude-dir=.git | head -15

# ── Rust 模式 ──────────────────────────────────────────────

# Newtype 模式
grep -rn "^pub struct [A-Z][a-zA-Z]*([^)]*);$" \
  --include="*.rs" --exclude-dir=.git | head -20

# Builder 模式
grep -rn "impl.*Builder\|fn build(self)\|\.build()" \
  --include="*.rs" --exclude-dir=.git | head -20

# Typestate 模式（编译期状态机）
grep -rn "PhantomData\|std::marker::PhantomData\|struct.*<State>" \
  --include="*.rs" --exclude-dir=.git | head -15

# Error 枚举 / thiserror / anyhow
grep -rn "thiserror\|anyhow\|#\[derive.*Error\]\|impl.*Error for" \
  --include="*.rs" --include="Cargo.toml" --exclude-dir=.git | head -20

# ── Linux kernel 模式 ─────────────────────────────────────

# kobject / device model
grep -rn "kobject\|kref\|device_register\|driver_register\|bus_register\|class_create" \
  --include="*.c" --include="*.h" --exclude-dir=.git | head -20

# RCU（Read-Copy-Update）
grep -rn "rcu_read_lock\|rcu_assign_pointer\|rcu_dereference\|synchronize_rcu\|call_rcu" \
  --include="*.c" --include="*.h" --exclude-dir=.git | head -20

# Workqueue / delayed work
grep -rn "INIT_WORK\|queue_work\|schedule_work\|INIT_DELAYED_WORK\|hrtimer\|timer_list" \
  --include="*.c" --exclude-dir=.git | head -20

# ── Android 模式 ─────────────────────────────────────────

# MVVM / ViewModel / LiveData
grep -rn "ViewModel\|LiveData\|MutableLiveData\|Flow\|StateFlow\|Repository" \
  --include="*.java" --include="*.kt" \
  --exclude-dir=.git | head -20

# Binder 服务模式
grep -rn "extends.*Stub\|implements.*Binder\|IBinder\|onTransact\|addService\|getService" \
  --include="*.java" --include="*.kt" --include="*.cpp" \
  --exclude-dir=.git | head -20

# ── 通用模式 ──────────────────────────────────────────────

# Factory / Builder / Observer（GoF）
grep -rn "Factory\|Builder\|createInstance\|newInstance\|Create[A-Z]\|EventEmitter\|subscribe\|publish\|emit\|listener" \
  --include="*.c" --include="*.cpp" --include="*.h" \
  --include="*.rs" --include="*.java" --include="*.kt" \
  --include="*.go" --include="*.ts" --include="*.py" \
  --exclude-dir=node_modules --exclude-dir=.git | head -30

# 非平凡算法 / 数据结构
grep -rn "sort\|search\|hash\|tree\|graph\|queue\|priority\|bloom\|consistent\|b-tree\|skip.list\|red.black\|AVL\|trie" \
  --include="*.c" --include="*.cpp" --include="*.rs" \
  --include="*.go" --include="*.java" \
  --exclude-dir=.git | grep -v "test\|Test" | head -20
```

**产出**：`mental-model/09-pattern.md`

```markdown
# 模式目录

## 架构风格

**主要风格：** [例：宏内核 / 分层驱动模型 / 组件化 Android / 事件驱动]
**证据：** [关键结构证据]
**含义：** [这对代码库意味着什么——优势、限制]

## 语言惯用模式

### [模式名称]（语言 / 分类）
**使用位置：** `path/to/file.h` 及 N 处
**使用方式：** [简述]
**为何合适：** [理由]

例：
- **RAII**（C++）：`src/base/ScopedFd.h` — 自动关闭文件描述符
- **Opaque Pointer**（C）：`include/device.h` — 隐藏驱动内部状态
- **Newtype**（Rust）：`src/types.rs` — 类型安全的 ID 封装
- **kobject**（Linux kernel）：贯穿 drivers/ — 统一引用计数和 sysfs 集成

## 系统级模式

[Linux kernel / Android / 嵌入式特有模式：Device Model、RCU、Binder、Workqueue 等]

## 算法与数据结构

[任何非平凡算法：用途、复杂度、位置]

## 惯用法与约定

[反复出现的编码惯例：错误返回风格（负 errno / Option<T> / Result<T>）、命名约定、注释风格等]

## 模式空白

[可能改善代码库但当前缺失的模式]
```

---

### Phase 3：综合（Synthesis）

完成全部 9 个维度后，写入 `mental-model/10-synthesis.md`：

```markdown
# 心智模型综合

## 30 秒版本

[3-5 句话：这是什么软件、怎么构建的、关键特征是什么？]

## 心智模型地图

```
        [ Arch 架构 ]              [ Concept 概念 ]
        模块与组件关系               领域词汇与抽象
              \                        /
               \                      /
           [ Spatial 空间结构 ]
           层次、包含、协同关系
               /                      \
              /                        \
        [ Flow 时间流 ]          [ Pattern 模式 ]
        请求 / 事件序列              架构风格与设计模式
              \                        /
               \                      /
          [ Concern 关注点 ]  [ Feature 功能 ]
          内存/并发/安全/性能       核心能力与价值
                      \        /
                       \      /
                 [ Decision 决策 ]
                 选择、权衡、规则
                         |
                 [ Evolution 演化 ]
                 历史、趋势、方向
```

## 关键洞察

1. [关于这个代码库最重要的洞察]
2. [第二个洞察]
3. [第三个洞察]

## 风险与机会

| 风险 | 可能性 | 影响 | 建议缓解措施 |
|------|--------|------|-------------|
| ... | ... | ... | ... |

## 新工程师学习路径

1. 从这里开始：[维度] → 读 [文件]
2. 然后：[维度] → 读 [文件]
3. 然后：...

## 待向团队询问的问题

[从代码本身无法确定、架构师应该主动询问的事项]
```

---

## 产出文件清单

```
mental-model/
  00-overview.md          ← 仓库画像（Phase 0）
  01-arch.html            ← 架构图（HTML + SVG，深色主题，布局跟随风格）
  01-arch-style.md        ← 架构风格识别报告（主要风格 + 证据 + 含义）
  02-concept.md           ← 领域概念词汇表
  03-flow.excalidraw      ← 时间流图（可多个）
  04-spatial.md           ← 空间结构树与分层图
  05-decision.md          ← 决策知识（ADR、权衡、规则）
  06-evolution.md         ← 历史演化时间线
  07-concern.md           ← 关注点矩阵（内存/并发/安全/性能等）
  08-feature-function.md  ← 功能目录
  09-pattern.md           ← 模式目录
  10-synthesis.md         ← 综合心智模型
```

---

## 执行顺序

按以下顺序执行，每个维度为后续维度奠基：

```
00-overview → 04-spatial → 01-arch → 02-concept
           → 08-feature → 03-flow
                       → 09-pattern → 07-concern → 05-decision
06-evolution（可在任意时刻并行执行）
10-synthesis（最后）
```

---

## 质量标准

### 每个维度
- 用具体的文件路径和行号支撑关键判断
- 区分"发现的"与"推断的"：对不确定内容用"可能是…"、"推测…"
- 每个维度至少 3 条实质性观察（不是能套用到任何仓库的泛泛之词）

### 语言自适应
- 确认主要语言后，**重点使用对应的 grep 命令**，不要对 C 项目花时间搜索 ORM 或 HTTP 路由
- 对于 C/C++ 项目，内存管理和并发安全是**第一类关注点**，必须深入分析
- 对于 Linux kernel，Kconfig、MAINTAINERS、Documentation/ 是不可忽略的信息源
- 对于 Android，AIDL、AndroidManifest.xml、build.gradle 模块划分是架构分析核心

### 视觉产出
- `01-arch.html`：必须展示至少 5 个组件及其关系，渲染后检验
- `03-flow.excalidraw`：必须覆盖至少 2 条最重要的系统流

### 文字产出
- 任何维度都不应为空——信息不可获得时说明原因
- 引用的概念在代码中有可追溯的证据

### 整体
- `10-synthesis.md` 的"30 秒版本"必须对从未看过这个仓库的工程师有意义
- "新工程师学习路径"必须是可实际操作的具体步骤
