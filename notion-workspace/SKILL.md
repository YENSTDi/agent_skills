---
name: notion-workspace
description: |
  Ryan 的「❇️ 分享工作區」Notion 操作技能。
  
  觸發情境：
  (1) 查看/搜尋「分享工作區」內容
  (2) 新增/編輯「工作事項」或「例行事項」
  (3) 查詢 API 文件、專案狀態、服務項目
  (4) 任何提到 Notion、工作事項、例行事項、分享工作區的請求
---

# 分享工作區 Skill

Ryan 的 1111 人力銀行「❇️ 分享工作區」操作指南。

## 工作區結構

```
❇️ 分享工作區 (Wiki Database)
│   URL: https://www.notion.so/2a41cc48bc5a806a84d6f1e053849e36
│   Data Source: collection://2a41cc48-bc5a-8034-8006-000b451acaea
│
├── 工作事項 (子 Database) - 追蹤 API 服務與專案
│   URL: https://www.notion.so/2a41cc48bc5a80c58520da921c76171b
│   Data Source: collection://2a41cc48-bc5a-8095-b0a0-000b7bd50d72
│   TODO: https://www.notion.so/2ca1cc48bc5a80ff97e3e577fbdd0eb5
│
├── 例行事項 (子 Database) - 追蹤例行工作
│   URL: https://www.notion.so/2a41cc48bc5a80aea607ea757c2c2bb3
│   Data Source: collection://2a41cc48-bc5a-802b-bf70-000b8fd61c68
│
├── MCP Server (子 Database) - MCP 伺服器開發追蹤
│   URL: https://www.notion.so/2c41cc48bc5a809194b5d3c959e4db1d
│   Data Source: collection://2c41cc48-bc5a-80bb-a4a2-000b74d8594d
│   TODO: https://www.notion.so/2ca1cc48bc5a809e8cc1f52f68d3cce1
│
└── Data Pipeline (子 Database) - 資料管線文件
    URL: https://www.notion.so/2ca1cc48bc5a8031a313c7c50af1fb97
    Data Source: collection://2ca1cc48-bc5a-800a-b8bf-000bc5bdfc99
    TODO: https://www.notion.so/2ca1cc48bc5a80069274d35b84032fbd
```

## Database Schema

### 工作事項
| 欄位 | 類型 | 選項 |
|------|------|------|
| Name | title | - |
| 使用對象 | multi_select | 求職, 求才, 外部-商用 |
| AWS Resource | multi_select | ALB, API Gateway, Lambda, ECS |
| CICD | select | True, False |
| Code | text | 程式碼連結 |
| API Docs | text | API 文件連結 |
| 使用中 | checkbox | - |

### 例行事項
| 欄位 | 類型 |
|------|------|
| Name | title |

### MCP Server
| 欄位 | 類型 |
|------|------|
| Name | title |

### Data Pipeline
| 欄位 | 類型 |
|------|------|
| Name | title |
| Code | url |

#### 現有頁面
| 名稱 | 頁面 ID | 說明 |
|------|---------|------|
| 求職者履歷資料 | 2ca1cc48bc5a80deaa9de449cbb302bd | MSSQL → DynamoDB (1111TalentResume) |
| 求職者求職歷史紀錄 | 2ca1cc48bc5a810b80e6c30e4cc8b1e9 | MSSQL → DynamoDB (1111TalentHistory) |

## 快速操作

### 搜尋
```python
# 搜尋整個分享工作區
notion-search(page_url="2a41cc48bc5a806a84d6f1e053849e36", query="關鍵字")

# 搜尋工作事項
notion-search(data_source_url="collection://2a41cc48-bc5a-8095-b0a0-000b7bd50d72", query="關鍵字")

# 搜尋例行事項
notion-search(data_source_url="collection://2a41cc48-bc5a-802b-bf70-000b8fd61c68", query="關鍵字")

# 搜尋 MCP Server
notion-search(data_source_url="collection://2c41cc48-bc5a-80bb-a4a2-000b74d8594d", query="關鍵字")

# 搜尋 Data Pipeline
notion-search(data_source_url="collection://2ca1cc48-bc5a-800a-b8bf-000bc5bdfc99", query="關鍵字")
```

