# Comprehensive Developer Guide: Building & Maintaining Schemas in FastAPI with SQLAlchemy 2.x + Pydantic v2/v3

This guide follows a **progressive, layered approach** that mirrors real-world FastAPI project architecture:

```
API Endpoint (FastAPI) 
    → Schemas (Pydantic v2/v3) 
        → Service Layer (business logic) 
            → DAO / Repository Layer (SQLAlchemy 2.x) 
                → ORM Models
```

We assume **SQLAlchemy 2.0+** (declarative style with `DeclarativeBase`) and **Pydantic v2+** (the current standard in 2025).

## 1. Base Schema Design

### 1.1 Shared Base Classes & Inheritance

```python
# schemas/base.py
from pydantic import BaseModel
from datetime import datetime
from typing import Annotated
import uuid

class TimestampMixin(BaseModel):
    created_at: Annotated[datetime, "Creation timestamp"]
    updated_at: Annotated[datetime, "Last update timestamp"]

class UUIDMixin(BaseModel):
    id: Annotated[uuid.UUID, "Primary key"]

# Optional: Config for all schemas
class SchemaBase(BaseModel):
    model_config = {
        "from_attributes": True,          # replaces orm_mode=True
        "extra": "forbid",                 # strict by default
        "populate_by_name": True,         # allow alias/by_name
    }
```

### 1.2 ORM Models vs Pydantic Schemas – Key Differences

| Feature                     | SQLAlchemy ORM Model                  | Pydantic Schema                         |
|-----------------------------|---------------------------------------|------------------------------------------|
| Purpose                     | Persistence & DB mapping              | API contracts (input/output)             |
| Validation                  | Limited (constraints at DB level)     | Rich runtime validation                  |
| Serialization               | Manual or via `__dict__`              | Automatic `.model_dump()` / `.model_dump_json()` |
| `from_attributes` / `orm_mode` | N/A                                 | Required to construct from ORM instances |
| Relationships               | Eager/lazy loading, backrefs          | Flattened or nested schemas only         |
| Defaults                    | DB-level or Python default            | Python-level + validation                |

**Rule of thumb**: **Never expose ORM models directly in responses** (security risk: lazy-loads, private attrs).

### 1.3 Naming & Modularity Best Practices

```
schemas/
├── __init__.py
├── base.py               # mixins & base classes
├── user/
│   ├── __init__.py
│   ├── create.py
│   ├── update.py
│   ├── response.py
│   └── enums.py
├── product/
│   └── ...               # same structure per domain entity
└── common/
    ├── pagination.py
    └── responses.py
```

- Prefix schemas: `UserCreate`, `UserUpdate`, `UserResponse`
- One file per schema type when entity is complex
- Use `tags` and `prefix` in routers to keep URLs consistent

## 2. CRUD Schemas

```python
# schemas/user/create.py
from pydantic import EmailStr, Field
from ..base import SchemaBase
from ..common.enums import UserRole

class UserCreate(SchemaBase):
    email: EmailStr
    password: str = Field(..., min_length=8, max_length=72)
    full_name: str | None = None
    role: UserRole = UserRole.USER

# schemas/user/update.py
from typing import Annotated
from pydantic import Field

class UserUpdate(SchemaBase):
    email: Annotated[EmailStr | None, "New email"] = None
    full_name: Annotated[str | None, Field(default=None)] = None
    role: Annotated[UserRole | None, "Change role"] = None

    # Enable partial updates
    model_config = {"extra": "ignore"}  # allow unknown fields to be dropped
```

```python
# schemas/user/response.py
from uuid import UUID
from datetime import datetime
from ..base import SchemaBase, TimestampMixin, UUIDMixin

class UserResponse(UUIDMixin, TimestampMixin, SchemaBase):
    email: str
    full_name: str | None
    role: UserRole
    is_active: bool
```

Delete usually returns minimal info:

```python
class UserDeleteResponse(SchemaBase):
    detail: str = "User deleted successfully"
    id: UUID
```

## 3. Response Models & Envelopes

Standard envelope (highly recommended for consistency):

```python
# schemas/common/responses.py
from typing import Generic, TypeVar, Any
from pydantic import BaseModel, Field
from pydantic.generics import GenericModel

T = TypeVar("T")

class APIResponse(GenericModel, Generic[T]):
    success: bool = True
    data: T | None = None
    error: str | None = None
    meta: dict[str, Any] | None = None

class PaginatedResponse(APIResponse[T]):
    meta: dict[str, Any] = Field(..., example={"total": 150, "page": 2, "size": 20})
```

Usage in endpoint:

```python
@router.get("/", response_model=PaginatedResponse[list[UserResponse]])
def list_users(...):
    return PaginatedResponse(data=users, meta=pagination_meta)
```

## 4. Validations

### 4.1 Field-level

```python
from pydantic import field_validator, Field
import re

class UserCreate(...):
    username: str = Field(..., pattern=r"^[a-zA-Z0-9_]{3,30}$")

    @field_validator("password")
    @classmethod
    def password_strength(cls, v: str) -> str:
        if not any(c.isupper() for c in v):
            raise ValueError("Password must contain uppercase")
        return v
```

### 4.2 Cross-field / Root Validators (Pydantic v2+)

```python
from pydantic import model_validator

class UserCreate(...):
    @model_validator(mode="after")
    def check_passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError("Passwords do not match")
        return self
```

### 4.3 Nullables & Defaults

```python
description: str | None = Field(default=None, max_length=500)
# vs
description: str = Field(default="", max_length=500)  # prefer explicit empty string if needed
```

