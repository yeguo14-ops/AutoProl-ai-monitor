# AotuPC_ai_monitor · 全栈资源监控平台 · 架构与技术点解读

> 自我讲述：理解每个模块做什么、用了什么技术、为什么这么设计。
> 全文约 2000 字，读完能对着架构图讲清楚整条链路。

---

## 一、这个平台是做什么的

一句话：**一个跨语言的服务器资源监控与智能告警平台**。它持续采集各被监控主机的 CPU / 内存 / 磁盘 / 系统日志，落库存储，用 AI（大模型）对日志做语义归类与风险判断，达到阈值或检测到高风险时自动发邮件告警，并提供 Web 页面展示。

它不是一个单语言单进程程序，而是 **Python + Java + MySQL 三种技术栈的分布式协作**，用 **Docker 容器编排**把它们组织起来。

## 二、整体架构：三容器 + 一个 Agent

```
┌─────────────── 被监控主机（Agent 模式）───────────────┐
│  agent.py（Python，systemd 守护）→ 采集指标+日志 → HTTP 上报  │
└────────────────────────┬──────────────────────────────┘
                         │ POST /api/report
┌────────────────────────▼──────────────────────────────┐
│  docker-compose 三容器（控制端）                        │
│                                                        │
│  [web]  Python: Flask 展示 + 主循环采集调度 + 告警      │
│    │      （main.py 采集 / app.py Flask / alerter 告警）│
│    │ HTTP /api/ai/analyze/log                          │
│  [java-ai] Java Spring Boot: 调用 DeepSeek 大模型分析日志│
│    │                                                   │
│  [mysql] MySQL 8：指标与主机历史数据存储                │
└────────────────────────────────────────────────────────┘
```

关键设计：**采集有两种模式**。一是控制端容器内由 `main.py` 主循环自己采集（`RUN_MONITOR_SCHEDULER=true`）；二是**被监控主机上装 Agent**（`agent.py`），它独立采集并通过 HTTP 把数据上报给控制端 Web。第二种是生产推荐做法——控制端容器不该拿自己的容器 ID 当主机名，而且可以横向监控任意多台机器。

## 三、各模块与技术点拆解（每个 part 做了什么）

### 1. Python 监控核心（monitor_project）

按**三层分层**组织，对齐 Spring 的 Controller/Service/DAO：

- **入口层（根目录）**

  - `main.py`：采集入口1。`sys.pycache_prefix=./cache/` 控制字节码缓存位置；启动时加载 `.env`、打日志、进 `main_loop()`。
  - `app.py`：Flask Web 服务，对外提供 `/api/hosts`、`/api/report`、`/health` 等 REST 接口，渲染 `templates/` 下的页面。**Agent 就是往这里的 `/api/report` 上报数据。**
  - `agent.py`：被控端 Agent，采集本机指标后 POST 到控制端，采集入口2。

- **业务层（monitor/services/）**——只做判断，不直接碰数据库/邮件

  - `scheduler.py`：主循环调度，按 `COLLECT_INTERVAL` 周期驱动采集→落库→告警，是"心脏"。
  - `collector.py`：用 **psutil** 采集 CPU / 内存 / 磁盘 / 系统日志尾部；把日志交给 AI 分析，并融合 Java 侧返回的动态 CPU 基线。
  - `alerter.py`：告警决策。**融合动态阈值 + AI 风险等级**：若 Java 侧返回了动态 CPU 基线则用动态值，否则用环境变量静态阈值；CPU/内存/磁盘逐项比较，AI 判定 HIGH/CRITICAL 时追加"高风险日志"告警。
  - `ai_analyzer.py`：本地关键词兜底分析（error/timeout/oom/denied），当 AI 服务不可用时降级。

- **数据层（monitor/repo/）**——只负责读写外部资源

  - `storage.py`：数据落库 MySQL；写失败时降级到本地 JSON 文件缓存并支持重试，保证数据不丢。
  - `redis_client.py`：**Redis** 连接 + **告警冷却防刷**（5 分钟内同一类型告警只发一次，靠 `last_{type}_alert_time` 键实现）。
  - `email_sender.py`：**smtplib** 发送告警邮件，支持失败重试、HTML 正文格式化、AI 增强告警内容。
  - `ai_java_client.py`：用 **requests/HTTP** 调用 Java 服务的 `/api/ai/*` 接口，并支持 `JAVA_AI_IGNORE_ENV_PROXY` 避免走全局代理。
  - `log_tail.py`：读取系统日志尾部（`SYSLOG_TAIL_LINES` 行）。
  - `hostname.py`：解析主机名，识别 Docker 容器 ID 避免误当主机名。
  - 根 `logger_config.py`：统一日志配置，写 `logs/monitor.log`。

### 2. Java AI 服务（ai_monitor_service）

一个 **Spring Boot 3.2 + Java 17 + Gradle** 的独立微服务，专门负责"把日志交给大模型分析"：

