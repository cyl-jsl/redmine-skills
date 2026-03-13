# Redmine API Skill 設計文件

## 概述

建立一個 Claude Code Skill，讓 AI agent 能以自然語言操作 Redmine REST API，涵蓋專案管理者與執行者的日常操作：工時登打、議題管理、專案建置、狀態查詢等。

## 需求摘要

- **使用者角色**：專案管理者 + 專案執行者
- **高頻操作**：工時登打
- **中頻操作**：議題 CRUD、狀態追蹤、指派
- **低頻操作**：專案/子專案建立、活動類型管理
- **擴充性**：skill 隨時可能新增操作，結構需支援低成本擴充
- **跨平台**：macOS 與 Windows 皆可使用
- **Redmine 環境**：公司內網自架，API Key 認證

## 設計決策

### 1. 架構：單一 Skill + references 按需載入（方案 C）

採用單一 SKILL.md 搭配路由表，按使用者意圖載入對應 reference。

**理由**：
- 單一 skill 不浪費 metadata 預算（相較於拆成多個 skill）
- 路由表讓 AI 精準載入對的 reference（相較於無路由的純 references）
- 擴充只需加一份 reference + 路由表加一行

### 2. 互動模式：安全分級

- **讀取操作**（查詢）：AI 直接執行，回報結果
- **寫入操作**（建立/更新/刪除）：AI 先顯示操作摘要，確認後執行

### 3. 技術方式：原則導向混用

- 預設用最簡單的方式（curl）
- 當操作涉及批次、鏈式或複雜解析時，自然切換為 Python
- skill 給予原則性指導，不硬性規定使用哪種方式

### 4. 連線設定：獨立設定檔

使用 `~/.redmine.json`，跨平台通用：

```json
{
  "url": "https://redminesrv.ksi.com.tw",
  "api_key": "your-api-key-here"
}
```

### 5. 部署方式：repo + symlink

- **Source of truth**：`/Users/ccy/repos/sideprojects/redmine-skills/`（git repo）
- **全域 skill**：`~/.claude/skills/redmine` → symlink 指向 repo
- 團隊成員 clone repo 後建立 symlink 即可使用，`git pull` 即時更新

## 檔案結構

```
redmine-skills/                    ← git repo
├── SKILL.md                       ← 核心原則 + 路由表（<5k tokens）
├── setup.md                       ← 安裝指引（symlink、設定檔、驗證）
├── references/
│   ├── api-common.md              ← 認證、錯誤處理、分頁、呼叫範本
│   ├── api-time-entries.md        ← 工時登打/查詢/統計
│   ├── api-issues.md              ← 議題 CRUD、指派、附件
│   └── api-projects.md            ← 專案/子專案、成員、活動類型、枚舉
└── docs/
    └── superpowers/
        └── specs/
            └── 2026-03-13-redmine-skill-design.md  ← 本文件
```

## SKILL.md 設計

### Frontmatter

```yaml
---
name: redmine
description: >
  當使用者以自然語言要求操作 Redmine 時觸發——登打工時、建立/查詢/更新議題、
  管理專案與子專案、查詢狀態與統計。
  主動更新時機：發現新的 API 端點需求或互動模式時，更新對應 references。
---
```

### 核心原則

| 原則 | 說明 |
|------|------|
| 安全分級 | 讀取操作直接執行；寫入操作先摘要確認再執行 |
| 最小呼叫 | 優先用最簡單的方式完成；當複雜度升高時自然切換為更適合的方式 |
| 設定檔驅動 | 連線資訊從 `~/.redmine.json` 讀取，skill 本體不含任何憑證 |
| 意圖路由 | 根據使用者意圖查路由表，按需載入對應 reference，不要一次全讀。複合意圖時依序載入所有相關 reference |
| 連線韌性 | 設定檔不存在時引導使用者建立（指向 setup.md）；API Key 無效時提示檢查設定檔 |

### 載入策略

每次執行 API 操作時，SKILL.md 中直接包含最精簡的共用知識（認證 header 格式、基本錯誤碼），不需額外載入 api-common.md。api-common.md 作為進階參考（分頁、批次模式、呼叫範本），在需要時才載入。

### 互動範例

```
使用者：「幫我記今天在 issue #123 花了 2 小時做開發」
→ 載入 api-time-entries.md
→ 組 POST /time_entries.json 請求
→ 顯示確認摘要（issue #123, 2h, 開發, 今天）
→ 使用者確認 → 執行
```

### 路由表

| 使用者意圖 | 參考文件 |
|-----------|---------|
| 工時登打、查詢工時、工時統計 | `references/api-time-entries.md` |
| 建立/查詢/更新/指派議題、上傳附件 | `references/api-issues.md` |
| 專案、子專案、成員、活動類型 | `references/api-projects.md` |
| 認證、錯誤處理、分頁、共用模式 | `references/api-common.md` |

## References 設計

### api-common.md

- 認證方式：Header `X-Redmine-API-Key`
- 請求/回應格式：`Content-Type: application/json`
- 分頁：`?offset=0&limit=25`，回應含 `total_count`
- 錯誤處理：401（API Key）、404（資源不存在）、422（欄位錯誤，顯示 Redmine 回傳訊息）
- curl 與 Python 呼叫範本各一

### api-time-entries.md

- `POST /time_entries.json` — 登打工時（issue_id/project_id、hours、activity_id、comments、spent_on）
- `GET /time_entries.json` — 查詢工時（user_id、project_id、from/to 篩選）
- `PUT /time_entries/{id}.json` — 更新工時
- `DELETE /time_entries/{id}.json` — 刪除工時
- 自然語言情境範例

### api-issues.md

- `POST /issues.json` — 建立議題（project_id、subject、tracker_id、priority_id、assigned_to_id）
- `GET /issues.json` — 查詢議題（多種篩選條件、排序）
- `PUT /issues/{id}.json` — 更新狀態、指派、新增留言（notes）
- `DELETE /issues/{id}.json` — 刪除議題（高風險，需額外確認）
- 附件上傳：`POST /uploads.json` → 取得 token → 附加到 issue
- 自然語言情境範例

### api-projects.md

- `POST /projects.json` — 建立專案（identifier、name、parent_id）
- `GET /projects.json` — 查詢專案列表
- `POST /memberships.json` — 成員管理
- `GET /enumerations/time_entry_activities.json` — 活動類型
- Tracker / Status / Priority 枚舉查詢
- 自然語言情境範例

## setup.md 內容

1. Clone repo
2. 建立 symlink（macOS/Linux 與 Windows 指令）
3. 建立 `~/.redmine.json` 並填入 URL 和 API Key
4. 驗證步驟：請 AI「查詢我的 Redmine 帳號資訊」

## 何時更新這份 Skill

| 情境 | 更新什麼 |
|------|----------|
| 發現新的 API 端點需求 | 新增 reference + 路由表加一行 |
| 發現 API 行為與文件不符 | 更新對應 reference |
| 使用者反覆需要確認同類型寫入操作 | 考慮調整安全分級原則 |
| 發現新的互動模式（如常用的複合操作） | 更新 SKILL.md 互動範例或新增 reference |

## 擴充方式

新增 Redmine 操作時：
1. 建立 `references/api-{domain}.md`
2. 在 SKILL.md 路由表新增一行
3. 不需修改 SKILL.md 主體邏輯
