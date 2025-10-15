# VSCode 远程开发与服务器代理两套方案  
**—— 给需要在远程服务器上用 AI 插件与外网的同学的一份可复用指南**

> 面向对象：本地 Windows/macOS 能挂 VPN，但远程服务器（AutoDL、实验室/云服务器等）出不了网。  
> 目标：  
> - **A：只改 VSCode（扩展在本机 UI 运行）** → 无论远程能否出网，ChatGPT/Codex/Gemini/Claude Code 等扩展都能用。  
> - **B：让远程服务器自己出网并自启（clash-for-AutoDL）** → 服务器可 `pip/apt/git/curl` 正常访问外网。  

> 文中所有 `http://127.0.0.1:7890`、端口 `7890` 均为**示例**。  
> 如果你的本机代理/服务器代理端口不是 7890，请按实际端口修改（如 1080、8889、2080…）。  
> 协议也可能是 `socks5://`，请按你的代理软件实际配置调整。

---

## 目录
- [A. 只改 VSCode（本机有 VPN 即可）](#a-只改-vscode本机有-vpn-即可)
  - [A-1 打开“用户设置（JSON）”](#a-1-打开用户设置json)
  - [A-2 一次性粘贴以下配置（可整段）](#a-2-一次性粘贴以下配置可整段)
  - [A-3 验证扩展是否在 UI 本地运行](#a-3-验证扩展是否在-ui-本地运行)
- [B. 让远程服务器自己出网并自启（clash-for-AutoDL）](#b-让远程服务器自己出网并自启clash-for-autodl)
  - [B-1 备份原配置](#b-1-备份原配置)
  - [B-2 获取并启动 `clash-for-AutoDL`](#b-2-获取并启动-clash-for-autodl)
  - [B-3 向 `~/.bashrc` 追加“按需启用代理”的函数（整段粘贴）](#b-3-向-bashrc-追加按需启用代理的函数整段粘贴)
  - [B-4 无 systemd 场景：登录时自动启动 clash](#b-4-无-systemd-场景登录时自动启动-clash)
  - [B-5 可选：再加一条开机自启（cron）](#b-5-可选再加一条开机自启cron)
  - [B-6 验证外网与环境变量](#b-6-验证外网与环境变量)
- [只做 A / 只做 B / 两者都做？](#只做-a--只做-b--两者都做)
- [常见坑位与排查](#常见坑位与排查)
- [FAQ](#faq)
- [致谢与许可](#致谢与许可)

---

## A. 只改 VSCode（本机有 VPN 即可）
> 目的：让 **AI 插件在本机 UI 侧运行**，登录与联网全部走你的**本机代理**。  
> 结论：即使远程服务器不能出网，VSCode 内的 ChatGPT/Codex/Gemini/Claude Code 等扩展仍可正常使用。

### A-1 打开“用户设置（JSON）”
- 快捷键：`Ctrl + Shift + P`  
- 输入：`Open User Settings (JSON)` 或 `Preferences: Open User Settings (JSON)`  
- 回车打开 `settings.json`。

### A-2 一次性粘贴以下配置（可整段）
> **端口请改成你的本机代理端口**；如是 `socks5://` 也请改协议。  
> 如超出长度限制，请分两次粘贴到同一个 JSON 文件中。

```json
{
  // —— 让 VSCode 自身与终端走你的“本机代理”（示例端口 7890，改成你的端口/协议）——
  "http.proxy": "http://127.0.0.1:7890",
  "http.proxyStrictSSL": false,

  // —— 让“远程终端面板”（在 VSCode 窗口内的那个终端）继承本机代理 —— 
  "terminal.integrated.env.linux": {
    "http_proxy": "http://127.0.0.1:7890",
    "https_proxy": "http://127.0.0.1:7890",
    "HTTP_PROXY": "http://127.0.0.1:7890",
    "HTTPS_PROXY": "http://127.0.0.1:7890",
    "no_proxy": "localhost,127.0.0.1,::1"
  },

  // —— 关键：强制特定扩展在“本机 UI”运行（而不是远端服务器）——
  // 下面只演示 ChatGPT 扩展（openai.chatgpt）。其他扩展（Gemini、Codex、Claude Code 等）同理，按扩展 ID 添加为 ["ui"] 即可。
  "remote.extensionKind": {
    "openai.chatgpt": ["ui"]
    // "扩展ID": ["ui"]
  }
}
```

> 🔍 扩展 ID 范例：  
> - OpenAI ChatGPT 扩展：`openai.chatgpt`  
> - 其他厂商扩展可在其扩展详情页右侧找到 **Identifier**，按 `"扩展ID": ["ui"]` 添加。

### A-3 验证扩展是否在 UI 本地运行
- `Ctrl + Shift + P` → `Developer: Reload Window`  
- `Ctrl + Shift + P` → `Developer: Show Running Extensions`  
- 找到目标扩展（如 `openai.chatgpt`），确认 `Runs: UI` / `Local`。  
  若仍显示 `SSH: Remote`，请重开窗口或再次检查 `remote.extensionKind` 是否填写正确。

> ✅ 做完 A：VSCode 里的 AI 扩展“与远端出网解耦”。  
> ⚠️ 服务器上的 `pip/apt/git` 是否能出网，取决于 **B**。

---

## B. 让远程服务器自己出网并自启（clash-for-AutoDL）
> 目的：使服务器本身具备访问外网的能力（`pip/apt/git/curl`），并在登录/开机后**自动**启动代理与注入环境变量。  
> 项目来源：[`VocabVictor/clash-for-AutoDL`](https://github.com/VocabVictor/clash-for-AutoDL)

> **再次强调端口**：以下出现的 `7890` 都是示例。请与你的 `clash` 配置 `mixed-port` / `port` 保持一致。

### B-1 备份原配置
```bash
cp -f ~/.bashrc ~/.bashrc.bak.$(date +%s) 2>/dev/null || true
cp -f ~/.bash_profile ~/.bash_profile.bak.$(date +%s) 2>/dev/null || true
```

### B-2 获取并启动 `clash-for-AutoDL`
```bash
cd ~
git clone https://github.com/VocabVictor/clash-for-AutoDL.git || true
cd ~/clash-for-AutoDL
# 第一次运行会引导下载/检查配置
source ./start.sh
```

验证端口是否监听（示例监听 `7890`/`5334`）：
```bash
lsof -i -P -n | grep LISTEN | grep -E ':7890|:5334' || echo "no clash ports"
```

### B-3 向 `~/.bashrc` 追加“按需启用代理”的函数（整段粘贴）
> **一整段粘到 `~/.bashrc` 最末尾**；若端口不是 `7890`，请修改注释行后的默认值。  

```bash
cat >> ~/.bashrc <<'EOF'
# ---- Proxy helpers (CLEAN) ----
proxy_on() {
  local host="127.0.0.1"
  local cfg="$HOME/clash-for-AutoDL/conf/config.yaml"
  local port=""

  # 从 clash 配置自动解析端口：优先 mixed-port, 其次 port
  if [ -f "$cfg" ]; then
    port=$(awk -F': *' '/^mixed-port:/{print $2; f=1} END{if(!f)exit 0}' "$cfg" 2>/dev/null)
    [ -z "$port" ] && port=$(awk -F': *' '/^port:/{print $2; f=1} END{if(!f)exit 0}' "$cfg" 2>/dev/null)
  fi
  [ -z "$port" ] && port=7890   # ← 示例端口；改成你实际的代理端口

  export http_proxy="http://$host:$port"
  export https_proxy="$http_proxy"
  export all_proxy="$http_proxy"
  export HTTP_PROXY="$http_proxy"
  export HTTPS_PROXY="$http_proxy"
  export ALL_PROXY="$all_proxy"
  export no_proxy="127.0.0.1,localhost,::1,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16"
  export NO_PROXY="$no_proxy"
  echo "Proxy ON -> $http_proxy"
}

proxy_off() {
  unset http_proxy https_proxy all_proxy HTTP_PROXY HTTPS_PROXY ALL_PROXY no_proxy NO_PROXY proxy PROXY
  echo "Proxy OFF"
}

# 端口监听检查：优先用 ss，没有就用 lsof
is_port_listening_7890() {
  if command -v ss >/dev/null 2>&1; then
    ss -ltn 2>/dev/null | grep -q ':7890' && return 0
  fi
  lsof -iTCP:7890 -sTCP:LISTEN >/dev/null 2>&1 && return 0
  return 1
}

# 先清掉可能遗留的坏变量（如 http://:7890），再按需启用
proxy_off 2>/dev/null || true
if is_port_listening_7890; then
  proxy_on
fi
# ---- END ----
EOF
```

> 说明：这段逻辑保证了**clash 未监听时不注入代理变量**，避免产生 `http://:7890` 这样的“坏变量”。

### B-4 无 systemd 场景：登录时自动启动 clash
> 许多容器/镜像没有 `systemd`。用 `.bash_profile` 在**每次登录**时后台拉起 clash。

```bash
cat > ~/.bash_profile <<'EOF'
# 确保加载 .bashrc（含 proxy_on/off 与自动注入逻辑）
if [ -f ~/.bashrc ]; then source ~/.bashrc; fi

# 若 clash/mihomo 未在运行则启动（后台）
if ! pgrep -f 'mihomo|clash' >/dev/null 2>&1; then
  ( cd ~/clash-for-AutoDL && nohup bash -lc "source ./start.sh" >> ~/clash-for-AutoDL/boot.log 2>&1 & )
fi
EOF
```

> 提示：若你的环境支持 `systemd`，也可以写成 `systemd` 服务，但在常见容器里通常不可用。

### B-5 可选：再加一条开机自启（cron）
> 这步**可选**。个别环境下登录前就要自动启动 clash，可增加 `@reboot`。

```bash
crontab -l 2>/dev/null | { cat; echo '@reboot /bin/bash -lc "cd ~/clash-for-AutoDL && source ./start.sh"'; } | crontab -
```

### B-6 验证外网与环境变量
```bash
# 1) 端口监听
lsof -i -P -n | grep LISTEN | grep -E ':7890|:5334' || echo "no clash ports"

# 2) 变量是否已注入（有值才对）
echo "$http_proxy | $https_proxy | $all_proxy"

# 3) 外网连通性
curl -I -m 10 https://www.google.com
```

> 若遇到 `curl: (5) Unsupported proxy syntax in 'http://127.0.0.1:null'`，说明你的端口没有被正确解析：  
> - 检查 `clash-for-AutoDL/conf/config.yaml` 中的 `mixed-port` / `port`；  
> - 或手动把脚本中的默认 `port=7890` 改成你的实际端口。

---

## 只做 A / 只做 B / 两者都做？
- **只做 A**：希望 VSCode 里 AI 扩展可用，**不关心**服务器自身是否能出网。  
- **只做 B**：需要服务器本身能出网（下载依赖、训练时拉数据、`apt/pip/git` 等）。  
- **两者都做**：插件稳定 + 服务器也能独立出网 → 最顺手的日常开发体验。距离Codex能直接远程调试代码只差最后一步，把本地~/.codex文件夹粘贴到远程服务器再重启vscode或重新连接ssh就可以实现真正的远程vibe coding啦！

---

## 常见坑位与排查
1) **扩展仍跑在远端**  
- 重载窗口；检查 `remote.extensionKind` 是否把扩展 ID 设为 `["ui"]`。  
- `Developer: Show Running Extensions` 确认 `Runs: UI/Local`。

2) **`http://:7890` 坏变量**  
- 说明在 clash 未监听时就写入了代理变量。  
- 使用本指南 B-3 的“**按需启用**”逻辑可彻底避免。

3) **端口与协议不符**  
- 你的代理可能是 `socks5://127.0.0.1:1080`；请全局替换并与实际软件一致。  
- VSCode 的 `http.proxy` 支持 `http://` 与 `socks5://`。

4) **容器没有 `systemd`**  
- 正常，请用 `.bash_profile`（B-4）+（可选）`cron @reboot`（B-5）方式自启。

5) **VSCode 远程终端连外网慢/不稳**  
- 可以同时设置 VSCode 的 `terminal.integrated.env.linux` 与服务器 `.bashrc`，不冲突。  
- 也可仅依赖服务器 `.bashrc` 注入，二选一或都设均可。

---

## FAQ
**Q1：扩展 ID 哪里看？**  
A：在扩展详情页右侧找 **Identifier**，形如 `publisher.name`，例如 `openai.chatgpt`。

**Q2：为什么一定要把扩展设为 UI 运行？**  
A：这样扩展的登录与网络都走你**本机**；即使远端网络受限，扩展照样可用。

**Q3：服务器代理端口不是 7890 怎么办？**  
A：将文内所有 `7890` 替换为你的端口（含 `~/.bashrc` 中解析失败时的**默认值**）。

**Q4：我只想让 `pip` 走代理，不想全局走？**  
A：可以不调用 `proxy_on`，在用到时临时指定：  
```bash
https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 pip install xxx
```

---

## 致谢与许可
- 服务器代理部分基于开源项目 **clash-for-AutoDL** 的实践与包装。  
- 文档建议使用 **MIT License** 方便复用与二次修改。欢迎提交 Issue/PR 补充你所在学校/云厂商环境的差异与最佳实践。

> Happy hacking & good luck with your experiments! 🚀
