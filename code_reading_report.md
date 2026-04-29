下面是我基于仓库源码做的**项目框架介绍 + agent harness 代码解读报告**。重点放在你关心的 **agent harness / swarm / coordinator** 部分。

---

# OpenHarness 项目框架概览

一句话理解：

> **OpenHarness 是一个面向开源 Agent 系统的运行时框架**，核心能力是把“大模型 + 工具 + 权限 + 会话 + 多 agent 协作”组织成一个可运行、可扩展、可检查的 CLI/TUI 系统。  
> 入口是 `oh`，核心包在 `src/openharness/`。

README 里官方自己也把能力总结成这些模块：`agent loop / tools / skills / plugins / memory / permissions / multi-agent coordination / provider workflows / React TUI / ohmo`，见 `README.zh-CN.md:8`。

---

## 一、项目的总体模块划分

从目录结构看，`src/openharness/` 可以分成这几层：

### 1. 入口与 UI 层
- `src/openharness/cli.py`
- `src/openharness/ui/*`
- `frontend/terminal/*`

职责：
- 提供命令行入口、交互模式、React TUI
- 处理 slash commands、权限确认、会话恢复、前端展示

你可以把它看成“壳层”。

---

### 2. 模型调用与 Agent Loop 核心
- `src/openharness/engine/query_engine.py`
- `src/openharness/engine/query.py`
- `src/openharness/engine/messages.py`
- `src/openharness/api/*`

职责：
- 维护对话消息
- 调用模型
- 处理工具调用、工具结果回填
- 控制 agent 一轮任务能跑多少 turn

这是整个 harness 的“心脏”。

---

### 3. 工具系统
- `src/openharness/tools/*`
- `src/openharness/tools/base.py`

职责：
- 定义工具 schema、执行逻辑、权限控制接入点
- 包括 `read_file / edit_file / grep / glob / bash / agent / task_* / team_*` 等

这是 agent 的“手”。

---

### 4. 权限、Hook、配置
- `src/openharness/permissions/*`
- `src/openharness/hooks/*`
- `src/openharness/config/*`

职责：
- 控制工具是否允许执行
- 与 UI 的权限确认联动
- 支持 hook 扩展与运行时配置

这是 harness 的“治理层”。

---

### 5. Skills / Plugins / MCP
- `src/openharness/skills/*`
- `src/openharness/plugins/*`
- `src/openharness/mcp/*`

职责：
- 给 agent 注入任务模板、能力包、外部工具资源
- 让 agent 不只是“裸 LLM”

这是 harness 的“能力扩展层”。

---

### 6. 多 Agent / Swarm / Coordinator
- `src/openharness/coordinator/*`
- `src/openharness/swarm/*`
- `src/openharness/tasks/*`

职责：
- 生成、管理、通信多个 agent
- 支持 subprocess / in-process 等执行后端
- 维护 team、mailbox、权限同步、worktree 隔离

这是你最关注的 **agent harness 核心增强层**。

---

## 二、整体执行链路

先给一个高层流程图式理解：

1. 用户在 `oh` 或 TUI 中输入 prompt  
2. `cli.py` / UI 组装运行环境  
3. 创建 `QueryEngine`
4. `QueryEngine.submit_message()` 把用户输入变成消息，交给 `run_query(...)`
5. 模型输出：
   - 普通文本 -> 直接展示
   - 工具调用 -> 走 ToolRegistry 执行
6. 工具结果再作为消息回到模型
7. 循环直到任务完成或 turn 达到上限
8. 如果用了 `agent` 工具，就会额外 spawn 子 agent，进入 swarm/coordinator 流程

---

# Agent Harness 重点解读报告

下面正式进入你关心的部分。

---

## 三、什么是这个项目里的 Agent Harness

在 OpenHarness 里，**Agent Harness** 不是单指一个 agent，而是：

> 一套围绕 agent 运行的“基础设施”：  
> **prompt 管理 + tool 调度 + permission 管控 + session/context + subagent spawn + team coordination + mailbox + backend 执行层**

也就是说，LLM 只是大脑的一部分，真正让它像“工程 agent”一样工作的是 harness。

这里最关键的几块源码是：

