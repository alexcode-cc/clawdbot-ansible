---
title: Docker Ubuntu 24.04 部署指南
description: 在 Mac Mini M4 上使用 Docker Ubuntu 24.04 容器部署 Clawdbot 的完整說明
---

# Docker Ubuntu 24.04 部署指南（Mac Mini M4）

在 Mac Mini M4 上透過 Docker 執行 Ubuntu 24.04 容器，並在容器內依照標準 Ubuntu 部署流程安裝 Clawdbot。

## 為什麼選擇 Docker Ubuntu？

直接在 macOS 上部署 Clawdbot 會遇到多項相容性問題（參見 [Mac Mini M4 部署指南](./mac-mini-m4-deployment.md)）。使用 Docker Ubuntu 容器的優點：

| 特性 | 原生 macOS | Docker Ubuntu |
|------|-----------|---------------|
| Ansible Playbook 完整支援 | ❌ 部分失敗 | ✅ 全部通過 |
| UFW + DOCKER-USER 防火牆 | ❌ | ✅（需 privileged） |
| Fail2ban SSH 防護 | ❌ | ✅ |
| systemd 服務管理 | ❌ | ✅（需 systemd 映像檔） |
| unattended-upgrades | ❌ | ✅ |
| Node.js (apt) 安裝 | ❌ | ✅ |
| Tailscale VPN | ✅（原生 App） | ⚠️ 需額外設定 |

**簡言之**：Docker Ubuntu 方案讓你在 Mac Mini M4 上獲得與 Linux 伺服器幾乎一致的部署體驗。

---

## 目錄

