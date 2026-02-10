---
name: mise-clean-dev-workflow
description: Use when setting up development environments on macOS with system cleanliness requirements, avoiding system-level pollution from language runtimes and development tools
---

# Mise 洁癖开发工作流

## Overview

使用 mise 作为单一入口管理所有开发工具和运行时，实现 macOS 系统的零污染开发环境。核心原则：**系统只保留 mise 和 git，其他一切通过 mise 管理**。

## When to Use

- 需要在 macOS 上管理多个项目的开发环境
- 不希望 Homebrew 安装大量语言运行时（node/python/go 等）
- 需要为不同项目使用不同版本的工具
- 希望删除项目时连带清理所有相关依赖
- 团队需要统一的开发环境配置

**When NOT to use:**
- 只需要单一版本的工具且永不切换
- 系统已经完全配置好且无需隔离

## Core Pattern

### 传统方式（污染系统）

```bash
# 系统级安装，难以清理
brew install node python go rust
npm install -g typescript eslint
pip install black poetry
cargo install ripgrep fd-find

# 版本冲突时很痛苦
brew unlink node@18 && brew link node@20
```

### Mise 方式（完全隔离）

```bash
# 系统只装 mise
brew install mise git

# 所有工具通过 mise 管理
mise use -g node@22
mise use -g python@3.12
mise use -g "cargo:ripgrep"

# 项目级隔离
cd project-a && mise use node@20
cd project-b && mise use node@22
```

## Quick Reference

| 操作 | 命令 |
|------|------|
| 安装全局工具 | `mise use -g node@22` |
| 安装项目工具 | `mise use python@3.12` |
| 安装所有配置工具 | `mise install` |
| 查看已安装 | `mise ls` |
| 清理未使用版本 | `mise prune` |
| 查看任务 | `mise tasks ls` |
| 运行任务 | `mise run <task>` |
| 信任项目配置 | `mise trust` |
| 诊断问题 | `mise doctor` |

## Implementation

### 1. Shell 配置（双模式）

```bash
# ~/.zprofile - 非交互式（IDE、脚本）
eval "$(mise activate zsh --shims)"

# ~/.zshrc - 交互式终端
eval "$(mise activate zsh)"
```

**shims vs PATH 区别：**

| 特性 | shims 模式 | PATH 模式 |
|------|-----------|-----------|
| 适用场景 | IDE、脚本、CI | 交互式终端 |
| 版本切换 | 调用时解析 | 进入目录自动切换 |
| 环境变量 | 仅对 mise 工具生效 | 全局生效 |
| hooks | 不触发 | 自动触发 |

### 2. 全局配置

```toml
# ~/.config/mise/config.toml
[tools]
node = "22"
python = "3.12"
"npm:prettier" = "latest"
"cargo:fd-find" = "latest"

[settings]
disable_backends = ["asdf", "vfox"]
auto_install = true
```

### 3. 项目配置

```toml
# mise.toml
min_version = "2024.11.1"

[tools]
node = "20"
python = { version = "3.12", virtualenv = ".venv" }
"aqua:hashicorp/terraform" = "1.5.0"

[env]
NODE_ENV = "development"
DATABASE_URL = { required = "请在 mise.local.toml 中配置" }

# 从 .env 加载
_.file = ".env"

# 添加项目 bin 到 PATH
_.path = ["./node_modules/.bin", "./scripts"]

[tasks]
dev = "npm run dev"
build = "npm run build"

[tasks.test]
description = "运行测试"
run = "npm test"
depends = ["build"]

[hooks]
enter = "echo '项目环境已就绪'"
postinstall = "npm install"
```

### 4. 本地覆盖（不提交）

```toml
# mise.local.toml - 加入 .gitignore
[env]
DATABASE_URL = "postgres://localhost:5432/mydb"
AWS_PROFILE = "personal"
```

### 5. 后端选择

