# 04 - 多 VM 管理與資源配置

> 本文件說明點樣充分利用 Mac 嘅 RAM 同 CPU，開闢多個具備「獨立狀態（State）」同「獨立儲存（Storage）」嘅 VM，並介紹用完即棄（Disposable）測試流程。

---

## 一、 多 VM 的並發運行機制

Lima 支援同時開啟多個虛擬機，每個 VM 擁有：
1. **獨立名稱**（如 `oc1`、`oc2`、`sandbox`、`worker`）。
2. **獨立虛擬硬碟**（存放在 `~/.lima/<VM名稱>/diffdisk`）。
3. **獨立網絡監聽端口**（自動分配不同之本地 SSH 及轉發端口）。
4. **獨立記憶體分配與進程空間**。

---

## 二、 建立全新獨立 VM

### 1. 使用現有加固配置建立第二個 VM
只需指定一個新名稱：
```bash
limactl start template:hardened --name=sandbox
```
該 VM 會完全隔離，喺 `sandbox` 裝嘅任何嘢絕唔會影響到 `hardened`。

### 2. 自訂 CPU 與記憶體規格
可根據任務負載，喺啟動時動態指定資源參數：
```bash
# 輕量級輔助 VM (2 核心，2GB RAM)
limactl start template:hardened --name=worker --cpus=2 --memory=2GiB

# 重量級編譯 VM (4 核心，8GB RAM)
limactl start template:hardened --name=heavy-builder --cpus=4 --memory=8GiB
```

---

## 三、 多 VM 管理速查

### 1. 列出所有虛擬機與狀態
```bash
limactl list
```
輸出範例：
```text
NAME        STATUS     SSH                VMTYPE    ARCH       CPUS    MEMORY    DISK
hardened    Running    127.0.0.1:50151    vz        aarch64    4       4GiB      100GiB
sandbox     Running    127.0.0.1:51234    vz        aarch64    2       2GiB      100GiB
worker      Stopped    127.0.0.1:0        vz        aarch64    2       2GiB      100GiB
```

### 2. 進入特定 VM
```bash
limactl shell oc1     # 進入 oc1
limactl shell oc2     # 進入 oc2
```

### 3. 個別停止與啟動（釋放 RAM）
當某個專案暫時唔用，停用該 VM 可即時將 RAM 完整釋放返畀 macOS：
```bash
limactl stop sandbox       # 停止並釋放其記憶體
limactl start sandbox      # 再次喚醒
```

---

## 四、 用完即棄測試沙盒（Disposable Sandbox）

如果你需要進行高風險實驗（例如測試不知名開源代碼、極端破壞性測試）：

```bash
# 1. 開啟臨時沙盒
limactl start template:hardened --name=disposable-lab

# 2. 進入內部隨意測試
limactl shell disposable-lab

# 3. 測試完畢，徹底銷毀
limactl stop disposable-lab
limactl delete disposable-lab
```
> **安全保證**：執行 `limactl delete` 後，該 VM 嘅磁碟鏡像、安裝內容及歷史記錄將會從 Mac 本地磁碟中**百分之百徹底抹除**，零殘留。

---

## 五、 磁碟空間機制 (Thin Provisioning)

雖然每個 VM 預設宣告磁碟上限為 100GiB，但底層採用**精簡配置（Thin Provisioning / Sparse File）**：
- 如果 VM 內部只佔用 3GB，它在 Mac 實體磁碟上就**只佔 3GB**。
- 不會因為開了 3 個 VM 就直接佔用 300GB 磁碟空間。