### 新增工作事項
```python
notion-create-pages(
  parent={"data_source_id": "2a41cc48-bc5a-8095-b0a0-000b7bd50d72"},
  pages=[{
    "properties": {
      "Name": "服務名稱",
      "使用對象": "[\"求職\", \"求才\"]",
      "AWS Resource": "[\"Lambda\", \"ECS\"]",
      "CICD": "True",
      "Code": "https://github.com/...",
      "API Docs": "https://docs...",
      "使用中": "__YES__"
    },
    "content": "# 說明\n詳細內容..."
  }]
)
```

### 新增例行事項
```python
notion-create-pages(
  parent={"data_source_id": "2a41cc48-bc5a-802b-bf70-000b8fd61c68"},
  pages=[{"properties": {"Name": "事項名稱"}, "content": "詳細說明"}]
)
```

### 新增 Data Pipeline
```python
notion-create-pages(
  parent={"data_source_id": "2ca1cc48-bc5a-800a-b8bf-000bc5bdfc99"},
  pages=[{
    "properties": {
      "Name": "管線名稱",
      "Code": "https://github.com/..."
    },
    "content": "## 概述\n說明...\n\n---\n\n## 資料來源\n<table>...</table>"
  }]
)
```

### 更新頁面
```python
# 更新屬性
notion-update-page(data={
  "page_id": "頁面ID",
  "command": "update_properties",
  "properties": {"CICD": "True", "使用中": "__YES__"}
})

# 更新內容
notion-update-page(data={
  "page_id": "頁面ID",
  "command": "replace_content",
  "new_str": "新內容"
})
```

## TODO 頁面

各 Database 的 TODO 頁面，用於追蹤待辦事項：

| Database | TODO 頁面 ID | URL |
|----------|-------------|-----|
| 工作事項 | 2ca1cc48bc5a80ff97e3e577fbdd0eb5 | https://www.notion.so/2ca1cc48bc5a80ff97e3e577fbdd0eb5 |
| MCP Server | 2ca1cc48bc5a809e8cc1f52f68d3cce1 | https://www.notion.so/2ca1cc48bc5a809e8cc1f52f68d3cce1 |
| Data Pipeline | 2ca1cc48bc5a80069274d35b84032fbd | https://www.notion.so/2ca1cc48bc5a80069274d35b84032fbd |

### 讀取 TODO
```python
# 讀取 MCP Server TODO
notion-fetch(id="2ca1cc48bc5a809e8cc1f52f68d3cce1")

# 讀取工作事項 TODO
notion-fetch(id="2ca1cc48bc5a80ff97e3e577fbdd0eb5")

# 讀取 Data Pipeline TODO
notion-fetch(id="2ca1cc48bc5a80069274d35b84032fbd")
```

## 注意事項

- Checkbox: `"__YES__"` / `"__NO__"`
- Multi-select: JSON array 字串 `"[\"選項1\", \"選項2\"]"`
- 「分享工作區」本身是 Wiki，屬性無法透過 API 更新
- 更新前先 `notion-fetch` 讀取現有內容

## Data Pipeline 頁面格式

參考「求職者求職歷史紀錄」頁面格式：
```markdown
## 概述
說明...
- **目標表**: `TableName` (region)
- **執行位置**: 地端主機 IP
- **排程**: 說明

---

## 資料來源
<table header-row="true">...</table>

---

## DynamoDB 結構
```javascript
Table: TableName
├── PK: ...
└── SK: ...
```

---

## 查詢模式
```python
# 範例查詢
```

---

## 使用方式
```bash
# 全量/增量同步指令
```

---

## 特性
- 特性 1
- 特性 2

---

## 資料保留
- 說明...
```