```toml
[tools]
# 核心语言（内置，最快）
node = "22"
python = "3.12"
go = "latest"

# CLI 工具（aqua 有签名验证）
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:sharkdp/fd" = "latest"
"aqua:junegunn/fzf" = "latest"

# GitHub Release
"github:astral-sh/uv" = "latest"

# 语言生态工具
"npm:eslint" = "latest"
"pipx:black" = "latest"
"cargo:tokei" = "latest"
```

## Tasks 系统

### TOML 任务

```toml
[tasks]
# 简单命令
clean = "rm -rf dist node_modules"

# 带依赖
test = { run = "npm test", depends = ["build"] }

# 多命令
lint = ["eslint --fix src/", "prettier --write src/"]

# 完整配置
[tasks.deploy]
description = "部署到生产"
run = "kubectl apply -f k8s/"
confirm = "确定部署到生产吗？"
env = { NODE_ENV = "production" }
depends = ["build", "test"]
```

### 文件任务

```
mise-tasks/
├── setup           # 项目初始化
├── db/
│   ├── migrate     # 数据库迁移
│   └── seed        # 填充数据
└── docker/
    ├── up
    └── down
```

```bash
#!/usr/bin/env bash
# mise-tasks/setup
#MISE description="初始化项目"
#MISE depends=["install"]

set -euo pipefail
npm install
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 增量执行

```toml
[tasks.build]
run = "npm run build"
sources = ["src/**/*.ts", "package.json"]
outputs = ["dist/**"]
```

配合 `mise watch build` 实现文件监听自动构建。

## 完整工作流

### 克隆新项目

```bash
cd ~/projects/oss
git clone https://github.com/org/project.git
cd project

# 信任并安装
mise trust
mise install

# 查看可用任务
mise tasks ls

# 启动开发
mise run dev
```

### 创建新项目

```bash
mkdir my-project && cd my-project

# 初始化 mise
mise use node@20
mise use python@3.12

# 定义任务
cat > mise.toml << 'EOF'
[tools]
node = "20"

[tasks]
dev = "npm run dev"
build = "npm run build"
test = "npm test"
setup = "npm install"
EOF

mise run setup
```

## 环境管理

```bash
# 多环境配置
mise.toml                    # 基础
mise.development.toml        # 开发覆盖
mise.production.toml         # 生产覆盖
mise.local.toml              # 本地覆盖（gitignore）
```

```toml
# mise.development.toml
[env]
NODE_ENV = "development"
API_URL = "http://localhost:3000"
```

```bash
# 切换环境
MISE_ENV=production mise run deploy
mise -E staging run test
```

## 污染地图

| 路径 | 内容 | 清理方式 |
|------|------|---------|
| `~/.local/share/mise/installs/` | 所有工具运行时 | `mise prune` 或删目录 |
| `~/.local/share/mise/shims/` | shim 符号链接 | `mise reshim` 自动管理 |
| `~/.local/share/mise/cache/` | 下载缓存 | `mise cache clear` |
| `~/.config/mise/config.toml` | 全局配置 | 手动编辑 |
| 项目 `mise.toml` | 项目配置 | 跟着项目走 |
| 项目 `node_modules/`、`.venv/` | 项目依赖 | 删项目目录即清理 |

## Common Mistakes

| 错误 | 后果 | 修复 |
|------|------|------|
| 用 brew 装 node/python | 版本冲突，难以清理 | 卸载，改用 `mise use -g` |
| 只用 shims 模式 | 环境变量和 hooks 不生效 | 交互式终端加 PATH 模式 |
| 忘记 `mise trust` | 配置不生效 | 首次进入项目运行 trust |
| 全局安装 npm/pip 包 | 污染 mise 管理的运行时 | 用 `mise use -g "npm:xxx"` |
| 忽略 mise.local.toml | 敏感信息提交到 git | 加入 .gitignore |

## Real-World Impact

- **系统纯净**：brew list 只有 mise、git 等基础设施
- **版本隔离**：不同项目使用不同 node/python 版本无冲突
- **一键还原**：`rm -rf ~/.local/share/mise/` 回到干净系统
- **团队一致**：mise.toml 提交到 git，所有人环境相同
- **CI 友好**：同样的配置在 GitHub Actions 直接使用
