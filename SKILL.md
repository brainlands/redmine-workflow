---
name: redmine-workflow
description: 一条命令启动完整需求工作流：输入 Redmine 需求地址或需求号 → 自动创建 Git 分支 → 分析需求文档生成前端技术文档 → 可选创建前端子任务并指派给当前用户。激活条件：用户提到"需求"、"分支"、"redmine"、"创建分支"、"技术文档"、"前端任务"、"red-branch"等关键词，或粘贴了 Redmine 需求链接。
---

# Redmine 需求工作流 Skill

基于 `red-cli` + `jojogithook` API + 钉钉文档 MCP 的完整需求开发工作流自动化。

## 激活条件

当用户满足以下任一条件时激活此 skill：

- 粘贴了 Redmine 需求链接（如 `https://your-redmine-domain.com/issues/526205`）
- 输入了纯需求号（如 `526205`）
- 提到"创建分支"、"开需求"、"接任务"、"需求工作流"
- 说"帮我创建分支"、"帮我开需求"
- 任何涉及 Redmine + 分支 + 需求的组合意图

## 前置依赖

| 依赖              | 安装方式                                       | 作用                                            |
| ----------------- | ---------------------------------------------- | ----------------------------------------------- |
| `red-cli`         | `brew install red-cli`                         | Redmine CLI 工具，已配置在 `~/.red/config.json` |
| `curl`, `jq`      | macOS 自带 / `brew install jq`                 | API 调用和 JSON 解析                            |
| `git`             | macOS 自带                                     | 本地分支操作                                    |
| `red-branch` 脚本 | 位于 `~/.local/bin/red-branch` 或本 skill 目录 | 分支创建自动化                                  |
| 钉钉文档 MCP      | 已配置在 `~/.claude/settings.json`             | 创建/写入钉钉技术文档                           |

## 关键配置文件

```
~/.red/config.json — red-cli 配置（包含 api-key, server, user-id）
```

关键字段提取方式：

```bash
API_KEY=$(jq -r '."api-key"' ~/.red/config.json)
SERVER=$(jq -r '.server' ~/.red/config.json)
USER_ID=$(jq -r '."user-id"' ~/.red/config.json)
```

## 关键 API 端点

| 用途             | 方法 | URL                                                                                          | 说明                                 |
| ---------------- | ---- | -------------------------------------------------------------------------------------------- | ------------------------------------ |
| 查询需求         | GET  | `{SERVER}/issues/{ID}.json`                                                                  | 获取需求详情、标题、项目、自定义字段 |
| 蟥询用户         | GET  | `{SERVER}/users/{USER_ID}.json`                                                              | 获取用户 login 名                    |
| 查询项目         | GET  | `{SERVER}/projects/{PROJECT_ID}.json`                                                        | 获取项目 identifier                  |
| 获取 GitLab 项目 | GET  | `https://your-githook-domain.com/api/v1/getuserproject?username={login}&identifier={identifier}` | Header: `token: {YOUR_GITHOOK_TOKEN}`    |
| 创建 GitLab 分支 | POST | `https://your-githook-domain.com/api/v1/newbranch?id={gitlab_project_id}&branch={branch_name}`   | Header: `token: {YOUR_GITHOOK_TOKEN}`    |
| 创建子任务       | POST | `{SERVER}/issues.json`                                                                       | 在 Redmine 上创建"开发任务"子任务    |

### Redmine 自定义字段 ID 映射

| ID  | 名称             | 用途                               |
| --- | ---------------- | ---------------------------------- |
| 5   | 需求文档链接     | 需求页上的产品需求文档 URL         |
| 6   | 技术方案文档链接 | 前端技术方案文档 URL（写入子任务） |

### Tracker ID 映射

| ID  | 名称     | 用途                           |
| --- | -------- | ------------------------------ |
| 2   | 开发任务 | 创建前端子任务时使用此 tracker |

## 完整工作流（8 步）

用户输入需求号或 Redmine 地址后，按以下步骤执行。每一步的结果都要向用户展示，关键决策点（Step 4、Step 6）必须询问用户。

---

