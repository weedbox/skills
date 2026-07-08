# Weedbox Skills

專為 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 設計的 Weedbox 框架開發技能集。

這些 skills 讓 Claude Code 能夠更有效地協助開發者使用 Weedbox 框架進行模組化應用程式開發。

## 什麼是 Skills？

Skills 是 Claude Code 的擴展功能，透過提供特定領域的知識和指引，讓 AI 助手能夠更專業地協助特定類型的開發任務。

## 可用的 Skills

### project-dev

協助建立和規劃 Weedbox 專案結構，包含：

- 專案結構設定
- 應用程式入口點 (main.go) 與 Cobra CLI
- 三階段模組載入 (modules.go)
- 設定檔配置

### module-dev

協助開發 Weedbox 框架模組，包含：

- 建立新模組
- 使用 Uber FX 進行依賴注入
- 設定管理 (Viper)
- 生命週期 hooks
- 資料模型設計
- 事件處理
- 建立模組專屬的 skills 文件

**三種模組開發方法：**

| 方法 | 說明 | 適用場景 |
|------|------|----------|
| Method 1: Manual FX | 手動控制 FX 依賴注入 | 需要複雜 FX 註解或細粒度控制 |
| Method 2: Weedbox Generic | 使用 `weedbox.Module[P]` 泛型 (推薦) | 新模組、簡單依賴、減少樣板程式碼 |
| Method 3: Interface Module | 使用 `fxmodule.InterfaceModule` 建立可抽換實作 | Connector 類型、需要可互換後端的模組 |

**Module Skills：**

每個模組可以擁有自己的 `.skills/` 目錄，存放開發文件：

```
pkg/mymodule/.skills/mymodule-development.md
pkg/mymodule_apis/.skills/mymodule-apis-development.md
```

### common-modules

`github.com/weedbox/common-modules` 的參考文件 — 可重用的 weedbox 基礎設施模組：

- 設定管理（Viper、TOML、環境變數）
- 結構化日誌與健康檢查端點
- HTTP 伺服器（Gin），支援 CORS 與靜態檔案/SPA
- 資料庫 connector（PostgreSQL、SQLite、GORM）
- NATS 訊息傳遞與內嵌 JetStream 伺服器
- Redis 快取、SMTP 郵件、排程器
- Swagger/OpenAPI 文件

### user-modules

`github.com/weedbox/user-modules` 的參考文件 — 可重用的使用者管理、認證與 RBAC 模組：

- 使用者 CRUD，搭配 bcrypt 密碼雜湊與 UUID v7 ID
- JWT 認證與 refresh token 輪替
- 以 privy 為基礎的角色權限控制 (RBAC)
- 可直接使用的使用者、認證、角色/資源管理 REST API 端點
- 可擴充的權限系統與合併 API
- 選用的全域 JWT 驗證 middleware

### workqueue-modules

`github.com/weedbox/workqueue-modules` 的參考文件 — 以單一 `WorkQueue` interface 為核心的任務佇列模組：

- 背景任務的入列與消費，支援多個 consumer 競爭
- 延遲/排程投遞、指數退避重試
- 死信佇列（列出/重新入列/刪除）
- 可互換的後端：memory、GORM（任意 dialect）、PostgreSQL 原生（SKIP LOCKED + LISTEN/NOTIFY）、NATS JetStream
- 供撰寫新後端使用的一致性測試套件

### crud-api-dev

協助建立完整的 CRUD API，包含：

- 業務邏輯層 (Manager) 實作
- 使用 Gin 框架的 HTTP API 層
- 請求綁定模式 (URI/Body/Query 分離)
- 回應結構與錯誤處理
- QueryHelper 分頁、搜尋、篩選
- GORM 資料庫模型與索引設計

## 安裝方式

### 使用 add-skill（推薦）

