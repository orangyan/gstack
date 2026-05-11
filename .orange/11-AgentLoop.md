# 11 AgentLoop 实现

## gstack 不实现自己的 AgentLoop

gstack 复用 Claude Code 的原生 while-loop ReAct。技能只是 Claude 的**指令文档（系统提示）**，不是可执行文件。

---

## 11.1 gstack 的"Agent Loop"本质

```
Claude Code 原生 Loop (query.ts 驱动):
    用户输入 /qa https://staging.myapp.com
        ↓
    Claude Code 发现 ~/.claude/skills/gstack/qa/SKILL.md
        ↓
    将 SKILL.md 内容注入到系统提示/上下文
        ↓
    ┌──── ReAct 循环 ────────────────────────────────┐
    │  while (!done && turns < maxTurns):             │
    │      response = LLM(messages + tools)           │
    │      if response.tool_use:                      │
    │          result = executeTool(tool_use)         │
    │          messages.append(tool_result)           │
    │      else:                                      │
    │          done = true (最终回复)                  │
    └────────────────────────────────────────────────┘
```

**SKILL.md 在这个循环中的角色:**
- 不是代码，是**指令**
- Claude 读取 SKILL.md，理解"接下来做什么"
- 每一步都由 LLM 根据 SKILL.md 指令决策

---

## 11.2 E2E 测试中的 AgentLoop

E2E 测试通过 `session-runner.ts` 启动独立的 Claude Code 进程：

```typescript
// test/helpers/session-runner.ts
const proc = spawn('claude', [
  '-p',
  '--model', 'claude-sonnet-4-6',
  '--output-format', 'stream-json',
  '--verbose',
  '--dangerously-skip-permissions',
  '--max-turns', String(maxTurns),  // 默认 15
  '--allowed-tools', allowedTools.join(','),
], { stdin: 'pipe', stdout: 'pipe' });

// 流式 NDJSON 输出
for await (const line of proc.stdout) {
  const event = JSON.parse(line);
  if (event.type === 'assistant') turns++;
  if (event.type === 'result') result = event;
}
```

**测试中的 Loop 控制:**
- `max-turns`: 默认 15，防止无限循环（E2E 测试中）
- `allowed-tools`: 明确限制工具范围（Bash Read Write）
- `stream-json`: 流式 NDJSON，可观察每一步

---

## 11.3 与其他项目的 AgentLoop 对比

| 维度 | gstack | deer-flow | hermes-agent | nanobot |
|------|--------|-----------|--------------|---------|
| **实现方式** | 复用 Claude Code 原生 Loop | LangGraph 中间件链 | 手写 while 循环 | 异步事件总线 |
| **主要文件** | SKILL.md (指令) + claude 二进制 | `middleware.py` | `agent_loop.py` | `event_bus.py` |
| **中间件/钩子** | ❌ (由 SKILL.md 文字描述) | ✅ (Python 类) | ✅ (函数列表) | ✅ (事件处理器) |
| **最大轮次** | --max-turns (默认 15 for E2E) | max_turns (默认 90) | max_agent_turns (默认 90) | 无硬限制 |
| **工具并行** | ✅ (Claude Code 原生) | ✅ (LangGraph Send) | ✅ (asyncio.gather) | ❌ (顺序) |
| **流式输出** | ✅ (stream-json) | ✅ (LangGraph stream) | ✅ (AsyncGenerator) | ❌ |
| **可观察性** | ✅ (NDJSON 每步可见) | ✅ (LangGraph events) | ✅ (WandB) | ⚠️ (日志) |

---

## 11.4 PTY 模式下的 AgentLoop

对于需要交互式 TTY 的场景（如 plan-mode），gstack 使用真实 PTY：

```typescript
// test/helpers/claude-pty-runner.ts
const proc = Bun.spawn({
  cmd: ['claude', '--model', model, ...args],
  stdin: 'pipe',
  stdout: 'pipe',
  terminal: true,   // ← 关键：真实 PTY
});

// 解析 ANSI 转义后的渲染文本
// 验证 AskUserQuestion 在 plan-mode 下的 UI 渲染
```

**使用场景:** 验证 AskUserQuestion 在 plan mode 中不走工具路径（直接渲染到终端），确保交互式 UI 正确。

---

## 11.5 gstack Loop 的独特优势

**无需维护 Loop 代码的好处:**
1. 自动受益于 Claude Code 的所有改进（流式、并行工具、内存压缩）
2. 技能只需关注"做什么"，不需关注"怎么循环"
3. 用户的 `--max-turns` 参数对所有技能生效

**代价:**
- 无法直接控制 Loop 行为（中间件、钩子）
- 依赖 Claude Code 版本（claude binary 更新可能影响行为）
- 调试需要分析 NDJSON 流，而非断点