- Agent loop：`src/openharness/engine/query_engine.py:19`
- 子 agent 工具：`src/openharness/tools/agent_tool.py:38`
- agent 定义系统：`src/openharness/coordinator/agent_definitions.py:60`
- swarm 类型抽象：`src/openharness/swarm/types.py:257`
- in-process 执行：`src/openharness/swarm/in_process.py:196`
- subprocess 执行：`src/openharness/swarm/subprocess_backend.py:28`
- team 生命周期：`src/openharness/swarm/team_lifecycle.py:1`
- mailbox 通信：`src/openharness/swarm/mailbox.py:102`
- 权限同步：`src/openharness/swarm/permission_sync.py:101`
- coordinator 模式：`src/openharness/coordinator/coordinator_mode.py:186`

---

## 四、核心一：Agent Loop 是怎么工作的

### 1. `QueryEngine` 是主控制器

`QueryEngine` 在 `src/openharness/engine/query_engine.py:19` 定义。

它的职责很明确：
- 保存消息历史 `self._messages`
- 保存当前模型、system prompt、tool registry、permission checker
- 接收用户消息
- 调用底层 `run_query(...)`
- 在工具调用和模型响应之间循环

关键入口是：

- `submit_message()`：`src/openharness/engine/query_engine.py:147`
- `continue_pending()`：`src/openharness/engine/query_engine.py:192`

### 2. 为什么它是 harness 的核心

因为 agent 是否“像 agent 一样行动”，不在于模型本身，而在于这个循环：

- 用户消息进入
- 模型决定要不要调用工具
- harness 执行工具
- 把结果再反馈给模型
- 直到模型给出完成答复

这其实就是典型的 **tool-augmented agent loop**。

### 3. coordinator 信息如何注入

`_build_coordinator_context_message()` 在 `src/openharness/engine/query_engine.py:117`。

如果当前会话处于 coordinator 模式，它会给消息流额外加一个 synthetic user message，告诉主 agent：

- worker 有哪些工具
- 当前协作上下文是什么

这说明这个项目不是把多 agent 逻辑硬编码到模型里，而是**通过消息注入 + 工具限制**来塑造角色行为。

---

## 五、核心二：Agent Definition 系统

### 1. AgentDefinition 是角色模板

定义在 `src/openharness/coordinator/agent_definitions.py:60`。

它本质上是一个 agent 的配置模型，包含：

- `name`
- `description`
- `system_prompt`
- `tools`
- `disallowed_tools`
- `skills`
- `mcp_servers`
- `model`
- `effort`
- `permission_mode`
- `max_turns`
- `background`
- `initial_prompt`
- `memory`
- `isolation`
- `permissions`
- `subagent_type`

这很重要，因为它说明 OpenHarness 的 agent 不是“临时 prompt 拼一下”，而是有**结构化角色定义**的。

### 2. 它的设计价值

这层设计把“agent 角色”从业务逻辑中抽离出来：

- 角色能力可配置
- prompt 可配置
- 工具权限可配置
- 模型和 effort 可配置
- 是否 worktree 隔离也可配置

所以这个框架的 agent harness 是 **role-driven** 的，不是只有一个万能 agent。

### 3. 内建 agent prompt

文件里还内置了几个角色 prompt，比如：

- general-purpose：`src/openharness/coordinator/agent_definitions.py:160`
- Explore：`src/openharness/coordinator/agent_definitions.py:166`
- Plan：`src/openharness/coordinator/agent_definitions.py:200`
- Verification：`src/openharness/coordinator/agent_definitions.py:251`

你学习时可以重点看这些 prompt，因为它们反映了这个项目对不同 worker 的分工哲学：

- `Explore`：只读探索
- `Plan`：只读设计
- `Verification`：偏破坏性验证
- `general-purpose`：通用执行

---

## 六、核心三：`agent` 工具如何 spawn 子 Agent

### 1. 入口：`AgentTool`

定义在 `src/openharness/tools/agent_tool.py:38`。

输入参数包括：

- `description`
- `prompt`
- `subagent_type`
- `model`
- `command`
- `team`
- `mode`

执行入口在 `execute()`：`src/openharness/tools/agent_tool.py:45`

### 2. 执行流程

`AgentTool.execute()` 做了几件事：

#### 第一步：校验 mode
支持：
- `local_agent`
- `remote_agent`
- `in_process_teammate`

见 `src/openharness/tools/agent_tool.py:46`

#### 第二步：按 `subagent_type` 查角色定义
通过 `get_agent_definition(...)` 加载角色模板，见 `src/openharness/tools/agent_tool.py:52`

#### 第三步：构造 `TeammateSpawnConfig`
见 `src/openharness/tools/agent_tool.py:68`

