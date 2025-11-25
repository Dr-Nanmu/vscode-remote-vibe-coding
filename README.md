# 远程开发网络与插件使用实战手册

> 目标：A 仅靠本机代理，让 VSCode 远程也能用各类 AI 插件；B 让远程服务器自己出网并可按需/自启代理（以 mihomo/clash-for-AutoDL 为例）。
> 
> 
> 读者画像：本地 Windows/macOS 已有“梯子”，远程（AutoDL 容器/云 GPU/服务器）默认直连。
> 
> 约定：文中端口以 `7890`（HTTP）为例，请替换为你的实际端口。
> 

---

## 目录

- 两种诉求（可解耦）
- Part A：VSCode 仅靠本机代理也能用 AI 插件
    - A1. 最小必要设置（settings.json）
    - A2.（仅 Codex）同步 `.codex` 以便远程调试时“读取远程文件”
- Part B：远程服务器按需走代理
    - B0. 仓库与基础安装
    - B1. 使用去注入版 `start.sh`（备份→替换）
    - B2. 新建自启脚本 `/etc/profile.d/99-clash-autostart.sh`
    - B3. 新建代理函数库 `~/.proxy_funcs.sh`
    - B4. 修改 `~/.bashrc`
    - B5. 开机后验证步骤（Ubuntu）
- 常见问题与排查

---

## 两种诉求（可解耦）

1. **只想在 VSCode 远程时照常用 AI 插件**：扩展运行在**本机 UI 侧**，走本机代理，远程是否能出国无关 → 做 **Part A** 即可。
2. **远程也要能 `pip/git/wget` 出网**：在远程安装 mihomo/Clash，并通过环境变量为命令行/服务提供代理 → 做 **Part B**。

> 多数人 A 就够用；当你要在远程拉包/访问外网时，再补做 B。
> 

---

## Part A：VSCode 仅靠本机代理也能用 AI 插件

**核心**：让联网扩展跑在 **UI（本机）** 侧；必要时让 VSCode 自身走本机 HTTP 代理。

### A1. 最小必要设置（settings.json）

> 仅保留与“插件可用”强相关的键；端口以 7890 为例，扩展 ID 请按你实际安装填写。
> 

```json
{
  "http.proxy": "http://127.0.0.1:7890",
  "http.proxyStrictSSL": false,
  "remote.extensionKind": {
    "openai.chatgpt": ["ui"],
    "google.geminicodeassist": ["ui"],
    "Anthropic.claude-code-assist": ["ui"]
  }
}
```

**如何找到扩展的 ID（Identifier）**：在 VSCode 里打开“扩展”侧栏 → 搜索并点开目标扩展 → **详情（Details）** 页 → 找到 **Identifier** 字段（形如 `publisher.name`），点击旁边的复制按钮或选中复制该值，填入上面的键名处。

### A2.（仅 Codex）同步 `.codex` 以便远程调试时“读取远程文件”

- 目的：让 **Codex** 在 VSCode 远程调试时更好地**索引与解析远程工程文件**，方便断点/跳转与联动调试。
- 做法（仅当你使用 **Codex** 时执行）：将本地 `.codex`（或扩展设置指向的目录）同步到远程主目录：
    
    ```bash
    scp -r ~/.codex user@server:~/.codex
    # Windows 可用 WinSCP / Termius / MobaXterm 图形化上传
    ```
    
- 其他 AI 插件通常**不需要**这一步。
- **如果还想让Codex用远程虚拟环境测试则需要配置Part B并取消扩展跑在UI（本机）侧的设置。**

---

## Part B：让远程服务器按需走代理

**目标**：登录后自动拉起 mihomo；是否启用代理由 `proxy_on / proxy_off` 控制；也可用 `autoproxy_maybe` 跟随端口自动开关环境变量。

### B0. 仓库与基础安装

按照官方仓库完成基础安装与订阅配置：

- 仓库链接：`https://github.com/VocabVictor/clash-for-AutoDL`

> 注：仓库本身的问题以其 README 为准；本文只在后续给出 start.sh 的替换方案。
> 

### B1. 使用去注入版 `start.sh`（备份→替换）

- 仓库中提供了一份“去注入版” `start.sh`（已移除向 `~/.bashrc` 注入的逻辑）。
- 操作：先备份原 `start.sh`，再将该文件替换为仓库提供的版本，并确保具有可执行权限。

### B2. 新建自启脚本 `/etc/profile.d/99-clash-autostart.sh`

一键粘贴下面的命令设置登录时自动拉起 clash：

```bash
sudo bash -lc 'cat > /etc/profile.d/99-clash-autostart.sh <<"EOF"
#!/bin/sh
# 在你每次登录时检查 mihomo 是否已运行，未运行则启动。不会重复起。
if ! pgrep -f "mihomo-linux" >/dev/null 2>&1; then
  (
    cd /root/clash-for-AutoDL || exit 0
    export SKIP_BASHRC_INJECT=1
    /bin/bash -lc '''source ./stop.sh >/dev/null 2>&1 || true; printf "n
" | source ./start.sh''' >> /root/clash-for-AutoDL/boot.log 2>&1
  ) &
fi
EOF
chmod +x /etc/profile.d/99-clash-autostart.sh'
```

### B3. 新建代理函数库 `~/.proxy_funcs.sh`

