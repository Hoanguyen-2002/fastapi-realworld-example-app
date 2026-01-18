---
name: Python Clean Code
description: Các nguyên tắc và techniques để viết Python code sạch, dễ maintain như senior developer
---

# Python Clean Code

Skill này cung cấp các nguyên tắc viết code sạch, dễ đọc và maintain cho Python developers.

## 1. Naming Conventions

### Variables và Functions
```python
# ✅ Good - Descriptive và lowercase với underscore
user_count = get_active_users()
is_authenticated = check_user_authentication(user)
calculate_total_price(items)

# ❌ Bad
uc = getActiveUsers()  # Camel case, abbreviation
x = check(u)           # Single letter, vague
```

### Classes
```python
# ✅ Good - PascalCase, noun-based
class UserRepository:
    pass

class OrderService:
    pass

# ❌ Bad
class userRepository:  # lowercase
class DoOrder:         # Verb-based
```

### Constants
```python
# ✅ Good - UPPERCASE với underscore
MAX_CONNECTIONS = 100
DATABASE_URL = "postgresql://..."
DEFAULT_PAGE_SIZE = 20
```

## 2. Function Design

### Single Responsibility Principle

```python
# ❌ Bad - Function làm quá nhiều việc
def process_user_registration(data: dict):
    # Validate
    if not data.get("email"):
        raise ValueError("Email required")
    # Create user
    user = User(**data)
    db.add(user)
    db.commit()
    # Send email
    send_email(user.email, "Welcome!")
    # Log
    logger.info(f"User {user.id} created")
    return user

# ✅ Good - Tách thành các functions nhỏ
def validate_user_data(data: dict) -> None:
    if not data.get("email"):
        raise ValueError("Email required")

def create_user(db: Session, data: dict) -> User:
    user = User(**data)
    db.add(user)
    db.commit()
    return user

def process_user_registration(db: Session, data: dict) -> User:
    validate_user_data(data)
    user = create_user(db, data)
    send_welcome_email.delay(user.email)  # Async task
    logger.info(f"User {user.id} created")
    return user
```

### Function Arguments

```python
# ❌ Bad - Quá nhiều arguments
def create_user(name, email, password, age, address, phone, role, department):
    pass

# ✅ Good - Sử dụng dataclass/Pydantic
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str
    password: str
    age: int | None = None
    address: str | None = None
    phone: str | None = None
    role: str = "user"
    department: str | None = None

def create_user(data: UserCreate) -> User:
    pass
```

## 3. Type Hints

### Basic Type Hints

```python
from typing import Optional
from collections.abc import Sequence

def get_user(user_id: int) -> User | None:
    """Return User or None if not found"""
    pass

def get_users(
    skip: int = 0,
    limit: int = 10,
    is_active: bool | None = None
) -> list[User]:
    pass

def process_items(items: Sequence[str]) -> dict[str, int]:
    return {item: len(item) for item in items}
```

### Generic Types

```python
from typing import TypeVar, Generic

T = TypeVar("T")

class Repository(Generic[T]):
    def get(self, id: int) -> T | None:
        raise NotImplementedError
    
    def create(self, obj: T) -> T:
        raise NotImplementedError

class UserRepository(Repository[User]):
    def get(self, id: int) -> User | None:
        return self.db.query(User).filter(User.id == id).first()
```

## 4. Error Handling

### Specific Exceptions

```python
# ❌ Bad - Catch all exceptions
try:
    result = process_data(data)
except Exception as e:
    print(f"Error: {e}")

# ✅ Good - Catch specific exceptions
try:
    result = process_data(data)
except ValueError as e:
    logger.warning(f"Invalid data: {e}")
    raise HTTPException(status_code=400, detail=str(e))
except ConnectionError as e:
    logger.error(f"Connection failed: {e}")
    raise HTTPException(status_code=503, detail="Service unavailable")
```

### Custom Exceptions

```python
class DomainException(Exception):
    """Base exception for domain errors"""
    pass

class UserNotFoundException(DomainException):
    def __init__(self, user_id: int):
        self.user_id = user_id
        super().__init__(f"User {user_id} not found")

class InsufficientBalanceException(DomainException):
    def __init__(self, required: float, available: float):
        self.required = required
        self.available = available
        super().__init__(f"Required {required}, but only {available} available")
```

## 5. Context Managers