它会把：
- 名字
- team
- prompt
- cwd
- parent_session_id
- model
- command
- system_prompt
- permissions
- task_type

打包成统一配置对象。

#### 第四步：调用 backend spawn
见 `src/openharness/tools/agent_tool.py:65` 和 `:82`

注意一个很关键的设计：

> 虽然支持 `in_process_teammate` mode，但 `AgentTool` 默认拿的是 `subprocess` backend

源码注释直接说明原因，见 `src/openharness/tools/agent_tool.py:61`：

- subprocess 能接入 `BackgroundTaskManager`
- task tools 可以查询它
- in-process 返回的是 asyncio 内部 ID，不方便统一管理

这说明 OpenHarness 在产品层面更偏向**可观测、可管理的子任务系统**，而不是只追求轻量。

---

## 七、核心四：Swarm 类型抽象

### 1. `TeammateSpawnConfig`

定义在 `src/openharness/swarm/types.py:257`

这是 swarm 子系统最核心的数据结构之一。字段包括：

- `name`, `team`, `prompt`, `cwd`, `parent_session_id`
- `model`
- `command`
- `system_prompt`
- `system_prompt_mode`
- `color`
- `permissions`
- `plan_mode_required`
- `allow_permission_prompts`
- `worktree_path`
- `session_id`
- `subscriptions`
- `task_type`

这相当于“spawn 一个 agent 所需的完整执行描述”。

### 2. `SpawnResult`

定义在 `src/openharness/swarm/types.py:321`

描述 spawn 的结果：
- `task_id`
- `agent_id`
- `backend_type`
- `success`
- `error`
- `pane_id`

### 3. `TeammateExecutor` 协议

定义在 `src/openharness/swarm/types.py:357`

要求后端统一实现：
- `spawn(...)`
- `send_message(...)`
- `shutdown(...)`

这是一层很标准的接口抽象。好处是：

- 上层 `agent tool` 不关心底层是 subprocess 还是 in-process
- 后续可以接 tmux / iterm2 / remote backend

这就是 harness 的扩展性基础。

---

## 八、核心五：两种子 Agent 执行后端

---

### A. subprocess backend

定义在 `src/openharness/swarm/subprocess_backend.py:28`

#### 它做什么
每个子 agent 作为一个独立进程运行，通过任务管理器管理。

#### 关键流程
在 `spawn()`：`src/openharness/swarm/subprocess_backend.py:47`

它会：

1. 生成 `agent_id`
2. 构造继承参数：
   - `build_inherited_cli_flags(...)` `src/openharness/swarm/subprocess_backend.py:55`
   - `build_inherited_env_vars()` `src/openharness/swarm/subprocess_backend.py:60`
3. 组装命令
4. 用 `BackgroundTaskManager` 创建 agent task，见 `src/openharness/swarm/subprocess_backend.py:79`
5. 保存 `agent_id -> task_id` 映射

#### 为什么重要
这是最“工程化”的方式：
- 进程隔离好
- 可独立停止
- 可读取输出
- 可接任务工具链

所以如果你从“产品落地”角度看，这个实现更实用。

---

### B. in-process backend

定义在 `src/openharness/swarm/in_process.py:1`

这个文件其实很值得仔细看，因为它展示了作者对“轻量多 agent 并发”的设计思路。

#### 核心思想
多个 teammate 不是子进程，而是当前 Python 进程里的多个 `asyncio.Task`。

#### 关键机制 1：`TeammateAbortController`
定义在 `src/openharness/swarm/in_process.py:52`

它同时支持：
- graceful cancel
- force cancel

这是为了让 worker 能优雅结束，而不是只能粗暴杀掉。

#### 关键机制 2：`TeammateContext`
定义在 `src/openharness/swarm/in_process.py:113`

保存每个 agent 的运行时上下文：
- `agent_id`
- `agent_name`
- `team_name`
- `parent_session_id`
- `color`
- `plan_mode_required`
- `abort_controller`
- `message_queue`
- `status`
- `started_at`
- `tool_use_count`
- `total_tokens`

#### 关键机制 3：ContextVar 隔离
定义在 `src/openharness/swarm/in_process.py:173`

通过 `ContextVar` 保存当前 teammate context，访问器是：
- `get_teammate_context()` `src/openharness/swarm/in_process.py:178`
- `set_teammate_context()` `src/openharness/swarm/in_process.py:186`

这个设计很漂亮，因为它避免了在所有函数调用链里手动传递 agent 上下文。

