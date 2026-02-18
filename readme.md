# macOS 从 0 安装 OpenClaw + 配置 Feishu/Lark + 修复 Feishu 插件安装失败（workspace:*）+ Gateway 开机自启（LaunchAgent）

本文走**用户态安装（`install-cli.sh`）**：不依赖 `sudo` / Homebrew，安装到 `~/.openclaw`，自带 Node 运行时。  
Feishu/Lark 通过 **WebSocket 事件订阅**接收消息，无需暴露公网 webhook；需安装 `@openclaw/feishu` 插件。  
macOS 上 Gateway 可由 `openclaw gateway install` 安装为 LaunchAgent（开机登录自启、崩溃可重启）。

## 0) 一次性前置：安装 Xcode Command Line Tools（必须）

没有 CLT 时，`npm` 遇到 git/编译链路会失败（常见于依赖包含 git 拉取或原生编译扩展）。

```bash
xcode-select --install
```

验收：

```bash
xcode-select -p
git --version
```

## 1) 安装 OpenClaw（用户态，无 sudo）

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

说明：`install-cli.sh` 会安装 Node + OpenClaw 到本地前缀 `~/.openclaw`，不需要 root。

## 2) 配置 PATH（必须做，否则必踩“找不到 openclaw / spawn npm ENOENT”）

### 2.1 永久加入 OpenClaw CLI 到 PATH

```bash
grep -q 'OpenClaw (CLI path)' ~/.zshrc 2>/dev/null || cat >> ~/.zshrc <<'EOF_ZSH'

# OpenClaw (CLI path)
export PATH="$HOME/.openclaw/bin:$PATH"
EOF_ZSH

source ~/.zshrc
command -v openclaw
openclaw --version
```

### 2.2 永久加入 OpenClaw 自带 Node/npm 到 PATH（插件安装强依赖）

```bash
grep -q 'OpenClaw bundled Node' ~/.zshrc 2>/dev/null || cat >> ~/.zshrc <<'EOF_ZSH'

# OpenClaw bundled Node (for plugin installs)
export PATH="$HOME/.openclaw/tools/node/bin:$PATH"
EOF_ZSH

source ~/.zshrc
command -v node && node -v
command -v npm && npm -v
```

## 3)（可选）zsh 允许交互式 `#` 注释（避免粘贴脚本时报 `command not found: #`）

```bash
grep -q 'INTERACTIVE_COMMENTS' ~/.zshrc 2>/dev/null || cat >> ~/.zshrc <<'EOF_ZSH'

# allow comments in interactive zsh
setopt INTERACTIVE_COMMENTS
EOF_ZSH

source ~/.zshrc
```

## 4) 安装 Feishu/Lark 插件（官方方式）

Feishu 渠道需要安装插件 `@openclaw/feishu`。

```bash
openclaw plugins install @openclaw/feishu
openclaw plugins info feishu
```

安全/行为说明：`openclaw plugins install` 会以 `npm install --ignore-scripts` 的方式安装依赖（不跑 lifecycle scripts）。

## 5) 若 Feishu 插件安装失败：一键修复 `workspace:*`（`EUNSUPPORTEDPROTOCOL`）

### 5.1 进入插件目录，打印真实 npm 错误（verbose）

```bash
export PATH="$HOME/.openclaw/tools/node/bin:$PATH"
hash -r

cd ~/.openclaw/extensions/feishu
rm -rf node_modules package-lock.json
npm install --omit=dev --ignore-scripts --loglevel verbose
```

如果看到 `Unsupported URL Type "workspace:" workspace:*` / `EUNSUPPORTEDPROTOCOL`，这是 npm 在非 monorepo 场景无法解析 `workspace:` 协议的典型报错。

### 5.2 Patch：移除 `workspace:*` 依赖并重装

```bash
cd ~/.openclaw/extensions/feishu

python3 - <<'PY'
import json, pathlib
p = pathlib.Path("package.json")
d = json.loads(p.read_text())
changed = False

for k in ["dependencies","devDependencies","peerDependencies","optionalDependencies"]:
    deps = d.get(k) or {}
    for name, ver in list(deps.items()):
        if isinstance(ver, str) and ver.startswith("workspace:"):
            print(f"remove {k}.{name} = {ver}")
            deps.pop(name, None)
            changed = True
    if deps:
        d[k] = deps
    else:
        d.pop(k, None)

p.write_text(json.dumps(d, indent=2) + "\n")
print("patched package.json (removed workspace:* deps)" if changed else "no workspace:* deps found")
PY

rm -rf node_modules package-lock.json
npm install --omit=dev --ignore-scripts
```

验收（可选探针）：

```bash
node -e "require('@sinclair/typebox'); console.log('typebox ok')"
```

插件状态验收：

```bash
openclaw plugins info feishu
```

## 6) 启动 Gateway（先跑起来再做自启）

### 6.1 前台启动（用于立即验证）

```bash
openclaw gateway
```

另开终端 probe：

```bash
openclaw channels status --probe
```

## 7) Gateway 挂载 LaunchAgent（开机自启 + 关终端后台运行）

macOS 下 Gateway 可以作为 LaunchAgent 托管；CLI 支持 `openclaw gateway install`。

```bash
openclaw gateway install
openclaw gateway restart
openclaw gateway status --deep
```

若你使用 `install-cli.sh`（用户态 bundled Node），但发现 `gateway install` 生成的 plist 使用了系统/Homebrew 的 node，可参考已报告问题的处理思路：确保 `PATH` 中 `~/.openclaw/tools/node/bin` 优先，然后 `openclaw gateway uninstall && openclaw gateway install` 重装服务。

## 8) 配置 Feishu（从 `enabled/not configured` 到可用）

Feishu/Lark 通过 WebSocket 事件订阅接收消息；按官方文档在飞书开放平台完成应用创建、权限与事件订阅配置后，把对应信息写入 OpenClaw 配置。

推荐走向导配置（最省事）：

```bash
openclaw onboard
```

配置完成后重启并验证：

```bash
openclaw gateway restart
openclaw channels status --probe
openclaw logs --follow
```

## 9) 最终通关验收（复制粘贴执行）

```bash
openclaw --version
openclaw plugins info feishu
openclaw gateway status --deep
openclaw channels status --probe
```