```python
from contextlib import contextmanager, asynccontextmanager

# Synchronous context manager
@contextmanager
def get_db_session():
    session = SessionLocal()
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# Async context manager
@asynccontextmanager
async def get_async_session():
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise

# Usage
async def create_user(data: UserCreate):
    async with get_async_session() as session:
        user = User(**data.model_dump())
        session.add(user)
        return user
```

## 6. Dataclasses và Pydantic

### Dataclasses cho internal data

```python
from dataclasses import dataclass, field
from datetime import datetime

@dataclass
class UserStats:
    total_orders: int = 0
    total_spent: float = 0.0
    last_login: datetime | None = None
    tags: list[str] = field(default_factory=list)
    
    @property
    def average_order_value(self) -> float:
        if self.total_orders == 0:
            return 0.0
        return self.total_spent / self.total_orders
```

### Pydantic cho validation

```python
from pydantic import BaseModel, Field, field_validator, model_validator

class OrderCreate(BaseModel):
    product_id: int = Field(..., gt=0)
    quantity: int = Field(..., ge=1, le=100)
    discount_code: str | None = None
    
    @field_validator("discount_code")
    @classmethod
    def validate_discount_code(cls, v: str | None) -> str | None:
        if v is not None:
            return v.upper().strip()
        return v
    
    @model_validator(mode="after")
    def validate_order(self) -> "OrderCreate":
        if self.quantity > 10 and not self.discount_code:
            raise ValueError("Large orders require discount code")
        return self
```

## 7. Comprehensions và Generators

```python
# List comprehension - khi cần toàn bộ list
squares = [x**2 for x in range(10) if x % 2 == 0]

# Generator expression - khi xử lý từng item
total = sum(x**2 for x in range(1000000))

# Dict comprehension
user_emails = {user.id: user.email for user in users}

# Set comprehension
unique_domains = {email.split("@")[1] for email in emails}

# Generator function cho large datasets
def read_large_file(file_path: str):
    with open(file_path, "r") as f:
        for line in f:
            yield line.strip()

# Async generator
async def fetch_paginated_data(url: str):
    page = 1
    while True:
        response = await client.get(f"{url}?page={page}")
        data = response.json()
        if not data:
            break
        for item in data:
            yield item
        page += 1
```

## 8. SOLID Principles

### Dependency Inversion

```python
from abc import ABC, abstractmethod

# Abstract interface
class EmailSender(ABC):
    @abstractmethod
    async def send(self, to: str, subject: str, body: str) -> bool:
        pass

# Concrete implementations
class SMTPEmailSender(EmailSender):
    async def send(self, to: str, subject: str, body: str) -> bool:
        # SMTP implementation
        pass

class SendGridEmailSender(EmailSender):
    async def send(self, to: str, subject: str, body: str) -> bool:
        # SendGrid API implementation
        pass

# Service depends on abstraction
class NotificationService:
    def __init__(self, email_sender: EmailSender):
        self._email_sender = email_sender
    
    async def notify_user(self, user: User, message: str):
        await self._email_sender.send(
            to=user.email,
            subject="Notification",
            body=message
        )
```

## 9. Logging Best Practices

```python
import logging
import structlog

# Configure structured logging
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
)

logger = structlog.get_logger()

# Usage với context
async def process_order(order_id: int):
    log = logger.bind(order_id=order_id)
    log.info("Processing order started")
    try:
        result = await do_process(order_id)
        log.info("Order processed successfully", result=result)
    except Exception as e:
        log.error("Order processing failed", error=str(e))
        raise
```

## 10. Testing Patterns

```python
import pytest
from unittest.mock import AsyncMock, patch

# Fixtures
@pytest.fixture
def sample_user():
    return User(id=1, email="test@example.com", username="testuser")

@pytest.fixture
def mock_repository():
    repo = AsyncMock(spec=UserRepository)
    return repo

# Test with mocking
@pytest.mark.asyncio
async def test_get_user_service(mock_repository, sample_user):
    mock_repository.get_by_id.return_value = sample_user
    service = UserService(mock_repository)
    
    result = await service.get_user(1)
    
    assert result.id == 1
    mock_repository.get_by_id.assert_called_once_with(1)

# Parameterized tests
@pytest.mark.parametrize("email,expected", [
    ("valid@email.com", True),
    ("invalid", False),
    ("", False),
])
def test_email_validation(email: str, expected: bool):
    assert validate_email(email) == expected
```