#### 关键机制 4：执行循环
主入口是 `start_in_process_teammate(...)`：`src/openharness/swarm/in_process.py:196`

它会：
1. 创建 `TeammateContext`
2. 绑定到当前 async context
3. 初始化 mailbox
4. 执行 query loop
5. 轮询 mailbox
6. 结束时给 leader 发 idle notification

这说明 in-process backend 并不是“简陋 mock”，而是完整的 teammate runtime。

---

## 九、核心六：mailbox —— 多 Agent 之间如何通信

定义在 `src/openharness/swarm/mailbox.py:102`

这是 swarm 部分最有“harness 基础设施”味道的模块之一。

### 1. 通信模型
每条消息都是一个 JSON 文件，存放在：

`~/.openharness/teams/<team>/agents/<agent_id>/inbox/`

见说明 `src/openharness/swarm/mailbox.py:1`

### 2. 消息类型
`MessageType` 定义在 `src/openharness/swarm/mailbox.py:27`，包括：

- `user_message`
- `permission_request`
- `permission_response`
- `sandbox_permission_request`
- `sandbox_permission_response`
- `shutdown`
- `idle_notification`

### 3. 为什么用文件邮箱
它的优点是：
- 简单
- 跨进程天然兼容
- 不依赖额外消息队列服务
- 易调试，可直接看磁盘文件

对于开源 CLI 项目，这是很实用的取舍。

### 4. `TeammateMailbox`
类定义在 `src/openharness/swarm/mailbox.py:102`

提供：
- `write()` `src/openharness/swarm/mailbox.py:126`
- `read_all()` `src/openharness/swarm/mailbox.py:153`
- `mark_read()` `src/openharness/swarm/mailbox.py:183`
- `clear()` `src/openharness/swarm/mailbox.py:211`

并且通过 `.tmp -> os.replace` 实现原子写入，见 `src/openharness/swarm/mailbox.py:144`

这说明作者很清楚并发下文件通信的基本风险。

---

## 十、核心七：权限同步机制

定义在 `src/openharness/swarm/permission_sync.py:1`

这是 OpenHarness 区别于简单“多进程 agent demo”的关键之一。

### 1. 为什么需要它
多 agent 环境里，worker 可能要执行敏感工具：
- bash
- edit_file
- write_file

如果每个 worker 各自弹权限确认，会很混乱。  
所以这里设计了**leader-worker 权限同步协议**。

### 2. 主要数据结构

#### `SwarmPermissionRequest`
定义在 `src/openharness/swarm/permission_sync.py:101`

包含：
- 谁发起的
- 哪个 team
- 哪个工具
- tool_use_id
- 描述
- input
- suggestion
- 状态
- 反馈
- permission_updates

#### `PermissionResolution`
定义在 `src/openharness/swarm/permission_sync.py:209`

表示 leader 或 worker 的决策结果。

### 3. 两种通信模式
文件头注释说得很清楚，见 `src/openharness/swarm/permission_sync.py:3`

支持：
- **文件式 pending/resolved**
- **mailbox 式请求/响应**

### 4. 只读工具自动放行
`_READ_ONLY_TOOLS` 在 `src/openharness/swarm/permission_sync.py:76`

例如：
- `read_file`
- `glob`
- `grep`
- `web_fetch`
- `web_search`
- `task_get`
- `task_list`

`_is_read_only()` 在 `src/openharness/swarm/permission_sync.py:91`

这意味着 swarm 权限体系不是“所有都问”，而是有分级策略。

### 5. 测试能帮助理解
`tests/test_swarm/test_permission_sync.py:122` 验证了：
- 只读工具会自动批准
- 写操作会委托给 `PermissionChecker`

这是 harness 层治理逻辑的体现。

---

## 十一、核心八：Team 生命周期管理

定义在 `src/openharness/swarm/team_lifecycle.py:1`

### 1. 持久化位置
团队元数据存放在：

`~/.openharness/teams/<name>/team.json`

见 `src/openharness/swarm/team_lifecycle.py:3`

### 2. 主要对象

#### `TeamMember`
定义在 `src/openharness/swarm/team_lifecycle.py:92`

表示一个成员，包括：
- `agent_id`
- `name`
- `backend_type`
- `joined_at`
- `model`
- `color`
- `plan_mode_required`
- `cwd`
- `worktree_path`
- `permissions`
- `status`

#### `TeamFile`
定义在 `src/openharness/swarm/team_lifecycle.py:189`

