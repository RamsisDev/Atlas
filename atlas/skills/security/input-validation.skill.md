---
name: input-validation
description: >
  Rules and patterns for validating and sanitizing all external input following the
  OWASP Input Validation Cheat Sheet, Pydantic v2, and FluentValidation (.NET).
  Covers allowlisting, data type enforcement, file upload validation, and output
  encoding. Required by Dev Agents handling user input, D05 Security agents, and
  D02 Security Pre-scanner for any feature accepting external data.
applies-to: [dev-agent, security-lead, security-prescanner, qa-agent]
stack: none
layer: api
version: 1.0
---

# Input Validation

<context>
This skill applies to every code path that accepts data from external sources: HTTP
request bodies, query parameters, path parameters, headers, file uploads, message
queue payloads, and CLI arguments. Validation MUST occur server-side at the entry
point (API layer) before the data reaches the application or domain layers.
</context>

<rules>
## General Principles
- All input MUST be validated server-side regardless of any client-side validation.
- Validation MUST use an allowlist (permit known-good) approach, not a denylist (block known-bad).
- Input MUST be validated for: type, format, length, range, and allowed characters before processing.
- Invalid input MUST be rejected immediately with a clear error; it MUST NOT be sanitized and silently accepted.
- Validation errors MUST return structured responses that identify the invalid field without revealing system internals.

## String Inputs
- All string fields MUST have a defined maximum length enforced at the API layer.
- Free-text fields accepting user content MUST be length-limited and stripped of leading/trailing whitespace.
- Fields with a fixed format (email, UUID, phone, postal code) MUST be validated against a strict regex or format library.
- String inputs used in downstream contexts (SQL, HTML, shell) MUST be parameterized/escaped at the point of use, not pre-sanitized at input.

## Numeric Inputs
- Numeric fields MUST declare explicit minimum and maximum allowed values.
- Integer overflow MUST be considered; use 64-bit integers for IDs and quantities where values can grow large.
- Currency and financial values MUST use decimal types, not floating-point (float/double).

## File Uploads
- Accepted file types MUST be validated by MIME type detection (inspecting file magic bytes), not by file extension alone.
- Maximum file size MUST be enforced before reading the file contents into memory.
- Uploaded file names MUST be replaced with a server-generated UUID before storage; the original name MUST NOT be used as a file path component.
- Uploaded files MUST be stored outside the web root — never in a publicly accessible directory.
- Content scanning (virus/malware) SHOULD be applied before making files available to other users.

## Structured Data (JSON / XML)
- JSON payloads MUST be schema-validated before accessing fields; unexpected fields SHOULD be ignored or rejected.
- XML inputs MUST disable external entity processing (XXE) by disabling DOCTYPE declarations and external entities.
- Nested/recursive structures MUST have a depth limit to prevent stack overflows.

## Output Encoding
- Data written to HTML MUST be HTML-encoded (entity encoding); never use `innerHTML` with unencoded data.
- Data written to JavaScript contexts MUST be JavaScript-escaped.
- Data written to SQL MUST be parameterized, not encoded — encoding is not sufficient to prevent injection.
- Data written to shell commands MUST use parameterized APIs or a strict allowlist.
</rules>

<patterns>
## Pydantic v2 Request Validation (Python / FastAPI)

```python
from pydantic import BaseModel, EmailStr, Field, field_validator
import re

class CreateUserRequest(BaseModel):
    model_config = {"strict": True}  # reject coercions

    username: str = Field(min_length=3, max_length=50, pattern=r"^[a-zA-Z0-9_]+$")
    email: EmailStr
    age: int = Field(ge=18, le=120)
    bio: str = Field(default="", max_length=500)

    @field_validator("username")
    @classmethod
    def no_reserved_words(cls, v: str) -> str:
        reserved = {"admin", "root", "system"}
        if v.lower() in reserved:
            raise ValueError("Username is reserved.")
        return v
```

## File Upload Validation (Python)

