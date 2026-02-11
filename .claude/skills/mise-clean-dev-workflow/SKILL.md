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
- 需要管理项目密钥/敏感配置且不想明文存储
- Monorepo 多项目统一管理

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
| 移除项目工具 | `mise unuse python` |
| 安装所有配置工具 | `mise install` |
| 查看已安装 | `mise ls` |
| 检查过期版本 | `mise outdated` |
| 升级工具 | `mise upgrade` |
| 清理未使用版本 | `mise prune` |
| 搜索可用工具 | `mise search <keyword>` |
| 查看任务 | `mise tasks ls` |
| 运行任务 | `mise run <task>` |
| 监听文件变化运行 | `mise watch <task>` |
| 信任项目配置 | `mise trust` |
| 信任目录下所有配置 | `mise trust --all` |
| 诊断问题 | `mise doctor` |
| PATH 诊断 | `mise doctor path` |
| 格式化配置文件 | `mise fmt` |
| 生成锁文件 | `mise lock` |
| 进入子 shell 环境 | `mise en` |
| 从 nvm 迁移 | `mise sync node` |
| 从 pyenv 迁移 | `mise sync python` |

## Implementation

### 1. Shell 配置（双模式）

```bash
# ~/.zshenv - 所有 shell 类型都会加载（IDE、脚本、非交互式）
# ⚠️ 必须用绝对路径，因为 .zshenv 加载时 Homebrew PATH 尚未生效
eval "$(/opt/homebrew/bin/mise activate zsh --shims)"

# ~/.zshrc - 交互式终端（PATH 模式会覆盖 shims）
eval "$(mise activate zsh)"
```

**为什么用 `.zshenv` 而不是 `.zprofile`：**
- `.zshenv` 对所有 shell 类型生效（登录/非登录、交互/非交互）
- `.zprofile` 仅对登录 shell 生效，非登录子 shell 中 shims 不可用
- 交互式终端中 `.zshrc` 的 PATH 模式会自动覆盖 shims，不会冲突

**⚠️ 常见陷阱：**
- `.zshenv` 中必须用 `/opt/homebrew/bin/mise`（Apple Silicon）或 `/usr/local/bin/mise`（Intel）的绝对路径
- 确保配置文件使用 LF 换行符（非 CRLF），否则会报 `command not found: ^M`
- 检查方法：`file ~/.zshenv`，修复：`sed -i '' 's/\r$//' ~/.zshenv`

**shims vs PATH 区别：**

| 特性 | shims 模式 | PATH 模式 |
|------|-----------|-----------|
| 适用场景 | IDE、脚本、CI | 交互式终端 |
| 版本切换 | 调用时解析 | 进入目录自动切换 |
| 环境变量 | 仅对 mise 工具生效 | 全局生效 |
| hooks | 不触发 | 自动触发 |

### 2. 全局配置（增强版）

```toml
# ~/.config/mise/config.toml
[tools]
node = "22"
python = "3.12"
"npm:prettier" = "latest"
"cargo:fd-find" = "latest"

[settings]
# 后端控制 - 禁用不需要的后端减少噪音
disable_backends = ["asdf", "vfox"]
auto_install = true
# 版本锁定 - mise use 时自动 pin 精确版本
pin = true
# 锁文件 - 确保团队安装完全一致的二进制
lockfile = true
# command not found 自动安装
not_found_auto_install = true
# 颜色主题（可选：default, base16, catppuccin, dracula）
color_theme = "catppuccin"
# 性能调优
jobs = 8                            # 并行安装数
fetch_remote_versions_cache = "4h"  # 远程版本缓存（默认 1h）
env_cache = true                    # [实验] 缓存环境变量计算结果
env_cache_ttl = "2h"                # 环境缓存 TTL

# 安全 - aqua 后端启用签名验证
[settings.aqua]
cosign = true
slsa = true

# 状态提示 - 进入目录时显示工具和环境变量
[settings.status]
show_tools = true
show_env = true
missing_tools = "always"
```

