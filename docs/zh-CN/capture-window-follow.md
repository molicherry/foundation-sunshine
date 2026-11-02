# 会话级窗口跟随捕获（Plan4）

## 是什么

默认情况下，Sunshine 捕获**一整块显示器**，所以通过 Moonlight 串流 Playnite 时，Playnite 启动的**窗口化游戏**在手机上只占画面的一小块。

本功能新增一个**本机会话级控制接口**：允许可信的本地客户端（例如 Playnite 插件）把**某一个正在串流的会话**的捕获目标，从"整块显示器"临时切换为"跟随一个窗口"：

- 只有发请求的那个会话受影响，其他会话/其他场景不受影响；
- 不修改 `sunshine.conf`、不修改全局捕获设置；
- 请求失败/窗口不存在时自动回退到显示器捕获，串流不中断；
- 请求关闭或会话结束时自动恢复原行为。

## 工作方式

1. 每个视频会话都有一个 `streaming::capture_target_t`（由 `stream::session::alloc()` 创建），它同时被会话和捕获后端通过 `video::config_t::capture_target` 共享。
2. 本地控制端点向该对象写入 `window_capture_override_t`，并递增 `window_override_generation`。
3. 捕获线程（`src/video.cpp` 的 `captureThread`）发现 generation 变化后触发一次捕获重建（复用现有 reinit 机制）。
4. Windows WGC 后端在 init 时读取 override：
   - `foreground`：跟随当前前台窗口（排除桌面/任务栏/工具窗口/最小化窗口/Sunshine 自身窗口）。为避免游戏退出瞬间误跟随，前台候选还会被过滤：只接受 **Sunshine 本次启动的应用进程树**（Playnite 及其子进程）或 **捕获启用前后 10 秒内新启动的进程**（覆盖 Steam/启动器链路）；两者都不满足时回退显示器捕获；
   - `title`：按窗口标题子串匹配；
   - `pid`：跟随指定进程的窗口。
5. 捕获过程中每 ~250ms 检查一次目标变化：前台切换到另一个合格窗口、目标窗口关闭/最小化、或窗口重新出现时，返回 `reinit` 让捕获线程重建。
6. 如果当前找不到合格窗口，捕获回退到显示器画面，并持续重试；窗口出现后自动切回窗口捕获（并通知客户端）。

> 与原来的"全局窗口捕获"（`capture_target=window` + `window_title`）的区别：那是**全局配置、所有会话生效、需要重启**；本功能是**会话级、运行时生效、只影响一个会话**。

## HTTP API

- 路径：`GET|POST /api/capture/window-follow`
- 认证：与 Web UI 相同的鉴权；**仅允许本机（localhost）请求**；`authenticate()` + `require_localhost()`。
- Windows 之外返回 404（路由不注册）。
- 要求以**用户模式**运行（WGC 不支持 SYSTEM 服务模式），否则返回 409 `wgc_requires_user_mode`。

### GET 查询

```http
GET /api/capture/window-follow
```

响应示例：

```json
{
  "ok": true,
  "feature": "window-follow",
  "supported": true,
  "sessions": [
    {
      "session_id": 1,
      "client_uuid": "…",
      "client_name": "Moonlight",
      "app_name": "Playnite",
      "running": true,
      "control_only": false,
      "override": { "enabled": true, "mode": "foreground", "window_title": "", "process_id": 0 },
      "observed": {
        "active": true,
        "fallback_to_display": false,
        "window_title": "Example Game",
        "process_id": 1234,
        "hwnd": "0x0000000000012345",
        "fallback_reason": ""
      }
    }
  ]
}
```

`supported=false` 时 `reason` 给出原因（如 `wgc_requires_user_mode`）。

### POST 开启 / 关闭 / 查询

```http
POST /api/capture/window-follow
Content-Type: application/json

{
  "action": "enable",            // enable | disable | status（默认 enable）
  "mode": "foreground",          // foreground | title | pid（默认 foreground）
  "window_title": "",            // mode=title 时必填
  "process_id": 0,               // mode=pid 时必填
  "session_id": 0,               // 可选：0=自动选择唯一的活动视频会话
  "client_uuid": ""              // 可选：按客户端证书 UUID 选择会话
}
```

选择会话规则：

- 指定 `session_id` → 按会话 ID 匹配；
- 指定 `client_uuid` → 按客户端证书 UUID 匹配；
- 都不指定：恰好 1 个活动视频会话 → 自动选中；0 个 → 404；多个 → 409 要求显式指定（**绝不**静默修改多个会话）。