表示一个 team 的持久化快照，包括：
- name / description
- lead_agent_id
- hidden_pane_ids
- members
- allowed_paths
- metadata

### 3. 为什么这层重要
因为 swarm 不只是“spawn 两个任务”，它还需要：
- 可恢复
- 可查询
- 可跟踪 team 成员
- 可记录共享权限范围

所以 team lifecycle 模块把多 agent 协作提升成“可管理实体”。

---

## 十二、核心九：Coordinator 模式

定义在 `src/openharness/coordinator/coordinator_mode.py:186`

这是 Agent Harness 里“主 agent 统筹 worker”设计的关键。

### 1. 什么是 coordinator mode
当环境变量 `CLAUDE_CODE_COORDINATOR_MODE` 打开时，当前 agent 会以“调度者”的视角运行，见：

- `is_coordinator_mode()` `src/openharness/coordinator/coordinator_mode.py:186`

### 2. coordinator 的特殊工具
`get_coordinator_tools()` 在 `src/openharness/coordinator/coordinator_mode.py:216`

它保留了三类调度工具：
- `agent`
- `send_message`
- `task_stop`

这很有代表性：

> coordinator 不是亲自干活，而是负责**派工、沟通、停止**

### 3. worker 可用工具上下文
`get_coordinator_user_context()` 在 `src/openharness/coordinator/coordinator_mode.py:221`

它会把 worker 的工具能力整理成一段上下文注入给 coordinator。

这是一种很实用的 prompt engineering 方式：
- coordinator 知道 worker 能做什么
- coordinator 不需要猜测子 agent 的能力边界

### 4. task 通知协议
`TaskNotification` 定义在 `src/openharness/coordinator/coordinator_mode.py:79`

以及：
- `format_task_notification()` `src/openharness/coordinator/coordinator_mode.py:109`
- `parse_task_notification()` `src/openharness/coordinator/coordinator_mode.py:129`

它把 worker 完成结果包装成 XML 风格通知，再交给 coordinator 汇总。

这个设计说明：
- worker 结果不是简单 stdout
- 而是结构化可汇总事件

---

## 十三、spawn_utils：父子 agent 的上下文继承

定义在 `src/openharness/swarm/spawn_utils.py:1`

这是一个很容易被忽略但很关键的模块。

### 1. `get_teammate_command()`
`src/openharness/swarm/spawn_utils.py:70`

决定 spawn 子 agent 用什么命令：
1. 环境变量覆盖
2. 当前 Python 解释器
3. `openharness` 可执行文件
4. fallback 到 `python`

### 2. `build_inherited_cli_flags()`
`src/openharness/swarm/spawn_utils.py:96`

把这些信息传给子 agent：
- model
- system prompt
- permission mode
- settings path
- teammate mode
- plugin dirs

### 3. `build_inherited_env_vars()`
`src/openharness/swarm/spawn_utils.py:185`

转发父进程环境，比如：
- provider 相关
- proxy 相关
- cert 相关
- `OPENHARNESS_MODEL`
- `OPENHARNESS_API_FORMAT`

这个模块的核心价值是：

> 确保子 agent 不是“孤儿进程”，而是继承父 agent 的运行语境。

这就是 harness 的连续性。

---

# 十四、从设计角度总结：这个 agent harness 的特点

如果你从架构学习的角度看，这套实现有几个鲜明特点。

---

## 1. 不是“模型中心”，而是“运行时中心”

很多 agent 项目是：
- prompt 很重
- runtime 很薄

OpenHarness 反过来更像：
- prompt 只是配置的一部分
- runtime、tooling、permission、task、swarm 才是重点

所以它更接近一个 **agent operating runtime**。

---

## 2. 多 agent 是一等公民，不是附加功能

从这些模块就能看出来：

- `agent_tool.py`
- `swarm/types.py`
- `swarm/in_process.py`
- `swarm/subprocess_backend.py`
- `swarm/mailbox.py`
- `swarm/permission_sync.py`
- `swarm/team_lifecycle.py`
- `coordinator/coordinator_mode.py`

说明作者不是“顺手加了个 subagent”，而是把 agent teamwork 当成完整子系统设计。

---

## 3. 工程取向很强

几个明显信号：

- subprocess backend 默认优先，方便 task 管理
- mailbox 用文件实现，简单可靠
- team 元数据落盘
- 权限请求可转发给 leader
- CLI flags / env 可继承
- ContextVar 做 in-process 隔离

这套设计明显是为了**真实开发环境可用**，而不只是学术 demo。

