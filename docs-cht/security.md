# 安全機制分析

## 多層防禦架構總覽

本專案實施了 **8 層防禦機制**，確保即使某一層被突破，其他層仍能提供保護：

```
┌───────────────────────────────────────────────────────┐
│ 第 1 層：UFW 防火牆（僅允許 SSH + Tailscale）          │
├───────────────────────────────────────────────────────┤
│ 第 2 層：Fail2ban（SSH 暴力破解防護）                  │
├───────────────────────────────────────────────────────┤
│ 第 3 層：DOCKER-USER iptables 鏈                      │
│（阻擋所有外部對 Docker 容器的存取）                     │
├───────────────────────────────────────────────────────┤
│ 第 4 層：Localhost 綁定（容器埠口僅綁定 127.0.0.1）    │
├───────────────────────────────────────────────────────┤
│ 第 5 層：非 Root 容器（以 clawdbot 使用者執行）         │
├───────────────────────────────────────────────────────┤
│ 第 6 層：Systemd 安全強化                              │
│（NoNewPrivileges、PrivateTmp、ProtectSystem）          │
├───────────────────────────────────────────────────────┤
│ 第 7 層：範圍限定 Sudo 權限                            │
│（僅允許管理 clawdbot 服務與 Tailscale）                │
├───────────────────────────────────────────────────────┤
│ 第 8 層：自動安全更新（unattended-upgrades）           │
└───────────────────────────────────────────────────────┘
```

---

## 第 1 層：UFW 防火牆

### 實作位置

`roles/clawdbot/tasks/firewall-linux.yml`

### 規則設定

```
預設政策：
  - 進入流量（incoming）：DENY（拒絕）
  - 外出流量（outgoing）：ALLOW（允許）
  - 路由流量（routed）：DENY（拒絕）

允許規則：
  - SSH (22/tcp)：ALLOW — 遠端管理
  - Tailscale (41641/udp)：ALLOW — VPN 通訊
```

### 技術細節

```yaml
# 設定預設政策
- name: Set UFW default policies
  community.general.ufw:
    direction: "{{ item.direction }}"
    policy: "{{ item.policy }}"
  loop:
    - { direction: 'incoming', policy: 'deny' }
    - { direction: 'outgoing', policy: 'allow' }
    - { direction: 'routed', policy: 'deny' }
```

### 為什麼需要 UFW？

UFW（Uncomplicated Firewall）是 iptables 的前端，提供簡潔的規則管理。它作為第一道防線，阻擋所有未授權的進入流量。

### 驗證方式

```bash
sudo ufw status verbose
# 預期輸出：
# Status: active
# Default: deny (incoming), allow (outgoing), deny (routed)
# 22/tcp     ALLOW IN    Anywhere
# 41641/udp  ALLOW IN    Anywhere
```

---

## 第 2 層：Fail2ban

### 實作位置

`roles/clawdbot/tasks/firewall-linux.yml`

### 設定

```ini
[DEFAULT]
bantime = 3600    # 封鎖時間：1 小時
findtime = 600    # 觀察時間窗口：10 分鐘
maxretry = 5      # 最大嘗試次數：5 次
backend = systemd  # 使用 systemd journal 讀取日誌

[sshd]
enabled = true
port = ssh
filter = sshd
```

### 運作原理

1. Fail2ban 監控 systemd journal 中的 SSH 登入紀錄
2. 如果同一 IP 在 10 分鐘內失敗登入 5 次
3. 自動將該 IP 加入 iptables 封鎖清單，持續 1 小時
4. 封鎖時間到期後自動解除

### 為什麼需要 Fail2ban？

SSH 是暴露在網際網路上的服務，容易遭受暴力破解攻擊。Fail2ban 自動化了 IP 封鎖流程，大幅降低被攻破的風險。

### 管理指令

```bash
# 檢查狀態
sudo fail2ban-client status sshd

# 查看已封鎖的 IP
sudo fail2ban-client status sshd | grep "Banned IP"

# 手動解除封鎖
sudo fail2ban-client set sshd unbanip <IP_ADDRESS>
```

