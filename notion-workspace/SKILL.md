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
│
└── 例行事項 (子 Database) - 追蹤例行工作
    URL: https://www.notion.so/2a41cc48bc5a80aea607ea757c2c2bb3
    Data Source: collection://2a41cc48-bc5a-802b-bf70-000b8fd61c68
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

## 快速操作

### 搜尋
```python
# 搜尋整個分享工作區
notion-search(page_url="2a41cc48bc5a806a84d6f1e053849e36", query="關鍵字")

# 搜尋工作事項
notion-search(data_source_url="collection://2a41cc48-bc5a-8095-b0a0-000b7bd50d72", query="關鍵字")

# 搜尋例行事項  
notion-search(data_source_url="collection://2a41cc48-bc5a-802b-bf70-000b8fd61c68", query="關鍵字")
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

## 注意事項

- Checkbox: `"__YES__"` / `"__NO__"`
- Multi-select: JSON array 字串 `"[\"選項1\", \"選項2\"]"`
- 「分享工作區」本身是 Wiki，屬性無法透過 API 更新
- 更新前先 `notion-fetch` 讀取現有內容