一键粘贴下面的命令设置手动/自动代理开关函数：

```bash
cat > ~/.proxy_funcs.sh <<'EOF'
__PROXY_PORT="${__PROXY_PORT:-7890}"
__PROXY_HOST="${__PROXY_HOST:-127.0.0.1}"

proxy_on() {
  local is_quiet="${1:-false}"
  export http_proxy="http://${__PROXY_HOST}:${__PROXY_PORT}"
  export https_proxy="$http_proxy"
  export no_proxy="127.0.0.1,localhost"
  export HTTP_PROXY="$http_proxy"
  export HTTPS_PROXY="$https_proxy"
  export NO_PROXY="$no_proxy"
  [ "$is_quiet" != "true" ] && echo -e "\033[0;32m[√] 代理已开启 -> $http_proxy\033[0m"
}

proxy_off() {
  local is_quiet="${1:-false}"
  unset http_proxy https_proxy no_proxy HTTP_PROXY HTTPS_PROXY NO_PROXY
  [ "$is_quiet" != "true" ] && echo -e "\033[0;31m[×] 代理已关闭\033[0m"
}

# 只要端口在监听就自动开代理，否则自动关
autoproxy_maybe() {
  # 优先 ss
  if command -v ss >/dev/null 2>&1; then
    if ss -lnt 2>/dev/null | grep -q ":${__PROXY_PORT} "; then
      proxy_on true; return
    fi
  fi
  # 回退 lsof
  if command -v lsof >/dev/null 2>&1; then
    if lsof -iTCP -sTCP:LISTEN -P -n 2>/dev/null | grep -q ":${__PROXY_PORT}"; then
      proxy_on true; return
    fi
  fi
  # 回退 netstat
  if command -v netstat >/dev/null 2>&1; then
    if netstat -lntp 2>/dev/null | grep -q ":${__PROXY_PORT} "; then
      proxy_on true; return
    fi
  fi
  proxy_off true
}
EOF
```

### B4. 修改 `~/.bashrc`

一键粘贴下面的命令备份原文件并在末尾追加修改：

```bash
# 备份（可选）
cp -a ~/.bashrc ~/.bashrc.bak.$(date +%F-%H%M%S) 2>/dev/null || true

# 仅当未添加过时，在 ~/.bashrc 末尾追加三行
if ! grep -Fq '[ -f ~/.proxy_funcs.sh ] && . ~/.proxy_funcs.sh' ~/.bashrc; then
  cat >> ~/.bashrc <<'EOF'

# --- proxy auto begin ---
[ -f ~/.proxy_funcs.sh ] && . ~/.proxy_funcs.sh
proxy_off true 2>/dev/null || true
autoproxy_maybe 2>/dev/null || true
# --- proxy auto end ---
EOF
fi

# 让改动立刻生效
source ~/.bashrc
```

### B5. 重启一次自启效果检查步骤（Ubuntu）

先安装依赖（Ubuntu/Debian）：

```bash
sudo apt-get update
sudo apt-get install -y iproute2 lsof net-tools curl
```

然后按顺序执行自查：

```bash
# 1) 进程与端口（mihomo 是否已在监听）
pgrep -af 'mihomo|clash' || echo "未看到 mihomo 进程"
ss -lnt | awk 'NR==1 || /:7890[[:space:]]/'

# 2) 函数是否可用
type proxy_on proxy_off autoproxy_maybe

# 3) 切换变量并查看三元组
proxy_off; echo "$http_proxy | $https_proxy | $NO_PROXY"
proxy_on;  echo "$http_proxy | $https_proxy | $NO_PROXY"

# 4) 外网连通性（示例）
curl -I -m 8 https://www.google.com || true
```

**期望**：

- `ss` 显示 `LISTEN *:7890`；
- `type` 显示三者为 `function`；
- `proxy_on` 后三元组为 `http://127.0.0.1:7890 | http://127.0.0.1:7890 | 127.0.0.1,localhost`；
- `curl` 返回 `HTTP/2 200`。

### B6. 日常使用与小贴士

- 开启代理：`proxy_on`（静默：`proxy_on true`）；关闭：`proxy_off`。
- 自动跟随端口：`autoproxy_maybe`。
- **注意**，在VSCode远程连接终端上执行proxy_on命令默认会将7890自动转发，导致远程服务器关机以后本机代理挂了，请在settings.json中添加以下内容：

```json
{
  "remote.autoForwardPorts": false,
  "remote.portsAttributes": {
    "7890": { "onAutoForward": "ignore", "label": "keep-local-proxy" }
  }
}
```
---

## 常见问题与排查

1. **扩展仍跑在远端**：`remote.extensionKind` 的键名需要填扩展 **Identifier**（扩展详情页可复制）。
2. **命令行能出网，但某些进程不走代理**：环境变量只影响其子进程；守护/容器需要在其启动环境单独传递代理变量。
3. **历史脚本污染了 ****`~/.bashrc`**：本方案已禁用注入；若历史已污染，备份后用 B4 的“一键命令”覆盖写入。
4. **证书/SSL 报错**：优先修 CA 或换源；不建议长期关闭 SSL 校验。
5. **端口冲突或未监听**：确认 `conf/config.yaml` 的 `port`，或在控制面板切换/保存后重启；必要时改为其他空闲端口。
