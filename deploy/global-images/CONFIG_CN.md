# 三件套配置说明（默认开 / 关）

本文说明 `deploy/global-images` 一键部署用到的配置：哪些默认打开、哪些默认关闭、各自做什么。

改配置的入口：

| 改哪里 | 生效方式 |
|---|---|
| `.env` | 改完后重新 `./start-all.sh`（会重建容器，volume 数据保留） |
| 启动命令前缀（如 `PULL=1`、`PROXY_FULL_STACK=0`） | 当次启动生效 |
| `start-proxy.sh` / `start-memory-core.sh` 生成的 YAML | **不要手改**，每次启动都会覆盖 |

完整 LLM 字段示例见 [`.env.example`](./.env.example)。Proxy 全量 YAML 字段见 [`MemoryProxy/config.example.yaml`](../../MemoryProxy/config.example.yaml)。

---

## 1. 能力开关一览（先看这张表）

以 **`./start-all.sh`** 为准（推荐用法）。单独跑 `./start-proxy.sh` 时，表里标了差异。

| 能力 | `start-all.sh` 默认 | 单独 `start-proxy.sh` 默认 | 作用 | 怎么改 |
|---|---|---|---|---|
| 上游 LLM 转发 | **开** | **开** | 把 Claude Code 等请求转到 `PROXY_UPSTREAM_*` | 必填，关了 proxy 没意义 |
| Auth（校验 user_key） | **开** | 关 | 用 `sk-mem-...` 调 Core `/v3/meta/auth/verify` 得到 user_id | `PROXY_ENABLE_AUTH` / `PROXY_FULL_STACK` |
| Session Init（选 Team/Agent/Task） | **开** | 关 | 新会话弹表单绑定资产；依赖 Auth | `PROXY_ENABLE_SESSION_INIT` |
| Tdai 记忆注入（L0 写入 / L2L3 注入） | **开** | 关 | 对话写入 Core，并把画像/场景注入 prompt | `PROXY_ENABLE_TDAI` |
| Skill 注入（`<cloud_skills>` / `<skill_tools>`） | **开** | **开** | 把可复用 skill 注入 system prompt | `injection.injectors` 含 `skill`（脚本写死） |
| Knowledge 工具注入（`<knowledge_tools>`） | **关** | **关** | 把 wiki / code-graph 的 tools 入口注入 prompt，供 Agent curl KS | 见 [§4.3](#43-knowledge-注入默认关) |
| Cost Guard | **关** | **关** | 按费用/模型做路由与拦截 | 脚本写死 `costGuard.enabled: false` |
| Redis | **关** | **关** | 多副本共享 session / 限流计数 | 脚本写死 `redis.enabled: false` |
| Gateway Bearer 鉴权 | **关** | **关** | Core 要求 `Authorization: Bearer` | `.env` 里 `MEMORY_CORE_GATEWAY_API_KEY` 留空 |
| 记忆抽取（L0→L1→L2→L3） | **开** | （Core 侧，与 proxy 无关） | 后台把对话抽成记忆 | Core 生成配置，见 [§3](#3-memory-core-生成配置默认开) |
| Skill 抽取 | **开** | （Core 侧） | 从对话归档并抽出 skill | 同上 |
| 向量 Embedding | **关** | （Core 侧） | 记忆/skill 向量检索 | `embedding.provider: none`，现用 BM25 |
| LLM 全文 I/O 日志 | **开** | **开** | 把每次 LLM 入参/出参打到日志 | `TDAI_LLM_LOG_IO=0` 关闭 |
| 启动时拉最新镜像 | **关** | **关** | `docker pull` 覆盖本地 tag | `PULL=1 ./start-all.sh` |

`PROXY_FULL_STACK=1` 会同时打开 Auth + Session Init + Tdai。`start-all.sh` 默认就是 `1`；只跑 `start-proxy.sh` 时默认 `0`（纯转发 + Skill 注入，不做鉴权/选团队/记记忆）。

关掉完整流水线：

```bash
PROXY_FULL_STACK=0 ./start-proxy.sh
```

---

## 2. `.env` 配置项

### 2.1 必填（没有默认值，填 `REPLACE_ME` 会拒绝启动）

两组 LLM **互相独立**，可以指向同一家，也可以 memory 用便宜模型、proxy 用强模型。

| 变量 | 作用 |
|---|---|
| `MEMORY_LLM_BASE_URL` | Core + Hub（wiki ingest / 记忆抽取）调用的 OpenAI 兼容地址 |
| `MEMORY_LLM_API_KEY` | 上一行端点的 Key |
| `MEMORY_LLM_MODEL` | 内部模型 ID |
| `PROXY_UPSTREAM_URL` | 用户对话经 Proxy 转发到的上游 LLM |
| `PROXY_UPSTREAM_API_KEY` | 上游 Key（替换客户端带来的 Authorization） |
| `PROXY_UPSTREAM_MODEL` | 面向用户的模型 ID（Claude Code `--model` 应对齐这个） |
| `KNOWLEDGE_PUBLIC_BASE_URL` | 写入知识资源 `service_url` 的对外地址，**必须含 `/v3`**。Agent 用它 curl `tools/list` / `tools/call`。Core **不会**访问这个地址 |

本机 Docker 示例：`http://host.docker.internal:8424/v3`  
前面挂 Nginx（80 统一入口）示例：`http://10.13.7.104/v3`

### 2.2 有默认值，一般不用改

| 变量 | 默认 | 作用 |
|---|---|---|
| `MEMORY_CORE_IMAGE` / `MEMORY_HUB_IMAGE` / `PROXY_IMAGE` | Docker Hub `:latest` | 镜像。本地构建可改 `:local` |
| `MEMORY_LLM_PROTOCOL` | `openai` | Hub 调 LLM 的协议：`openai` 或 `anthropic` |
| `MEMORY_CORE_PORT` | `8420` | Core 宿主机端口，**不要对公网开放** |
| `PANEL_PORT` | `8125` | Panel UI |
| `KNOWLEDGE_PORT` | `8424` | Knowledge API |
| `PROXY_PORT` | `8096` | Proxy（coding agent 入口） |
| `MEMORY_CORE_VOLUME` | `tdai-memory-core-data` | Core 数据卷（SQLite / 记忆） |
| `PANEL_VOLUME` | `tdai-panel-data` | Hub 数据卷（wiki / code-graph） |
| `MEMORY_CORE_ADMIN_USERNAME` | `admin` | 首次 `init-admin` 的用户名 |
| `MEMORY_PROMPT_MODE` | `chat` | 记忆抽取风格。`chat`=通用对话；`code`=偏代码改动/工具用法（闲聊可能抽出 0 条） |
| `MEMORY_EMBEDDING_PROVIDER` | `none`（关） | 外部 embedding。`openai` 表示 OpenAI 兼容 `/v1/embeddings` |
| `MEMORY_EMBEDDING_BASE_URL` | 复用 `MEMORY_LLM_BASE_URL` | embeddings 地址，到 `/v1` 这一层 |
| `MEMORY_EMBEDDING_API_KEY` | 复用 `MEMORY_LLM_API_KEY` | embeddings Key |
| `MEMORY_EMBEDDING_MODEL` | 空 | 模型 ID，例如 `harrier-oss-v1-0.6b` |
| `MEMORY_EMBEDDING_DIMENSIONS` | `0` | 向量维数，必须和模型输出一致（该模型实测 1024） |
| `MEMORY_EMBEDDING_SEND_DIMENSIONS` | `false` | 请求体是否带 `dimensions`；非 OpenAI 官方模型一般关 |

### 2.3 默认关闭 / 留空（有意如此）

| 变量 | 默认 | 作用 | 什么时候改 |
|---|---|---|---|
| `MEMORY_CORE_GATEWAY_API_KEY` | **空** | Core 的 Bearer 门。空 = 关闭门（本地/当前开源部署必须如此） | **不要填**。Proxy 调 `/v3/meta/auth/verify` 目前不带 Bearer，填了 Auth / Session Init 会失败 |
| `MEMORY_HUB_PROXY_PUBLIC_URL` | **未设置** | Panel 卡片的 Proxy 根 URL，**同时**写入 Proxy `injection.externalGatewayUrl`（prompt 里 bridge curl 的 base） | 未设时卡片探测 LAN IP；bridge curl 回落容器网卡 IP。公司 Nginx / 域名入口**必须显式写 origin**（不要带 `:8096` 或路径），例如 `https://memhub.example.com`。设成 `=""` 则卡片回落到 gateway 地址 |

admin 的 `user_key` **不是** `.env` 项：首次启动随机生成，写在 `./.admin-key`。

### 2.4 启动时环境变量（不必写进 `.env`）

| 变量 | 默认 | 作用 |
|---|---|---|
| `PULL` | `0`（关） | `1` 时先 `docker pull` 三个镜像 |
| `PROXY_FULL_STACK` | `start-all.sh` 为 `1`；单独 `start-proxy.sh` 为 `0` | 一键开关 Auth + Session Init + Tdai |
| `PROXY_ENABLE_AUTH` | 随 FULL_STACK | 只开鉴权 |
| `PROXY_ENABLE_SESSION_INIT` | 随 FULL_STACK | 开了会**自动**把 Auth 也打开 |
| `PROXY_ENABLE_TDAI` | 随 FULL_STACK | 记忆读写 / L2L3 注入 |
| `TDAI_LLM_LOG_IO` | `1`（开） | Core / Hub / Proxy 的 LLM 全文入参出参日志。`0` / `false` / `off` / `no` 关闭 |

---

## 3. Memory Core 生成配置（默认开）

`start-memory-core.sh` 每次启动覆盖 `.memory-core-config/tdai-gateway.yaml`。standalone + 本地 SQLite，**不依赖 Redis / VDB / COS**。

| 项 | 默认 | 作用 |
|---|---|---|
| `deployMode` | `standalone` | 单机模式 |
| `stateBackend` | `local` | 状态存在本地，不走 Redis |
| `memory.capture.enabled` | **开** | 接收并保存 L0 对话 |
| `memory.extraction.enabled` | **开** | L1 原子记忆抽取 + 去重 |
| `memory.recall.enabled` | **开** | 召回已有记忆 |
| `memory.pipeline.enableWarmup` | **开** | 流水线预热 |
| `skill.enabled` | **开** | Skill 模块 |
| `skill.extraction.enabled` | **开** | 后台从对话抽 skill |
| `memory.embedding.provider` | **`.env` 未配则为 `none`（关）** | 配了 `MEMORY_EMBEDDING_*` 则走外部向量；skill 检索仍是 BM25 |
| `memory.storeBackend` | `sqlite` | 记忆落盘 |

流水线节奏（可改生成脚本，目前没有对应 `.env`）：

- 每 5 轮对话尝试跑 pipeline
- L1 空闲 600s 后处理
- L2 在 L1 之后延迟 90s，间隔 15–60 分钟
- Persona（L3）每 50 次触发

---

## 4. Proxy 生成配置

`start-proxy.sh` 每次覆盖 `.proxy-config/config.yaml`。

### 4.1 `start-all.sh` 下实际打开的

| 项 | 值 | 作用 |
|---|---|---|
| `auth.enabled` | `true` | 校验客户端 `user_key` |
| `sessionInit.enabled` | `true` | 首轮选 Team / Agent / Task；`headerAutoSelect` 也开（Hermes / OpenClaw 可用 `x-team-id` 等跳过表单） |
| `tdai.enabled` | `true` | 对接 Core 记忆 |
| `tdai.memory.writeL0` | `true` | 每轮对话写入 L0 |
| `tdai.memory.recallL1` | `true` | 配置了 L1 召回（当前注入器不再每轮塞进 user prompt，避免打爆 prompt cache） |
| `tdai.memory.injectL2L3` | `true` | 把 L2 场景 / L3 画像注入 system prompt |
| `injection.enabled` | `true` | 注入管线总开关 |
| `injection.injectors` | `skill`, `knowledge`, `tdai-memory` | 要挂哪些注入器（knowledge 还要看下一节） |
| `injection.externalGatewayUrl` | 来自 `MEMORY_HUB_PROXY_PUBLIC_URL`（未设则省略） | prompt 里 skill/memory/session-bridge 的 curl base。未写时 Proxy 回落到容器网卡 IP |
| `extraction.enabled` | **开**（YAML 未写，走代码默认） | 对话结束后抽 skill、写 L0 |

### 4.2 脚本写死关闭的

| 项 | 默认 | 作用 | 若要打开 |
|---|---|---|---|
| `costGuard.enabled` | **关** | 费用守卫 / 便宜模型兜底路由 | 改 `start-proxy.sh` 生成段，并配 cheap-model 上游 |
| `creditReport.url` | **空（关）** | 腾讯内部用量计费上报。空则不探针、不上报 | 云上再填 MemoryPlus 地址 |
| `redis.enabled` | **关** | 共享缓存与限流。单机不需要 | 多副本时再开，并配 `redis.host` |
| `storage.enabled` | **关**（代码默认） | 用 COS/SQLite 替 Redis 存注入缓存 | 见 `config.example.yaml` 的 `storage:` |
| `opik` / `langfuse` / `clickhouse` | **关** | 可观测与用量上报 | 在完整 YAML 里配 |
| `skillRuntime.allowLlmWrite` | **关** | 是否允许模型经 skill-bridge **写入** skill | 默认只读检索 |

### 4.3 Knowledge 注入默认关

脚本虽然把 `knowledge` 写进了 `injection.injectors`，但 **没有** 写 `knowledge.enabled: true`。Proxy 代码要求三者同时成立才会注册 `<knowledge_tools>`：

1. injectors 含 `knowledge`
2. `knowledge.enabled == true`
3. `knowledge.serviceToken` 非空

因此当前开源一键部署里：**知识库仍可在 Panel 里建，Agent 不会自动在 prompt 里看到 tools 入口。**

若要打开，在 `start-proxy.sh` 生成的 YAML 里补：

```yaml
knowledge:
  enabled: true
  endpoint: "http://memory-core:8420"
  serviceToken: "${MEMORY_CORE_GATEWAY_API_KEY}"
  serviceId: default
```

注意：`MEMORY_CORE_GATEWAY_API_KEY` 为空时 `serviceToken` 也是空，第 3 条仍不满足。这是已知限制，与 Bearer 门必须留空是同一件事。

---

## 5. Memory Hub（Panel + Knowledge）

脚本传给容器的关键环境变量（一般不用在 `.env` 再写一遍）：

| 变量 | 脚本里的值 | 作用 |
|---|---|---|
| `LLM_MODE` | `custom` | Knowledge **直连** `MEMORY_LLM_*`，不绕 Core 的 LLM proxy |
| `KNOWLEDGE_LLM_BINDING_SYNC` | `0` | 不把 Core 的 llm_binding 同步进 KS（custom 模式应关） |
| `REMOTE_INSTANCE_URL` | `http://memory-core:8420` | Panel 后端调 Core（Docker 内网，与公网域名无关） |
| `REMOTE_INSTANCE_PROXY_URL` | 来自 `MEMORY_HUB_PROXY_PUBLIC_URL` | Panel 卡片展示用。同一变量还会写入 Proxy `injection.externalGatewayUrl` |
| `TMC_CALLBACK_URL` | 容器内 `http://127.0.0.1:8125` | KS ingest 完成回调 Panel |

Langfuse（`LANGFUSE_*`）默认全空 = **关**。三个都填了，KS 的 LLM 调用才会上报。

---

## 6. 公司 Nginx / 域名入口

本仓库不再附带 `nginx.conf`。对外入口由公司统一 Nginx 反代到本机端口。

`.env` 必须写成**不带业务端口**的对外 origin，否则卡片、知识 `service_url`、prompt 里的 bridge curl 都会绕过反代、打到 LAN IP：

```bash
KNOWLEDGE_PUBLIC_BASE_URL=https://memhub.example.com/v3
MEMORY_HUB_PROXY_PUBLIC_URL=https://memhub.example.com
```

Core `:8420` 不要对公网开放。公司 Nginx 至少要把 `/claude-code`、`/codebuddy`、`/v1`、`/session-bridge`、`/skill-bridge`、`/memory-bridge` 转到 `:8096`，把 `/v3/wiki`、`/v3/code-graph`、`/v3/tools` 转到 `:8424`，把 `/` 转到 Panel `:8125`。

---

## 7. 和调用方向的关系

改对外 URL 时只影响 **Agent 所在电脑** 和 **浏览器**，不影响容器互访：

```
浏览器  → Panel :8125 → Core :8420（内网）
Agent   → Proxy :8096 → Core :8420（内网）
         → 上游 LLM（PROXY_UPSTREAM_*）
Agent   → Knowledge 的 KNOWLEDGE_PUBLIC_BASE_URL（公网/内网 IP，不经过 Core）
Core    → 只出站 MEMORY_LLM_* ，不访问 Panel / Knowledge / Proxy
```
