# Browser Use OpenSource Plugin (Codex & Antigravity)

将现有 Browser Use CLI 技能打包为本地插件，同时支持 **Codex** 与 **Antigravity**，通过 CDP 操作本机 Chrome，复用浏览器已有的登录状态。

这是由 [HadesScience](https://github.com/HadesScience) 维护的独立适配封装。浏览器控制能力来自上游 [Browser Use](https://github.com/browser-use/browser-use) 和 [Browser Harness](https://github.com/browser-use/browser-harness)，本仓库不包含它们的运行时。

## 工作方式

客户端（Codex 或 Antigravity）加载插件中的 `browser-use-opensource` 技能，通过终端将 Python 代码传给 `browser-use` 的标准输入。插件没有附加 MCP 服务，也不会自动安装、升级或重启 Browser Use。

插件使用独立标识 `browser-use-opensource`，避免与旧版 `browser-use` 本地插件的自动迁移规则冲突。

仓库同时提供：
- **Codex 清单与本地市场**：`.codex-plugin/plugin.json`、`.agents/plugins/marketplace.json`
- **Antigravity 原生插件清单**：根目录 `plugin.json`
- **共享技能**：`skills/browser-use-opensource/SKILL.md`（包含 Codex PowerShell 调用说明、Antigravity Windows 无 UTF-8 BOM `cmd.exe /c` 管道调用说明，以及共享 daemon 和终端沙箱连接限制的处理规则）

## 前置条件

- 已安装支持 stdin Python 工作流的 `browser-use` CLI，并加入 PATH。本封装基于 `browser-use 0.13.8` 随附的技能整理。
- Chrome 已按[上游连接说明](https://github.com/browser-use/browser-harness/blob/main/install.md)启用远程调试。
- 目标客户端：
  - **Codex**：支持本地插件市场和 `codex plugin` 命令；或
  - **Antigravity**：支持 `~/.gemini/config/plugins/` 本地插件自动发现与 `agy plugin` 命令。

已有安装可直接复用；需要安装 Browser Use 时，请遵循[上游仓库文档](https://github.com/browser-use/browser-use)。

## 未检测到本地环境时

先检查命令是否可用，再决定是否安装。以下安装步骤只适用于缺失的组件，已有可用环境直接跳过。

### 1. 检查 CLI

Windows PowerShell：

```powershell
Get-Command browser-use -ErrorAction SilentlyContinue
Get-Command uv -ErrorAction SilentlyContinue
```

macOS / Linux：

```bash
command -v browser-use
command -v uv
```

如果能够找到 `browser-use`，先运行 `browser-use --doctor`。找不到命令也可能只是 PATH 尚未刷新：重新打开终端，或在已使用 uv 安装的情况下运行 `uv tool list` 确认，不要直接重复安装。

### 2. 缺少 uv 时，从官方渠道安装

按 [Astral 官方 uv 安装文档](https://docs.astral.sh/uv/getting-started/installation/)选择适合系统的方式。Windows 可以使用官方文档列出的 WinGet 命令：

```powershell
winget install --id=astral-sh.uv -e
```

macOS / Linux 可以使用官方安装脚本：

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

安装后重新打开终端，执行 `uv --version` 确认命令可用。

### 3. 缺少 Browser Use CLI 时安装

[Browser Use 官方 CLI 指南](https://github.com/browser-use/browser-use#path-2-cli)采用 uv 和 Python 3.12。仅在尚未安装 CLI 时执行：

```text
uv tool install --python 3.12 browser-use
```

uv 会按需准备 Python 3.12 和工具依赖，不需要先在系统中手动配置 Python 环境。若安装后终端仍找不到命令，执行 `uv tool update-shell`，然后重新打开终端。

本插件已包含技能文件，无需再执行 `browser-use skill install` 注册同名独立技能。这里安装的是 CLI 运行环境，插件本身仍按下方步骤安装。

### 4. 安装 Chrome 并允许本地连接

没有 Chrome 时，从 [Google Chrome 官方网站](https://www.google.com/chrome/)下载安装。启动 Chrome 后，按 [Browser Harness 官方连接说明](https://github.com/browser-use/browser-harness/blob/main/install.md#if-chrome-blocks-it)操作：

1. 打开 `chrome://inspect/#remote-debugging`。
2. 勾选允许当前浏览器实例远程调试。
3. 执行 `browser-use --doctor`，再运行下方的连接验证示例。
4. 如果 Chrome 显示连接授权弹窗，允许本次连接，并等待原命令完成。

本地 Chrome 控制不需要 Browser Use Cloud API Key。只有选择可选云端浏览器服务时，才需要按上游文档配置云端认证。

如果当前 Chrome 没有上述远程调试选项，或诊断仍失败，请查看官方连接文档，不要通过清除浏览器资料或反复重装工具处理。

## 安装插件

### 方式一：在 Antigravity 中安装

Antigravity 自动扫描全局插件目录 `~/.gemini/config/plugins/<插件名>/`（Windows 下为 `%USERPROFILE%\.gemini\config\plugins\browser-use-opensource\`），识别根目录 `plugin.json` 与 `skills/browser-use-opensource/SKILL.md`。

**Windows PowerShell**：

```powershell
git clone https://github.com/HadesScience/browser-use-opensource-plugin.git
$target = "$env:USERPROFILE\.gemini\config\plugins\browser-use-opensource"
New-Item -ItemType Directory -Force -Path "$target\skills" | Out-Null
Copy-Item -Path ".\browser-use-opensource-plugin\plugin.json" -Destination "$target\plugin.json" -Force
Copy-Item -Path ".\browser-use-opensource-plugin\skills\browser-use-opensource" -Destination "$target\skills\browser-use-opensource" -Recurse -Force

agy plugin validate $target --strict --json
agy plugin list
```

**macOS / Linux**：

```bash
git clone https://github.com/HadesScience/browser-use-opensource-plugin.git
mkdir -p ~/.gemini/config/plugins/browser-use-opensource/skills
cp ./browser-use-opensource-plugin/plugin.json ~/.gemini/config/plugins/browser-use-opensource/plugin.json
cp -R ./browser-use-opensource-plugin/skills/browser-use-opensource ~/.gemini/config/plugins/browser-use-opensource/skills/

agy plugin validate ~/.gemini/config/plugins/browser-use-opensource --strict --json
agy plugin list
```

安装后刷新 Customizations 页面（或重启 Antigravity 并新开会话），即可在 Customizations 中单独启用/停用 **Browser Use OpenSource** 插件。

### 方式二：在 Codex 中安装

直接通过 GitHub 仓库注册市场并安装：

```text
codex plugin marketplace add HadesScience/browser-use-opensource-plugin
codex plugin add browser-use-opensource@browser-use-codex
codex plugin list --marketplace browser-use-codex --json
```

重新打开 Codex 或新建会话，使插件技能被加载。若之前已独立注册同名技能，确认插件安装成功后再移除重复项。

也可以克隆仓库后从本地注册：

```text
git clone https://github.com/HadesScience/browser-use-opensource-plugin.git
codex plugin marketplace add ./browser-use-opensource-plugin
codex plugin add browser-use-opensource@browser-use-codex
```

## 验证连接

### 在 Antigravity (Windows) 中验证

Antigravity 的 Windows PowerShell 终端在开启 UTF-8 输出编码时，直接用 PowerShell 管道（`| browser-use`）可能在 `stdin` 开头注入 UTF-8 BOM（`U+FEFF`），引起 Python `SyntaxError: invalid non-printable character U+FEFF`。在 Antigravity 下应将管道放在 `cmd.exe /c` 内部执行：

```powershell
cmd.exe /c 'echo print(cdp("Browser.getVersion")["product"]) | browser-use'
```

读取当前浏览页面与标签页列表（使用 `ensure_ascii=True` 避免编码污染）：

```powershell
cmd.exe /c 'echo import json; print(json.dumps({"page_info": page_info(), "pages": [{"title": t.get("title"), "url": t.get("url"), "attached": t.get("attached")} for t in cdp("Target.getTargets").get("targetInfos", []) if t.get("type") == "page"]}, ensure_ascii=True)) | browser-use'
```

### 在 Codex / 普通 PowerShell 中验证

```powershell
@'
print(cdp("Browser.getVersion")["product"])
'@ | browser-use
```

成功时会输出当前 Chrome 版本。终端必须能够访问本机 Chrome 和共享 daemon；如果终端沙箱无法连接 localhost，通过正常的工具权限流程请求宿主机执行环境。不要仅凭沙箱内的检查失败就删除运行数据或重启共享 daemon。

## 来源与许可

- 上游项目：[browser-use/browser-use](https://github.com/browser-use/browser-use)
- 技能来源：[Browser Use SKILL.md](https://github.com/browser-use/browser-use/blob/main/browser_use/skills/browser-use/SKILL.md)
- 连接实现与文档：[browser-use/browser-harness](https://github.com/browser-use/browser-harness)

技能保留了上游工作流，并增加 Codex 与 Antigravity 在 Windows 下的调用说明。本仓库按 MIT 许可证发布，保留 Browser Use 的原始版权声明，详见 [LICENSE](LICENSE)。
