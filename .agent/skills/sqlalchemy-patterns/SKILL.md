---
name: SQLAlchemy Advanced Patterns
description: Patterns và techniques nâng cao khi làm việc với SQLAlchemy ORM trong FastAPI
---

# SQLAlchemy Advanced Patterns

Skill này cung cấp các patterns nâng cao khi làm việc với SQLAlchemy, đặc biệt trong context của FastAPI applications.

## 1. Async SQLAlchemy Setup

### Engine và Session Configuration

```python
from sqlalchemy.ext.asyncio import (
    create_async_engine,
    AsyncSession,
    async_sessionmaker
)
from sqlalchemy.orm import DeclarativeBase

DATABASE_URL = "postgresql+asyncpg://user:pass@localhost/db"

engine = create_async_engine(
    DATABASE_URL,
    echo=True,  # Log SQL queries
    pool_size=5,
    max_overflow=10,
    pool_pre_ping=True,  # Check connection health
)

async_session = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False,
)

class Base(DeclarativeBase):
    pass
```

### Dependency cho FastAPI

```python
from collections.abc import AsyncGenerator
from fastapi import Depends

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    async with async_session() as session:
        try:
            yield session
            await session.commit()
        except Exception:
            await session.rollback()
            raise
```

## 2. Model Design Patterns

### Base Model với Common Fields

```python
from datetime import datetime
from sqlalchemy import DateTime, func
from sqlalchemy.orm import Mapped, mapped_column

class TimestampMixin:
    created_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now()
    )
    updated_at: Mapped[datetime] = mapped_column(
        DateTime(timezone=True),
        server_default=func.now(),
        onupdate=func.now()
    )

class SoftDeleteMixin:
    deleted_at: Mapped[datetime | None] = mapped_column(
        DateTime(timezone=True),
        default=None
    )
    
    @property
    def is_deleted(self) -> bool:
        return self.deleted_at is not None

class User(Base, TimestampMixin, SoftDeleteMixin):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True, index=True)
    username: Mapped[str] = mapped_column(unique=True)
```

### Relationships

```python
from sqlalchemy import ForeignKey
from sqlalchemy.orm import relationship, Mapped, mapped_column

class User(Base):
    __tablename__ = "users"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    email: Mapped[str] = mapped_column(unique=True)
    
    # One-to-Many
    articles: Mapped[list["Article"]] = relationship(
        back_populates="author",
        lazy="selectin"  # Eager loading
    )
    
    # Many-to-Many
    followed_users: Mapped[list["User"]] = relationship(
        secondary="followers",
        primaryjoin="User.id == followers.c.follower_id",
        secondaryjoin="User.id == followers.c.following_id",
    )

class Article(Base):
    __tablename__ = "articles"
    
    id: Mapped[int] = mapped_column(primary_key=True)
    title: Mapped[str]
    author_id: Mapped[int] = mapped_column(ForeignKey("users.id"))
    
    author: Mapped["User"] = relationship(back_populates="articles")
    tags: Mapped[list["Tag"]] = relationship(
        secondary="article_tags",
        lazy="selectin"
    )
```

## 3. Repository Pattern

### Base Repository

```python
from typing import TypeVar, Generic
from sqlalchemy import select, update, delete
from sqlalchemy.ext.asyncio import AsyncSession

ModelType = TypeVar("ModelType", bound=Base)

class BaseRepository(Generic[ModelType]):
    def __init__(self, session: AsyncSession, model: type[ModelType]):
        self._session = session
        self._model = model
    
    async def get_by_id(self, id: int) -> ModelType | None:
        result = await self._session.execute(
            select(self._model).where(self._model.id == id)
        )
        return result.scalar_one_or_none()
    
    async def get_all(
        self,
        skip: int = 0,
        limit: int = 100
    ) -> list[ModelType]:
        result = await self._session.execute(
            select(self._model).offset(skip).limit(limit)
        )
        return list(result.scalars().all())
    
    async def create(self, **kwargs) -> ModelType:
        obj = self._model(**kwargs)
        self._session.add(obj)
        await self._session.flush()
        await self._session.refresh(obj)
        return obj
    
    async def update(self, id: int, **kwargs) -> ModelType | None:
        await self._session.execute(
            update(self._model)
            .where(self._model.id == id)
            .values(**kwargs)
        )
        return await self.get_by_id(id)
    
    async def delete(self, id: int) -> bool:
        result = await self._session.execute(
            delete(self._model).where(self._model.id == id)
        )
        return result.rowcount > 0
```

