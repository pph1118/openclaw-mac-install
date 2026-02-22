# macOS 从 0 安装 OpenClaw + 配置 Feishu/Lark + 修复 Feishu 插件安装失败（workspace:*）+ Gateway 开机自启（LaunchAgent）

下面这份是从 0 开始在 macOS（含 Mac mini）安装 OpenClaw、解决“飞书插件 install failed（workspace:*）”、并最终把 Gateway 挂到 LaunchAgent 实现开机自启/关终端后台常驻的完整教程。所有关键步骤都按你反复遇到的坑做了“必过分支”。

## 0. 目标验收标准（最后你要看到什么）

完成后你至少要满足：

- `openclaw --version` 有输出（CLI 可用）
- `openclaw plugins info feishu` 显示 `Status: loaded`（插件可加载）
- `openclaw gateway status --deep` 显示 LaunchAgent 已 loaded/running（后台服务）
- `openclaw channels status --probe` 不再报 “Gateway not reachable / 1006”（网关可达）

`status --deep` / `channels status --probe` 是官方推荐的健康检查入口。

## 1. 先选安装方式：推荐用户态 install-cli.sh（最稳，避开 sudo/Homebrew）

OpenClaw 官方提供两类安装器：

- `install.sh`：偏“全局 npm + 可能触发 Homebrew/依赖安装”
- `install-cli.sh`：用户态前缀安装，会在 `~/.openclaw` 下装独立 Node，然后用 `npm --prefix <prefix>` 安装 OpenClaw，并生成 `~/.openclaw/bin/openclaw wrapper`

推荐你用：

```bash
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

（这一步本质流程见官方 “Installer Internals”。）

## 2. 必做前置：确保 macOS 有 Xcode Command Line Tools（否则 git/npm 会炸）

你之前遇到过的典型报错：`xcode-select: note: No developer tools were found...`，这会导致 npm 在遇到 git 依赖时直接失败。

安装（会弹系统对话框）：

```bash
xcode-select --install
```

验收：

```bash
xcode-select -p
git --version
```

## 3. 可选但强烈建议：把 GitHub 的 SSH 依赖强制改写为 HTTPS（避免 “unknown git error / permission denied”）

有些依赖会用 `ssh://git@github.com/...`，这会要求你配置 GitHub SSH key（否则 npm 会在 `git ls-remote` 处挂）。GitHub 官方 SSH 文档说明了这条路径。

如果你不想折腾 SSH key，直接做 git URL rewrite（通用技巧）即可：

```bash
git config --global url."https://github.com/".insteadOf ssh://git@github.com/
git config --global url."https://github.com/".insteadOf git@github.com:
```

## 4. 安装 OpenClaw（用户态）

```bash
curl -fsSL --proto '=https' --tlsv1.2 https://openclaw.ai/install-cli.sh | bash -s -- --version 2026.2.19-2
```

安装完成后，必须把 CLI 写入 PATH（永久）：

```bash
grep -q 'OpenClaw (CLI path)' ~/.zshrc 2>/dev/null || cat >> ~/.zshrc <<'EOF_ZSH'

# OpenClaw (CLI path)
export PATH="$HOME/.openclaw/bin:$PATH"
EOF_ZSH
source ~/.zshrc
command -v openclaw
openclaw --version
```

## 5. 关键：把 OpenClaw 自带的 Node/npm 加入 PATH（否则向导装插件会报 spawn npm ENOENT）

`install-cli.sh` 会把 Node 放到 `<prefix>/tools/node-v<version>` 并提供 `<prefix>/tools/node` 链接。  
但是你的 shell PATH 不一定包含它，导致插件安装时找不到 npm / node。

把 bundled node/npm 写入 `~/.zshrc`（永久）：

```bash
grep -q 'OpenClaw bundled Node' ~/.zshrc 2>/dev/null || cat >> ~/.zshrc <<'EOF_ZSH'

# OpenClaw bundled Node (for plugin installs)
export PATH="$HOME/.openclaw/tools/node/bin:$PATH"
EOF_ZSH
source ~/.zshrc
command -v node && node -v
command -v npm && npm -v
```

## 6. 跑一次向导（可选，但推荐：它会写好 config 并提示你配置飞书）

```bash
openclaw onboard
```

官方飞书文档也推荐通过 onboarding 添加飞书渠道，并在完成后用 `gateway status` / `logs --follow` 验证。

## 7. 飞书插件安装（重点：处理 “npm install failed” 的 workspace:* 坑）

### 7.1 先按官方方式安装飞书插件

官方文档明确飞书需要独立插件：

```bash
openclaw plugins install @openclaw/feishu
```

### 7.2 如果你遇到：Plugin install failed: npm install failed:（向导不回显原因）

直接进入插件目录用 verbose 把真实错误挖出来：

```bash
cd ~/.openclaw/extensions/feishu
rm -rf node_modules package-lock.json
npm install --omit=dev --ignore-scripts --loglevel verbose
```

OpenClaw 官方说明：`openclaw plugins install` 安装依赖时会用 `npm install --ignore-scripts`（不跑 lifecycle scripts），你手动装依赖也建议保持一致。

### 7.3 如果 verbose 显示：Unsupported URL Type "workspace:": workspace:*

这就是你反复遇到的核心问题：`workspace:*` 在 npm 场景下会触发 `EUNSUPPORTEDPROTOCOL`（npm 自己也有相关 issue/讨论）。

通用修复：把插件 `package.json` 里所有 `workspace:*` 依赖移除（或改成普通 semver），然后再装依赖。

直接用一键 patch（移除所有 `workspace:*`）：

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
            deps.pop(name, None)
            changed = True
    if deps: d[k] = deps
    else: d.pop(k, None)
