# 13 SKILL 管理

## gstack 的 SKILL.md 系统

gstack 的 SKILL.md 是整个框架的核心——技能不是可执行代码，而是 Claude 的**指令文档**。这是 gstack 与其他 Agent 框架最根本的区别。

---

## 13.1 SKILL.md 的本质

```
SKILL.md ≠ 脚本文件
SKILL.md = Claude 的"岗位说明书"
```

当用户输入 `/qa https://staging.myapp.com` 时：
1. Claude Code 找到 `~/.claude/skills/gstack/qa/SKILL.md`
2. Claude **读取**该文件，理解自己要做什么
3. Claude **自己决策**每一步（调用 Bash、Read、AskUserQuestion）
4. SKILL.md 是指令，不是代码

---

## 13.2 技能生命周期

```
.tmpl 文件 (开发者维护)
    ↓ bun run gen:skill-docs
.md 文件 (生成，提交到 git)
    ↓ setup 脚本 (符号链接)
~/.claude/skills/gstack/{skill}/SKILL.md
    ↓ Claude Code 运行时
Claude 读取并执行
```

**关键**: `.tmpl` 是真相来源，`.md` 是生成物。解决合并冲突时，只改 `.tmpl`，然后重新生成。

---

## 13.3 SKILL.md 格式

```yaml
---
name: qa
preamble-tier: 4
version: 1.8.0.0
description: |
  Systematically tests a staging URL for bugs. (gstack)
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
triggers:
  - test a staging URL
  - find bugs in my app
voice-triggers:
  - "run QA on"
  - "test my staging"
---

{{PREAMBLE}}

# QA Engineer

## Step 0: Setup
```bash
# Preamble bash 代码块（每次技能启动时运行）
{{PREAMBLE_BASH}}
```

## Step 1: Navigate to URL
...

## Step N: Generate Report
{{QA_METHODOLOGY}}
```

---

## 13.4 模板占位符系统（完整列表）

| 占位符 | 功能 |
|--------|------|
| `{{PREAMBLE}}` | 完整 preamble（版本检查、会话跟踪、配置读取、学习搜索） |
| `{{BROWSE_SETUP}}` | 标准化 browse 设置指令 |
| `{{COMMAND_REFERENCE}}` | 从 commands.ts 自动生成的命令参考 |
| `{{SNAPSHOT_FLAGS}}` | snapshot 命令的 flag 说明 |
| `{{BASE_BRANCH_DETECT}}` | PR/main 分支检测脚本 |
| `{{QA_METHODOLOGY}}` | QA 方法论（系统性 bug 发现流程） |
| `{{DESIGN_METHODOLOGY}}` | 设计评审方法论 |
| `{{TEST_COVERAGE_AUDIT_SHIP}}` | Ship 时的测试覆盖率审计 |
| `{{REVIEW_DASHBOARD}}` | 审查就绪仪表板格式 |
| `{{CHANGELOG_WORKFLOW}}` | CHANGELOG 编辑规则 |
| `{{LEARNINGS_SEARCH}}` | 学习搜索代码块 |
| `{{LEARNINGS_LOG}}` | 学习记录代码块 |
| `{{GBRAIN_CONTEXT_LOAD}}` | GBrain 上下文加载（GBrain host 特有） |

---

## 13.5 Preamble Tier 系统

```
Preamble Tier 1 (browse, benchmark):
    ├── 版本检查 (gstack-update-check)
    ├── 会话跟踪 (touch ~/.gstack/sessions/$PPID)
    └── 基础遥测

Preamble Tier 2 (investigate, retro):
    ├── Tier 1 全部
    ├── AskUserQuestion 格式规范
    ├── 写作风格 V1 注入
    └── 上下文恢复提示

Preamble Tier 3 (office-hours, plan reviews):
    ├── Tier 2 全部
    ├── Repo mode section
    └── Search Before Building 原则

Preamble Tier 4 (ship, review, qa):
    ├── Tier 3 全部
    ├── 完整 LEARNINGS_SEARCH
    ├── 配置读取 (PROACTIVE, EXPLAIN_LEVEL, QUESTION_TUNING)
    └── 遥测写入
```

---

## 13.6 多 Host 技能生成

同一份 `.tmpl`，生成不同 Host 的 SKILL.md：

| Host | 工具名 | 路径重写 | frontmatter 处理 |
|------|--------|---------|-----------------|
| claude | Bash | 无 | 完整 |
| codex | Shell | `~/.claude/` → `$GSTACK_ROOT/` | 白名单字段 |
| kiro | Bash | `~/.claude/` → `$GSTACK_ROOT/` | 截断 description |
| openclaw | ExecuteBash | `~/.claude/` → `$GSTACK_ROOT/` | 格式适配 |
| cursor | Terminal | 无 | AI rules 格式 |

---

## 13.7 与其他项目的 SKILL 系统对比

| 维度 | gstack | deer-flow | hermes-agent | nanobot |
|------|--------|-----------|--------------|---------|
| **技能格式** | SKILL.md (指令文档) | SKILL.md (指令文档) | Skills.db (SQL) | skills/ (代码文件) |
| **执行方式** | Claude 读取并决策 | Claude 读取并决策 | SELECT + 注入提示 | Python 函数调用 |
| **生成机制** | tmpl → gen-skill-docs | 手写 | LLM 学习后写入 | 手写 |
| **版本控制** | git (.tmpl + .md) | git | SQLite | git |
| **跨模型支持** | ✅ (10+ hosts) | ❌ | ❌ | ❌ |
| **自进化** | ⚠️ (学习 → 更新 tmpl) | ✅ (LLM 自动更新) | ✅ (LLM 自动写入) | ❌ |
| **Token 预算** | 40KB 警告上限 | 无 | 无 | 无 |
| **验证** | frontmatter schema 验证 | 无 | SQL schema | 无 |

---

## 13.8 技能质量保证

**静态验证 (免费, <1s):**
```bash
bun test test/skill-validation.test.ts
# 检查: frontmatter 有效性, allowed-tools 合法性, name 与目录名匹配
```

**LLM 质量评估 (~$0.15/run):**
```bash
bun run test:evals
# claude-sonnet-4-6 评分: clarity(1-5), completeness(1-5), actionability(1-5)
```

**E2E 行为验证 (~$3.85/run):**
```bash
bun run test:e2e
# claude -p 全流程执行，LLM judge 评分实际输出质量
```

**Slop-scan (免费):**
```bash
bun run slop          # 全文件扫描
bun run slop:diff     # 只扫描当前分支改动
```
