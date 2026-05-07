# TESTSPEC.md — FastAPI + PostgreSQL + DICOM 後端測試規範

> **版本**：1.1  
> **狀態**：生產規範（Production Standard）  
> **適用範圍**：本專案所有測試檔案，包含現有測試與未來新增測試  
> **維護原則**：擴充優先，不得無故重新設計既有架構  
> **變更紀錄**：v1.1 — 反映 TestClient fixture 化、test_validators.py 拆分、monkeypatch 引入、儲存路徑修正等實際改動

---

## 目錄

1. [測試哲學](#1-測試哲學)
2. [Fake Data 慣例](#2-fake-data-慣例)
3. [TestClient 慣例](#3-testclient-慣例)
4. [資料庫測試策略](#4-資料庫測試策略)
5. [API 測試慣例](#5-api-測試慣例)
6. [Mocking 策略](#6-mocking-策略)
7. [測試檔案組織結構](#7-測試檔案組織結構)
8. [未來擴充規則](#8-未來擴充規則)
9. [AI 貢獻者規則（Claude / Copilot）](#9-ai-貢獻者規則)

---

## 1. 測試哲學

### 1.1 測試範疇

本專案採用**三層測試架構**，各層職責明確，不得混用：

| 層級 | 定義 | 對應檔案範例 |
|------|------|------------|
| **單元測試（Unit）** | 測試單一函式或 class，完全隔離外部依賴 | `test_validators.py` |
| **整合測試（Integration）** | 使用真實 SQLite in-memory DB，測試 DB 寫入/讀取邏輯 | `test_dicom_service.py`（DB 操作段落） |
| **API 測試（API / End-to-End）** | 使用 FastAPI `TestClient`，mock 掉 DB service layer，測試 HTTP 行為 | `test_query_api.py` |

### 1.2 架構假設

- 框架：FastAPI
- ORM：SQLAlchemy（Session 管理透過 FastAPI dependency injection）
- 測試 DB：SQLite（`sqlite:///./test.db`）用於整合測試
- API mock 層：`unittest.mock.patch` 直接 patch `main` 模組中的 service 函式
- 不使用 `pytest-asyncio`（目前 endpoint 為同步）；若未來引入 async，需另訂規則

### 1.3 核心原則

- **隔離性**：API 測試不應依賴真實 DB；整合測試不應依賴外部儲存
- **速度優先**：單元測試與 API 測試必須毫秒級完成
- **可重複性**：每次測試結果必須與環境無關（deterministic）

---

## 2. Fake Data 慣例

### 2.1 `_FakeRow` 基礎類別

#### 定義

```python
class _FakeRow:
    pass
```

`_FakeRow` 是一個**最小化的假 ORM 物件**，用來模擬 SQLAlchemy model instance 的回傳值。它刻意保持空白，透過 `__dict__` 動態賦值以維持彈性。

#### 使用時機

- 僅用於 **API 測試**（`test_*_api.py`）中，作為 mock 函式的回傳值
- **不得**用於整合測試（整合測試使用真實 model instance）
- **不得**用於需要觸發 SQLAlchemy relationship 的情境

#### 命名規則

| 規則 | 說明 |
|------|------|
| 以底線開頭 `_FakeRow` | 表示為測試內部工具，非公開 API |
| 每個測試檔案各自定義 | 目前各檔案重複定義；**改善計畫見 2.5 節** |

---

### 2.2 `make_*()` 工廠函式

#### 現有定義（來自 `conftest.py`）

```python
def make_study(id=1, study_instance_uid="1.2.3"):
    obj = _FakeRow()
    obj.__dict__ = {"id": id, "study_instance_uid": study_instance_uid}
    return obj

def make_series(id=10, study_id=1, series_instance_uid="1.2.3.4"):
    obj = _FakeRow()
    obj.__dict__ = {"id": id, "study_id": study_id, "series_instance_uid": series_instance_uid}
    return obj

def make_instance(id=100, series_id=10, sop_instance_uid="1.2.3.4.5"):
    obj = _FakeRow()
    obj.__dict__ = {"id": id, "series_id": series_id, "sop_instance_uid": sop_instance_uid}
    return obj
```

#### 命名規則

| 規則 | 範例 |
|------|------|
| 前綴固定為 `make_` | `make_study`, `make_series`, `make_instance` |
| 後綴對應 ORM model 名稱（小寫） | `Study` → `make_study()` |
| 所有參數必須有預設值 | 確保最簡單的呼叫為零參數 |

#### 必填欄位規則

每個 `make_*()` 函式必須至少包含以下欄位：

| 物件 | 必填欄位 |
|------|---------|
| `make_study()` | `id`, `study_instance_uid` |
| `make_series()` | `id`, `study_id`, `series_instance_uid` |
| `make_instance()` | `id`, `series_id`, `sop_instance_uid` |
| `make_patient()`（未來） | `id`, `patient_id` |

#### 擴充規則

- 新增欄位時，**保持現有參數順序**，新欄位加在最後
- 新增欄位**必須有預設值**，不得破壞現有測試的呼叫方式
- 若 model 有新欄位但測試不需要，可不加入 `make_*()` 函式

#### `make_mock_ds()` — pydicom Dataset 工廠

```python
def make_mock_ds(patient_id="P001", study_uid="1.2.3", modality="US"):
    ds = MagicMock()
    ds.PatientID = patient_id
    ds.StudyInstanceUID = study_uid
    ds.Modality = modality
    return ds
```

- 定義於 `conftest.py`，專屬 DICOM validation 測試
- 命名規則：`make_mock_<物件名稱>()`，明確標示為 MagicMock 物件

---

### 2.3 預設值規範

所有 `make_*()` 函式的預設 UID 值必須遵循：

- **Study UID**：`"1.2.3"`
- **Series UID**：`"1.2.3.4"`
- **Instance UID**：`"1.2.3.4.5"`
- **Patient ID**：`"P001"`
- **id（整數）**：Study=`1`，Series=`10`，Instance=`100`（遞增十倍，易於辨識）

---

### 2.4 已識別問題與改善建議(已改善)

#### ⚠️ 問題：`_FakeRow` 與 `make_*()` 在多個測試檔案中重複定義

**現況**：`_FakeRow` 與工廠函式目前定義於 `test_query_api.py`。若未來新增 `test_upload_api.py`、`test_ai_api.py`，每個檔案都複製一份，將造成維護困難。

**風險**：
- 若 model 新增欄位，需逐一修改所有檔案
- 不同測試檔案的預設值可能不一致，產生隱性 bug

**改善方案**：建立 `tests/conftest.py`（詳見 2.5 節）

---

### 2.5 ✅ 改善後的標準結構：`tests/conftest.py`

建立 `tests/conftest.py`，集中管理共用工具：

```python
# tests/conftest.py
from unittest.mock import MagicMock

# ── Fake ORM Row ──────────────────────────────────────────
class _FakeRow:
    """測試用最小化 ORM 物件，僅用於 API 測試的 mock 回傳值。"""
    pass


# ── ORM 工廠函式 ──────────────────────────────────────────
def make_study(id=1, study_instance_uid="1.2.3"):
    obj = _FakeRow()
    obj.__dict__ = {"id": id, "study_instance_uid": study_instance_uid}
    return obj


def make_series(id=10, study_id=1, series_instance_uid="1.2.3.4"):
    obj = _FakeRow()
    obj.__dict__ = {"id": id, "study_id": study_id, "series_instance_uid": series_instance_uid}
    return obj


def make_instance(id=100, series_id=10, sop_instance_uid="1.2.3.4.5"):
    obj = _FakeRow()
    obj.__dict__ = {"id": id, "series_id": series_id, "sop_instance_uid": sop_instance_uid}
    return obj


# ── pydicom Dataset 工廠 ──────────────────────────────────
def make_mock_ds(patient_id="P001", study_uid="1.2.3", modality="US"):
    ds = MagicMock()
    ds.PatientID = patient_id
    ds.StudyInstanceUID = study_uid
    ds.Modality = modality
    return ds
```

> **執行時間**：此改善為**低風險、低成本**，建議在下一個測試檔案新增前完成。

---

## 3. TestClient 慣例

### 3.1 現行實例化方式（Fixture 模式）

所有 TestClient 必須透過 `conftest.py` 中定義的 fixture 注入，不得在模組層級建立。

**`api_client` — 純 HTTP 測試（無 DB override）**：
```python
# conftest.py
@pytest.fixture
def api_client():
    with TestClient(app) as client:
        yield client
```
適用於：`test_query_api.py`、`test_validators.py` 中需要 HTTP 的測試。

**`db_client` — 整合測試（含 DB override）**：
```python
# conftest.py
@pytest.fixture
def db_client():
    app.dependency_overrides[get_db] = override_get_db
    with TestClient(app) as client:
        yield client
    app.dependency_overrides.clear()
```
適用於：`test_dicom_service.py` 中需要真實 DB 的 HTTP 測試。

### 3.2 TestClient 使用規則

| 測試類型 | Fixture | 說明 |
|---------|---------|------|
| **API 測試**（mock DB） | `api_client` | 無 DB override，適合純 HTTP 行為測試 |
| **整合測試**（真實 SQLite） | `db_client` | 含 `get_db` override，測試真實 DB 寫入邏輯 |

**禁止在模組層級建立 TestClient**：
```python
# ❌ 禁止
client = TestClient(app)
app.dependency_overrides[get_db] = override_get_db

# ✅ 正確：透過 fixture 參數注入
def test_health_check(db_client):
    response = db_client.get("/health")
```

### 3.3 ✅ 已解決：模組層級 `client` 的副作用問題

**原問題**：兩個測試檔案在模組層級建立 `client = TestClient(app)`，`dependency_overrides` 永久覆寫全域 app 狀態，造成測試間互相污染。

**解決方案**：已遷移至 fixture 模式（見 §3.1），`dependency_overrides` 在 fixture yield 後自動 `clear()`，每個測試擁有獨立狀態。

### 3.4 Import 順序規範

```python
# 標準 import 順序（所有測試檔案遵循）
import pytest
from fastapi.testclient import TestClient
from unittest.mock import patch, MagicMock
from main import app
# 視需要引入：from tests.conftest import make_study, make_series, make_instance
```

---

## 4. 資料庫測試策略

### 4.1 兩種測試模式

#### 模式 A：整合測試（使用 SQLite）

- **適用情境**：測試 `DatabaseService` 的實際 CRUD 邏輯
- **DB 引擎**：`sqlite:///./test.db`（檔案型，非 in-memory）
- **隔離機制**：`autouse=True` 的 fixture，每個測試前 `create_all`，測試後 `drop_all`

```python
@pytest.fixture(autouse=True)
def setup_teardown():
    Base.metadata.create_all(bind=engine)
    yield
    Base.metadata.drop_all(bind=engine)
```

- **Session 管理**：手動建立 `db = TestingSessionLocal()`，測試結束後 `db.close()`
- **不使用 rollback**：目前以 `drop_all` 取代，確保完全清潔

#### 模式 B：API 測試（mock DB service）

- **適用情境**：測試 HTTP endpoint 的回應格式、狀態碼、參數傳遞
- **DB 層**：完全透過 `@patch("main.<function_name>")` 繞過
- **不建立任何 DB 連線**

### 4.2 選擇指南

```
需要測試 DB schema 是否正確？        → 模式 A（整合測試）
需要測試 upsert / 查詢邏輯？         → 模式 A（整合測試）
需要測試 API 回傳格式？              → 模式 B（API 測試）
需要測試 404 / 422 行為？            → 模式 B（API 測試）
需要測試 DICOM 解析邏輯？            → 單元測試（make_mock_ds）
```

### 4.3 ⚠️ 已識別問題：SQLite 與 PostgreSQL 的差異

**現況**：整合測試使用 SQLite，但生產環境使用 PostgreSQL。

**潛在風險**：
- PostgreSQL 特有功能（如 `JSONB`、array 欄位、`ON CONFLICT DO UPDATE`）在 SQLite 中行為不同或不支援
- PostgreSQL 的 constraint 行為（如 `UNIQUE`、`FOREIGN KEY` 的 deferred）與 SQLite 有差異

**現階段立場**：維持 SQLite，不強制要求切換。

**未來擴充規則**（當以下任一情況發生時，必須討論是否引入 `pytest-postgresql` 或 Docker PostgreSQL）：
- 模型中引入 PostgreSQL 專有欄位類型
- `upsert` 邏輯使用 PostgreSQL 特有語法
- 測試中出現 SQLite/PostgreSQL 行為不一致的 bug

### 4.4 Session 使用規則

- 整合測試中，Session **不得跨測試函式共享**
- 每個測試透過 `db` fixture 注入獨立 Session，fixture 結束後自動 `close()`

```python
# conftest.py
@pytest.fixture
def db():
    db = TestingSessionLocal()
    yield db
    db.close()

# 測試函式
def test_database_upsert_patient(db):
    patient = DatabaseService.upsert_patient(db, "PATIENT_001")
    assert patient.patient_id == "PATIENT_001"
```

- **不得**在測試函式內手動建立 `db = TestingSessionLocal()` 或呼叫 `db.close()`

---

## 5. API 測試慣例

### 5.1 Class 組織規則

每個 endpoint 對應一個 `Test<動詞><資源>` class：

```python
class TestListStudies:      # GET /studies
class TestGetSeries:        # GET /series/{id}
class TestGetInstance:      # GET /instances/{id}
class TestGetInstanceFile:  # GET /instances/{id}/file
class TestAiSegment:        # POST /ai/segment/{id}
```

**命名規則**：
- `TestList<資源複數>` — 列表型 endpoint
- `TestGet<資源單數>` — 單一資源查詢
- `TestCreate<資源>` — 建立資源
- `TestUpdate<資源>` — 更新資源
- `TestDelete<資源>` — 刪除資源
- `Test<動詞><資源>` — 動作型 endpoint（如 `TestAiSegment`）

### 5.2 必要測試案例（每個 endpoint 必須涵蓋）

#### 成功案例（Success Case）

```python
def test_returns_<資源>_when_found(self, mock_get):
    mock_get.return_value = make_<資源>(id=<預期id>)
    response = client.get("/<endpoint>/<id>")
    assert response.status_code == 200
    assert response.json()["id"] == <預期id>
```

#### 404 案例

```python
def test_returns_404_when_not_found(self, mock_get):
    mock_get.return_value = None
    response = client.get("/<endpoint>/999")
    assert response.status_code == 404
    assert "not found" in response.json()["detail"].lower()
```

> **規範**：404 回應的 `detail` 欄位**必須**包含 `"not found"`（case-insensitive）。

#### ID 傳遞驗證（ID Propagation Test）

```python
def test_id_passed_correctly(self, mock_get):
    mock_get.return_value = make_<資源>(id=<特定id>)
    client.get("/<endpoint>/<特定id>")
    call_args = mock_get.call_args[0]
    assert call_args[1] == <特定id>
```

> **目的**：確認 FastAPI path parameter 正確傳入 service 函式，防止硬編碼或型別錯誤。

#### 驗證失敗案例（Validation Failure Case）

```python
def test_returns_400_when_<欄位>_invalid(self, ...):
    response = client.post("/<endpoint>", json={"field": "<invalid_value>"})
    assert response.status_code == 400
    assert "error" in response.json()
```

#### 空列表案例（List Endpoints）

```python
def test_returns_empty_list(self, mock_get):
    mock_get.return_value = []
    response = client.get("/<endpoint>")
    assert response.status_code == 200
    assert response.json() == {"<key>": []}
```

### 5.3 測試函式命名規則

| 情境 | 命名格式 |
|------|---------|
| 成功回傳 | `test_returns_<回傳內容>_when_found` |
| 找不到資源 | `test_returns_404_when_not_found` |
| 驗證失敗 | `test_returns_400_when_<欄位>_<原因>` |
| ID 傳遞 | `test_id_passed_correctly` |
| 空列表 | `test_returns_empty_list` |
| 檔案不存在 | `test_returns_404_when_file_missing_on_disk` |
| 狀態確認 | `test_returns_<狀態>_status_when_<條件>` |

### 5.4 Response 斷言規則

- **永遠先斷言 `status_code`**，再斷言 body
- 字串比對使用 `.lower()` 正規化：`assert "not found" in response.json()["detail"].lower()`
- 不斷言完整 body（除非測試明確要求），只斷言**關鍵欄位**
- 列表型回應斷言 `len()` 與特定元素，不做 full-object 比對

---

## 6. Mocking 策略

### 6.1 使用 `@patch` 的時機

| 情境 | 做法 |
|------|------|
| 測試 API endpoint 的 HTTP 行為 | `@patch("main.<service_function>")` |
| 測試 DICOM 解析流程 | `patch("main.pydicom.dcmread", return_value=mock_ds)` |
| 測試檔案系統存在性 | `@patch("os.path.exists")` |
| 測試 FileResponse 行為 | `patch("main.FileResponse")` |

**patch 目標規則**：永遠 patch **被使用的地方**（`main.<name>`），而非**被定義的地方**（`db_service.<name>`）。

```python
# ✅ 正確
@patch("main.get_all_studies")

# ❌ 錯誤
@patch("db_service.get_all_studies")
```

### 6.2 使用 `_FakeRow` / `make_*()` 的時機

- 當 mock 函式需要回傳一個**看起來像 ORM 物件**的東西時
- 回傳值需要有屬性存取（`obj.id`, `obj.study_instance_uid`）時
- **不需要**觸發 SQLAlchemy 的 lazy loading 或 relationship 時

### 6.3 使用 `MagicMock` 的時機

- 需要模擬有方法呼叫的物件（如 `ds.PatientID`）
- 不需要精確控制屬性結構，只需要物件「存在」時
- 模擬 pydicom `FileDataset` 這類外部 library 物件

### 6.4 **不應 mock** 的情境

| 情境 | 原因 |
|------|------|
| 整合測試中的 `DatabaseService` 方法 | 測試的核心目的就是驗證這些方法 |
| SQLAlchemy Session 的基本操作 | 應使用真實 SQLite session |
| FastAPI 本身的 routing 邏輯 | TestClient 已涵蓋 |
| `ValidationError` 的拋出 | 應讓真實的 validator 執行 |

### 6.5 `monkeypatch` vs `@patch`

- 本專案**統一使用 `@patch`**（來自 `unittest.mock`）patch 函式與物件
- `pytest monkeypatch` fixture 用於**環境變數與物件屬性覆寫**，限以下情境：
  - `monkeypatch.setenv("STORAGE_PATH", ...)` — 覆寫環境變數
  - `monkeypatch.setattr(obj, "attr", value)` — 覆寫已實例化物件的屬性
- 兩者可同時使用，職責不重疊：

```python
def test_upload_stores_file_locally(tmp_path, monkeypatch):
    monkeypatch.setenv("STORAGE_PATH", str(tmp_path))       # 環境變數
    monkeypatch.setattr(storage_service, "base_path", tmp_path)  # 物件屬性
    # 函式層級 mock 仍用 @patch
```

---

## 7. 測試檔案組織結構

### 7.1 目錄結構

```
tests/
├── conftest.py                  # 共用 fixtures、_FakeRow、make_*() 工廠函式、db/db_client/api_client fixtures
├── test_dicom_service.py        # DICOM 上傳整合測試（DB + HTTP）
├── test_validators.py           # DICOM validation 單元測試（純邏輯，無 DB）
├── test_query_api.py            # 查詢 API 的 HTTP 行為測試
├── test_upload_api.py           # （未來）上傳 API 的 HTTP 行為測試
├── test_ai_api.py               # （未來）AI pipeline API 測試
├── test_auth_api.py             # （未來）Authentication API 測試
└── test_storage_service.py      # （未來）儲存後端服務測試
```

### 7.2 檔案命名規則

| 類型 | 命名格式 | 範例 |
|------|---------|------|
| API endpoint 測試 | `test_<資源或功能>_api.py` | `test_query_api.py`, `test_upload_api.py` |
| Service / 業務邏輯測試 | `test_<service名稱>_service.py` | `test_dicom_service.py` |
| 工具函式測試 | `test_<模組名稱>.py` | `test_validators.py` |
| 共用測試工具 | `conftest.py` | （固定命名） |

### 7.3 Class 命名規則

```
Test<動詞><資源名稱>

TestListStudies
TestGetSeries
TestGetInstance
TestCreateInstance
TestAiSegment
TestAiResult
TestUploadDicom
```

### 7.4 函式命名規則

所有測試函式必須以 `test_` 開頭，後接**動詞短語描述行為**：

```python
# ✅ 正確
def test_returns_series_when_found(self, mock_get): ...
def test_returns_404_when_not_found(self, mock_get): ...
def test_id_passed_correctly(self, mock_get): ...

# ❌ 錯誤
def test_series(self): ...
def test_404(self): ...
def test_1(self): ...
```

### 7.5 Helper / 工廠函式命名規則

```python
# ORM 物件工廠
make_<model小寫>()          # make_study(), make_series(), make_instance()

# 外部物件工廠
make_mock_<物件小寫>()      # make_mock_ds()

# 私有測試工具
_<CamelCase>               # _FakeRow
```

### 7.6 測試分組策略

- 同一個 endpoint 的所有測試**必須**放在同一個 class 中
- 不同 endpoint 的測試**不得**混入同一個 class
- 整合測試（DB）與 API 測試（mock）**不得**放在同一個檔案

---

## 8. 未來擴充規則

### 8.1 AI Pipeline 測試（`test_ai_api.py`）

**現有基礎**：`TestAiSegment` 與 `TestAiResult` 已在 `test_query_api.py` 中建立。

**未來新增規則**：
- AI endpoint 的測試必須**獨立抽出**至 `test_ai_api.py`
- 非同步 AI job 狀態（`queued` / `processing` / `completed` / `failed`）必須各有一個測試案例
- AI 結果驗證：不斷言結果的具體數值，只斷言結構（如 `"result" in data`）

```python
class TestAiSegmentAsync:
    def test_returns_queued_on_submission(self): ...
    def test_returns_404_when_instance_not_found(self): ...
    def test_id_passed_correctly(self): ...

class TestAiJobStatus:
    def test_returns_processing_status(self): ...
    def test_returns_completed_with_result(self): ...
    def test_returns_failed_with_error(self): ...
```

### 8.2 ✅ 已完成：儲存後端測試路徑隔離

- 使用 `tmp_path` fixture（pytest 內建）管理暫存檔案
- `STORAGE_PATH` 硬編碼問題已修正，改用 `monkeypatch.setenv` + `monkeypatch.setattr`
- 儲存後端測試可在 CI 環境中正常執行

```python
# ✅ 現行標準做法
def test_upload_stores_file_locally(tmp_path, monkeypatch):
    monkeypatch.setenv("STORAGE_PATH", str(tmp_path))
    monkeypatch.setattr(storage_service, "base_path", tmp_path)
    ...
    expected_path = tmp_path / "PATIENT_ID" / "STUDY_UID" / "file.dcm"
    assert expected_path.exists()
```

### 8.3 非同步 Job 測試

**前提**：若未來引入 Celery、RQ 或 BackgroundTasks：

- Job 排程邏輯與 Job 執行邏輯**分開測試**
- 排程測試：mock 掉 job worker，只測試「任務是否被加入佇列」
- 執行測試：直接呼叫 job 函式，不透過佇列

```python
# 排程測試
@patch("tasks.celery_app.send_task")
def test_segment_job_is_queued(self, mock_send): ...

# 執行測試（直接呼叫）
def test_segment_job_produces_output(): ...
```

### 8.4 Authentication 測試（`test_auth_api.py`）

**前提**：若未來加入 JWT / API Key 驗證：

- 建立 `make_auth_headers(token="test-token")` 工廠函式（加入 `conftest.py`）
- 每個受保護 endpoint 的測試 class **必須**包含：
  - `test_returns_401_when_no_token`
  - `test_returns_403_when_insufficient_permission`（若有 RBAC）
  - `test_succeeds_with_valid_token`
- Token 產生邏輯放在 `conftest.py`，不在個別測試中重複

### 8.5 WebSocket 測試

**前提**：若未來加入 WebSocket endpoint：

- 使用 `TestClient` 的 `with client.websocket_connect("/ws") as ws:` 模式
- 每個 WebSocket 測試必須有明確的連線關閉邏輯（避免測試 hang）
- 建立獨立的 `test_ws_<功能>.py` 檔案

### 8.6 Frontend 整合測試

- **不在本 repository 中實作**前端整合測試
- 若需要 E2E 測試，使用獨立的 `e2e/` 目錄與 Playwright/Cypress
- 本 `tests/` 目錄只包含後端測試

### 8.7 測試規模擴充門檻

當以下任一條件達成時，必須重新審視測試架構：

| 條件 | 應對措施 |
|------|---------|
| 測試檔案超過 **10 個** | 建立 `tests/unit/`, `tests/integration/`, `tests/api/` 子目錄 |
| 整合測試出現 SQLite/PostgreSQL 不一致 bug | 引入 Docker PostgreSQL 或 `pytest-postgresql` |
| `conftest.py` 超過 **200 行** | 拆分為 `conftest_db.py`, `conftest_fake_data.py` 等 |
| CI 測試時間超過 **5 分鐘** | 引入 `pytest-xdist` 平行執行 |

---

## 9. AI 貢獻者規則

> **本節為強制規範。所有 AI 工具（Claude、Copilot、Cursor 等）在生成或修改測試程式碼時，必須遵守以下規則。**

---

### 9.1 核心原則：擴充優先，不得重新設計

```
❌ 禁止：引入新的測試框架（如 Hypothesis、testcontainers）而不討論
❌ 禁止：重寫現有可運作的測試
❌ 禁止：新增未在本規範中定義的設計模式
❌ 禁止：建立新的 fixture 層次（如巢狀 fixture 依賴樹）
❌ 禁止：將 `make_*()` 函式移入 class 內部
❌ 禁止：使用 `pytest.mark.parametrize` 取代明確的測試函式（除非有明確討論）

✅ 必須：遵循現有的 `make_*()` 工廠函式模式
✅ 必須：新 endpoint 對應新的 `Test<動詞><資源>` class
✅ 必須：每個 class 包含 success、404、id_propagation 三個基本測試
✅ 必須：從 `conftest.py` 引入共用工具，不得在測試檔案中重複定義
✅ 必須：patch 目標為 `main.<function>`，不得 patch 定義處
```

### 9.2 新增測試的標準流程

在生成新測試前，AI 貢獻者必須依序確認：

1. **確認 `conftest.py` 中是否已有可用的工廠函式**
   - 若有 → 直接引入使用
   - 若無 → 新增至 `conftest.py`，不在測試檔案中定義

2. **確認對應的 test class 是否已存在**
   - 若有 → 在既有 class 中新增測試方法
   - 若無 → 建立新的 `Test<動詞><資源>` class

3. **確認 patch 目標**
   - 永遠使用 `@patch("main.<function_name>")`

4. **確認必要測試案例是否完整**
   - `test_returns_<內容>_when_found`
   - `test_returns_404_when_not_found`
   - `test_id_passed_correctly`

### 9.3 禁止的生成模式

AI 貢獻者**不得**生成以下模式：

```python
# ❌ 禁止：在測試檔案中重複定義 _FakeRow
class _FakeRow:
    pass  # 應從 conftest.py 引入

# ❌ 禁止：巢狀 fixture
@pytest.fixture
def study(db_session):  # 不建立依賴 fixture 的 fixture
    return db_session.add(Study(...))

# ❌ 禁止：測試中使用魔法數字而不說明
assert response.json()["count"] == 42  # 42 從哪來？

# ❌ 禁止：單一測試函式測試多個行為
def test_series_endpoint(self, mock_get):
    # 測試成功案例
    mock_get.return_value = make_series()
    response = client.get("/series/1")
    assert response.status_code == 200
    # 測試 404（不應在同一函式中）
    mock_get.return_value = None
    response = client.get("/series/999")
    assert response.status_code == 404

# ❌ 禁止：patch 定義處
@patch("db_service.get_series_by_id")  # 應 patch main.get_series_by_id
```

### 9.4 必須生成的模式

```python
# ✅ 標準 API 測試 class 結構
from tests.conftest import make_study, make_series, make_instance

class TestGetSeries:

    @patch("main.get_series_by_id")
    def test_returns_series_when_found(self, mock_get):
        mock_get.return_value = make_series(id=10)
        response = client.get("/series/10")
        assert response.status_code == 200
        assert response.json()["id"] == 10

    @patch("main.get_series_by_id")
    def test_returns_404_when_not_found(self, mock_get):
        mock_get.return_value = None
        response = client.get("/series/999")
        assert response.status_code == 404
        assert "not found" in response.json()["detail"].lower()

    @patch("main.get_series_by_id")
    def test_id_passed_correctly(self, mock_get):
        mock_get.return_value = make_series(id=42)
        client.get("/series/42")
        call_args = mock_get.call_args[0]
        assert call_args[1] == 42
```

### 9.5 規範變更流程

若 AI 貢獻者認為現有規範需要調整：

1. **不得**直接在測試程式碼中引入新模式而不更新 TESTSPEC.md
2. 必須先在 PR / commit message 中說明：「建議修改 TESTSPEC.md 第 X 節」
3. 等待人工審核後，同步更新 TESTSPEC.md 與測試程式碼

---

## 附錄 A：現有測試問題彙整

以下為本規範制定時識別的現有問題，供未來改善參考：

| 問題 | 位置 | 優先級 | 狀態 |
|------|------|-------|---------|
| `_FakeRow` 與 `make_*()` 重複定義 | `test_query_api.py` | 🔴 高 | ✅ 已完成：遷移至 `conftest.py` |
| 硬編碼 `STORAGE_PATH = "./storage"` | `test_dicom_service.py` | 🔴 高 | ✅ 已完成：改用 `tmp_path` + `monkeypatch` |
| 模組層級 `dependency_overrides` 永久覆寫 | `test_dicom_service.py` | 🟡 中 | ✅ 已完成：遷移至 `db_client` fixture |
| `test_dicom_service.py` 混合整合測試與單元測試 | `test_dicom_service.py` | 🟡 中 | ✅ 已完成：validation 測試拆分至 `test_validators.py` |
| `TestAiSegment` / `TestAiResult` 放在 query API 測試中 | `test_query_api.py` | 🟢 低 | 🔲 待處理：未來拆分至 `test_ai_api.py` |

---

## 附錄 B：快速參考卡

```
新增 API endpoint 測試時：
1. 在對應的 test_*_api.py 建立 Test<動詞><資源> class
2. 從 conftest.py 引入 make_*() 函式
3. 加入 success / 404 / id_propagation 三個測試
4. patch 目標：@patch("main.<function_name>")
5. 函式簽名注入 api_client fixture（不得使用模組層級 client）

新增整合測試時：
1. 放在 test_<service>_service.py
2. 函式簽名注入 db fixture（不得手動建立 TestingSessionLocal）
3. HTTP 測試注入 db_client fixture
4. DB schema 隔離由 conftest.py 中 reset_db（autouse=True）處理
5. 不使用 TestClient，直接呼叫 DatabaseService

新增工廠函式時：
1. 放在 tests/conftest.py
2. 命名：make_<model小寫>() 或 make_mock_<物件>()
3. 所有參數必須有預設值
```

---

*本文件為 TESTSPEC.md v1.1，基於 `test_dicom_service.py`、`test_query_api.py`、`test_validators.py` 的實際程式碼分析制定，並反映 fixture 化重構後的現行架構。*
