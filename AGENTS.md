# 给 AI 助手看的（本仓库 = 一个 DSH 插件）

`@dsh-cbw/memory`（注册名 `memory`）：DSH 长期记忆后端，按工作区共享记忆（同一工作区多会话共用一份记忆），经 `systemPrompt.context` 注入。

## 派生关系

派生自 [Yuuyuko-uu/dsh-fish-memory](https://github.com/Yuuyuko-uu/dsh-fish-memory)（Yuuyuko-uu 插件许可 1.2）。相对上游的改动见 `D:\DSH-Dev\docs\sub-projects\dsh-cbw-memory.md`。

## 仓库结构

- `lib/index.js` —— host 半端：工具注册 + 记忆存储/检索/沉淀
- `lib/client.js` —— 浏览器半端：设置页
- `cordis.patch.yml` —— DSH bundle patch
- `LICENSE` / `THIRD_PARTY_NOTICES.md` —— 许可证合规（派生声明 + 借鉴来源）

## DSH-Dev 工作区协调（强制）

本仓库是 `D:\DSH-Dev` 工作区的子项目。工作区有多个会话并行，靠 cross-talk 文件协调。**启动时和工作中必须遵守**，否则会跟其他会话撞车或漏接派给本会话的任务。

- **cross-talk 文件**：`D:\DSH-Dev\docs\cross-talk\dsh-cbw-memory.md`
- **分片文件**：`D:\DSH-Dev\docs\sub-projects\dsh-cbw-memory.md`

### 启动时必须执行

1. **读 `D:\DSH-Dev\docs\cross-talk\` 全目录**：了解其他会话在做什么、有无对自己的通知/请求
2. **读自己的 cross-talk 文件**：确认上次状态，准备更新
3. **逐个检查其他会话文件**的"对其他会话的通知/请求"段有无给自己派的任务；有则处理

### 工作中必须执行

1. **有进展/困难/经验/跨会话通知时，更新自己的 cross-talk 文件**（覆盖式更新）
2. **只改自己的文件**：本子项目仓库 + 自己的分片 + 自己的 cross-talk 文件；**不碰根 README 主体、其他分片、其他 cross-talk 文件**
3. **回复其他会话的请求**：处理完派给本会话的任务后，在自己的 cross-talk 文件"对其他会话的通知/请求"段留回复（格式：`→ @<对方会话>：<事项> 已完成/已处理`）

### 提交推送

- **所有 git 指令用 fgit**：`D:/Tool/fgit/fgit.exe <子命令>`，bash 工具里**不要加 `2>&1`**（会导致超时）
- **逐条执行**：不要 chained `&&`，逐条 add → commit → push
- **暂存**：只 `git add` 相关文件，**不要 `git add -A`**
- **push 前 rebase**：`git pull --rebase origin main` 后再 push

> 根工作区完整约定见 `D:\DSH-Dev\AGENTS.md`。本段是其子项目适用部分的摘录，有冲突以根文件为准。