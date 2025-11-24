
# 📘 **3. Pydantic Basics (FastAPI Context)**

This section introduces the foundational concepts of Pydantic within a FastAPI application. Understanding these principles ensures clean schema definitions, predictable validation, and consistent serialization across your API.

---

## **3.1 Pydantic Model Structure**

A Pydantic model defines the shape, constraints, and validation logic for your data.

### **Key Characteristics**

* Each model is a Python class inheriting from `BaseModel`.
* Fields are declared using type hints.
* Validation runs automatically when the model is created.
* Models can represent:

  * Incoming request bodies
  * Outgoing responses
  * Internal data conversions

### **Example**

```python
class User(BaseModel):
    id: UUID
    email: str
    is_active: bool = True
```

---

## **3.2 `Field()` Usage**

The `Field()` function provides metadata and constraints for each attribute.

### **Common Parameters**

* `default`: default value
* `default_factory`: function to generate a default
* `gt`, `ge`, `lt`, `le`: numeric constraints
* `min_length`, `max_length`: string constraints
* `pattern`: regex validation
* `description`: schema documentation for OpenAPI
* `deprecated`: mark fields as deprecated

### **Example**

```python
name: str = Field(..., min_length=3, max_length=50, description="User name")
```

---

## **3.3 Optional vs Nullable**

In Pydantic (v2+):

### **Optional**

Means *the field may be omitted*.

```python
name: Optional[str]
```

### **Nullable**

Means *the field value may be `None`*.

Pydantic allows both semantics at the same time depending on config.
But generally:

* `Optional[str] = None` → may be missing OR may be `None`
* `str | None` is identical in v2+

### Example:

```python
phone: str | None = None  # allowed: missing, None, or valid string
```

Still, **“Optional” does NOT mean nullable unless you set a default of `None`**.

---

## **3.4 Strict Mode**

Strict mode enforces that input types cannot be coerced.

### **Without strict mode**

```python
age: int
```

Incoming `"25"` (string) will be accepted and converted to `25` (int).

### **With strict mode**

```python
age: StrictInt
```

Incoming `"25"` will fail validation.

Strict types include:

* `StrictStr`
* `StrictInt`
* `StrictBool`
* `StrictFloat`

Strict mode is recommended for APIs requiring strong type guarantees.

---

## **3.5 Custom Validators**

Custom validation allows extra logic beyond type hints.

### **Pydantic v2 (FastAPI expects this)**

Use `@field_validator`.

```python
from pydantic import field_validator

class User(BaseModel):
    email: str

    @field_validator("email")
    def validate_email(cls, value):
        if "@" not in value:
            raise ValueError("Invalid email format")
        return value
```

Validators can be:

* **per-field**
* **root-level** (using `@model_validator(mode="after")`)
* **pre** or **post** processing

Reusable validators should be extracted into utilities.

---

## **3.6 Differences Between Pydantic v2 and v3**

Pydantic v3 introduces refinements but the fundamentals are similar.
FastAPI currently targets v2+, so knowing the differences matters.

### **1️⃣ `@field_validator` replaces `@validator`**

* v1 used:

  ```python
  @validator("email")
  ```
* v2 uses:

  ```python
  @field_validator("email")
  ```
* v3 continues the v2 approach, dropping older APIs.

### **2️⃣ Serialization Changes**

`model.dict()` is deprecated.

Use:

```python
model.model_dump()
```

For JSON:

```python
model.model_dump_json()
```

### **3️⃣ Config Model Updates**

Old:

```python
class Config:
    orm_mode = True
```

New:

```python
model_config = {
    "from_attributes": True,
    "extra": "forbid"
}
```

### **4️⃣ Validation Pipeline Rewritten**

Pydantic v2 introduced a fully rewritten validation engine (`pydantic-core`), giving:

* Better performance
* Stricter type handling
* Cleaner error messages

### **5️⃣ Form Parsing**

FastAPI form parsing still uses:

```python
from fastapi import Form
```

Example:

```python
class Login(BaseModel):
    username: str = Form(...)
    password: str = Form(...)
```

Pydantic does *not* handle form parsing itself.
FastAPI injects form values before the model validates them.
