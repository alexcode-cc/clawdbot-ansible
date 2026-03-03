# 專案總覽

## 專案背景

OpenClaw Ansible Installer 是一個自動化安裝腳本，用於在 Debian/Ubuntu Linux 系統上部署 [OpenClaw](https://github.com/openclaw/openclaw) — 一個 AI 訊息閘道器（Gateway），可連接 WhatsApp、Telegram、Signal 等即時通訊平台。

本專案的核心設計理念為：

- **安全優先（Security First）**：多層防禦，預設封鎖所有不必要的連線
- **一鍵安裝（One Command Install）**：`curl | bash` 即可完成全部設定
- **僅限本機（Localhost Only）**：所有容器埠口綁定 `127.0.0.1`
- **縱深防禦（Defense in Depth）**：UFW + DOCKER-USER + localhost 綁定 + 非 root 容器

> **平台支援**：本安裝器目前**僅支援 Debian/Ubuntu Linux**。macOS 已從主 Playbook 中移除支援（會在 pre_tasks 明確報錯退出）。如需在 Mac Mini M4 上部署，請參閱 [Docker Ubuntu 部署指南](./docker-ubuntu-deployment.md)。

## 功能摘要

| 功能 | 說明 |
|------|------|
| 防火牆優先 | UFW + Docker 隔離（DOCKER-USER 鏈） |
| SSH 暴力破解防護 | Fail2ban，5 次失敗後封鎖 1 小時 |
| 自動安全更新 | unattended-upgrades 自動套用安全補丁 |
| Tailscale VPN | 安全遠端存取，無需暴露服務 |
| Docker | Docker CE + Compose V2 |
| 雙安裝模式 | Release（npm 套件）與 Development（原始碼建構） |
| 非 Root 容器 | OpenClaw 容器以 openclaw 使用者執行 |
| pnpm 安裝 | 使用 `pnpm install -g openclaw@latest` |

## 技術棧

```
┌─────────────────────────────────────────────────┐
│                 基礎設施層                        │
│  Ansible 2.14+ / Python 3 / Jinja2              │
├─────────────────────────────────────────────────┤
│                 作業系統層                        │
│  Debian 11+ / Ubuntu 20.04+                     │
├─────────────────────────────────────────────────┤
│                 安全層                            │
│  UFW / DOCKER-USER / Fail2ban / unattended-upgrades │
├─────────────────────────────────────────────────┤
│                 容器化層                          │
│  Docker CE / Docker Compose V2                   │
├─────────────────────────────────────────────────┤
│                 執行環境層                        │
│  Node.js 22.x / pnpm                            │
├─────────────────────────────────────────────────┤
│                 網路層                            │
│  Tailscale VPN / SSH / localhost 綁定             │
├─────────────────────────────────────────────────┤
│                 應用層                            │
│  OpenClaw AI Gateway                             │
│  (WhatsApp / Telegram / Signal)                  │
└─────────────────────────────────────────────────┘
```

## 專案目錄結構

```
openclaw-ansible/
├── .github/
│   └── workflows/
│       └── lint.yml                 # GitHub Actions CI — YAML/Ansible lint + 語法檢查
│
├── docs/                            # 英文技術文件
│   ├── architecture.md              # 架構說明
│   ├── configuration.md             # 配置指南
│   ├── development-mode.md          # 開發模式安裝指南
│   ├── installation.md              # 安裝說明
│   ├── security.md                  # 安全架構
│   └── troubleshooting.md           # 疑難排解
│
├── docs-cht/                        # 繁體中文技術文件（本目錄）
│
├── roles/
│   └── openclaw/                    # 主要 Ansible 角色
│       ├── defaults/
│       │   └── main.yml             # 預設變數定義
│       ├── files/
│       │   ├── clawdbot-setup.sh    # 安裝後設定腳本
│       │   └── show-lobster.sh      # 顯示龍蝦 ASCII 藝術
│       ├── handlers/
│       │   └── main.yml             # 服務重啟處理器（Docker、Fail2ban）
│       ├── tasks/
│       │   ├── main.yml             # 任務調度入口
│       │   ├── system-tools.yml     # 系統工具調度
│       │   ├── system-tools-linux.yml   # Linux 系統工具（apt）
│       │   ├── tailscale-linux.yml  # Linux Tailscale 安裝
│       │   ├── user.yml             # 系統使用者建立與配置
│       │   ├── docker-linux.yml     # Linux Docker CE 安裝
│       │   ├── firewall-linux.yml   # Linux UFW + Fail2ban 配置
│       │   ├── nodejs.yml           # Node.js 22.x + pnpm 安裝
│       │   ├── openclaw.yml         # OpenClaw 安裝調度
│       │   ├── openclaw-release.yml     # Release 模式安裝（npm）
│       │   └── openclaw-development.yml # Development 模式安裝（原始碼）
│       └── templates/
│           ├── daemon.json.j2               # Docker daemon 設定範本
│           ├── openclaw-config.yml.j2       # OpenClaw 設定檔範本
│           ├── openclaw-host.service.j2     # Systemd 服務單元範本
│           ├── show-lobster.sh.j2           # 龍蝦 ASCII 藝術腳本範本
│           └── vimrc.j2                     # Vim 編輯器設定範本
│
├── tests/                           # Docker 整合測試
│   ├── Dockerfile.ubuntu2404        # 測試用 Ubuntu 映像檔
│   ├── run-tests.sh                 # 測試執行腳本
│   └── verify.yml                   # 驗證 Playbook
│
├── .ansible-lint                    # Ansible Lint 規則設定
├── .gitignore                       # Git 忽略規則
├── .yamllint                        # YAML Lint 規則設定
├── AGENTS.md                        # AI Agent 開發指南
├── CHANGELOG.md                     # 版本更新日誌
├── install.sh                       # 一鍵安裝腳本（curl | bash 入口）
├── LICENSE                          # MIT 授權條款
├── playbook.yml                     # 主 Ansible Playbook
├── README.md                        # 專案主文件（英文）
├── RELEASE_NOTES_v2.0.0.md         # v2.0.0 發行說明
├── requirements.yml                 # Ansible Galaxy 集合依賴
├── run-playbook.sh                  # Playbook 執行包裝腳本
└── UPGRADE_NOTES.md                 # 升級指南
```

## 關鍵依賴

### Ansible Galaxy 集合

定義於 `requirements.yml`：

| 集合 | 最低版本 | 用途 |
|------|---------|------|
| `community.docker` | ≥3.4.0 | Docker Compose V2 管理 |
| `community.general` | ≥8.0.0 | UFW 防火牆、Git Config 等模組 |
| `ansible.posix` | ≥1.5.0 | SSH 授權金鑰管理 |

### 系統套件

**Linux（apt）**：
- Shells：zsh
- 編輯器：vim、nano
- 版本控制：git、git-lfs
- 網路工具：curl、wget、nmap、socat、telnet 等
- 除錯工具：strace、lsof、htop、iotop 等
- 系統工具：tmux、tree、jq、rsync 等
- 建構工具：build-essential

## 安裝模式比較

| 特性 | Release 模式（預設） | Development 模式 |
|------|---------------------|------------------|
| 來源 | npm 套件庫 | GitHub 儲存庫 |
| 安裝方式 | `pnpm install -g openclaw@latest` | `git clone` + `pnpm build` |
| 二進位位置 | pnpm 全域安裝 | Symlink → `~/code/openclaw/bin/openclaw.js` |
| 更新方式 | `pnpm install -g openclaw@latest` | `git pull` + `pnpm build` |
| 磁碟使用 | ~150 MB | ~400 MB |
| 適用場景 | 正式環境 | 開發與測試 |
| 額外功能 | — | `openclaw-rebuild`、`openclaw-dev`、`openclaw-pull` 別名 |

## 平台支援矩陣

| 功能 | Debian/Ubuntu | macOS | 狀態 |
|------|--------------|-------|------|
| 基礎安裝 | ✅ | ❌ | 僅 Linux |
| Docker | Docker CE | — | 僅 Linux |
| 防火牆 | UFW + DOCKER-USER | — | 僅 Linux |
| Fail2ban | ✅ | ❌ | 僅 Linux |
| systemd | ✅ | ❌ | 僅 Linux |
| 自動安全更新 | ✅ | ❌ | 僅 Linux |
| Tailscale | ✅ | — | 僅 Linux（主 Playbook） |
| pnpm + OpenClaw | ✅ | — | 僅 Linux（主 Playbook） |

> macOS 使用者請參閱 [Docker Ubuntu 部署指南](./docker-ubuntu-deployment.md)，以 Docker 容器方式取得完整 Linux 相容性。

## 版本歷程

目前版本為 **v2.0.0**，主要變更包含：

- 專案由 Clawdbot 更名為 OpenClaw
- 移除 macOS 支援（主 Playbook 明確拒絕 Darwin 系統）
- 新增 Development 安裝模式
- 改善 pnpm 安裝流程
- 移除自動建立 config.yml（改由 `openclaw onboard` 處理）
- 新增 Docker 容器化執行架構
- 新增 `ansible.posix` 集合依賴

詳細變更請參考 `CHANGELOG.md`。