### Step 1: 解析输入，提取需求号

从用户输入中解析需求号：

- 如果输入是 URL（如 `https://your-redmine-domain.com/issues/526205`），提取末尾数字部分
- 如果输入是纯数字（如 `526205`），直接使用
- 如果输入包含其他文字，尝试提取其中的数字

```bash
# 解析逻辑示例
INPUT="https://your-redmine-domain.com/issues/526205"
ISSUE_ID=$(echo "$INPUT" | grep -oE '[0-9]+' | tail -1)
```

向用户展示：`需求号: {ISSUE_ID}`

---

### Step 2: 查询 Redmine 需求详情

通过 Redmine API 获取完整需求信息：

```bash
ISSUE_JSON=$(curl -s -H "X-Redmine-API-Key: $API_KEY" "${SERVER}/issues/${ISSUE_ID}.json")
```

提取并展示以下信息：

| 字段             | jq 路径                 | 示例                             |
| ---------------- | ----------------------- | -------------------------------- | ------- | ------------ |
| 标题             | `.issue.subject`        | 【P0】风控接入-品牌/班级拍摄活动 |
| 项目名           | `.issue.project.name`   | 中台-课内                        |
| 项目 ID          | `.issue.project.id`     | 131                              |
| Tracker          | `.issue.tracker.name`   | 需求【强审批】                   |
| 状态             | `.issue.status.name`    | 新建                             |
| 优先级           | `.issue.priority.name`  | P0                               |
| 需求文档链接     | `.issue.custom_fields[] | select(.id==5)                   | .value` | 钉钉文档 URL |
| 技术方案文档链接 | `.issue.custom_fields[] | select(.id==6)                   | .value` | 可能为空     |

同时获取用户名和项目 identifier：

```bash
USER_LOGIN=$(curl -s -H "X-Redmine-API-Key: $API_KEY" "${SERVER}/users/${USER_ID}.json" | jq -r '.user.login')
PROJECT_IDENTIFIER=$(curl -s -H "X-Redmine-API-Key: $API_KEY" "${SERVER}/projects/${PROJECT_ID}.json" | jq -r '.project.identifier')
```

向用户展示完整需求信息摘要。

---

### Step 3: 创建 Git 分支

使用 `red-branch` 脚本或直接调用 API 创建分支。

**方案 A：使用脚本（推荐）**

```bash
red-branch {ISSUE_ID} --project {PROJECT_NAME}
```

**方案 B：直接调用 API**

```bash
# 1. 获取 GitLab 项目列表
GITLAB_PROJECTS=$(curl -s -H "token: ${GITHOOK_TOKEN}" \
  "https://your-githook-domain.com/api/v1/getuserproject?username=${USER_LOGIN}&identifier=${PROJECT_IDENTIFIER}")

# 2. 如果只有一个项目，直接使用；多个项目需用户选择
# 3. 创建远程分支
curl -s -X POST -H "token: ${GITHOOK_TOKEN}" \
  "https://your-githook-domain.com/api/v1/newbranch?id={GITLAB_PROJECT_ID}&branch=feature/${USER_LOGIN}/${ISSUE_ID}"

# 4. 本地 git 操作
git checkout -b feature/${USER_LOGIN}/${ISSUE_ID}
```

**分支名格式**：`feature/{USER_LOGIN}/{ISSUE_ID}`（如 `feature/albert/526205`）

如果用户已在某个 git 仓库目录中，自动创建本地分支并切换。如果不在 git 仓库中，只创建远程分支并提示用户手动操作。

向用户展示：`✅ 分支已创建: feature/albert/526205`

---

### Step 4: 分析需求文档，生成前端技术文档 ⭐ 决策点

**这一步必须询问用户，不可跳过询问。**

从 Step 2 提取的"需求文档链接"（custom_field id=5）中获取需求文档内容。如果链接为空，告知用户并询问是否手动提供。

向用户展示选项：

> 📋 需求文档链接: {URL}
>
> 接下来可以帮你生成前端技术方案文档，请选择：
>
> 1. **自动生成** — AI 根据需求文档内容自动生成前端技术方案文档
> 2. **基于模板生成** — 使用前端技术文档模板，AI 填充具体内容
> 3. **跳过此步** — 不生成技术文档，直接进入下一步

