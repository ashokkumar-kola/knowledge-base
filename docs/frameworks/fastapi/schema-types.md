
# 📘 **4. Schema Types**

In a production-grade FastAPI application, schema design must be deliberate and consistent.
Schemas define how data flows between layers:

➡️ **Client Request** → **FastAPI** → **Pydantic Schema** → **Service Layer** → **DAO/ORM** → **DB** → **Response Schema** → **Client**

To keep the system maintainable, predictable, and secure, we divide schemas into **five categories**:

1. **Base Schema**
2. **Create Schema**
3. **Update Schema**
4. **Response Schema**
5. **Delete Response Schema**
6. **Error Response Schema**

Each schema type has strict rules and a clear purpose.

---

# 📘 **4.1 Base Schema**

### ✅ Purpose

The **Base Schema** defines fields shared across create, update, and response schemas.
It contains **common attributes**, but **never** any required fields.

### 🎯 What Base Schema *should* contain

* Common optional fields (e.g., `name`, `city`, `description`)
* Shared validators (email, phone, dates, etc.)
* Documentation rules (`Field(description="...")`)

### 🚫 What Base Schema *should NOT contain*

| Forbidden in Base Schema                    | Why                                |
| ------------------------------------------- | ---------------------------------- |
| IDs (UUID, int, etc.)                       | DB-generated or immutable          |
| Required fields                             | Create schema responsibility       |
| DB-only fields (`created_at`, `updated_at`) | ORM concern                        |
| Foreign keys                                | Response or ORM layer handles this |
| Internal flags                              | Should not be client-controlled    |

### 📄 Example Pattern

```python
class EventBase(BaseModel):
    name: Optional[str] = Field(default=None)
    location: Optional[str] = Field(default=None)
    email: Optional[str] = Field(default=None)

    @field_validator("email")
    def validate_email(cls, v):
        return validate_email_util(v)

    model_config = {
        "extra": "forbid"
    }
```

---

# 📘 **4.2 Create Schema (POST)**

### ✅ Purpose

Schema used when creating new records.
Contains **required fields** and supports **file uploads** or **form-data** patterns.

### 🎯 Responsibilities

* Enforce required fields.
* Validate all input.
* Accept multipart form-data via `Form()` and `UploadFile`.

### 🎒 Create Schema Must Include

* Required fields with `...` / no default
* Optional fields with defaults
* File upload fields
* Form parsing if needed

### 📄 Example (with JSON)

```python
class EventCreate(EventBase):
    name: str = Field(..., min_length=3)
    start_at: datetime = Field(...)
    end_at: datetime = Field(...)
```

### 📄 Example (with multipart form-data + file)

```python
class EventCreate(EventBase):
    name: str = Form(...)
    start_at: datetime = Form(...)
    end_at: datetime = Form(...)
    file: UploadFile | None = None
```

---

# 📘 **4.3 Update Schema (PATCH)**

### ✅ Purpose

Used for modifying an existing record.
**All fields must be optional** to support partial updates.

### 🎯 Responsibilities

* Allow partial updates.
* Normalize empty strings (`""`) → `None`, if needed.
* Enforce “PATCH rules”: no required fields.

### Strategies

#### **PATCH**

* Only update provided fields.

#### **PUT**

* Replace entire record (rarely used in modern APIs).
* Should require all fields → NOT common.

### 🔧 Optional: Automatic Empty String Normalization

Empty strings from UI should be treated as missing values:

```python
@model_validator(mode="before")
def normalize_empty(cls, data):
    return {k: (None if v == "" else v) for k, v in data.items()}
```

### 📄 Example

```python
class EventUpdate(EventBase):
    name: Optional[str] = None
    start_at: Optional[datetime] = None
    end_at: Optional[datetime] = None

    # All fields optional → safe for PATCH
```

---

# 📘 **4.4 Response Schema (GET)**

### ✅ Purpose

Defines what the API returns back to the client.
It must contain only **safe**, **non-internal**, **read-only** values.

### 🎯 Responsibilities

* Convert DB models → API-friendly format
* Convert IDs → string
* Hide internal DB attributes
* Provide consistent response formatting

### 🛑 Do NOT expose

| Forbidden                      | Reason                  |
| ------------------------------ | ----------------------- |
| Internal fields                | Should stay server-side |
| Passwords / secrets            | Security                |
| Metadata not meant for clients | Noise or risk           |

### 📄 Example

```python
class EventResponse(EventBase):
    id: str
    created_at: datetime
    updated_at: datetime

    model_config = {
        "from_attributes": True  # allows ORM -> schema conversion
    }
```

---

# 📘 **4.5 Delete Response Schema**

Deletion should always return a **standard, predictable** response.

### 🎯 Recommended Format

* A boolean success flag
* Optional message
* Optional deleted ID

### 📄 Example

```python
class DeleteResponse(BaseModel):
    success: bool = True
    message: str = "Deleted successfully"
    id: str | None = None
```

This keeps delete responses consistent across all endpoints.

---

# 📘 **4.6 Error Response Schema**

Used for returning structured errors instead of raw FastAPI text.

### Types of Errors

1. **Pydantic validation errors**
2. **Application-level errors** (custom exceptions)
3. **Service/DAO-level errors** (e.g., foreign key issues)

### 📄 Recommended Standard Error Shape

```python
class ErrorResponse(BaseModel):
    error: str
    detail: str | None = None
    fields: dict[str, str] | None = None  # field-specific errors
```

### 📄 Example (Swagger / OpenAPI)

```json
{
  "error": "ValidationError",
  "detail": "Invalid email format",
  "fields": {
    "email": "Must contain '@'"
  }
}
```
