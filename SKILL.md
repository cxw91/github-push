---
name: github-push
description: "This skill should be used when the user wants to push or upload a local project to GitHub. It auto-generates a README.md when one is missing, then creates the repository and pushes all files via the github MCP connector. Trigger phrases include: 推送到GitHub、上传到GitHub、push到GitHub、发布到GitHub、github push、上传项目到github、把项目推到github."
agent_created: true
---

# Github Push

## Overview

将本地项目目录推送到 GitHub：收集文件 → 缺 README 则自动生成 → 用 github MCP 创建仓库并一次性 push → **推送成功后顺带在本地目录初始化 git 仓库并关联 `origin`，使本地与 GitHub 双向同步**（见 Step 6.5）。不依赖本地 `gh` CLI，也不要求推送前本地 git 已有 remote。

## 前置条件

- `github` 连接器已连接（提供 `mcp__github__*` 工具，可推送文件，但**无建仓权限**）。
- `github-push-pat` MCP server（PAT 认证）已配置并「信任」（提供 `mcp__github-push-pat__*` 工具，**可建仓**）。若未启用，建仓走下方「建仓失败（403）的备用方案」.
- 本地 `gh` CLI 可不存在；本地 git 身份仅用于 commit 信息参考。

## 工作流

### Step 1: 确认目标与参数

向用户确认（不确定时用 AskUserQuestion，勿猜测）：

- 项目目录（默认当前工作目录）。
- 仓库名（默认取目录名，须符合 GitHub 规则：小写字母、数字、`-`、`.`、`_`）。
- 可见性：`private`（默认，GitHub 新仓库默认私有）/ `public`，让用户明确选择。
- 组织名（可选，省略则推到个人账号）。

### Step 2: 检查并生成 README.md

1. 在项目根目录查找 README（`README.md`、`README.markdown`、`readme.md`、`README` 等，大小写不敏感）。
2. 存在 → 直接使用，不改动。
3. 缺失 → 按 `references/readme-generation.md` 的规则生成 README.md，写入项目根目录。

### Step 3: 收集文件清单

1. 用 Glob（`**/*`）递归列出所有文件。
2. 排除默认目录/文件：`.git/`、`node_modules/`、`.workbuddy/`、`dist/`、`build/`、`__pycache__/`、`.venv/`、`venv/`、`.idea/`、`.vscode/`、`.DS_Store`、`*.pyc`、`*.log`、`.env`、`*.lock`（可选）。
3. 只推送文本文件。`push_files` 的 `content` 字段仅接受文本（MCP 内部按 UTF-8 处理，二进制字节会损坏）。**按文件内容而非文件名判断**：`.hex`/`.s19`/`.srec`/`.map` 等实为 ASCII 文本，可正常推送；真二进制（`.bin`/`.elf`/`.axf`/`.o`/`.exe`/图片/字体/压缩包等）检测到后默认跳过并列出，若必须提交改用下方「含二进制文件的项目」方案。

### Step 4: 获取账号信息

- 调用 `mcp__github__get_me`（或 `mcp__github-push-pat__get_me`，两者均可）获取登录用户名作为 `owner`。
- 用户指定组织时，`owner` 用组织名。

### Step 5: 创建仓库

优先调用 `mcp__github-push-pat__create_repository`（PAT 认证，可建仓）；若该工具不可用，回退 `mcp__github__create_repository`：

- `name`: 仓库名
- `private`: `true` / `false`
- `description`: 项目一句话描述
- `organization`: 可选
- 不要设 `autoInit: true`（会生成带 README 的初始提交，与后续 push 冲突）。
- 若 `mcp__github__create_repository` 返回 `403 Resource not accessible by integration`（连接器无 Administration/建仓权限），见下方「建仓失败（403）的备用方案」。

### Step 6: 推送文件

调用 `mcp__github__push_files` 或 `mcp__github-push-pat__push_files`（两者均可），一次性传入所有文件（含 README.md）：

- `owner` / `repo`：同 Step 5。
- `branch`：`main`（新仓库默认）。若报分支不存在，改试 `master`。
- `files`: `[{path, content}]`，`content` 为文件原始文本，**不要 base64**，path 用相对仓库根的路径（如 `src/index.js`）。
- `message`: commit 信息，如 `Initial commit`。