---

## 第 3 層：DOCKER-USER iptables 鏈

### 問題背景

**Docker 會繞過 UFW**。當你執行 `docker run -p 80:80 nginx` 時，即使 UFW 封鎖了 80 埠，Docker 會直接操作 iptables，讓該埠對外開放。

### 解決方案

使用 `DOCKER-USER` 鏈。這是 Docker 文件中官方建議的方式，Docker 保證會在處理容器流量前先評估此鏈的規則。

### 實作位置

`roles/clawdbot/tasks/firewall-linux.yml` → 寫入 `/etc/ufw/after.rules`

### 規則內容

```
*filter
:DOCKER-USER - [0:0]

# 允許已建立的連線（容器回應外部請求）
-A DOCKER-USER -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT

# 允許 localhost 流量
-A DOCKER-USER -i lo -j ACCEPT

# 封鎖所有透過外部介面進入的 Docker 容器流量
-A DOCKER-USER -i <default_interface> -j DROP

COMMIT
```

### 動態介面偵測

本專案**不硬編碼**網路介面名稱（如 `eth0`），而是動態偵測：

```yaml
- name: Get default network interface
  ansible.builtin.shell:
    cmd: |
      set -o pipefail
      ip route | grep default | awk '{print $5}' | head -n1
    executable: /bin/bash
  register: default_interface
```

這確保在不同的雲端環境（如 `ens3`、`ens160`、`enp0s3` 等）中都能正常運作。

### 為什麼不設定 `iptables: false`？

在 `/etc/docker/daemon.json` 中設定 `"iptables": false` **會完全破壞容器網路**：
- 容器無法存取外部網路
- DOCKER-USER 鏈不會被建立或評估
- 容器間的通訊也會中斷

正確的做法是保持 `"iptables": true`，搭配 DOCKER-USER 鏈來控制流量。

### 驗證方式

```bash
# 查看 DOCKER-USER 鏈規則
sudo iptables -L DOCKER-USER -n -v

# 測試 Docker 隔離
sudo docker run -d -p 80:80 --name test-nginx nginx
curl http://YOUR_SERVER_IP:80     # 應該失敗/超時
curl http://localhost:80           # 應該成功
sudo docker rm -f test-nginx
```

---

## 第 4 層：Localhost 綁定

### 設計原則

所有 Docker 容器的埠口映射都必須使用 `127.0.0.1` 前綴：

```yaml
# ✅ 正確 — 僅本地可存取
ports:
  - "127.0.0.1:3000:3000"

# ❌ 錯誤 — 對所有介面開放
ports:
  - "3000:3000"       # 等同於 0.0.0.0:3000:3000
  - "0.0.0.0:3000:3000"
```

### 為什麼需要 Localhost 綁定？

這是**縱深防禦**的一環。即使 DOCKER-USER 鏈配置錯誤或被修改，localhost 綁定仍能確保服務不會暴露在外部網路上。

### 存取方式

既然服務僅綁定在 localhost，要從外部存取需要：

1. **SSH Tunnel**（推薦用於偵錯）：
   ```bash
   ssh -L 3000:localhost:3000 user@server
   # 在本機瀏覽 http://localhost:3000
   ```

2. **Tailscale VPN**（推薦用於日常使用）：
   ```bash
   # 在伺服器上連線 Tailscale
   sudo tailscale up
   # 從你的裝置透過 Tailscale IP 存取
   ```

---

## 第 5 層：非 Root 容器

### 設計原則

- 建立 `clawdbot` 系統使用者（非特權）
- Clawdbot 進程以 `clawdbot` 使用者身份執行
- 容器內的進程也以非 root 身份執行

### 為什麼不用 Root？

**最小權限原則**。如果容器或應用程式被攻破，攻擊者只擁有 `clawdbot` 使用者的權限，無法：
- 修改系統檔案
- 存取其他使用者的資料
- 安裝惡意軟體到系統目錄
- 修改防火牆規則