#### 选项 1：自动生成

1. 通过钉钉文档 MCP 读取需求文档内容
2. 分析需求，梳理前端改动点
3. 自动生成完整的前端技术方案文档（包含：改动文件列表、数据流设计、组件设计、接口对接、风险点等）
4. 通过钉钉文档 MCP 创建文档并写入

#### 选项 2：基于模板生成

使用以下前端技术文档模板结构：

```markdown
# 前端技术方案文档

## 1. 需求概述

- 需求编号：{ISSUE_ID}
- 需求标题：{ISSUE_SUBJECT}
- Redmine 链接：{SERVER}/issues/{ISSUE_ID}

## 2. 前端改动范围

| 文件路径 | 改动类型       | 说明 |
| -------- | -------------- | ---- |
| ...      | 新增/修改/删除 | ...  |

## 3. 数据流设计

- 请求接口：...
- 数据结构：...
- 状态管理：...

## 4. 组件设计

- 新增组件：...
- 修改组件：...
- 组件交互：...

## 5. 接口对接

| 接口名 | 方法     | 路径 | 说明 |
| ------ | -------- | ---- | ---- |
| ...    | GET/POST | ...  | ...  |

## 6. 风险点与注意事项

- ...

## 7. 测试要点

- ...
```

1. 通过钉钉文档 MCP 读取需求文档内容
2. 按模板结构填充内容
3. 通过钉钉文档 MCP 创建文档并写入

#### 选项 3：跳过

直接进入 Step 5，不生成技术文档。

#### 钉钉文档 MCP 使用方式

使用钉钉文档 MCP 工具创建技术文档：

1. 先用 MCP 工具搜索现有文档或参考模板（如果用户提供模板链接）
2. 用 MCP 工具创建新文档（文档标题格式：`【前端技术方案】{需求标题}`）
3. 用 MCP 工具向文档写入内容
4. 获取文档链接，供后续步骤使用

##### ⚠️ 关键：技术文档必须作为需求文档的子文档创建

技术文档必须放在需求文档下面，作为子文档（sub-document），而不是独立的文档或放到同层文件夹下。

**正确做法：`create_document` 时传入需求文档的 nodeId 作为 `folderId`**

```bash
# 从需求文档链接中提取 nodeId
# 例如链接 https://alidocs.dingtalk.com/i/nodes/R1zknDm0WR3jGgpRix1oq6AxVBQEx5rG
# nodeId = R1zknDm0WR3jGgpRix1oq6AxVBQEx5rG

# 创建文档时，直接传入需求文档的 nodeId 作为 folderId
create_document({
  name: "【前端技术方案】{需求标题}",
  folderId: "{需求文档的nodeId}",   # ← 关键！传入需求文档 nodeId，不是父文件夹 nodeId
  markdown: "{技术方案内容}"
})
```

**❌ 错误做法：先创建再移动**

```bash
# ❌ 不要这样做！move_document 的 targetFolderId 只接受真正的文件夹节点，
#    传入文档 nodeId 会导致文档被回收/丢失
# 1. create_document({...})              # 不传 folderId，文档创建到"我的文档"根目录
# 2. move_document({targetFolderId: 需求文档nodeId})  # ❌ 会失败，文档被回收！

# ❌ 也不要传需求文档的父文件夹 nodeId，那样技术文档会成为需求文档的同层兄弟，而不是子文档
# get_document_info(需求文档) → folderId = "gvNG4YZ7..."  # 这是父文件夹
# create_document({folderId: "gvNG4YZ7..."})  # ❌ 放到了同层文件夹下，不是子文档
```

**钉钉文档 MCP 的 folderId 参数行为对照**：

| 操作              | 传入文档 nodeId         | 传入文件夹 nodeId   | 不传                   |
| ----------------- | ----------------------- | ------------------- | ---------------------- |
| `create_document` | ✅ 创建为该文档的子文档 | ✅ 创建到该文件夹下 | 创建到"我的文档"根目录 |
| `move_document`   | ❌ 文档会被回收/丢失    | ✅ 移动到该文件夹下 | 移动到"我的文档"根目录 |