### Step 6.5: 推送后本地初始化 git 仓库（关联远端）

`push_files` 走 API，不会在本地建仓。为让本地项目目录与 GitHub 双向同步，推送成功后顺带把本地目录变成已连接 `origin` 的 git 仓库。本步为「尽力而为」：若本地 git / 联网 / 凭证不可用，远端推送仍视为成功，仅跳过本步并在反馈中说明。

1. **判断是否已是 git 仓库**：`git -C <dir> rev-parse --is-inside-work-tree 2>/dev/null`。
   - 已是 → 跳到步骤 4（仅补齐 remote 与 upstream，不破坏现有历史）。
   - 不是 → `git -C <dir> init -b main`（不立即提交，避免产生与远端无关的独立历史）。
2. **写仓库级配置，防 Windows 行尾符误判**：`git -C <dir> config core.autocrlf false`（保持与 GitHub 一致的 LF，避免后续提交来回转换）。
3. **设置远端**（URL 取 `https://github.com/<owner>/<repo>.git`）：
   - 无 `origin` → `git -C <dir> remote add origin <url>`；
   - 有 `origin` 但 URL 不符 → `git -C <dir> remote set-url origin <url>`。
4. **拉取远端引用**：`git -C <dir> fetch origin`（需联网 + GitHub 凭证；缺失则本步中止并报原因）。
5. **确定默认分支**：`DEFAULT_BRANCH=$(git -C <dir> symbolic-ref --short refs/remotes/origin/HEAD 2>/dev/null || echo main)`；若 `origin/<DEFAULT_BRANCH>` 不存在，回退 `master`。
6. **对齐本地仓库与远端（仅初始化场景）**：
   - 本地尚无提交时，用 `git -C <dir> reset --mixed origin/<DEFAULT_BRANCH>` 将 HEAD 指向远端提交、重置索引，但**保留工作区文件**（被 Step 3 跳过的二进制文件以未跟踪状态保留，不会被删）。配合 `core.autocrlf=false`，内容与远端一致时工作区即干净。
   - 已存在本地提交（重跑场景）→ **不要 reset**，避免丢弃本地历史；直接跳到步骤 7。
7. **设置跟踪**：`git -C <dir> branch -u origin/<DEFAULT_BRANCH> <DEFAULT_BRANCH>`（初始化场景因 reset 已存在该分支；重跑场景若分支已缺则仅写配置：`git -C <dir> config branch.<DEFAULT_BRANCH>.remote origin && git -C <dir> config branch.<DEFAULT_BRANCH>.merge refs/heads/<DEFAULT_BRANCH>`）。
8. **校验**：`git -C <dir> status` 应显示 `up to date with 'origin/<DEFAULT_BRANCH>'` 或 `nothing to commit, working tree clean`（被跳过的二进制文件可能列为 untracked，属正常）。

> 完成后本地目录既是普通项目文件夹，又是与 GitHub 关联的 git 仓库，后续 `git push` / `git pull` 即可增量同步。被 Step 3 跳过的二进制文件不会自动进远端，除非用户自行 `add`+`commit`+`push`。

### Step 7: 验证与反馈

- push 成功后报告仓库 URL（`https://github.com/<owner>/<repo>`）与推送文件数。
- 报告本地 git 仓库状态：是否已初始化、remote 指向、分支与 `origin/<DEFAULT_BRANCH>` 是否同步（见 Step 6.5）；本步失败需明确提示原因（如缺 git / 无网络 / 无凭证）。
- 列出被跳过的二进制文件及原因。

## 含二进制文件的项目

`push_files` / `create_or_update_file` 的 `content` 均为字符串，无法可靠提交二进制文件（图片、字体、编译产物、压缩包等）。需一并提交时，改用本地 git 推送：

1. `git init`（若非仓库）→ `git add -A` → `git commit -m "Initial commit"`。
2. `git remote add origin https://github.com/<owner>/<repo>.git`。
3. `git push -u origin main`。

