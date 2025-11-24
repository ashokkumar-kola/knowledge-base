
# 2. CRUD Schemas

The official FastAPI tutorial (both the original SQLAlchemy version and the newer SQLModel version) uses **exactly three schemas per entity**:

| Operation | Schema name (official) | Purpose & Key Characteristics |
| --------- | ---------------------- | ----------------------------- |
| Create    | `UserCreate`           | Input only. All fields required except auto-generated ones (no `id`, no timestamps). Used for `POST`. |
| Update    | `UserUpdate`           | Partial update. **All fields optional** (`| None = None`). FastAPI automatically treats unset fields as not-to-be-updated when you use `exclude_unset=True`. Used for `PUT`/`PATCH`. |
| Read (Get)| `UserResponse` / `User` (in tutorial) | Output only. Includes `id`, timestamps, computed fields, relationships if needed. Used as `response_model` for `GET`, `POST`, `PUT`. |
| Delete    | No dedicated schema in official tutorial | Usually just return `{"detail": "Deleted"}` or status 204. If you want a body, a tiny schema like `DeleteResponse(detail=str)` is fine. |

## Official Example (adapted from FastAPI SQLAlchemy tutorial)

```python
# schemas/user.py
from pydantic import BaseModel, EmailStr, ConfigDict
from datetime import datetime
from uuid import UUID
from typing import Annotated

class UserBase(BaseModel):
    email: EmailStr
    full_name: str | None = None

# CREATE – strict input
class UserCreate(UserBase):
    password: str

# UPDATE – everything optional for partial PATCH
class UserUpdate(UserBase):
    password: str | None = None

    model_config = ConfigDict(extra="ignore")  # recommended for PATCH

# RESPONSE – what the API actually returns
class UserResponse(UserBase):
    model_config = ConfigDict(from_attributes=True)
    id: UUID
    created_at: Annotated[datetime, "UTC creation time"]
    is_active: bool = True
```

## How FastAPI Handles Partial Updates (official way)

```python
@router.patch("/{user_id}", response_model=UserResponse)
def update_user(
    user_id: UUID,
    user_in: UserUpdate,          # ← all fields optional
    db: Session = Depends(get_db)
):
    update_data = user_in.model_dump(exclude_unset=True)  # ← crucial!
    db_user = crud.update_user(db, user_id, update_data)
    return UserResponse.model_validate(db_user)
```

`exclude_unset=True` is the **official recommended pattern** for partial updates (see FastAPI docs → Request Body → Partial updates).

## Delete Schema (minimal or none)

The official tutorial returns nothing (204 No Content) or a simple dict:

```python
@router.delete("/{user_id}", status_code=204)
def delete_user(user_id: UUID, db: Session = Depends(get_db)):
    crud.delete_user(db, user_id)
    return None
```

If you want a JSON body:

```python
class Message(BaseModel):
    detail: str = "User deleted successfully"

@router.delete("/{user_id}", response_model=Message)
...
return Message()
```

This covers everything the official FastAPI, Pydantic v2, and SQLAlchemy 2.x documentations explicitly say about base schema design and CRUD schemas. The next sections (response envelopes, advanced validation, etc.) can be added later if needed.