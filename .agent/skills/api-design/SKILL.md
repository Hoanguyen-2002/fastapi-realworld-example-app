---
name: REST API Design Principles
description: Nguyên tắc thiết kế REST API chuyên nghiệp theo chuẩn industry
---

# REST API Design Principles

## 1. URL Structure

```python
# ✅ Good - Nouns, plural, lowercase
GET    /api/v1/users              # List users
GET    /api/v1/users/123          # Get user
POST   /api/v1/users              # Create user
PUT    /api/v1/users/123          # Update user
DELETE /api/v1/users/123          # Delete user

# Nested resources
GET    /api/v1/users/123/articles

# ❌ Bad - Verbs, singular
GET    /api/v1/getUser/123
POST   /api/v1/createArticle
```

## 2. HTTP Methods & Status Codes

| Method | Use | Status |
|--------|-----|--------|
| GET | Read | 200 OK |
| POST | Create | 201 Created |
| PUT | Full update | 200 OK |
| PATCH | Partial update | 200 OK |
| DELETE | Remove | 204 No Content |

### Error Codes
- **400** Bad Request - Invalid input
- **401** Unauthorized - Not authenticated  
- **403** Forbidden - No permission
- **404** Not Found
- **409** Conflict - Duplicate
- **422** Validation Error
- **429** Rate Limited

## 3. Query Parameters

```python
@router.get("/users")
async def list_users(
    # Pagination
    page: int = Query(1, ge=1),
    page_size: int = Query(20, ge=1, le=100),
    # Filtering
    status: str | None = Query(None, regex="^(active|inactive)$"),
    # Sorting
    sort_by: str = Query("created_at"),
    order: str = Query("desc", regex="^(asc|desc)$"),
    # Search
    q: str | None = Query(None, min_length=2),
):
    pass
```

## 4. Response Structure

```python
class PaginatedResponse(BaseModel, Generic[T]):
    items: list[T]
    total: int
    page: int
    page_size: int
    
class ErrorResponse(BaseModel):
    success: bool = False
    error: str
    code: str | None = None
    details: dict | None = None
```

## 5. Authentication

```python
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security)
) -> User:
    token = credentials.credentials
    payload = decode_jwt(token)
    return await user_repo.get_by_id(payload["sub"])

# Role-based access
def require_role(*roles: Role):
    async def checker(user: User = Depends(get_current_user)):
        if user.role not in roles:
            raise HTTPException(403, "Not authorized")
        return user
    return checker
```

## 6. Versioning

```python
# URL Path (recommended)
v1_router = APIRouter(prefix="/api/v1")
v2_router = APIRouter(prefix="/api/v2")

app.include_router(v1_router)
app.include_router(v2_router)
```

## 7. Documentation

```python
@router.get(
    "/users/{user_id}",
    summary="Get user by ID",
    responses={
        200: {"description": "User found"},
        404: {"description": "User not found"},
    }
)
async def get_user(
    user_id: int = Path(..., description="User ID", ge=1)
):
    """Get a user by their unique identifier."""
    pass
```
