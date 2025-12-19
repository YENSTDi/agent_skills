# 程式碼異味 (Code Smells) 與重構

## 函式相關

### 過長函式 (Long Method)

```python
# ❌ 差：一個函式做太多事
def process_application(application):
    # 驗證（20 行）
    # 解析履歷（30 行）
    # 計算匹配分數（25 行）
    # 發送通知（15 行）
    # 更新資料庫（20 行）
    ...

# ✅ 好：拆分成小函式
def process_application(application: Application) -> ProcessingResult:
    validated = validate_application(application)
    resume_data = parse_resume(validated.resume)
    scores = calculate_match_scores(resume_data)
    notify_relevant_parties(application, scores)
    return save_processing_result(application, scores)
```

### 過多參數 (Long Parameter List)

```python
# ❌ 差
def create_job_posting(title, description, company, location, 
                       salary_min, salary_max, skills, benefits,
                       remote, contract_type, experience_years):
    ...

# ✅ 好：使用 dataclass 或 Pydantic
@dataclass
class JobPostingRequest:
    title: str
    description: str
    company: str
    location: str
    salary_range: tuple[int, int] | None = None
    skills: list[str] = field(default_factory=list)
    remote: bool = False

def create_job_posting(request: JobPostingRequest) -> JobPosting:
    ...
```

### 旗標參數 (Flag Argument)

```python
# ❌ 差：布林參數改變行為
def get_users(include_inactive=False):
    if include_inactive:
        return db.query(User).all()
    return db.query(User).filter(User.active == True).all()

# ✅ 好：分成兩個明確函式
def get_active_users() -> list[User]:
    return db.query(User).filter(User.active == True).all()

def get_all_users() -> list[User]:
    return db.query(User).all()
```

## 類別相關

### 過大類別 (Large Class)

```python
# ❌ 差：God Class
class UserManager:
    def create_user(self): ...
    def update_user(self): ...
    def delete_user(self): ...
    def send_email(self): ...
    def generate_report(self): ...
    def process_payment(self): ...
    def upload_avatar(self): ...

# ✅ 好：單一職責
class UserRepository:
    def create(self, user: User) -> User: ...
    def update(self, user: User) -> User: ...
    def delete(self, user_id: int) -> None: ...

class EmailService:
    def send(self, to: str, subject: str, body: str) -> None: ...

class PaymentService:
    def process(self, payment: Payment) -> PaymentResult: ...
```

### 資料類別 (Data Class) - 貧血模型

```python
# ❌ 差：只有資料沒有行為
class Job:
    def __init__(self):
        self.title = ""
        self.salary_min = 0
        self.salary_max = 0

def calculate_average_salary(job):
    return (job.salary_min + job.salary_max) / 2

# ✅ 好：行為與資料在一起
@dataclass
class Job:
    title: str
    salary_min: int
    salary_max: int

    @property
    def average_salary(self) -> float:
        return (self.salary_min + self.salary_max) / 2
    
    def is_in_budget(self, budget: int) -> bool:
        return self.salary_min <= budget
```

## 重複程式碼

### DRY (Don't Repeat Yourself)

```python
# ❌ 差：重複邏輯
def get_active_jobs():
    jobs = db.query(Job).filter(Job.status == "active").all()
    return [{"id": j.id, "title": j.title} for j in jobs]

def get_featured_jobs():
    jobs = db.query(Job).filter(Job.featured == True).all()
    return [{"id": j.id, "title": j.title} for j in jobs]

# ✅ 好：抽取共用邏輯
def _serialize_jobs(jobs: list[Job]) -> list[dict]:
    return [{"id": j.id, "title": j.title} for j in jobs]

def get_active_jobs() -> list[dict]:
    jobs = db.query(Job).filter(Job.status == "active").all()
    return _serialize_jobs(jobs)

def get_featured_jobs() -> list[dict]:
    jobs = db.query(Job).filter(Job.featured == True).all()
    return _serialize_jobs(jobs)
```

## 條件複雜度

### 巢狀條件 (Nested Conditionals)

```python
# ❌ 差：深層巢狀
def process_order(order):
    if order:
        if order.is_valid:
            if order.has_stock:
                if order.payment_confirmed:
                    return ship_order(order)
    return None

# ✅ 好：Guard Clauses
def process_order(order: Order | None) -> ShipmentResult | None:
    if order is None:
        return None
    if not order.is_valid:
        return None
    if not order.has_stock:
        return None
    if not order.payment_confirmed:
        return None
    return ship_order(order)

# ✅ 更好：Early return + 清晰條件
def process_order(order: Order | None) -> ShipmentResult | None:
    if not _is_order_ready_to_ship(order):
        return None
    return ship_order(order)

def _is_order_ready_to_ship(order: Order | None) -> bool:
    return (
        order is not None
        and order.is_valid
        and order.has_stock
        and order.payment_confirmed
    )
```

## 魔術數字/字串

```python
# ❌ 差
if user.login_attempts > 5:
    lock_account(user)

if job.status == "active":
    ...

# ✅ 好
MAX_LOGIN_ATTEMPTS = 5

class JobStatus(str, Enum):
    ACTIVE = "active"
    CLOSED = "closed"
    DRAFT = "draft"

if user.login_attempts > MAX_LOGIN_ATTEMPTS:
    lock_account(user)

if job.status == JobStatus.ACTIVE:
    ...
```
