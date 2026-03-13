# Redmine API — 專案管理與枚舉 (Projects & Enumerations)

本文件涵蓋專案的建立、查詢、更新、刪除、成員管理，以及各種枚舉資料的查詢。

---

## POST /projects.json — 建立專案

```bash
curl -s -X POST \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project": {
      "name": "Phase 2",
      "identifier": "phase-2",
      "parent_id": 1,
      "description": "第二階段開發",
      "is_public": false,
      "inherit_members": true
    }
  }' \
  "${URL}/projects.json"
```

### 欄位說明

- **必要欄位**：`name`、`identifier`
- **可選欄位**：
  - `description`：專案描述
  - `parent_id`：父專案 ID（建立子專案時使用）
  - `is_public`：是否公開（預設依 Redmine 設定）
  - `inherit_members`：是否繼承父專案成員

### identifier 規則

- 只允許小寫英數字與短橫線（`-`）
- 長度 1–100 字元
- **建立後不可修改**，請謹慎命名

---

## GET /projects.json — 查詢專案列表

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/projects.json?include=trackers,issue_categories,enabled_modules,time_entry_activities"
```

### include 參數說明

- `trackers`：該專案啟用的 tracker 列表
- `issue_categories`：議題分類
- `enabled_modules`：啟用的模組（例如 issue_tracking、time_tracking）
- `time_entry_activities`：工時活動類型

---

## GET /projects/{id}.json — 查詢單一專案

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/projects/project-x.json?include=trackers,issue_categories,enabled_modules,time_entry_activities"
```

### 注意事項

- `id` 可以是數字 ID 或 identifier 字串（例如 `project-x`）
- 支援相同的 `include` 參數

---

## PUT /projects/{id}.json — 更新專案

```bash
curl -s -X PUT \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "project": {
      "name": "Phase 2 (Updated)",
      "description": "更新後的描述",
      "is_public": true
    }
  }' \
  "${URL}/projects/phase-2.json"
```

### 注意事項

- `identifier` 建立後**不可修改**，不需傳入此欄位
- 成功回應為 HTTP 204 No Content（無回應 body）

---

## DELETE /projects/{id}.json — 刪除專案

```bash
curl -s -X DELETE \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/projects/phase-2.json"
```

### 警告：極高風險操作

- 會**永久刪除**專案下所有議題、工時記錄、成員等所有資料
- **無法復原**
- 執行前務必：
  1. 顯示專案名稱與 identifier 讓使用者確認
  2. 查詢並顯示該專案現有議題數量
  3. 要求使用者明確輸入確認文字後才執行

---

## POST /projects/{id}/memberships.json — 新增成員

```bash
curl -s -X POST \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "membership": {
      "user_id": 5,
      "role_ids": [3]
    }
  }' \
  "${URL}/projects/project-x/memberships.json"
```

### 欄位說明

- `user_id`：要加入的使用者 ID
- `role_ids`：角色 ID 陣列（可指定多個角色）；可透過 `GET /roles.json` 查詢可用角色

---

## GET /projects/{id}/memberships.json — 查詢成員

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/projects/project-x/memberships.json"
```

回應包含每位成員的 `user`（id、name）與 `roles`（id、name）資訊。

---

## 枚舉查詢 (Enumerations)

枚舉資料通常是靜態設定，建議在工作流程開始時一次查詢並快取，避免重複呼叫。

| 端點 | 說明 | 回應 key |
|------|------|----------|
| `GET /enumerations/time_entry_activities.json` | 活動類型（工時用） | `time_entry_activities` |
| `GET /trackers.json` | Tracker 列表（Bug、Feature 等） | `trackers` |
| `GET /issue_statuses.json` | 狀態列表（新建、進行中、已解決等） | `issue_statuses` |
| `GET /enumerations/issue_priorities.json` | 優先權列表（低、普通、高、緊急） | `issue_priorities` |
| `GET /roles.json` | 角色列表（管理者、開發者等） | `roles` |
| `GET /users/current.json` | 當前使用者資訊 | `user` |

### GET /enumerations/time_entry_activities.json

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/enumerations/time_entry_activities.json"
```

### GET /trackers.json

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/trackers.json"
```

### GET /issue_statuses.json

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/issue_statuses.json"
```

### GET /enumerations/issue_priorities.json

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/enumerations/issue_priorities.json"
```

### GET /roles.json

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/roles.json"
```

### GET /users/current.json

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/users/current.json"
```

---

## 自然語言情境範例

| 使用者說的話 | 對應 API 操作 |
|-------------|--------------|
| 「建一個子專案叫『Phase 2』，放在 project-x 下面」 | `POST /projects.json`，設定 `parent_id` 為 project-x 的 ID |
| 「列出所有專案」 | `GET /projects.json` |
| 「把 user_id 5 加到 project-x，角色是開發者」 | `POST /projects/project-x/memberships.json`，傳入對應的 `role_ids` |
| 「列出所有可用的 tracker」 | `GET /trackers.json` |
| 「查一下我是誰」 | `GET /users/current.json` |
| 「列出 project-x 的所有成員」 | `GET /projects/project-x/memberships.json` |
| 「查一下有哪些工時活動類型」 | `GET /enumerations/time_entry_activities.json` |
