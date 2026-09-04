---
name: wechat-devtools-automation-recovery
description: Diagnose and recover WeChat Mini Program DevTools automation, preview, and real-device verification. Use when the build exists but automation cannot connect, times out, or reports "target project window is not opened with automation enabled", or when a page renders in the simulator but goes blank on a real device — covering "自动化连不上", timeout, white/blank screen, stale preview QR, and simulator file-storage errors. Do not use for writing or modifying mini-program source code, business-logic debugging, or routine builds without an automation or rendering failure to recover from.
---

# WeChat DevTools automation recovery

Use this skill when the mini-program build is available but DevTools automation reports connection failure, timeout, or "target project window is not opened with automation enabled", or when a page renders in the simulator but goes blank on a real device.

## Clean artifact before recovery

Do not diagnose a connection, blank-screen, or stale-module report against an old imported bundle. Before every recovery attempt and before handing a manual/real-device check to the user:

1. Locate the project's generated `mp-weixin` artifact (usually `<package-dir>/dist/build/mp-weixin` for uni-app; verify `app.json` and `project.config.json` exist).
2. Clear the DevTools `file` and `compile` caches for that artifact, then rebuild using the project's own build script (detect from `package.json` scripts, usually `build:mp-weixin`; match the project's package manager).

Never use `storage`, `auth`, or `all` as routine recovery because that changes test state; clear those only when the failure is proven to be in that specific subsystem. In the final handoff, name the cleared cache types, build result, artifact path, and the steps that were or were not verified.

Treat the current working tree — not Git `HEAD` — as the source of the recovery run. Inventory changes with `git status --short` without stashing, resetting, or switching them away. Do not report that the latest code is running until a stable compiled marker from the current source is verified in the rebuilt artifact.

## First classify the failure

Do not restart the application backend just because the automator cannot connect. Check the application stack separately, then inspect the DevTools ports:

~~~bash
PORT=9420
lsof -nP -iTCP:$PORT -sTCP:LISTEN
ps -axo pid,ppid,stat,command | grep -i 'wechatwebdevtools|Electron --cli'
~~~

The current macOS DevTools can expose separate IDE and automation ports, and may split listeners by process/address family:

- the CLI parent/Electron process usually listens on the IDE HTTP port (for example `127.0.0.1:9420`) and may return ordinary HTTP `404`;
- the DevTools child owns the `miniprogram-automator` WebSocket on the explicit automation port (for example `*:9422`), often through an IPv6 listener.

Do not treat an HTTP listener on 9420 as proof that automator is available. Test the automation port with all loopback forms before restarting anything:

~~~js
const endpoints = [
  "ws://localhost:" + AUTO_PORT,
  "ws://[::1]:" + AUTO_PORT,
  "ws://127.0.0.1:" + AUTO_PORT,
];
~~~

Prefer `ws://localhost:$AUTO_PORT` or `ws://[::1]:$AUTO_PORT`; if only IPv4 works, use it but keep the process/listener evidence. A failed connection to the IDE HTTP port is a DevTools transport mismatch, not a backend outage.

## Start the right DevTools mode

Use the installed CLI, the absolute release artifact, and a separate automation port. On the current macOS CLI, `--auto-port` is accepted by the parser even though it is missing from `auto --help`; the `miniprogram-automator` package still needs it:

~~~bash
CLI="/Applications/wechatwebdevtools.app/Contents/MacOS/cli"
ARTIFACT="<detected>/dist/build/mp-weixin"
IDE_PORT=9420
AUTO_PORT=9422
"$CLI" auto --project "$ARTIFACT" --port "$IDE_PORT" --auto-port "$AUTO_PORT"
lsof -nP -iTCP:$IDE_PORT -sTCP:LISTEN
lsof -nP -iTCP:$AUTO_PORT -sTCP:LISTEN
~~~

Read `$CLI auto --help` first, but verify the actual WebSocket listener with `lsof`; do not blindly use the IDE HTTP port. If the IDE HTTP server is already running, reuse its known port and choose an unused automation port. Quit only the known DevTools instance when a port must be changed; do not kill an unknown process.

