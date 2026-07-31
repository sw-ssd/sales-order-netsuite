## ADDED Requirements

### Requirement: 文章管理（CMS）
系統維護公告與內容文章。

#### Scenario: 文章 CRUD
- **WHEN** 管理員 POST/PATCH/DELETE `/api/v1/articles`
- **THEN** 系統建立/更新/刪除 Article 記錄（title, slug, content, category, tags 等）

#### Scenario: 條件查詢
- **WHEN** 使用者 GET `/api/v1/articles` 傳入篩選條件（platform, category, language 等）
- **THEN** 系統回傳符合條件的文章列表
