# PEP 8 檢查清單

## 縮排與空白

- [ ] 使用 4 個空格縮排（非 Tab）
- [ ] 續行對齊開始括號，或使用懸掛縮排
- [ ] 頂層定義之間空兩行
- [ ] 類別內方法之間空一行
- [ ] 函式內邏輯區塊可用空行分隔

```python
# ✅ 續行對齊
result = some_function(arg_one, arg_two,
                       arg_three, arg_four)

# ✅ 懸掛縮排
result = some_function(
    arg_one, arg_two,
    arg_three, arg_four,
)
```

## 行長度

- [ ] 程式碼行 ≤ 88 字元（Black 標準）或 79 字元（傳統 PEP 8）
- [ ] 註解與 docstring ≤ 72 字元
- [ ] URL 可超出長度限制

## 運算子空白

```python
# ✅ 正確
x = 1
y = x + 1
result = (a + b) * (c - d)
my_list[1:10]
func(arg=value)

# ❌ 錯誤
x=1
y = x+1
my_list[1 : 10]
func(arg = value)
```

## 逗號

```python
# ✅ 尾隨逗號（多行時）
ALLOWED_HOSTS = [
    "localhost",
    "127.0.0.1",
    "example.com",  # 尾隨逗號
]

# ✅ 單行不需要
ALLOWED_HOSTS = ["localhost", "127.0.0.1"]
```

## 註解

```python
# ✅ 行內註解：兩個空格後
x = x + 1  # Compensate for border

# ✅ 區塊註解：與程式碼同縮排
# This is a block comment
# that spans multiple lines.
def complex_function():
    ...
```

## 命名總覽

| 類型 | 風格 | 範例 |
|------|------|------|
| 模組 | snake_case | `job_matcher.py` |
| 套件 | snake_case | `recommendation_engine` |
| 類別 | PascalCase | `JobRecommender` |
| 例外 | PascalCase + Error | `InvalidResumeError` |
| 函式 | snake_case | `calculate_match_score` |
| 方法 | snake_case | `get_recommendations` |
| 變數 | snake_case | `user_profile` |
| 常數 | UPPER_SNAKE_CASE | `MAX_RESULTS` |
| 型別變數 | PascalCase | `T`, `JobType` |

## 比較

```python
# ✅ 正確
if x is None:
if x is not None:
if not x:
if isinstance(obj, MyClass):

# ❌ 錯誤
if x == None:
if x != None:
if x == False:
if type(obj) == MyClass:
```

## 字串

```python
# ✅ 一致使用單引號或雙引號
name = "Ryan"
query = "SELECT * FROM jobs"

# ✅ f-string（Python 3.6+）
message = f"Hello, {name}!"

# ❌ 避免 % 或 .format()（除非必要）
message = "Hello, %s!" % name
```
