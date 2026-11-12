# 01 - 點解揀 Lima？安全性與架構深入研究

> 本文件深入探討點解喺 macOS「冇 Admin 權限、要高安全防禦、又要有完整開發工具」嘅情況下，選擇 Lima 為最佳方案。

---

## 一、 技術方案對比 (Docker vs Podman vs Lima vs OrbStack)

喺 macOS 上想行 Linux / 容器開發環境，主要有以下選手：

| 評估維度 | Docker Desktop | Podman (macOS) | OrbStack | **Lima / Colima** |
|---|---|---|---|---|
| **運行本質** | 虛擬機 + 容器引擎 | 虛擬機 + 容器引擎 | 輕量虛擬機 + 容器 | **原生虛擬機 (Ubuntu)** 或容器 |
| **安裝需否 Admin** | ❌ **必須 Admin**（寫入 `/Applications`、裝 Privileged Helper） | ⚠️ 官方 `.pkg` 需 Admin；免安裝版缺 `gvproxy`/`vfkit` 依賴 | ❌ **必須 Admin** | ✅ **100% 免 Admin**（直接擺用戶目錄 `~/.local`） |
| **商用授權** | 商業付費 (Docker Business) | 開源免費 (Apache 2.0) | 商業付費 | **開源免費 (Apache 2.0 / CNCF)** |
| **預設網絡暴露** | 經常預設綁定 `0.0.0.0`（LAN 可見） | Localhost 導向 | Localhost 導向 | ✅ **完全用戶模式 NAT，只向外發起，零外部暴露** |
| **檔案權限控制** | 預設雙向讀寫 | 雙向讀寫 | 雙向讀寫 | ✅ **精細粒度掛載，預設強制唯讀 (Read-Only)** |
| **後台 Root 守護進程** | 依賴 Root Daemon | Linux 端 Rootless，Mac VM 端仍需控制 | 專用 Helper | ✅ **純普通用戶進程，零 Root 守護** |

**結論：** 對於冇 admin 權限嘅企業或受管 Mac 機，**Lima 係唯一可以免 admin、同時具備最高自定義安全加固能力嘅頂級開源方案**。

---

## 二、 專案背景與可靠性調查

Lima 並非名不見經傳嘅個人小專案，而係現代雲原生基礎設施嘅核心組件：
- **CNCF 孵化專案**：由 Cloud Native Computing Foundation 管轄（同 Kubernetes、containerd 同一級別組織）。
- **廣泛工業應用**：
  - **AWS Finch**：Amazon 官方開源容器工具，底層採用 Lima。
  - **Rancher Desktop**：SUSE 開發嘅企業級 Kubernetes 桌面環境，底層採用 Lima。
  - **Colima**：最熱門嘅 Mac Docker 輕量替代品，本質就係 Lima 嘅封裝。
- **活躍維護**：最新版本 v2.3.0（2026 年底仍持續推出高頻更新與安全公告修補）。
- **供應鏈透明**：提供官方簽名之 SBOM（軟件物料清單）與 SHA256SUMS。

---

## 三、 歷來 CVE 漏洞與安全審查

我哋調取咗 Lima 官方 GitHub Security Advisories 嘅全部公開記錄，總共得 4 條，而且全部已修復並具備清晰安全邊界：

### 1. `limactl shell --sync` TOCTOU 競爭條件 (Nov 2026)
- **嚴重性**：Low (CVSS 2.5)
- **成因**：當用戶使用實驗性嘅 `--sync` 雙向同步指令時，如果審批差異後，Guest 端可能有機會更動檔案。
- **防禦評估**：我哋採用靜態掛載（Mounts），**完全唔用 `--sync`**，因此不受影響。新版 v2.3.0 亦已修補。

### 2. QEMU Option ROM 主機檔案讀取 (Nov 2026)
- **嚴重性**：Moderate (CVSS 6.9)
- **成因**：Guest 端透過 QEMU 模擬器嘅 option ROM 機制嘗試讀取當前工作目錄。
- **防禦評估**：**我哋完全唔用 QEMU**！我哋強制配置 `vmType: vz`（Apple 原生 Virtualization Framework），QEMU 代碼路徑根本唔會被執行。

### 3. Guest Agent 提權 VM Root (CVE-2026-53657, Jun 2026)
- **嚴重性**：High (CVSS 8.2)
- **成因**：QEMU 驅動下，Guest 內部普通用戶可透過 Unix socket 提權成為 VM 內部的 root。
- **官方明確聲明**：*“This vulnerability is not exploitable on `vz` driver, as the guest agent uses vsocks instead of Unix sockets.”*
- **防禦評估**：我哋使用 Apple `vz` 驅動，該漏洞喺技術原理上完全無法被觸發。

### 4. 惡意磁碟映像讀取主機檔案 (2023)
- **嚴重性**：Low
- **成因**：惡意 qcow2 映像可指定 backing file。
- **防禦評估**：已於數年前徹底修復，且我哋強制只使用 Canonical 官方簽名認證之 Ubuntu 雲映像。

---

## 四、 本地加固架構安全模型

```
┌─────────────────────────────────── Mac 主機 (Host) ───────────────────────────────────┐
│                                                                                       │
│   ❌ ~/.ssh, ~/Documents, ~/Desktop 等等全部隱形 (完全不掛載)                            │
│                                                                                       │
│   📁 ~/lima-shared (Mac 實體目錄)                                                      │
│        │                                                                              │
│        │  (強制唯讀 virtiofs 掛載：writable: false)                                   │
│        ▼                                                                              │
│   ┌────────────────────────── Lima VM (Ubuntu) ──────────────────────────┐          │
│   │                                                                      │          │
│   │   📖 /Users/IMTCRS/lima-shared  (唯讀，VM 寫入即報 Read-only fs)       │          │
│   │                                                                      │          │
│   │   ✍️ /home/lima.guest (~)       (可讀可寫，但只限於 VM 專屬虛擬磁碟)    │          │
│   │                                                                      │          │
│   │   🌐 網絡出站 (NAT)：                                                │          │
│   │        - 可以上網 (curl, npm install, cargo)                         │          │
│   │        - ❌ 無法連接 Mac 的 127.0.0.1 服務 (hostLoopback: false)       │          │
│   │        - ❌ 局域網 (LAN) / 公網無法直連 VM                               │          │
│   │                                                                      │          │
│   │   🔌 端口轉發：                                                      │          │
│   │        - 只有本機 127.0.0.1 可連，外部局域網無法探測                     │          │
│   └──────────────────────────────────────────────────────────────────────┘          │
│                                                                                       │
│   💾 實體存儲隔離：~/.lima/hardened/diffdisk (單一映像檔，不污染主機目錄)               │
└───────────────────────────────────────────────────────────────────────────────────────┘
```

**核心安全原則總結**：
1. **VM 就算完全失陷（Compromised）**，黑客亦改唔到 Mac 上嘅任何重要文檔或設定。
2. **VM 碰唔到 Mac 本地服務**，防止惡意代碼探測本機其他開發伺服器（例如本地資料庫、內部 API）。
3. **VM 絕不對 LAN 廣播**，完全處於被動出站隔離狀態。
