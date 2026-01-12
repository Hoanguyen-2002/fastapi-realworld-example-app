# Hướng dẫn tiếp cận FastAPI cho Java Developer

Tài liệu này giúp bạn (Java Backend Developer) hiểu dự án FastAPI này thông qua việc so sánh với các khái niệm quen thuộc trong Java/Spring Boot.

---

## 1. So sánh cấu trúc thư mục

| Java (Spring Boot)          | Python (FastAPI - Dự án này) | Mô tả                                    |
|-----------------------------|------------------------------|------------------------------------------|
| `src/main/java/controller/` | `app/api/routes/`            | Xử lý HTTP request (REST endpoints)      |
| `src/main/java/service/`    | `app/services/`              | Business logic                           |
| `src/main/java/repository/` | `app/db/repositories/`       | Data access layer (CRUD operations)      |
| `src/main/java/model/`      | `app/models/domain/`         | Entity / Domain objects                  |
| `src/main/java/dto/`        | `app/models/schemas/`        | Request/Response DTOs (Pydantic models)  |
| `src/main/resources/`       | `app/db/queries/sql/`        | SQL queries, config files                |
| `Application.java`          | `app/main.py`                | Entry point                              |
| `application.yml`           | `app/core/settings/`         | Configuration                            |
| `pom.xml` / `build.gradle`  | `pyproject.toml`             | Dependency management                    |

---

## 2. So sánh các khái niệm cốt lõi

### 2.1. Entry Point (Điểm khởi chạy)

**Java (Spring Boot)**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Python (FastAPI)** - File: `app/main.py`
```python
from fastapi import FastAPI

app = FastAPI()

# Chạy bằng: uvicorn app.main:app --reload
```

> **Điểm khác biệt:** Python không có hàm `main()` như Java. Thay vào đó, bạn chạy server bằng `uvicorn` (tương tự Tomcat embedded).

---

### 2.2. Controller / REST Endpoints

**Java (Spring Boot)**
```java
@RestController
@RequestMapping("/api/articles")
public class ArticleController {
    
    @Autowired
    private ArticleService articleService;
    
    @GetMapping("/{slug}")
    public ResponseEntity<ArticleResponse> getArticle(@PathVariable String slug) {
        Article article = articleService.findBySlug(slug);
        return ResponseEntity.ok(new ArticleResponse(article));
    }
    
    @PostMapping
    public ResponseEntity<ArticleResponse> createArticle(
            @RequestBody @Valid ArticleRequest request,
            @AuthenticationPrincipal User user) {
        Article article = articleService.create(request, user);
        return ResponseEntity.status(HttpStatus.CREATED).body(new ArticleResponse(article));
    }
}
```

**Python (FastAPI)** - File: `app/api/routes/articles/articles_resource.py`
```python
from fastapi import APIRouter, Depends, Body

router = APIRouter()  # Tương đương @RestController

@router.get("/{slug}", response_model=ArticleInResponse)  # Tương đương @GetMapping
async def retrieve_article_by_slug(
    article: Article = Depends(get_article_by_slug_from_path),  # Dependency Injection
) -> ArticleInResponse:
    return ArticleInResponse(article=ArticleForResponse.from_orm(article))

@router.post("", status_code=201, response_model=ArticleInResponse)  # Tương đương @PostMapping
async def create_new_article(
    article_create: ArticleInCreate = Body(..., embed=True, alias="article"),  # @RequestBody
    user: User = Depends(get_current_user_authorizer()),  # @AuthenticationPrincipal
    articles_repo: ArticlesRepository = Depends(get_repository(ArticlesRepository)),  # @Autowired
) -> ArticleInResponse:
    # ... logic
    pass
```

| Java Annotation         | FastAPI Equivalent                       |
|-------------------------|------------------------------------------|
| `@RestController`       | `APIRouter()`                            |
| `@GetMapping`           | `@router.get()`                          |
| `@PostMapping`          | `@router.post()`                         |
| `@PathVariable`         | Khai báo trong path: `/{slug}`           |
| `@RequestBody`          | `Body(...)`                              |
| `@Valid`                | Pydantic model tự động validate          |
| `@Autowired`            | `Depends()`                              |

