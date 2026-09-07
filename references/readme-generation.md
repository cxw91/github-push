# README 自动生成规则

当项目根目录缺少 README 时，按以下规则生成。目标：结构清晰、信息真实、不虚构功能。

## 1. 项目名

优先级依次取：

1. `package.json` 的 `name`
2. `pyproject.toml` / `setup.py` / `setup.cfg` 的 `name`
3. `go.mod` 的 module 最后一段
4. `Cargo.toml` 的 `[package] name`
5. 目录名（转为可读形式，如 `my-app` → `my-app`）

## 2. 一句话描述

优先级：

1. `package.json` 的 `description`
2. 代码/docstring 中的模块说明
3. 根据文件内容推断项目用途

## 3. 技术栈识别

| 标志文件 | 类型 |
|---------|------|
| `package.json` | Node.js / 前端 |
| `requirements.txt`、`pyproject.toml`、`setup.py` | Python |
| `go.mod` | Go |
| `Cargo.toml` | Rust |
| `pom.xml` | Java / Maven |
| `*.csproj`、`*.sln` | .NET |
| `Gemfile` | Ruby |
| `composer.json` | PHP |
| 仅 `*.html`/`*.css`/`*.js` | 静态前端 |
| 无任何标志 | 通用 |

从对应清单提取依赖列表（如 `package.json` 的 `dependencies`、`requirements.txt` 行）填入「技术栈」。

## 4. 通用模板

```markdown
# {项目名}

> {一句话描述}

## 简介

{2-3 句说明项目是什么、解决什么问题}

## 功能特性

- {从代码/结构归纳的真实特性}

## 技术栈

- {语言/框架/依赖}

## 快速开始

### 环境要求

- {Node/Python/Go 等版本，可从配置文件或锁文件推断，不确定则写「建议最新 LTS」}

### 安装

{按项目类型给出对应命令}

### 使用

{启动/运行命令，优先从 package.json scripts、README 已有片段、入口文件推断}

## 项目结构

```text
{关键目录与文件，用注释说明用途}
```

## License

{仅当项目内有 LICENSE 文件时填写；否则省略或写「未指定」}
```

## 5. 各类型安装/使用命令

- **Node.js**：安装 `npm install`；使用取 `package.json` 的 `scripts.start` / `scripts.dev`，无则 `node <入口>`。
- **Python**：安装 `pip install -r requirements.txt`；使用 `python <入口>.py` 或 `scripts` 中的命令。
- **Go**：安装 `go mod download`；使用 `go run .` / `go build`。
- **Rust**：`cargo build --release`；使用 `cargo run`。
- **静态前端**：使用「直接用浏览器打开 `index.html`」，或 `npx serve`（有 package.json 时）。
- **通用/未知**：安装与使用写占位说明，不编造具体命令。

## 6. 原则

- 只写能确定的事实；无法确定的字段省略或写通用占位，不虚构。
- 功能特性从实际代码归纳，不写「高性能」「易扩展」等空话。
- 若项目已有部分说明材料（如旧 README 片段、注释），优先复用原文。
- 输出 Markdown，UTF-8 编码，换行 LF。
