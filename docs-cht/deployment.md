# 部署與安裝流程

## 安裝流程總覽

```
┌──────────────────────────────────────────────────────────┐
│                    安裝入口                               │
│                                                          │
│  方式 1：curl | bash（一鍵安裝）                          │
│  方式 2：git clone + run-playbook.sh（手動安裝）          │
│  方式 3：ansible-playbook（直接執行）                     │
└──────────────┬───────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│                  install.sh 安裝腳本                      │
│                                                          │
│  1. 偵測作業系統（Linux/macOS）                           │
│  2. 檢查 root/sudo 權限                                  │
│  3. 安裝 Ansible（若未安裝）                              │
│  4. 複製 clawdbot-ansible 儲存庫                          │
│  5. 安裝 Ansible Galaxy 集合                              │
│  6. 呼叫 run-playbook.sh                                 │
└──────────────┬───────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│               run-playbook.sh 包裝腳本                    │
│                                                          │
│  - root 執行：ansible-playbook playbook.yml              │
│               -e ansible_become=false                    │
│  - 非 root：ansible-playbook playbook.yml                │
│              --ask-become-pass                            │
│  - 成功後顯示下一步驟指引                                │
└──────────────┬───────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│                  playbook.yml 執行                        │
│                                                          │
│  pre_tasks:                                              │
│   ├── OS 偵測                                            │
│   ├── apt dist-upgrade（Linux）                          │
│   ├── 安裝 ACL                                           │
│   ├── 安裝 Ansible 集合                                  │
│   └── 安裝 Homebrew                                      │
│                                                          │
│  role: clawdbot                                          │
│   ├── system-tools（系統工具）                            │
│   ├── tailscale（VPN）                                   │
│   ├── user（使用者建立）                                  │
│   ├── docker（Docker 安裝）                              │
│   ├── firewall（防火牆配置）                              │
│   ├── nodejs（Node.js + pnpm）                           │
│   └── clawdbot（應用安裝）                               │
│                                                          │
│  post_tasks:                                             │
│   ├── 顯示 ASCII 藝術                                    │
│   └── 建立歡迎訊息                                       │
└──────────────┬───────────────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│                  安裝後手動步驟                            │
│                                                          │
│  1. sudo su - clawdbot                                   │
│  2. clawdbot onboard --install-daemon                    │
│  3.（選擇性）sudo tailscale up                           │
└──────────────────────────────────────────────────────────┘
```

---

## 安裝方式詳解

### 方式 1：一鍵安裝（curl | bash）

```bash
curl -fsSL https://raw.githubusercontent.com/pasogott/clawdbot-ansible/main/install.sh | bash
```

**`install.sh` 執行流程**：

1. **環境偵測**
   - 設定 256 色終端支援
   - 偵測 OS（`darwin*` → macOS，有 `apt-get` → Linux）
   - 不支援的 OS 直接退出

2. **權限處理**
   - root 使用者：不需要 sudo，傳遞 `-e ansible_become=false`
   - 非 root 使用者：使用 `--ask-become-pass` 提示輸入密碼

3. **前置安裝**
   - 檢查 Ansible 是否已安裝
   - 若未安裝，透過 `apt-get` 安裝

4. **儲存庫取得**
   - 在暫存目錄中 `git clone` 整個 clawdbot-ansible 儲存庫
   - 安裝 `requirements.yml` 中定義的 Ansible 集合

5. **執行 Playbook**
   - 呼叫 `run-playbook.sh`
   - 完成後清理暫存目錄

### 方式 2：手動安裝

```bash
# 安裝前置需求
sudo apt update && sudo apt install -y ansible git

# 複製儲存庫
git clone https://github.com/pasogott/clawdbot-ansible.git
cd clawdbot-ansible

# 安裝 Ansible 集合
ansible-galaxy collection install -r requirements.yml

# 執行安裝
./run-playbook.sh
```

### 方式 3：指定安裝模式

```bash
# Release 模式（預設）
ansible-playbook playbook.yml --ask-become-pass

# Development 模式
ansible-playbook playbook.yml --ask-become-pass -e clawdbot_install_mode=development

# 帶自訂變數
ansible-playbook playbook.yml --ask-become-pass -e @vars.yml
```

---