p.write_text(json.dumps(d, indent=2) + "\n")
print("patched (removed workspace:* deps)" if changed else "no workspace:* found")
PY

rm -rf node_modules package-lock.json
npm install --omit=dev --ignore-scripts
```

验收依赖可用（以你之前缺的 `@sinclair/typebox` 为例）：

```bash
node -e "require('@sinclair/typebox'); console.log('typebox ok')"
```

### 7.4 插件状态验收

```bash
openclaw plugins info feishu
```

看到 `Status: loaded` 即插件加载 OK。

## 8. 处理 “duplicate plugin id detected (feishu)”（有些机器会遇到）

如果你看到类似：
`plugin feishu: duplicate plugin id detected; later plugin may be overridden`  
这是近期已知 bug/冲突现象（官方仓库已有 issue）。

处理原则：系统里只保留一个 feishu 插件入口（要么用你 `~/.openclaw/extensions/feishu` 这份，要么用全局/内置那份，别同时存在）。

最稳的做法是：卸载冲突来源（用官方 `plugins uninstall`）。

```bash
openclaw plugins uninstall feishu
# 然后再重新安装你要保留的那一份
openclaw plugins install @openclaw/feishu
```

## 9. 启动 Gateway（先前台验证，再挂 LaunchAgent）

### 9.1 先前台跑起来（立即验证端口可达）

```bash
openclaw gateway
```

新开一个终端窗口跑：

```bash
openclaw channels status --probe
```

只要 Gateway 真在 `ws://127.0.0.1:18789` 监听，probe 就不会再报 “1006 abnormal closure”。

### 9.2 挂到开机自启（LaunchAgent），关终端也常驻

macOS 下 OpenClaw Gateway 可以作为 LaunchAgent 由 CLI 安装/管理。

执行：

```bash
openclaw gateway install
openclaw gateway restart
openclaw gateway status --deep
```

注意：近期有一个已知问题：`gateway install` 可能错误地使用系统/Homebrew 的 node，而不是 `install-cli.sh` 放在 `~/.openclaw/tools/node/bin/node` 的 bundled node，导致“CLI 能跑，服务启动环境不一致”。如果你装的是用户态版本，建议确保 PATH 里 bundled node 在最前，然后再 `gateway uninstall && gateway install` 重装服务。

服务层面最终验收（要看到 LaunchAgent loaded/running）：

```bash
openclaw gateway status --deep
```

## 10. 配置飞书渠道（让 “enabled, not configured” 变成可用）

官方飞书文档说明：它通过 WebSocket 事件订阅接收消息，不需要暴露公网 webhook。

推荐路径（向导式）：

```bash
openclaw onboard
```

向导会引导你：

- 创建飞书应用并拿到 App ID/Secret
- 写入 OpenClaw 配置
- 启动/重启网关并验证日志

最后验收：

```bash
openclaw gateway status --deep
openclaw channels status --probe
openclaw logs --follow
```

## 11. 常用故障速查（你这套环境最常见的 4 个）

- `spawn npm ENOENT` -> 没把 `~/.openclaw/tools/node/bin` 加入 PATH（见第 5 节）
- `npm install failed`（向导没细节） -> 去插件目录 `npm install --loglevel verbose` 抓真因（见第 7 节）
- `EUNSUPPORTEDPROTOCOL workspace:*` -> patch 掉 `workspace:*`（见第 7.3 节）
- `Gateway not reachable / 1006` -> Gateway 没跑或服务未加载；先 `openclaw gateway`，再 `openclaw gateway install`（见第 9 节）

## 最终“一键化顺序清单”（你可以直接照抄跑）

```bash
# 0) CLT（保证 git/npm 编译链路不炸）
xcode-select --install

# 1) GitHub SSH -> HTTPS（可选但强烈建议）
git config --global url."https://github.com/".insteadOf ssh://git@github.com/
git config --global url."https://github.com/".insteadOf git@github.com:

# 2) 安装 OpenClaw（用户态）
curl -fsSL https://openclaw.ai/install-cli.sh | bash

# 3) PATH（CLI + bundled node/npm）
grep -q 'OpenClaw (CLI path)' ~/.zshrc 2>/dev/null || cat >> ~/.zshrc <<'EOF_ZSH'

# OpenClaw (CLI path)
export PATH="$HOME/.openclaw/bin:$PATH"

# OpenClaw bundled Node (for plugin installs)
export PATH="$HOME/.openclaw/tools/node/bin:$PATH"
EOF_ZSH
source ~/.zshrc

# 4) 装飞书插件
openclaw plugins install @openclaw/feishu || true

# 5) 如失败：修 workspace:* 并装依赖
cd ~/.openclaw/extensions/feishu
rm -rf node_modules package-lock.json
npm install --omit=dev --ignore-scripts --loglevel verbose || true

python3 - <<'PY'
import json, pathlib
p = pathlib.Path("package.json")
d = json.loads(p.read_text())
for k in ["dependencies","devDependencies","peerDependencies","optionalDependencies"]:
    deps = d.get(k) or {}
    for name, ver in list(deps.items()):
        if isinstance(ver, str) and ver.startswith("workspace:"):
            deps.pop(name, None)
    if deps: d[k]=deps
    else: d.pop(k, None)
p.write_text(json.dumps(d, indent=2) + "\n")
PY
rm -rf node_modules package-lock.json
npm install --omit=dev --ignore-scripts

# 6) 启动并挂载开机自启
openclaw gateway install
openclaw gateway restart
openclaw gateway status --deep

# 7) 跑向导配置飞书凭证
openclaw onboard

# 8) 最终验收
openclaw plugins info feishu
openclaw channels status --probe
```

如果你愿意把你当前这台机子执行 `openclaw gateway status --deep` 的输出贴出来，就可以确认状态啦
