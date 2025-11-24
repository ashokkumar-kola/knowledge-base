
# 📘 **2. Schema Design Philosophy**

This section explains the reasoning behind our schema structure, how data flows through the system, and the patterns we follow to maintain consistency and reliability across the codebase.

---

## **2.1 Why We Create Base, Create, Update, and Response Schemas**

Our schema architecture follows a *layered* and *purpose-driven* design. Each schema type serves a specific role:

### **Base Schema**

* Contains common fields shared across Create, Update, and Response schemas.
* Helps avoid duplication of attribute definitions.
* Defines the core identity of the entity (e.g., person_name, metadata).

### **Create Schema**

* Represents the data required to create a new record.
* Includes fields that are mandatory for creation.
* Typically strict, with minimal optional fields.
* Performs more aggressive validation (e.g., required dates, required foreign keys).

### **Update Schema**

* Represents partial updates (`PATCH`).
* All fields are optional.
* Only provided fields are validated and updated.
* Supports flexible, incremental changes without overwriting unset data.

### **Response Schema**

* Defines the exact structure returned from the API.
* Ensures response formatting is clean, predictable, and decoupled from SQLAlchemy.
* Prevents leaking internal DB structures into external API contracts.

This separation ensures clarity of intent, maintainability, and long-term extensibility as APIs evolve.

---

## **2.2 Data Flow: Request → Service → DAO → DB → Response**

A request travels through multiple layers before reaching the database and returning to the client. The sequence below ensures separation of concerns:

```
Client Request  
    ↓ (validated by FastAPI + Pydantic)
Request Schema  
    ↓  
Service Layer (business rules, orchestration)  
    ↓  
DAO / Repository (SQLAlchemy session operations)  
    ↓  
Database (PostgreSQL)  
    ↓  
ORM Model (SQLAlchemy result)  
    ↓  
Response Schema (Pydantic)  
    ↓  
Client Response
```

### Key Guarantees:

* **Validation happens before business logic.**
* **Service layer never receives unvalidated or incomplete data.**
* **Database layer receives only clean, typed fields.**
* **Response schemas fully control what is exposed externally.**

This layered flow prevents bugs, enforces data integrity, and creates predictable API behavior.

---

## **2.3 Naming Conventions**

To maintain consistency across the codebase, we follow these conventions:

### **Schema Names**

* `UserBase`, `UserCreate`, `UserUpdate`, `UserResponse`
* Always suffix with the intended purpose (`Create`, `Update`, etc.)
* Use PascalCase for class names.

### **Field Names**

* Snake_case for all attributes.
* Follow DB column names where possible to avoid mapping confusion.

### **Validation Methods**

* Named using `validate_<fieldname>` for clarity.
* Group utilities under `/utils/validators.py`.

### **Service & DAO Layers**

* Services: `<Entity>Service` (business logic)
* DAO/Repository: `<Entity>Repository` or `<Entity>DAO` (data access logic)

Clear naming ensures predictable structure and easier onboarding.

---

## **2.4 Re-Using Shared Logic**

To avoid duplicated validation or transformation logic, we extract reusable utilities:

### **Shared validators**

Placed in `/utils/validators.py`, e.g.:

* `validate_email_util`
* `validate_phone_util`
* `validate_zipcode_util`
* `parse_date_util`
* `validate_enum_choice`

### **Base Models**

Shared fields and configuration (e.g., `orm_mode`, `from_attributes`) are placed in a `BaseSchema`.

### **Reusable Mixins**

Used for patterns like timestamps:

* `CreatedUpdatedMixin`
* `SoftDeleteMixin`

### **Common Response Envelopes**

Use a shared API response wrapper for consistency:

```json
{
  "status": "success",
  "message": "...",
  "data": { ... }
}
```

Re-use reduces bugs, improves clarity, and avoids schema explosion.

---

## **2.5 Validation Philosophy**

Validation happens in **layers**, not all at once.
Our philosophy:

### **1. Validate early**

Pydantic validates incoming data *before* business logic runs.

### **2. Reject ambiguous or partial data**

* Missing fields in Create = error
* Empty strings in Update = treated as None
* Unrecognized fields = rejected unless explicitly allowed

### **3. Use strict, explicit types**

* `datetime` for timestamps
* `UUID` for IDs
* `EmailStr` or custom regex for emails
* Enums for controlled sets of values

### **4. Prefer reusable validators over inline checks**

* Avoid rewriting regex or logic in every schema.

### **5. Never trust client data**

Even if validation passes, service layer performs additional checks such as:

* Cross-field checks
* Date ordering (start < end)
* Event type inference rules

### **6. Error messages must be descriptive**

Errors should guide the client on what failed and how to fix it.

Strong validation ensures consistent data, stable APIs, and defensive programming practices.

---
