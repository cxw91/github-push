# github-push

> 将本地项目推送到 GitHub 的 WorkBuddy 技能，缺少 README 时自动生成。

## 简介

`github-push` 是一个 WorkBuddy 技能，用于把本地项目目录推送到 GitHub 仓库。它通过 github MCP 连接器创建仓库并一次性推送全部文件，若项目缺少 README.md 则自动生成，不依赖本地 `gh` CLI。

## 功能特性

- 自动生成 README.md（识别 Node / Python / Go / Rust / Java / 静态前端等项目类型）
- 通过 github MCP 创建仓库并批量推送文件
- 排除 `node_modules` / `.git` / `.env` 等冗余与敏感文件
- 区分文本与二进制文件，含二进制时切换本地 git 推送方案

## 快速开始

### 环境要求

- WorkBuddy，且 `github` 连接器已连接

### 使用

在对话中说「推送到 GitHub」「上传到 GitHub」「push 到 GitHub」等即可触发。

## 项目结构

```text
github-push/
├── SKILL.md                        # 技能定义与工作流
└── references/
    └── readme-generation.md        # README 自动生成规则与模板
```
