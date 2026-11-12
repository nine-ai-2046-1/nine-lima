# 05 - 常見問題與「逐樣解鎖」指南

> 貫徹方針：**「預設全封鎖，遇到阻礙先逐樣解鎖」**。
> 本文件記錄日常開發如果遇到限制，點樣用最安全嘅方式一步一步解開。

---

## 🌟 核心場景：雙 VM 互通與 Web/WebSocket 服務 (已實測驗證 ✅)

### 需求目標：
- VM 1（`oc1`）運行 Server（監聽端口 **7001**）。
- VM 2（`oc2`）運行 Server/Client（監聽端口 **7002**）。
- **要求**：
  1. Mac 主機可以連到 7001 同 7002。
  2. VM 2 可以連到 VM 1 的 7001，VM 1 可以連到 VM 2 的 7002。
  3. **極高安全防禦**：同一 WiFi 局域網（Intranet）及公網（Public Internet）**絕對連唔到**。

### 設定方式（已全自動部署）：
在兩個 VM 的 `lima.yaml` 中配置：
```yaml
networkOpts:
  hostLoopback: true   # 允許透過 host.lima.internal 在本機回環內互通
```

### 連線位址一覽表：

| 連線方向 | 請求目標位址 | 協議 | 說明 |
|---|---|---|---|
| **Mac ➔ VM 1** | `http://127.0.0.1:7001` 或 `ws://127.0.0.1:7001` | HTTP / WebSocket | 直接連線，零 Proxy |
| **Mac ➔ VM 2** | `http://127.0.0.1:7002` 或 `ws://127.0.0.1:7002` | HTTP / WebSocket | 直接連線，零 Proxy |
| **VM 2 ➔ VM 1**| `http://host.lima.internal:7001` 或 `ws://host.lima.internal:7001` | HTTP / WebSocket | 經由本機內部通道直達 |
| **VM 1 ➔ VM 2**| `http://host.lima.internal:7002` 或 `ws://host.lima.internal:7002` | HTTP / WebSocket | 經由本機內部通道直達 |
| **外部 WiFi / LAN** | ❌ `http://<你的LAN_IP>:7001` 或 `7002` | 全部被拒絕 | 物理級防禦，綁定 `127.0.0.1` |

---

## 解鎖場景一：需要一個「可寫入」的輸出目錄

### 狀況：
你在 VM 內編譯出產物（如 build 出來的 dist、產生的報告），希望直接放到 Mac 上查看，但 `~/lima-shared` 是唯讀的。

### 安全解決方案（受控可寫目錄）：
**不要直接把整個 Mac 家目錄設為可寫！** 正確做法是只開闢一個專用子目錄：

1. **Mac 端建立輸出目錄**：
   ```bash
   mkdir -p ~/lima-shared/out
   ```
2. **在配置中追加獨立可寫掛載**：
   在自定義 YAML 或現有 VM 實例配置（`~/.lima/<名稱>/lima.yaml`）中加入：
   ```yaml
   mounts:
     - location: "~/lima-shared"
       writable: false
     - location: "~/lima-shared/out"
       writable: true
   ```
3. 這樣既保護了其他代碼不被篡改，又允許將結果輸出至 `out` 目錄。

---

## 解鎖場景二：需要在 VM 內部運行 Docker 容器

### 狀況：
你想在 VM 內部運行 `docker run`、`docker compose` 或測試容器鏡像。

### 安全解決方案（用戶態 Rootless 容器）：
Lima 原生支援在 Guest 內部啟用 containerd / Docker：

1. 在模板 `hardened.yaml` 中將 containerd 改為用戶模式：
   ```yaml
   containerd:
     user: true       # 啟用 rootless containerd / nerdctl
     system: false
   ```
2. 重啟或重建 VM 後，即可在 VM 內部直接運行：
   ```bash
   lima nerdctl run --rm -p 8080:8080 nginx
   ```
   *注意：這是在 VM 內部跑容器，不會在 Mac 主機上留下任何特權 Daemon。*

---

## 解鎖場景三：需要掛載其他特定代碼目錄

### 狀況：
你有些代碼放在 `/Users/IMTCRS/work/project-a`，不想複製到 `~/lima-shared`。

### 安全解決方案：
**單獨以唯讀方式追加該目錄**：
```yaml
mounts:
  - location: "~/lima-shared"
    writable: false
  - location: "/Users/IMTCRS/work/project-a"
    writable: false   # 堅持唯讀原則
```

---

## 🛠️ 常見問題排查 (FAQ)

### Q1: 終端機提示 `limactl: command not found`？
**解答**：因為沒有 admin 權限，程序安裝在個人目錄。請確保終端載入了路徑：
```bash
export PATH="$HOME/.local/bin:$PATH"
```
（已寫入 `~/.zshrc`，通常重新打開終端即可自動生效）。

### Q2: 關閉 Terminal 後，VM 會不會自己死咗？
**解答**：唔會。Lima 的 VM 由後台 `hostagent` 進程維護，關閉 Terminal 不會中斷 VM。只要 Mac 沒有關機或睡眠，VM 會一直在後台運行。

### Q3: 怎樣完全關閉 VM 以省電？
```bash
limactl stop oc1
limactl stop oc2
```

### Q4: 怎樣驗證安全防禦是否依然生效？
任何時候都可以執行測試套件進行全方位檢查：
```bash
bash ~/lima-shared/verify-security.sh
```
若全部通過，即表示安全邊界固若金湯。
