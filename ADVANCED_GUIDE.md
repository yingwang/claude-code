# Claude Code Advanced Usage Guide / Claude Code 高级用法指南

> Practical tips extracted from deep analysis of Claude Code's source code.
>
> 基于 Claude Code 源码深度分析，提炼出的实用技巧。

---

## 1. Writing Effective Prompts / 怎么写 Prompt 更高效

### How the System Prompt is Assembled / System Prompt 拼装流程

Claude Code's system prompt has two layers:
1. **Static layer** (globally cached): identity, core rules, tool instructions, security policies
2. **Dynamic layer** (per-session): CLAUDE.md content, git state, MCP instructions, language preferences

Claude Code 的 system prompt 分为两层：
1. **静态层**（全局缓存）：身份、核心规则、工具说明、安全策略
2. **动态层**（每次会话变）：CLAUDE.md 内容、git 状态、MCP 指令、语言偏好

**Key insight / 关键洞察**: CLAUDE.md content is injected into the dynamic layer of the system prompt, with **higher priority** than ad-hoc chat instructions. / CLAUDE.md 的内容注入到 system prompt 动态层，优先级**高于**聊天中的临时指令。

### Tips / 实用技巧

**1. Put recurring rules in CLAUDE.md, not in chat / 反复用的规则写进 CLAUDE.md**
- CLAUDE.md is cached and loaded every turn / 每轮对话都会加载
- Hierarchy / 层级: `~/.claude/CLAUDE.md` (global) → project root `CLAUDE.md` → `.claude/rules/*.md`
- MEMORY.md limit: 200 lines or 25KB / MEMORY.md 上限 200 行或 25KB

**2. Reference code with `path:line_number` format / 引用代码用 `path:line_number` 格式**
- The system prompt teaches the model this convention / System prompt 教了模型这个惯例
- Example: "Check the weight calculation at `quant/strategy.py:132`"

**3. Break large tasks into subtasks / 大任务拆成子任务**
- The model is taught to "use TaskCreate to track progress, mark each done" / 模型被教导用 TaskCreate 跟踪进度
- Explicit breakdown = more reliable execution / 主动拆 = 更靠谱

**4. Specify "research only" vs "make changes" / 明确说「只研究」还是「直接改」**
- The system prompt emphasizes "read code before modifying" / System prompt 强调先读再改
- For research, explicitly say "Do not modify files" / 只想调研就说 "Do not modify files"

**5. Don't ask for time estimates / 别要求时间估算**
- The system prompt explicitly forbids predicting duration / System prompt 禁止预测耗时

### Keywords That Trigger Better Behavior / 触发更好行为的关键词

| You say / 你说的 | Model behavior / 模型反应 |
|--------|-----------|
| "read X first" | Always reads before modifying / 先读再改 |
| "don't over-engineer" | No unnecessary abstractions / 不加多余抽象 |
| "just fix the bug" | No drive-by refactoring / 不顺手重构 |
| "run tests after" | Runs tests after changes / 改完跑测试 |

### Context Limits (hardcoded) / 上下文限制（源码硬编码）

| Limit / 限制项 | Value / 值 |
|--------|-----|
| Model context window | 200K tokens (Opus/Sonnet 4.6: up to 1M) |
| Opus output tokens | 64K default, 128K max |
| Sonnet output tokens | 32K default, 128K max |
| Single tool result | 50,000 chars |
| All tool results per message | 200,000 chars |
| Git status truncation | 2,000 chars |
| MEMORY.md | 200 lines or 25KB |
| Images | 5MB base64, 2000×2000px |
| PDF | 100 pages, 20 per read |

---

## 2. Hook System / Hook 系统

### 26 Hook Events / 26 个 Hook 事件

