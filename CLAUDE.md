# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 Claude Code 插件仓库 (AI DevKit)，包含可在对话中调用的技能定义。插件配置在 `.claude-plugin/plugin.json` 中，技能注册在 `openspec/config.yaml`。

## 技能定义

以下技能可用：

- **code-philosophy** (`skills/code-philosophy/SKILL.md`): 迭代开发哲学，强调：
  - 先构建简单方案，根据反馈优化
  - 保持函数专注和可组合
  - 最小化接口，灵活实现
  - 小而可逆的决策
- **git-commit** (`skills/git-commit/SKILL.md`): 编写有效 git 提交信息的指南，使用祈使语气
- **typescript-style** (`skills/typescript-style/SKILL.md`): TypeScript 最佳实践，涵盖命名、类型、异步模式、错误处理和代码组织
- **karpathy-guidelines** (`skills/karpathy-guidelines/SKILL.md`): AI/ML 开发哲学，采用系统化、科学的方法

## 开发命令

```bash
# 添加marketplace
/plugin marketplace add jintianxiyu/myAgent

# 安装插件
/plugin install bics-ai-devkit@bics-ai-devkit
```

## 添加新技能

1. 在 `skills/` 目录下创建 `<skill-name>/SKILL.md`
2. 在 `openspec/config.yaml` 中注册技能
3. 在 `.claude-plugin/plugin.json` 中添加命令映射（如需要）

## 架构

```
.claude-plugin/          # 插件元数据和市场配置
openspec/     # 技能注册表 - 定义可用技能及其命令
skills/                  # 单个技能定义文件 (SKILL.md 格式)
```

## 注意事项

- 技能通过 `Skill("skill-name")` 调用，或使用斜杠命令 `/skill-name`
- 插件配置在 `.claude-plugin/plugin.json`，技能注册在 `openspec/config.yaml`
- 本地修改技能后需要重新安装插件才能生效

## 调用技能

在对话中使用 `Skill` 工具调用技能。例如：

- `Skill("code-philosophy")` - 获取迭代开发方法指导
- `Skill("git-commit")` - 获取提交信息指南
- `Skill("typescript-style")` - 获取 TypeScript 最佳实践
- `Skill("karpathy-guidelines")` - 获取 AI/ML 开发指导

技能可通过斜杠命令激活（如 `/code-philosophy`），或在对话相关时自动触发。