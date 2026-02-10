---
title: Mac Mini M4 部署指南
description: 在 Mac Mini M4 (Apple Silicon) 上從零開始部署 Clawdbot 的完整說明
---

# Mac Mini M4 部署指南

本文件說明如何在一台全新的 Mac Mini M4（Apple Silicon, macOS）上從零開始部署 Clawdbot。

> **重要提醒**：本專案的 macOS 支援尚未完全成熟，有部分任務缺少 macOS 分支（詳見「[已知問題與限制](#已知問題與限制)」章節）。本指南同時提供自動化安裝的補充步驟與完整手動安裝流程，確保你能順利完成部署。

---

## 目錄

- [環境需求](#環境需求)
- [已知問題與限制](#已知問題與限制)
- [方式一：自動化安裝（建議）](#方式一自動化安裝建議)
- [方式二：完整手動安裝](#方式二完整手動安裝)
- [安裝後配置](#安裝後配置)
- [Tailscale VPN 設定](#tailscale-vpn-設定)
- [macOS 專屬安全建議](#macos-專屬安全建議)
- [服務管理（無 systemd）](#服務管理無-systemd)
- [驗證清單](#驗證清單)
- [疑難排解](#疑難排解)

---

## 環境需求

### 硬體

- Mac Mini M4（Apple Silicon, ARM64）
- 建議至少 16 GB RAM
- 建議至少 50 GB 可用磁碟空間

### 軟體

- macOS 15 (Sequoia) 或更新版本（Mac Mini M4 出廠即搭載）
- 管理員帳號（具 sudo 權限）
- 穩定的網際網路連線

### 開始之前

1. 完成 macOS 初始設定精靈
2. 開啟「終端機」（Terminal.app）或安裝 iTerm2
3. 確認你的使用者具有管理員權限：
   ```bash
   # 應顯示包含 "admin" 的群組
   groups $(whoami)
   ```

---

## 已知問題與限制

在 Mac Mini M4 上執行本 Ansible Playbook 時，有以下 **已知問題** 需要手動處理：

### 問題 1：Node.js 安裝缺少 macOS 支援

`roles/clawdbot/tasks/nodejs.yml` 使用 `apt`（Debian/Ubuntu 套件管理工具），在 macOS 上 **會失敗**。

**解法**：在執行 Playbook 前先手動安裝 Node.js，或在執行 Playbook 時跳過 nodejs 任務並手動補裝。

### 問題 2：使用者家目錄路徑

`defaults/main.yml` 中 `clawdbot_home` 預設為 `/home/clawdbot`。macOS 的使用者目錄通常位於 `/Users/`，且 `/home` 由 autofs 管理，直接建立目錄可能失敗。

**解法**：執行 Playbook 時覆寫變數為 `/Users/clawdbot`。

### 問題 3：sudoers 設定參照 Linux 路徑

`user.yml` 中的 sudoers 設定參照 `/usr/bin/systemctl` 和 `/usr/bin/journalctl`，這些在 macOS 上不存在。設定會建立成功但規則無效。

**影響**：無直接危害，但也無實際作用。macOS 不使用 systemd。

### 問題 4：無 systemd

macOS 使用 launchd 而非 systemd。`clawdbot onboard --install-daemon` 在 macOS 上可能無法自動安裝系統服務。

**解法**：手動建立 launchd plist 或使用前台模式執行。

### 問題 5：安全功能縮減

macOS 上 **不可用** 的安全功能：

| 功能 | Linux | macOS | 替代方案 |
|------|-------|-------|---------|
| UFW + DOCKER-USER | ✅ | ❌ | macOS Application Firewall（基礎） |
| Fail2ban | ✅ | ❌ | 無內建替代（可考慮第三方工具） |
| unattended-upgrades | ✅ | ❌ | 系統偏好設定 → 軟體更新 → 自動更新 |
| systemd 安全強化 | ✅ | ❌ | 無直接替代 |

---

## 方式一：自動化安裝（建議）

### 步驟 1：安裝 Xcode Command Line Tools

```bash
xcode-select --install
```

在彈出視窗中點選「安裝」，等待完成（約 5-10 分鐘）。

### 步驟 2：安裝 Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

安裝完成後，依照提示將 Homebrew 加入 PATH：

```bash
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

驗證：

```bash
brew --version
```

### 步驟 3：安裝 Ansible 與 Python

```bash
brew install ansible python3
```

驗證：

```bash
ansible --version
python3 --version
```

### 步驟 4：預先安裝 Node.js 與 pnpm（解決問題 1）

由於 Playbook 的 `nodejs.yml` 不支援 macOS，需要先手動安裝：

```bash
# 安裝 Node.js 22.x
brew install node@22

# 將 Node.js 22 加入 PATH（如果不是預設版本）
echo 'export PATH="/opt/homebrew/opt/node@22/bin:$PATH"' >> ~/.zprofile
source ~/.zprofile

# 安裝 pnpm
npm install -g pnpm

# 驗證
node --version    # 應為 v22.x.x
pnpm --version
```

### 步驟 5：複製儲存庫並安裝集合

```bash
git clone https://github.com/pasogott/clawdbot-ansible.git
cd clawdbot-ansible

# 安裝 Ansible Galaxy 集合
ansible-galaxy collection install -r requirements.yml
```

### 步驟 6：執行 Playbook

使用以下指令執行，覆寫 macOS 不相容的預設值：

```bash
ansible-playbook playbook.yml --ask-become-pass \
  -e clawdbot_home=/Users/clawdbot
```

**開發模式**（從原始碼建構）：

```bash
ansible-playbook playbook.yml --ask-become-pass \
  -e clawdbot_home=/Users/clawdbot \
  -e clawdbot_install_mode=development
```

**帶 SSH 金鑰**：

```bash
ansible-playbook playbook.yml --ask-become-pass \
  -e clawdbot_home=/Users/clawdbot \
  -e "clawdbot_ssh_keys=['$(cat ~/.ssh/id_ed25519.pub)']"
```

> **注意**：Playbook 執行到 `nodejs.yml` 時會失敗（因為使用 apt）。此時 Node.js 已在步驟 4 手動安裝，你可以使用 `--skip-tags` 或在失敗後繼續手動完成剩餘步驟（參見下方「安裝後配置」）。

### 步驟 7：處理 Node.js 任務失敗

如果 Playbook 在 `nodejs.yml` 階段失敗，Node.js 已經透過 Homebrew 安裝好了。接下來手動完成 Clawdbot 安裝：

```bash
# 切換到 clawdbot 使用者
sudo su - clawdbot

# 確認 Node.js 和 pnpm 可用
node --version
pnpm --version

# 如果 pnpm 不可用，手動安裝
npm install -g pnpm

# 建立 pnpm 目錄
mkdir -p ~/.local/share/pnpm ~/.local/share/pnpm/store ~/.local/bin

# 配置 pnpm
pnpm config set global-dir ~/.local/share/pnpm
pnpm config set global-bin-dir ~/.local/bin

# 安裝 Clawdbot（Release 模式）
pnpm install -g clawdbot@latest

# 驗證
~/.local/bin/clawdbot --version
```

---

## 方式二：完整手動安裝

如果你偏好完全手動控制安裝過程，按以下步驟逐一執行。

### 1. 基礎環境

```bash
# Xcode Command Line Tools
xcode-select --install

# Homebrew
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> ~/.zprofile
eval "$(/opt/homebrew/bin/brew shellenv)"
```

### 2. 系統工具

```bash
brew install \
  zsh vim nano \
  git git-lfs \
  curl wget netcat nmap socat telnet \
  htop \
  tmux tree jq unzip rsync
```

### 3. Tailscale VPN

```bash
brew install --cask tailscale

# 安裝後從「應用程式」開啟 Tailscale，或使用 CLI：
open /Applications/Tailscale.app
```

### 4. 建立 clawdbot 使用者

```bash
# 建立使用者（macOS 方式）
sudo sysadminctl -addUser clawdbot \
  -fullName "Clawdbot Service User" \
  -home /Users/clawdbot \
  -shell /bin/zsh \
  -password -  # 會提示輸入密碼

# 如果需要無密碼使用者（作為服務帳號），也可以用 dscl：
# sudo dscl . -create /Users/clawdbot
# sudo dscl . -create /Users/clawdbot UserShell /bin/zsh
# sudo dscl . -create /Users/clawdbot NFSHomeDirectory /Users/clawdbot
# sudo dscl . -create /Users/clawdbot UniqueID 550
# sudo dscl . -create /Users/clawdbot PrimaryGroupID 20
# sudo createhomedir -c -u clawdbot
```

### 5. Docker Desktop

```bash
brew install --cask docker

# 啟動 Docker Desktop
open /Applications/Docker.app

# 等待 Docker 完全啟動（選單列出現 Docker 圖示）
# 首次啟動需要同意授權條款

# 驗證
docker --version
docker compose version
```

> **Apple Silicon 注意事項**：Docker Desktop 原生支援 ARM64。大部分映像檔都有 ARM64 版本，少數舊映像檔會透過 Rosetta 2 模擬執行。

### 6. macOS 防火牆

```bash
# 檢查狀態
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate

# 啟用防火牆
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# 允許 Tailscale 通過防火牆
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --add /Applications/Tailscale.app
```

也可以透過圖形介面設定：
**系統設定** → **網路** → **防火牆** → 開啟

### 7. Node.js 與 pnpm

```bash
# 安裝 Node.js 22.x
brew install node@22

# 確保 Node.js 22 在 PATH 中
echo 'export PATH="/opt/homebrew/opt/node@22/bin:$PATH"' >> /Users/clawdbot/.zshrc

# 安裝 pnpm
npm install -g pnpm

# 驗證
node --version    # v22.x.x
pnpm --version
```

### 8. Oh My Zsh 與 Shell 設定

```bash
# 切換到 clawdbot 使用者
sudo su - clawdbot

# 安裝 Oh My Zsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" "" --unattended

# 設定 .zshrc
cat >> ~/.zshrc << 'EOF'
# Homebrew
eval "$(/opt/homebrew/bin/brew shellenv)"

# pnpm
export PNPM_HOME="$HOME/.local/share/pnpm"
export PATH="$HOME/.local/bin:$PNPM_HOME:$PATH"

# 色彩支援
export TERM=xterm-256color
export COLORTERM=truecolor
export CLICOLOR=1
export LSCOLORS=ExFxCxDxBxegedabagacad

# 別名
alias ls='ls -G'
alias grep='grep --color=auto'
alias ll='ls -lah'
EOF

source ~/.zshrc
```

### 9. 安裝 Clawdbot

#### Release 模式

```bash
# 以 clawdbot 使用者執行
sudo su - clawdbot

# 建立目錄
mkdir -p ~/.clawdbot/{sessions,credentials,data,logs}
chmod 700 ~/.clawdbot/credentials
mkdir -p ~/.local/share/pnpm/store ~/.local/bin

# 配置 pnpm
pnpm config set global-dir ~/.local/share/pnpm
pnpm config set global-bin-dir ~/.local/bin

# 安裝
pnpm install -g clawdbot@latest

# 驗證
~/.local/bin/clawdbot --version
```

#### Development 模式

```bash
# 以 clawdbot 使用者執行
sudo su - clawdbot

# 建立目錄
mkdir -p ~/.clawdbot/{sessions,credentials,data,logs}
chmod 700 ~/.clawdbot/credentials
mkdir -p ~/.local/share/pnpm/store ~/.local/bin ~/code

# 配置 pnpm
pnpm config set global-dir ~/.local/share/pnpm
pnpm config set global-bin-dir ~/.local/bin

# 複製原始碼
cd ~/code
git clone https://github.com/clawdbot/clawdbot.git
cd clawdbot

# 安裝依賴並建構
pnpm install
pnpm build

# 建立 symlink
ln -sf ~/code/clawdbot/bin/clawdbot.js ~/.local/bin/clawdbot
chmod +x ~/code/clawdbot/bin/clawdbot.js

# 驗證
~/.local/bin/clawdbot --version

# 新增開發別名到 .zshrc
cat >> ~/.zshrc << 'EOF'

# Clawdbot 開發模式
export CLAWDBOT_DEV_DIR="$HOME/code/clawdbot"
alias clawdbot-rebuild='cd ~/code/clawdbot && pnpm build'
alias clawdbot-dev='cd ~/code/clawdbot'
alias clawdbot-pull='cd ~/code/clawdbot && git pull && pnpm install && pnpm build'
EOF
```

---

## 安裝後配置

### 步驟 1：切換到 clawdbot 使用者

```bash
sudo su - clawdbot
```

### 步驟 2：確認環境

```bash
echo "使用者: $(whoami)"
echo "家目錄: $HOME"
echo "Node.js: $(node --version)"
echo "pnpm: $(pnpm --version)"
echo "Clawdbot: $(~/.local/bin/clawdbot --version 2>/dev/null || echo '未安裝')"
echo "Docker: $(docker --version 2>/dev/null || echo '未安裝')"
echo "Homebrew: $(brew --version 2>/dev/null | head -1 || echo '未安裝')"
```

### 步驟 3：執行 Clawdbot 初始設定

```bash
clawdbot onboard --install-daemon
```

此指令會引導你：
- 建立設定檔 `~/.clawdbot/clawdbot.json`
- 選擇訊息提供者（WhatsApp / Telegram / Signal）
- 設定 AI 模型 API 金鑰

> **macOS 注意**：`--install-daemon` 在 macOS 上可能無法自動安裝 launchd 服務。如果此步驟失敗，請參考下方「[服務管理](#服務管理無-systemd)」章節手動設定。

### 步驟 4：手動設定（替代方案）

如果 `clawdbot onboard` 不可用或你偏好手動設定：

```bash
# 執行互動式設定
clawdbot configure

# 或直接編輯設定檔
nano ~/.clawdbot/clawdbot.json

# 登入訊息提供者
clawdbot providers login

# 測試閘道器
clawdbot gateway

# 診斷檢查
clawdbot doctor
```

---

## Tailscale VPN 設定

### GUI 方式（推薦）

1. 從「應用程式」開啟 **Tailscale**
2. 點選選單列中的 Tailscale 圖示
3. 點選「Log in」
4. 在瀏覽器中完成登入流程

### CLI 方式

```bash
# 啟動並登入
sudo /Applications/Tailscale.app/Contents/MacOS/Tailscale up

# 使用 auth key 自動登入（適用於自動化）
sudo /Applications/Tailscale.app/Contents/MacOS/Tailscale up --authkey tskey-auth-xxxxx

# 檢查狀態
/Applications/Tailscale.app/Contents/MacOS/Tailscale status

# 查看分配的 IP
/Applications/Tailscale.app/Contents/MacOS/Tailscale ip -4
```

取得 auth key：https://login.tailscale.com/admin/settings/keys

### 建立 Tailscale CLI 別名

```bash
# 加入 .zshrc 方便使用
echo 'alias tailscale="/Applications/Tailscale.app/Contents/MacOS/Tailscale"' >> ~/.zshrc
source ~/.zshrc

# 之後可以直接使用
tailscale status
```

---

## macOS 專屬安全建議

由於 macOS 缺少 Linux 上的部分安全功能，建議進行以下額外設定：

### 1. 啟用 FileVault 磁碟加密

**系統設定** → **隱私與安全** → **FileVault** → 開啟

### 2. 啟用自動更新

**系統設定** → **一般** → **軟體更新** → 啟用自動更新

### 3. 啟用 macOS 防火牆

```bash
# CLI 方式
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on

# 啟用隱身模式（不回應 ping）
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on
```

### 4. 停用不必要的遠端服務

```bash
# 檢查目前開啟的遠端服務
sudo systemsetup -getremotelogin          # SSH
sudo systemsetup -getremoteappleevents    # Apple Remote Events

# 如果不需要 SSH 遠端登入，關閉它
# sudo systemsetup -setremotelogin off
```

### 5. 設定 SSH（如果需要遠端存取）

```bash
# 啟用 SSH
sudo systemsetup -setremotelogin on

# 僅允許特定使用者 SSH 登入
sudo dseditgroup -o create -q com.apple.access_ssh
sudo dseditgroup -o edit -a clawdbot -t user com.apple.access_ssh
sudo dseditgroup -o edit -a YOUR_ADMIN_USER -t user com.apple.access_ssh

# 建議：使用金鑰認證，停用密碼登入
# 編輯 /etc/ssh/sshd_config：
#   PasswordAuthentication no
#   PubkeyAuthentication yes
```

### 6. Docker Desktop 安全設定

開啟 Docker Desktop → **Settings** → **General**：
- 確認「Use Virtualization Framework」已勾選（Apple Silicon 原生支援）

Docker Desktop → **Settings** → **Resources**：
- 依需求限制 CPU 和記憶體使用量（建議：4 CPU / 8 GB）

---

## 服務管理（無 systemd）

macOS 不支援 systemd。以下是在 macOS 上管理 Clawdbot 服務的替代方案。

### 方案 1：手動前台執行

適用於開發與測試：

```bash
# 以 clawdbot 使用者執行
sudo su - clawdbot
clawdbot gateway
```

### 方案 2：使用 tmux 背景執行

```bash
# 建立 tmux session
sudo su - clawdbot
tmux new-session -d -s clawdbot 'clawdbot gateway'

# 查看 session
tmux list-sessions

# 附著到 session
tmux attach-session -t clawdbot

# 從 session 分離（保持執行）
# 按 Ctrl+B 然後按 D
```

### 方案 3：建立 launchd 服務（推薦用於正式環境）

建立 launchd plist 檔案，讓 Clawdbot 開機自動啟動：

```bash
# 以管理員身份建立 plist
sudo tee /Library/LaunchDaemons/com.clawdbot.gateway.plist << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.clawdbot.gateway</string>

    <key>UserName</key>
    <string>clawdbot</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/clawdbot/.local/bin/clawdbot</string>
        <string>gateway</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/clawdbot</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>HOME</key>
        <string>/Users/clawdbot</string>
        <key>PATH</key>
        <string>/Users/clawdbot/.local/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin</string>
        <key>PNPM_HOME</key>
        <string>/Users/clawdbot/.local/share/pnpm</string>
    </dict>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/clawdbot/.clawdbot/logs/gateway.stdout.log</string>

    <key>StandardErrorPath</key>
    <string>/Users/clawdbot/.clawdbot/logs/gateway.stderr.log</string>

    <key>ThrottleInterval</key>
    <integer>10</integer>
</dict>
</plist>
EOF

# 設定權限
sudo chown root:wheel /Library/LaunchDaemons/com.clawdbot.gateway.plist
sudo chmod 644 /Library/LaunchDaemons/com.clawdbot.gateway.plist
```

管理 launchd 服務：

```bash
# 載入服務（啟動）
sudo launchctl load /Library/LaunchDaemons/com.clawdbot.gateway.plist

# 卸載服務（停止）
sudo launchctl unload /Library/LaunchDaemons/com.clawdbot.gateway.plist

# 檢查狀態
sudo launchctl list | grep clawdbot

# 查看日誌
tail -f /Users/clawdbot/.clawdbot/logs/gateway.stdout.log
tail -f /Users/clawdbot/.clawdbot/logs/gateway.stderr.log
```

---

## 驗證清單

安裝完成後，逐一確認以下項目：

```bash
# 1. Homebrew
brew --version
# ✅ Homebrew 4.x.x

# 2. Node.js
node --version
# ✅ v22.x.x

# 3. pnpm
pnpm --version
# ✅ 9.x.x 或更新

# 4. Docker
docker --version
docker compose version
# ✅ Docker Desktop with Compose V2

# 5. Tailscale
/Applications/Tailscale.app/Contents/MacOS/Tailscale version
# ✅ Tailscale 已安裝

# 6. Clawdbot（切換到 clawdbot 使用者後）
sudo su - clawdbot
~/.local/bin/clawdbot --version
# ✅ Clawdbot 已安裝

# 7. 目錄結構
ls -la ~/.clawdbot/
# ✅ sessions/ credentials/ data/ logs/ 存在

# 8. macOS 防火牆
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --getglobalstate
# ✅ Firewall is enabled

# 9. Tailscale 連線
/Applications/Tailscale.app/Contents/MacOS/Tailscale status
# ✅ 已連線到 Tailnet（如有設定）

# 10. Docker 運作正常
docker run --rm hello-world
# ✅ Hello from Docker!
```

---

## 疑難排解

### Homebrew 安裝失敗

```bash
# 確認 Xcode Command Line Tools 已安裝
xcode-select -p
# 應顯示 /Library/Developer/CommandLineTools

# 如果未安裝
xcode-select --install
```

### Docker Desktop 無法啟動

- 確認已同意 Docker Desktop 的授權條款
- 確認在「系統設定」→「隱私與安全」中允許 Docker
- Apple Silicon 請確認使用的是 ARM64 版本

```bash
# 檢查 Docker Desktop 進程
ps aux | grep Docker

# 重新安裝
brew reinstall --cask docker
```

### /home/clawdbot 建立失敗

macOS 的 `/home` 由 autofs 管理。如果 Playbook 嘗試在此建立目錄：

```bash
# 查看 autofs 設定
cat /etc/auto_master

# 解法 1（推薦）：使用 /Users/clawdbot
# 在 Playbook 中覆寫：-e clawdbot_home=/Users/clawdbot

# 解法 2（不推薦）：停用 /home 的 autofs
# 需要編輯 /etc/auto_master 並移除 /home 那一行
# 然後重啟 autofs: sudo automount -vc
```

### pnpm install -g 權限錯誤

```bash
# 確認目錄所有權
ls -la ~/.local/share/pnpm/
ls -la ~/.local/bin/

# 修復所有權
sudo chown -R clawdbot:staff ~/.local/
```

### Node.js 版本不正確

```bash
# 查看 Homebrew 安裝的 Node.js
brew list --versions | grep node

# 如果安裝了多個版本
brew unlink node
brew link node@22 --force

# 確認 PATH 中 Node.js 22 優先
which node
node --version
```

### Clawdbot onboard 在 macOS 上失敗

如果 `clawdbot onboard --install-daemon` 的 daemon 安裝步驟失敗：

```bash
# 先完成 onboard 的設定部分（不含 daemon）
clawdbot onboard

# 然後手動建立 launchd 服務
# 參考上方「服務管理」→「方案 3」
```

### Playbook 在 nodejs.yml 階段失敗

這是預期行為（`nodejs.yml` 不支援 macOS）。確認 Node.js 已透過 Homebrew 安裝後，可以手動完成 Clawdbot 安裝：

```bash
# 確認 Node.js 已安裝
node --version
pnpm --version

# 手動完成 Clawdbot 安裝步驟
# 參考「方式二：完整手動安裝」→ 步驟 9
```

---

## 快速參考卡

```
╔══════════════════════════════════════════════════════════╗
║              Mac Mini M4 部署快速參考                     ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  安裝：                                                  ║
║    brew install ansible node@22 python3                  ║
║    npm install -g pnpm                                   ║
║    brew install --cask docker tailscale                  ║
║                                                          ║
║  Playbook（覆寫 macOS 路徑）：                           ║
║    ansible-playbook playbook.yml --ask-become-pass \     ║
║      -e clawdbot_home=/Users/clawdbot                   ║
║                                                          ║
║  設定 Clawdbot：                                         ║
║    sudo su - clawdbot                                    ║
║    clawdbot onboard                                      ║
║                                                          ║
║  服務管理：                                              ║
║    sudo launchctl load   /Library/LaunchDaemons/...plist ║
║    sudo launchctl unload /Library/LaunchDaemons/...plist ║
║    sudo launchctl list | grep clawdbot                   ║
║                                                          ║
║  日誌：                                                  ║
║    tail -f ~/.clawdbot/logs/gateway.stdout.log           ║
║                                                          ║
║  Tailscale：                                             ║
║    alias tailscale="/Applications/Tailscale.app/...      ║
║      .../Contents/MacOS/Tailscale"                       ║
║    tailscale status                                      ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```
