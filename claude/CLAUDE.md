# CLAUDE.md

Version: 0.1
Purpose: 提供 Claude 與其他 AI Coding Agent 在此 Repository 工作時必須遵守的固定操作規範。

---

# 文件定位

本文件為 Repository-Level AI Operating Contract。

目的：

- 固定 AI 行為
- 降低 context drift
- 避免 AI 任意重構
- 降低非預期修改
- 建立穩定可預測的輸出
- 提升跨 Session 開發一致性
- 提升跨 AI 平台相容性

此文件可被：

- Claude
- GPT
- Gemini
- Cursor
- Copilot
- Roo
- Cline
- 未來 AI Coding Agent

直接閱讀並理解。

本文件屬於永久性 Repository 規範。

---

# 核心原則

## 1. Minimal Change Principle（最小修改原則）

僅修改完成任務所必要的內容。

避免：

- 不必要重構
- 大規模格式化
- 無關優化
- 任意 rename
- 修改 unrelated logic
- 架構重寫

優先：

- patch existing code
- 保持既有架構
- 保持 backward compatibility
- 小範圍增量修改

---

## 2. Patch-First Strategy（Patch 優先策略）

對於既有檔案：

- 優先提供 patch-style 修改
- 避免整份檔案重寫
- 僅輸出必要修改區塊

對於新檔案：

- 可提供完整檔案內容

---

## 3. Scope Isolation（範圍隔離）

禁止修改：

- unrelated files
- unrelated functions
- unrelated APIs
- unrelated tests
- unrelated business logic

所有修改都必須直接與當前任務相關。

---

## 4. Architecture Preservation（架構保護）

必須尊重既有專案架構。

禁止：

- 任意改變 architecture
- 合併原本分層
- 移動 business logic 到不正確層級
- 任意更換 framework
- 重組整個 project structure

必須保留：

- folder structure
- naming convention
- dependency strategy
- layering strategy

---

## 5. Backward Compatibility（向後相容）

避免破壞：

- existing API
- existing schema
- public interface
- current tests
- existing behavior

若必須產生 breaking change：

- 必須明確說明
- 必須解釋 impact
- 不可 silent breaking change

---

# 允許行為（Allowed Behavior）

除非使用者明確禁止，AI 可以：

- 新增小型 helper function
- 新增 focused tests
- 新增 minimal imports
- 新增 isolated feature files
- 小幅改善 local readability
- 修正明確 bug

---

# 禁止行為（Forbidden Behavior）

除非使用者明確要求，AI 不可：

- 大規模重構
- rename 核心檔案
- 修改 architecture
- 升級 dependency
- 新增 framework
- 修改 CI/CD
- 修改 unrelated tests
- silent behavior change
- 移除 backward compatibility
- 引入 hidden side effects
- 任意新增 abstraction
- 過度工程化

---

# 輸出規則（Output Rules）

每次實作任務時，輸出順序應為：

1. Analysis
2. Implementation Plan
3. Modified Files
4. Code Changes
5. Risks / Notes
6. Test Recommendation

---

# 檔案輸出規則

## Existing Files

既有檔案：

- 優先輸出 patch section
- 僅提供必要修改區塊
- 避免輸出完整未修改檔案

---

## New Files

新檔案：

- 可直接提供完整檔案內容

---

# 修改透明性（Modification Transparency）

所有修改必須說明：

- 為什麼需要修改
- 修改了什麼
- 哪些行為被影響
- 哪些檔案被修改

禁止：

- 無說明直接輸出大量 code
- 隱藏行為變更

---

# 測試規則（Testing Expectations）

適用情況下：

- 應補 unit tests
- 應保持既有 test behavior
- 避免重寫 unrelated tests
- 優先 isolated tests

---

# Dependency 規則

禁止：

- 任意新增 dependency
- 任意升級 major version

若必須新增 dependency：

- 必須解釋原因
- 必須說明 impact
- 必須保持 minimal footprint

---

# Database 規則

禁止：

- 非必要 schema 修改
- 任意 rename table/column
- destructive migration

Schema 修改必須：

- 明確說明
- 提供 migration note
- 優先保持 backward compatibility

---

# API 規則

必須保持：

- response structure
- status code consistency
- endpoint naming consistency

避免：

- silent API contract changes

---

# Security Rules（安全規則）

必須避免：

- unsafe file operations
- path traversal
- raw SQL injection risk
- secret exposure
- unsafe deserialization

必須驗證：

- external input
- request payload
- file path
- user-provided data

---

# Preferred Engineering Style（偏好工程風格）

優先：

- explicit logic
- readable code
- small focused changes
- deterministic behavior
- low coupling
- composable functions

避免：

- magic behavior
- hidden state
- speculative abstraction
- overengineering

---

# AI Uncertainty Handling（AI 不確定性處理）

若 repository context 不完整：

AI 必須：

- 明確說明 assumption
- 不可 hallucinate architecture
- 必要時要求 clarification
- 不可虛構不存在系統

---

# 任務優先級（Priority Order）

優先順序：

1. Correctness
2. Safety
3. Scope Control
4. Backward Compatibility
5. Readability
6. Performance
7. Optimization

---

# 建議工作流程（Recommended Workflow）

建議流程：

1. 分析 existing structure
2. 確認 minimal change set
3. 提出 implementation plan
4. Incremental implementation
5. 驗證 compatibility
6. 建議 tests

---

# Cross-AI Compatibility（跨 AI 相容）

本文件應避免：

- provider-specific prompt assumptions
- model-specific behavior assumptions

應保持：

- machine-readable structure
- stable headings
- deterministic rule definitions

---

# Override Rule（覆寫規則）

使用者明確指令優先於本文件。

但 AI 仍應：

- 主動警告風險
- 說明 breaking changes
- 說明 architecture impact

---

# End of Specification

