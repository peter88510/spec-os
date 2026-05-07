# MedPACS Development Specification

> 版本：1.0 | 建立日期：2026-05
> 核心哲學：「向外擴充，拒絕破壞 (Extend Outward, Never Break)」

---

## 1️⃣ 核心開發原則 (Immutable Principles)

### 禁止行為 ❌
- 重構既有系統或模組
- 變更既有 API 的 request / response 行為
- 刪除任何既有功能、欄位、或端點
- 重新設計資料庫 Schema 或儲存架構

### 允許行為 ✅
- 向外擴充 (Incremental Extension)：新增 function、class、endpoint
- 在不影響舊邏輯的前提下，擴展既有模組
- 新增欄位時須保持向後相容（nullable 或提供 default）

---

## 2️⃣ 新功能設計 SOP (New Feature Workflow)

在輸出任何代碼前，必須依序完成以下判斷：

### Step 1 — 優先復用 (Reuse First)

| 功能類型 | 優先放置位置 | 操作方式 |
|---|---|---|
| 資料庫邏輯 | `db_service.py` | add function |
| DB 連線/Session | `db.py` | add connection logic |
| 儲存介面/抽象層 | `storage.py` | extend interface |
| 儲存相關 | `storage_backend.py` | extend class |
| AI 邏輯 | `ai/service.py` | stub or logic |
| 驗證規則 | `validation/` | new rule file |
| API 入口 | `main.py` | only endpoints |
| ORM 模型 | `models.py` | add column / relationship |
| 系統設定 | `config.py` | add env variable |
| Service 層測試 | `test_dicom_service.py` | add test function（同類型） |
| API 層測試 | `test_query_api.py` | add test function（同類型） |
| 新類型測試 | `test_<feature_name>.py` | 新增獨立測試檔案 |

### Step 2 — 檔案最小化
- 優先擴展既有 Class 或 Function，不隨意建立新檔案。

### Step 3 — 獨立模組化（例外條件）
僅當功能符合以下條件，才允許新增 `.py` 檔案：
- 功能具備**高度獨立性**（如新 AI 引擎、獨立 Validator）
- 與既有模組**無耦合**或耦合極低
- 單一模組超過合理行數上限（建議 500 行）

---

## 3️⃣ 檔案輸出標準 (Output Formats)

### 📂 情況 A：新增檔案

- 輸出「完整檔案內容」，確保可直接 Copy-Paste 使用
- 頂部標示：`filename: path/to/file.py`

### 📂 情況 B：修改既有檔案

**嚴禁全文輸出**，僅允許輸出以下內容：

```
# [檔案名稱] 修改位置：<Class 名稱 / Function 名稱 / 行號區間>

# --- import 補丁（如有）---
from xxx import yyy

# --- 新增 Function / Class 區塊 ---
def new_function(...):
    ...
```

---

## 4️⃣ 文檔與測試規範

### 文檔規範
- **Markdown 修改**：僅允許新增 Section 或 Patch 區塊，禁止 Rewrite 全文
- **README.md**：僅允許新增 API 說明區塊
- **QUICKSTART.md**：僅允許新增測試指令（Manual test commands）
- **IMPLEMENTATION.md**：新增架構說明時，以新 Section 附加，不改動既有內容

### 測試規範
- 沿用既有測試風格（如 `FakeRow` / `Mock`）
- 嚴禁引入新的測試框架
- 新功能測試須附於對應既有測試檔案末尾

---

## 5️⃣ 回應格式要求 (Final Response Structure)

每次交付新功能時，必須按此順序呈現：

1. **【Package 規劃說明】**：告知新增功能將放置於哪個既有/新模組，以及理由。
2. **【代碼/文檔實作】**：
   - (新檔案) 完整代碼
   - (修改檔) 局部 Patch 區塊
3. **【測試與文件補丁】**：提供對應的測試 Sample 與 README 增補內容。
