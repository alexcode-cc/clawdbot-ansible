# 架構分析

## Ansible Playbook 結構

### 主 Playbook（playbook.yml）

整個安裝流程由 `playbook.yml` 統一調度，結構分為三大區段：

```
playbook.yml
├── pre_tasks     ← 前置作業（OS 偵測、apt 升級、安裝 ACL 與 Ansible 集合）
├── roles         ← 主要角色執行（openclaw）
└── post_tasks    ← 後續作業（歡迎訊息、ASCII 藝術顯示）
```

#### 執行配置

```yaml
hosts: localhost          # 在本機執行
connection: local         # 使用本地連線
become: true              # 以 root 權限執行
```

**重要設計**：本 Playbook 設計為在目標機器上直接執行，而非透過 SSH 遠端部署。

#### pre_tasks 前置作業

| 步驟 | 說明 |
|------|------|
| 1. 啟用色彩終端 | 設定 `TERM=xterm-256color` |
| 2. 偵測作業系統 | 設定 `is_macos`、`is_linux`、`is_debian` fact |
| 3. 拒絕 macOS | 若偵測到 Darwin 系統，輸出錯誤訊息並終止 |
| 4. 更新系統套件 | `apt update && apt dist-upgrade`（僅 Debian/Ubuntu） |
| 5. 安裝 ACL | 用於權限提升（`acl` 套件） |
| 6. 安裝 Ansible 集合 | `ansible-galaxy collection install -r requirements.yml` |

#### post_tasks 後續作業

| 步驟 | 說明 |
|------|------|
| 1. 顯示 ASCII 藝術 | 龍蝦圖案 + 安裝成功訊息（`show-lobster.sh.j2`） |
| 2. 建立歡迎訊息 | 首次登入 openclaw 使用者時顯示的操作指南 |
| 3. 通知完成 | 輸出安裝完成訊息 |

---

## 角色（Role）設計

本專案只有一個角色 `openclaw`，採用 **調度器模式（Orchestrator Pattern）**：

### 任務執行順序

```
roles/openclaw/tasks/main.yml（調度入口）
│
├── 1. system-tools.yml         ← 系統工具安裝
│   └── system-tools-linux.yml  ← apt 安裝（Linux）
│
├── 2. tailscale-linux.yml      ← Tailscale VPN 安裝
│   （條件：tailscale_enabled == true）
│
├── 3. user.yml                 ← 使用者建立與配置
│   ├── 建立 openclaw 系統使用者
│   ├── 配置 scoped sudoers
│   ├── 建立 .ssh 目錄
│   └── 設定 SSH 授權金鑰
│
├── 4. docker-linux.yml         ← Docker CE 安裝
│   （條件：not ci_test）
│
├── 5. firewall-linux.yml       ← 防火牆配置
│   ├── UFW + Fail2ban
│   ├── DOCKER-USER 鏈
│   └── daemon.json
│   （條件：not ci_test）
│
├── 6. nodejs.yml               ← Node.js + pnpm 安裝
│
└── 7. openclaw.yml             ← OpenClaw 應用安裝
    ├── openclaw-release.yml    ← pnpm install -g（Release 模式）
    └── openclaw-development.yml ← git clone + build（Dev 模式）
```

### 任務順序的重要性

**Docker 必須在防火牆之前安裝**，原因如下：

1. `firewall-linux.yml` 需要寫入 `/etc/docker/daemon.json`，這要求 `/etc/docker` 目錄已存在
2. `firewall-linux.yml` 的 handler 需要重啟 Docker 服務，所以 Docker 服務必須已安裝
3. 如果順序顛倒，`daemon.json` 的目標目錄不存在，Ansible 會失敗

---

## 變數系統

### 預設變數（defaults/main.yml）

所有可配置的變數都定義在 `roles/openclaw/defaults/main.yml`：

