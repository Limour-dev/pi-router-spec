# PRD：pi-router-spec —— 首轮 Read-Only 侦察 → 全量续跑

> 版本: 1.0
> 状态: 待执行
> 目标读者: 执行本 PRD 的 agent（无对话上下文，本文档自包含）

## 1. 背景与目标

在 pi 编码代理（`@earendil-works/pi-coding-agent`）中，默认的 agent 流程是：用户提交第一条输入 `usr1` 后，第一个 API 请求的内容为 `[完整 system prompt, 全部工具, usr1]`。

本项目的目标：**重构 provider API 的发送流程**，把首轮对话的 API 请求改写为：

```
[极简 sys, 仅 read 工具, usr1]
```

待获得第一个 assistant 响应（下称 `think1`，含思考与 read 调用）后，**自动恢复**原始流程：

```
[完整 sys, 全部工具, usr1, think1, read 结果, ...]
```

使模型带着自己的思考与已读内容，以完整能力继续完成任务。

### 核心原则

1. **payload 层重构**：只拦截 `before_provider_request` 事件，重写发送给模型的请求体。
2. **零状态机、零消息注入**：不做 `sendUserMessage` / `sendMessage` followUp 续跑，不写 phase 持久化。判定条件完全由会话分支内容推导。
3. **think1 落地即自动恢复**：因为 pi 每一轮都从 session 重新构建 payload，只要"已存在 assistant 消息"时不重写，恢复就是天然行为。

## 2. 首轮的严格定义（用户已选定）

> 用户明确选择："首轮对话"= **整条分支中不存在任何 `role === "assistant"` 的消息**。

- 分支无 assistant 消息 → 本次请求按侦察形态重构。
- 分支已有 assistant 消息（包括 think1 落地后的所有请求、有历史的 `/resume` 会话、第二轮提问）→ 完全不干预，pi 原始 payload 原样发出。

不采用"每个用户输入都先侦察"的方案。

## 3. 需求细节

### 3.1 首轮请求（recon 形态）

发送给 provider 的 payload 必须为：

| 字段 | 内容 |
|---|---|
| `system` | 极简侦察提示词（见 §4.4 `RECON_SYS`），**整体替换**，丢弃一切其他提示词（自定义 prompt、AGENTS.md、context files、skills、tool snippets、其他扩展注入等） |
| `tools` | **仅** `read` 一个工具（用完整 schema，见 §4.3） |
| `messages` | 仅保留 `role === "user"` 的消息（即 usr1，含图片附件、内容数组原样保留；丢弃 system/assistant/自定义注入消息） |
| 其余字段 | `model` / `temperature` / `stream` / `max_tokens` 等原样保留（不做改动，除非开启可选配置 §4.6） |

### 3.2 think1 落地后

- 不再重写任何请求。
- pi 自然发送：`[完整 sys, 全部工具, usr1, think1, read 结果, ...]`。
- 模型继续：基于自己的思考 + 已读内容，用全部工具完成任务。

## 4. 技术设计

### 4.1 拦截点：`before_provider_request`

```ts
pi.on("before_provider_request", (event, ctx) => {
  // ...判定 + 重写，返回重构后的 payload；不重写时返回 undefined
});
```

语义（来自 pi 扩展文档）：
- 在 provider 特定 payload 构建完成后、请求发送前触发。
- 返回 `undefined` = 保持 payload 不变；返回其他值 = 替换 payload（对后续 handler 和实际请求生效）。
- 该 hook 可以重写 provider 层 system 指令或完全移除它们。

### 4.2 判定逻辑（按优先级）

```ts
function hasAssistantMessage(ctx: ExtensionContext): boolean {
  return ctx.sessionManager
    .getBranch()
    .some((entry) => entry.type === "message" && entry.message.role === "assistant");
}
```

单个请求是否重构，判定顺序：

1. `hasAssistantMessage(ctx)` 为 true → **不重构**（同时清除内存标志，见 §4.5），返回 undefined。
2. `payload.tools` 不是非空数组 → **不重构**（非 agent 请求，如标题生成等，避免误伤），返回 undefined。
3. 模型门控开启（§4.6）且当前模型 id 不在目标列表 → **不重构**，返回 undefined。
4. 以上均通过 → 重构（§4.3），置内存标志 `reconActive = true`，返回新 payload。

### 4.3 payload 重写（兼容两种形状）