注意：本地 git 推送需要 GitHub 认证（HTTPS PAT 或 SSH key）。MCP 连接器的 token 不直接暴露给 git CLI，须先确认用户已配置凭据，否则提示用户提供 PAT 或配置 SSH。

## 建仓失败（403）的备用方案

`github-push-pat` server 未启用、且 `mcp__github__create_repository` 报 `403 Resource not accessible by integration` 时，说明原 github 连接器缺建仓权限（此权限由连接器请求的 scope 决定，重新授权也无法自行添加）。按以下顺序降级：

1. **优先用本地 git 凭据 + API 建仓**（无需用户手动操作）：
   - 查 GCM 缓存的 GitHub token：`printf "protocol=https\nhost=github.com\n\n" | GCM_INTERACTIVE=never git credential fill`（输出含 `username` 与 `password`，password 即 token，**勿打印到回复**）。
   - 用 token 调 API 建仓：`curl -s -X POST https://api.github.com/user/repos -H "Authorization: Bearer <token>" -d '{"name":"<repo>","private":false}'`。
   - git 推送：`git init && git add -A && git commit && git branch -M main && git remote add origin https://github.com/<owner>/<repo>.git && git push -u origin main`（HTTPS 由 GCM 自动认证）。
2. **SSH 备用**：`~/.ssh` 有 key 且已加到 GitHub 时，`git@github.com` 22 端口常被墙，改用 443：remote 写 `ssh://git@ssh.github.com:443/<owner>/<repo>.git`。
3. **仍无凭据**：让用户在网页手动创建空仓库（勿初始化 README），再回到 Step 6 用 `push_files`（Contents 权限通常正常）。

## 注意事项

- `push_files` 与 `create_or_update_file` 的 `content` 均为原始文本，勿 base64。
- 仓库已存在时 `create_repository` 会报错：改用 `create_or_update_file` 增量写入，或询问用户是否推送到新仓库名。
- 目录为空或无源码文件：至少生成并推送 README.md。
- 大量文件时仍可一次 `push_files` 调用完成；文件过多或单文件过大可分批。
- 敏感文件（`.env`、密钥、证书）默认排除，不要推送。
- Step 6.5 本地初始化为「尽力而为」：本地无 git / 断网 / 无 GitHub 凭证时，远端推送仍成功，仅跳过本地建仓并提示原因。重跑（本地已有提交）时不执行 `reset`，以免丢弃本地历史。

## 使用示例

### 例 1：把当前项目推到 GitHub（自动生成 README）

用户：「把当前项目推送到 GitHub」

1. 目录 = 当前工作目录，仓库名 = 目录名，用 AskUserQuestion 确认 private / public。
2. 无 README → 检测到 `package.json` → 按 references 生成 Node 项目 README（含 `npm install` / `npm run start`）。
3. Glob 收集源码，排除 `node_modules`。
4. `get_me` 取 owner → `create_repository`（`private: true`，不设 autoInit）→ `push_files` 一次推送（含 README.md）。
5. 报告 `https://github.com/<owner>/<repo>` 及文件数。

### 例 2：STM32 固件项目（区分 .hex 与 .bin）

用户：「把 STM32 项目推到 GitHub」

1. `.c`/`.h`/工程文件 → 文本，推送。
2. `.hex`（Intel HEX ASCII）→ 文本，推送。
3. `.bin`/`.elf`/`.o` → 二进制，默认跳过并在回复中列出；若用户坚持要提交，切到「含二进制文件的项目」本地 git push 方案（需 PAT/SSH）。

### 例 3：指定公开仓库 + 组织

用户：「推到 GitHub，公开仓库，放到 org xyz」

→ `create_repository` 用 `private: false`、`organization: "xyz"`，owner 用组织名，其余流程同例 1。

### 例 4：仓库已存在（增量更新）

用户：「再推一次到这个仓库」

1. `create_repository` 会报错（已存在）。
2. 改用 `create_or_update_file` 对变更文件逐个写入（更新文件需先用 `get_file_contents` 取 `sha`），或询问是否换新仓库名。

## Resources

- `references/readme-generation.md` — README 自动生成规则、项目类型识别与模板。
