<div align="center">

# agent-browser-cli

面向 Agent 的浏览器感知与控制 CLI，把真实 Chrome 会话变成可复用的标签页扫描、页面 JS、Cookie、CDP 和截图能力。

浏览器感知 · 页面控制 · Chrome 登录态复用 · CDP · 条件等待 · Agent Skill 集成

<p>
  <a href="https://github.com/sleepinginsummer/agent-browser-cli"><img src="https://img.shields.io/badge/CLI-agentbrowsercli-2ea44f" alt="CLI agentbrowsercli"></a>
  <a href="https://github.com/sleepinginsummer/agent-browser-cli/blob/main/LICENSE"><img src="https://img.shields.io/badge/License-MIT-green" alt="License MIT"></a>
  <a href="https://github.com/sleepinginsummer/agent-browser-cli"><img src="https://img.shields.io/badge/sys-win%2Fmac%2Flinux-0078D6?labelColor=0078D6&color=C0C0C0" alt="sys win/mac/linux"></a>
  <a href="https://github.com/sleepinginsummer/agent-browser-cli/releases"><img src="https://img.shields.io/badge/release-v0.3.1--beta.1-orange" alt="release v0.3.1-beta.1"></a>
  <a href="https://github.com/sleepinginsummer/agent-browser-cli/pulls"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen" alt="PRs welcome"></a>
</p>

