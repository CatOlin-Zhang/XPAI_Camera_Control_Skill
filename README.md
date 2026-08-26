# XPAI Camera Control

基于 **MCP（Model Context Protocol）** 的本地 IP 摄像头控制系统，支持在局域网内发现、连接和控制创维（Skyworth）IP 摄像头。通过标准化的 MCP stdio 传输协议，可与任意 MCP 客户端（如 Claude Desktop）无缝集成。

---

## 功能概览

| 模块 | 功能 |
|------|------|
| **设备管理** | 局域网设备发现、ONVIF/USB 连接、凭证缓存与云端授权 |
| **流媒体** | RTSP 流获取、截图、本地 MP4 录像、存储管理 |
| **PTZ 云台** | 8 方向移动、变焦控制、物理极限保护、校准归位 |
| **事件监控** | 后台事件监听、告警快照、事件持久化与自动恢复 |
| **补光控制** | 日夜模式切换、补光方式设置（双协议支持） |
| **图像设置** | 亮度、对比度、饱和度、锐度、翻转参数调节 |
| **侦测追踪** | 人形/车辆/区域/移动/越界侦测能力查询与启停 |
| **WebRTC** | 实时 WebRTC 预览流管理 |

共计 **20 个 MCP 工具**，覆盖 8 个核心功能模块。

---

## 技术栈

- **Python 3.12+**
- **MCP Server** — 消息通信协议服务框架
- **ONVIF (onvif-zeep)** — 网络视频设备协议支持
- **OpenCV** — 图像处理与摄像头控制
- **FFmpeg** — 多媒体处理（通过 ffmpeg-downloader 自动管理）
- **PyYAML** — 配置文件解析
- **Requests / Psutil** — HTTP 请求与系统资源监控

---

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 配置摄像头（可选）

在项目根目录创建 `config.yaml`，首次连接时也可由工具自动创建：

```yaml
cameras:
  - name: "camera-01"
    connection_type: "onvif"
    ip: "192.168.1.100"
    port: 8080
    username: "admin"
    password: "your_password"
    rtsp_port: 554
    rtsp_path: "/stream1"
    device_class: "password_required"
```

### 3. 启动 MCP 服务器

```bash
python scripts/mcp_server.py
```

### 4. 配置 MCP 客户端

将以下配置添加到你的 MCP 客户端（如 Claude Desktop 的 `claude_desktop_config.json`）：

```json
{
  "mcpServers": {
    "xpai-camera-control": {
      "command": "python",
      "args": ["C:/path/to/xpai-camera-control/scripts/mcp_server.py"]
    }
  }
}
```

---

## 工作流程

系统遵循分阶段工作流：

1. **Phase 0 — 会话初始化**：检查 `config.yaml` 中已注册的摄像头，加载缓存凭证
2. **Phase 1 — 设备发现**：若无已注册设备，使用 `search_devices()` 扫描局域网
3. **Phase 2 — 连接授权**：缓存凭证 → 用户输入密码 → 云端授权等多层级认证
4. **Phase 3 — 流媒体操作**：获取流 URL、截图、录像
5. **Phase 4 — 扩展功能**：PTZ 控制、事件监控、补光/图像调节、侦测追踪

详细流程参见 [references/WORKFLOW.md](references/WORKFLOW.md)。

---

## 项目结构

```
xpai-camera-control/
├── scripts/
│   ├── toolkit/          # 核心功能模块（编译为 .pyd）
│   ├── mcp_server.py     # MCP 服务器主程序
│   └── _paths.py         # 路径兼容层
├── references/
│   ├── commands/         # 各模块工具 API 参考文档
│   ├── CONFIG.md         # config.yaml 配置 Schema
│   ├── WORKFLOW.md       # 工作流程示例
│   └── EVENT_INTEGRATION.md  # 事件存储消费者契约
├── SKILL.md              # 技能定义与使用说明
├── requirements.txt      # Python 依赖
└── __init__.py
```

---

## MCP 工具列表

| 工具 | 模块 | 说明 |
|------|------|------|
| `get_registered_cameras` | 设备管理 | 读取已注册的摄像头配置 |
| `register_camera` | 设备管理 | 持久化摄像头凭证到 config.yaml |
| `search_devices` | 设备管理 | 局域网设备发现 |
| `connect_device` | 设备管理 | 连接摄像头（支持多协议认证） |
| `disconnect_device` | 设备管理 | 断开连接并释放资源 |
| `get_audio_video_stream` | 流媒体 | 获取 RTSP 流 URL 及元数据 |
| `capture_video_screenshot` | 流媒体 | 截取单帧保存为 JPEG |
| `toggle_recording` | 流媒体 | 启动/停止/查询本地录像 |
| `manage_storage_status` | 流媒体 | 查询磁盘使用或配置存储策略 |
| `control_ptz` | PTZ | 方向移动与变焦控制 |
| `get_ptz_parameters` | PTZ | 读取当前 PTZ 位置和范围 |
| `calibrate_ptz` | PTZ | 物理校准与归位 |
| `stop_ptz` | PTZ | 紧急停止 |
| `manage_camera_events` | 事件 | 事件监听（start/stop/poll/wait） |
| `start_webrtc_stream` | WebRTC | 启动 WebRTC 预览流 |
| `stop_webrtc_stream` | WebRTC | 停止 WebRTC 预览流 |
| `manage_illumination` | 补光 | 查询和设置日夜/补光模式 |
| `manage_image_settings` | 图像 | 查询和设置图像参数 |
| `query_tracking_capabilities` | 侦测 | 查询侦测能力 |
| `set_tracking` | 侦测 | 启用/禁用侦测与追踪 |

---

## 注意事项

- **MCP-only 原则**：所有操作必须通过 MCP 工具进行，禁止直接导入脚本模块
- **连接状态在内存中**：MCP 进程重启后需重新 `connect_device`
- **端口探测**：ONVIF 端口通常不是 80，应使用 `search_devices` 返回的端口
- **中文路径**：已内置处理 Windows 中文路径下的 `cv2.imwrite` 兼容问题
- **PTZ 物理极限**：运动控制内置边界检测，返回 `degraded` 结果表示已达极限

---

## 文档索引

| 文档 | 说明 |
|------|------|
| [SKILL.md](SKILL.md) | 技能定义与完整使用说明 |
| [references/CONFIG.md](references/CONFIG.md) | config.yaml 配置参考 |
| [references/WORKFLOW.md](references/WORKFLOW.md) | 工作流程与调用示例 |
| [references/EVENT_INTEGRATION.md](references/EVENT_INTEGRATION.md) | 事件存储消费者集成指南 |
| [references/commands/](references/commands/) | 各模块 API 参考文档 |

---

## 许可证

本项目版权归作者所有，详见 [GitHub 仓库](https://github.com/CatOlin-Zhang/XPAI_Camera_Control_Skill)。
