# Redmine API Skill for Claude Code

讓 Claude Code 透過自然語言操作 Redmine REST API——工時登打、議題管理、專案建置、狀態查詢等。

## 功能

- **工時管理**：登打、查詢、更新、刪除工時記錄
- **議題管理**：建立、查詢、更新、指派議題，上傳附件
- **專案管理**：建立專案/子專案、成員管理、枚舉查詢
- **安全分級**：讀取操作直接執行，寫入操作需確認後才執行

## 安裝

### 1. Clone

```bash
git clone https://github.com/ccy/redmine-skills.git
```

### 2. 建立 Symlink

```bash
# macOS / Linux
ln -s /path/to/redmine-skills ~/.claude/skills/redmine

# Windows (管理員)
mklink /D %USERPROFILE%\.claude\skills\redmine \path\to\redmine-skills
```

### 3. 設定連線

建立 `~/.redmine.json`：

```json
{
  "url": "https://your-redmine-server.com",
  "api_key": "your-api-key-here"
}
```

API Key 取得方式：Redmine → 我的帳戶 → API 存取金鑰

### 4. 驗證

跟 Claude 說「查詢我的 Redmine 帳號資訊」，應回傳你的使用者資料。

## 架構

```
SKILL.md                        ← 核心 skill（原則 + 路由表）
setup.md                        ← 安裝指引
references/
├── api-common.md               ← 分頁、批次、錯誤處理、呼叫範本
├── api-time-entries.md         ← 工時 CRUD
├── api-issues.md               ← 議題 CRUD、附件上傳
└── api-projects.md             ← 專案、成員、枚舉查詢
```

採用「單一 SKILL.md + 路由表按需載入 reference」架構，AI 根據使用者意圖精準載入對應文件，不浪費 context。

## 擴充

新增 Redmine 操作：

1. 建立 `references/api-{domain}.md`
2. 在 `SKILL.md` 路由表新增一行
3. 不需修改主體邏輯

## License

MIT
