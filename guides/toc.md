# Lima 安全隔離開發環境指南 (TOC & 入口文檔)

> 本文檔集記錄咗喺 macOS 環境下，**免 Admin 權限**、以**超高安全加固**模式運行 Ubuntu 虛擬機（Lima），提供 Node.js、npm、npx、Python、Rust、Git、GitHub CLI 等工具鏈嘅完整研究、安裝同使用指南。
>
> 語言：廣東話 (Cantonese)
> 目標讀者：開發者、團隊成員、或接手的 AI Agent。

---

## 🤖 給 AI Agent 的操作指令 (Agent Guidance)

如果你係一個接收到呢個專案嘅 AI Agent，請先閱讀以下定位：
1. **本機環境**：macOS (Apple Silicon `arm64`)，當前用戶**冇 admin/root 權限**。
2. **二進制檔案位置**：`~/.local/bin/limactl` 及 `~/.local/bin/lima`（已加入 PATH）。
3. **加固模板**：位於 `~/.lima/_templates/hardened.yaml`。
4. **現有執行中 VM**：名稱為 `oc1` 及 `oc2`，驅動為 Apple 原生 `vz`。
5. **安全原則**：**「預設全封鎖，遇到阻礙先逐樣解鎖」**。任何時候都唔好隨意打開 Mac 全盤寫入權限或綁定 `0.0.0.0` 網絡。

---

## 📚 文檔目錄 (Table of Contents)

### 1. [01-why-lima-and-security.md](./01-why-lima-and-security.md) — 點解揀 Lima？安全性與架構研究
- Docker Desktop vs Podman vs Lima 綜合比較
- 點解 Lima 可以做到「免 Admin 安裝」
- 歷來安全公告與 CVE 漏洞深入審查（包括 vz 驅動防禦分析）
- 固有安全邊界與風險評估

### 2. [02-installation-and-hardening.md](./02-installation-and-hardening.md) — 免 Admin 安裝與超高安全加固實踐
- 官方 Tarball 下載與官方 SHA256SUMS 校驗
- 免 Root 部署到 `~/.local/`
- 超高安全加固配置 `hardened.yaml` 逐行解析
- 19 項自動化安全測試套裝 (`verify-security.sh`)

### 3. [03-usage-and-workflows.md](./03-usage-and-workflows.md) — 日常使用手冊與工作流
- 點樣進入 VM（3 種方法：互動 Shell、單次指令、SSH）
- 檔案共享與隔離原理（Mac ↔ VM 檔案點樣傳）
- 關鍵概念：`~` 是 VM 內部磁碟，絕非 Mac 磁碟！
- 預裝工具鏈驗證（Node v24、npm/npx、Python 3.14、Rust 1.99、gh、vim）

### 4. [04-multi-vm-and-lifecycle.md](./04-multi-vm-and-lifecycle.md) — 多 VM 管理與資源配置
- 運行多個不同狀態（State）與獨立儲存（Storage）嘅 VM
- 自訂 RAM、CPU 配置策略
- 動態擴展硬碟（Thin Provisioning）機制
- 用完即棄（Disposable / Ephemeral）沙盒建立與徹底銷毀

### 5. [05-troubleshooting-and-unlocks.md](./05-troubleshooting-and-unlocks.md) — 常見問題與「逐樣解鎖」指南
- 遇到做唔到嘅嘢嗰陣，點樣安全地逐步鬆綁封鎖
- 如何增加可寫輸出目錄
- 如何在 VM 內啟用 Docker/containerd
- 常見指令速查表 (Cheat Sheet)

---

## ⚡ 30 秒極速上手 (Quick Start)

```bash
# 1. 確保 PATH 載入
export PATH="$HOME/.local/bin:$PATH"

# 2. 睇下現有 VM
limactl list

# 3. 直接入去 Ubuntu 開工
limactl shell hardened

# 4. 跑安全驗證測試
bash ~/lima-shared/verify-security.sh
```