### 3. 项目配置（完整示例）

```toml
# mise.toml
min_version = "2025.1.0"

[tools]
node = "20"
# 原生 Python venv 集成 - mise 自动创建和激活虚拟环境
python = { version = "3.12", postinstall = "pip install -e '.[dev]'" }
"aqua:hashicorp/terraform" = "1.5.0"

# 共享变量 - 可在 tera 模板中用 {{ vars.APP_NAME }} 引用
[vars]
APP_NAME = "my-app"
PORT = "3000"

[env]
NODE_ENV = "development"
# 必填变量 - 未设置时 mise 报错并提示
DATABASE_URL = { required = "请在 mise.local.toml 中配置" }
# 延迟求值 - 等工具安装后再解析（可引用工具路径）
NODE_PATH = { value = "{{ env.HOME }}/.local/share/mise/installs/node/20/lib", tools = true }

# 从 .env 加载（支持 .env / .env.json / .env.yaml）
_.file = ".env"
# 加载并自动脱敏（日志中不显示值）
# _.file = { path = ".env.production", redact = true }

# 添加项目 bin 到 PATH
_.path = ["./node_modules/.bin", "./scripts"]

# 从 shell 脚本捕获环境变量（执行脚本并 diff 环境）
# _.source = "./scripts/load-env.sh"

# 原生 Python venv（比 virtualenv 字段更强大）
[env._.python]
venv = { path = ".venv", create = true }

# 敏感变量脱敏 - 匹配的变量值在日志/输出中显示为 [redacted]
[redactions]
"*_TOKEN"
"*_SECRET"
"*_KEY"
"DATABASE_*"

[tasks]
dev = "npm run dev"
build = "npm run build"

[tasks.test]
description = "运行测试"
run = "npm test"
depends = ["build"]

[hooks]
enter = "echo '项目环境已就绪: {{ vars.APP_NAME }}'"
postinstall = "npm install"
```

### 4. 本地覆盖（不提交）

```toml
# mise.local.toml - 加入 .gitignore
[env]
DATABASE_URL = "postgres://localhost:5432/mydb"
AWS_PROFILE = "personal"
```

### 5. 后端选择（完整指南）

```toml
[tools]
# ── 核心语言（内置后端，最快，有预编译二进制） ──
node = "22"
python = "3.12"
go = "latest"
rust = "stable"
ruby = "3.3"
java = "21"
bun = "latest"
deno = "latest"
zig = "latest"

# ── Aqua 后端（推荐用于 CLI 工具，支持 cosign/SLSA 签名验证） ──
"aqua:BurntSushi/ripgrep" = "latest"
"aqua:sharkdp/fd" = "latest"
"aqua:junegunn/fzf" = "latest"
"aqua:jqlang/jq" = "latest"
"aqua:hashicorp/terraform" = "1.5.0"

# ── GitHub Release（aqua 没收录时用这个） ──
"github:astral-sh/uv" = "latest"
"github:casey/just" = "latest"

# ── 语言生态包管理器 ──
"npm:eslint" = "latest"           # Node.js CLI 工具
"npm:typescript" = "latest"
"pipx:black" = "latest"           # Python CLI 工具（通过 pipx/uvx 隔离安装）
"pipx:poetry" = "latest"
"cargo:tokei" = "latest"          # Rust crate（优先用 cargo-binstall）
"gem:rails" = "latest"            # Ruby gem
"go:golang.org/x/tools/gopls" = "latest"  # Go module

# ── 其他后端 ──
# "ubi:owner/repo" = "latest"     # Universal Binary Installer
# "http:https://example.com/tool-{version}-{os}-{arch}.tar.gz" = "1.0"  # 直接 HTTP 下载
# "conda:numpy" = "latest"        # [实验] Conda 包
# "dotnet:fantomas" = "latest"    # [实验] .NET 工具
# "spm:nicklockwood/SwiftFormat" = "latest"  # [实验] Swift Package Manager
```

