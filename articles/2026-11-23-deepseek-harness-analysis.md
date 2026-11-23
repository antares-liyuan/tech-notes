# DeepSeek Harness（DSH）分析：原理、第三方模型支持，以及与 Codex / Claude Code / Hermes Agent 的对比

> 分析对象：[deepseek.com/harness](https://www.deepseek.com/harness/) · [github.com/deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness)
> 分析日期：2026-11-23
> 分析方式：阅读官网、仓库 README、架构文档、子系统文档和各包 README。**没有安装运行，也没有调用任何接口。**
> 说明：DSH 部分以仓库文档为准；Codex、Claude Code、Hermes Agent 部分基于公开资料和我们的已有了解，**本次没有逐项核实最新版本**。

## 要点

- **DSH 是 DeepSeek 开源的通用智能体框架（agent harness）**，不只是写代码的工具，也能处理文档、表格、资料查证、定时任务。形态包括桌面端、Web UI、无界面运行、SDK 和 ACP 服务。
- **核心机制是「一切皆插件」**：基于 [Cordis](https://github.com/cordiverse/cordis) 框架，连智能体循环、模型适配器、工具注册表、会话日志本身都是插件，都可以通过配置替换。没有需要打补丁的特权内核。
- **可以使用第三方模型，主体功能不受影响**。内置 Anthropic、OpenAI、Kimi、GLM 等提供商，也能接任意中转站或自建服务，支持 4 种 API 协议。差别主要在 DeepSeek 专属的几项：账号登录、模型列表自动跟随、官方请求附加字段（含会话日志上传）。
- **隐私提示**：使用 DeepSeek 官方 API 时，DSH **默认会把会话日志增量上传给 DeepSeek**，可以在设置里关掉。用第三方模型时不上传。

---

## 一、项目概况

| 项目 | 内容 |
| --- | --- |
| 出品方 | 杭州深度求索人工智能基础技术研究有限公司（DeepSeek） |
| 仓库简介 | DeepSeek Harness: Everything is a Plugin. |
| 简称 | `dsh` |
| 语言 | TypeScript（另有 Python SDK） |
| 许可证 | MIT |
| 仓库创建 | 2026-08-13 |
| 最近更新 | 2026-11-22 |
| Star / Fork | 约 28.6 万 / 3.6 万（2026-11-23 GitHub API 数据） |
| 仓库规模 | 约 2.3 万个文件，monorepo，60 个左右的包 |
| 阶段 | 开发者预览版，官方明确说**未来会有破坏兼容性的变更** |
| 理论基础 | 论文 *A Programming Paradigm for Spatiotemporal Composability*（arXiv 2608.25512） |

**运行方式：**

```sh
# 方式一：npm，一行启动 Web UI（默认 http://127.0.0.1:3080）
npx @deepseek-ai/dsh web

# 方式二：源码
git clone https://github.com/deepseek-ai/deepseek-harness.git
cd deepseek-harness
pnpm install && pnpm run build && pnpm dsh web
```

另有 macOS（Apple 芯片）和 Windows 64 位桌面版，从官网下载。

**官网展示的能力**：日常办公（整理资料、分析表格、写文档、做汇报）、写代码改仓库、查资料并标注来源、后台长任务、定时任务（例如每周五 17:00 生成周报）、通过对话「创造模式」生成插件、开发者执行轨迹查看工具。实验性功能有智能体团队、自动授权审查、自动化任务、语音输入。

---

## 二、原理和机制

### 1. 底座：Cordis 与「一切皆插件」

Cordis 是 DSH 底层的插件框架。每个插件向共享上下文（`ctx`）贡献三类东西：

- **服务**：例如 `ctx.llm`（模型调用）、`ctx.tools`（工具注册表）、`ctx.sessions`（会话日志）
- **类型化事件**：其他插件可以监听、拦截
- **可逆的副作用**：插件注册的东西，在插件卸载时自动撤销

产品的每一部分都是插件，包括模型适配器、工具注册表、会话日志，**以及智能体循环本身**。所以要改 DSH 的行为，不需要改内核，而是在旁边挂一个插件，或者替换掉某个插件。这也是它支持热重载（HMR）和「在对话中让 AI 给自己写插件并当场装上」的基础。

### 2. Profile 与组合包：一个运行中的 DSH 是一棵插件树

- **组合包（bundle）**：一组插件配置的分发格式。`dsh-base` 是共享底层，包含模型适配器、工具、持久化、沙箱和审批策略、设置、凭据、遥测；再往上叠 `dsh-web-app`（浏览器应用）、`dsh-headless`（一次性运行器）、`dsh-sdk-app`（SDK JSON-RPC 服务）、`dsh-acp-app`（ACP 自动化服务）。
- **Profile**：具名的组装方案，随发行版提供 `web`、`headless`、`sdk`、`sdk-minimal`、`acp` 几个模板。
- **分层覆盖**：启动时依次应用「组合包 → profile 的 `cordis.patch.yml` → home patch → 命令行 `--patch`」，后面的层可以按 id 替换前面任何一个插件的配置，或插入新插件。
- 运行 `dsh --profile web --dump-config` 可以打印当前启动的整棵配置树，其中任何一项都能被你自己的 patch 替换。

### 3. 智能体循环（Agent Loop）

- **步骤（step）**：一次模型请求，加上它调用的工具。
- **轮次（turn）**：零个或多个步骤。领取用户输入时开始，没有未完成的工作时结束。

一个轮次的流程大致是：

1. 领取输入，快照本次会话选择的模型，组装系统提示词片段和本步可用的工具 schema
2. 触发 `agent/pre-step`，插件可以接纳、改写或拒绝这条输入
3. 发起模型请求，流式接收回复
4. 执行工具调用：`tools/pre-execute → tools/execute → tools/post-execute`
5. 如果工具结果还需要再请求模型，或者有新输入到达，进入下一步；否则结束轮次

其中 `agent/pre-step`、`llm/stream` 和三个 `tools/*` 事件是**瀑布式事件**：每个监听器必须调用 `next()` 才会交给下一个。这就是插件拦截和改写模型请求、工具调用的切入点，例如审批、沙箱、自动审查都挂在这里。

### 4. 会话日志是唯一事实来源

- 会话日志是**只追加**的 `SessionEvent` 日志，模型看到的上下文由 `deriveMessages()` 从日志里投影出来。
- 分叉（fork）、恢复、文本记录、遥测、持久化，都从日志派生。
- 每条助手消息都嵌入产生它的完整流式记录；失败、重试、取消也作为日志事件保留，但不进入模型历史。
- 日志用 zstd 压缩的 JSONL 存储；格式有版本号，升级时只新增、不覆盖旧数据。

### 5. 主要功能模块（都是插件包）

| 类别 | 包 |
| --- | --- |
| 执行 | shell、subprocess、terminal（持久终端）、ssh（远程执行）、ptc-runtime |
| 安全 | sandbox、guard（循环和工具防护）、permission-presets、approval |
| 文件和代码 | fs、lsp（语言服务器）、deliverables（每轮产出物） |
| 能力扩展 | skill（技能加载）、mcp、web（搜索和抓取）、browser-use、computer-use |
| 上下文 | context、compaction（上下文压缩）、spill（超长输出落盘） |
| 协作和任务 | subagent（子智能体）、team（智能体团队）、jobs（后台任务）、workflow、todo、plan、goal、schedule（定时任务） |
| 自我修改 | extensions（运行时自修改）、plugin-manager |
| 接口 | api、sdk、acp、web（Web UI）、desktop（Electron） |

### 6. 沙箱和权限

- **沙箱模式**：`read-only`、`workspace-write`、`danger-full-access`，只管文件写入，**不管网络和进程可见性**。
- **后端**：Linux 用 bwrap/Landlock，macOS 用 Seatbelt，Windows 用 ACL 受限令牌；SSH 远程也能执行同样的策略。
- **权限预设**：把沙箱模式和审批策略捆绑成选项。默认有 `workspace-write`（工作区可写 + 询问）和 `danger-full-access`（完全访问 + 不询问）；实验性的 Auto review 可以自动审查授权。
- 官方安全说明写得很直白：项目**尚未经过安全审计**，沙箱和审批**不保证隔离**；建议在一次性虚拟机或容器里运行，只给最小权限，不要暴露敏感凭据。

### 7. 兼容 Claude Code 生态

仓库里有专门的桥接层：

- `dsh-experimental-claude-code-mods`：让 Claude Code 的模组（mods）在 DSH 里运行。兼容基准是 Claude Code 2.1.287，文档逐项列出了行为差异。
- `dsh-hooks-claude-code`：兼容 Claude Code 的 hooks 配置。
- 注意：桥接后的模组在进程内运行，拥有进程的全部权限，**不加沙箱**。

### 8. 产品遥测和请求标识

- 所有发往模型提供商的请求都会带上 DSH 的 `User-Agent` 标识，第三方模型也一样。
- 产品埋点通过 OpenTelemetry 发送，文档说明只在桌面端采集交互事件，普通 Web 客户端不采集产品事件，也不自动采集会话数据。

---

## 三、使用第三方模型有没有影响

### 结论：能用，主体功能不受影响；少了几项 DeepSeek 专属功能

DSH 的模型层是**和提供商无关的**：核心 `llm` 包定义统一的消息、内容块和流式分片格式，各适配器把不同厂商的协议翻译成这套格式。智能体循环、工具、沙箱、子智能体、技能、MCP 这些都建立在统一格式之上，**不依赖具体模型**。

### 模型适配器

| 适配器 | 作用 |
| --- | --- |
| `llm-deepseek` | DeepSeek Messages 协议、推理强度、图片输入、Files API |
| `llm-deepseek-api-key` | 用 API Key 连接 DeepSeek 官方，自动跟随官方模型列表 |
| `llm-deepseek-account` | 用 DeepSeek 账号登录调用（不回退到 API Key） |
| `llm-pi-ai` | 多提供商适配器，基于 pi-ai 的提供商目录，也支持手工声明的网关 |

两类适配器可以同时挂载，DeepSeek 和第三方模型能在同一个 DSH 里切换使用。

### 接入第三方的两种方式

1. **内置第三方提供商**：在「设置 → 模型 → 添加模型提供商」里选，例如 `anthropic`、`openai`、`moonshotai`（Kimi）、`zai`（GLM）等，填 API Key 即可，端点和模型列表由内置目录提供。另有 `openai-codex` 这类走 OAuth 登录的提供商。
2. **自定义模型 API**：适用于中转站、公司网关、自建服务。填 Provider ID、基础 URL、协议、密钥和模型，协议支持 4 种：
   - OpenAI Chat Completions
   - OpenAI Responses
   - Anthropic Messages
   - Gemini API

   支持「获取可用模型」自动探测，探测失败可以手动填模型 ID，效果一样。

密钥只写、不回显，存在 `$DSH_HOME/.credentials.yaml`，配置文件里只保留凭据引用。模型变更从下一次请求生效，不用重启。

### 换成第三方后的具体差异

| 项目 | DeepSeek 官方 | 第三方模型 |
| --- | --- | --- |
| 智能体循环、工具、沙箱、子智能体、技能、MCP、定时任务等 | ✅ | ✅ 不受影响 |
| DeepSeek 账号登录 | ✅ | ❌ 只能用对方的 API Key 或 OAuth |
| 模型列表自动跟随官方发布（约 10 分钟刷新） | ✅ | 内置目录或手动维护 |
| 官方请求扩展字段（`dsh_session_log`、`dsh_plugin_packages`） | 默认发送 | ❌ 不发送 |
| 会话日志上传 | **默认开启**，每次请求最多 8 MiB 增量 | 不上传 |
| 推理强度、图片和文件限制 | 由 DeepSeek 适配器控制 | 由各适配器自己声明，能力可能不同 |
| 请求带 DSH 的 `User-Agent` | ✅ | ✅ |

### 需要注意的三点

1. **会话日志上传（隐私）**：用 DeepSeek 官方 API 时，`dsh-session-log-deepseek` 插件默认开启，会把会话日志增量附在请求里发给 DeepSeek。关闭方法：「设置 → 通用 → 在使用官方模型 API 时上传 Session Log」，或在 profile 配置里设 `enabled: false`。对数据敏感的场景，这一条值得重点关注。用第三方模型时不涉及。
2. **效果可能有差异**：文档没有明说系统提示词和内置技能是不是针对 DeepSeek 模型调优的，**信息不足**。实际体验和工具调用的稳定性，取决于所选模型自己的智能体能力，建议实测。
3. **中转站要选对协议**：一个提供商只能用一种协议；网关同时支持两种协议时，需要建两个提供商。

---

## 四、与 Codex、Claude Code、Hermes Agent 对比

> ⚠️ 本节中 Codex、Claude Code、Hermes Agent 的信息基于公开资料和已有了解，**本次没有逐项核实最新版本**，以各自官方文档为准。

| 维度 | DeepSeek Harness | OpenAI Codex CLI | Claude Code | Hermes Agent |
| --- | --- | --- | --- | --- |
| 出品方 | DeepSeek | OpenAI | Anthropic | Nous Research |
| 开源 | ✅ MIT | ✅ Apache-2.0 | ❌ 闭源 | ✅ MIT |
| 主要语言 | TypeScript | Rust | TypeScript（闭源发布） | Python |
| 定位 | 通用智能体框架：办公 + 编程 + 自建 Harness | 编程智能体 | 编程智能体，延伸到通用任务 | 通用个人智能体，强调长期记忆和自我改进 |
| 主要形态 | 桌面端、Web UI、Headless、SDK、ACP | 终端 CLI、IDE 插件、云端任务 | 终端 CLI、IDE 插件、桌面端和网页端 | 终端 CLI，加多平台消息网关（Telegram 等） |
| 架构特点 | 一切皆插件，连智能体循环都可替换 | 固定内核，通过配置、MCP 扩展 | 固定内核，通过 hooks、插件、模组、子智能体扩展 | 固定内核，通过技能、工具集、记忆扩展 |
| 第三方模型 | ✅ 一等支持：内置多家 + 4 种协议自定义网关 | 可配置其他兼容 OpenAI 的提供商，但主要针对 OpenAI 模型优化 | 官方只支持 Claude（含 Bedrock、Vertex）；可借助兼容 Anthropic 协议的端点接其他模型，非官方支持 | ✅ 模型无关，支持 OpenRouter、各家 API、本地模型 |
| 沙箱 | Linux bwrap/Landlock、macOS Seatbelt、Windows ACL；read-only / workspace-write / danger-full-access | macOS Seatbelt、Linux Landlock 等，模式命名和 DSH 相同 | 权限审批规则为主，另有沙箱选项 | 支持本地、Docker、SSH 等多种执行后端 |
| 子智能体 / 团队 | ✅ 子智能体 + 实验性智能体团队 | 有 | ✅ 子智能体 | ✅ 子智能体 |
| 技能 / MCP | ✅ / ✅ | ✅ / ✅ | ✅ / ✅ | ✅ / ✅ |
| 定时任务 | ✅ 内置 | 信息不足 | 部分形态支持 | ✅ 内置 |
| 长期记忆 | 通过插件和 MCP 记忆指南 | AGENTS.md 项目说明为主 | CLAUDE.md 和记忆功能 | ✅ 核心卖点：持久记忆 + 自动沉淀技能 |
| 生态兼容 | 兼容 Claude Code 模组和 hooks；ACP | AGENTS.md 标准 | 插件市场、模组 | 兼容 agentskills 等技能格式 |
| 成熟度 | 开发者预览，未做安全审计，会有破坏性变更 | 较成熟 | 最成熟，用户最多 | 迭代快 |

### 怎么理解它们的区别

- **Claude Code 和 Codex 是「产品」**：内核固定，开箱即用，在自家模型上调得最好。适合只想把活干完、不想折腾架构的用户。
- **DSH 是「框架 + 产品」**：同样开箱即用，但每一层都能拆开替换，目标之一是让你**构建自己的 Harness**。代价是概念多（Cordis、profile、bundle、patch），而且还在预览期，接口会变。
- **Hermes Agent 是「个人助理型」**：重点不在写代码，而在长期运行、跨平台聊天、记住你、把做过的事沉淀成技能。
- **关于 DSH 的「兼容」策略**：它的沙箱模式命名和 Codex 一致，又专门做了 Claude Code 模组和 hooks 的桥接。可以看出它有意降低用户从这两个工具迁移过来的成本。

### 选型建议

| 场景 | 建议 |
| --- | --- |
| 主力写代码、追求稳定 | Claude Code 或 Codex |
| 想用国产模型或多模型自由切换，又要完整的智能体功能 | DSH |
| 要在智能体框架上做二次开发、定制自己的 Agent 产品 | DSH（插件化最彻底，MIT 许可） |
| 想要一个长期在线、接入聊天软件、会记事的个人助手 | Hermes Agent |
| 对数据和安全要求高 | 任何一个都放在虚拟机或容器里跑；用 DSH 配 DeepSeek 官方 API 时，先关闭会话日志上传 |

---

## 五、尚未确认的问题

- 系统提示词和内置技能是否针对 DeepSeek 模型做了专门调优：信息不足。
- 不同第三方模型在 DSH 里的实际工具调用效果：没有实测。
- 桌面端和 Web 端在功能上的完整差异：信息不足。
- 仓库 Star 数异常高（约 28.6 万），只照 GitHub API 数据记录，不代表活跃度或质量。

---

## 附：分析方法和资料来源

1. 抓取官网 `deepseek.com/harness` 页面文本。
2. 通过 GitHub API 获取仓库元数据和完整文件树（约 2.3 万个文件）。
3. 阅读的仓库文档：
   - `README.zh.md`、`AGENTS.md`、`SAFETY.zh.md`
   - `docs/architecture.zh.md`（架构）
   - `docs/user/guide/providers.zh.md`（配置模型）
   - `docs/subsystems/claude-code-mods.zh.md`、`sandbox.zh.md`、`permission-presets.zh.md`、`product-telemetry.zh.md`
   - `packages/llm/` 下各包 README，`packages/session/session-log-deepseek/README.zh.md`
   - `.agents/notes/.../mandatory-app-attribution-headers.zh.md`
4. 没有安装或运行 DSH。以上内容基于 2026-11-23 的 `master` 分支，项目处于预览期，后续可能变化。

---

<sub>免责声明：本文基于 DeepSeek Harness 官网和公开仓库文档整理，不代表 DeepSeek 官方说明，可能存在理解偏差；项目处于预览期，以官方最新文档为准。Codex、Claude Code、Hermes Agent 部分未逐项核实最新版本。文中提及的产品名称及商标归各自所有者所有。</sub>