## Verify a real protocol operation

Connection alone is insufficient. Use a bounded script and perform navigation, runtime evaluation, and a screenshot:

~~~js
import automator from "miniprogram-automator";

const mp = await automator.connect({ wsEndpoint: "ws://localhost:9422" });
await mp.reLaunch("/pages/index/index");
const page = await mp.currentPage();
await page.waitFor(1000);
const state = await mp.evaluate(() => ({
  route: getCurrentPages().at(-1)?.route,
  platform: wx.getSystemInfoSync().platform,
}));
await mp.screenshot({ path: "tmp/devtools-automation.png" });
await mp.disconnect();
~~~

In the installed `miniprogram-automator` version, runtime evaluate belongs to `mp`; `page.evaluate` is not available. Put timeouts around connect, navigation, evaluation, and screenshot so a protocol hang becomes explicit evidence.

Navigation methods are not transaction acknowledgements. A timeout or a stale returned Page object is not enough to declare failure: wait briefly, then read `currentPage` and `pageStack`. If the route changed, record the operation as eventually successful and investigate the page lifecycle only if the final state is wrong.

## Blank screenshot with non-zero page geometry

If `currentPage` is the expected route and selector geometry is non-zero but the simulator screenshot is uniformly white, inspect native rendering surfaces before blaming data or API:

- An always-present native component (e.g. `<page-container>`, `video`, `map`, `canvas`) can render an opaque full-screen native surface in the current DevTools simulator, even with overlay disabled, while the Vue/WXML nodes remain measurable underneath it.
- Treat each native surface as a separate lifecycle owner: make empty/guard containers visually inert (transparent, minimal size, zero opacity), and stop or unmount native media on hide/unmount with a static fallback until the first frame is ready.
- Geometry alone is not visual acceptance — rebuild the imported artifact and re-capture the screenshot after the fix.

## Real-device blank pages and mobile JavaScript compatibility

A preview QR identifies one uploaded compiled mini-program snapshot. The phone downloads that frontend package from WeChat; it does not read source files or hot-reload from the developer's Mac. An old QR continues to open its old snapshot; after any source fix, rebuild and generate a new QR.

When DevTools renders a page but the same preview is blank on a phone, inspect shared imports before blaming API data. A module-level browser global can throw before the page mounts and blank every page importing that module — for example `TextDecoder` is not universal across WeChat mobile JavaScript runtimes:

- do not instantiate browser-only globals unconditionally at module load;
- feature-detect them (`typeof globalThis.TextDecoder === "function"`) and provide a fallback;
- add a regression check that the module remains loadable when the global is absent;
- retest every page sharing the import, then clear `file` and `compile`, rebuild, and generate a new preview QR.

Treat this as a runtime-compatibility failure when several pages sharing one import are blank only on the phone. Treat it as stale provenance only when the new compiled marker is absent from the uploaded artifact.

## File-storage error triage

For `saveFile:fail exceeded the maximum size of the file storage limit`:

1. Run the clean-artifact preflight above, especially the DevTools `file` cache cleanup, then retry the same route.
2. Check the app's saved-file list and inspect the actual `saveFile`/`downloadFile` callers. A zero-file/zero-byte list after the error is evidence for DevTools temporary/compiler pollution, not proof that a product path is wrong.
3. Preserve legitimate business files. Remove or change a write only after the clean-cache retry and call-site inspection identify it as the cause.

## When a restart is justified

Restart only the DevTools process when all loopback endpoints fail, the project window is stale, or `lsof` shows a listener owned by an unrelated old DevTools instance. Before stopping it, verify the PID, parent, executable path, and project scope. Relaunch the same absolute project artifact and repeat the protocol operation.

Do not restart the complete application stack for this class of failure unless the independent health check is also failing.

## Evidence to report

Record the project artifact, port listeners, endpoint results, navigation route, runtime state, screenshot path, and whether the result was protocol-verified or only visually checked. Never include tokens, cookies, signatures, user identifiers, or `.env` contents.