## 5. Pydantic v2 → v3 Migration Notes (2025 Context)

| v2                              | v3 (2024+)                              | Migration Tip                              |
|---------------------------------|------------------------------------------|--------------------------------------------|
| `orm_mode=True`                 | `model_config["from_attributes"] = True` | Global or per-model                        |
| `allow_population_by_field_name`| `populate_by_name=True`                  | Same meaning                               |
| `@validator`                    | `@field_validator` + `mode="before/after"` | Different signature                        |
| `@root_validator`               | `@model_validator(mode="after")`         | Use `mode="after"` for most cases          |
| `.dict()`                       | `.model_dump()` / `.model_dump_json()`   | New methods                                |
| `Config.arbitrary_types_allowed`| Still exists, but prefer `Annotated`    |                                            |

**Important**: FastAPI 0.104+ fully supports Pydantic v3. Upgrade with:

```bash
pip install "pydantic>=2.9" "fastapi[standard]"
```

## 6. API Calling Examples

### Create User (curl)

```bash
curl -X POST http://localhost:8000/users/ \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@example.com",
    "password": "Secure123!",
    "full_name": "John Doe"
  }'
```

Response:

```json
{
  "success": true,
  "data": {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "email": "john@example.com",
    "full_name": "John Doe",
    "role": "USER",
    "created_at": "2025-11-18T10:00:00Z",
    "updated_at": "2025-11-18T10:00:00Z"
  }
}
```

### Partial Update (PATCH)

```bash
curl -X PATCH http://localhost:8000/users/123 \
  -H "Content-Type: application/json" \
  -d '{"full_name": "Johnny Doe"}'
```

### Error Example

```json
{
  "success": false,
  "error": "ValidationError",
  "detail": [
    {
      "loc": ["body", "password"],
      "msg": "Password must contain uppercase",
      "type": "value_error"
    }
  ]
}
```

## 7. Data Mapping & Edge Cases

### Safe Mapping from ORM → Schema

```python
user_schema = UserResponse.from_orm(db_user)           # deprecated
user_schema = UserResponse.model_validate(db_user)     # v2/v3 way
# or
user_schema = UserResponse(**db_user.__dict__)          # simple cases
```

### Handling Binary, JSON, Dates

```python
# models.py
from sqlalchemy.dialects.postgresql import BYTEA, JSONB
avatar: Mapped[bytes | None] = mapped_column(BYTEA)

# schema
avatar: str | None = Field(None, description="Base64 encoded avatar")
```

Date/time best practice:

```python
from datetime import datetime
from zoneinfo import ZoneInfo

class TimestampMixin(BaseModel):
    created_at: Annotated[datetime, Field(default_factory=lambda: datetime.now(ZoneInfo("UTC")))]
```

Always store UTC, serialize with `Z` suffix.

## 8. Service Layer

```python
# services/user.py
from typing import Sequence
from dao.user import UserDAO
from schemas.user import UserCreate, UserUpdate, UserResponse

class UserService:
    def __init__(self, dao: UserDAO):
        self.dao = dao

    async def create_user(self, data: UserCreate) -> UserResponse:
        # business logic: hash password, check uniqueness, etc.
        hashed = bcrypt.hash(data.password)
        db_user = await self.dao.create({**data.model_dump(), "password_hash": hashed})
        return UserResponse.model_validate(db_user)

    async def update_user(self, user_id: UUID, data: UserUpdate) -> UserResponse:
        update_data = data.model_dump(exclude_unset=True)  # ← crucial for partial
        db_user = await self.dao.update(user_id, update_data)
        return UserResponse.model_validate(db_user)
```

## 9. DAO / Repository Layer (SQLAlchemy 2.0 Style)

```python
# dao/base.py
from sqlalchemy.ext.asyncio import AsyncSession

class BaseDAO:
    def __init__(self, session: AsyncSession, model):
        self.session = session
        self.model = model

    async def get_by_id(self, id: UUID):
        return await self.session.get(self.model, id)

# dao/user.py
from sqlalchemy import select, update, delete
from sqlalchemy.orm import selectinload

class UserDAO(BaseDAO):
    async def filter_active(self, page: int, size: int):
        stmt = (
            select(User)
            .options(selectinload(User.profile))
            .where(User.is_active.is_(True))
            .order_by(User.created_at.desc())
            .limit(size)
            .offset((page - 1) * size)
        )
        result = await self.session.execute(stmt)
        return result.scalars().all()
```

## 10. Putting It All Together – End-to-End Example

```python
# api/v1/users.py
@router.post("/", response_model=APIResponse[UserResponse], status_code=201)
async def create_user(
    payload: UserCreate,
    service: UserService = Depends()
):
    user = await service.create_user(payload)
    return APIResponse(data=user)

@router.patch("/{user_id}", response_model=APIResponse[UserResponse])
async def update_user(
    user_id: UUID,
    payload: UserUpdate,
    service: UserService = Depends()
):
    user = await service.update_user(user_id, payload)
    return APIResponse(data=user)
```

### Final Checklist When Adding a New Entity

- [ ] Create SQLAlchemy model with proper relationships & indexes
- [ ] Create `Create`, `Update`, `Response` schemas (inherit mixins)
- [ ] Add validators & enums
- [ ] Create DAO with common query methods
- [ ] Create Service with business logic
- [ ] Add router with consistent response envelope
- [ ] Write integration tests for all CRUD + validation errors
- [ ] Update OpenAPI docs tags & descriptions

Following this structure yields clean, maintainable, and scalable FastAPI codebases that survive years of iteration.

Happy coding! 🚀