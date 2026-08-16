# pi-router-spec

pi 扩展：**首轮 Read-Only 侦察 → 全量续跑**。

交流讨论：<https://linux.do/>

在 provider payload 层（`before_provider_request`）重构首轮请求：

- 首轮（分支中不存在任何 `role === "assistant"` 的消息）→ 发送 `[极简 RECON_SYS, 仅 read 工具, 仅 user 消息]`
- think1 落地后（分支已有 assistant 消息）→ 完全不干预，pi 原始 payload 原样发出

判定完全由会话分支内容推导：零状态机、零消息注入、零持久化。

## 文件

- `pi-router-spec.ts` —— 单文件扩展（默认导出 factory，接收 `ExtensionAPI`）
- `package.json` —— pi package 清单（`pi.extensions` 声明扩展入口）
- `PRD.md` —— 需求与验收标准（§7 共 11 条）

## 安装
本仓库是标准 pi package（`package.json` 的 `pi` manifest + `pi-package` keyword），可通过 `pi install` 从 git 安装：

```bash
pi install git:github.com/Limour-dev/pi-router-spec
```

默认写入用户级 settings（`~/.pi/agent/settings.json`）；加 `-l` 写入项目级 settings（`.pi/settings.json`，可随仓库共享）：

```bash
pi install -l git:github.com/Limour-dev/pi-router-spec
```

安装后在 pi 内 `/reload` 生效。

### 仅引用单文件（开发迭代，不经 pi install）

在 `~/.pi/settings.json` 中加一行（仅引用本仓库文件，不复制）：
```json
{ "extensions": ["/home/limour/pi-router-spec/pi-router-spec.ts"] }
```

## 配置

模型门控常量位于文件顶部（§4.6）：

```ts
const TARGET_MODEL_IDS: string[] = ["deepseek-v4-pro", "deepseek-v4-flash"];
// 每条目为正则模式，忽略大小写（子串匹配）："deepseek-v4-flash" 也能匹配 "deepseek-v4-flash-0731"
// 空数组 = 对所有模型生效（不门控）
```

## 验证

重构发生时扩展会在调试输出打印：

```
[pi-router-spec] recon request { model, toolCount, messageRoles }
```

详细验收标准见 `PRD.md` §7（首轮重构 / 自动恢复 / 无工具调用降级 / 非首轮不触发 / /resume 不触发 / 模型门控 / /reload 热重载等）。

## 明确不做（Scope Out，见 PRD §9）

- 不做多轮 read 循环的 recon 阶段（严格一个 API 请求往返）
- 不做 followUp / sendUserMessage 续跑机制
- 不做 phase 状态持久化与跨会话恢复
- 不做 Flash/Pro band 分类、persona 注入、工具精简 schema
- 不做 read 次数上限与强制收尾