成功响应与 GET 相同，并额外包含 `selected_session_id` / `selected_client_uuid`。

### 错误码

| HTTP | error | 含义 |
|---|---|---|
| 400 | `invalid_action` / `invalid_mode` / `window_title_required_for_title_mode` / `process_id_required_for_pid_mode` | 参数错误 |
| 401 | — | 鉴权失败 |
| 403 | — | 非本机请求 |
| 404 | `no_active_video_session` / `session_not_found` / `client_session_not_found` | 没有可控制的会话 |
| 409 | `multiple_video_sessions_specify_session_id` / `wgc_requires_user_mode` | 多会话需显式指定 / 服务模式不支持 |
| 500 | `session_capture_target_unavailable` | 会话缺少捕获目标对象 |

## 与 Playnite 插件联动

配套 Playnite 插件（独立项目）：

- `OnGameStarted`（延迟等待窗口出现）→ `POST {action:enable, mode:foreground}`
- `OnGameStopped` → `POST {action:disable}`
- 无活动串流时会话列表为空 → 端点返回 404，插件静默跳过（本地玩不受影响）

## 兼容性与默认行为

- **默认全部关闭**：没有任何客户端调用该端点时，捕获路径、配置解析、Web UI 与原生 Sunshine 完全一致。
- 不新增必填配置项；`sunshine.conf` 与 `apps.json` 无需迁移。
- 不影响：桌面串流、从 Moonlight 直接启动应用、Web UI 配置、虚拟显示器/Zako Direct（窗口跟随会话会临时改用 WGC 捕获，会话内生效）。
- 多会话场景：本版本要求"恰好一个活动视频会话"才会自动选中；多会话必须显式指定 `session_id`，且窗口跟随只重建该会话所在的共享捕获线程（若多个会话共享同一捕获线程，重建会影响该线程上的会话，请避免多会话混用）。

## 已知限制

1. 窗口跟随基于 **WGC**：需要 Windows 10 1903+、用户模式运行；**独占全屏 / 受 DRM 保护的窗口可能无法捕获**，此时自动回退显示器捕获。
2. 依赖"前台窗口"判断：切换到桌面/任务栏/工具窗口不会切换目标（保持当前窗口）。前台候选受进程过滤限制（Sunshine 启动的应用进程树 / 启动后新进程），普通浏览器等旧进程不会被跟随；游戏退出后即使别的窗口抢到前台也只会回退显示器捕获，不会串到其它应用。需要更严格时用 `pid` 模式。
3. 仅覆盖 **异步共享捕获路径**（Windows 硬件编码器默认走此路径）；软件编码的同步捕获路径不会运行时重建，只会在会话开始时应用一次。
4. HDR：窗口捕获的 HDR 元数据与显示器捕获不完全等价；HDR 串流建议仍用显示器捕获。
5. 游戏窗口标题/前台切换依赖 Windows 窗口状态；某些启动器链（Playnite → 启动器 → 游戏）会短暂跟随启动器窗口，随后切到游戏窗口。

## 手工验证清单

1. 以**用户模式**启动 Sunshine（非 SYSTEM 服务），Windows 10 22H2+，WGC 可用。
2. Moonlight 连接 → 启动 Playnite（Fullscreen）→ 手机看到 Playnite。
3. 本地执行：

   ```powershell
   # 列出会话
   curl.exe -k "https://127.0.0.1:47990/api/capture/window-follow"
   # 开启窗口跟随
   curl.exe -k -X POST "https://127.0.0.1:47990/api/capture/window-follow" -H "Content-Type: application/json" -d '{"action":"enable","mode":"foreground"}'
   # 关闭
   curl.exe -k -X POST "https://127.0.0.1:47990/api/capture/window-follow" -H "Content-Type: application/json" -d '{"action":"disable"}'
   ```

4. 在 Playnite 里启动一个**窗口化**游戏：手机画面应从"桌面/Playnite"切换为该游戏窗口并铺满；日志出现 `Window-follow: capturing window [...]` 或 `Window-follow: switching captured window to [...]`。
5. 关闭游戏：手机画面应恢复为 Playnite 全屏 UI。
6. 不启动游戏时用浏览器/桌面串流：行为与打补丁前一致（默认不影响任何场景）。

## 回滚

```powershell
cd <foundation-sunshine>
git apply -R plan4-window-follow.patch   # 反向应用补丁
# 或 git checkout -- src cmake
```

没有配置迁移，回滚后无需任何清理。

