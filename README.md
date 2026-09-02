# wechat-devtools-automation-recovery

微信开发者工具自动化连不上、白屏、真机空白时的诊断与恢复 AI agent skill。

先清 file/compile 缓存重新编译再排查（不拿旧产物诊断）；先分类故障——别因自动化连不上就重启后端，要分清 IDE HTTP 端口（9420，会返回 404）和自动化 WebSocket 端口（9422），localhost/`[::1]`/127.0.0.1 三种回环都试；用 `miniprogram-automator` 做协议级验证（connect + reLaunch + evaluate + screenshot，全程带超时）。

覆盖：白屏但几何非零时查原生渲染层（page-container/video/map/canvas 可能盖住 Vue 节点）；真机空白但模拟器正常时查模块级浏览器全局（如 TextDecoder 在手机运行时不存在，要 feature-detect + 降级）；旧 QR 是旧快照，改完代码要重建生成新 QR；文件存储超限时先清 file 缓存重试再查调用方。证据上报不含 token/cookie/.env。

适用于任何微信小程序项目。

## 安装

把 `SKILL.md` 复制到你的 agent skill 目录下即可。

## License

MIT