**后端选择优先级：** 核心内置 > aqua（有签名验证） > github > ubi > 语言包管理器 > http

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
# 脱敏 - 隐藏任务输出中的敏感值
redactions = ["AWS_*", "DEPLOY_TOKEN"]

# 任务专属工具版本（不影响项目其他部分）
[tasks.legacy-build]
run = "npm run build"
tools = { node = "18" }

# 混合串行/并行执行
[tasks.ci]
run = [
  "npm run lint",                          # 先串行执行 lint
  { tasks = ["test:unit", "test:e2e"] },   # 然后并行执行两个测试
  "npm run build",                         # 最后串行执行 build
]
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

### 任务参数（usage 定义）

```toml
[tasks.deploy]
description = "部署到指定环境"
# usage 定义参数，支持位置参数、选项、flag
usage = '''
arg "environment" help="目标环境" {
  choices "staging" "production"
}
option "-t --tag" help="镜像标签" default="latest"
flag "-d --dry-run" help="仅预览不执行"
'''
run = """
echo "部署 {{ usage.environment }} 标签 {{ usage.tag }}"
{% if usage.dry_run %}
echo "[DRY RUN] 跳过实际部署"
{% else %}
kubectl set image deployment/app app=myapp:{{ usage.tag }}
{% endif %}
"""
```

```bash
mise run deploy staging --tag v1.2.3
mise run deploy production --dry-run
mise run deploy --help  # 自动生成帮助文档
```

### 远程任务

```toml
# 从 HTTP 加载任务脚本
[tasks.setup]
file = "https://raw.githubusercontent.com/org/shared-tasks/main/setup.sh"

# 从 Git 仓库加载任务目录
[task_config]
includes = [
  "./mise-tasks",
  "git::https://github.com/org/shared-tasks?ref=main#mise-tasks"
]
```

### Prepare 自动依赖安装（实验性）

```toml
# 自动检测并运行依赖安装（如 npm install、pip install 等）
# 当 package.json 比 node_modules 新时自动执行
[prepare.npm]
auto = true

# 自定义 prepare provider
[prepare.codegen]
run = "make generate"
sources = ["schema/*.graphql"]
outputs = ["src/generated/"]
description = "生成 GraphQL 类型"
```

## 安全加固

### Lockfile（确定性安装）

```bash
# 生成锁文件 - 记录精确的下载 URL 和校验和
mise lock

# 锁定模式安装 - 必须匹配锁文件
mise install --locked
```

```toml
# mise.toml
[settings]
lockfile = true   # 启用锁文件（默认已启用）
# locked = true   # 严格模式：安装时必须有锁文件
```

生成的 `mise.lock` 提交到 git，确保团队和 CI 安装完全一致的二进制文件。

### Paranoid 模式

```toml
# ~/.config/mise/config.toml
[settings]
paranoid = true
```

启用后：
- **所有配置文件**都需要 trust（不仅是有模板/env 的）
- 配置文件内容变更后需要**重新 trust**（基于内容哈希）
- 禁止通过短名安装社区插件（必须用完整 git URL）
- 所有请求强制 HTTPS

### 密钥管理：age 加密（实验性）

```bash
# 生成 age 密钥（首次）
age-keygen -o ~/.config/mise/age.txt

# 加密敏感环境变量（交互式输入）
mise set --age-encrypt --prompt DB_PASSWORD
```

```toml
# mise.toml - 加密值安全提交到 git
[env]
DB_PASSWORD = { age = "YWdlLWVuY3J5cHRpb24ub3JnL3YxCi0+..." }
# age 加密的值自动标记为 redacted，日志中不会泄露
```

密钥查找顺序：`MISE_AGE_KEY` 环境变量 > `settings.age.key_file` > `~/.config/mise/age.txt` > SSH 密钥

### 密钥管理：SOPS 集成（实验性）

```bash
# 用 sops + age 加密 .env 文件
sops --encrypt --age $(cat ~/.config/mise/age.txt | grep public | cut -d: -f2 | tr -d ' ') \
  .env.json > .env.encrypted.json
```