> **结论**：`create_document` 的 `folderId` 支持传入文档 nodeId（创建子文档），但 `move_document` 的 `targetFolderId` 只支持真正的文件夹节点。因此必须在创建时就指定正确的 `folderId`，不能先创建再移动。

##### MCP 工具调用方式

如果 MCP 工具未作为 Claude 内置工具自动加载（纯 CLI 模式下可能出现），可通过 curl 直接调用 MCP streamable-http 协议：

```bash
MCP_URL="https://mcp-gw.dingtalk.com/server/{server_id}?key={key}"

# 1. 初始化连接
curl -s -X POST -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-03-26","capabilities":{},"clientInfo":{"name":"claude-code","version":"1.0"}}}' \
  "$MCP_URL"

# 2. 获取需求文档内容
curl -s -X POST -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{"name":"get_document_content","arguments":{"nodeId":"{需求文档nodeId}"}}}' \
  "$MCP_URL"

# 3. 创建子文档（关键：folderId = 需求文档 nodeId）
curl -s -X POST -H "Content-Type: application/json" -H "Accept: application/json, text/event-stream" \
  -d "$(jq -n --arg name '{文档标题}' --arg md '{markdown内容}' --arg folderId '{需求文档nodeId}' \
    '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{"name":"create_document","arguments":{"name":$name,"folderId":$folderId,"markdown":$md}}}' )" \
  "$MCP_URL"
```

MCP URL 和 key 从 `~/.claude/settings.json` 的 `mcpServers` 配置中获取。

向用户展示：`✅ 技术文档已创建: {钉钉文档链接}（已作为需求文档的子文档）`

---

### Step 5: 询问是否创建前端子任务 ⭐ 决策点

**必须询问用户，不可自动创建。**

> 📋 接下来可以帮你在 Redmine 上创建前端子任务，请选择：
>
> 1. **创建前端子任务** — 在当前需求下创建"开发任务"子任务，指派给你，并附上技术文档链接
> 2. **跳过此步** — 不创建子任务

如果用户选择跳过，工作流结束，输出最终摘要。

---

### Step 6: 创建 Redmine 前端子任务

通过 Redmine API 创建子任务：

```bash
# 构建创建请求
curl -s -X POST \
  -H "X-Redmine-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "issue": {
      "project_id": {PROJECT_ID},
      "tracker_id": 2,
      "status_id": 1,
      "priority_id": {从父需求继承},
      "subject": "前端开发 - {父需求标题}",
      "description": "前端技术方案文档链接: {技术文档URL}\n\n分支: feature/{USER_LOGIN}/{ISSUE_ID}",
      "parent_issue_id": {ISSUE_ID},
      "assigned_to_id": {USER_ID},
      "custom_fields": [
        {"id": 6, "value": "{技术文档URL}"}
      ]
    }
  }' \
  "${SERVER}/issues.json"
```

关键参数说明：

| 参数               | 值                       | 说明                            |
| ------------------ | ------------------------ | ------------------------------- |
| `tracker_id`       | `2`                      | "开发任务" tracker              |
| `parent_issue_id`  | `{ISSUE_ID}`             | 父需求号                        |
| `assigned_to_id`   | `{USER_ID}`              | 当前用户（从 red-cli 配置获取） |
| `custom_fields[0]` | id=6, value=技术文档 URL | "技术方案文档链接"字段          |

从创建响应中提取新任务的 ID 和 URL。

向用户展示：`✅ 前端子任务已创建: #{NEW_TASK_ID} — {SERVER}/issues/{NEW_TASK_ID}`

---

### Step 7: 将技术文档链接写入父需求（可选）

如果父需求的"技术方案文档链接"字段（custom_field id=6）为空，询问用户是否也将技术文档链接更新到父需求上：

> 📋 父需求的"技术方案文档链接"目前为空，是否将生成的技术文档链接也更新到父需求 #{ISSUE_ID} 上？
>
> 1. **更新** — 将技术文档链接写入父需求
> 2. **不更新** — 只保留在子任务上

