# FastAPI + SQLAlchemy + Pydantic: 2: Base Schema Design & CRUD Schemas  
*Based strictly on the official FastAPI, SQLAlchemy 2.x, and Pydantic v2 documentations (as of November 2025)*

## 1. Base Schema Design

### 1.1 How to Define Base Schemas (Shared Fields, Inheritance)

Pydantic v2 encourages **inheritance of `BaseModel`** and **configuration via `ConfigDict`** for shared behaviour.

```python
# schemas/base.py
from pydantic import BaseModel, ConfigDict
from datetime import datetime
from typing import Annotated
from uuid import UUID

class Timestamped(BaseModel):
    created_at: Annotated[datetime, "When the record was created"]
    updated_at: Annotated[datetime, "When the record was last updated"]

class Identified(BaseModel):
    id: UUID

# Global config applied to every schema that inherits from this
class APISchema(BaseModel):
    model_config = ConfigDict(
        from_attributes=True,      # ← official replacement for orm_mode=True
        extra="forbid",            # strict by default (recommended)
        populate_by_name=True,     # allow alias/by_name if you use aliases
    )
```

Usage:

```python
class UserResponse(Identified, Timestamped, APISchema):
    email: str
    name: str | None = None
```

This is the exact pattern shown in the **Pydantic v2 docs → Models → ConfigDict** and used throughout the **FastAPI SQL databases tutorial** (even though the tutorial still uses SQLModel, the principle is identical).

### 1.2 Difference Between ORM Models and Pydantic Schemas

| Aspect                        | SQLAlchemy ORM Model                                      | Pydantic Schema (BaseModel)                                      |
| ----------------------------- | --------------------------------------------------------- | ---------------------------------------------------------------- |
| Purpose                       | Define database tables & persistence                      | Define API contracts (request / response)                        |
| Lives in                      | `models.py`                                               | `schemas.py`                                                     |
| Validation                    | Database-level constraints only                           | Rich runtime validation, filtering, serialization                |
| Construction from ORM         | N/A                                                       | Requires `from_attributes=True` to do `Schema.model_validate(orm_instance)` |
| Exposure in API               | **Never return directly** (risk of lazy-loading, private attrs) | Safe to return – FastAPI uses `response_model` to convert automatically |
| Official recommendation       | SQLAlchemy 2.0 docs: use `DeclarativeBase` + `Mapped[...] = mapped_column()` | FastAPI docs: always use separate Pydantic models for input/output |

> **FastAPI docs (SQL Databases tutorial)** explicitly separate ORM models from Pydantic schemas and never expose ORM instances directly in responses.

### 1.3 Best Practices for Naming and Modularity (from official examples & community consensus)

| Practice                                   | Official / Recommended Pattern                                 |
| ------------------------------------------ | -------------------------------------------------------------- |
| File organisation                          | `schemas/user.py` or `schemas/user/create.py` etc. (per-entity) |
| Naming convention                          | `UserCreate`, `UserUpdate`, `UserPartialUpdate`, `UserResponse` |
| Input vs Output                            | Input schemas → no `id`, no timestamps; Output → include them  |
| Never expose ORM models directly           | Always use `response_model=UserResponse`                       |
| Use `exclude_unset=True` for partial updates | `data.model_dump(exclude_unset=True)` (official way)          |
| Group by domain                            | One folder per bounded context (`schemas/auth/`, `schemas/product/`) |

This mirrors the structure used in the official FastAPI “Bigger Applications” example and the widely-cited best-practice repos.
