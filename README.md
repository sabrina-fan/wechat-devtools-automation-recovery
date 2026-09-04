# wechat-devtools-automation-recovery

Diagnose and recover WeChat Mini Program DevTools automation when the build is fine but the automator cannot connect, times out, or pages go blank on real devices.

## Why

DevTools automation failures are rarely what they look like. The IDE HTTP port responds while the automation WebSocket silently listens on a different port. A preview QR opens a stale uploaded snapshot instead of your latest code. A page that renders perfectly in the simulator goes blank on a phone because of a browser-only global that isn't available in the mobile JavaScript runtime.

This skill walks through the full diagnosis chain — port classification, protocol-level verification, native-layer blank screens, and mobile JS compatibility — before recommending a restart.

## Install

### Option A — let your agent install it

Give your agent this repo URL and ask it to add the skill:

```
https://github.com/sabrina-fan/wechat-devtools-automation-recovery
```

### Option B — manual

Copy the `wechat-devtools-automation-recovery/` directory into your agent's skills folder (e.g. `~/.zcode/skills/`).

## Configuration

- **DevTools CLI path**: auto-detected on macOS (`/Applications/wechatwebdevtools.app/Contents/MacOS/cli`); override by setting the path in your environment.
- **Ports**: IDE HTTP defaults to `9420`, automation WebSocket defaults to `9422`. Both are auto-discovered via `lsof`.
- **miniprogram-automator**: must be installed in the project or globally (`npm i miniprogram-automator`).
- No API keys or credentials required.

## Usage

Trigger it when automation fails to connect, times out, or a page renders in the simulator but goes blank on a real device. The skill will:

1. Verify the artifact is current (clean caches, rebuild if needed).
2. Classify the failure: IDE-HTTP vs automation-WebSocket port, stale QR, native-layer blank screen, or mobile JS compatibility.
3. Start the right DevTools mode with the correct automation port.
4. Run a protocol-level verification script (connect, navigate, evaluate, screenshot).
5. Report evidence: port listeners, endpoint results, route, screenshot — without exposing tokens or credentials.

## Compatibility

- **macOS** — primary platform (uses `lsof`, macOS DevTools CLI paths, Computer Use for GUI).
- **Windows / Linux** — DevTools CLI path and process inspection commands differ; adapt the port-check commands to the local platform.

## Security & Boundary

This skill diagnoses and recovers automation/preview failures. It does not write or modify source code, debug business logic, or perform routine builds. It never logs tokens, cookies, signatures, user identifiers, or `.env` contents.