---

## 第 6 層：Systemd 安全強化

### 實作位置

`roles/clawdbot/templates/clawdbot-host.service.j2`

### 安全指令

| 指令 | 說明 |
|------|------|
| `NoNewPrivileges=true` | 禁止進程及其子進程獲得新的權限 |
| `PrivateTmp=true` | 使用獨立的 /tmp 目錄，防止 tmp 注入攻擊 |
| `ProtectSystem=strict` | 將 /usr、/boot、/efi 掛載為唯讀 |
| `ProtectHome=read-only` | 將所有使用者的家目錄掛載為唯讀 |
| `ReadWritePaths=~/.clawdbot` | 明確允許寫入設定目錄 |
| `ReadWritePaths=~/.local` | 明確允許寫入本地資料目錄 |

### 效果

即使 Clawdbot 進程被攻破，攻擊者：
- 無法提升權限（NoNewPrivileges）
- 無法存取其他進程的暫存檔（PrivateTmp）
- 無法修改系統二進位檔（ProtectSystem）
- 只能寫入 `~/.clawdbot` 和 `~/.local` 目錄

---

## 第 7 層：範圍限定 Sudo 權限

### 實作位置

`roles/clawdbot/tasks/user.yml` → `/etc/sudoers.d/clawdbot`

### 允許的指令

```
# 服務控制 — 僅限 clawdbot 服務
systemctl start/stop/restart/status/enable/disable clawdbot
systemctl daemon-reload

# Tailscale — 診斷 + 連線/斷線
tailscale status
tailscale up *
tailscale down
tailscale ip/version/ping/whois

# 日誌存取 — 僅限 clawdbot 日誌
journalctl -u clawdbot *
```

### 未授權的操作

clawdbot 使用者**無法**：
- `sudo apt install` — 安裝系統套件
- `sudo ufw` — 修改防火牆規則
- `sudo systemctl restart docker` — 管理 Docker 服務
- `sudo su` — 切換為 root
- 其他任何未明確列出的指令

### 為什麼要限制 Sudo？

如果 clawdbot 擁有完整的 root 權限，一旦應用程式被攻破，攻擊者就能接管整台伺服器。範圍限定的 sudo 將損害控制在最小範圍內。

---

## 第 8 層：自動安全更新

### 實作位置

`roles/clawdbot/tasks/firewall-linux.yml`

### 設定

```
# /etc/apt/apt.conf.d/20auto-upgrades
APT::Periodic::Update-Package-Lists "1";      # 每日更新套件清單
APT::Periodic::Unattended-Upgrade "1";        # 每日執行無人值守升級
APT::Periodic::AutocleanInterval "7";         # 每週清理舊套件

# /etc/apt/apt.conf.d/50unattended-upgrades
# 僅安裝安全更新（不包含功能更新）
Allowed-Origins:
  - ${distro_id}:${distro_codename}-security
  - ${distro_id}ESMApps:${distro_codename}-apps-security
  - ${distro_id}ESM:${distro_codename}-infra-security

# 自動重啟：關閉（需要手動重啟）
Automatic-Reboot "false";
```

### 為什麼需要自動更新？

安全漏洞的修補應該越快越好。自動安全更新能在漏洞公開後盡快套用修補，縮小漏洞利用的時間窗口。

### 注意事項

- 僅安裝安全更新，不會自動安裝功能更新
- 自動重啟已關閉，需要手動監控
- 監控待重啟狀態：`cat /var/run/reboot-required 2>/dev/null`

---

## Docker Daemon 設定分析

### 設定檔

`roles/clawdbot/templates/daemon.json.j2` → `/etc/docker/daemon.json`

