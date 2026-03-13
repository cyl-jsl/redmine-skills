# Redmine API Skill for Claude Code

讓 Claude Code 透過自然語言操作 Redmine REST API——工時登打、議題管理、專案建置、狀態查詢等。

## 功能

- **工時管理**：登打、查詢、更新、刪除工時記錄
- **議題管理**：建立、查詢、更新、指派議題，上傳附件
- **專案管理**：建立專案/子專案、成員管理、枚舉查詢
- **安全分級**：讀取操作直接執行，寫入操作需確認後才執行
- **零接觸憑證**：API Key 透過 CLI wrapper 內部處理，永不出現在對話中

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

設定檔案權限：

```bash
chmod 600 ~/.redmine.json
```

### 4. 驗證

跟 Claude 說「查詢我的 Redmine 帳號資訊」，應回傳你的使用者資料（API Key 自動遮蔽）。

## 架構

```
SKILL.md                        ← 核心 skill（原則 + 路由表 + 安全禁令）
setup.md                        ← 安裝指引
bin/
└── redmine-api                 ← CLI wrapper（認證封裝、回應淨化）
references/
├── api-common.md               ← 分頁、批次、錯誤處理
├── api-time-entries.md         ← 工時 CRUD
├── api-issues.md               ← 議題 CRUD、附件上傳
└── api-projects.md             ← 專案、成員、枚舉查詢
```

採用「單一 SKILL.md + 路由表按需載入 reference」架構，AI 根據使用者意圖精準載入對應文件，不浪費 context。

## 安全設計

- **零接觸憑證**：AI 不讀取設定檔、不組認證 header，一切透過 `bin/redmine-api` CLI wrapper
- **回應淨化**：自動遮蔽 `api_key`、`password` 等敏感欄位
- **權限檢查**：啟動時驗證 `~/.redmine.json` 檔案權限 ≤ 600
- **強制 HTTPS**：拒絕 HTTP 明文連線

## 擴充

新增 Redmine 操作：

1. 建立 `references/api-{domain}.md`
2. 在 `SKILL.md` 路由表新增一行
3. 不需修改主體邏輯

## License

MIT