```toml
# mise.toml - 自动解密 SOPS 加密的 env 文件
[env]
_.file = { path = ".env.encrypted.json", redact = true }
```

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

### 配置文件层级与优先级

```bash
# 加载顺序（后者覆盖前者）
~/.config/mise/config.toml       # 全局
mise.toml                        # 项目基础
mise.development.toml            # 环境覆盖
mise.local.toml                  # 本地覆盖（gitignore）
mise.development.local.toml      # 环境+本地覆盖（gitignore）
```

### 早期初始化配置（.miserc.toml）

```toml
# .miserc.toml - 在所有其他配置之前加载
# 用于设置影响配置发现本身的选项
MISE_ENV = "development"
ceiling_paths = ["/Users"]           # 停止向上搜索配置的边界
ignored_config_paths = ["~/vendor"]  # 忽略特定路径的配置
```

### 多环境组合

```bash
# 同时激活多个环境（后者优先）
MISE_ENV=ci,test mise run test
mise -E staging -E debug run deploy
```

### Shell 变量展开

```toml
# mise.toml
[settings]
env_shell_expand = true  # 启用 $VAR 和 ${VAR:-default} 语法

[env]
APP_DIR = "${HOME}/apps/${APP_NAME:-myapp}"
LOG_DIR = "${APP_DIR}/logs"
PATH_EXTRA = "${APP_DIR}/bin"
```

### Tera 模板（在 env 和 tasks 中可用）

```toml
[env]
# 动态计算值
NUM_WORKERS = "{{ num_cpus() }}"
GIT_SHA = "{{ exec(command='git rev-parse --short HEAD') }}"
# 带缓存的命令执行（避免重复调用）
AWS_ACCOUNT = "{{ exec(command='aws sts get-caller-identity --query Account --output text', cache_duration='1h') }}"
# 路径操作
PROJECT_NAME = "{{ cwd | basename }}"
# 条件逻辑
RUST_LOG = "{% if env.CI is defined %}warn{% else %}debug{% endif %}"
```

## Hooks 系统（实验性）

### 五种 Hook 类型

```toml
[hooks]
# 进入项目目录时触发（仅首次）
enter = "echo '欢迎回到 {{ vars.APP_NAME }}'"

# 离开项目目录时触发
leave = "echo '离开项目'"

# 每次目录切换时触发（项目内 cd）
cd = "echo '当前目录: $PWD'"

# 工具安装前/后
preinstall = "echo '即将安装工具...'"
postinstall = "echo '工具安装完成: $MISE_INSTALLED_TOOLS'"
```

### 工具级 postinstall

```toml
[tools]
# 安装 node 后自动安装 pnpm
node = { version = "22", postinstall = "npm install -g pnpm" }
# 安装 python 后自动安装常用包
python = { version = "3.12", postinstall = "pip install ipython" }
```

Hook 中可用的环境变量：`MISE_PROJECT_ROOT`、`MISE_PREVIOUS_DIR`、`MISE_ORIGINAL_CWD`、`MISE_INSTALLED_TOOLS`（JSON）。

### Watch Files（文件监听 Hook）

```toml
# 配置文件变化时自动执行命令（不同于 mise watch 任务）
[[watch_files]]
patterns = ["package.json", "package-lock.json"]
run = "npm install"

[[watch_files]]
patterns = ["requirements*.txt"]
run = "pip install -r requirements.txt"
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

## CI/CD 集成

### 自动生成配置

```bash
# 生成 GitHub Actions workflow
mise generate github-action

# 生成 bootstrap 脚本（用于 CI 环境快速初始化）
mise generate bootstrap

# 生成 devcontainer.json（VS Code Remote Containers）
mise generate devcontainer

# 生成 git pre-commit hook（提交前自动运行 lint 等）
mise generate git-pre-commit

# 生成任务文档
mise generate task-docs
```

### GitHub Actions 最佳实践

```yaml
# .github/workflows/ci.yml
- uses: jdx/mise-action@v2
  with:
    install: true        # 安装 mise.toml 中定义的工具
    cache: true          # 缓存已安装的工具
