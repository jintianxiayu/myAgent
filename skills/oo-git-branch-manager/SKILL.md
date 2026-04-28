---
name: oo-git-branch-manager
description: |
  Git 分支管理技能。当用户提到"创建功能分支"、"创建分支"、"feature分支"、"功能分支已完成创建PR"、"创建PR"时自动触发。支持从auto_deploy_production/master切出feature分支，以及向preview分支创建PR。团队使用Bitbucket作为Git仓库。
  注意：这个技能只处理分支创建和PR创建，不处理其他git操作。
---

# Git Branch Manager

本技能用于OO团队的 Git 分支规范，基于 Bitbucket 仓库。

## 仓库配置

在开始之前，需要配置以下环境变量：

- `BITBUCKET_API_TOKEN`: Bitbucket API Token（个人访问令牌）
- `BITBUCKET_USERNAME`: Bitbucket 用户名(邮箱)

## 分支命名规范

**被管理的分支类型：**
- `auto_deploy_dev` - Dev 环境部署分支
- `auto_deploy_qa` - QA 环境部署分支
- `auto_deploy_production` - Production 环境部署分支
- `preview/v{a.b.c.d}` - 预览分支，a.b.c.d 为版本号，d 可选
- `release/v{a.b.c.d}` - 发布分支

**Feature 分支格式：**
```
feature/v{version}-{developer_name}_{jira_id}_{feature_description}
```

其中：
- `version`：从源分支（auto_deploy_production 或 master）的 package.json 推导
- `developer_name`：从 git config `user.name` 获取
- `jira_id`：可选（缺失时分支名中该部分省略）
- `feature_description`：功能点简要描述，需转为英文

## 版本号推导规则

当源版本号为 `X.Y.Z.W` 时：
- 如果 W 存在，新版本号为 `X.Y.(Z+1)`
- 如果 W 不存在（格式为 `X.Y.Z`），新版本号仍为 `X.Y.(Z+1)`

**示例：**
- `1.2.3.1` → `1.2.4`
- `1.2.3` → `1.2.4`

## 工作流程

### 流程一：创建 Feature 分支

当用户请求创建功能分支时，执行以下步骤：

**步骤 1：确定目标仓库**

请求用户指定要操作的仓库路径。检查该路径是否为有效的 git 仓库。

**步骤 2：确定源分支**

按以下优先级确定源分支：
1. `auto_deploy_production`
2. `master`

如果两者都不存在，提示用户无法创建分支。

**步骤 3：获取版本号**

从源分支的 package.json 中读取 version 字段，按照版本推导规则生成新版本号。

**步骤 4：获取开发者名称**

从 git config 获取 user.name 作为 developer_name。

**步骤 5：收集分支信息**

请求用户提供以下信息（按需）：
- `jira_id`：如果用户未提供，则从分支名中省略
- `feature_description`：功能点简要描述

**步骤 6：构建分支名**

按格式构建：`feature/v{version}-{developer_name}_{jira_id}_{feature_description}`

- 如果提供了 jira_id：`feature/v1.2.4-zhangsan_PROJ-123_add_login`
- 如果未提供 jira_id：`feature/v1.2.4-zhangsan_add_login`

**步骤 7：创建分支**

```bash
git checkout -b {branch_name} {source_branch}
```

显示创建结果给用户。

---

### 流程二：创建 PR

当用户请求为已完成的功能分支创建 PR 时，执行以下步骤：

**步骤 1：确定仓库**

请求用户指定仓库路径。验证远程仓库配置是否指向 Bitbucket。

**步骤 2：确定源分支和目标分支**

1. 获取当前分支或用户指定的源分支
2. 推导目标 preview 分支：
   - 从源分支提取版本号（例如 `feature/v1.2.4-zhangsan_PROJ-123_add_login` → `1.2.4`）
   - 目标分支为 `preview/v1.2.4`
3. 向用户确认目标分支是否正确

**步骤 3：获取远程仓库信息**

从 git remote 获取仓库 URL，提取 workspace 和仓库 slug。

**步骤 4：创建 PR**

使用 Bitbucket REST API 创建 PR：

```bash
curl -X POST "https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests" \
  -H "Authorization: Bearer {BITBUCKET_API_TOKEN}" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "{PR标题}",
    "source": {
      "branch": {
        "name": "{源分支名}"
      }
    },
    "destination": {
      "branch": {
        "name": "{目标分支名}"
      }
    },
    "close_source_branch": false
  }'
```

**步骤 5：返回 PR 链接**

显示创建的 PR 链接给用户。

---

## 错误处理

| 场景 | 处理方式 |
|------|----------|
| 仓库路径无效 | 提示用户检查路径 |
| 未找到源分支 | 提示用户确认仓库状态 |
| package.json 不存在或无 version 字段 | 提示用户无法推导版本号 |
| git config 缺少 user.name | 提示用户配置 git 用户名 |
| Bitbucket API 调用失败 | 显示错误信息，提示检查 token 权限 |
| 目标分支不存在 | 提示用户检查分支名是否正确 |

---

## 输出格式

### 创建分支成功

```
✓ 分支已创建

分支名：feature/v1.2.4-zhangsan_PROJ-123_add_login
源分支：auto_deploy_production
版本：1.2.4
开发者：zhangsan
JIRA：PROJ-123
描述：add_login

当前已切换到新分支。
```

### 创建 PR 成功

```
✓ PR 已创建

标题：feat: 添加登录功能
源分支：feature/v1.2.4-zhangsan_PROJ-123_add_login
目标分支：preview/v1.2.4
PR 链接：https://bitbucket.org/{workspace}/{repo}/pull-requests/{id}
```

---

## 限制与注意事项

1. **仅支持本地分支创建**：不自动推送分支到远程
2. **需要手动配置**：BITBUCKET_API_TOKEN 和 BITBUCKET_USERNAME 需提前设置
3. **PR 标题**：需要用户确认或提供自动生成的标题
4. **分支存在性检查**：创建分支前应检查分支是否已存在
5. **所有操作需要用户确认后再执行**
