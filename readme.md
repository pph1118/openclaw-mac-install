# OpenClaw on macOS + Feishu/Lark (从 0 安装到开机自启)

> 目标：在 macOS（含 Mac mini）上从零安装 OpenClaw（用户态，无 sudo），接入 Feishu/Lark，
> 修复常见 `npm install failed`（`workspace:*`），并把 Gateway 挂载为 LaunchAgent，实现开机自启、关终端后台运行。

---

## 0) 你将得到什么（验收标准）

完成后应满足：

- `openclaw --version` 有输出（CLI 可用）
- `openclaw plugins info feishu` 显示 `Status: loaded`（插件加载成功）
- `openclaw gateway status --deep` 显示 LaunchAgent 已 `loaded/running`（后台服务已托管）
- `openclaw channels status --probe` 不再报 `Gateway not reachable` / WebSocket `1006`

Feishu 插件官方安装命令：`openclaw plugins install @openclaw/feishu`  
macOS Gateway 可通过 `openclaw gateway install` 安装为 LaunchAgent（开机自启）。  
（见官方文档）  

---

## 1) 前置依赖（必须）

### 1.1 安装 Xcode Command Line Tools（提供 git/编译工具链）
```bash
xcode-select --install