---

## 4. 角色定义清晰，利于 specialization

通过 `AgentDefinition`，子 agent 可以有不同：
- prompt
- tools
- skills
- permission mode
- model
- isolation

这对实现“研究员 / 规划师 / 验证员 / 执行员”很重要。

---

# 十五、建议你按什么顺序读源码

如果你主要学 agent harness，我建议按这个顺序读：

### 第一层：先看总入口
1. `README.zh-CN.md:8`
2. `src/openharness/cli.py:1`

目的是先知道项目对外暴露了什么能力。

---

### 第二层：看 agent loop
3. `src/openharness/engine/query_engine.py:19`

重点看：
- `submit_message()` `src/openharness/engine/query_engine.py:147`
- `continue_pending()` `src/openharness/engine/query_engine.py:192`

你要先理解“单 agent 是怎么工作的”。

---

### 第三层：看 agent 角色定义
4. `src/openharness/coordinator/agent_definitions.py:60`

重点看：
- `AgentDefinition`
- 内建 prompt，如 `Explore`、`Plan`、`Verification`

---

### 第四层：看 subagent 入口
5. `src/openharness/tools/agent_tool.py:38`

重点看：
- `execute()` `src/openharness/tools/agent_tool.py:45`

理解“主 agent 怎么生成 worker”。

---

### 第五层：看 swarm 抽象
6. `src/openharness/swarm/types.py:257`
7. `src/openharness/swarm/registry.py:97`
8. `src/openharness/swarm/spawn_utils.py:96`

理解“spawn config / backend / 环境继承”三件套。

---

### 第六层：看两个 backend
9. `src/openharness/swarm/subprocess_backend.py:28`
10. `src/openharness/swarm/in_process.py:196`

对比：
- subprocess 偏生产可管控
- in-process 偏轻量并发

---

### 第七层：看协作协议
11. `src/openharness/swarm/mailbox.py:102`
12. `src/openharness/swarm/permission_sync.py:101`
13. `src/openharness/swarm/team_lifecycle.py:189`

这是多 agent harness 的基础设施。

---

### 第八层：看 coordinator
14. `src/openharness/coordinator/coordinator_mode.py:221`

理解 leader 如何：
- 认知 worker 能力
- 收集结果
- 作为 orchestrator 工作

---

### 第九层：配合测试阅读
15. `tests/test_swarm/test_in_process.py:78`
16. `tests/test_swarm/test_permission_sync.py:122`
17. `tests/test_swarm/test_team_lifecycle.py:82`
18. `tests/test_real_large_tasks.py:169`

测试是理解这类基础设施最有效的材料之一。

---

# 十六、学习时你最该抓住的几个“关键问题”

读这套代码时，建议你每看一个模块都问：

### 1. 这个模块解决了什么“agent 运行时问题”？
比如：
- spawn 子 agent
- 子 agent 如何通信
- 如何停掉子 agent
- 权限由谁决定

### 2. 状态放在哪里？
这个项目里状态分散在：
- QueryEngine 内存消息
- task manager
- mailbox 文件
- team.json
- env vars
- ContextVar

### 3. 哪些是“角色层”，哪些是“运行时层”？
- role：`AgentDefinition`
- runtime：`swarm/*`, `query_engine.py`, `tools/*`

### 4. 单 agent 和多 agent 的边界在哪里？
- 单 agent：`engine/*`
- 多 agent：`tools/agent_tool.py + swarm/* + coordinator/*`

---

# 十七、给你的简短结论

如果只用一句话概括这部分源码：

> OpenHarness 的 agent harness，本质上是一套**以 QueryEngine 为单 agent 内核、以 AgentTool + Swarm 为多 agent扩展、以 mailbox/team/permission_sync 为协作基础设施**的工程化运行时系统。

它的学习价值主要在于：

- 不是只讲 prompt，而是讲 **runtime**
- 不只是 tool use，而是讲 **agent orchestration**
- 不只是子进程，而是讲 **权限、状态、通信、持久化**

---

如果你愿意，我下一步可以继续给你做两份更细的材料之一：

1. **“agent harness 核心源码导读版”**  
   我按文件逐个讲，每个文件讲“作用、关键类、关键函数、调用关系”。

2. **“OpenHarness 多 Agent 时序图报告”**  
   我把“主 agent -> agent tool -> spawn backend -> mailbox -> worker -> task notification -> coordinator 汇总”画成文字时序图，学习起来会更直观。
