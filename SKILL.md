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
| 零接觸憑證 | **禁止**使用 Read 工具讀取 `~/.redmine.json`。所有 API 呼叫一律透過 `bin/redmine-api` CLI 工具，認證由 wrapper 內部處理，API Key 不得出現在對話中 |
| 意圖路由 | 根據使用者意圖查下方路由表，按需載入對應 reference，不要一次全讀。複合意圖時依序載入所有相關 reference |
| 連線韌性 | CLI 工具會自動檢查設定檔；若不存在，引導使用者參考 `setup.md` 建立 |

## CLI 工具

所有 API 操作透過 `bin/redmine-api` 執行，此工具：

- 內部讀取 `~/.redmine.json`，自動處理認證
- 自動遮蔽回應中的敏感欄位（`api_key`、`password` 等）
- 驗證設定檔權限（須 ≤ 600）與 HTTPS 連線
- 錯誤時輸出結構化的中文訊息

```
用法：
  bin/redmine-api <METHOD> <ENDPOINT> [JSON_BODY]
  bin/redmine-api upload <FILE_PATH>
```

## 基本錯誤碼

| HTTP 狀態碼 | 意義 | CLI 輸出 |
|-------------|------|----------|
| 200/201 | 成功 | 輸出 JSON 結果 |
| 401 | API Key 無效 | 提示檢查 `~/.redmine.json` |
| 403 | 權限不足 | 告知需要更高權限 |
| 404 | 資源不存在 | 告知資源不存在 |
| 422 | 欄位錯誤 | 顯示 errors 陣列 |

## 意圖路由表

| 使用者意圖 | 載入的 Reference |
|-----------|-----------------|
| 工時登打、查詢工時、工時統計 | `references/api-time-entries.md` |
| 建立/查詢/更新/指派議題、上傳附件 | `references/api-issues.md` |
| 專案、子專案、成員、活動類型、枚舉查詢 | `references/api-projects.md` |
| 分頁處理、批次操作、進階錯誤處理 | `references/api-common.md` |

## 互動流程

1. 判斷使用者意圖，從路由表找到對應 reference
2. 用 Read 工具載入該 reference
3. 依 reference 中的 API 說明，透過 `bin/redmine-api` 組合請求
4. **讀取操作**：直接用 Bash 執行 CLI 工具，回報結果
5. **寫入操作**：先顯示操作摘要（什麼操作、哪個資源、關鍵欄位），等使用者確認後才執行

## 安全禁令

- **禁止** 使用 Read 工具讀取 `~/.redmine.json`
- **禁止** 在 curl 或任何命令中手動帶入 API Key
- **禁止** 在對話中顯示、複述或推測 API Key 內容