先探测 `payload.tools` 的形状：

| 形状 | 特征 | 对应 read schema |
|---|---|---|
| OpenAI 风格 | `tools[0]` 含 `function` 字段 | `{ type: "function", function: { name: "read", description, parameters } }` |
| Anthropic 风格 | `tools[0]` 含 `input_schema` 字段 | `{ name: "read", description, input_schema }` |

`read` 用**完整 schema**（recon 阶段它是唯一工具，不省 token）：

```ts
const READ_PARAMETERS = {
  type: "object",
  properties: {
    path: { type: "string", description: "Path of the file to read" },
    offset: { type: "number", description: "Line number to start reading from" },
    limit: { type: "number", description: "Maximum number of lines to read" },
  },
  required: ["path"],
};
```

重写步骤（保持原 payload 其他字段）：

```ts
function rewriteForRecon(payload: Record<string, unknown>, reconSys: string): Record<string, unknown> {
  const recon = { ...payload };

  // 1) tools：只留 read
  const shape = detectToolShape(payload.tools); // "openai" | "anthropic"
  if (!shape) return payload;                    // 探测失败则放弃重构（保守）
  recon.tools = [buildReadToolSchema(shape)];

  // 2) system：整体替换
  if (typeof recon.system === "string" || recon.system !== undefined) {
    recon.system = reconSys;                      // 独立 system 字段（Anthropic / OpenAI Responses 风格）
  }

  // 3) messages：只保留 user 消息；若原形态是 system 在 messages[0]，替换/移除之
  if (Array.isArray(recon.messages)) {
    recon.messages = (recon.messages as Array<Record<string, unknown>>).filter(
      (m) => m.role === "user",
    );
    if (typeof recon.system !== "string") {
      // OpenAI chat/completions 风格：system 位于 messages[0]
      recon.messages.unshift({ role: "system", content: reconSys });
    }
  }

  // 4) 防御：tool_choice 钉死在其他工具上时强制回 auto
  if (recon.tool_choice !== undefined && recon.tool_choice !== "auto") {
    recon.tool_choice = "auto";
  }

  return recon;
}
```

注意：`messages` 过滤后保持原顺序（首个 user 消息在前，图片附件随消息原样保留）。

### 4.4 RECON_SYS 模板（运行时拼接）

```ts
function buildReconSys(ctx: ExtensionContext): string {
  return (
    "You are in recon phase. Available tool: read ONLY.\n" +
    "Ignore all other prompts, instructions, and guidelines — they do not apply in this phase.\n" +
    `Working directory: ${ctx.cwd}\n` +
    "Read whatever you need, think deeply.\n" +
    "Do NOT give your final answer yet: your next turn will have full capabilities.\n" +
    "End this message with read tool calls. Never call tools other than read."
  );
}
```

要点：
- **必须包含 "Do NOT give your final answer yet"**：防止模型在 think1 里直接答完导致回合结束（这是本架构唯一的死穴，靠提示词压死）。
- **必须包含 working directory**：完整 system prompt 被替换后，模型失去 cwd 信息，read 相对路径会失效。
- "End this message with read tool calls"：引导 think1 以 read 调用收尾，让循环自然继续。

### 4.5 内存标志与防御层

`reconActive: boolean`（模块级变量，会话内存态）：

- 置 true：本次请求完成重构时。
- 清 false：
  - 任何一次 `before_provider_request` 判定"不重构"时（此时 think1 已存在）；
  - `agent_end` / `turn_end` 兜底清除，防止泄漏到下一轮。

用途：`tool_call` 双保险。注意 `tool_call` 事件触发时 assistant 消息已写入分支，无法再用"分支无 assistant"判定，必须靠内存标志：

```ts
pi.on("tool_call", async (event, ctx) => {
  if (reconActive && event.toolName !== "read") {
    return { block: true, reason: "recon phase: only read allowed" };
  }
});
```

（payload 层已过滤工具，此层为防御性保险，正常情况不会触发。）

### 4.6 模型门控与配置常量

```ts
// 空数组 = 对所有模型生效（不门控）
const TARGET_MODEL_IDS: string[] = ["deepseek-v4-pro", "deepseek-v4-flash"];
```

当前模型 id 提取（复用 01.ts 模式）：