### Specific Repository

```python
from sqlalchemy import select, or_

class UserRepository(BaseRepository[User]):
    def __init__(self, session: AsyncSession):
        super().__init__(session, User)
    
    async def get_by_email(self, email: str) -> User | None:
        result = await self._session.execute(
            select(User).where(User.email == email)
        )
        return result.scalar_one_or_none()
    
    async def search(self, query: str) -> list[User]:
        result = await self._session.execute(
            select(User).where(
                or_(
                    User.email.ilike(f"%{query}%"),
                    User.username.ilike(f"%{query}%")
                )
            )
        )
        return list(result.scalars().all())
    
    async def get_with_articles(self, user_id: int) -> User | None:
        from sqlalchemy.orm import selectinload
        
        result = await self._session.execute(
            select(User)
            .options(selectinload(User.articles))
            .where(User.id == user_id)
        )
        return result.scalar_one_or_none()
```

## 4. Query Optimization

### Eager Loading Strategies

```python
from sqlalchemy.orm import selectinload, joinedload, subqueryload

# selectinload - Best cho collections (1 query thêm per relationship)
async def get_users_with_articles(db: AsyncSession):
    result = await db.execute(
        select(User).options(selectinload(User.articles))
    )
    return result.scalars().all()

# joinedload - Best cho single object relationships
async def get_articles_with_author(db: AsyncSession):
    result = await db.execute(
        select(Article).options(joinedload(Article.author))
    )
    return result.unique().scalars().all()

# Nested eager loading
async def get_articles_full(db: AsyncSession):
    result = await db.execute(
        select(Article).options(
            joinedload(Article.author),
            selectinload(Article.tags),
            selectinload(Article.comments).selectinload(Comment.author)
        )
    )
    return result.unique().scalars().all()
```

### Selecting Specific Columns

```python
from sqlalchemy import select

# Select specific columns
async def get_user_emails(db: AsyncSession) -> list[tuple[int, str]]:
    result = await db.execute(
        select(User.id, User.email).where(User.is_active == True)
    )
    return result.all()

# Using Row objects
async def get_user_summaries(db: AsyncSession):
    result = await db.execute(
        select(
            User.id,
            User.username,
            func.count(Article.id).label("article_count")
        )
        .join(Article, User.id == Article.author_id, isouter=True)
        .group_by(User.id)
    )
    return [
        {"id": row.id, "username": row.username, "articles": row.article_count}
        for row in result.all()
    ]
```

## 5. Complex Queries

### Subqueries

```python
from sqlalchemy import select, func

async def get_users_with_article_count(db: AsyncSession):
    # Subquery để đếm articles
    article_count = (
        select(
            Article.author_id,
            func.count(Article.id).label("count")
        )
        .group_by(Article.author_id)
        .subquery()
    )
    
    result = await db.execute(
        select(User, article_count.c.count)
        .outerjoin(article_count, User.id == article_count.c.author_id)
    )
    return result.all()
```

### CTEs (Common Table Expressions)