| 變數 | 預設值 | 說明 |
|------|--------|------|
| `ci_test` | `false` | CI 測試模式（跳過 Docker/防火牆） |
| `tailscale_enabled` | `false` | 是否安裝 Tailscale |
| `tailscale_authkey` | `""` | Tailscale 驗證金鑰（留空需手動連線） |
| `nodejs_version` | `"22.x"` | Node.js 版本 |
| `openclaw_port` | `3000` | OpenClaw 閘道器埠口 |
| `openclaw_config_dir` | `{{ openclaw_home }}/.openclaw` | 設定檔目錄 |
| `openclaw_user` | `openclaw` | 系統使用者名稱 |
| `openclaw_home` | `/home/openclaw` | 使用者家目錄 |
| `openclaw_install_mode` | `"release"` | 安裝模式（`release` 或 `development`） |
| `openclaw_repo_url` | `https://github.com/openclaw/openclaw.git` | Git 儲存庫 URL（Dev 模式） |
| `openclaw_repo_branch` | `"main"` | Git 分支（Dev 模式） |
| `openclaw_code_dir` | `{{ openclaw_home }}/code` | 原始碼目錄（Dev 模式） |
| `openclaw_repo_dir` | `{{ openclaw_code_dir }}/openclaw` | 儲存庫路徑（Dev 模式） |
| `openclaw_ssh_keys` | `[]` | SSH 公鑰清單 |

### 動態 Facts

在 `playbook.yml` 的 `pre_tasks` 中設定的動態 facts：

| Fact | 用途 |
|------|------|
| `is_macos` | 是否為 macOS（若為 true 則終止） |
| `is_linux` | 是否為 Linux（Debian 系列） |
| `is_debian` | 是否為 Debian 或 Ubuntu |

### 變數覆寫優先順序

Ansible 的變數優先順序（由低到高）：

1. `roles/openclaw/defaults/main.yml`（最低優先）
2. `vars.yml` 檔案（`-e @vars.yml`）
3. 命令列變數（`-e key=value`）（最高優先）

---

## Handlers（處理器）

定義於 `roles/openclaw/handlers/main.yml`：

| Handler | 觸發條件 | 動作 |
|---------|----------|------|
| `Restart docker` | `daemon.json` 變更 | 重啟 Docker 服務 |
| `Restart fail2ban` | `jail.local` 變更 | 重啟 Fail2ban 服務 |

Handlers 的特性：
- 僅在觸發它的任務狀態為 `changed` 時才會執行
- 在角色所有任務完成後統一執行（而非立即執行）
- 多次觸發同一 handler 只會執行一次

---

## Templates（Jinja2 範本）

### daemon.json.j2

Docker Daemon 的設定檔，用於 UFW 與 Docker 的協作：

```json
{
  "iptables": true,        // 允許 Docker 管理 iptables（必要！不可設為 false）
  "ip-forward": true,       // 啟用 IP 轉發
  "userland-proxy": false,  // 停用使用者空間代理（提升效能）
  "live-restore": true,     // Docker daemon 重啟時保持容器運行
  "ip6tables": false,       // 停用 IPv6（簡化防火牆規則）
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",      // 日誌檔案最大 10MB
    "max-file": "3"         // 最多保留 3 個日誌檔案
  },
  "default-address-pools": [
    {
      "base": "172.17.0.0/12",
      "size": 24
    }
  ]
}
```

**關鍵設計**：`iptables: true` 是必要的。若設為 `false`，Docker 將無法建立容器網路，DOCKER-USER chain 也不會被評估。

### openclaw-host.service.j2

Systemd 服務單元，包含安全強化設定：

```ini
[Service]
Type=simple
User=openclaw
NoNewPrivileges=true       # 禁止提權
PrivateTmp=true            # 隔離 /tmp
ProtectSystem=strict       # 系統目錄唯讀
ProtectHome=read-only      # 家目錄唯讀
ReadWritePaths=~/.openclaw # 僅允許寫入設定目錄
ReadWritePaths=~/.local    # 允許寫入本地資料
```

### openclaw-config.yml.j2

OpenClaw 應用程式的設定範本，包含：
- 訊息提供者設定（WhatsApp/Telegram/Signal）
- AI 模型設定（Anthropic/OpenAI）
- 閘道器設定（埠口、Web UI）
- 日誌設定
- 安全設定（白名單、速率限制）

> **注意**：v2.0.0 起，此範本不再由 Ansible 自動部署。改由 `openclaw onboard` 指令在使用者互動時建立。

### show-lobster.sh.j2

安裝完成後顯示的龍蝦 ASCII 藝術腳本，動態顯示開放的埠口資訊（根據 `tailscale_enabled` 變數決定顯示內容）。

### vimrc.j2

