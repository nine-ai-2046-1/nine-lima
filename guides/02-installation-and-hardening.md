# 02 - 免 Admin 安裝與超高安全加固實踐

> 本文件記錄點樣喺完全**冇 Admin / Root 權限**嘅 macOS 上，從官方二進制包部署 Lima，並利用自定義模板建立一個具備「最高安全標準」嘅 Ubuntu VM。

---

## 一、 安裝步驟（免 Admin 部署）

### 1. 建立個人專屬目錄
```bash
mkdir -p ~/.local/bin ~/.local/lima-2.3.0 ~/.lima/_templates ~/lima-shared
```

### 2. 下載官方 Release 與校驗 Hash
```bash
cd /tmp
# 下載 Lima v2.3.0 Darwin arm64 官方發行包
curl -fL -o lima.tar.gz https://github.com/lima-vm/lima/releases/download/v2.3.0/lima-2.3.0-Darwin-arm64.tar.gz

# 下載官方 SHA256SUMS 並進行校驗
curl -fL -o SHA256SUMS https://github.com/lima-vm/lima/releases/download/v2.3.0/SHA256SUMS
grep 'lima-2.3.0-Darwin-arm64.tar.gz' SHA256SUMS
shasum -a 256 lima.tar.gz
```
> **安全驗證標準**：計算得出之 Hash 值必須與官方公開之 `b7af444bd8f91d3c307477e562f2cf3bd7595d62b9429185e4fa841563578cfc` 完全一致！

### 3. 解壓並軟連結至 `PATH`
```bash
# 解壓到用戶本地目錄
tar -xzf lima.tar.gz -C ~/.local/lima-2.3.0

# 建立軟連結
ln -sf ~/.local/lima-2.3.0/bin/limactl ~/.local/bin/limactl
ln -sf ~/.local/lima-2.3.0/bin/lima ~/.local/bin/lima

# 驗證安裝
export PATH="$HOME/.local/bin:$PATH"
limactl --version
# 輸出：limactl version 2.3.0
```

---

## 二、 超高加固模板配置 (`hardened.yaml`)

加固配置文件存放於 `~/.lima/_templates/hardened.yaml`：

```yaml
# ==================================================================== #
# HARDENED Lima template — "預設全封鎖，遇到阻礙先逐樣解鎖"
# ==================================================================== #

minimumLimaVersion: 2.0.0

# 1. 基礎映像：官方 Canonical 雲映像，受官方簽名認證
base:
  - template:_images/ubuntu

# 2. 虛擬化驅動：Apple 原生 Virtualization.framework
# 徹底避開 QEMU 模擬器及其相關 CVE
vmType: vz

# 3. 網絡隔離：停用所有外部網橋 (vmnet / socket_vmnet)
# 僅採用用戶模式 NAT，只允許出站請求，無入站路徑，無公網 IP
networks: []

networkOpts:
  # 解鎖（對應指南 05「逐樣解鎖」場景）：允許 Guest 經 NAT 存取 Mac 主機
  # 127.0.0.1 上嘅服務，例如本機 Clash 代理 (127.0.0.1:7890)。
  # 代價：VM 內程式亦可連到 Mac 本地服務。要回復嚴格隔離改返 false 並重啟 VM。
  hostLoopback: true

# 4. 存儲隔離：僅掛載單一指定資料夾，且強制唯讀
mounts:
  - location: "~/lima-shared"
    writable: false

# 5. SSH 加固
ssh:
  localPort: 0             # 自動挑選隨機端口，僅綁定 127.0.0.1
  forwardAgent: false      # 嚴禁將 Mac 的 SSH Agent 洩漏入 VM
  forwardX11: false        # 停用 X11 轉發
  loadDotSSHPubKeys: false # 嚴禁讀取主機 ~/.ssh 目錄

# 6. 容器引擎精簡：關閉 containerd，縮減攻擊面
containerd:
  user: false
  system: false

# 7. 自動安全更新：開機自動 apt upgrade
upgradePackages: true

# 8. 官方工具自動 Provisioning
provision:
  - mode: system
    script: |
      #!/bin/bash
      set -eux
      export DEBIAN_FRONTEND=noninteractive
      apt-get update -q
      # 只從 Ubuntu 官方簽名源安裝 Node.js 與 npm（不使用 curl|bash 第三方腳本）
      apt-get install -y -q nodejs npm ca-certificates
```

### 啟動加固 VM：
```bash
limactl start template:hardened --name=hardened --progress
```

---

## 三、 自動化安全合規驗證 (19 項測試)

喺 Mac 本地建立並執行驗證腳本 `~/lima-shared/verify-security.sh`，實測結果全數通過：

```text
═══ 0. Instance state ═══
  ✅ PASS: instance running (vz / aarch64)

═══ 1. VM cannot WRITE any Mac data ═══
  ✅ PASS: write to ~/lima-shared blocked (Read-only file system)
  ✅ PASS: write to Mac home blocked (path not even visible)
  ✅ PASS: no evil file exists on Mac
  ✅ PASS: no evil file in Mac home

═══ 2. VM cannot READ sensitive Mac files (~/.ssh etc.) ═══
  ✅ PASS: ~/.ssh not visible inside guest
  ✅ PASS: shared folder readable (by design)

═══ 3. VM CAN write inside itself (its own disk) ═══
  ✅ PASS: write+read inside VM works (isolation != broken)

═══ 4. Outbound internet works (fetch from internet) ═══
  ✅ PASS: HTTPS outbound to npm registry works (HTTP 200)
  ✅ PASS: DNS resolution works

═══ 5. Guest cannot reach host's loopback services (hostLoopback=false) ═══
  ✅ PASS: canary listener up on Mac 127.0.0.1
  ✅ PASS: guest blocked from host 127.0.0.1 services

═══ 6. Nothing exposed to LAN/public (binds are 127.0.0.1 only) ═══
  host-side bind address(es): 127.0.0.1:18924
  ✅ PASS: guest service bound to 127.0.0.1 only
  ✅ PASS: LAN IP (10.129.13.155) cannot reach guest service

═══ 7. Toolchain: node / npm / npx ═══
  ✅ PASS: node v24.19.0
  ✅ PASS: npm 11.16.0
  ✅ PASS: npx 11.16.0

═══ 8. SSH access works (local only) ═══
  ✅ PASS: SSH reachable at 127.0.0.1:50151
  ✅ PASS: SSH NOT reachable from LAN IP (connection dropped)

════════════════════════════════
RESULT: 19 passed, 0 failed
════════════════════════════════
```