```python
from sqlalchemy import select, func, literal

async def get_follower_chain(db: AsyncSession, user_id: int, depth: int = 3):
    # Recursive CTE để get follower chain
    followers_cte = (
        select(
            Follower.following_id.label("user_id"),
            literal(1).label("depth")
        )
        .where(Follower.follower_id == user_id)
        .cte(name="followers", recursive=True)
    )
    
    followers_recursive = (
        select(
            Follower.following_id,
            (followers_cte.c.depth + 1).label("depth")
        )
        .join(followers_cte, Follower.follower_id == followers_cte.c.user_id)
        .where(followers_cte.c.depth < depth)
    )
    
    followers_cte = followers_cte.union_all(followers_recursive)
    
    result = await db.execute(
        select(User)
        .join(followers_cte, User.id == followers_cte.c.user_id)
        .distinct()
    )
    return result.scalars().all()
```

### Window Functions

```python
from sqlalchemy import select, func, over

async def get_articles_with_rank(db: AsyncSession):
    rank = func.row_number().over(
        partition_by=Article.author_id,
        order_by=Article.created_at.desc()
    ).label("rank")
    
    result = await db.execute(
        select(Article, rank)
    )
    return result.all()
```

## 6. Transactions

### Explicit Transaction Control

```python
from sqlalchemy.ext.asyncio import AsyncSession

async def transfer_balance(
    db: AsyncSession,
    from_user_id: int,
    to_user_id: int,
    amount: float
):
    async with db.begin():  # Start transaction
        from_user = await db.get(User, from_user_id, with_for_update=True)
        to_user = await db.get(User, to_user_id, with_for_update=True)
        
        if from_user.balance < amount:
            raise InsufficientBalanceError()
        
        from_user.balance -= amount
        to_user.balance += amount
        # Auto-commit khi exit context
```

### Savepoints

```python
async def complex_operation(db: AsyncSession):
    async with db.begin():
        # Main operation
        user = await create_user(db, user_data)
        
        async with db.begin_nested():  # Savepoint
            try:
                await create_profile(db, user.id, profile_data)
            except Exception:
                # Rollback chỉ profile, giữ user
                pass
        
        # Continue with user
        await send_welcome_email(user)
```

## 7. Bulk Operations

```python
from sqlalchemy import insert, update

async def bulk_create_users(db: AsyncSession, users_data: list[dict]):
    await db.execute(
        insert(User),
        users_data
    )

async def bulk_update_status(
    db: AsyncSession,
    user_ids: list[int],
    status: str
):
    await db.execute(
        update(User)
        .where(User.id.in_(user_ids))
        .values(status=status)
    )

# Upsert (INSERT ... ON CONFLICT)
from sqlalchemy.dialects.postgresql import insert as pg_insert

async def upsert_user(db: AsyncSession, user_data: dict):
    stmt = pg_insert(User).values(**user_data)
    stmt = stmt.on_conflict_do_update(
        index_elements=[User.email],
        set_={
            "username": stmt.excluded.username,
            "updated_at": func.now()
        }
    )
    await db.execute(stmt)
```

## 8. Events và Hooks

```python
from sqlalchemy import event
from sqlalchemy.orm import Session

@event.listens_for(User, "before_insert")
def user_before_insert(mapper, connection, target):
    target.username = target.username.lower()

@event.listens_for(User, "after_update")
def user_after_update(mapper, connection, target):
    # Log changes, send notifications, etc.
    logger.info(f"User {target.id} updated")

# Session events
@event.listens_for(Session, "after_commit")
def after_commit(session):
    # Clear caches, trigger events, etc.
    pass
```

## 9. Testing Patterns

```python
import pytest
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.pool import StaticPool

@pytest.fixture
async def db_engine():
    engine = create_async_engine(
        "sqlite+aiosqlite:///:memory:",
        poolclass=StaticPool,
    )
    async with engine.begin() as conn:
        await conn.run_sync(Base.metadata.create_all)
    yield engine
    await engine.dispose()

@pytest.fixture
async def db(db_engine):
    async with async_sessionmaker(db_engine)() as session:
        yield session
        await session.rollback()

@pytest.mark.asyncio
async def test_create_user(db: AsyncSession):
    repo = UserRepository(db)
    user = await repo.create(email="test@test.com", username="test")
    
    assert user.id is not None
    assert user.email == "test@test.com"
```