如果选择更新：

```bash
curl -s -X PUT \
  -H "X-Redmine-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"issue": {"custom_fields": [{"id": 6, "value": "{技术文档URL}"}]}}' \
  "${SERVER}/issues/${ISSUE_ID}.json"
```

---

### Step 8: 输出最终摘要

汇总所有已完成操作，向用户展示完整摘要：

```
✅ 需求工作流完成!

  需求号:    #{ISSUE_ID}
  标题:      {ISSUE_SUBJECT}
  Redmine:   {SERVER}/issues/{ISSUE_ID}

  ✅ 分支:      feature/{USER_LOGIN}/{ISSUE_ID}
  ✅ GitLab:    https://your-gitlab-domain.com/{GITLAB_PATH}/-/tree/feature/{USER_LOGIN}/{ISSUE_ID}

  ✅ 技术文档:  {钉钉文档链接} (如果生成了)
  ✅ 前端任务:  #{NEW_TASK_ID} — {SERVER}/issues/{NEW_TASK_ID} (如果创建了)
```

---

## 错误处理

| 场景                   | 处理方式                                                                                                                                                    |
| ---------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 需求号不存在           | 提示用户检查需求号，展示 API 返回的错误信息                                                                                                                 |
| 需求文档链接为空       | 询问用户是否手动提供链接，或选择跳过文档生成                                                                                                                |
| GitLab 项目列表为空    | 提示项目未关联 GitLab，可能需要先在 Redmine 上手动创建分支                                                                                                  |
| 分支已存在             | 提示分支已存在，询问是否切换到该分支                                                                                                                        |
| 钉钉文档 MCP 不可用    | 提示 MCP 未连接，询问用户是否手动创建文档后提供链接。若 MCP 工具未自动加载，可通过 curl 直接调用 MCP streamable-http 协议（详见 Step 4 "MCP 工具调用方式"） |
| 钉钉文档子文档创建失败 | `create_document` 的 `folderId` 必须传入需求文档的 nodeId（而非父文件夹 nodeId），不能先创建再移动（`move_document` 不支持将文档移到另一个文档节点下）      |
| 技术文档放到了错误位置 | 检查 `create_document` 的 `folderId` 参数：应该是需求文档的 nodeId，不是父文件夹 nodeId，也不是留空                                                         |
| git 仓库有未提交修改   | 提示用户选择 stash 或取消                                                                                                                                   |
| 创建子任务失败         | 展示 API 返回的错误信息，建议用户手动创建                                                                                                                   |

## 快速参考

### 用户常见输入 → 工作流映射

| 用户输入                          | 触发行为                                 |
| --------------------------------- | ---------------------------------------- |
| `526205`                          | 完整工作流：解析为需求号 → 执行 Step 1-8 |
| `https://your-redmine-domain.com/issues/526205` | 同上，从 URL 提取需求号                  |
| `帮我创建分支 526205`             | 只执行 Step 1-3（创建分支）              |
| `帮我开需求 526205`               | 完整工作流                               |
| `需求 526205 技术文档`            | Step 1-2 + Step 4（只生成技术文档）      |
| `526205 只创建分支`               | 只执行 Step 1-3                          |

### 分支名格式

**固定格式**：`feature/{Redmine_login}/{需求号}`

示例：`feature/albert/526205`

用户名来源：Redmine 用户 login 名（从 `~/.red/config.json` 的 `user-id` → Redmine API → `.user.login`）

---

## 伴生工具

本 skill 目录下包含 `red-branch` 脚本，也可独立使用：

```bash
# 脚本位置
~/.local/bin/red-branch          # 全局可用（PATH 中）
~/.claude/skills/redmine-workflow/red-branch  # skill 目录备份

# 独立使用方式
red-branch 526205                  # 交互式
red-branch 526205 --project edu-ae # 指定项目
red-branch 526205 --list           # 列出项目
red-branch 526205 --dry-run        # 预览
red-branch 526205 --local-only     # 仅本地
red-branch --help                  # 帮助
```
