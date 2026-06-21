# Hermes Agent — 编译、发布指南（Linux/WSL）

> 版本：v0.15.1 | 构建：setuptools | 包管理：uv（推荐）/ pip | 发布：GitHub Release + CalVer

---

## 1. 环境准备

### 1.1 依赖

- Python ≥ 3.11（项目精确锁定 3.11+）
- uv（推荐）或 pip
- git

### 1.2 两种安装方式

| 方式 | 用途 | 路径 |
|------|------|------|
| **curl 一键安装** | 生产使用 | `~/.hermes/hermes-agent/`（uv 管理） |
| **源码开发** | 改代码 / 构建 / 发布 | 任意目录，推荐 `~/hermes-agent/` |

### 1.3 一键安装（生产）

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 验证
hermes --version
```

安装脚本自动完成：安装 uv → 创建 venv → `pip install hermes-agent[all]` → 注册 `hermes` 命令。

### 1.4 源码开发

```bash
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent

# 创建 venv（二选一）
python3 -m venv venv                  # 标准
uv venv                               # uv（更快）

# 激活
source venv/bin/activate

# 开发模式安装（代码改动立即生效）
pip install -e .

# 安装全部可选依赖
pip install -e ".[all]"

# 验证
python -m hermes_cli.main --version
```

> `pip install -e .` 只需执行一次。之后修改代码直接运行，无需重新构建。

---

## 2. 构建（Build）

### 2.1 安装构建工具

```bash
source venv/bin/activate
pip install build
```

### 2.2 构建产物

```bash
python -m build
```

产出在 `dist/` 目录：

```
dist/
├── hermes_agent-0.15.1-py3-none-any.whl   # wheel（推荐）
└── hermes_agent-0.15.1.tar.gz              # sdist 源码包
```

> wheel 是 `py3-none-any`（纯 Python，无平台依赖），所有系统通用。

### 2.3 本地验证构建产物

```bash
# 安装 wheel（跳过依赖，仅验证打包正确性）
pip install dist/hermes_agent-0.15.1-py3-none-any.whl --force-reinstall --no-deps

python -m hermes_cli.main --version
```

---

## 3. 发布（Release）

### 3.1 版本号管理

版本号定义在三处，发布时必须同步更新：

| 文件 | 位置 |
|------|------|
| `pyproject.toml` | `version = "0.15.1"` |
| `hermes_cli/__init__.py` | `__version__` |
| `acp_registry/agent.json` | ACP 清单版本 |

测试强制校验版本一致性（`tests/acp/test_registry_manifest.py`）。

### 3.2 发布脚本

项目内置 `scripts/release.py`，自动生成 changelog 并创建 GitHub Release：

```bash
# 预览 changelog（dry run）
python scripts/release.py

# 预览并指定版本号策略
python scripts/release.py --bump minor

# 正式发布
python scripts/release.py --bump minor --publish

# 首次发布
python scripts/release.py --bump minor --publish --first-release

# 指定日期（CalVer 格式）
python scripts/release.py --bump minor --publish --date 2026.3.15
```

`--bump` 选项：`patch`（默认）/ `minor` / `major`

### 3.3 手动发布流程

如果不用 release.py，完整流程：

```bash
# 1. 确认版本号已更新
grep 'version = ' pyproject.toml

# 2. 构建
python -m build

# 3. 上传到 PyPI（需要 token）
pip install twine
twine upload dist/*

# 4. 创建 Git Tag 并推送
git tag v0.15.1
git push origin v0.15.1

# 5. 在 GitHub 创建 Release（附 changelog + dist 产物）
gh release create v0.15.1 dist/* --title "v0.15.1" --notes-file CHANGELOG.md
```

---

## 4. 测试

```bash
source venv/bin/activate

# 运行全部测试（排除 integration）
pytest

# 并行测试（更快）
python scripts/run_tests_parallel.py

# 仅 lint
python scripts/lint_diff.py
```

测试配置（`pyproject.toml`）：超时 30s/测试，跳过 `integration` 标记。

---

## 5. 依赖策略

### 核心依赖 vs 懒加载

| 类型 | 安装时机 | 示例 |
|------|----------|------|
| **核心依赖** | `pip install -e .` 时立即安装 | openai, rich, pydantic, prompt_toolkit |
| **可选依赖** | `pip install -e ".[all]"` 或按需 | edge-tts, anthropic, mcp |
| **懒加载** | 首次使用时自动安装 | firecrawl, fal, elevenlabs, telegram |

懒加载机制在 `tools/lazy_deps.py` 中实现——当用户首次启用某个后端时，自动 `pip install`。

### 常用 optional-dependencies

```bash
pip install -e ".[dev]"          # 开发：pytest, ruff, debugpy
pip install -e ".[mcp]"          # MCP 服务器支持
pip install -e ".[messaging]"    # Telegram, Discord, Slack
pip install -e ".[edge-tts]"     # TTS 语音
pip install -e ".[anthropic]"    # Anthropic 原生 provider
pip install -e ".[all]"          # 全部非懒加载依赖
```

---

## 6. WSL 特别说明

### Windows/Linux venv 不通用

Windows venv 产出的 Python 二进制是 PE 格式（`.exe`），Linux 需要 ELF 格式。
**同一份源码目录下的 `venv/` 不能跨系统使用。**

```bash
# 在 /mnt/e/ 共享目录下开发时，需要分别建 venv
# Windows: venv/Scripts/activate
# WSL:     venv/bin/activate

# 如果已有 Windows venv，WSL 里要重建：
cd /mnt/e/AI/hermes-agent
mv venv venv-windows          # 备份 Windows 版
python3.11 -m venv venv       # 创建 Linux 版
source venv/bin/activate
pip install -e .
```

### 文件路径对照

| Windows | WSL |
|---------|-----|
| `C:\Users\skype\.hermes\` | `/home/skype/.hermes/` |
| `E:\AI\hermes-agent\` | `/mnt/e/AI/hermes-agent/` |
| `~` = `C:\Users\skype` | `~` = `/home/skype` |

> WSL 的 `~` 是 `/home/skype/`，**不是** `/mnt/c/Users/skype/`。两边配置互相独立。

### WSL 代理问题

```bash
# 清除代理（直连时）
unset HTTPS_PROXY HTTP_PROXY https_proxy http_proxy ALL_PROXY all_proxy

# 使用国内镜像加速
pip install -e . -i https://pypi.tuna.tsinghua.edu.cn/simple

# 让特定域名绕过代理
export no_proxy="localhost,127.0.0.1,::1,api.deepseek.com"
```

---

## 7. 速查表

| 场景 | 命令 |
|------|------|
| 开发安装 | `pip install -e .` |
| 构建发布包 | `python -m build` |
| 发布到 PyPI | `twine upload dist/*` |
| GitHub Release | `python scripts/release.py --bump minor --publish` |
| 运行测试 | `pytest` |
| 启动 CLI | `hermes` 或 `python -m hermes_cli.main` |
| 查看版本 | `hermes --version` |
| 诊断问题 | `hermes doctor` |
