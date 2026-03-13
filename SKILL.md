---
name: redmine
description: >
  當使用者以自然語言要求操作 Redmine 時觸發——登打工時、建立/查詢/更新議題、
  管理專案與子專案、查詢狀態與統計。
  主動更新時機：發現新的 API 端點需求或互動模式時，更新對應 references。
---

# Redmine API Skill

## 核心原則

| 原則 | 說明 |
|------|------|
| 安全分級 | 讀取操作（GET）直接執行並回報結果；寫入操作（POST/PUT/DELETE）先顯示操作摘要，等使用者確認後才執行 |
| 最小呼叫 | 預設用 curl 完成；批次、鏈式或需要複雜解析時切換為 Python |
| 設定檔驅動 | 連線資訊從 `~/.redmine.json` 讀取，skill 本體不含任何憑證 |
| 意圖路由 | 根據使用者意圖查下方路由表，按需載入對應 reference，不要一次全讀。複合意圖時依序載入所有相關 reference |
| 連線韌性 | 設定檔不存在時引導使用者建立（指向 setup.md）；API Key 無效時提示檢查設定檔 |

## 連線設定

每次操作前，先用 Read 工具讀取 `~/.redmine.json` 取得連線資訊：

```json
{
  "url": "https://redminesrv.ksi.com.tw",
  "api_key": "your-api-key-here"
}
```

若檔案不存在，引導使用者參考 `setup.md` 建立設定檔。

## 認證 Header

所有 API 請求須帶：

```
X-Redmine-API-Key: {api_key}
Content-Type: application/json
```

## 基本錯誤碼

| HTTP 狀態碼 | 意義 | 處理方式 |
|-------------|------|----------|
| 200/201 | 成功 | 回報結果 |
| 401 | API Key 無效 | 提示使用者檢查 `~/.redmine.json` 中的 api_key |
| 403 | 權限不足 | 告知使用者該操作需要更高權限 |
| 404 | 資源不存在 | 告知使用者指定的資源（議題/專案等）不存在 |
| 422 | 欄位錯誤 | 顯示 Redmine 回傳的 errors 陣列內容 |

## 意圖路由表

| 使用者意圖 | 載入的 Reference |
|-----------|-----------------|
| 工時登打、查詢工時、工時統計 | `references/api-time-entries.md` |
| 建立/查詢/更新/指派議題、上傳附件 | `references/api-issues.md` |
| 專案、子專案、成員、活動類型、枚舉查詢 | `references/api-projects.md` |
| 分頁處理、批次操作、進階錯誤處理 | `references/api-common.md` |

## 互動流程

1. 讀取 `~/.redmine.json` 取得 `url` 和 `api_key`
2. 判斷使用者意圖，從路由表找到對應 reference
3. 用 Read 工具載入該 reference
4. 依 reference 中的 API 說明組合請求
5. **讀取操作**：直接用 Bash 執行 curl，回報結果
6. **寫入操作**：先顯示操作摘要（什麼操作、哪個資源、關鍵欄位），等使用者確認後才執行