---

### 2.3. Dependency Injection

**Java (Spring Boot)**
```java
@Service
public class ArticleService {
    
    @Autowired
    private ArticleRepository articleRepository;
    
    @Autowired
    private UserRepository userRepository;
}
```

**Python (FastAPI)** - Sử dụng `Depends()`
```python
# File: app/api/dependencies/database.py

def get_repository(repo_type: Type[BaseRepository]):
    async def _get_repo(
        conn: Connection = Depends(_get_connection_from_pool),
    ) -> BaseRepository:
        return repo_type(conn)  # Tạo instance repository với connection
    return _get_repo

# Sử dụng trong route:
async def create_article(
    articles_repo: ArticlesRepository = Depends(get_repository(ArticlesRepository)),
):
    pass
```

> **Quan trọng:** FastAPI dùng **Function-based DI** thay vì **Annotation-based DI** như Spring. Mỗi request sẽ gọi function trong `Depends()` để lấy dependency.

---

### 2.4. Repository Pattern (Data Access Layer)

**Java (Spring Data JPA)**
```java
@Repository
public interface ArticleRepository extends JpaRepository<Article, Long> {
    Optional<Article> findBySlug(String slug);
    
    @Query("SELECT a FROM Article a WHERE a.author.username = :username")
    List<Article> findByAuthorUsername(@Param("username") String username);
}
```

**Python (Dự án này)** - File: `app/db/repositories/articles.py`
```python
class ArticlesRepository(BaseRepository):
    def __init__(self, conn: Connection) -> None:
        super().__init__(conn)
        self._profiles_repo = ProfilesRepository(conn)
    
    async def get_article_by_slug(self, *, slug: str) -> Article:
        article_row = await queries.get_article_by_slug(self.connection, slug=slug)
        if article_row:
            return await self._get_article_from_db_record(article_row, ...)
        raise EntityDoesNotExist(f"article with slug {slug} does not exist")
    
    async def filter_articles(self, *, tag: str = None, author: str = None, ...) -> List[Article]:
        # Sử dụng Pypika để build dynamic query (tương tự Criteria API / QueryDSL)
        query = Query.from_(articles).select(...)
        if tag:
            query = query.join(articles_to_tags).on(...)
        # ...
```

| Java Concept                  | Python Equivalent (Dự án này)           |
|-------------------------------|------------------------------------------|
| `JpaRepository`               | `BaseRepository` (custom base class)     |
| `@Query` (JPQL/Native SQL)    | `aiosql` + raw SQL files                 |
| Criteria API / QueryDSL       | `Pypika` (query builder)                 |
| `EntityManager`               | `asyncpg.Connection`                     |
| `@Transactional`              | `async with self.connection.transaction()` |

---

### 2.5. DTO / Request-Response Objects

**Java**
```java
// Request DTO
public class ArticleRequest {
    @NotBlank
    private String title;
    
    @NotBlank
    private String body;
}

// Response DTO
public class ArticleResponse {
    private String slug;
    private String title;
    private AuthorResponse author;
}
```

**Python (Pydantic)** - File: `app/models/schemas/articles.py`
```python
from pydantic import BaseModel, Field

# Request DTO (Input)
class ArticleInCreate(BaseModel):
    title: str = Field(..., min_length=1)  # @NotBlank
    body: str = Field(..., min_length=1)
    description: str = Field("")
    tags: List[str] = Field(default_factory=list)

# Response DTO (Output)  
class ArticleForResponse(BaseModel):
    slug: str
    title: str
    author: Profile
    
    class Config:
        orm_mode = True  # Cho phép convert từ ORM object
```

> **Ưu điểm Pydantic:** Validation tự động, serialize/deserialize JSON, type hints rõ ràng.

---

