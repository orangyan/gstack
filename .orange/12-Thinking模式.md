# 12 Thinking 模式

## gstack 的 Thinking 模式控制

gstack 本身不直接控制 Extended Thinking API 参数。Thinking 模式由底层模型和 Claude Code 配置决定。但 gstack 通过 **Model Overlay** 机制在生成时注入模型特定的行为指令。

---

## 12.1 Model Overlay 机制

**生成时注入 (scripts/resolvers/model-overlay.ts):**

```
gen-skill-docs --model claude-opus-4-7
    │
    ▼
model-overlay.ts 检查模型类型
    ├── 如果是 Opus 4.7 → 注入 Opus 特定行为指令
    │   (e.g., "花更多时间在边缘案例分析上")
    └── 如果是 Sonnet → 使用标准指令
```

**注意:** 这是**文本指令**注入（注入到 SKILL.md），而非 API 参数级别的 `thinking: {type: "enabled", budget_tokens: N}`。

---

## 12.2 Opus 4.7 专项 E2E 测试

```
test/skill-e2e-opus-47.test.ts
    ├── 专门针对 claude-opus-4-7 模型运行
    ├── 测试分类: 'periodic' (每周，非 CI gate)
    └── 验证 Opus 的特定行为是否符合预期
```

---

## 12.3 与其他项目的 Thinking 模式对比

| 维度 | gstack | deer-flow | hermes-agent | nanobot |
|------|--------|-----------|--------------|---------|
| **Extended Thinking** | ❌ (文字指令代替) | ✅ (budget_tokens) | ✅ (reasoning_effort) | ✅ (thinking_content) |
| **控制粒度** | 模型特定文本指令 | 全局 budget_tokens 参数 | per-call reasoning_effort | 全局 thinking 开关 |
| **Thinking 记录** | ❌ | ❌ | ✅ reasoning_per_turn | ✅ thinking_content |
| **动态调整** | 重新生成 SKILL.md | 运行时参数 | 运行时参数 | 运行时参数 |
| **模型特化** | ✅ (Opus 4.7 专项) | ❌ | ✅ (NousHermes 优化) | ❌ |

---

## 12.4 gstack 的行为调优哲学

gstack 认为**精心设计的文字指令**比 Extended Thinking API 更可控：

1. **Preamble Tier 系统**: 不同复杂度的技能注入不同深度的思考指令
   - Tier 1: 轻量指令（browse 命令用）
   - Tier 4: 完整指令（ship/review 这类高复杂度任务用）

2. **写作风格 V1** (`docs/designs/PLAN_TUNING_V1.md`):
   - 专门设计的输出风格指令（避免 AI 味）
   - 用户可通过 `gstack-config set explain_level terse` 切换到简洁模式

3. **Slop-scan 质量检测** (`scripts/slop-scan`):
   - 检测 AI 生成代码的质量模式
   - 不是为了"看起来像人类"，而是为了真正的代码质量

---

## 12.5 隐式 Thinking 触发

虽然 gstack 不调用 Extended Thinking API，但通过以下方式隐式促进深度推理：

```
SKILL.md 中的指令:
    - "Before writing code, think through the edge cases"
    - "Run gstack-learnings-search first to recall past mistakes"
    - "Challenge the assumption: why are we building this?"  (/office-hours)
    - "List 3 hypotheses before testing"  (/investigate)
```

这些指令通过 Claude 的标准推理（而非 Extended Thinking）实现了类似深度思考的效果。