## 各任務詳細安裝流程

### 1. 系統工具安裝（system-tools）

#### Linux（apt）

安裝的套件分類：

| 類別 | 套件 |
|------|------|
| Shell | zsh |
| 編輯器 | vim, nano |
| 版本控制 | git, git-lfs |
| 網路工具 | curl, wget, nmap, socat, telnet, netcat, dnsutils, traceroute, tcpdump 等 |
| 除錯工具 | strace, lsof, gdb, htop, iotop, iftop, sysstat, procps |
| 系統工具 | tmux, tree, jq, unzip, rsync, less |
| 建構工具 | build-essential, file |

額外設定：
- 將 clawdbot 使用者預設 shell 改為 zsh
- 部署全域 Vim 設定
- 設定 `.bashrc` 和 `.zshrc`（Homebrew PATH、pnpm PATH、色彩支援、別名）

#### macOS（brew）

透過 Homebrew 安裝同等工具集（部分 Linux 專用工具不適用於 macOS）。

#### 共用設定

- 安裝 oh-my-zsh（以 clawdbot 使用者身份）
- 配置 Git 全域設定（預設分支、編輯器、別名等）

### 2. Tailscale VPN 安裝

#### Linux

```
1. 新增 Tailscale GPG 金鑰
2. 新增 Tailscale apt 套件庫
3. apt update
4. apt install tailscale
5. systemctl enable + start tailscaled
6. 檢查連線狀態，若未連線則顯示指引
```

#### macOS

```
1. 檢查 /Applications/Tailscale.app 是否存在
2. 若不存在：brew install --cask tailscale
3. 檢查執行狀態
4. 若未連線則顯示指引
```

### 3. 使用者建立（user）

```
1. 建立 clawdbot 系統使用者
   - system: true
   - shell: /bin/bash
   - home: /home/clawdbot

2. 配置範圍限定 sudo 權限
   - /etc/sudoers.d/clawdbot
   - 使用 visudo 驗證語法

3. 取得 clawdbot UID
4. 啟用 loginctl lingering
5. 建立 /run/user/<UID> 目錄
6. 儲存 UID 為 fact（供 systemd 範本使用）

7. 建立 .ssh 目錄（0700 權限）
8. 新增 SSH 授權金鑰（若有提供）

9. 設定 .bashrc 環境變數
   - XDG_RUNTIME_DIR
   - DBUS_SESSION_BUS_ADDRESS
```

### 4. Docker 安裝

#### Linux（Docker CE）

```
1. 安裝前置套件（ca-certificates, curl, gnupg, lsb-release）
2. 建立 /etc/apt/keyrings 目錄
3. 新增 Docker GPG 金鑰
4. 新增 Docker apt 套件庫
5. apt update
6. 安裝 Docker CE 套件：
   - docker-ce
   - docker-ce-cli
   - containerd.io
   - docker-buildx-plugin
   - docker-compose-plugin
7. systemctl enable + start docker
8. 將 clawdbot 使用者加入 docker 群組
9. 重置 SSH 連線（套用群組變更）
```

#### macOS（Docker Desktop）

```
1. 檢查 /Applications/Docker.app 是否存在
2. 若不存在：brew install --cask docker
3. 等待 /var/run/docker.sock（最多 120 秒）
4. 驗證 docker --version
```

### 5. 防火牆配置

#### Linux

**Fail2ban 安裝**：
```
1. apt install fail2ban
2. 寫入 /etc/fail2ban/jail.local
3. systemctl enable + start fail2ban
```

**unattended-upgrades 安裝**：
```
1. apt install unattended-upgrades apt-listchanges
2. 寫入 /etc/apt/apt.conf.d/20auto-upgrades
3. 寫入 /etc/apt/apt.conf.d/50unattended-upgrades
```

**UFW 防火牆**：
```
1. apt install ufw
2. 設定預設政策（deny incoming, allow outgoing, deny routed）
3. 允許 SSH (22/tcp)
4. 允許 Tailscale (41641/udp)
5. 偵測預設網路介面
6. 寫入 DOCKER-USER 鏈到 /etc/ufw/after.rules
7. 建立 /etc/docker 目錄
8. 部署 daemon.json（透過 template → 觸發 Restart docker handler）
9. 啟用 + 重載 UFW
```

