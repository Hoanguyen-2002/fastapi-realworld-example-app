---
name: FastAPI Best Practices
description: Hướng dẫn các best practices khi phát triển ứng dụng với FastAPI framework
---

# FastAPI Best Practices

Skill này cung cấp các best practices và patterns quan trọng khi làm việc với FastAPI.

## 1. Project Structure

Sử dụng cấu trúc thư mục clean architecture:

```
app/
├── api/
│   └── routes/           # Endpoints/Controllers
├── core/
│   ├── config.py         # Configuration settings
│   └── security.py       # Authentication/Authorization
├── db/
│   ├── repositories/     # Data access layer
│   └── models/           # SQLAlchemy models
├── schemas/              # Pydantic schemas
├── services/             # Business logic layer
└── main.py               # Application entry point
```

## 2. Dependency Injection

### ✅ Đúng cách - Sử dụng `Depends` cho DI

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession

async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session

@router.get("/users/{user_id}")
async def get_user(
    user_id: int,
    db: AsyncSession = Depends(get_db),
    current_user: User = Depends(get_current_user)
):
    return await user_service.get_user(db, user_id)
```

### ❌ Sai cách - Hardcode dependencies

```python
# KHÔNG NÊN làm thế này
@router.get("/users/{user_id}")
async def get_user(user_id: int):
    db = create_session()  # Hardcoded
    return await user_service.get_user(db, user_id)
```

## 3. Async/Await Pattern

### ✅ Đúng cách - Async cho I/O operations

```python
from httpx import AsyncClient

async def fetch_external_data(url: str) -> dict:
    async with AsyncClient() as client:
        response = await client.get(url)
        return response.json()
```

### ✅ Đúng cách - Async database queries

```python
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user_by_id(db: AsyncSession, user_id: int) -> User | None:
    result = await db.execute(select(User).where(User.id == user_id))
    return result.scalar_one_or_none()
```

## 4. Pydantic Schemas Best Practices

### Base Schema Pattern

```python
from pydantic import BaseModel, Field, ConfigDict

class UserBase(BaseModel):
    """Base schema với common fields"""
    email: str = Field(..., description="User's email address")
    username: str = Field(..., min_length=3, max_length=50)

class UserCreate(UserBase):
    """Schema cho tạo user mới"""
    password: str = Field(..., min_length=8)

class UserRead(UserBase):
    """Schema cho response"""
    id: int
    is_active: bool = True
    
    model_config = ConfigDict(from_attributes=True)

class UserUpdate(BaseModel):
    """Schema cho update - tất cả fields optional"""
    email: str | None = None
    username: str | None = None
```

## 5. Error Handling

### Custom Exception Handler

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse

class AppException(Exception):
    def __init__(self, status_code: int, detail: str):
        self.status_code = status_code
        self.detail = detail

@app.exception_handler(AppException)
async def app_exception_handler(request: Request, exc: AppException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"detail": exc.detail, "type": "app_error"}
    )
```

### HTTPException với context

```python
from fastapi import HTTPException, status

def get_user_or_404(db: Session, user_id: int) -> User:
    user = db.query(User).filter(User.id == user_id).first()
    if not user:
        raise HTTPException(
            status_code=status.HTTP_404_NOT_FOUND,
            detail=f"User with id {user_id} not found"
        )
    return user
```

## 6. Response Models

### Sử dụng response_model để control output

```python
@router.get("/users/{user_id}", response_model=UserRead)
async def get_user(user_id: int, db: AsyncSession = Depends(get_db)):
    """Response tự động filter theo UserRead schema"""
    return await user_service.get_user(db, user_id)

@router.get("/users", response_model=list[UserRead])
async def list_users(
    skip: int = 0,
    limit: int = Query(default=10, le=100),
    db: AsyncSession = Depends(get_db)
):
    return await user_service.list_users(db, skip=skip, limit=limit)
```

## 7. Background Tasks

```python
from fastapi import BackgroundTasks

def send_email_notification(email: str, message: str):
    # Long-running task
    pass

@router.post("/users/")
async def create_user(
    user: UserCreate,
    background_tasks: BackgroundTasks,
    db: AsyncSession = Depends(get_db)
):
    new_user = await user_service.create_user(db, user)
    background_tasks.add_task(
        send_email_notification,
        email=new_user.email,
        message="Welcome!"
    )
    return new_user
```

## 8. Security Best Practices

### JWT Authentication

```python
from datetime import datetime, timedelta
from jose import JWTError, jwt
from passlib.context import CryptContext

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

def create_access_token(data: dict, expires_delta: timedelta | None = None) -> str:
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    # ... get user from db
```

## 9. Testing với pytest

```python
import pytest
from httpx import AsyncClient
from app.main import app

@pytest.fixture
async def client():
    async with AsyncClient(app=app, base_url="http://test") as ac:
        yield ac

@pytest.mark.asyncio
async def test_create_user(client: AsyncClient):
    response = await client.post(
        "/api/users/",
        json={"email": "test@example.com", "username": "testuser", "password": "password123"}
    )
    assert response.status_code == 201
    data = response.json()
    assert data["email"] == "test@example.com"
```

## 10. Performance Tips

1. **Sử dụng async** cho tất cả I/O operations
2. **Connection pooling** với SQLAlchemy async engine
3. **Caching** với Redis cho frequent queries
4. **Pagination** cho list endpoints
5. **Select specific columns** thay vì `SELECT *`

```python
# Efficient query với select columns
from sqlalchemy import select

stmt = select(User.id, User.email, User.username).where(User.is_active == True)
result = await db.execute(stmt)
```
