# 測試模式與最佳實踐

## 測試命名

```python
# 格式：test_<被測單元>_<情境>_<預期結果>
def test_calculate_match_score_with_exact_skills_returns_perfect_score():
    ...

def test_parse_resume_with_invalid_format_raises_error():
    ...

def test_job_search_with_empty_query_returns_all_jobs():
    ...
```

## AAA 模式 (Arrange-Act-Assert)

```python
def test_job_matcher_returns_top_matches():
    # Arrange - 準備測試資料
    candidate = Candidate(skills=["python", "ml"])
    jobs = [
        Job(id=1, required_skills=["python", "ml"]),
        Job(id=2, required_skills=["java"]),
    ]
    matcher = JobMatcher()

    # Act - 執行被測行為
    result = matcher.find_matches(candidate, jobs)

    # Assert - 驗證結果
    assert len(result) == 1
    assert result[0].job_id == 1
```

## Fixtures (pytest)

```python
import pytest
from datetime import datetime

@pytest.fixture
def sample_job() -> Job:
    """建立測試用職缺。"""
    return Job(
        id=1,
        title="Python Developer",
        company="Test Corp",
        created_at=datetime(2024, 1, 1),
    )

@pytest.fixture
def job_repository(db_session) -> JobRepository:
    """建立已注入測試資料庫的 Repository。"""
    return JobRepository(session=db_session)

def test_job_repository_finds_by_id(job_repository, sample_job):
    job_repository.save(sample_job)
    found = job_repository.find_by_id(1)
    assert found == sample_job
```

## 參數化測試

```python
import pytest

@pytest.mark.parametrize("input_salary,expected_level", [
    (30000, "junior"),
    (60000, "mid"),
    (100000, "senior"),
    (150000, "lead"),
])
def test_categorize_salary_level(input_salary, expected_level):
    result = categorize_salary(input_salary)
    assert result == expected_level

@pytest.mark.parametrize("invalid_input", [
    None,
    -1,
    "not a number",
    float("inf"),
])
def test_categorize_salary_with_invalid_input_raises(invalid_input):
    with pytest.raises(ValueError):
        categorize_salary(invalid_input)
```

## Mocking

```python
from unittest.mock import Mock, patch, AsyncMock

# Mock 外部服務
def test_job_notifier_sends_email(mocker):
    mock_email = mocker.patch("app.services.email.send_email")
    
    notifier = JobNotifier()
    notifier.notify_new_job(job_id=1, user_email="test@example.com")
    
    mock_email.assert_called_once_with(
        to="test@example.com",
        subject="New Job Alert",
        body=mocker.ANY,
    )

# Mock async 函式
@pytest.mark.asyncio
async def test_async_job_fetcher():
    mock_client = AsyncMock()
    mock_client.get.return_value = {"jobs": []}
    
    fetcher = JobFetcher(client=mock_client)
    result = await fetcher.fetch_jobs()
    
    assert result == []
```

## 測試覆蓋率邊界

```python
# 確保測試涵蓋邊界條件
class TestPagination:
    def test_first_page(self):
        result = paginate(items, page=1, size=10)
        assert result.page == 1
        assert result.has_previous is False

    def test_last_page(self):
        result = paginate(items, page=10, size=10)
        assert result.has_next is False

    def test_empty_result(self):
        result = paginate([], page=1, size=10)
        assert result.items == []
        assert result.total == 0

    def test_page_size_exceeds_total(self):
        result = paginate([1, 2, 3], page=1, size=100)
        assert len(result.items) == 3
```

## 測試組織結構

```
tests/
├── conftest.py              # 共用 fixtures
├── unit/                    # 單元測試
│   ├── test_models.py
│   ├── test_services.py
│   └── test_utils.py
├── integration/             # 整合測試
│   ├── test_api.py
│   └── test_database.py
└── e2e/                     # 端對端測試
    └── test_job_application_flow.py
```

## pytest.ini 設定

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_functions = test_*
addopts = -v --tb=short --strict-markers
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
```