| Event / 事件 | When / 触发时机 | Use case / 典型用途 |
|------|---------|---------|
| **PreToolUse** | Before tool execution / 工具执行前 | Block dangerous ops, modify args / 拦截危险操作 |
| **PostToolUse** | After tool execution / 工具执行后 | Inject context / 注入额外上下文 |
| **PostToolUseFailure** | Tool call fails / 工具调用失败 | Error handling / 错误处理 |
| **UserPromptSubmit** | User submits prompt / 提交 prompt | Auto-add context / 自动补充上下文 |
| **SessionStart** | Session begins / 会话开始 | Load env vars, register watch paths / 加载环境变量 |
| **SessionEnd** | Session terminates / 会话结束 | Log collection / 日志收集 |
| **Stop** | Before model finishes reply / 回复前 | Verify output quality / 验证输出质量 |
| **StopFailure** | API error ends turn / API 出错 | Error notification / 错误通知 |
| **SubagentStart/Stop** | Agent start/end / Agent 启停 | Agent monitoring / Agent 监控 |
| **PreCompact/PostCompact** | Before/after compaction / 压缩前后 | Preserve context / 保留关键上下文 |
| **FileChanged** | File modified / 文件变更 | Auto-reload config / 自动重载配置 |
| **CwdChanged** | Directory changed / 切换目录 | Reload .env / 重新加载 .env |
| **ConfigChange** | Config modified / 配置变更 | Hot reload / 热更新 |
| **PermissionRequest** | Permission dialog / 权限弹窗 | Auto approve/deny / 自动审批 |
| **PermissionDenied** | Auto-mode denies tool / 自动拒绝 | Audit logging / 审计日志 |
| **TaskCreated/Completed** | Task lifecycle / 任务生命周期 | Task tracking / 任务追踪 |
| **Notification** | System notification / 系统通知 | External alerts / 外部告警 |
| **Elicitation** | MCP requests input / MCP 请求输入 | User input handling / 用户输入处理 |
| **InstructionsLoaded** | CLAUDE.md loaded / 加载规则 | Rule auditing / 规则审计 |
| **WorktreeCreate/Remove** | Worktree lifecycle / Worktree 生命周期 | Cleanup / 清理 |

### 5 Hook Types / 5 种 Hook 类型

```json
// 1. Command — Run shell command / 跑 shell 命令
{"type": "command", "command": "echo $ARGUMENTS | jq .tool_name"}

// 2. Prompt — LLM evaluation (Haiku) / 用小模型判断
{"type": "prompt", "prompt": "Is this safe? $ARGUMENTS"}

// 3. Agent — Multi-turn agent verification / 多轮 agent 验证
{"type": "agent", "prompt": "Verify the implementation is correct"}

// 4. HTTP — POST to external service / POST 到外部服务
{"type": "http", "url": "https://webhook.example.com/hooks"}

// 5. Function — Runtime callback (not persisted) / 运行时回调（不存盘）
```

