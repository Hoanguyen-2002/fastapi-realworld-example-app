# FastAPI RealWorld Example App - Tổng quan kiến trúc

## 🎯 Dự án này làm gì?

Đây là backend API cho một ứng dụng **Medium.com clone** (còn gọi là "Conduit"), implement theo spec của [RealWorld](https://github.com/gothinkster/realworld). Dự án cung cấp các tính năng:

- **Authentication**: Đăng ký, đăng nhập với JWT token
- **User Profiles**: Xem profile, follow/unfollow người dùng
- **Articles**: CRUD bài viết, lọc theo tag/author/favorited
- **Comments**: Comment trên bài viết
- **Tags**: Gắn tag cho bài viết
- **Favorites**: Yêu thích bài viết
- **Feed**: Xem bài viết từ những người mình follow

---

## 🏗️ Kiến trúc tổng quan

```mermaid
flowchart TB
    subgraph Client
        FE[Frontend App]
    end
    
    subgraph "FastAPI Application"
        direction TB
        MAIN[main.py]
        
        subgraph "API Layer"
            AUTH[Authentication Router]
            USERS[Users Router]
            PROF[Profiles Router]
            ART[Articles Router]
            COMM[Comments Router]
            TAGS[Tags Router]
        end
        
        subgraph "Dependency Injection"
            DEP_DB[Database Deps]
            DEP_AUTH[Auth Deps]
        end
        
        subgraph "Repository Layer"
            REPO_USER[UsersRepository]
            REPO_ART[ArticlesRepository]
            REPO_PROF[ProfilesRepository]
            REPO_COMM[CommentsRepository]
            REPO_TAG[TagsRepository]
        end
        
        subgraph "Data Access"
            AIOSQL[aiosql Queries]
            SQL[.sql Files]
            PYPIKA[pypika Query Builder]
        end
        
        DB[(PostgreSQL)]
    end
    
    FE --> AUTH & USERS & PROF & ART & COMM & TAGS
    AUTH & USERS & PROF & ART & COMM & TAGS --> DEP_DB & DEP_AUTH
    DEP_DB --> REPO_USER & REPO_ART & REPO_PROF & REPO_COMM & REPO_TAG
    REPO_USER & REPO_ART & REPO_PROF & REPO_COMM & REPO_TAG --> AIOSQL & PYPIKA
    AIOSQL --> SQL
    SQL & PYPIKA --> DB
```

---

## 📁 Cấu trúc thư mục

```
app/
├── main.py                 # Entry point, khởi tạo FastAPI app
├── api/
│   ├── routes/             # API endpoints
│   │   ├── authentication.py   # POST /api/users/login, POST /api/users
│   │   ├── users.py            # GET/PUT /api/user
│   │   ├── profiles.py         # GET /api/profiles/{username}, follow/unfollow
│   │   ├── articles/           # CRUD articles, feed, favorites
│   │   ├── comments.py         # CRUD comments cho article
│   │   └── tags.py             # GET /api/tags
│   ├── dependencies/       # FastAPI Depends() factories
│   │   ├── database.py         # Inject Repository instances
│   │   └── authentication.py   # Inject current user from JWT
│   └── errors/             # Exception handlers
├── core/
│   ├── config.py           # Settings (database URL, secret key, etc.)
│   └── events.py           # Startup/shutdown events (DB pool)
├── db/
│   ├── repositories/       # Data access layer
│   │   ├── users.py
│   │   ├── articles.py
│   │   ├── comments.py
│   │   ├── profiles.py
│   │   └── tags.py
│   ├── queries/
│   │   ├── sql/            # Native SQL files (aiosql)
│   │   │   ├── users.sql
│   │   │   ├── articles.sql
│   │   │   ├── comments.sql
│   │   │   ├── profiles.sql
│   │   │   └── tags.sql
│   │   ├── queries.py      # aiosql loader
│   │   └── tables.py       # pypika table definitions
│   └── migrations/         # Alembic migrations
├── models/
│   ├── domain/             # Business entities (Pydantic)
│   │   ├── users.py        # User, UserInDB
│   │   ├── articles.py     # Article
│   │   ├── comments.py     # Comment
│   │   └── profiles.py     # Profile
│   └── schemas/            # Request/Response schemas (Pydantic)
├── services/               # Business logic (JWT, password hashing)
└── resources/              # String constants
```

---

## 📊 Database Schema

```mermaid
erDiagram
    users {
        int id PK
        text username UK
        text email UK
        text salt
        text hashed_password
        text bio
        text image
        timestamp created_at
        timestamp updated_at
    }
    
    articles {
        int id PK
        text slug UK
        text title
        text description
        text body
        int author_id FK
        timestamp created_at
        timestamp updated_at
    }
    
    tags {
        text tag PK
    }
    
    commentaries {
        int id PK
        text body
        int author_id FK
        int article_id FK
        timestamp created_at
        timestamp updated_at
    }
    
    articles_to_tags {
        int article_id FK
        text tag FK
    }
    
    favorites {
        int user_id FK
        int article_id FK
    }
    
    followers_to_followings {
        int follower_id FK
        int following_id FK
    }
    
    users ||--o{ articles : "writes"
    users ||--o{ commentaries : "writes"
    users ||--o{ favorites : "favorites"
    users ||--o{ followers_to_followings : "follows"
    articles ||--o{ commentaries : "has"
    articles ||--o{ articles_to_tags : "tagged"
    articles ||--o{ favorites : "favorited_by"
    tags ||--o{ articles_to_tags : "used_in"
```

---

## 🔄 Luồng hoạt động chính (High-Level)

### 1. Tổng quan luồng Request

```mermaid
flowchart LR
    subgraph "Client"
        U[👤 User]
    end
    
    subgraph "API Layer"
        R[🔀 Router]
    end
    
    subgraph "Business Logic"
        S[⚙️ Service]
        RP[📦 Repository]
    end
    
    subgraph "Data"
        DB[(🗄️ Database)]
    end
    
    U -->|HTTP Request| R
    R -->|Validate & Process| S
    S -->|Data Access| RP
    RP -->|SQL Query| DB
    DB -->|Data| RP
    RP -->|Domain Model| S
    S -->|Response| R
    R -->|HTTP Response| U
```

---

### 2. User Journey: Đăng ký → Viết bài → Tương tác

```mermaid
flowchart TB
    subgraph "🔐 Authentication"
        A1[Đăng ký tài khoản] --> A2[Đăng nhập]
        A2 --> A3[Nhận JWT Token]
    end
    
    subgraph "✍️ Content Creation"
        B1[Viết bài mới] --> B2[Gắn Tags]
        B2 --> B3[Publish Article]
    end
    
    subgraph "💬 Interaction"
        C1[Đọc bài viết]
        C2[Comment bài viết]
        C3[Favorite bài viết]
        C4[Follow tác giả]
    end
    
    subgraph "📰 Feed"
        D1[Xem Feed cá nhân]
        D2[Lọc theo Tag/Author]
    end
    
    A3 --> B1
    A3 --> C1
    B3 --> C1
    C1 --> C2 & C3 & C4
    C4 --> D1
    C1 --> D2
```

---

### 3. Luồng Authentication (Đăng ký / Đăng nhập)

```mermaid
flowchart LR
    subgraph "Đăng ký"
        R1[📝 Nhập thông tin] --> R2{Username/Email<br/>đã tồn tại?}
        R2 -->|Có| R3[❌ Lỗi 400]
        R2 -->|Không| R4[✅ Tạo User]
        R4 --> R5[🎫 Trả về Token]
    end
    
    subgraph "Đăng nhập"
        L1[📧 Nhập Email + Password] --> L2{Email<br/>tồn tại?}
        L2 -->|Không| L3[❌ Lỗi 400]
        L2 -->|Có| L4{Password<br/>đúng?}
        L4 -->|Sai| L3
        L4 -->|Đúng| L5[🎫 Trả về Token]
    end
```

---

### 4. Luồng Article (Tạo / Đọc / Sửa / Xóa)

```mermaid
flowchart TB
    subgraph "📖 READ"
        direction LR
        R1[GET /articles] --> R2[Lọc theo tag/author/favorited]
        R3[GET /articles/:slug] --> R4[Chi tiết bài viết]
        R5[GET /articles/feed] --> R6[Bài từ người đang follow]
    end
    
    subgraph "✏️ CREATE"
        direction LR
        C1[POST /articles] --> C2[Tạo slug từ title]
        C2 --> C3[Lưu article + tags]
    end
    
    subgraph "🔄 UPDATE"
        direction LR
        U1[PUT /articles/:slug] --> U2{Là tác giả?}
        U2 -->|Có| U3[Cập nhật nội dung]
        U2 -->|Không| U4[❌ 403 Forbidden]
    end
    
    subgraph "🗑️ DELETE"
        direction LR
        D1[DELETE /articles/:slug] --> D2{Là tác giả?}
        D2 -->|Có| D3[Xóa article]
        D2 -->|Không| D4[❌ 403 Forbidden]
    end
```

---

### 5. Luồng Follow & Feed

```mermaid
flowchart LR
    subgraph "👥 Follow System"
        F1[User A] -->|Follow| F2[User B]
        F1 -->|Follow| F3[User C]
        F1 -->|Follow| F4[User D]
    end
    
    subgraph "📰 Feed Generation"
        F2 -->|Viết bài| A1[Article 1]
        F3 -->|Viết bài| A2[Article 2]
        F4 -->|Viết bài| A3[Article 3]
        
        A1 & A2 & A3 --> FEED[🏠 Feed của User A]
    end
    
    FEED -->|Sắp xếp theo thời gian| RESULT[Danh sách bài viết]
```

---

### 6. Luồng Favorites & Comments

```mermaid
flowchart TB
    subgraph "❤️ Favorites"
        FA1[Xem bài viết] --> FA2{Đã favorite?}
        FA2 -->|Chưa| FA3[POST favorite → ❤️ +1]
        FA2 -->|Rồi| FA4[DELETE favorite → 💔 -1]
    end
    
    subgraph "💬 Comments"
        CO1[Xem bài viết] --> CO2[GET comments]
        CO2 --> CO3[Danh sách comments]
        CO1 --> CO4[POST comment → Thêm comment]
        CO4 --> CO5{Là tác giả comment?}
        CO5 -->|Có| CO6[DELETE → Xóa comment]
    end
```

---

### 7. Data Flow Overview

```mermaid
flowchart TB
    subgraph "👤 User Actions"
        UA1[Register/Login]
        UA2[Create Article]
        UA3[Follow User]
        UA4[Favorite Article]
        UA5[Add Comment]
    end
    
    subgraph "🗄️ Database Tables"
        T1[(users)]
        T2[(articles)]
        T3[(tags)]
        T4[(followers_to_followings)]
        T5[(favorites)]
        T6[(commentaries)]
        T7[(articles_to_tags)]
    end
    
    UA1 --> T1
    UA2 --> T2
    UA2 --> T3
    UA2 --> T7
    UA3 --> T4
    UA4 --> T5
    UA5 --> T6
    
    T1 -.->|author_id| T2
    T1 -.->|author_id| T6
    T2 -.->|article_id| T5
    T2 -.->|article_id| T6
    T2 -.->|article_id| T7
```

---

## 🔧 API Endpoints Summary

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|---------------|
| **Authentication** |
| POST | `/api/users` | Register new user | ❌ |
| POST | `/api/users/login` | Login user | ❌ |
| **User** |
| GET | `/api/user` | Get current user | ✅ |
| PUT | `/api/user` | Update current user | ✅ |
| **Profiles** |
| GET | `/api/profiles/{username}` | Get profile | ❌ (optional) |
| POST | `/api/profiles/{username}/follow` | Follow user | ✅ |
| DELETE | `/api/profiles/{username}/follow` | Unfollow user | ✅ |
| **Articles** |
| GET | `/api/articles` | List/Filter articles | ❌ (optional) |
| GET | `/api/articles/feed` | Get feed | ✅ |
| GET | `/api/articles/{slug}` | Get article | ❌ (optional) |
| POST | `/api/articles` | Create article | ✅ |
| PUT | `/api/articles/{slug}` | Update article | ✅ (author only) |
| DELETE | `/api/articles/{slug}` | Delete article | ✅ (author only) |
| POST | `/api/articles/{slug}/favorite` | Favorite article | ✅ |
| DELETE | `/api/articles/{slug}/favorite` | Unfavorite article | ✅ |
| **Comments** |
| GET | `/api/articles/{slug}/comments` | Get comments | ❌ (optional) |
| POST | `/api/articles/{slug}/comments` | Create comment | ✅ |
| DELETE | `/api/articles/{slug}/comments/{id}` | Delete comment | ✅ (author only) |
| **Tags** |
| GET | `/api/tags` | Get all tags | ❌ |

---

## 🗃️ Domain Models

### User & UserInDB

```python
class User(RWModel):
    username: str           # Unique username
    email: str              # Unique email
    bio: str = ""           # User bio
    image: Optional[str]    # Avatar URL

class UserInDB(User):
    id_: int                # Database ID
    salt: str               # Password salt
    hashed_password: str    # Hashed password
    created_at: datetime
    updated_at: datetime
```

### Article

```python
class Article(RWModel):
    id_: int
    slug: str               # URL-friendly identifier
    title: str
    description: str
    body: str
    tags: List[str]         # Tag names
    author: Profile         # Author profile
    favorited: bool         # Is favorited by current user?
    favorites_count: int    # Total favorites
    created_at: datetime
    updated_at: datetime
```

### Comment

```python
class Comment(RWModel):
    id_: int
    body: str
    author: Profile         # Comment author
    created_at: datetime
    updated_at: datetime
```

### Profile

```python
class Profile(RWModel):
    username: str
    bio: str
    image: Optional[str]
    following: bool         # Is current user following this profile?
```

---

## 🔗 Relationship Summary

| Relationship | Type | Description |
|-------------|------|-------------|
| User → Articles | One-to-Many | User viết nhiều bài |
| User → Comments | One-to-Many | User viết nhiều comment |
| User ↔ User (followers) | Many-to-Many | User follow nhiều user khác |
| User ↔ Article (favorites) | Many-to-Many | User favorite nhiều bài, bài có nhiều favorites |
| Article → Comments | One-to-Many | Bài có nhiều comment |
| Article ↔ Tags | Many-to-Many | Bài có nhiều tags, tag thuộc nhiều bài |

---

## 🔍 Native SQL Queries hiện tại

### [users.sql](file:///c:/Users/vieth/Documents/fastapi-realworld-example-app/app/db/queries/sql/users.sql)
- `get-user-by-email` - Tìm user theo email
- `get-user-by-username` - Tìm user theo username
- `create-new-user` - Tạo user mới
- `update-user-by-username` - Cập nhật user

### [articles.sql](file:///c:/Users/vieth/Documents/fastapi-realworld-example-app/app/db/queries/sql/articles.sql)
- `add-article-to-favorites` - Favorite bài viết
- `remove-article-from-favorites` - Unfavorite
- `is-article-in-favorites` - Check đã favorite chưa
- `get-favorites-count-for-article` - Đếm favorites
- `get-tags-for-article-by-slug` - Lấy tags của bài
- `get-article-by-slug` - Lấy bài theo slug
- `create-new-article` - Tạo bài mới
- `add-tags-to-article` - Gắn tags
- `update-article` - Cập nhật bài
- `delete-article` - Xóa bài
- `get-articles-for-feed` - Lấy bài từ người đang follow

### [comments.sql](file:///c:/Users/vieth/Documents/fastapi-realworld-example-app/app/db/queries/sql/comments.sql)
- `get-comments-for-article-by-slug` - Lấy comments
- `get-comment-by-id-and-slug` - Lấy 1 comment
- `create-new-comment` - Tạo comment
- `delete-comment-by-id` - Xóa comment

### [profiles.sql](file:///c:/Users/vieth/Documents/fastapi-realworld-example-app/app/db/queries/sql/profiles.sql)
- `is-user-following-for-another` - Check follow status
- `subscribe-user-to-another` - Follow
- `unsubscribe-user-from-another` - Unfollow

### [tags.sql](file:///c:/Users/vieth/Documents/fastapi-realworld-example-app/app/db/queries/sql/tags.sql)
- `get-all-tags` - Lấy tất cả tags
- `create-new-tags` - Tạo tags mới

---

## ⚙️ Data Access Pattern hiện tại

### 1. aiosql (Native SQL)
```python
# queries.py
queries = aiosql.from_path(Path(__file__).parent / "sql", "asyncpg")

# Repository usage
user_row = await queries.get_user_by_email(self.connection, email=email)
```

### 2. pypika (Query Builder)
Dùng cho dynamic queries như filter articles:
```python
query = Query.from_(articles).select(...)
if tag:
    query = query.join(articles_to_tags).on(...)
if author:
    query = query.join(users).on(...)
articles_rows = await self.connection.fetch(query.get_sql(), *params)
```

---

## 📝 Key Insights cho ORM Migration

1. **N+1 Query Problem**: Method `_get_article_from_db_record` gọi 4 queries riêng lẻ cho mỗi article (tags, favorites_count, is_favorited, author profile). SQLAlchemy ORM với eager loading sẽ tối ưu hơn.

2. **Many-to-Many operations**: Hiện tại dùng INSERT/DELETE trực tiếp vào junction tables. SQLAlchemy relationship collections sẽ đơn giản hơn.

3. **Subqueries**: Nhiều SQL queries dùng subquery để lookup ID từ username/slug. ORM sẽ handle relationships tự động.

4. **Dynamic filtering**: pypika Query builder sẽ được thay bằng SQLAlchemy Core `select().where().join()`.

5. **Transaction management**: Hiện tại dùng `connection.transaction()`. SQLAlchemy session sẽ handle commits/rollbacks.
