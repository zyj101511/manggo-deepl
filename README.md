# DeepLX 翻译 for Manggo

一个用于 [Manggo](https://www.yxwza.org/) 的文本翻译插件。通过自行运行的 [DeepLX-Serverless](https://github.com/guobao2333/DeepLX-Serverless) 翻译单词、短语和句子。支持自动识别源语言，也可手动选择源语言和目标语言。

> **非官方项目。**本插件与 DeepL 及 Manggo 官方均无隶属或授权关系。翻译由用户自行配置的 DeepLX-Serverless 服务提供，并非 DeepL 官方 API。插件包不包含该服务。

## 功能

- 自动识别源语言，也可在 Manggo 中手动选择源语言和目标语言。
- 未指定目标语言时默认译为中文；可手动改为英文、日文等其他语言。
- 显示主译文。
- 可连接本机或自行部署的 DeepLX-Serverless 服务。

本插件**不提供音标、朗读、例句、词性或词典同义词**。

## 安装并启动 DeepLX-Serverless

先安装 [Git](https://git-scm.com/downloads) 和 **Node.js 22 或更新版本**，再按操作系统运行以下命令。DeepLX-Serverless 是独立项目，不能只安装本插件就使用翻译。以下步骤依据[上游部署说明](https://github.com/guobao2333/DeepLX-Serverless#deploy--%E9%83%A8%E7%BD%B2)。

**Windows（PowerShell）：**

```powershell
node --version
git clone https://github.com/guobao2333/DeepLX-Serverless.git
cd DeepLX-Serverless
npm.cmd install
npm.cmd run start
```

**Linux / macOS（终端）：**

```bash
node --version
git clone https://github.com/guobao2333/DeepLX-Serverless.git
cd DeepLX-Serverless
npm install
npm run start
```

`npm install` 只需在首次安装或更新依赖时运行。`npm run start` 会持续运行；手动启动时，关闭终端或按 `Ctrl+C` 就会停止服务。Windows 上用 `npm.cmd` 可避开 PowerShell 对 `npm.ps1` 的执行限制。项目默认监听 `http://localhost:6119`。

**验证服务：**保持启动终端开启，再打开另一个终端发送测试请求。Windows PowerShell：

```powershell
$body = @{ text = '你好，世界！'; source_lang = 'zh'; target_lang = 'en' } | ConvertTo-Json
Invoke-RestMethod -Uri 'http://localhost:6119/translate' -Method Post -ContentType 'application/json' -Body $body
```

Linux / macOS：

```bash
curl -X POST http://localhost:6119/translate -H 'Content-Type: application/json' -d '{"text":"你好，世界！","source_lang":"zh","target_lang":"en"}'
```

返回结果中出现 `data` 译文字段，说明服务可用。若请求失败，先检查启动终端的报错和网络连接。[接口格式见上游文档](https://github.com/guobao2333/DeepLX-Serverless/wiki/API-%E5%8F%82%E6%95%B0)。

## 开机后台启动 DeepLX

先确认手动启动和测试都成功，再关闭手动运行的服务，按系统选择一种方法。下面直接运行 `node src/server.js`，与该项目的 `npm run start` [启动脚本](https://github.com/guobao2333/DeepLX-Serverless/blob/main/package.json)相同；不必让终端窗口一直开着。

### Windows：任务计划程序

1. 在开始菜单打开**任务计划程序**，选择“创建任务”。“常规”中命名为 `DeepLX`，选“**不管用户是否登录都要运行**”。
2. “触发器”中新建“**启动时**”。
3. “操作”中新建“启动程序”，填写下面三项（如果你的安装位置不同，改成实际路径）：

   | 项目 | 示例值 |
   | --- | --- |
   | 程序或脚本 | `C:\Program Files\nodejs\node.exe` |
   | 添加参数 | `src\server.js` |
   | 起始于 | DeepLX-Serverless所在目录 |

4. 在“设置”中取消“如果任务运行时间超过 3 天则停止”；笔记本用户还应检查“条件”中是否限制为接通电源才运行。保存时 Windows 可能要求输入账户密码。
5. 右键新任务选择“运行”，用上面的测试请求确认成功。要停止自动启动，可在任务计划程序中禁用该任务。

可在 PowerShell 运行 `(Get-Command node).Source` 查找实际的 `node.exe` 路径。“起始于”是 **DeepLX-Serverless 目录**，不是 Manggo 插件目录。Windows 的任务计划程序支持[启动时触发](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks-create)和[工作目录](https://learn.microsoft.com/en-us/windows/win32/taskschd/execaction-workingdirectory)。

### Linux：systemd

适用于使用 systemd 的发行版。先用 `whoami`、`pwd` 和 `command -v node` 查出用户名、DeepLX 项目绝对路径和 Node 可执行文件路径。创建服务文件：

```bash
sudo nano /etc/systemd/system/deeplx.service
```

填入以下内容，并把 `YOUR_USER`、项目目录和 `/usr/bin/node` 改成刚查到的真实值：

```ini
[Unit]
Description=DeepLX-Serverless
Wants=network-online.target
After=network-online.target

[Service]
Type=simple
User=YOUR_USER
WorkingDirectory=/home/YOUR_USER/DeepLX-Serverless
ExecStart=/usr/bin/node /home/YOUR_USER/DeepLX-Serverless/src/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

保存后启用、立即启动并检查状态：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now deeplx
sudo systemctl status deeplx
```

查看报错用 `journalctl -u deeplx -e`；停止并取消自启动用 `sudo systemctl disable --now deeplx`。如果 Node 是通过 nvm 安装，`ExecStart` 必须填 `command -v node` 显示的完整路径。[systemd 的工作目录说明](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html)。

### macOS：launchd

此方法在**用户登录后**自动后台启动。先在 DeepLX 项目目录运行 `pwd` 和 `command -v node`，记下两个绝对路径。创建配置文件：

```bash
mkdir -p ~/Library/LaunchAgents
nano ~/Library/LaunchAgents/com.zyj101511.deeplx.plist
```

粘贴以下内容，并替换 `YOUR_USER` 和 `/opt/homebrew/bin/node`；Intel Mac 的 Node 位置也可能是 `/usr/local/bin/node`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.zyj101511.deeplx</string>
  <key>ProgramArguments</key><array>
    <string>/opt/homebrew/bin/node</string>
    <string>/Users/YOUR_USER/DeepLX-Serverless/src/server.js</string>
  </array>
  <key>WorkingDirectory</key><string>/Users/YOUR_USER/DeepLX-Serverless</string>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/tmp/deeplx.out.log</string>
  <key>StandardErrorPath</key><string>/tmp/deeplx.err.log</string>
</dict></plist>
```

保存后加载；以后每次登录会自动启动：

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.zyj101511.deeplx.plist
launchctl print gui/$(id -u)/com.zyj101511.deeplx
```

停止并取消当前登录会话中的启动：`launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.zyj101511.deeplx.plist`；如需永久取消，再删除该 plist 文件。报错可看 `/tmp/deeplx.err.log`。[Apple 的 LaunchAgents 说明](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html)。

## 安装 Manggo 插件

1. 下载仓库中的 [`manggo-deeplx.mplugin`](./manggo-deeplx.mplugin)。
2. 在 Manggo 的“插件管理”中安装该文件。
3. 到“翻译服务”设置中添加 **DeepLX 翻译**。如果服务在本机运行，接口地址保持默认值即可。
4. 翻译时在 Manggo 里选择语言。源语言选“自动识别”即可让 DeepLX 识别；也可以分别手动指定源语言和目标语言。

更新插件时重新安装新版 `.mplugin`。DeepLX 服务需要在使用翻译时保持运行；电脑关机或服务停止后，本机地址便无法连接。

## 设置

“DeepLX 翻译”服务中的设置：

| 设置 | 默认值 | 作用 |
| --- | --- | --- |
| DeepLX 服务地址 | `http://localhost:6119` | 可改为自行部署的服务地址，也可填写带 `/translate` 的地址 |

插件为 Manggo 提供中、英、日、韩、法、德、西、意、葡、俄、荷、波等语言选项，具体语言对是否可用取决于上游。

## 常见问题

**为什么提示无法连接 DeepLX？**  确认服务已经启动，并检查服务设置中的地址。若服务运行在另一台设备，`localhost` 需要改为那台设备可访问的地址。

**需要 DeepL 官方 API Key 吗？**  不需要。DeepLX-Serverless 使用非官方的网页版接口，但你需要自行运行或部署它。

## 数据来源与声明

- 插件依据 [Manggo 原生插件开发指南](https://github.com/Pylogmon/manggo-plugin) 编写。
- 翻译服务由 [DeepLX-Serverless](https://github.com/guobao2333/DeepLX-Serverless) 实现，其许可证与使用要求以原项目为准。
- DeepL 名称属于其权利人；本插件不保证非官方接口持续可用。翻译文本会发送到用户配置的服务，并由该服务请求上游。

## 作者

[zyj101511](https://github.com/zyj101511)

## 支持项目

如果这个插件对你有帮助，欢迎到项目的 GitHub 页面点一个 Star ⭐

![star](assets/star.webp)