### Configuration Example / 配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo '$ARGUMENTS' | jq -e '.tool_input | test(\"rm -rf\") | not' || (echo 'Blocked rm -rf' >&2; exit 2)",
            "if": "Bash(*)"
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "cat .env 2>/dev/null && echo '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"watchPaths\":[\".env\"]}}'",
            "statusMessage": "Loading environment..."
          }
        ]
      }
    ]
  }
}
```

### Exit Code Semantics / Exit Code 语义

| Exit Code | PreToolUse effect / 效果 | Other events / 其他事件 |
|-----------|----------------|-------------|
| 0 | Allow execution / 允许执行 | Continue / 正常继续 |
| 2 | **Block execution**, stderr shown to model / **阻止执行** | stderr shown to model / 显示给模型 |
| Other | Non-blocking error / 非阻塞错误 | Non-blocking error / 继续 |

### Advanced Patterns / 高级模式

**Async background hook / 异步后台**:
```json
{"type": "command", "command": "npm test", "async": true}
```

**Async + rewake (wake model on failure) / 异步+唤醒（失败时唤醒模型修复）**:
```json
{"type": "command", "command": "npm test", "asyncRewake": true}
```

**One-time hook (runs once, then removed) / 一次性 hook**:
```json
{"type": "command", "command": "npm install", "once": true}
```

**Conditional execution / 条件执行**:
```json
{"type": "command", "command": "audit.sh", "if": "Bash(git *)"}
```

---

## 3. MCP Server Integration / MCP Server 集成

### Configuration Format / 配置格式

```json
// ~/.claude/settings.json, .claude/settings.json, or .mcp.json
{
  "mcpServers": {
    "my-server": {
      "type": "stdio",
      "command": "npx",
      "args": ["my-mcp-server"],
      "env": {"API_KEY": "xxx"}
    },
    "remote-api": {
      "type": "sse",
      "url": "https://api.example.com/mcp",
      "headers": {"Authorization": "Bearer token"}
    },
    "websocket-api": {
      "type": "ws",
      "url": "wss://api.example.com/mcp"
    }
  }
}
```

### Transport Types / 传输类型

| Type / 类型 | Use case / 场景 | Notes / 特点 |
|------|------|------|
| `stdio` | Local tools / 本地工具 | High process overhead, most stable / 进程开销大但最稳定 |
| `sse` | Cloud APIs / 云端 API | Auto-reconnect, recommended for remote / 自动重连 |
| `http` | Sync APIs / 同步 API | ~30s timeout / 超时约 30s |
| `ws` | Real-time / 实时更新 | Requires TLS / 需要 TLS |
| `sdk` | IDE extensions / IDE 扩展 | VS Code / JetBrains only |

### Connection Management / 连接管理

- **Exponential backoff reconnection / 指数退避重连**: initial 1s, max 30s, max 5 attempts
- **Concurrency limits / 并发限制**: local 2-4, remote ~20
- **OAuth caching / OAuth 缓存**: 401 failures cached for 15 min

### Tool Naming / 工具命名规则

MCP tools in Claude Code are named: `mcp__<server_name>__<tool_name>`
- Special chars replaced with `_` / 特殊字符替换为 `_`
- **Watch out**: tools with similar names may collide after normalization / 名字相似的工具可能冲突

### Tips for Building MCP Servers / 开发 MCP Server 的建议

1. **Keep tool descriptions concise** — the model decides when to call based on description / 模型靠描述决定何时调用
2. **Return results <100KB** — larger results get truncated / 超出会被截断
3. **Use env vars for secrets** — supports `$ENV_VAR` expansion / 支持环境变量展开
4. **Implement Resources** — let Claude list and read your data / 让 Claude 列出和读取数据
5. **Return structured errors** — avoid raw stack traces / 别返回 stack trace
6. **Support ToolListChanged notification** — triggers reconnect on dynamic tool changes / 工具变更时触发重连
7. **Implement elicitation** — for OAuth flows and user confirmations / 用于 OAuth 和用户确认

### Config Priority / 配置优先级

`.mcp.json` (project local) > `~/.claude/settings.json` (user) > `.claude/settings.json` (project shared) > Enterprise policy

---

## 4. Agent / Subagent System / Agent 系统

### Agent Types / Agent 类型

| Type / 类型 | Use / 用途 | Tool access / 工具权限 |
|------|------|---------|
| **general-purpose** | Multi-step tasks / 通用多步任务 | All tools / 全部 |
| **Explore** | Codebase exploration (read-only) / 代码探索 | Read-only / 只读 |
| **Plan** | Architecture design (read-only) / 架构设计 | Read-only / 只读 |
| **verification** | Test and verify / 测试验证 | All tools / 全部 |
| **claude-code-guide** | Claude Code help / 使用帮助 | Read + search / 读取+搜索 |

### When to Use What / 什么时候用什么

| Situation / 场景 | Pattern / 模式 |
|---------|---------|
| Need output before proceeding / 需要结果才能继续 | Sync agent (default) / 同步（默认） |
| Can work in parallel / 可以并行 | `run_in_background: true`, multiple in one message / 一条消息多个 |
| Need safe isolation / 需要隔离 | `isolation: "worktree"` |
| Multi-agent coordination / 多 Agent 协作 | `--agent-teams` + SendMessage |
| Cache efficiency / 省 token | Fork agent (omit `subagent_type`) / 省略 subagent_type |

### Parallel Agents / 并行启动

**Multiple agents in one message = true parallelism / 一条消息多个 Agent = 真并行**:
```
Agent({ description: "Research A", prompt: "...", run_in_background: true })
Agent({ description: "Research B", prompt: "...", run_in_background: true })
Agent({ description: "Research C", prompt: "...", run_in_background: true })
```

### Worktree Isolation / Worktree 隔离

```
Agent({
  description: "Experimental refactor",
  prompt: "...",
  isolation: "worktree"
})
```
- Runs in an isolated git worktree / 在独立 git worktree 中运行
- No changes → auto-cleanup / 没改动自动清理
- Has changes → returns worktree path and branch / 有改动返回路径和分支

### Writing Good Agent Prompts / Agent Prompt 写法

**Good / 好**:
```
Analyze the authentication flow in /project/src/auth.ts.
Focus on the verifyToken function (~line 45).
Research only — do not modify files.
Report: function signature, call chain, potential issues.
```

**Bad / 差**:
```
Check if there's anything wrong with auth.
```

### Key Rules / 关键规则

1. **Don't nest agents** — use Coordinator mode to orchestrate / 别嵌套，用 Coordinator
2. **Reads can parallelize, writes should serialize** (unless different files) / 读并行，写串行
3. **Give agents full context** — file paths, line numbers, expected outcome / 给完整上下文
4. **Specify "research only" vs "make changes"** / 明确说只研究还是改代码
5. **Fork agents** (omit `subagent_type`) reuse parent's prompt cache / Fork 复用缓存更省 token

### Custom Agents / 自定义 Agent

Place Markdown files in `.claude/agents/` / 在 `.claude/agents/` 下放 Markdown 文件:

```markdown
---
description: "Run and fix tests"
model: sonnet
tools: ["Bash", "Read", "Edit", "Write", "Grep", "Glob"]
maxTurns: 50
---

