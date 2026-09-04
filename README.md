# wechat-devtools-automation-recovery

构建没问题但自动化连不上、超时、真机白屏时，诊断并恢复微信小程序开发者工具的自动化能力。

## 为什么需要

开发者工具自动化失败往往不是表面看到的原因。IDE 的 HTTP 端口有响应，但自动化的 WebSocket 静默监听在另一个端口上。预览二维码打开的是旧的上传快照而不是最新代码。模拟器里完美渲染的页面在真机上白屏，因为某个浏览器专属全局对象在移动端 JS 运行时里不存在。

这个 skill 走完整的诊断链——端口分类、协议级验证、原生层白屏、移动端 JS 兼容性——在建议重启之前就把根因找到。

## 安装

### 方式 A — 交给 agent 安装

把仓库地址给你的 agent，让它安装：

```
https://github.com/sabrina-fan/wechat-devtools-automation-recovery
```

### 方式 B — 手动安装

把 `wechat-devtools-automation-recovery/` 目录复制到你的 agent skill 目录下（如 `~/.zcode/skills/`）。

## 配置

- **开发者工具 CLI 路径**：macOS 下自动检测（`/Applications/wechatwebdevtools.app/Contents/MacOS/cli`）；可在环境中覆盖。
- **端口**：IDE HTTP 默认 `9420`，自动化 WebSocket 默认 `9422`，均通过 `lsof` 自动发现。
- **miniprogram-automator**：需在项目或全局安装（`npm i miniprogram-automator`）。
- 无需 API key 或凭据。

## 使用方法

当自动化连不上、超时、或模拟器渲染正常但真机白屏时触发。skill 会：

1. 确认产物是最新的（清缓存、重新构建）。
2. 分类故障：IDE HTTP 端口 vs 自动化 WebSocket 端口、旧二维码、原生层白屏、或移动端 JS 兼容性。
3. 用正确的自动化端口启动开发者工具。
4. 跑协议级验证脚本（连接、导航、求值、截图）。
5. 上报证据：端口监听情况、端点结果、路由、截图——不泄露 token 或凭据。

## 兼容性

- **macOS** — 主要平台（使用 `lsof`、macOS 开发者工具 CLI 路径、Computer Use 驱动 GUI）。
- **Windows / Linux** — 开发者工具 CLI 路径和进程检查命令不同，需按平台适配端口检查命令。

## 安全与边界

这个 skill 诊断和恢复自动化/预览故障。不写或改源码、不调试业务逻辑、不做常规构建。绝不记录 token、cookie、签名、用户标识或 `.env` 内容。
