# Redmine API 進階參考

本文件涵蓋分頁、批次操作、錯誤處理等進階模式。
基本認證資訊請參考 SKILL.md，本文件僅說明進階用法。

---

## 1. 認證方式

所有 API 請求需帶入以下 Header：

```
X-Redmine-API-Key: {api_key}
Content-Type: application/json
```

設定檔位置：`~/.redmine.json`

```json
{
  "url": "https://your-redmine.example.com",
  "api_key": "your_api_key_here"
}
```

---

## 2. 分頁

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

### curl 分頁範例

```bash
curl -s \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  "${REDMINE_URL}/issues.json?offset=0&limit=100&project_id=myproject"
```

---

## 3. 錯誤處理

| HTTP 狀態碼 | 說明 | 處理建議 |
|------------|------|---------|
| 401 | API Key 無效 | 檢查 `~/.redmine.json` 中的 `api_key` |
| 403 | 權限不足 | 確認該帳號有存取該資源的權限 |
| 404 | 資源不存在 | 確認 ID 或路徑是否正確 |
| 422 | 驗證錯誤 | 讀取回應中的 `errors` 陣列 |

### 422 錯誤回應格式

```json
{"errors": ["Subject cannot be blank"]}
```

### 錯誤處理範例（shell）

```bash
RESPONSE=$(curl -s -o /tmp/response.json -w "%{http_code}" \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  "${REDMINE_URL}/issues.json")

HTTP_CODE=$RESPONSE
if [ "$HTTP_CODE" = "422" ]; then
  echo "驗證錯誤："
  cat /tmp/response.json | python3 -c "import json,sys; d=json.load(sys.stdin); print('\n'.join(d.get('errors', [])))"
fi
```

---

## 4. curl 呼叫範本

### GET

```bash
curl -s \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  "${REDMINE_URL}/issues.json"
```

### POST

```bash
curl -s -X POST \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"issue": {"project_id": 1, "subject": "新議題"}}' \
  "${REDMINE_URL}/issues.json"
```

### PUT

```bash
curl -s -X PUT \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  -d '{"issue": {"subject": "更新後的標題"}}' \
  "${REDMINE_URL}/issues/123.json"
```

### DELETE

```bash
curl -s -X DELETE \
  -H "X-Redmine-API-Key: ${API_KEY}" \
  -H "Content-Type: application/json" \
  "${REDMINE_URL}/issues/123.json"
```

---

## 5. Python 呼叫範本

使用 `subprocess` + `curl` 方式，無需安裝任何第三方套件。

```python
import json, subprocess

def redmine_request(url, api_key, method, endpoint, data=None):
    """Generic Redmine API request"""
    cmd = ["curl", "-s", "-X", method,
           "-H", f"X-Redmine-API-Key: {api_key}",
           "-H", "Content-Type: application/json"]
    if data:
        cmd.extend(["-d", json.dumps(data)])
    cmd.append(f"{url}{endpoint}")
    result = subprocess.run(cmd, capture_output=True, text=True)
    if result.stdout:
        return json.loads(result.stdout)
    return None

def redmine_get_all(url, api_key, endpoint, resource_key, params=None):
    """Fetch all pages of a paginated resource"""
    all_items = []
    offset = 0
    limit = 100
    while True:
        query = f"?offset={offset}&limit={limit}"
        if params:
            query += "&" + "&".join(f"{k}={v}" for k, v in params.items())
        data = redmine_request(url, api_key, "GET", f"{endpoint}{query}")
        all_items.extend(data.get(resource_key, []))
        if offset + limit >= data.get("total_count", 0):
            break
        offset += limit
    return all_items
```

### 使用範例

```python
import json

# 讀取設定
with open(os.path.expanduser("~/.redmine.json")) as f:
    config = json.load(f)

url = config["url"]
api_key = config["api_key"]

# 取得單一資源
issue = redmine_request(url, api_key, "GET", "/issues/123.json")

# 建立資源
new_issue = redmine_request(url, api_key, "POST", "/issues.json",
    data={"issue": {"project_id": 1, "subject": "新議題"}})

# 取得所有時間記錄（自動分頁）
all_entries = redmine_get_all(url, api_key, "/time_entries.json", "time_entries",
    params={"spent_on": "><2026-01-01|2026-03-13"})
```

---

## 6. 批次操作模式

### 適用時機

當需要處理超過 25 筆資料時，應使用分頁批次取得所有資料。

### 模式說明

使用 `redmine_get_all` 自動處理分頁，一次最多取 100 筆，直到取完所有資料為止。

### 範例：取得指定日期區間的所有時間記錄

```python
import json, os, subprocess

# 讀取設定
with open(os.path.expanduser("~/.redmine.json")) as f:
    config = json.load(f)

url = config["url"]
api_key = config["api_key"]

# 取得 2026 年 Q1 所有時間記錄
all_entries = redmine_get_all(
    url, api_key,
    "/time_entries.json",
    "time_entries",
    params={
        "spent_on": "><2026-01-01|2026-03-31",
        "user_id": "me"
    }
)

print(f"共取得 {len(all_entries)} 筆時間記錄")

# 計算總工時
total_hours = sum(e["hours"] for e in all_entries)
print(f"總工時：{total_hours} 小時")
```

### 批次更新模式

```python
# 取得所有符合條件的議題後逐一更新
issues = redmine_get_all(url, api_key, "/issues.json", "issues",
    params={"status_id": "open", "assigned_to_id": "me"})

for issue in issues:
    redmine_request(url, api_key, "PUT", f"/issues/{issue['id']}.json",
        data={"issue": {"custom_fields": [{"id": 5, "value": "reviewed"}]}})
    print(f"已更新議題 #{issue['id']}: {issue['subject']}")
```