```python
import magic  # python-magic — reads file magic bytes
from uuid import uuid4
from pathlib import Path

ALLOWED_MIME_TYPES = {"image/png", "image/jpeg", "application/pdf"}
MAX_FILE_SIZE_BYTES = 10 * 1024 * 1024  # 10 MB

async def validate_and_store_upload(file: UploadFile) -> str:
    # 1. Size check (before reading all content into memory)
    content = await file.read(MAX_FILE_SIZE_BYTES + 1)
    if len(content) > MAX_FILE_SIZE_BYTES:
        raise HTTPException(413, "File too large.")

    # 2. MIME type by magic bytes — not by extension
    mime = magic.from_buffer(content, mime=True)
    if mime not in ALLOWED_MIME_TYPES:
        raise HTTPException(415, f"File type '{mime}' not allowed.")

    # 3. Replace original filename with UUID
    safe_name = f"{uuid4()}.{mime.split('/')[-1]}"
    dest = Path(settings.UPLOAD_DIR) / safe_name  # outside web root
    dest.write_bytes(content)
    return safe_name
```

## FluentValidation (.NET)

```csharp
public class CreateUserCommandValidator : AbstractValidator<CreateUserCommand>
{
    public CreateUserCommandValidator()
    {
        RuleFor(x => x.Username)
            .NotEmpty()
            .Length(3, 50)
            .Matches(@"^[a-zA-Z0-9_]+$").WithMessage("Username may only contain letters, digits, and underscores.");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress();

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120);

        RuleFor(x => x.Bio)
            .MaximumLength(500);
    }
}
```

## XXE Prevention (C# / XmlDocument)

```csharp
var settings = new XmlReaderSettings
{
    DtdProcessing = DtdProcessing.Prohibit,   // disables DOCTYPE
    XmlResolver = null,                        // disables external entities
    MaxCharactersFromEntities = 1024,
};
using var reader = XmlReader.Create(stream, settings);
```

## Structured Validation Error Response

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "Validation Failed",
  "status": 422,
  "errors": {
    "username": ["Username may only contain letters, digits, and underscores."],
    "email": ["The Email field is not a valid email address."]
  }
}
```
</patterns>

<anti-patterns>
- **Extension-only file type check**: `if filename.endswith(".pdf")` — attacker uploads `shell.php` renamed to `shell.php.pdf`; always check MIME magic bytes.
- **Sanitizing instead of rejecting**: replacing `<script>` with empty string — double-encoding and context-switching attacks can bypass sanitization; reject invalid input.
- **Storing original file names**: `Path(upload_dir) / user_filename` — path traversal via `../../etc/passwd` in the filename; always generate a server-side UUID name.
- **No max length on text fields**: allows storing megabytes of data in a single field — database DoS or log flooding.
- **Denylist regex**: `pattern = r"[^<>\"']"` — impossible to enumerate all dangerous characters; use an allowlist of permitted characters instead.
- **Float for currency**: `price: float = 19.99` — binary floating-point cannot represent most decimal fractions; use `Decimal` in Python or `decimal` in C#.
- **Client-side validation only**: JavaScript form validation skipped by any HTTP client — always re-validate server-side.
- **Exposing validation details**: `"error": "Column 'users.email' has a unique constraint violation"` — reveals schema; return generic field-level errors only.
</anti-patterns>

<checklist>
- [ ] All input fields have explicit type, max length, and format constraints declared at the API layer.
- [ ] File uploads validated by MIME magic bytes, size limit, and UUID rename before storage.
- [ ] Uploaded files stored outside the web-accessible directory.
- [ ] XML input has DTD processing and external entities disabled.
- [ ] Numeric fields have explicit min/max ranges; currency uses decimal types.
- [ ] Validation errors return RFC 7807 Problem Details with field-level messages — no internal details.
- [ ] No client-side-only validation paths exist (server always re-validates).
- [ ] No sanitization of dangerous input — invalid input is rejected outright.
</checklist>