| 設定 | 值 | 說明 |
|------|-----|------|
| `iptables` | `true` | **必須**保持 true，否則 DOCKER-USER 鏈不工作 |
| `ip-forward` | `true` | 啟用 IP 轉發，容器才能存取外部網路 |
| `userland-proxy` | `false` | 停用使用者空間代理，改用 iptables NAT |
| `live-restore` | `true` | Docker daemon 重啟時保持容器運行 |
| `ip6tables` | `false` | 停用 IPv6（簡化安全規則） |
| `log-driver` | `json-file` | 使用 JSON 檔案記錄日誌 |
| `log-opts.max-size` | `10m` | 單一日誌檔最大 10MB |
| `log-opts.max-file` | `3` | 最多保留 3 個輪替日誌 |

---

## Tailscale VPN 的安全角色

### 用途

Tailscale 提供安全的遠端存取管道，無需將任何服務埠口暴露在公網上：

```
你的裝置 ──(Tailscale VPN)──→ 伺服器 ──(localhost)──→ Clawdbot:3000
```

### 與防火牆的整合

UFW 僅允許 Tailscale 的 UDP 41641 埠，Tailscale 使用 WireGuard 協定進行加密通訊。

### 安全建議

1. 使用 **臨時金鑰（ephemeral keys）** 而非永久金鑰
2. 在 Tailscale 管理介面設定 **ACL 規則**，限制哪些裝置可以存取
3. 不要將 auth key 提交到 Git 儲存庫

---

## macOS 安全限制

macOS 的安全機制較為有限：

| 功能 | Linux | macOS |
|------|-------|-------|
| UFW 防火牆 | ✅ 完整配置 | ❌（使用 Application Firewall） |
| DOCKER-USER 鏈 | ✅ | ❌（Docker Desktop 自行管理） |
| Fail2ban | ✅ | ❌ 不可用 |
| unattended-upgrades | ✅ | ❌ 不可用 |
| Systemd 安全強化 | ✅ | ❌（無 systemd） |

**建議**：macOS 適合用於開發環境，正式部署應使用 Linux。

---

## 安全檢查清單

安裝完成後，使用以下指令驗證所有安全層：

```bash
# 1. 檢查 UFW 防火牆
sudo ufw status verbose

# 2. 檢查 Fail2ban
sudo fail2ban-client status sshd

# 3. 檢查 DOCKER-USER 鏈
sudo iptables -L DOCKER-USER -n -v

# 4. 外部埠掃描（僅 SSH 應該開啟）
nmap -p- YOUR_SERVER_IP

# 5. 測試 Docker 容器隔離
sudo docker run -d -p 80:80 --name test-nginx nginx
curl http://YOUR_SERVER_IP:80     # 應該超時/失敗
curl http://localhost:80           # 應該成功
sudo docker rm -f test-nginx

# 6. 檢查自動更新
sudo systemctl status unattended-upgrades

# 7. 檢查 Tailscale
sudo tailscale status

# 8. 檢查開放埠口（僅 SSH + localhost 服務）
sudo ss -tlnp
```

---

## 攻擊面分析

### 外部攻擊面

| 暴露服務 | 埠口 | 保護機制 |
|---------|------|----------|
| SSH | 22/tcp | Fail2ban + UFW + 公鑰認證 |
| Tailscale | 41641/udp | WireGuard 加密 + Tailscale ACL |

### 內部攻擊面（若應用被攻破）

| 可能的操作 | 保護機制 |
|-----------|----------|
| 提權 | NoNewPrivileges + 限定 sudo |
| 修改系統 | ProtectSystem=strict |
| 存取其他使用者 | ProtectHome=read-only |
| 存取暫存檔 | PrivateTmp=true |
| 修改防火牆 | sudo 不允許 ufw 指令 |
| 暴露埠口 | DOCKER-USER + localhost 綁定 |

---

## 已知安全限制

1. **`curl | bash` 安裝模式**：本質上存在風險，建議高安全環境先審計程式碼
2. **IPv6 已停用**：如果網路使用 IPv6，需要額外檢視防火牆規則
3. **macOS 支援不完整**：缺少 Fail2ban、unattended-upgrades、systemd 強化等
4. **自動重啟已關閉**：安全更新後可能需要手動重啟
5. **密碼登入未禁用**：本 Playbook 未自動禁用 SSH 密碼登入（建議手動設定）