[AI 一句话安装](#ai-一句话安装) · [手动安装](#手动安装) · [Chrome 扩展](#chrome-扩展) · [更新](#更新) · [更新日志](CHANGELOG.md) · [卸载](#卸载) · [友情链接](#友情链接)

中文 | [English](README_EN.md)

</div>

`agent-browser-cli` 是一个面向 Agent 的浏览器感知与控制工具。它通过 Chrome 扩展连接用户真实浏览器，保留登录态和 Cookie，提供标签页扫描、页面 JS 执行、Cookie 读取、CDP 控制、截图、文件上传、下拉框点击等能力。

本项目不是 Selenium / Playwright。它更适合在已有浏览器会话中辅助 Agent 精确读取页面和执行操作。

## 项目信息

- 当前版本：`0.3.1-beta.1`
- 支持平台：Windows（包括 WSL）/ Mac / Linux
- 浏览器：Chrome，需加载拓展 `assets/tmwd_cdp_bridge`
- Linux 支持前提：本机 Chrome / Chromium 需要支持安装扩展
- WSL 支持前提：需使用 `WSL 2.0.0+`，并建议在 Windows `11 22H2+` 下启用 `networkingMode=mirrored`，以便 WSL 连接宿主机 `localhost` 上的 Chrome 桥接服务

## 致谢

本项目的浏览器控制能力提取并改造自 [GenericAgent](https://github.com/lsdefine/GenericAgent) 项目中的 Web 工具链，包括 `TMWebDriver`、`simphtml` 和 `tmwd_cdp_bridge` 扩展相关思路与实现。

感谢 GenericAgent 项目提供的浏览器桥接、页面简化、CDP 控制和实践 SOP。本仓库在此基础上做了面向独立使用和 CLI 调用的整理与增强。

## AI 一句话安装

```text
请阅读 https://github.com/sleepinginsummer/agent-browser-cli/blob/main/AI_INSTALL.md，按说明安装 CLI、加载 Chrome 扩展，并添加 `skills/agent-browser-cli/SKILL.md`。
```

## 改进内容

- 从 GenericAgent 中拆出浏览器控制能力，使用cli 提供给codex、claude code、opencode使用。GenericAgent浏览器插件不需要重新安装，可以共用同一个插件
- 避免每次命令都重新初始化浏览器连接。
- 新增启动锁，避免多个 CLI 并发启动时重复绑定底层端口。
- 增加skill：`skills/agent-browser-cli/SKILL.md`，提供ai参考使用。
- 若干优化，缩短命令执行时间
- rust实现cli端

## 性能参考

以下为常驻服务已启动、Chrome 扩展已连接时的实测参考，实际耗时会受页面复杂度、网络、Chrome 状态和返回数据量影响。

| 操作 | 参考耗时 |
| --- | --- |
| 打开百度标签页 | 约 `0.10s` |
| 注入 JS 输入关键词并点击搜索 | 约 `0.27s` |
| 打开百度并搜索“小猫”合计 | 约 `0.37s` |
| `scan --tab --text-only` 读取页面文本 | 约 `0.04-0.12s` |
| `exec 'return document.title'` 注入简单 JS | 约 `0.04-0.12s` |
| `exec 'return document.body.innerText'` 读取正文 | 多数 `0.04-0.05s`，偶发约 `0.30s` |
| 查询 DOM 链接列表 | 约 `0.27-0.36s` |
| `exec --monitor` 页面变化摘要 | 约 `0.72-0.88s` |

一般判断：普通读页面和简单 JS 注入是 `50ms` 级；复杂 DOM 查询主要取决于页面结构和返回数据量，常见约 `300ms`；`--monitor` 会额外生成页面变化摘要，通常接近 `0.8s`。

与原 Python 调用链的参考对比：

| 对比项 | Python 版本 | Rust CLI 版本 |
| --- | --- | --- |
| 启动方式 | 每次调用更容易触发 Python 进程、模块加载和连接初始化开销 | CLI 命令复用常驻服务，避免重复初始化浏览器连接 |
| 简单读页面 / JS 注入 | 通常受进程启动和 Python 调用链影响，延迟更不稳定 | 常见 `0.04-0.12s` |
| 连续多次调用 | 多次短命令开销更明显 | 更适合 Agent 高频调用 |

该对比只用于说明架构差异带来的性能趋势；具体耗时仍取决于页面复杂度、Chrome 状态和返回数据量。

## 目录结构

```text
.
├── Cargo.toml                    # Rust 工程配置
├── src/                          # Rust CLI / 常驻服务 / bridge
├── assets/tmwd_cdp_bridge/       # Chrome MV3 扩展
├── assets/simphtml_opt.js        # 页面简化脚本
├── assets/simphtml_find_list.js  # 列表识别脚本
├── npm/                          # npm 启动脚本
└── skills/agent-browser-cli/     # skill
```

## 手动安装

### npm 安装

```bash
npm install -g @sleepinsummer/agent-browser-cli
agent-browser-cli tabs
```

### 本地源码构建

```bash
cargo build --release
./target/release/agent-browser-cli tabs
```


## Chrome 扩展

1. 推荐从 [最新 Release](https://github.com/sleepinginsummer/agent-browser-cli/releases/latest) 下载 `chrome-extensions.zip`，下载后解压，Chrome 打开 `chrome://extensions`，开启“开发者模式”，点击“加载已解压的扩展程序”，选择解压后的 `tmwd_cdp_bridge` 目录。

2. 本地源码构建时，也可以直接加载扩展目录：

```text
assets/tmwd_cdp_bridge
```

3. Chrome 需要至少打开一个正常网页标签页，不要只停留在 `about:blank` 或 `chrome://` 页面。
4. 扩展连接后会在页面右侧显示 Chrome 插件提示角标。角标支持拖动位置，鼠标悬浮时展开；10 秒无命令后自动隐藏，也可以点击 `本次隐藏` 手动隐藏，本次服务连接周期内不再显示，约 300 秒服务断开并下次重连后恢复。

###  自定义Chrome插件的ws监听端口

- `18765`：默认插件 WebSocket 端口，Chrome 扩展连接使用，可通过 `agent-browser-cli set-extension-port <port>` 修改。
- `18767`：CLI HTTP API 端口，供 CLI 复用会话，不能作为插件端口使用。

CLI 修改插件端口：

```bash
agent-browser-cli set-extension-port 18766
```

该命令会写入配置文件；如果 daemon 正在运行，会自动重启 daemon，让新端口立即生效。

也可以手动修改配置文件。配置文件位于 `~/.agent-browser-cli/config.json`，不存在时会自动生成：

```json
{
  "extension_port": 18765
}
```

手动修改示例：

```json
{
  "extension_port": 18766
}
```

手动改配置后需要执行 `agent-browser-cli restart`，daemon 才会按新端口重新监听。

Chrome 插件 popup 中也可以修改插件端口并立即重连。插件端口必须和 CLI 配置中的 `extension_port` 一致。



## 快速自检

```bash
agent-browser-cli tabs
agent-browser-cli open https://www.baidu.com
```

成功时会返回：

```json
{
  "ok": true,
  "result": {
    "status": "success",
    "metadata": {
      "tabs_count": 1
    }
  }
}
```

## 常用命令

README 只保留快速入口；完整命令和浏览器操作 SOP 见 [skills/agent-browser-cli/SKILL.md](./skills/agent-browser-cli/SKILL.md)。

```bash
agent-browser-cli tabs
```

## 更新

完整版本记录见 [CHANGELOG.md](./CHANGELOG.md)。

ai一句话更新
```text
请阅读 https://github.com/sleepinginsummer/agent-browser-cli/blob/main/AI_INSTALL.md，按说明更新 CLI、重新下载插件zip让用户指定位置，用户手动加载 Chrome 扩展，并更新相关 SKILL.md`。
```

如果 Chrome 扩展有更新，在 `chrome://extensions` 中重新下载zpi覆盖，然后重新加载 `assets/tmwd_cdp_bridge` 扩展。


## 卸载

先停止常驻服务：

```bash
agent-browser-cli stop
```

然后按需清理：

```bash
rm -f .agent-browser-cli.log .agent-browser-cli.lock
rm -rf ~/.agents/skills/agent-browser-cli
```

最后在 Chrome 扩展管理页中移除 `TMWD CDP Bridge` 扩展，或删除已加载的 `assets/tmwd_cdp_bridge` 扩展配置。



## 友情链接

- [LINUX DO - 新的理想型社区](https://linux.do/)
- [GenericAgent--复旦团队研发|仅仅~3K行代码 Self-Evolving Agent](https://github.com/lsdefine/GenericAgent/tree/main)

## 许可证

MIT License. See [LICENSE](./LICENSE).