#### macOS

```
1. 檢查 Application Firewall 狀態
2. 若停用則啟用
3. 允許 Tailscale 通過防火牆
```

### 6. Node.js 安裝

```
1. 安裝前置套件
2. 新增 NodeSource GPG 金鑰
3. 新增 NodeSource apt 套件庫（node_22.x）
4. apt update
5. apt install nodejs
6. npm install -g pnpm
7. 驗證版本（node --version, pnpm --version）
```

### 7. Clawdbot 應用安裝

**共用步驟**：
```
1. 建立目錄結構：
   - ~/.clawdbot/
   - ~/.clawdbot/sessions/
   - ~/.clawdbot/credentials/  (0700)
   - ~/.clawdbot/data/
   - ~/.clawdbot/logs/

2. 建立 pnpm 目錄：
   - ~/.local/share/pnpm/
   - ~/.local/share/pnpm/store/
   - ~/.local/bin/

3. 配置 pnpm global-dir 和 global-bin-dir
4. 設定 .bashrc（PNPM_HOME, PATH）
```

**Release 模式**：
```
5a. pnpm install -g clawdbot@latest（以 clawdbot 使用者身份）
6a. 驗證安裝（clawdbot --version）
```

**Development 模式**：
```
5b. 建立 ~/code 目錄
6b. git clone clawdbot 儲存庫
7b. pnpm install（安裝依賴）
8b. pnpm build（建構原始碼）
9b. 驗證 dist/ 目錄存在
10b. 建立 symlink：~/.local/bin/clawdbot → ~/code/clawdbot/bin/clawdbot.js
11b. 設定可執行權限
12b. 驗證安裝
13b. 新增開發別名到 .bashrc
```

---

## 安裝後配置

### 步驟 1：切換使用者

```bash
sudo su - clawdbot
```

首次登入時會顯示歡迎訊息與操作指南。

### 步驟 2：執行初始設定

```bash
clawdbot onboard --install-daemon
```

此指令會：
- 引導你完成設定精靈
- 建立 `~/.clawdbot/clawdbot.json`
- 設定訊息提供者（WhatsApp/Telegram/Signal）
- 安裝並啟動 systemd daemon

### 步驟 3：連線 Tailscale（選擇性）

```bash
# 返回有 sudo 權限的使用者
exit

# 連線 Tailscale
sudo tailscale up
```

---

## 變數配置方式

### 方式 1：命令列傳遞

```bash
ansible-playbook playbook.yml --ask-become-pass \
  -e clawdbot_install_mode=development \
  -e "clawdbot_ssh_keys=['ssh-ed25519 AAAAC3... user@host']"
```

### 方式 2：變數檔案

```yaml
# vars.yml
clawdbot_install_mode: development
clawdbot_ssh_keys:
  - "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGxxxxxxxx user@host"
clawdbot_repo_url: "https://github.com/YOUR_USERNAME/clawdbot.git"
clawdbot_repo_branch: "main"
tailscale_authkey: "tskey-auth-xxxxxxxxxxxxx"
```

```bash
ansible-playbook playbook.yml --ask-become-pass -e @vars.yml
```

### 方式 3：Ansible Vault（敏感資料）

```bash
# 建立加密檔案
ansible-vault create secrets.yml
# 內容：vault_tailscale_authkey: tskey-auth-xxxxx

# 使用加密變數
ansible-playbook playbook.yml --ask-become-pass \
  -e @secrets.yml --ask-vault-pass
```

---

## CI/CD 流程

### GitHub Actions 工作流程

定義於 `.github/workflows/lint.yml`，在 push 到 `main`/`development` 分支或建立 PR 時自動執行：

| Job | 工具 | 說明 |
|-----|------|------|
| `yaml-lint` | yamllint | 檢查所有 YAML 檔案格式 |
| `ansible-lint` | ansible-lint | 檢查 Ansible 最佳實踐 |
| `syntax-check` | ansible-playbook --syntax-check | 驗證 Playbook 語法 |

### 本機測試

```bash
# 語法檢查
ansible-playbook playbook.yml --syntax-check

# 乾跑（不實際執行）
ansible-playbook playbook.yml --check

# 乾跑 + 顯示差異
ansible-playbook playbook.yml --check --diff

# 完整安裝（在測試 VM 上）
ansible-playbook playbook.yml --ask-become-pass
```