You are a test-fixing specialist. Run tests, analyze failures, fix code.
你是一个测试修复专家。运行测试，分析失败原因，修复代码。
```

**Definition fields / 定义字段**:

| Field | Description / 说明 |
|-------|-----------|
| `description` | 3-5 word summary / 简短描述 |
| `model` | `sonnet`, `opus`, `haiku`, or `inherit` |
| `tools` | Tool allowlist / 工具白名单 (e.g., `["Bash", "Read"]`) |
| `disallowedTools` | Tool denylist / 工具黑名单 |
| `maxTurns` | Turn limit / 轮数上限 (default: 200) |
| `background` | Always run in background / 总是后台运行 |
| `isolation` | `worktree` or `remote` |
| `permissionMode` | `default`, `plan`, or `yolo` |
| `omitClaudeMd` | Skip CLAUDE.md (saves tokens for read-only agents) / 跳过 CLAUDE.md |

### SendMessage Communication / 通信

```
SendMessage({ to: "agent-name", message: "Task done, result is..." })
SendMessage({ to: "*", message: "Broadcast to all teammates" })
```

---

## Appendix: Environment Variables / 附录：环境变量速查

| Variable / 变量 | Effect / 作用 |
|------|------|
| `CLAUDE_CODE_SIMPLE` | Minimal prompt (cwd + date only) / 极简 prompt |
| `CLAUDE_CODE_DISABLE_CLAUDE_MDS` | Skip all CLAUDE.md files / 跳过所有 CLAUDE.md |
| `CLAUDE_CODE_COORDINATOR_MODE=1` | Enable Coordinator orchestration / 启用编排模式 |
| `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` | Enable Agent Teams / 启用 Agent Teams |
| `CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1` | Disable background tasks / 禁用后台任务 |
| `CLAUDE_DEBUG=1` | Show MCP debug logs / 显示 MCP 调试日志 |

---

## Appendix: Hook Event Quick Reference / 附录：Hook 事件速查

<details>
<summary>All 26 hook events with input fields / 全部 26 个事件及输入字段</summary>

| Event | Matcher field | Key input fields |
|-------|--------------|-----------------|
| PreToolUse | `tool_name` | `tool_name`, `tool_input` |
| PostToolUse | `tool_name` | `tool_name`, `tool_input`, `response` |
| PostToolUseFailure | `tool_name` | `tool_name`, `error`, `error_type` |
| UserPromptSubmit | — | `prompt` |
| SessionStart | `source` | `source` (startup/resume/clear/compact) |
| SessionEnd | `reason` | `reason` (clear/logout/exit/other) |
| Stop | — | (empty) |
| StopFailure | `error` | `error` (rate_limit/auth_failed/billing/etc.) |
| SubagentStart | — | agent context |
| SubagentStop | — | agent context |
| PreCompact | — | compaction details |
| PostCompact | — | summary |
| Notification | `notification_type` | `message`, `type` |
| PermissionRequest | `tool_name` | `tool_name`, `tool_input` |
| PermissionDenied | `tool_name` | `tool_name`, `reason` |
| Setup | `trigger` | `trigger` (init/maintenance) |
| TeammateIdle | — | teammate context |
| TaskCreated | — | task details |
| TaskCompleted | — | task details |
| Elicitation | — | `mcp_server_name`, `message`, `requested_schema` |
| ElicitationResult | — | user response |
| ConfigChange | `source` | `source`, `file_path` |
| InstructionsLoaded | — | `file_path`, `memory_type` |
| WorktreeCreate | — | worktree details |
| WorktreeRemove | — | worktree details |
| CwdChanged | — | `old_cwd`, `new_cwd` |
| FileChanged | filename | `file_path`, `event` (change/add/unlink) |

</details>
