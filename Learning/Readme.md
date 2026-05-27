# Hermes Agent 开发流程指南

## 前置条件

- Windows 10/11 + Git Bash / WSL2
- Python 3.11+（本项目使用 3.13.5）
- 虚拟环境已创建：`venv/`

---

## 1. WSL 环境配置

### 网络与代理

WSL2 通过代理访问外网时，代理可能阻断 HTTPS 隧道导致 `Proxy CONNECT aborted`。

**诊断步骤：**
```bash
# 查看 WSL 代理设置
echo $HTTPS_PROXY
echo $no_proxy

# 测试直连（绕过代理）
curl -v --noproxy '*' https://api.deepseek.com/v1/models \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY"
```

**修复 — 让特定域名绕过代理：**
```bash
# 临时生效
export no_proxy="localhost,127.0.0.1,::1,api.deepseek.com"
export NO_PROXY="$no_proxy"

# 持久化（写入 ~/.bashrc）
echo 'export no_proxy="$no_proxy,api.deepseek.com"' >> ~/.bashrc
echo 'export NO_PROXY="$no_proxy,api.deepseek.com"' >> ~/.bashrc
```

**如果需要走代理（国内直连不通）：**
1. 确认 Windows 代理软件开启了 **允许局域网连接 (Allow LAN)**
2. 确认代理端口与 `$HTTPS_PROXY` 一致
3. 确认代理支持 HTTPS CONNECT 隧道
4. WSL 中获取 Windows 代理 IP：`cat /etc/resolv.conf | grep nameserver`

### WSL 中 pip 代理问题

pip 安装依赖时也会被代理拦截报 `ProxyError`。需要彻底清除代理变量：

```bash
# 清除所有代理变量
unset HTTPS_PROXY HTTP_PROXY https_proxy http_proxy ALL_PROXY all_proxy

# 确认已清除
env | grep -i proxy

# 如果 unset 不生效（venv 继承了环境变量），用 env -u 强制清除
env -u HTTPS_PROXY -u HTTP_PROXY -u https_proxy -u http_proxy -u ALL_PROXY \
  pip install pyyaml -i https://pypi.tuna.tsinghua.edu.cn/simple

# 装完依赖后再安装项目
env -u HTTPS_PROXY -u HTTP_PROXY -u https_proxy -u http_proxy -u ALL_PROXY \
  pip install -e . -i https://pypi.tuna.tsinghua.edu.cn/simple
```

> 国内网络用清华镜像 `-i https://pypi.tuna.tsinghua.edu.cn/simple` 加速。

### WSL 中 python 命令不存在

WSL 默认只有 `python3`，需要创建软链接：

```bash
sudo ln -s /usr/bin/python3 /usr/bin/python
```

### WSL 和 Windows venv 不能共用

项目目录 `venv/` 被 WSL 和 Windows 共享，但两边的 venv **不能混用**。
WSL 中 `pip install -e .` 会写入 `/usr/bin/python` 路径，导致 Windows 侧报错：
`did not find executable at '/usr/bin\python.exe': ???????????`

**解决方案：各自使用独立的 venv 目录**

```bash
# WSL 中使用单独的 venv
python3 -m venv venv-linux
source venv-linux/bin/activate
pip install -e .
```

```powershell
# Windows 中使用单独的 venv
python -m venv venv
.\venv\Scripts\activate
pip install -e .
```

如果 Windows venv 已被 WSL 污染，删掉重建：
```powershell
deactivate
Remove-Item -Recurse -Force venv
python -m venv venv_win
.\venv\Scripts\activate
pip install -e .
```

> 建议：在 `.gitignore` 中添加 `venv-linux/` 避免 WSL venv 被提交。

### 文件路径对照

| Windows 路径 | WSL 路径 |
|-------------|----------|
| `C:\Users\skype\.hermes\.env` | `/mnt/c/Users/skype/.hermes/.env` |
| `C:\Users\skype\.hermes\config.yaml` | `/mnt/c/Users/skype/.hermes/config.yaml` |
| `~` (Windows) | `/mnt/c/Users/skype` |
| `~` (WSL) | `/home/用户名/` |

> 注意：hermes 跑在哪个系统，`.env` 就读哪个系统的 `~`。Windows 侧运行读 Windows 路径，WSL 侧运行读 Linux 路径。

---

## 2. 环境准备（首次）

```bash
# 创建虚拟环境（如已有则跳过）
python -m venv venv

# 激活虚拟环境（Git Bash）
linux: source venv/Scripts/activate     # Linux 下是 bin/ 不是 Scripts/
windows: venv/bin/activate

# 安装项目依赖（开发模式，代码改动立即生效）
pip install -e .

# 安装构建工具（仅正式打包时需要）
venv/Scripts/pip3.13.exe install build
```

验证安装：

```bash
python -m hermes_cli.main --version
# 输出：Hermes Agent v0.12.0 (2026.4.30) ...
```

---

## 3. 开发调试（日常使用）

开发模式下改完代码直接运行，无需重新构建：

```bash
source venv/Scripts/activate


# 1. 改代码（用任意编辑器）
#    示例：编辑 hermes_cli/banner.py 第 325 行的 format_banner_version_label()

# 2. 验证（改完直接跑，不用构建）
python -c "from hermes_cli.banner import format_banner_version_label; print(format_banner_version_label())"

# 3. 启动看效果
python -m hermes_cli.main
```

---

## 4. 正式构建打包（发布时使用）

```bash
source venv/Scripts/activate

# 构建
python -m build

# 产出在 dist/ 目录：
#   hermes_agent-0.12.0-py3-none-any.whl
#   hermes_agent-0.12.0.tar.gz

# 安装编译产物
venv/Scripts/pip3.13.exe install dist/hermes_agent-0.12.0-py3-none-any.whl --force-reinstall --no-deps

# 验证
python -m hermes_cli.main --version
```

---

## 5. 启动方式速查

```bash
source venv/Scripts/activate

# 交互式聊天（主入口）
python -m hermes_cli.main

# 单次提问模式
python -m hermes_cli.main -z "你的问题"

# 指定模型启动
python -m hermes_cli.main -m <model>

# 首次配置向导
python -m hermes_cli.main setup

# 诊断问题
python -m hermes_cli.main doctor
```

---

## 6. 流程对比

| 场景 | 步骤 | 说明 |
|------|------|------|
| **开发调试** | 改代码 → 直接运行 | `pip install -e .` 后代码改动立即生效 |
| **正式发布** | 改代码 → 构建 → 安装 → 验证 | `python -m build` 生成 wheel/sdist |

---

## 7. 注意事项

- Windows 上 venv 的 pip 路径是 `venv/Scripts/pip3.13.exe`，不是 `venv/bin/pip`
- 开发模式 `pip install -e .` 只需安装一次，之后改代码直接运行
- API Key 配置在 `~/.hermes/.env` 中
