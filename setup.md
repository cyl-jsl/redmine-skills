# Redmine Skill 安裝指引

## 1. Clone Repo

```bash
git clone <repo-url> /path/to/redmine-skills
```

## 2. 建立 Symlink

### macOS / Linux

```bash
ln -s /path/to/redmine-skills ~/.claude/skills/redmine
```

### Windows (以管理員身分執行)

```cmd
mklink /D %USERPROFILE%\.claude\skills\redmine \path\to\redmine-skills
```

## 3. 建立設定檔

建立 `~/.redmine.json`：

```json
{
  "url": "https://your-redmine-server.com",
  "api_key": "your-api-key-here"
}
```

### API Key 取得方式

1. 登入 Redmine
2. 點選右上角「我的帳戶」
3. 在頁面右側找到「API 存取金鑰」
4. 點選「顯示」或「重新產生」取得 API Key

## 4. 驗證

跟 AI 說：

> 查詢我的 Redmine 帳號資訊

應回傳你的使用者名稱、信箱等資料。若出現 401 錯誤，請檢查 `~/.redmine.json` 中的 `api_key` 是否正確。
