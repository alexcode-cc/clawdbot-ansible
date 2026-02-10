# Clawdbot Ansible 安裝器 — 專案技術文件（繁體中文）

> 本目錄包含 Clawdbot Ansible 安裝器的深入技術分析文件，以繁體中文撰寫，供開發者參考。

## 文件索引

| 文件 | 說明 |
|------|------|
| [專案總覽](./overview.md) | 專案背景、功能摘要、目錄結構、技術棧一覽 |
| [架構分析](./architecture.md) | Ansible 角色設計、任務執行順序、Playbook 結構、變數系統 |
| [安全機制分析](./security.md) | 多層防禦架構、UFW/DOCKER-USER/Fail2ban/Systemd 安全強化 |
| [部署與安裝流程](./deployment.md) | 安裝腳本、部署流程、Release/Development 模式、CI/CD |
| [開發指南](./development.md) | 如何修改本專案、新增任務、測試檢查清單、常見錯誤 |

## 快速導覽

如果你是第一次接觸本專案，建議依以下順序閱讀：

1. **[專案總覽](./overview.md)** — 先了解專案的目的與全貌
2. **[架構分析](./architecture.md)** — 理解 Ansible 的執行邏輯與任務編排
3. **[安全機制分析](./security.md)** — 了解多層防禦的設計理念
4. **[部署與安裝流程](./deployment.md)** — 掌握安裝與部署的完整流程
5. **[開發指南](./development.md)** — 開始動手修改與擴展

## 專案資訊

- **專案名稱**：Clawdbot Ansible Installer
- **授權協議**：MIT License
- **目標平台**：Debian 11+ / Ubuntu 20.04+ / macOS 11+
- **Ansible 版本**：2.14+
- **目前版本**：v2.0.0
- **儲存庫**：https://github.com/pasogott/clawdbot-ansible
