# Redmine API — 工時記錄 (Time Entries)

工時登打是最高頻率的操作，本文件涵蓋完整的 CRUD 流程。

---

## POST /time_entries.json — 登打工時

```bash
curl -s -X POST \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{
    "time_entry": {
      "issue_id": 123,
      "hours": 2.0,
      "activity_id": 9,
      "comments": "開發功能",
      "spent_on": "2026-03-13"
    }
  }' \
  "${URL}/time_entries.json"
```

### 欄位說明

- **必要欄位**：`hours`，且須指定 `issue_id` 或 `project_id` 之一
- **可選欄位**：
  - `activity_id`：工時類別（例如開發、測試），可透過 `GET /enumerations/time_entry_activities.json` 查詢（詳見 `api-projects.md`）
  - `comments`：備註說明
  - `spent_on`：工時日期，格式 `YYYY-MM-DD`，預設為今天
  - `user_id`：指定使用者（僅管理員可代填）

---

## GET /time_entries.json — 查詢工時

```bash
curl -s -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/time_entries.json?user_id=me&from=2026-03-10&to=2026-03-14&limit=100"
```

### 篩選參數

| 參數         | 說明                          |
|--------------|-------------------------------|
| `user_id`    | 使用者 ID，`me` 代表自己      |
| `project_id` | 依專案篩選                    |
| `from`       | 起始日期，格式 `YYYY-MM-DD`   |
| `to`         | 結束日期，格式 `YYYY-MM-DD`   |
| `limit`      | 每頁筆數（最大 100）          |
| `offset`     | 分頁偏移量                    |

### 回應格式

- 回應包含 `time_entries` 陣列與 `total_count`
- 每筆工時記錄包含：`id`、`project`、`issue`、`user`、`activity`、`hours`、`comments`、`spent_on`

---

## PUT /time_entries/{id}.json — 更新工時

```bash
curl -s -X PUT \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"time_entry": {"hours": 3.0, "comments": "修正時數"}}' \
  "${URL}/time_entries/456.json"
```

### 說明

- 可更新任意欄位：`hours`、`activity_id`、`comments`、`spent_on`
- 成功時回傳空白內容（HTTP 200）

---

## DELETE /time_entries/{id}.json — 刪除工時

```bash
curl -s -X DELETE \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  "${URL}/time_entries/456.json"
```

### 說明

- 成功時回傳空白內容（HTTP 200）
- **寫入操作**：需確認後執行

---

## 自然語言情境範例

| 使用者說的話 | 對應 API 操作 |
|---|---|
| 「幫我記今天在 issue #123 花了 2 小時做開發」 | POST /time_entries.json，`issue_id=123`，`hours=2`，`activity=開發`，`spent_on=今天` |
| 「查一下我這週的工時」 | GET /time_entries.json?`user_id=me&from=本週一&to=今天` |
| 「把工時 #456 改成 3 小時」 | PUT /time_entries/456.json，`hours=3` |
| 「刪除工時 #456」 | DELETE /time_entries/456.json（需確認） |
| 「這個月在 project-x 總共花了多少時間？」 | GET with `project_id` + `from`/`to`，加總所有 `hours` |
| 「幫我把昨天在 issue #789 的工時從 1 小時改成 1.5 小時」 | 先 GET 查找對應工時記錄 ID，再 PUT 更新 `hours=1.5` |