- `controller/AIController`：REST 入口。`POST /api/ai/analyze/log`（日志分析）、`GET /api/ai/baseline/cpu`（CPU 基线）。Controller 只接收转发，不写业务。
- `service/DeepSeekService`：核心。用 **WebFlux WebClient** 发起 HTTPS 请求调用 **DeepSeek 大模型**；用 **@Cacheable + Caffeine** 缓存分析结果（缓存键是整条日志的 **SHA-256**，由 `LogAnalysisCacheKeyGenerator` 生成，避免 hashCode 碰撞）；用 `Mono` + `boundedElastic` 避免阻塞 Servlet 线程；对模型返回非法 JSON / 调用失败 / 中文 errorType 都有兜底降级。**API Key 通过环境变量注入，日志脱敏，不硬编码。**
- `model.dto` / `client`：接口 DTO（AiLogAnalysisRequest/Response、CpuBaselineResponse）与 DeepSeek 平台请求/响应结构。

### 3. 数据存储（MySQL 8）

`storage.py` 把每台主机每轮采集的指标写入 MySQL（表按主机、时间组织），Web 端从库里读数据做展示，历史数据可回看。

## 四、部署与一键启动

**Docker 多容器编排（compose v3.3，兼容旧版 docker-compose）**：

```bash
cp docker/defaults.compose.env .env   # 首次：生成配置（端口/密码/镜像名）
docker-compose build web && docker-compose up -d
```

三服务依赖链：`mysql → java-ai → web`（`depends_on` 列表顺序保证先起数据库再起业务）。web 容器 `env_file: .env` 注入所有配置，入口 `docker/entrypoint.sh`：后台跑 `python main.py`（调度，可选）+ 前台 `python app.py`（Flask）。Java 镜像 `ai-monitor-service:latest`，Python 用本仓库 Dockerfile 多阶段构建。端口做了宿主映射（MySQL 3307 / Flask 5000 / Java 8080），避免与本机冲突。

**Ansible 自动化部署**：`ansible/deploy.yml` 用 playbook 在目标 Ubuntu 上装 Docker、同步整个项目、执行 `docker-compose up -d --build`，实现"根目录一键部署到任意机器"。这就是你说的"Ansible 一键启动"——它把 Docker 的编排能力再包一层，变成**可复制、可批量、声明式**的部署。

> 小提示：当前 `deploy.yml` 的实现在被控端装 Agent（Python 上报模式）；`ansible/README.txt` 里描述的另一套"装 Docker + compose up"是更完整的控制端批量部署方案，面试时两种都能讲，但以代码实际为准。

## 五、完整数据流转时序

1. **Agent 侧**（被控主机）：`agent.py` 每 60s 用 psutil 采集指标 + 系统日志尾部 → `POST http://控制端:5000/api/report` 上报。
2. **控制端接收**：`app.py` 的 `/api/report` 接口收到数据，交给业务层处理。
3. **AI 分析**：`collector.py` 把日志文本交给 `ai_java_client` → 调 Java `/api/ai/analyze/log` → `DeepSeekService` 调 DeepSeek 大模型 → 返回 `{errorType, riskLevel, possibleCause, suggestion}`（结果在 Java 侧被 Caffeine 缓存，同一日志不重复付费调用）。若 Java 不可用，Python 侧 `ai_analyzer` 用关键词兜底。
4. **落库**：指标 + AI 结论经 `storage.py` 写入 MySQL（写失败降级本地 JSON + 重试）。
5. **告警决策**：`alerter.py` 比较 CPU/内存/磁盘与阈值（优先动态基线），并检查 AI riskLevel；触发后经 `redis_client` 冷却防刷 → `email_sender` 发邮件。
6. **展示**：`app.py` 从 MySQL 读历史数据，渲染 Web 页面供查看。

## 六、技术点速查表

| 技术点                             | 用在哪里                         | 解决什么问题              |
| ------------------------------- | ---------------------------- | ------------------- |
| Python psutil                   | collector/agent 采集           | 跨平台取 CPU/内存/磁盘      |
| Flask + 模板                      | app.py Web 展示                | REST 接口 + 页面渲染      |
| PyMySQL                         | storage 落库                   | 写 MySQL，失败降级文件缓存    |
| Redis + 冷却键                     | redis_client                 | 告警防刷，避免 5 分钟内重复轰炸   |
| smtplib + 重试                    | email_sender                 | 发告警邮件，失败自动重试        |
| 动态阈值 vs 静态阈值                    | alerter                      | 用 AI 基线自适应，否则用配置兜底  |
| Java Spring Boot + WebClient    | DeepSeekService              | 异步 HTTPS 调大模型，不阻塞线程 |
| @Cacheable + Caffeine + SHA-256 | DeepSeekService/KeyGenerator | 缓存分析结果，避免重复付费与碰撞    |
| 降级兜底                            | ai_analyzer / Service 异常处理   | AI 挂了系统仍可用          |
| docker-compose 三容器              | 部署编排                         | 异构语言互通，依赖有序         |
| Ansible                         | 批量部署                         | 一键把平台装到多台服务器        |
| 环境变量注入密钥                        | .env / 环境变量                  | 密钥不入库、日志脱敏          |

## 七、一句话总结

> "实现了一个跨语言的服务器监控告警平台：**Python** 负责采集、落库、告警与 Web 展示，**Java(Spring Boot)** 负责调用 **DeepSeek 大模型**做日志 AI 分析，**MySQL** 存储、**Redis** 做告警防刷，三者在 **docker-compose** 中编排成 mysql/java-ai/web 三容器协作，再用 **Ansible** 一键批量部署到多台服务器。技术点覆盖 psutil 采集、WebClient 异步调用大模型、Caffeine+@Cacheable 缓存、邮件告警、降级兜底与动态阈值告警。"
