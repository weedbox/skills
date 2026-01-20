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

**兩種模組開發方法：**

| 方法 | 說明 | 適用場景 |
|------|------|----------|
| Method 1: Manual FX | 手動控制 FX 依賴注入 | 需要複雜 FX 註解或細粒度控制 |
| Method 2: Weedbox Generic | 使用 `weedbox.Module[P]` 泛型 (推薦) | 新模組、簡單依賴、減少樣板程式碼 |

## 安裝方式

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
│   └── SKILL.md                              # 專案設定技能
└── module-dev/
    ├── SKILL.md                              # 模組開發技能
    └── references/
        ├── METHOD1_MANUAL_FX.md              # 手動 FX 模組詳細說明
        └── METHOD2_WEEDBOX_GENERIC.md        # Weedbox 泛型模組詳細說明
```

## 相關專案

- [weedbox/weedbox](https://github.com/weedbox/weedbox) - Weedbox 基礎模組框架
- [weedbox/common-modules](https://github.com/weedbox/common-modules) - 常用可重用模組

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
