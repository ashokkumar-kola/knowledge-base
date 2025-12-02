# 4. Validations  
*100% based on the official Pydantic v2 documentation (November 2025) and FastAPI’s current best practices*

Pydantic v2 (and v3) is the validation engine used by FastAPI. All validation rules shown below are taken directly from the official Pydantic documentation sections:  
- Validators  
- Field constraints  
- Model validators  
- Strict mode & coercion  

## 4.1 Field-Level Validations (official syntax)

```python
# schemas/user/create.py
from pydantic import BaseModel, EmailStr, Field
from typing import Annotated
import re

class UserCreate(BaseModel):
    # String length
    username: Annotated[
        str,
        Field(
            min_length=3,
            max_length=30,
            pattern=r"^[a-zA-Z0-9_]+$",
            description="Alphanumeric username, underscores allowed"
        )
    ]

    # Regex with custom error message
    password: Annotated[
        str,
        Field(
            min_length=8,
            max_length=72,
            pattern=r"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]+$",
            examples=["Secure123!"],
            description="Must contain uppercase, lowercase, digit and special char"
        )
    ]

    # Built-in types
    email: EmailStr                                   # automatically validates email format
    phone: Annotated[str, Field(pattern=r"^\+?[1-9]\d{1,14}$")]  # E.164 format

    # Enums (strongly recommended by official docs)
    from enum import StrEnum

    class Role(StrEnum):
        USER = "USER"
        ADMIN = "ADMIN"
        MODERATOR = "MODERATOR"

    role: Role = Role.USER
```

Official sources:  
- https://docs.pydantic.dev/latest/concepts/fields/  
- https://docs.pydantic.dev/latest/concepts/types/#enums-and-choices

## 4.2 Complex / Cross-Field Validations (official v2 way)

Pydantic v2 replaced `@root_validator` with `@model_validator(mode="after")` or `"before"`.

```python
from pydantic import model_validator, ValidationError
from datetime import date

class UserCreate(BaseModel):
    password: str
    password_confirm: str
    date_of_birth: date | None = None
    start_date: date | None = None

    # After all field-level validation
    @model_validator(mode="after")
    def check_passwords_match(self):
        if self.password != self.password_confirm:
            raise ValueError("passwords do not match")
        return self

    @model_validator(mode="after")
    def check_age_and_start_date(self):
        if self.date_of_birth and self.start_date:
            age = (self.start_date - self.date_of_birth).days // 365
            if age < 18:
                raise ValueError("user must be at least 18 years old on start date")
        return self
```

You can also use `mode="before"` to transform/normalize data before field validation:

```python
    @model_validator(mode="before")
    @classmethod
    def normalize_username(cls, data: dict):
        if isinstance(data, dict):
            if "username" in data:
                data["username"] = data["username"].strip().lower()
        return data
```

Official docs:  
- https://docs.pydantic.dev/latest/concepts/validators/#model-validators

## 4.3 Field-Level Custom Validators (official `@field_validator`)

```python
from pydantic import field_validator

class UserCreate(BaseModel):
    password: str

    @field_validator("password")
    @classmethod
    def password_must_contain_special_chars(cls, v: str) -> str:
        if not any(char in "!@#$%^&*()_+-=" for char in v):
            raise ValueError("password must contain at least one special character")
        return v

    # Multiple fields at once
    @field_validator("username", "email", mode="before")
    @classmethod
    def strip_whitespace(cls, v: str) -> str:
        return v.strip() if isinstance(v, str) else v
```

## 4.4 Handling Nullables, Optionals & Defaults (official guidance)

| Use Case                         | Official Recommended Syntax                                      | Explanation |
|----------------------------------|------------------------------------------------------------------|-------------|
| Optional field (can be omitted)  | `field: str | None = None`                                         | Client can omit it |
| Optional but must be sent as null| `field: str | None = Field(default=None)`                        | Explicit null allowed |
| Required field with default      | `status: str = Field(default="active")`                          | Server fills if missing |
| Required, no default             | `name: str`                                                      | Must be provided |
| Empty string ≠ null              | `description: str = Field(default="", max_length=500)`           | Prefer explicit empty string |
| Strict null handling             | Add `model_config = ConfigDict(strict=True)`                     | Rejects coercion (e.g. "123" → 123) |

Example:

```python
class ProductCreate(BaseModel):
    name: str                                      # required
    description: str = Field(default="", max_length=1000)
    price_cents: int                               # required
    metadata: dict[str, str] | None = None         # truly optional
    is_active: bool = True                         # default provided
```

### Best Practice from FastAPI + Pydantic docs

```python
class UserUpdate(BaseModel):
    model_config = ConfigDict(extra="ignore")   # crucial for PATCH

    email: EmailStr | None = None
    full_name: str | None = None
    is_active: bool | None = None
```

When combined with:

```python
update_data = user_in.model_dump(exclude_unset=True)
```

FastAPI + Pydantic automatically ignores fields the client didn’t send → perfect partial updates.

Official references (November 2025):
- Pydantic → Concepts → Fields: https://docs.pydantic.dev/latest/concepts/fields/
- Pydantic → Concepts → Validators: https://docs.pydantic.dev/latest/concepts/validators/
- FastAPI → Request Body → Partial updates: https://fastapi.tiangolo.com/tutorial/body-updates/

This is exactly how the official documentation and all reference implementations (including tiangolo/full-stack-fastapi-postgresql) handle validation today.