- [前置需求](#前置需求)
- [架構概覽](#架構概覽)
- [步驟一：安裝 Docker Desktop](#步驟一安裝-docker-desktop)
- [步驟二：建立 Ubuntu 24.04 容器](#步驟二建立-ubuntu-2404-容器)
- [步驟三：容器內環境準備](#步驟三容器內環境準備)
- [步驟四：執行 Ansible Playbook](#步驟四執行-ansible-playbook)
- [步驟五：安裝後配置](#步驟五安裝後配置)
- [容器內 Docker（Docker-in-Docker）](#容器內-dockerdocker-in-docker)
- [Tailscale VPN 設定](#tailscale-vpn-設定)
- [資料持久化](#資料持久化)
- [容器生命週期管理](#容器生命週期管理)
- [網路與埠口轉發](#網路與埠口轉發)
- [驗證清單](#驗證清單)
- [疑難排解](#疑難排解)
- [進階：使用 Docker Compose 管理](#進階使用-docker-compose-管理)

---

## 前置需求

### Mac Mini M4 主機端

- macOS 15 (Sequoia) 或更新版本
- **Docker Desktop** 已安裝並執行
- 建議 16 GB 以上 RAM（Docker 容器至少分配 4 GB）
- 50 GB 以上可用磁碟空間

### 安裝 Docker Desktop

如果尚未安裝：

```bash
# 安裝 Homebrew（如果還沒有）
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
eval "$(/opt/homebrew/bin/brew shellenv)"

# 安裝 Docker Desktop
brew install --cask docker

# 啟動 Docker Desktop
open /Applications/Docker.app

# 等待 Docker 啟動完成後驗證
docker --version
docker compose version
```

### Docker Desktop 建議設定

開啟 Docker Desktop → **Settings**：

- **General**：勾選「Use Virtualization Framework」
- **Resources** → **Advanced**：
  - CPUs：至少 4 核
  - Memory：至少 4 GB（建議 8 GB）
  - Disk image size：至少 40 GB

---

## 架構概覽

```
┌─────────────────────────────────────────────────────────┐
│  Mac Mini M4（macOS 主機）                               │
│                                                         │
│  ┌───────────────────────────────────────────────────┐  │
│  │  Docker Desktop (Apple Virtualization Framework)  │  │
│  │                                                   │  │
│  │  ┌─────────────────────────────────────────────┐  │  │
│  │  │  Ubuntu 24.04 容器（privileged 模式）        │  │  │
│  │  │                                             │  │  │
│  │  │  ┌─────────┐ ┌──────────┐ ┌─────────────┐   │  │  │
│  │  │  │ systemd │ │ Clawdbot │ │ UFW/Fail2ban│   │  │  │
│  │  │  └─────────┘ └──────────┘ └─────────────┘   │  │  │
│  │  │  ┌─────────┐ ┌──────────┐ ┌─────────────┐   │  │  │
│  │  │  │ Docker  │ │ Node.js  │ │ Tailscale   │   │  │  │
│  │  │  │  (DinD) │ │  22.x    │ │  (optional) │   │  │  │
│  │  │  └─────────┘ └──────────┘ └─────────────┘   │  │  │
│  │  │                                             │  │  │
│  │  │  Port: 127.0.0.1:3000 → Host:3000           │  │  │
│  │  └─────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────┘  │
│                                                         │
│  瀏覽器存取: http://localhost:3000                       │
└─────────────────────────────────────────────────────────┘
```

---

## 步驟一：安裝 Docker Desktop

（已在「前置需求」說明，確認 Docker Desktop 正常運作後繼續。）

```bash
# 確認 Docker 正常運行
docker info
```

---

## 步驟二：建立 Ubuntu 24.04 容器

### 方式 A：啟用 systemd 的容器（推薦）

使用 `--privileged` 模式並以 systemd 作為 init 進程，讓 systemd、UFW、Fail2ban 都能正常運作：

```bash
# 建立持久化資料的 Docker volumes
docker volume create clawdbot-home
docker volume create clawdbot-config

# 建立並啟動容器
docker run -d \
  --name clawdbot-ubuntu \
  --hostname clawdbot-server \
  --privileged \
  --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  -v clawdbot-home:/home/clawdbot \
  -v clawdbot-config:/opt/clawdbot-ansible \
  -p 127.0.0.1:3000:3000 \
  -p 127.0.0.1:2222:22 \
  --tmpfs /run \
  --tmpfs /run/lock \
  ubuntu:24.04 \
  /sbin/init
```

參數說明：

| 參數 | 說明 |
|------|------|
| `--privileged` | 啟用特權模式，systemd/iptables/UFW 需要 |
| `--cgroupns=host` | 使用主機的 cgroup namespace，systemd 需要 |
| `-v /sys/fs/cgroup:...` | 掛載 cgroup 檔案系統 |
| `-v clawdbot-home:...` | 持久化 clawdbot 使用者家目錄 |
| `-v clawdbot-config:...` | 持久化 Ansible playbook |
| `-p 127.0.0.1:3000:3000` | 轉發 Clawdbot 閘道器埠口 |
| `-p 127.0.0.1:2222:22` | 轉發 SSH 埠口（用於遠端管理） |
| `--tmpfs /run` | systemd 需要的暫存檔系統 |
| `/sbin/init` | 以 systemd 為 PID 1 啟動 |

等待容器啟動（約 5-10 秒）：

```bash
# 確認容器正在運行
docker ps

# 確認 systemd 已啟動
docker exec clawdbot-ubuntu systemctl status --no-pager
```

### 方式 B：簡易容器（不含 systemd）

如果不需要 systemd（僅做開發測試），使用簡易方式：

```bash
docker run -d \
  --name clawdbot-ubuntu \
  --hostname clawdbot-server \
  -v clawdbot-home:/home/clawdbot \
  -p 127.0.0.1:3000:3000 \
  ubuntu:24.04 \
  tail -f /dev/null
```

> **注意**：此方式下 systemd、UFW、Fail2ban 無法運作。Ansible Playbook 中依賴 systemd 的任務會失敗，需要手動跳過。

---

## 步驟三：容器內環境準備

### 進入容器

```bash
docker exec -it clawdbot-ubuntu bash
```

### 安裝基礎套件

```bash
# 更新套件清單
apt update && apt upgrade -y

# 安裝 Ansible 及必要工具
apt install -y \
  ansible \
  git \
  curl \
  wget \
  sudo \
  python3 \
  python3-pip \
  ca-certificates \
  gnupg \
  lsb-release \
  software-properties-common \
  systemd \
  dbus

# 驗證
ansible --version
python3 --version
git --version
```

### 複製 Ansible Playbook

```bash
# 進入掛載目錄
cd /opt/clawdbot-ansible

# 複製儲存庫
git clone https://github.com/pasogott/clawdbot-ansible.git .

# 安裝 Ansible Galaxy 集合
ansible-galaxy collection install -r requirements.yml
```

---

## 步驟四：執行 Ansible Playbook

### Release 模式（推薦）

```bash
# 在容器內以 root 執行（容器內已是 root）
cd /opt/clawdbot-ansible
ansible-playbook playbook.yml -e ansible_become=false
```

### Development 模式

```bash
cd /opt/clawdbot-ansible
ansible-playbook playbook.yml \
  -e ansible_become=false \
  -e clawdbot_install_mode=development
```

### 帶自訂變數

```bash
ansible-playbook playbook.yml \
  -e ansible_become=false \
  -e "clawdbot_ssh_keys=['ssh-ed25519 AAAAC3... user@host']"
```

### Playbook 執行過程

在容器內執行時，以下任務將依序完成：

```
✅ 1. system-tools.yml    → apt 安裝系統工具（完整支援）
✅ 2. tailscale.yml       → Tailscale 安裝（可選跳過）
✅ 3. user.yml            → 建立 clawdbot 使用者、sudoers、SSH 金鑰
✅ 4. docker.yml          → 安裝 Docker CE（Docker-in-Docker）
✅ 5. firewall.yml        → UFW + Fail2ban + DOCKER-USER（需 privileged）
✅ 6. nodejs.yml          → Node.js 22.x + pnpm（apt 安裝）
✅ 7. clawdbot.yml        → 安裝 Clawdbot（pnpm 或原始碼建構）
```

> **與原生 macOS 的差異**：所有 7 個步驟在 Docker Ubuntu 中都能完整執行，不需要手動補丁。

### 處理可能的錯誤

**如果 Homebrew 安裝失敗**：

Playbook 的 `pre_tasks` 會嘗試安裝 Homebrew。在容器環境中可能因缺少互動式 shell 而失敗。解決方式：

```bash
# 手動安裝 Homebrew（以非 root 使用者）
# 先建立 clawdbot 使用者
useradd -m -s /bin/bash clawdbot
su - clawdbot -c 'NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"'

# 然後重新執行 Playbook
ansible-playbook playbook.yml -e ansible_become=false
```

**如果 Docker CE 安裝後無法啟動**：

Docker-in-Docker 需要 `--privileged` 模式。如果使用了方式 B（簡易容器），Docker 服務會啟動失敗。參見「[容器內 Docker](#容器內-dockerdocker-in-docker)」章節。

**如果 Tailscale 啟動失敗**：

Tailscale 在容器內需要 TUN 裝置。參見「[Tailscale VPN 設定](#tailscale-vpn-設定)」章節。

---

## 步驟五：安裝後配置

### 切換到 clawdbot 使用者

```bash
# 在容器內
sudo su - clawdbot
```

### 確認環境

```bash
echo "使用者: $(whoami)"
echo "家目錄: $HOME"
echo "Node.js: $(node --version 2>/dev/null || echo '未找到')"
echo "pnpm: $(pnpm --version 2>/dev/null || echo '未找到')"
echo "Clawdbot: $(~/.local/bin/clawdbot --version 2>/dev/null || echo '未找到')"
echo "Docker: $(docker --version 2>/dev/null || echo '未找到')"
```

### 執行初始設定

```bash
clawdbot onboard --install-daemon
```

此指令會：
- 建立設定檔 `~/.clawdbot/clawdbot.json`
- 引導選擇訊息提供者（WhatsApp / Telegram / Signal）
- 設定 AI 模型 API 金鑰
- 安裝 systemd daemon 並啟動服務

### 驗證服務執行

```bash
# 檢查 systemd 服務狀態
sudo systemctl status clawdbot

# 查看日誌
sudo journalctl -u clawdbot -f

# 測試閘道器
curl http://localhost:3000
```

### 從 Mac 主機存取

Clawdbot 閘道器已透過 `-p 127.0.0.1:3000:3000` 轉發到主機：

```bash
# 在 Mac 主機上
curl http://localhost:3000

# 或用瀏覽器開啟
open http://localhost:3000
```

---

## 容器內 Docker（Docker-in-Docker）

Clawdbot 使用 Docker 來執行沙箱環境。在容器內有兩種方式使用 Docker：

### 方式 A：Docker-in-Docker（DinD）— Playbook 預設方式

如果使用 `--privileged` 模式啟動容器，Ansible Playbook 會在容器內安裝完整的 Docker CE。這是最接近實體 Linux 伺服器的方式。

```bash
# 容器內確認 Docker 運作
docker info
docker run --rm hello-world
```

### 方式 B：掛載主機 Docker Socket

另一種方式是讓容器共享 Mac 主機的 Docker。需要重新建立容器：

```bash
# 停止並移除舊容器
docker stop clawdbot-ubuntu && docker rm clawdbot-ubuntu

# 重新建立，掛載 Docker socket
docker run -d \
  --name clawdbot-ubuntu \
  --hostname clawdbot-server \
  --privileged \
  --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v clawdbot-home:/home/clawdbot \
  -v clawdbot-config:/opt/clawdbot-ansible \
  -p 127.0.0.1:3000:3000 \
  -p 127.0.0.1:2222:22 \
  --tmpfs /run \
  --tmpfs /run/lock \
  ubuntu:24.04 \
  /sbin/init
```

然後在容器內安裝 Docker CLI（不含 daemon）：

```bash
docker exec -it clawdbot-ubuntu bash

# 只安裝 CLI 工具
apt update
apt install -y docker-ce-cli docker-compose-plugin

# 驗證（使用主機 Docker）
docker ps
```

> **注意**：使用方式 B 時，需要在執行 Playbook 前跳過 Docker 安裝步驟，或在 Playbook 的 `docker-linux.yml` 階段失敗後手動處理。

---

## Tailscale VPN 設定

### 在容器內使用 Tailscale

Tailscale 在 Docker 容器內需要 TUN 裝置和 NET_ADMIN 能力。如果使用 `--privileged` 模式，這些已自動啟用。

```bash
# 容器內確認 Tailscale 已安裝（由 Playbook 完成）
tailscale version

# 啟動 tailscaled（如果 systemd 未自動啟動）
sudo systemctl start tailscaled

# 連線到 Tailnet
sudo tailscale up

# 使用 auth key 自動連線
sudo tailscale up --authkey tskey-auth-xxxxx

# 檢查狀態
tailscale status
tailscale ip -4
```

### 替代方案：在主機使用 Tailscale

如果容器內的 Tailscale 有問題，可以在 Mac 主機上安裝 Tailscale，透過主機網路存取容器服務：

```bash
# Mac 主機上安裝 Tailscale
brew install --cask tailscale
open /Applications/Tailscale.app

# 容器的 3000 埠已轉發到主機 localhost:3000
# 遠端裝置透過 Tailscale 連到 Mac 後，用 SSH tunnel 存取：
ssh -L 3000:localhost:3000 user@mac-tailscale-ip
```

---

## 資料持久化

### Docker Volumes

建立容器時已掛載兩個 volume：

| Volume | 容器路徑 | 用途 |
|--------|---------|------|
| `clawdbot-home` | `/home/clawdbot` | 使用者資料、設定、認證 |
| `clawdbot-config` | `/opt/clawdbot-ansible` | Ansible Playbook |

即使容器被刪除重建，這些資料都會保留。

### 管理 Volume

```bash
# 查看 volume
docker volume ls

# 檢查 volume 詳細資訊
docker volume inspect clawdbot-home

# 備份 volume 資料
docker run --rm \
  -v clawdbot-home:/data \
  -v $(pwd):/backup \
  ubuntu:24.04 \
  tar czf /backup/clawdbot-home-backup.tar.gz -C /data .

# 還原 volume 資料
docker run --rm \
  -v clawdbot-home:/data \
  -v $(pwd):/backup \
  ubuntu:24.04 \
  bash -c "cd /data && tar xzf /backup/clawdbot-home-backup.tar.gz"
```

### 備份排程

在 Mac 主機上建立定期備份（使用 cron 或 launchd）：

```bash
# 加入 crontab（每天凌晨 3 點備份）
crontab -e
# 加入以下行：
# 0 3 * * * docker run --rm -v clawdbot-home:/data -v ~/backups:/backup ubuntu:24.04 tar czf /backup/clawdbot-$(date +\%Y\%m\%d).tar.gz -C /data .
```

---

## 容器生命週期管理

### 基本操作

```bash
# 啟動容器
docker start clawdbot-ubuntu

# 停止容器
docker stop clawdbot-ubuntu

# 重啟容器
docker restart clawdbot-ubuntu

# 進入容器 shell
docker exec -it clawdbot-ubuntu bash

# 以 clawdbot 使用者進入
docker exec -it -u clawdbot clawdbot-ubuntu bash

# 查看容器日誌
docker logs clawdbot-ubuntu
docker logs -f clawdbot-ubuntu  # 持續追蹤

# 查看容器資源使用
docker stats clawdbot-ubuntu
```

### 容器自動重啟

設定容器在 Docker Desktop 啟動時自動恢復：

```bash
# 設定自動重啟策略
docker update --restart unless-stopped clawdbot-ubuntu
```

這樣當 Mac 重新開機且 Docker Desktop 啟動後，容器會自動恢復運行。

### SSH 進入容器

如果需要透過 SSH 連線到容器（例如從其他裝置）：

```bash
# 容器內安裝並啟動 SSH
docker exec clawdbot-ubuntu bash -c "apt install -y openssh-server && systemctl enable ssh && systemctl start ssh"

# 設定 clawdbot 使用者的密碼或 SSH 金鑰
docker exec clawdbot-ubuntu bash -c "echo 'clawdbot:YOUR_PASSWORD' | chpasswd"

# 從 Mac 主機 SSH 到容器
ssh clawdbot@localhost -p 2222

# 從其他裝置（經過 Tailscale）
ssh clawdbot@mac-tailscale-ip -p 2222
```

---

## 網路與埠口轉發

### 預設埠口映射

| 主機埠口 | 容器埠口 | 服務 |
|---------|---------|------|
| `127.0.0.1:3000` | `3000` | Clawdbot Gateway |
| `127.0.0.1:2222` | `22` | SSH |

### 新增埠口映射

如果需要額外的埠口，需要重新建立容器。或者使用更靈活的方式：

```bash
# 方式 1：重建容器（增加 -p 參數）
docker stop clawdbot-ubuntu && docker rm clawdbot-ubuntu
# 然後重新執行 docker run，加上新的 -p 參數

# 方式 2：使用 SSH tunnel（不需重建容器）
# 從 Mac 主機轉發容器內的 8080 埠
ssh -L 8080:localhost:8080 clawdbot@localhost -p 2222
```

### 容器內防火牆

在 `--privileged` 模式下，Ansible Playbook 會配置 UFW 防火牆。但容器的網路流量實際上由 Docker 的網路層管理。

```bash
# 容器內檢查 UFW 狀態
sudo ufw status verbose

# 容器內檢查 DOCKER-USER 鏈（如果有 DinD）
sudo iptables -L DOCKER-USER -n -v 2>/dev/null || echo "DOCKER-USER 鏈未建立"
```

> **注意**：容器內的 UFW 主要保護容器自身的服務。Mac 主機與容器之間的流量由 Docker 網路管理，不受容器內 UFW 影響。

---

## 驗證清單

### 容器外（Mac 主機）

```bash
# 1. Docker 容器運行中
docker ps | grep clawdbot-ubuntu
# ✅ 容器狀態為 Up

# 2. 埠口轉發正常
curl -s http://localhost:3000 > /dev/null && echo "✅ Port 3000 OK" || echo "❌ Port 3000 Failed"

# 3. SSH 進入容器正常
ssh -o ConnectTimeout=5 clawdbot@localhost -p 2222 exit 2>/dev/null && echo "✅ SSH OK" || echo "⚠️ SSH not configured"

# 4. Volume 存在
docker volume ls | grep clawdbot
# ✅ clawdbot-home 和 clawdbot-config 存在
```

### 容器內

```bash
docker exec -it clawdbot-ubuntu bash

# 5. systemd 運行中
systemctl is-system-running
# ✅ running 或 degraded（部分服務可能不適用於容器）

# 6. Clawdbot 已安裝
sudo su - clawdbot -c '~/.local/bin/clawdbot --version'
# ✅ 顯示版本號

# 7. Node.js 已安裝
node --version
# ✅ v22.x.x

# 8. pnpm 已安裝
pnpm --version
# ✅ 顯示版本號

# 9. Docker（DinD 或 socket）可用
docker --version
# ✅ Docker 已可用

# 10. UFW 防火牆
sudo ufw status
# ✅ Status: active

# 11. Fail2ban
sudo fail2ban-client status sshd
# ✅ Jail 已啟用

# 12. Clawdbot 服務
sudo systemctl status clawdbot
# ✅ active (running)（需先完成 onboard）
```

---

## 疑難排解

### 容器啟動後立即退出

```bash
# 查看退出原因
docker logs clawdbot-ubuntu

# 常見原因：cgroup 版本不相容
# 解決方式：確認使用了 --cgroupns=host 參數
```

### systemd 無法啟動

```bash
# 檢查是否使用了 /sbin/init 作為入口
docker inspect clawdbot-ubuntu --format '{{.Config.Cmd}}'
# 應顯示 [/sbin/init]

# 檢查是否啟用了 --privileged
docker inspect clawdbot-ubuntu --format '{{.HostConfig.Privileged}}'
# 應顯示 true

# 如果仍然失敗，嘗試使用 systemd 專用映像檔
docker stop clawdbot-ubuntu && docker rm clawdbot-ubuntu
docker run -d \
  --name clawdbot-ubuntu \
  --hostname clawdbot-server \
  --privileged \
  --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup:rw \
  -v clawdbot-home:/home/clawdbot \
  -v clawdbot-config:/opt/clawdbot-ansible \
  -p 127.0.0.1:3000:3000 \
  -p 127.0.0.1:2222:22 \
  --tmpfs /run \
  --tmpfs /run/lock \
  jrei/systemd-ubuntu:24.04 \
  /sbin/init
```

### Ansible Playbook 失敗於 Homebrew 安裝

```bash
# 在容器內手動安裝 Homebrew
apt install -y build-essential procps curl file git
useradd -m -s /bin/bash linuxbrew 2>/dev/null || true
su - clawdbot -c 'NONINTERACTIVE=1 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"'

# 重新執行 Playbook
cd /opt/clawdbot-ansible
ansible-playbook playbook.yml -e ansible_become=false
```

### Docker-in-Docker 啟動失敗

```bash
# 檢查 Docker daemon 狀態
systemctl status docker

# 查看詳細錯誤
journalctl -u docker -n 50

# 嘗試手動啟動
dockerd &

# 如果仍然失敗，改用 socket 掛載方式
# 需要重新建立容器，加上 -v /var/run/docker.sock:/var/run/docker.sock
```

### 容器佔用過多磁碟空間

```bash
# 在容器內清理
apt autoremove -y
apt clean

# 清理 Docker（如果使用 DinD）
docker system prune -a

# 在 Mac 主機上查看 Docker 磁碟使用
docker system df
```

### 容器重啟後 Clawdbot 未自動啟動

```bash
# 確認 systemd 服務已啟用
docker exec clawdbot-ubuntu systemctl enable clawdbot

# 確認自動重啟策略
docker update --restart unless-stopped clawdbot-ubuntu
```

---

## 進階：使用 Docker Compose 管理

對於更複雜的部署場景，可以使用 Docker Compose 管理容器。

在 Mac 主機上建立 `docker-compose.yml`：

```yaml
services:
  clawdbot:
    image: ubuntu:24.04
    container_name: clawdbot-ubuntu
    hostname: clawdbot-server
    privileged: true
    cgroup_parent: ""
    command: /sbin/init
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2222:22"
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - clawdbot-home:/home/clawdbot
      - clawdbot-config:/opt/clawdbot-ansible
    tmpfs:
      - /run
      - /run/lock
    restart: unless-stopped

volumes:
  clawdbot-home:
  clawdbot-config:
```

管理指令：

```bash
# 啟動
docker compose up -d

# 停止
docker compose down

# 重啟
docker compose restart

# 查看日誌
docker compose logs -f

# 進入容器
docker compose exec clawdbot bash
```

---

## 快速參考卡

```
╔══════════════════════════════════════════════════════════╗
║        Docker Ubuntu 24.04 部署快速參考                   ║
╠══════════════════════════════════════════════════════════╣
║                                                          ║
║  建立容器：                                               ║
║    docker run -d --name clawdbot-ubuntu --privileged \   ║
║      --cgroupns=host -v /sys/fs/cgroup:...:rw \          ║
║      -v clawdbot-home:/home/clawdbot \                   ║
║      -p 127.0.0.1:3000:3000 --tmpfs /run \               ║
║      ubuntu:24.04 /sbin/init                             ║
║                                                          ║
║  進入容器：                                               ║
║    docker exec -it clawdbot-ubuntu bash                  ║
║                                                          ║
║  執行 Playbook：                                          ║
║    cd /opt/clawdbot-ansible                              ║
║    ansible-playbook playbook.yml -e ansible_become=false ║
║                                                          ║
║  設定 Clawdbot：                                          ║
║    sudo su - clawdbot                                    ║
║    clawdbot onboard --install-daemon                     ║
║                                                          ║
║  從 Mac 存取：                                            ║
║    curl http://localhost:3000                            ║
║                                                          ║
║  容器管理：                                               ║
║    docker start/stop/restart clawdbot-ubuntu             ║
║    docker logs -f clawdbot-ubuntu                        ║
║                                                          ║
║  備份：                                                   ║
║    docker run --rm -v clawdbot-home:/data \              ║
║      -v $(pwd):/bk ubuntu:24.04 \                        ║
║      tar czf /bk/backup.tar.gz -C /data .                ║
║                                                          ║
╚══════════════════════════════════════════════════════════╝
```
