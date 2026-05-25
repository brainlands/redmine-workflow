# Redmine Workflow Skill

基于 `red-cli` + `githook` API + 钉钉文档 MCP 的完整需求开发工作流自动化工具。

## 功能特性

一条命令启动完整需求工作流：

- ✅ 自动创建 Git 分支
- ✅ 分析需求文档生成前端技术文档
- ✅ 可选创建前端子任务并指派给当前用户
- ✅ 自动关联技术文档链接

## 前置依赖

| 依赖 | 安装方式 | 作用 |
|------|----------|------|
| `red-cli` | `brew install red-cli` | Redmine CLI 工具 |
| `curl`, `jq` | macOS 自带 / `brew install jq` | API 调用和 JSON 解析 |
| `git` | macOS 自带 | 本地分支操作 |
| `red-branch` 脚本 | 位于本仓库 | 分支创建自动化 |
| 钉钉文档 MCP | 已配置 | 创建/写入钉钉技术文档 |

## 快速开始

### 1. 安装 red-cli

```bash
brew install red-cli
```

### 2. 配置 Redmine 认证

```bash
red auth login
```

按提示输入：
- Redmine 服务器地址（如 `https://your-redmine-domain.com`）
- 你的 API Key（在 Redmine 的 `/my/account` 页面获取）

### 3. 配置 ~/.red/config.json

确保配置文件包含以下字段：

```json
{
  "server": "https://your-redmine-domain.com",
  "api-key": "your-api-key-here",
  "user-id": "your-user-id"
}
```

## 使用方法

### 基础用法

```bash
# 交互式创建分支
red-branch 526205

# 指定项目创建分支
red-branch 526205 --project edu-ae

# 查看关联的项目列表
red-branch 526205 --list

# 预览操作（不实际执行）
red-branch 526205 --dry-run

# 只创建本地分支
red-branch 526205 --local-only
```

### 完整工作流（AI 助手自动执行）

当你向 AI 助手提供 Redmine 需求链接或需求号时，会自动触发以下流程：

1. **解析需求** - 从 URL 或需求号获取需求信息
2. **创建分支** - 自动创建 `feature/{username}/{issue_id}` 分支
3. **生成技术文档** - 分析需求文档，生成前端技术方案（可选）
4. **创建子任务** - 在 Redmine 上创建前端开发子任务（可选）

#### 示例输入

```
# 方式 1：直接输入需求号
526205

# 方式 2：粘贴 Redmine 链接
https://your-redmine-domain.com/issues/526205

# 方式 3：自然语言
"帮我创建分支 526205"
"帮我开需求 526205"
"需求 526205 技术文档"
```

## 分支命名规范

**格式**：`feature/{Redmine_login}/{需求号}`

**示例**：`feature/albert/526205`

## 配置文件说明

### ~/.red/config.json

```json
{
  "server": "https://your-redmine-domain.com",
  "api-key": "your-redmine-api-key",
  "user-id": "your-redmine-user-id"
}
```

### Redmine 自定义字段

| 字段 ID | 名称 | 用途 |
|---------|------|------|
| 5 | 需求文档链接 | 产品需求文档 URL |
| 6 | 技术方案文档链接 | 前端技术方案文档 URL |

### Tracker ID

| ID | 名称 | 用途 |
|----|------|------|
| 2 | 开发任务 | 创建前端子任务时使用 |

## 技术文档生成

工作流支持自动生成前端技术文档，包含：

1. **需求概述** - 需求编号、标题、链接
2. **前端改动范围** - 文件路径、改动类型、说明
3. **数据流设计** - 请求接口、数据结构、状态管理
4. **组件设计** - 新增/修改组件、组件交互
5. **接口对接** - 接口列表、方法、路径
6. **风险点与注意事项**
7. **测试要点**

技术文档会自动创建为需求文档的子文档（钉钉文档）。

## API 端点

### Redmine API

| 用途 | 方法 | URL |
|------|------|-----|
| 查询需求 | GET | `{SERVER}/issues/{ID}.json` |
| 查询用户 | GET | `{SERVER}/users/{USER_ID}.json` |
| 查询项目 | GET | `{SERVER}/projects/{PROJECT_ID}.json` |
| 创建子任务 | POST | `{SERVER}/issues.json` |

### jojogithook API

| 用途 | 方法 | URL |
|------|------|-----|
| 获取 GitLab 项目 | GET | `https://your-githook-domain.com/api/v1/getuserproject?username={login}&identifier={identifier}` |
| 创建分支 | POST | `https://your-githook-domain.com/api/v1/newbranch?id={gitlab_project_id}&branch={branch_name}` |

## 错误处理

| 场景 | 处理方式 |
|------|----------|
| 需求号不存在 | 提示检查需求号，展示 API 错误信息 |
| 需求文档链接为空 | 询问用户是否手动提供链接 |
| GitLab 项目列表为空 | 提示项目未关联 GitLab |
| 分支已存在 | 提示分支已存在，询问是否切换 |
| 钉钉文档 MCP 不可用 | 提示手动创建文档 |
| git 仓库有未提交修改 | 提示 stash 或取消 |

## 常见问题

### Q: 如何获取 Redmine API Key？

1. 登录 Redmine
2. 访问 `/my/account`
3. 找到 "API access key" 区域
4. 点击 "Show" 显示密钥并复制

### Q: 分支创建失败怎么办？

1. 检查 red-cli 配置是否正确
2. 确认项目在 GitLab 上有关联仓库
3. 使用 `--dry-run` 模式预览操作
4. 查看错误信息并排查

### Q: 技术文档生成失败？

1. 确认钉钉文档 MCP 已正确配置
2. 检查需求文档链接是否可访问
3. 可以手动创建文档后提供链接

## 项目结构

```
redmine-workflow/
├── SKILL.md          # Skill 定义文档
├── red-branch        # 分支创建自动化脚本
├── .gitignore        # Git 忽略配置
└── README.md         # 使用说明
```

## 许可证

MIT License

## 贡献

欢迎提交 Issue 和 Pull Request！

## 联系方式

- 作者：albert
- 邮箱：your-email@example.com
