---
name: python-clean-code
description: |
  Python 程式碼品質與風格指南，遵循 PEP 標準與 Clean Code 原則。
  觸發情境：
  (1) 撰寫任何 Python 程式碼
  (2) 程式碼審查或重構
  (3) 使用者要求遵循 PEP 8、Clean Code 或程式碼品質標準
  (4) 建立 Python 專案或模組
---

# Python Clean Code Skill

遵循 PEP 標準與 Clean Code 原則撰寫高品質 Python 程式碼。

## 核心原則

### 1. 命名規範 (PEP 8)

```python
# 變數與函式：snake_case
user_name = "Ryan"
def calculate_salary(base_amount: float) -> float: ...

# 類別：PascalCase
class JobRecommendationEngine: ...

# 常數：UPPER_SNAKE_CASE
MAX_RETRY_COUNT = 3
DEFAULT_TIMEOUT_SECONDS = 30

# 私有屬性：單底線前綴
class Service:
    def __init__(self):
        self._internal_cache = {}
    
    def _helper_method(self): ...

# 避免的命名
# ❌ l, O, I (易混淆)
# ❌ data, info, temp (無意義)
# ❌ process_data() (太模糊)
# ✅ parse_job_description(), validate_resume()
```

### 2. 函式設計

```python
# ✅ 好：單一職責、描述性名稱
def extract_skills_from_resume(resume_text: str) -> list[str]:
    """從履歷文本中提取技能列表。"""
    ...

# ❌ 差：做太多事
def process_resume(resume): ...  # 解析？驗證？儲存？

# 參數數量：理想 0-3 個，超過考慮用 dataclass
@dataclass
class JobSearchCriteria:
    keywords: list[str]
    location: str
    salary_min: int | None = None
    salary_max: int | None = None
    remote_only: bool = False

def search_jobs(criteria: JobSearchCriteria) -> list[Job]: ...
```

### 3. Type Hints (PEP 484/585)

```python
from typing import TypeAlias
from collections.abc import Callable, Iterator

# 基本型別（Python 3.10+）
def greet(name: str) -> str: ...
def get_scores() -> list[int]: ...
def get_user_map() -> dict[str, User]: ...

# Optional 用 | None
def find_job(job_id: int) -> Job | None: ...

# 複雜型別用 TypeAlias
JobId: TypeAlias = int
SkillVector: TypeAlias = list[float]
RecommendationCallback: TypeAlias = Callable[[Job], float]
```

### 4. Docstring (PEP 257)

使用 Google Style docstring：

```python
def match_candidate_to_job(
    candidate_embedding: list[float],
    job_embeddings: list[list[float]],
    threshold: float = 0.8
) -> list[tuple[int, float]]:
    """計算候選人與職缺的匹配分數。

    Args:
        candidate_embedding: 候選人的向量表示。
        job_embeddings: 職缺向量列表。
        threshold: 最低匹配分數門檻。

    Returns:
        符合門檻的 (職缺索引, 分數) 列表，按分數降序排列。

    Raises:
        ValueError: 當向量維度不匹配時。

    Example:
        >>> match_candidate_to_job([0.1, 0.2], [[0.1, 0.2], [0.9, 0.9]])
        [(0, 1.0)]
    """
```

### 5. Import 順序 (PEP 8)

```python
# 1. 標準函式庫
import os
import sys
from datetime import datetime
from pathlib import Path

# 2. 第三方套件
import numpy as np
import pandas as pd
from fastapi import FastAPI
from pydantic import BaseModel

# 3. 本地模組
from app.models import Job, Candidate
from app.services.matching import MatchingService
```

### 6. 錯誤處理

```python
# ✅ 好：具體例外、有意義的訊息
class JobNotFoundError(Exception):
    """職缺不存在。"""
    def __init__(self, job_id: int):
        self.job_id = job_id
        super().__init__(f"Job with ID {job_id} not found")

def get_job(job_id: int) -> Job:
    job = db.query(Job).filter_by(id=job_id).first()
    if job is None:
        raise JobNotFoundError(job_id)
    return job

# ❌ 差：裸 except、吞掉錯誤
try:
    do_something()
except:  # 太廣泛
    pass   # 吞掉錯誤
```

### 7. 程式碼組織

```python
# 單一檔案結構順序
"""模組 docstring。"""

# 1. Future imports
from __future__ import annotations

# 2. Imports（依上述順序）

# 3. 模組層級常數
DEFAULT_PAGE_SIZE = 20

# 4. 類別定義

# 5. 函式定義

# 6. if __name__ == "__main__":
```

## 進階指南

詳細參考資料請見：
- `references/pep8-checklist.md` - PEP 8 完整檢查清單
- `references/code-smells.md` - 常見程式碼異味與重構方法
- `references/testing-patterns.md` - 測試模式與最佳實踐

## 工具整合

在 CI/CD 中整合以下工具：

```yaml
# pyproject.toml
[tool.ruff]
line-length = 88
select = ["E", "F", "I", "N", "W", "UP", "B", "C4", "SIM"]

[tool.ruff.isort]
known-first-party = ["app"]

[tool.mypy]
python_version = "3.11"
strict = true
```

```bash
# 開發流程
ruff check . --fix  # Lint + 自動修復
ruff format .       # 格式化
mypy .              # 型別檢查
pytest              # 測試
```