# mise.lock 确保 CI 和本地安装完全一致
- run: mise run ci
```

## 迁移助手

### 从 nvm/pyenv/rbenv 迁移

```bash
# 同步已有的 node 版本（从 nvm/fnm/nodenv）
mise sync node

# 同步已有的 python 版本（从 pyenv）
mise sync python

# 同步已有的 ruby 版本（从 rbenv）
mise sync ruby
```

### 兼容 .node-version / .python-version 等

mise 自动识别 idiomatic 版本文件（`.node-version`、`.ruby-version`、`.python-version` 等），无需修改现有项目即可使用。

## 团队与企业配置

### 自动信任路径

```toml
# ~/.config/mise/config.toml
[settings]
# 自动信任这些路径下的所有配置（无需手动 mise trust）
trusted_config_paths = ["~/work", "~/projects"]

# 停止向上搜索配置的边界（避免意外加载上层配置）
ceiling_paths = ["/Users"]
```

### URL 替换（企业镜像/代理）

```toml
# ~/.config/mise/config.toml
[settings]
# 将 GitHub 下载替换为企业内网镜像
url_replacements = { "https://github.com" = "https://github.mycompany.com" }
```

### Shell 别名

```toml
# mise.toml - 定义项目级 shell 别名
[shell_alias]
ll = "ls -la"
dev = "mise run dev"
t = "mise run test"
```

## Common Mistakes

| 错误 | 后果 | 修复 |
|------|------|------|
| `.zshenv` 中用相对命令 `mise` | PATH 未初始化时报 `command not found: mise` | 改用绝对路径 `/opt/homebrew/bin/mise` |
| Shell 配置文件含 CRLF 换行符 | 报 `command not found: ^M` | `sed -i '' 's/\r$//' ~/.zshenv` |
| shims 放在 `.zprofile` 而非 `.zshenv` | 非登录子 shell 中工具不可用 | 移到 `.zshenv` 并用绝对路径 |
| 用 brew 装 node/python | 版本冲突，难以清理 | 卸载，改用 `mise use -g` |
| 只用 shims 模式 | 环境变量和 hooks 不生效 | 交互式终端加 PATH 模式 |
| 忘记 `mise trust` | 配置不生效 | 首次进入项目运行 trust |
| 全局安装 npm/pip 包 | 污染 mise 管理的运行时 | 用 `mise use -g "npm:xxx"` |
| 忽略 mise.local.toml | 敏感信息提交到 git | 加入 .gitignore |
| 不用 lockfile | 团队/CI 安装的二进制可能不同 | `mise lock` 并提交 mise.lock |
| 明文存储密钥在 .env | 泄露风险 | 用 age 加密或 SOPS |
| 不设 trusted_config_paths | 每个新项目都要手动 trust | 全局配置中设置信任路径 |
| 不设 ceiling_paths | 上层目录的配置意外生效 | 设置 `/Users` 等边界 |
| 忘记 `mise prune` | 旧版本工具占用磁盘 | 定期运行或设 `cache_prune_age` |

## Real-World Impact

- **系统纯净**：brew list 只有 mise、git 等基础设施
- **版本隔离**：不同项目使用不同 node/python 版本无冲突
- **一键还原**：`rm -rf ~/.local/share/mise/` 回到干净系统
- **团队一致**：mise.toml + mise.lock 提交到 git，所有人环境完全相同
- **CI 友好**：同样的配置在 GitHub Actions 直接使用，lockfile 保证一致性
- **密钥安全**：age/SOPS 加密 + redactions 脱敏，敏感信息不泄露
- **供应链安全**：aqua cosign/SLSA 验证 + lockfile 校验和，防止篡改
- **零手动配置**：hooks + prepare + watch_files 自动化一切环境准备工作
- **渐进迁移**：sync 命令 + idiomatic 版本文件兼容，无需一次性改造