全域 Vim 設定，提供開發友善的環境：
- 行號與相對行號
- 搜尋高亮
- 2 空格縮排
- 持久化 undo
- 滑鼠支援
- 狀態列增強
- 檔案類型特定設定（Python 4 空格、YAML 2 空格等）

---

## 資料流架構

### 安裝時資料流

```
使用者
  │
  ├── curl | bash ──→ install.sh
  │                    │
  │                    ├── 安裝 Ansible
  │                    ├── 複製 openclaw-ansible 儲存庫
  │                    ├── 安裝 Ansible Galaxy 集合
  │                    └── 執行 run-playbook.sh
  │                         │
  │                         └── ansible-playbook playbook.yml
  │                              │
  │                              ├── pre_tasks（環境準備、拒絕非 Linux）
  │                              ├── role: openclaw（主安裝邏輯）
  │                              └── post_tasks（歡迎訊息）
  │
  └── sudo docker exec -it openclaw openclaw login
       │
       └── 設定訊息提供者（WhatsApp/Telegram/Signal）
```

### 執行時資料流

```
systemd (openclaw.service)
  │
  └── docker compose（/opt/openclaw/docker-compose.yml）
       │
       └── openclaw 容器（Docker）
            │
            ├── 讀取 /home/openclaw/.openclaw/config.yml（volume 掛載）
            ├── 連線訊息提供者（WhatsApp/Telegram/Signal）
            ├── 連線 AI API（Anthropic/OpenAI）
            └── 監聽 127.0.0.1:3000（Web UI）
```

### 網路流量路徑

```
外部網路
  │
  ├── SSH (22/tcp) ──→ UFW ALLOW ──→ Fail2ban ──→ sshd
  ├── Tailscale (41641/udp) ──→ UFW ALLOW ──→ tailscaled
  ├── 其他埠口 ──→ UFW DENY ──→ 丟棄
  │
  └── Docker 容器流量
       │
       └── DOCKER-USER chain
            ├── ESTABLISHED/RELATED ──→ ACCEPT
            ├── localhost (lo) ──→ ACCEPT
            └── 外部介面 ──→ DROP
```

---

## 主機檔案佈局

```
/opt/openclaw/
├── Dockerfile
├── docker-compose.yml

/home/openclaw/.openclaw/
├── config.yml
├── sessions/
└── credentials/

/etc/systemd/system/
└── openclaw.service

/etc/docker/
└── daemon.json

/etc/ufw/
└── after.rules (DOCKER-USER chain)
```

---

## 設定管理策略

### Ansible 管理的設定

由 Ansible 安裝時建立並管理的設定：

| 設定檔 | 路徑 | 管理者 |
|--------|------|--------|
| Docker Daemon | `/etc/docker/daemon.json` | Ansible（template） |
| UFW after.rules | `/etc/ufw/after.rules` | Ansible（blockinfile） |
| Fail2ban jail | `/etc/fail2ban/jail.local` | Ansible（copy） |
| Sudoers | `/etc/sudoers.d/openclaw` | Ansible（copy） |
| Vim 設定 | `/etc/vim/vimrc.local` | Ansible（template） |
| 自動更新 | `/etc/apt/apt.conf.d/20auto-upgrades` | Ansible（copy） |
| 自動更新 | `/etc/apt/apt.conf.d/50unattended-upgrades` | Ansible（copy） |

### 使用者管理的設定

安裝後由使用者或 OpenClaw 管理的設定：

| 設定檔 | 路徑 | 管理者 |
|--------|------|--------|
| OpenClaw 設定 | `~/.openclaw/config.yml` | `openclaw onboard` 或手動編輯 |
| Docker Compose | `/opt/openclaw/docker-compose.yml` | Ansible（初始建立） |
| Sessions | `~/.openclaw/sessions/` | OpenClaw 執行時 |
| Credentials | `~/.openclaw/credentials/` | OpenClaw 執行時 |

---

## 冪等性（Idempotency）設計

本 Playbook 設計為可重複執行：

| 機制 | 範例 |
|------|------|
| `creates` 參數 | GPG key 下載使用 `creates:` 避免重複 |
| `stat` 模組 | 安裝前檢查應用是否已存在 |
| `changed_when` | 正確標記任務是否造成變更 |
| `when` 條件 | 根據環境狀態跳過不必要的任務 |
| Handler 去重 | 同一 handler 多次觸發只執行一次 |

這表示你可以安全地多次執行 Playbook，不會破壞已有的設定。
