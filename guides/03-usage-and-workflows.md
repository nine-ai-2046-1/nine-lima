# 03 - 日常使用手冊與工作流

> 本文件介紹點樣進入 VM、檔案傳遞與隔離邏輯、以及點樣使用預裝好嘅全套開發工具鏈。

---

## 一、 點樣進入 VM

### 1. 互動式進入 Shell（日常開發最常用）
喺 Mac 終端機直接輸入：
```bash
limactl shell oc1
```
你會見到 Prompt 變成：
```bash
lima@lima-hardened:~$
```
- **離開 VM 回到 Mac**：打 `exit` 或者撳鍵盤 `Ctrl + D`。

### 2. 設置快捷別名 (Alias)
為咗唔使每次打長指令，可以喺 Mac 終端執行：
```bash
echo "alias vm='limactl shell oc1'" >> ~/.zshrc
source ~/.zshrc
```
以後喺 Mac Terminal 打一個字 **`vm`** 即可秒速進入！

### 3. 非互動式單次指令執行
如果只係想由 Mac 觸發一次性任務，唔需要入去互動介面：
```bash
limactl shell oc1 -- node -v
limactl shell oc1 -- python3 script.py
limactl shell oc1 -- cargo test
```

---

## 二、 檔案共享與隔離關鍵觀念

### ⚠️ 重要提醒：`~` 到底係邊度？

- **喺 Mac 打 `ls ~`**：係你部 Mac 嘅真實家目錄 `/Users/IMTCRS`。
- **喺 VM 打 `ls ~`**：係 VM 內部虛擬硬碟嘅 `/home/lima.guest`。

**兩者完全無關！**
即使喺 VM 入面執行 `rm -rf ~/*` 或者產生大量垃圾檔案，你 Mac 本身嘅文件、桌面、下載目錄**完全 0 影響**。

---

## 三、 點樣傳檔案入 VM 開工？

因為我哋實施咗「最高安全唯讀策略」，檔案流動建議遵循以下模式：

### 模式 A：透過共享目錄 `~/lima-shared`（推薦）
1. **喺 Mac 端**：將代碼或資源放入 `~/lima-shared`：
   ```bash
   cp -r ~/my-project ~/lima-shared/
   ```
2. **入去 VM 端**：檔案會以**唯讀方式**出現在 `/Users/IMTCRS/lima-shared/`。
3. **複製至工作區開工**：
   ```bash
   cp -r /Users/IMTCRS/lima-shared/my-project ~/my-project
   cd ~/my-project
   npm install      # 此時喺 VM 專屬磁碟內寫入，不受限制且不會污染 Mac！
   ```

### 模式 B：使用 `limactl copy` 直接推入（類似 `docker cp`）
由 Mac 終端機直接將檔案傳送到 VM 嘅內部目錄：
```bash
# 複製單一檔案
limactl copy ./server.js hardened:~/

# 遞歸複製資料夾
limactl copy -r ./my-node-app hardened:~/
```

### 模式 C：直接於 VM 內透過 Git 拉取（最乾淨）
VM 擁有完整出站網絡，可直接於內部進行 Git 操作：
```bash
limactl shell oc1
git clone https://github.com/user/repo.git
```

---

## 四、 預裝工具鏈驗證清單

當前 VM `hardened` 已預裝並配置好以下工具（全數加入登入 Shell 之 PATH）：

| 工具 | 執行指令 | 當前已驗證版本 | 安裝來源 |
|---|---|---|---|
| **Node.js** | `node -v` | `v24.19.0` | Ubuntu 官方簽名倉庫 |
| **npm** | `npm -v` | `11.16.0` | 官方倉庫隨附 |
| **npx** | `npx -v` | `11.16.0` | 官方倉庫隨附 |
| **Python 3** | `python3 --version` | `3.14.8` | Ubuntu 官方倉庫 |
| **Pip 3** | `pip3 --version` | `26.1.2` | Ubuntu 官方倉庫 |
| **Git** | `git --version` | `2.55.0` | Ubuntu 官方倉庫 |
| **GitHub CLI**| `gh --version` | `2.103.3` | GitHub 官方 GPG 倉庫 |
| **Rust 語言** | `rustc --version`| `1.99.1` | 官方 Rustup 認證源 |
| **Cargo 工具**| `cargo --version`| `1.99.1` | 官方 Rustup 認證源 |
| **Vim 編輯器**| `vim --version` | `9.2` | Ubuntu 官方倉庫 |
