# Redmine API 進階參考

本文件涵蓋分頁、批次操作、錯誤處理等進階模式。

所有呼叫均透過 `bin/redmine-api` CLI 工具，認證由工具內部處理。

---

## 分頁

### 預設參數

- 預設：`?offset=0&limit=25`
- 最大 limit：`100`

### 回應格式

```json
{
  "issues": [...],
  "total_count": 150,
  "offset": 0,
  "limit": 25
}
```

- `total_count`：符合條件的總筆數
- `offset`：目前從第幾筆開始
- `limit`：本次回傳筆數上限

### 分頁查詢範例

```bash
bin/redmine-api GET '/issues.json?offset=0&limit=100&project_id=myproject'
```

---

## 錯誤處理

CLI 工具已內建結構化錯誤處理，錯誤時輸出至 stderr：

```json
{
  "error": {
    "http_code": 422,
    "message": "欄位驗證錯誤",
    "details": {"errors": ["Subject cannot be blank"]}
  }
}
```

| HTTP 狀態碼 | CLI 輸出的 message |
|------------|-------------------|
| 401 | API Key 無效，請檢查 ~/.redmine.json |
| 403 | 權限不足 |
| 404 | 資源不存在 |
| 422 | 欄位驗證錯誤（含 details） |

---

## 批次操作模式

當需要處理超過 25 筆資料時，應使用 Python 搭配 CLI 工具自動分頁：

```python
import json
import subprocess


def redmine_api(method, endpoint, data=None):
    """透過 CLI wrapper 呼叫 Redmine API"""
    cmd = ["bin/redmine-api", method, endpoint]
    if data:
        cmd.append(json.dumps(data))
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.returncode != 0:
        raise RuntimeError(result.stderr)
    return json.loads(result.stdout) if result.stdout.strip() else None


def redmine_get_all(endpoint, resource_key, params=None):
    """自動分頁取得所有資料"""
    all_items = []
    offset = 0
    limit = 100
    while True:
        query = f"?offset={offset}&limit={limit}"
        if params:
            query += "&" + "&".join(f"{k}={v}" for k, v in params.items())
        data = redmine_api("GET", f"{endpoint}{query}")
        all_items.extend(data.get(resource_key, []))
        if offset + limit >= data.get("total_count", 0):
            break
        offset += limit
    return all_items
```

### 範例：取得指定日期區間的所有工時

```python
all_entries = redmine_get_all(
    "/time_entries.json",
    "time_entries",
    params={"spent_on": "><2026-01-01|2026-03-31", "user_id": "me"}
)
total_hours = sum(e["hours"] for e in all_entries)
print(f"共 {len(all_entries)} 筆，總工時 {total_hours} 小時")
```

### 範例：批次更新議題

```python
issues = redmine_get_all(
    "/issues.json", "issues",
    params={"status_id": "open", "assigned_to_id": "me"}
)
for issue in issues:
    redmine_api("PUT", f"/issues/{issue['id']}.json",
                {"issue": {"notes": "批次更新備註"}})
    print(f"已更新 #{issue['id']}: {issue['subject']}")
```
