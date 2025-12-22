---
name: notion-knowledge
description: |
  Ryan 的「🧠 第二大腦 - 整合知識庫」Notion 操作技能。
  
  觸發情境：
  (1) 查看/搜尋知識庫內容
  (2) 新增/編輯知識頁面
  (3) 管理學習筆記、技術文件、專案知識
  (4) 任何提到「第二大腦」、「知識庫」、「Knowledge」的請求
  (5) 需要記錄或查找學習心得、技術筆記時
  (6) 查詢 Agent 可用的結構化知識
---

# 第二大腦 - 整合知識庫 Skill

Ryan 的「🧠 第二大腦 - 整合知識庫」操作指南。

## 知識流程

```
📝 Whiteboard (草稿/實踐)
      ↓ 整理
📚 知識區 (結構化知識)
      ↓ 標記 Agent可用=true
🤖 Agent 可查詢使用
```

## 知識庫結構

```
🧠 第二大腦 - 整合知識庫 (Wiki Database - 容器)
│   URL: https://www.notion.so/2751cc48bc5a80bca777c63db162e44d
│   Data Source: collection://2751cc48-bc5a-8094-8630-000b5c4422c9
│   ⚠️ Wiki 類型，屬性無法透過 API 更新
│
├── 📚 知識區 (Inline Database) - 結構化知識
│   URL: https://www.notion.so/2d11cc48bc5a808182a3efca5a085067
│   Data Source: collection://2d11cc48-bc5a-8053-a53f-000bf3b47d7e
│   用途：經過整理的可查詢知識
│
└── 📝 Whiteboard (Inline Database) - 草稿/實踐
    URL: https://www.notion.so/2d11cc48bc5a803587e9e50c1f10d86e
    Data Source: collection://2d11cc48-bc5a-804b-bc51-000b1b711256
    用途：隨手記錄、實驗筆記、待整理內容
```

## Database Schema

### 知識區
| 欄位 | 類型 | 說明 |
|------|------|------|
| Name | title | 知識標題 |
| 領域 | multi_select | AI/ML, NLP, ... |
| Ref | url | 參考來源連結 |
| 摘要 | rich_text | 知識重點摘要（供 Agent 快速理解） |
| 關鍵字 | multi_select | 搜尋用標籤 |
| Agent可用 | checkbox | 標記此知識已整理完成可被引用 |
| ID | text | 編號 |

### Whiteboard
| 欄位 | 類型 | 說明 |
|------|------|------|
| Name | title | 標題 |

## 快速操作

### 搜尋知識（優先搜尋 Agent 可用的知識）
```python
# 搜尋知識區
notion-search(
  data_source_url="collection://2d11cc48-bc5a-8053-a53f-000bf3b47d7e",
  query="搜尋關鍵字"
)

# 搜尋 Whiteboard
notion-search(
  data_source_url="collection://2d11cc48-bc5a-804b-bc51-000b1b711256",
  query="搜尋關鍵字"
)

# 全域搜尋整個第二大腦
notion-search(
  page_url="2751cc48bc5a80bca777c63db162e44d",
  query="搜尋關鍵字"
)
```

### 讀取頁面內容
```python
notion-fetch(id="頁面URL或ID")
```

### 新增知識頁面
```python
notion-create-pages(
  parent={"data_source_id": "2d11cc48-bc5a-8053-a53f-000bf3b47d7e"},
  pages=[{
    "properties": {
      "Name": "知識標題",
      "領域": "[\"AI/ML\"]",
      "Ref": "https://reference-url.com",
      "摘要": "這篇知識的重點摘要，讓 Agent 能快速理解內容",
      "關鍵字": "[\"keyword1\", \"keyword2\"]",
      "Agent可用": "__YES__"
    },
    "content": """# 主題

## 核心概念
重點說明...

## 詳細內容
...

## 參考資料
- [來源](URL)
"""
  }]
)
```

### 新增 Whiteboard 草稿
```python
notion-create-pages(
  parent={"data_source_id": "2d11cc48-bc5a-804b-bc51-000b1b711256"},
  pages=[{
    "properties": {"Name": "草稿標題"},
    "content": "隨手記錄的內容..."
  }]
)
```

### 更新頁面
```python
# 更新屬性（知識區可用，Whiteboard 可用）
notion-update-page(data={
  "page_id": "頁面ID",
  "command": "update_properties",
  "properties": {
    "摘要": "更新後的摘要",
    "Agent可用": "__YES__"
  }
})

# 更新內容
notion-update-page(data={
  "page_id": "頁面ID",
  "command": "replace_content",
  "new_str": "新內容"
})
```

## Agent 使用指南

### 查詢知識的策略
1. **優先搜尋知識區**：已整理的結構化知識
2. **參考摘要欄位**：快速理解知識要點
3. **檢查 Agent可用 標記**：確認知識已驗證可用
4. **需要細節時讀取完整頁面**：使用 `notion-fetch`

### 何時使用哪個區域
| 需求 | 區域 | 說明 |
|------|------|------|
| 查找已知概念 | 知識區 | 結構化、可靠的知識 |
| 記錄新想法 | Whiteboard | 快速記錄，之後整理 |
| 實驗筆記 | Whiteboard | 過程記錄 |
| 整理後的筆記 | 知識區 | 從 Whiteboard 整理過來 |

## 知識頁面模板

### 技術概念模板
```markdown
# [概念名稱]

## 一句話摘要
簡潔說明這是什麼。

## 核心概念
- 要點一
- 要點二

## 使用場景
何時該用這個技術/概念。

## 程式碼範例
\`\`\`python
# 範例
\`\`\`

## 常見陷阱
- 注意事項

## 參考資料
- [官方文件](URL)
```

### 問題解決模板
```markdown
# [問題描述]

## 問題現象
發生了什麼。

## 根本原因
為什麼會發生。

## 解決方案
如何解決。

## 預防措施
如何避免再次發生。
```

## 注意事項

- **最上層 Wiki**：屬性無法透過 API 更新，只能更新頁面內容
- **知識區/Whiteboard**：一般 Database，屬性和內容都可更新
- **Multi-select**：需使用 JSON array 字串格式 `"[\"Tag1\", \"Tag2\"]"`
- **Checkbox**：使用 `"__YES__"` / `"__NO__"`
- **更新前讀取**：使用 `notion-fetch` 讀取現有內容再更新

## 與其他 Skill 的關係

- `notion-workspace`：工作事項與例行事項管理（工作追蹤）
- `notion-knowledge`（本 Skill）：個人知識庫管理（學習成長）