### 2.6. Authentication / Security

**Java (Spring Security)**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) {
        http.authorizeRequests()
            .antMatchers("/api/articles").authenticated()
            .anyRequest().permitAll();
    }
}

// Trong Controller
@GetMapping("/me")
public User getCurrentUser(@AuthenticationPrincipal UserDetails user) {
    return userService.findByUsername(user.getUsername());
}
```

**Python (FastAPI)** - File: `app/api/dependencies/authentication.py`
```python
from fastapi.security import APIKeyHeader

# Tương tự Spring Security Filter
def get_current_user_authorizer(*, required: bool = True):
    return _get_current_user if required else _get_current_user_optional

async def _get_current_user(
    token: str = Depends(APIKeyHeader(name="Authorization")),
    users_repo: UsersRepository = Depends(get_repository(UsersRepository)),
) -> User:
    # Decode JWT token và lấy user
    username = get_username_from_token(token)
    return await users_repo.get_user_by_username(username=username)

# Sử dụng trong route (tương đương @AuthenticationPrincipal)
@router.get("/me")
async def get_current_user(
    user: User = Depends(get_current_user_authorizer()),
):
    return user
```

---

## 3. Async/Await - Điểm khác biệt quan trọng nhất

**Java (Blocking I/O - Truyền thống)**
```java
public Article getArticle(String slug) {
    // Thread bị block khi chờ database
    return articleRepository.findBySlug(slug).orElseThrow();
}
```

**Python (Non-blocking I/O - Async)**
```python
async def get_article(slug: str) -> Article:
    # Không block thread, có thể xử lý request khác trong lúc chờ DB
    return await articles_repo.get_article_by_slug(slug=slug)
```

> **Giải thích:**
> - `async def` = Hàm bất đồng bộ (coroutine)
> - `await` = Chờ kết quả mà không block thread
> - Tương tự **Project Reactor / WebFlux** trong Spring, nhưng cú pháp đơn giản hơn nhiều

---

## 4. Workflow làm việc hàng ngày

| Công việc               | Java/Maven/Gradle              | Python/Poetry                                |
|-------------------------|--------------------------------|----------------------------------------------|
| Cài dependencies        | `mvn install`                  | `poetry install`                             |
| Chạy app                | `mvn spring-boot:run`          | `uvicorn app.main:app --reload`              |
| Chạy test               | `mvn test`                     | `pytest`                                     |
| Thêm dependency mới     | Thêm vào `pom.xml`             | `poetry add <package>`                       |
| Format code             | IDE / Checkstyle               | `black .` (auto-formatter)                   |
| Database migration      | Flyway / Liquibase             | `alembic upgrade head`                       |

---

## 5. Tips cho Java Developer

1. **Đừng tìm `public static void main`** - Python không cần. File được chạy trực tiếp.

2. **Không có Interface/Impl pattern** - Python dùng duck typing. Bạn không cần `ArticleService` interface + `ArticleServiceImpl`.

3. **Decorators = Annotations** - `@router.get()` tương đương `@GetMapping`.

4. **Type hints là optional** - Nhưng dự án này dùng đầy đủ, giúp IDE hỗ trợ tốt hơn.

5. **Xem API docs tự động** - Chạy app rồi vào `http://localhost:8000/docs` (tương tự Swagger UI nhưng tự động generate).

---

## 6. Thứ tự đọc code đề xuất

1. **`app/main.py`** - Hiểu cách app khởi động
2. **`app/api/routes/api.py`** - Xem routing tổng quan
3. **`app/api/routes/articles/articles_resource.py`** - Xem 1 controller cụ thể
4. **`app/api/dependencies/authentication.py`** - Hiểu cách auth hoạt động
5. **`app/db/repositories/articles.py`** - Xem cách truy cập DB
6. **`app/models/schemas/articles.py`** - Xem cách định nghĩa DTO

---

Chúc bạn học tốt! Nếu có thắc mắc về bất kỳ phần nào, cứ hỏi nhé.