---

## 主機上的檔案佈局

安裝完成後，以下是主機上的重要檔案與目錄：

### 系統級設定

```
/etc/docker/daemon.json              # Docker daemon 設定
/etc/ufw/after.rules                 # DOCKER-USER iptables 規則
/etc/fail2ban/jail.local             # Fail2ban SSH 防護設定
/etc/sudoers.d/clawdbot              # 範圍限定 sudo 權限
/etc/apt/apt.conf.d/20auto-upgrades  # 自動更新設定
/etc/apt/apt.conf.d/50unattended-upgrades  # 安全更新來源
/etc/vim/vimrc.local                 # 全域 Vim 設定
/etc/apt/sources.list.d/docker.list  # Docker apt 套件庫
/etc/apt/sources.list.d/nodesource.list  # NodeSource apt 套件庫
/etc/apt/sources.list.d/tailscale.list   # Tailscale apt 套件庫
```

### 使用者級設定

```
/home/clawdbot/
├── .clawdbot/                  # Clawdbot 設定與資料
│   ├── clawdbot.json           # 應用設定（由 clawdbot onboard 建立）
│   ├── sessions/               # 工作階段資料
│   ├── credentials/            # 認證資料（0700 權限）
│   ├── data/                   # 應用資料
│   └── logs/                   # 應用日誌
├── .local/
│   ├── bin/
│   │   └── clawdbot            # 可執行檔或 symlink
│   └── share/pnpm/             # pnpm 全域套件
├── .ssh/
│   └── authorized_keys         # SSH 公鑰
├── .oh-my-zsh/                 # Oh My Zsh
├── .bashrc                     # Bash 設定（環境變數、別名）
├── .zshrc                      # Zsh 設定
│
└── code/                       # 僅 Development 模式
    └── clawdbot/               # Git 儲存庫
        ├── bin/clawdbot.js
        ├── dist/
        ├── src/
        └── package.json
```

---

## 疑難排解

### Ansible Playbook 執行失敗

| 問題 | 解決方式 |
|------|---------|
| Collection 缺少 | `ansible-galaxy collection install -r requirements.yml` |
| 權限不足 | 使用 `--ask-become-pass` 或以 root 執行 |
| Docker daemon 未運行 | `sudo systemctl start docker` 後重新執行 |
| apt 鎖定 | 等待其他 apt 進程完成或 `sudo rm /var/lib/dpkg/lock-frontend` |

### 容器無法存取網路

```bash
# 檢查 DOCKER-USER 是否允許已建立連線
sudo iptables -L DOCKER-USER -n -v

# 重啟 Docker + UFW
sudo systemctl restart docker
sudo ufw reload
```

### 埠口衝突

```bash
# 找出佔用 3000 埠的進程
sudo ss -tlnp | grep 3000

# 修改 Clawdbot 埠口
# 編輯 docker-compose.yml 或設定檔
```

### 防火牆鎖定（無法 SSH）

```bash
# 透過主機控制台存取
sudo ufw disable
sudo ufw allow 22/tcp
sudo ufw enable
```

### DBus 問題

```bash
# 檢查環境變數
echo $XDG_RUNTIME_DIR
echo $DBUS_SESSION_BUS_ADDRESS

# 手動設定
export XDG_RUNTIME_DIR=/run/user/$(id -u)
source ~/.bashrc
```

---

## 解除安裝

```bash
# 停止服務
sudo systemctl stop clawdbot
sudo systemctl disable clawdbot
sudo tailscale down

# 移除容器與資料
sudo docker compose -f /opt/clawdbot/docker-compose.yml down
sudo rm -rf /opt/clawdbot
sudo rm -rf /home/clawdbot/.clawdbot
sudo rm -f /etc/systemd/system/clawdbot.service
sudo systemctl daemon-reload

# 移除套件（選擇性）
sudo apt remove --purge tailscale docker-ce docker-ce-cli containerd.io docker-compose-plugin nodejs

# 移除使用者（選擇性）
sudo userdel -r clawdbot

# 重置防火牆（選擇性）
sudo ufw disable
sudo ufw --force reset
```
