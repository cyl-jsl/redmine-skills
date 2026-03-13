# Redmine API — 議題操作 (Issues)

本文件涵蓋議題（Issues）的 CRUD 操作與附件上傳流程。

所有呼叫均透過 `bin/redmine-api` CLI 工具，認證由工具內部處理。

---

## POST /issues.json — 建立議題

```bash
python bin/redmine-api POST /issues.json '{
  "issue": {
    "project_id": "project-x",
    "subject": "登入頁面壞了",
    "tracker_id": 1,
    "priority_id": 2,
    "assigned_to_id": 5,
    "description": "使用者無法登入系統"
  }
}'
```

### 欄位說明

- **必要欄位**：`project_id`、`subject`
- **可選欄位**：`tracker_id`、`status_id`、`priority_id`、`assigned_to_id`、`description`、`parent_issue_id`、`start_date`、`due_date`、`estimated_hours`、`category_id`、`fixed_version_id`
- tracker / status / priority IDs 可透過枚舉 API 查詢（詳見 `api-projects.md`）

### 成功回應

HTTP 201，回應 body 包含完整議題物件：

```json
{
  "issue": {
    "id": 123,
    "project": {"id": 1, "name": "project-x"},
    "subject": "登入頁面壞了",
    "status": {"id": 1, "name": "New"}
  }
}
```

---

## GET /issues.json — 查詢議題列表

```bash
python bin/redmine-api GET '/issues.json?project_id=project-x&status_id=open&assigned_to_id=me&sort=updated_on:desc&limit=25'
```

### 過濾參數

| 參數 | 說明 |
|------|------|
| `project_id` | 專案 ID 或識別碼 |
| `tracker_id` | 追蹤器 ID |
| `status_id` | 狀態 ID；特殊值：`open`、`closed`、`*`（全部） |
| `assigned_to_id` | 指派對象 ID；特殊值：`me`（目前使用者） |
| `sort` | 排序欄位，格式 `field:asc` 或 `field:desc` |
| `limit` | 每頁筆數（最大 100，預設 25） |
| `offset` | 起始偏移量（分頁用） |

### 取得關聯資料

加入 `include` 參數可一次取回關聯資料：

```bash
python bin/redmine-api GET '/issues.json?project_id=project-x&include=journals,relations,attachments'
```

### 回應格式

```json
{
  "issues": [...],
  "total_count": 150,
  "offset": 0,
  "limit": 25
}
```

---

## GET /issues/{id}.json — 查詢單一議題

```bash
python bin/redmine-api GET '/issues/123.json?include=children,attachments,relations,changesets,journals,watchers'
```

### include 選項

| 值 | 說明 |
|----|------|
| `children` | 子議題列表 |
| `attachments` | 附件列表 |
| `relations` | 關聯議題 |
| `changesets` | 關聯的版本控制提交 |
| `journals` | 變更記錄（含備註） |
| `watchers` | 關注者列表 |

---

## PUT /issues/{id}.json — 更新議題

```bash
python bin/redmine-api PUT /issues/100.json '{
  "issue": {
    "status_id": 3,
    "notes": "已修復，請驗證"
  }
}'
```

### 注意事項

- 可更新任何建立時的欄位（`subject`、`assigned_to_id`、`priority_id` 等）
- `notes` 欄位會新增一筆日誌記錄（留言）
- 最常見用途：更新狀態 + 加入備註
- 成功回應 `{"status": "ok", "http_code": 200}`

---

## DELETE /issues/{id}.json — 刪除議題

```bash
python bin/redmine-api DELETE /issues/200.json
```

### 注意事項

- **高風險操作**：需額外確認（顯示議題編號和標題）
- 建議先 GET 查詢議題資訊，確認後再執行刪除
- 成功回應 `{"status": "ok", "http_code": 200}`

---

## 附件上傳

附件上傳為兩步驟流程：先上傳檔案取得 token，再將 token 附加到議題。

### Step 1：上傳檔案取得 token

```bash
python bin/redmine-api upload /path/to/report.pdf
```

回應：

```json
{"upload": {"token": "abc123..."}}
```

### Step 2：附加到議題

```bash
python bin/redmine-api PUT /issues/200.json '{
  "issue": {
    "uploads": [
      {
        "token": "abc123...",
        "filename": "report.pdf",
        "content_type": "application/pdf"
      }
    ]
  }
}'
```

### 注意事項

- 可在同一個 PUT 請求中附加多個檔案（`uploads` 為陣列）
- token 有效期限通常為數小時，請在取得後盡快使用

---

## 自然語言情境範例

| 使用者說的話 | 對應操作 |
|---|---|
| 「在 project-x 建一個 bug，標題是『登入頁面壞了』」 | POST /issues.json |
| 「查一下指派給我的所有未關閉議題」 | GET /issues.json?assigned_to_id=me&status_id=open |
| 「把 issue #100 狀態改成已解決，備註『已修復』」 | PUT /issues/100.json（含 status_id + notes） |
| 「issue #200 加上附件 report.pdf」 | 附件兩步驟流程（upload → PUT） |
| 「列出 project-x 最近更新的 10 個議題」 | GET /issues.json?project_id=project-x&sort=updated_on:desc&limit=10 |
| 「把 issue #300 指派給 user_id 5」 | PUT /issues/300.json（含 assigned_to_id: 5） |
