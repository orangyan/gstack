# 10 Lead-Sub-Agent 架构

## gstack 没有 Orchestrator-Worker 层级

gstack 不实现传统的 "Lead → Sub-Agent" 派生关系。它使用**水平扩展**的 pair-agent 模式和 Conductor 并行工作区。

---

## 10.1 Conductor 并行工作区模式

gstack 的多 Agent 模式主要通过 Conductor（Anthropic 工作空间管理工具）实现水平扩展：

```
Conductor 协调器
    │
    ├── workspace-1: /office-hours  (claude -p)
    ├── workspace-2: /review        (claude -p)
    ├── workspace-3: 实现           (claude -p)
    ├── workspace-4: /qa            (claude -p)
    ├── workspace-5: /ship          (claude -p)
    ...最多 10-15 个并行会话
```

**关键特性:**
- 每个工作区是独立的 `claude -p` 进程（无状态）
- 技能边界决定何时停止（每个技能知道自己的终止条件）
- 决策在关键点汇合（批准计划、批准 PR）
- 技能之间通过 git 文件传递状态，不通过内存

---

## 10.2 pair-agent 多 Agent 浏览器共享

gstack 的多 AI Agent 协作通过 pair-agent 实现浏览器共享：

```
Claude Code (主 agent)
    │ /pair-agent
    ▼
browse server 生成 setup key
    │ 通过 ngrok 隧道共享
    ▼
远程 AI Agent (OpenClaw/Hermes/Codex/Cursor)
    ├── 兑换 setup key → session token (24h)
    ├── 创建独立 tab (tabPolicy: 'own-only')
    └── 有限命令权限 (read+write scope, 无 admin/control)
```

**关键设计:** 多个 Agent 共享同一个 Chromium 实例，但各有自己的 Tab，通过 scoped token + tabPolicy 隔离。

---

## 10.3 与其他项目的 Sub-Agent 派生方式对比

| 维度 | gstack | deer-flow | hermes-agent | nanobot |
|------|--------|-----------|--------------|---------|
| **派生机制** | Conductor 并行工作区 | `delegate_task` 工具 | `task` 工具 | `spawn` 直接调用 |
| **通信方式** | 文件系统（git） | 事件总线 | 消息传递 | 直接函数调用 |
| **隔离程度** | 独立进程 | 同进程不同 Node | 同进程不同函数 | 同进程 |
| **跨模型** | ✅ (pair-agent: OpenClaw/Codex/Hermes) | ❌ | ❌ | ❌ |
| **浏览器共享** | ✅ (shared Chromium + tab isolation) | ❌ | ❌ | ❌ |
| **并发度** | 10-15 (实测 Conductor 最大值) | 2-3 (默认) | 可配置 | 无限制 |
| **监控** | Conductor UI | 事件总线日志 | WandB | 无 |

---

## 10.4 /autoplan：顺序 Agent 链（伪 Lead-Sub）

gstack 中最接近 Lead-Sub 模式的是 `/autoplan`：

```
/autoplan
    ↓
Step 1: CEO Review (claude -p 调用 /plan-ceo-review)
    ↓ 生成 ceo-review.md
Step 2: Design Review (claude -p 调用 /plan-design-review)
    ↓ 生成 design-audit.md
Step 3: Eng Review (claude -p 调用 /plan-eng-review)
    ↓ 生成 eng-plan.md
Step 4: 汇总结果，仅呈现需要人类决策的品味判断
```

**特点:** 顺序执行（非并行），通过文件传递状态，"主 agent"负责协调三个子 agent 的输出。

---

## 10.5 /codex：跨模型二次审查

```
Claude Code (/review)          OpenAI Codex (/codex review)
        │                              │
        ▼                              ▼
   claude-review.md              codex-review.md
        │                              │
        └──────────┬───────────────────┘
                   ▼
           自动交叉对比:
           ├── 重叠发现（两个模型都发现）
           ├── Claude 唯一发现
           └── Codex 唯一发现
```

这是 gstack 中唯一真正的"跨模型多 Agent"模式。
