
# 📘 **5.1 String Field Validations**

### ✔️ 5.1.1 Length Constraints

Use `min_length`, `max_length` for all fields that must be bounded.

```python
name: Optional[str] = Field(default=None, min_length=2, max_length=100)
```

### ✔️ 5.1.2 Trimming Whitespace

Common practice: normalize inputs by removing accidental spaces.

```python
@field_validator("name")
def strip_name(cls, v):
    return v.strip() if isinstance(v, str) else v
```

### ✔️ 5.1.3 Prohibited Characters

Useful for names, city, state codes, etc.

```python
INVALID_CHARS = r"[<>;{}$]"

@field_validator("name")
def validate_chars(cls, v):
    if v and re.search(INVALID_CHARS, v):
        raise ValueError("Contains prohibited characters")
    return v
```

### ✔️ 5.1.4 Regex Patterns

Example: validating ZIP code or state code.

```python
ZIP_REGEX = re.compile(r"^\d{5}(-\d{4})?$")
STATE_REGEX = re.compile(r"^[A-Za-z]{2}$")
```

### ✔️ 5.1.5 Case Normalization

Automatically uppercase codes or lowercase emails.

```python
@field_validator("state")
def normalize_state(cls, v):
    return v.upper() if v else v
```

---

# 📘 **5.2 Number Validations**

### ✔️ 5.2.1 `ge` / `le`

Use for numeric ranges.

```python
age: Optional[int] = Field(default=None, ge=0, le=120)
```

### ✔️ 5.2.2 Float Edge Cases

Validate against `nan`, infinity, or overflow.

```python
@field_validator("price")
def validate_price(cls, v):
    if v is None:
        return v
    if math.isnan(v) or math.isinf(v):
        raise ValueError("Invalid float value")
    return round(v, 2)
```

---

# 📘 **5.3 Enum Validations**

Enums ensure predictable values in DB + API.

```python
class EventType(str, Enum):
    SERVICE = "Service"
    VISIT = "Visit"
```

Usage:

```python
event_type: Optional[EventType] = None
```

Pydantic auto-validates invalid enum values.

---

# 📘 **5.4 UUID Validations**

Use plain `UUID` type—Pydantic handles validation.

```python
chptr_id: Optional[UUID]
```

Invalid UUID → returns standard Pydantic error automatically.

To customize:

```python
@field_validator("chptr_id")
def validate_uuid(cls, v):
    if not isinstance(v, UUID):
        raise ValueError("Invalid UUID format")
    return v
```

---

# 📘 **5.5 Email Validations (without `EmailStr`)**

### ✔️ Custom Validator Pattern

Use a strict production-safe regex:

```python
EMAIL_REGEX = re.compile(
    r"^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$"
)

def validate_email_util(value: str | None):
    if not value:
        return value
    if not EMAIL_REGEX.match(value):
        raise ValueError("Invalid email format")
    return value
```

Usage:

```python
@field_validator("email")
def validate_email(cls, v):
    return validate_email_util(v)
```

---

# 📘 **5.6 URL Validations (without `HttpUrl`)**

Recommended strict regex:

```python
URL_REGEX = re.compile(
    r"^(https?://)[\w.-]+(\.[\w.-]+)+[/\w\-.?=&%]*$"
)

def validate_url_util(value):
    if not value:
        return value
    if not URL_REGEX.match(value):
        raise ValueError("Invalid URL format")
    return value
```

Usage:

```python
@field_validator("maps_link")
def validate_url(cls, v):
    return validate_url_util(v)
```

---

# 📘 **5.7 File Upload Validations**

### ✔️ 5.7.1 Size Limit

Recommended: validate in the route or dependency.

```python
MAX_SIZE_MB = 10

async def validate_file_size(file: UploadFile):
    content = await file.read()
    if len(content) > MAX_SIZE_MB * 1024 * 1024:
        raise HTTPException(400, f"File exceeds {MAX_SIZE_MB}MB limit.")
```

### ✔️ 5.7.2 MIME Type

Strict checking:

```python
ALLOWED_TYPES = {"image/jpeg", "image/png", "application/pdf"}

def validate_mime_type(file: UploadFile):
    if file.content_type not in ALLOWED_TYPES:
        raise HTTPException(400, "Unsupported file type.")
```

### ✔️ 5.7.3 Image Processing Rules

Check resolution or corrupt images:

```python
from PIL import Image

async def validate_image(file: UploadFile):
    try:
        img = Image.open(file.file)
        img.verify()
    except Exception:
        raise HTTPException(400, "Invalid or corrupted image file.")
```

---

# 📘 **5.8 Cross-Field Validations**

Used when **relationships between fields** need validation.

### ✔️ 5.8.1 "If X is provided → Y is required"

Example:
If `start_at` exists → `end_at` must exist.

```python
@model_validator(mode="after")
def validate_dates(cls, values):
    if values.start_at and not values.end_at:
        raise ValueError("end_at is required when start_at is provided.")
    return values
```

---

### ✔️ 5.8.2 Mutually Exclusive Fields

Exactly one must be provided.

```python
@model_validator(mode="after")
def validate_exclusive(cls, values):
    if values.email and values.phone:
        raise ValueError("Provide only one of: email or phone.")
    return values
```

---

### ✔️ 5.8.3 Comparative Rules (start < end)

```python
@model_validator(mode="after")
def validate_timeline(cls, values):
    if values.start_at and values.end_at:
        if values.start_at >= values.end_at:
            raise ValueError("start_at must be earlier than end_at")
    return values
```