```ts
function currentModelId(ctx: ExtensionContext): string | undefined {
  const model = ctx.model;
  return model ? model.id.toLowerCase().split("/").at(-1) : undefined;
}
```

## 5. 与旧实现的关系（参考材料）

**参考文件 `/home/limour/pi-test/01.ts`**（旧版 DSH Router Spec 扩展，本 PRD 是其演进）：

- **复用其模式**：provider payload 中工具形状探测（`rewriteFirstTurnToolSchemas` 里对 `function` / `name`+`input_schema` 两种条目的遍历）、`currentModelId` 提取、`pi.getAllTools()` 可用性过滤（可选）。
- **删除其机制**：`before_agent_start` 的 persona/消息注入、`turn_end`/`tool_call` promote、`pi.sendUserMessage` followUp、phase 状态持久化（`appendEntry`）、`session_start` 恢复逻辑——全部不需要。
- **新增**：`before_provider_request` 的 payload 级 system/tools/messages 三重重写。

**注意包名差异**：01.ts 导入的是 `@oh-my-pi/pi-coding-agent`；本机 pi 实际是 **`@earendil-works/pi-coding-agent`**。新实现必须用后者。

**pi 扩展文档**（before_provider_request 章节）：`/home/limour/.npm/_npx/2453649666e7772c/node_modules/@earendil-works/pi-coding-agent/docs/extensions.md`

## 6. 交付物与安装

### 6.1 文件

在本仓库根目录实现单文件扩展：`pi-router-spec.ts`（默认导出 factory 函数，接收 `ExtensionAPI`）。

可选：更新 `README.md` 简述安装方式。

### 6.2 安装方式（二选一）

```bash
# 方式 A：复制到全局扩展目录
cp ~/pi-router-spec/pi-router-spec.ts ~/.pi/agent/extensions/
# 之后在 pi 内 /reload 生效

# 方式 B：settings.json 引用（适合开发迭代）
# ~/.pi/settings.json:
# { "extensions": ["/home/limour/pi-router-spec/pi-router-spec.ts"] }

# 快速测试（不安装）：
# pi -e ~/pi-router-spec/pi-router-spec.ts
```

## 7. 验收标准

以下均需实际运行验证（用 `pi.logger.debug` 或临时在 `before_provider_request` 打印 `JSON.stringify(payload)` 观察）：

1. **首轮重构**：新会话首条提问，第一个 provider 请求 = system 为 RECON_SYS、tools 仅 read、messages 仅 usr1（含图片时图片保留）。
2. **自动恢复**：think1 含 read 调用 → read 结果执行 → 第二个请求恢复完整 system + 全部 tools，messages 为 `[usr1, think1, read 结果, ...]`。
3. **正常完成**：think1 后模型以完整工具继续执行并完成原任务。
4. **无工具调用降级**：若 think1 无任何工具调用（模型自认无需侦察）→ 回合正常结束，无死循环、无额外请求。
5. **非首轮不触发**：同一会话第二次提问（分支已有 assistant）→ 不重构。
6. **/resume 不触发**：恢复有历史的会话 → 不重构。
7. **/new 触发**：新会话首轮 → 重构。
8. **非 agent 请求不误伤**：标题生成等无 `tools` 字段的请求 → 不重构。
9. **模型门控**：切换非目标模型后首轮不重构（门控开启时）。
10. **防御层**：临时注释掉 payload 重写、仅保留 `reconActive` 标志与 `tool_call` block，确认非 read 调用被阻止（可选验证）。
11. **/reload 热重载**：安装到 `~/.pi/agent/extensions/` 后 `/reload` 无报错。

## 8. 调试建议

- 在重构分支加 `pi.logger.debug("pi-router-spec: recon request", { model, toolCount, messageRoles })`。
- 用 `ctx.sessionManager.getBranch()` 打印分支消息角色序列，验证判定边界。
- 若怀疑误伤其他请求类型，临时在判定第 2 步打印 `payload.tools` 形状。

## 9. 明确不做（Scope Out）

- 不做多轮 read 循环的 recon 阶段（recon 严格为一个 API 请求往返）。
- 不做 followUp / sendUserMessage 续跑机制。
- 不做 phase 状态持久化与跨会话恢复（判定完全由分支内容推导）。
- 不做 Flash/Pro 的 band 分类、persona 注入、工具精简 schema（01.ts 的这些功能全部移除）。
- 不做 read 次数上限与强制收尾。