使用 [add-skill](https://github.com/vercel-labs/add-skill) 工具將 skills 安裝到各種 AI 程式助手：

```bash
# 安裝到當前專案（互動模式）
npx add-skill weedbox/skills

# 全域安裝給 Claude Code 使用
npx add-skill weedbox/skills -g -a claude-code

# 只安裝特定 skill
npx add-skill weedbox/skills --skill module-dev

# 列出可用的 skills
npx add-skill weedbox/skills --list
```

**選項說明：**

| 參數 | 說明 |
|------|------|
| `-g, --global` | 安裝到使用者家目錄而非專案目錄 |
| `-a, --agent <agents...>` | 指定目標 agent（如 claude-code、opencode） |
| `-s, --skill <skills...>` | 安裝指定的 skills |
| `-l, --list` | 列出可用 skills，不進行安裝 |
| `-y, --yes` | 跳過確認提示 |

### 手動安裝

將此專案加入 Claude Code 的技能庫：

```bash
# 進入 Claude Code 設定目錄
cd ~/.claude/skills

# 克隆此專案
git clone https://github.com/weedbox/skills.git weedbox
```

或是在專案中引用：

```bash
# 在專案根目錄建立 .claude/skills 目錄
mkdir -p .claude/skills

# 將 skill 加入專案
cd .claude/skills
git clone https://github.com/weedbox/skills.git weedbox
```

## 使用方式

安裝後，Claude Code 會在適當的情境下自動使用這些技能。例如：

- 當你詢問如何建立 Weedbox 模組時
- 當你需要設定依賴注入時
- 當你在處理模組生命週期相關程式碼時

你也可以直接在對話中提及：

```
幫我建立一個新的 Weedbox 模組來處理用戶認證
```

## 專案結構

```
skills/
├── SKILL.md                                  # 根技能索引
├── README.md
├── README.zh-TW.md
├── LICENSE
├── project-dev/
│   ├── SKILL.md                              # 專案設定技能
│   └── references/
│       ├── licenses.md                       # 授權範本 (Apache-2.0, MIT, Proprietary)
│       └── docker.md                         # Dockerfile 範本與容器化指南
├── module-dev/
│   ├── SKILL.md                              # 模組開發技能
│   └── references/
│       ├── METHOD1_MANUAL_FX.md              # 手動 FX 模組詳細說明
│       ├── METHOD2_WEEDBOX_GENERIC.md        # Weedbox 泛型模組詳細說明
│       ├── METHOD3_INTERFACE_MODULE.md       # Interface/connector 模組詳細說明
│       ├── DATABASE_MODELS.md                # GORM 模型與索引慣例
│       └── MODULE_SKILLS.md                  # Module skills 撰寫指南
├── crud-api-dev/
│   ├── SKILL.md                              # CRUD API 開發技能
│   └── references/
│       ├── LOGIC_LAYER.md                    # 業務邏輯層詳細說明
│       └── API_LAYER.md                      # HTTP API 層詳細說明
├── common-modules/
│   ├── SKILL.md                              # Common modules 參考技能
│   └── modules/                              # 各模組文件 (configs, logger, http_server,
│                                             #   database, nats, redis, mailer, scheduler, ...)
├── user-modules/
│   ├── SKILL.md                              # User modules 參考技能
│   └── modules/
│       ├── permissions.md                    # 權限定義與擴充 API
│       ├── rbac.md                           # 以 privy 為基礎的 RBAC manager
│       ├── user.md                           # 使用者 CRUD 與密碼管理
│       ├── auth.md                           # JWT 認證與 middleware
│       ├── user_apis.md                      # 使用者 REST API 端點
│       ├── auth_apis.md                      # 認證 REST API 端點
│       ├── role_apis.md                      # 角色/資源 REST API 端點
│       └── http_token_validator.md           # 全域 JWT 驗證 middleware
└── workqueue-modules/
    ├── SKILL.md                              # Workqueue modules 參考技能
    └── modules/
        ├── workqueue.md                      # 共用 WorkQueue interface 與選項
        ├── memory_workqueue.md               # In-process 後端
        ├── gorm_workqueue.md                 # 資料庫後端（任意 GORM dialect）
        ├── postgres_workqueue.md             # PostgreSQL 原生後端
        ├── nats_workqueue.md                 # NATS JetStream 後端
        └── workqueuetest.md                  # 一致性測試套件
```

## 模組專屬 Skills

Weedbox 專案可以在 `.skills/` 目錄中包含模組專屬的 skills：

```
pkg/
├── product/
│   └── .skills/
│       └── product-development.md            # Product 模組文件
├── product_apis/
│   └── .skills/
│       └── product-apis-development.md       # Product API 文件
└── ...
```

這些 skills 記錄模組專屬的細節，例如資料模型、manager 方法、API 端點與使用範例。

## 相關專案

- [weedbox/weedbox](https://github.com/weedbox/weedbox) - Weedbox 基礎模組框架
- [weedbox/common-modules](https://github.com/weedbox/common-modules) - 常用可重用模組
- [weedbox/user-modules](https://github.com/weedbox/user-modules) - 使用者管理、認證與 RBAC 模組
- [weedbox/workqueue-modules](https://github.com/weedbox/workqueue-modules) - 任務佇列模組，支援可互換後端

## 貢獻

歡迎提交新的 skills 或改進現有內容：

1. Fork 此專案
2. 建立你的 feature branch (`git checkout -b feature/new-skill`)
3. 提交變更 (`git commit -m 'Add new skill'`)
4. 推送到 branch (`git push origin feature/new-skill`)
5. 開啟 Pull Request

### 新增 Skill 指南

每個 skill 應該包含：

1. `SKILL.md` - 主要技能文件，包含概覽和快速參考
2. `references/` - 詳細的參考文件 (選用)

SKILL.md 的 frontmatter 格式：

```yaml
---
name: skill-name
description: 簡短描述這個 skill 的用途和適用場景
---
```

## 授權

本專案採用 Apache License 2.0 授權 - 詳見 [LICENSE](LICENSE) 檔案。
