# FastAPI Comprehensive Reference

A complete reference for building with FastAPI — covering every class, function signature, parameter type, and pattern in the framework. Built from the source code of the FastAPI repository.

---

## Table of Contents

- [1. FastAPI Application](#1-fastapi-application)
- [2. APIRouter](#2-apirouter)
- [3. Route Decorators](#3-route-decorators)
- [4. Path Parameters](#4-path-parameters)
- [5. Query Parameters](#5-query-parameters)
- [6. Header Parameters](#6-header-parameters)
- [7. Cookie Parameters](#7-cookie-parameters)
- [8. Body Parameters](#8-body-parameters)
- [9. Form & File Parameters](#9-form--file-parameters)
- [10. UploadFile](#10-uploadfile)
- [11. Dependency Injection](#11-dependency-injection)
- [12. Security](#12-security)
- [13. Request & Response](#13-request--response)
- [14. HTTPException & Error Handling](#14-httpexception--error-handling)
- [15. Background Tasks](#15-background-tasks)
- [16. WebSockets](#16-websockets)
- [17. Middleware](#17-middleware)
- [18. Lifespan Events](#18-lifespan-events)
- [19. Static Files & Templates](#19-static-files--templates)
- [20. OpenAPI & Documentation](#20-openapi--documentation)
- [21. JSON Encoding](#21-json-encoding)
- [22. Sub-Applications & Mounting](#22-sub-applications--mounting)
- [23. Testing](#23-testing)
- [24. Status Codes](#24-status-codes)
- [25. Common Patterns](#25-common-patterns)

---

## 1. FastAPI Application

**Import:** `from fastapi import FastAPI`

**Source:** `fastapi/applications.py`

### Constructor

```python
app = FastAPI(
    *,
    debug: bool = False,
    routes: list[BaseRoute] | None = None,
    title: str = "FastAPI",
    summary: str | None = None,
    description: str = "",
    version: str = "0.1.0",
    openapi_url: str | None = "/openapi.json",
    openapi_tags: list[dict[str, Any]] | None = None,
    servers: list[dict[str, str | Any]] | None = None,
    dependencies: Sequence[Depends] | None = None,
    default_response_class: type[Response] = Default(JSONResponse),
    redirect_slashes: bool = True,
    docs_url: str | None = "/docs",
    redoc_url: str | None = "/redoc",
    swagger_ui_oauth2_redirect_url: str | None = "/docs/oauth2-redirect",
    swagger_ui_init_oauth: dict[str, Any] | None = None,
    middleware: Sequence[Middleware] | None = None,
    exception_handlers: dict[int | type[Exception], Callable[[Request, Any], Coroutine[Any, Any, Response]]] | None = None,
    on_startup: Sequence[Callable[[], Any]] | None = None,       # deprecated
    on_shutdown: Sequence[Callable[[], Any]] | None = None,      # deprecated
    lifespan: Lifespan[AppType] | None = None,
    terms_of_service: str | None = None,
    contact: dict[str, str | Any] | None = None,
    license_info: dict[str, str | Any] | None = None,
    openapi_prefix: str = "",                                     # deprecated
    root_path: str = "",
    root_path_in_servers: bool = True,
    responses: dict[int | str, dict[str, Any]] | None = None,
    callbacks: list[BaseRoute] | None = None,
    webhooks: APIRouter | None = None,
    deprecated: bool | None = None,
    include_in_schema: bool = True,
    swagger_ui_parameters: dict[str, Any] | None = None,
    generate_unique_id_function: Callable[[APIRoute], str] = Default(generate_unique_id),
    separate_input_output_schemas: bool = True,
    openapi_external_docs: dict[str, Any] | None = None,
    strict_content_type: bool = True,
    **extra: Any,
)
```

### Key Parameters

| Parameter | Description |
|-----------|-------------|
| `title` | Name shown in OpenAPI docs |
| `version` | API version string |
| `description` | Markdown description for OpenAPI |
| `summary` | Short summary of the API |
| `openapi_url` | Path to OpenAPI JSON. Set to `None` to disable |
| `docs_url` | Path to Swagger UI. Set to `None` to disable |
| `redoc_url` | Path to ReDoc. Set to `None` to disable |
| `dependencies` | Dependencies applied to every route |
| `lifespan` | Async context manager for startup/shutdown |
| `root_path` | Root path when behind a proxy |
| `servers` | Server URLs for OpenAPI schema |
| `separate_input_output_schemas` | Generate separate schemas for input vs output |
| `strict_content_type` | Validate Content-Type header on requests |

### Key Methods

```python
# Include a router
app.include_router(
    router: APIRouter,
    *,
    prefix: str = "",
    tags: list[str | Enum] | None = None,
    dependencies: Sequence[Depends] | None = None,
    responses: dict[int | str, dict[str, Any]] | None = None,
    deprecated: bool | None = None,
    include_in_schema: bool = True,
    default_response_class: type[Response] = Default(JSONResponse),
    callbacks: list[BaseRoute] | None = None,
    generate_unique_id_function: Callable[[APIRoute], str] = Default(generate_unique_id),
) -> None

# Add middleware
app.add_middleware(MiddlewareClass, **options)

# Exception handler decorator
@app.exception_handler(exc_class_or_status_code: int | type[Exception])
async def handler(request: Request, exc: Exception) -> Response: ...

# HTTP middleware decorator
@app.middleware("http")
async def middleware(request: Request, call_next) -> Response: ...

# Get the OpenAPI schema dict
app.openapi() -> dict[str, Any]
```

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `app.state` | `State` | Arbitrary state storage |
| `app.dependency_overrides` | `dict` | Override dependencies (for testing) |
| `app.router` | `APIRouter` | Internal router |
| `app.routes` | `list[BaseRoute]` | All registered routes |
| `app.openapi_schema` | `dict \| None` | Cached OpenAPI schema |
| `app.openapi_version` | `str` | OpenAPI spec version (default: "3.1.0") |

---

## 2. APIRouter

**Import:** `from fastapi import APIRouter`

**Source:** `fastapi/routing.py`

### Constructor

```python
router = APIRouter(
    *,
    prefix: str = "",
    tags: list[str | Enum] | None = None,
    dependencies: Sequence[Depends] | None = None,
    default_response_class: type[Response] = Default(JSONResponse),
    responses: dict[int | str, dict[str, Any]] | None = None,
    callbacks: list[BaseRoute] | None = None,
    routes: list[BaseRoute] | None = None,
    redirect_slashes: bool = True,
    default: ASGIApp | None = None,
    dependency_overrides_provider: Any | None = None,
    route_class: type[APIRoute] = APIRoute,
    on_startup: Sequence[Callable[[], Any]] | None = None,
    on_shutdown: Sequence[Callable[[], Any]] | None = None,
    lifespan: Lifespan[Any] | None = None,
    deprecated: bool | None = None,
    include_in_schema: bool = True,
    generate_unique_id_function: Callable[[APIRoute], str] = Default(generate_unique_id),
    strict_content_type: bool | DefaultPlaceholder = Default(True),
)
```

### Key Methods

The router has the same HTTP decorators as `FastAPI` (`get`, `post`, `put`, `delete`, `patch`, `options`, `head`, `trace`) plus:

```python
# Include another router (nested routers)
router.include_router(
    router: APIRouter,
    *,
    prefix: str = "",
    tags: list[str | Enum] | None = None,
    dependencies: Sequence[Depends] | None = None,
    default_response_class: type[Response] = Default(JSONResponse),
    responses: dict[int | str, dict[str, Any]] | None = None,
    callbacks: list[BaseRoute] | None = None,
    deprecated: bool | None = None,
    include_in_schema: bool = True,
    generate_unique_id_function: Callable[[APIRoute], str] = Default(generate_unique_id),
) -> None

# Programmatic route addition
router.add_api_route(
    path: str,
    endpoint: Callable[..., Any],
    *,
    response_model: Any = Default(None),
    status_code: int | None = None,
    tags: list[str | Enum] | None = None,
    dependencies: Sequence[Depends] | None = None,
    summary: str | None = None,
    description: str | None = None,
    response_description: str = "Successful Response",
    responses: dict[int | str, dict[str, Any]] | None = None,
    deprecated: bool | None = None,
    methods: list[str] | None = None,
    operation_id: str | None = None,
    response_model_include: IncEx | None = None,
    response_model_exclude: IncEx | None = None,
    response_model_by_alias: bool = True,
    response_model_exclude_unset: bool = False,
    response_model_exclude_defaults: bool = False,
    response_model_exclude_none: bool = False,
    include_in_schema: bool = True,
    response_class: type[Response] | DefaultPlaceholder = Default(JSONResponse),
    name: str | None = None,
    route_class_override: type[APIRoute] | None = None,
    callbacks: list[BaseRoute] | None = None,
    openapi_extra: dict[str, Any] | None = None,
    generate_unique_id_function: Callable[[APIRoute], str] = Default(generate_unique_id),
    strict_content_type: bool | DefaultPlaceholder = Default(True),
) -> None
```

---

## 3. Route Decorators

All HTTP method decorators (`@app.get()`, `@app.post()`, `@router.get()`, etc.) share the same parameter signature:

```python
@app.get(  # or .post, .put, .delete, .patch, .options, .head, .trace
    path: str,
    *,
    response_model: Any = Default(None),
    status_code: int | None = None,
    tags: list[str | Enum] | None = None,
    dependencies: Sequence[Depends] | None = None,
    summary: str | None = None,
    description: str | None = None,
    response_description: str = "Successful Response",
    responses: dict[int | str, dict[str, Any]] | None = None,
    deprecated: bool | None = None,
    operation_id: str | None = None,
    response_model_include: IncEx | None = None,
    response_model_exclude: IncEx | None = None,
    response_model_by_alias: bool = True,
    response_model_exclude_unset: bool = False,
    response_model_exclude_defaults: bool = False,
    response_model_exclude_none: bool = False,
    include_in_schema: bool = True,
    response_class: type[Response] = Default(JSONResponse),
    name: str | None = None,
    callbacks: list[BaseRoute] | None = None,
    openapi_extra: dict[str, Any] | None = None,
    generate_unique_id_function: Callable[[APIRoute], str] = Default(generate_unique_id),
)
```

### Decorator Parameter Reference

| Parameter | Description |
|-----------|-------------|
| `path` | URL path (supports `{param}` syntax) |
| `response_model` | Pydantic model for response validation & docs |
| `status_code` | HTTP status code for the response (default varies by method) |
| `tags` | Tags for grouping in OpenAPI docs |
| `dependencies` | Route-level dependencies |
| `summary` | Short summary in OpenAPI |
| `description` | Detailed description (defaults to docstring) |
| `response_description` | Description for the default response |
| `responses` | Additional response schemas for OpenAPI |
| `deprecated` | Mark as deprecated in OpenAPI |
| `operation_id` | Unique ID for OpenAPI operation |
| `response_model_exclude_unset` | Exclude fields not explicitly set |
| `response_model_exclude_defaults` | Exclude fields with default values |
| `response_model_exclude_none` | Exclude fields with None values |
| `include_in_schema` | Include in OpenAPI schema |
| `response_class` | Response class to use (default: `JSONResponse`) |
| `openapi_extra` | Extra OpenAPI schema data for this route |

---

## 4. Path Parameters

**Import:** `from fastapi import Path`

**Source:** `fastapi/params.py`

```python
Path(
    default: Any = ...,  # Always required (Ellipsis)
    *,
    default_factory: Callable[[], Any] | None = _Unset,
    annotation: Any | None = None,
    alias: str | None = None,
    alias_priority: int | None = _Unset,
    validation_alias: str | AliasPath | AliasChoices | None = None,
    serialization_alias: str | None = None,
    title: str | None = None,
    description: str | None = None,
    gt: float | None = None,
    ge: float | None = None,
    lt: float | None = None,
    le: float | None = None,
    min_length: int | None = None,
    max_length: int | None = None,
    pattern: str | None = None,
    regex: str | None = None,          # deprecated, use pattern
    discriminator: str | None = None,
    strict: bool | None = _Unset,
    multiple_of: float | None = _Unset,
    allow_inf_nan: bool | None = _Unset,
    max_digits: int | None = _Unset,
    decimal_places: int | None = _Unset,
    examples: list[Any] | None = None,
    example: Any | None = _Unset,      # deprecated
    openapi_examples: dict[str, Example] | None = None,
    deprecated: str | bool | None = None,
    include_in_schema: bool = True,
    json_schema_extra: dict[str, Any] | None = None,
    **extra: Any,
)
```

### Usage

```python
@app.get("/items/{item_id}")
async def read_item(
    item_id: int = Path(title="The ID of the item", ge=1),
):
    return {"item_id": item_id}

# With Annotated (preferred)
from typing import Annotated

@app.get("/items/{item_id}")
async def read_item(
    item_id: Annotated[int, Path(title="The ID of the item", ge=1)],
):
    return {"item_id": item_id}
```

---

## 5. Query Parameters

**Import:** `from fastapi import Query`

**Source:** `fastapi/params.py`

```python
Query(
    default: Any = Undefined,  # Optional by default
    *,
    # Same parameters as Path (see Section 4)
    # plus all validation parameters
)
```

### Usage

```python
@app.get("/items/")
async def read_items(
    q: Annotated[str | None, Query(max_length=50)] = None,
    skip: Annotated[int, Query(ge=0)] = 0,
    limit: Annotated[int, Query(le=100)] = 10,
):
    return {"q": q, "skip": skip, "limit": limit}

# List query parameters: /items?q=foo&q=bar
@app.get("/items/")
async def read_items(
    q: Annotated[list[str] | None, Query()] = None,
):
    return {"q": q}
```

---

## 6. Header Parameters

**Import:** `from fastapi import Header`

**Source:** `fastapi/params.py`

```python
Header(
    default: Any = Undefined,
    *,
    convert_underscores: bool = True,  # user_agent -> user-agent
    # Same parameters as Path (see Section 4)
)
```

### Usage

```python
@app.get("/items/")
async def read_items(
    user_agent: Annotated[str | None, Header()] = None,
    x_token: Annotated[list[str] | None, Header()] = None,
):
    return {"User-Agent": user_agent, "X-Token": x_token}
```

Note: `convert_underscores=True` (default) converts Python `user_agent` to HTTP header `user-agent`.

---

## 7. Cookie Parameters

**Import:** `from fastapi import Cookie`

**Source:** `fastapi/params.py`

```python
Cookie(
    default: Any = Undefined,
    *,
    # Same parameters as Path (see Section 4)
)
```

### Usage

```python
@app.get("/items/")
async def read_items(
    session_id: Annotated[str | None, Cookie()] = None,
):
    return {"session_id": session_id}
```

---

## 8. Body Parameters

**Import:** `from fastapi import Body`

**Source:** `fastapi/params.py`

```python
Body(
    default: Any = Undefined,
    *,
    embed: bool | None = None,
    media_type: str = "application/json",
    # Same validation parameters as Path (see Section 4)
    alias: str | None = None,
    title: str | None = None,
    description: str | None = None,
    gt: float | None = None,
    ge: float | None = None,
    lt: float | None = None,
    le: float | None = None,
    min_length: int | None = None,
    max_length: int | None = None,
    pattern: str | None = None,
    examples: list[Any] | None = None,
    openapi_examples: dict[str, Example] | None = None,
    deprecated: str | bool | None = None,
    include_in_schema: bool = True,
    json_schema_extra: dict[str, Any] | None = None,
    **extra: Any,
)
```

### Usage

```python
from pydantic import BaseModel

class Item(BaseModel):
    name: str
    price: float
    description: str | None = None

# Single body parameter (extracted from root JSON)
@app.post("/items/")
async def create_item(item: Item):
    return item

# Multiple body params (each becomes a key in JSON)
@app.put("/items/{item_id}")
async def update_item(
    item_id: int,
    item: Item,
    importance: Annotated[int, Body(gt=0)],
):
    return {"item_id": item_id, "item": item, "importance": importance}

# Embed a single body parameter under its field name
@app.put("/items/{item_id}")
async def update_item(
    item: Annotated[Item, Body(embed=True)],
):
    # Expects: {"item": {"name": "...", "price": ...}}
    return item
```

---

## 9. Form & File Parameters

### Form

**Import:** `from fastapi import Form`

**Source:** `fastapi/params.py` (inherits from `Body`)

```python
Form(
    default: Any = Undefined,
    *,
    media_type: str = "application/x-www-form-urlencoded",
    # Same parameters as Body (see Section 8)
)
```

### File

**Import:** `from fastapi import File`

**Source:** `fastapi/params.py` (inherits from `Form`)

```python
File(
    default: Any = Undefined,
    *,
    media_type: str = "multipart/form-data",
    # Same parameters as Form/Body
)
```

### Usage

```python
# Form data
@app.post("/login/")
async def login(
    username: Annotated[str, Form()],
    password: Annotated[str, Form()],
):
    return {"username": username}

# File upload
@app.post("/upload/")
async def upload(
    file: Annotated[UploadFile, File(description="A file to upload")],
):
    return {"filename": file.filename}

# Multiple files
@app.post("/upload-many/")
async def upload_many(
    files: Annotated[list[UploadFile], File()],
):
    return {"filenames": [f.filename for f in files]}

# File as bytes (small files only)
@app.post("/upload-bytes/")
async def upload_bytes(
    file: Annotated[bytes, File()],
):
    return {"file_size": len(file)}
```

---

## 10. UploadFile

**Import:** `from fastapi import UploadFile`

**Source:** `fastapi/datastructures.py`

### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `filename` | `str \| None` | Original filename from the client |
| `size` | `int \| None` | File size in bytes |
| `headers` | `Headers` | Multipart headers |
| `content_type` | `str \| None` | Content-Type of the uploaded file |
| `file` | `BinaryIO` | Underlying Python file object (SpooledTemporaryFile) |

### Async Methods

```python
await upload_file.read(size: int = -1) -> bytes     # Read bytes (-1 = all)
await upload_file.write(data: bytes) -> None         # Write bytes
await upload_file.seek(offset: int) -> None          # Move file position
await upload_file.close() -> None                    # Close the file
```

All async methods run the underlying sync I/O in a threadpool.

---

## 11. Dependency Injection

### Depends

**Import:** `from fastapi import Depends`

**Source:** `fastapi/params.py`

```python
@dataclass(frozen=True)
class Depends:
    dependency: Callable[..., Any] | None = None
    use_cache: bool = True
    scope: Literal["function", "request"] | None = None
```

| Parameter | Description |
|-----------|-------------|
| `dependency` | Callable to execute. If `None`, infers from type annotation |
| `use_cache` | Cache result within same request (default `True`) |
| `scope` | `"function"` or `"request"` for generator cleanup timing |

### Function Dependencies

```python
async def common_parameters(
    q: str | None = None,
    skip: int = 0,
    limit: int = 100,
):
    return {"q": q, "skip": skip, "limit": limit}

@app.get("/items/")
async def read_items(
    commons: Annotated[dict, Depends(common_parameters)],
):
    return commons
```

### Class Dependencies

```python
class CommonQueryParams:
    def __init__(self, q: str | None = None, skip: int = 0, limit: int = 100):
        self.q = q
        self.skip = skip
        self.limit = limit

@app.get("/items/")
async def read_items(
    commons: Annotated[CommonQueryParams, Depends()],  # inferred from type
):
    return {"q": commons.q}
```

### Generator Dependencies (with cleanup)

```python
# Sync generator
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Async generator
async def get_async_db():
    db = AsyncSessionLocal()
    try:
        yield db
    finally:
        await db.close()

@app.get("/items/")
async def read_items(db: Annotated[Session, Depends(get_db)]):
    return db.query(Item).all()
```

### Nested Dependencies

Dependencies can have their own dependencies, forming a tree that FastAPI resolves automatically:

```python
def query_extractor(q: str | None = None) -> str | None:
    return q

def query_or_cookie_extractor(
    q: Annotated[str | None, Depends(query_extractor)],
    last_query: Annotated[str | None, Cookie()] = None,
) -> str:
    return q or last_query or ""

@app.get("/items/")
async def read_items(
    query: Annotated[str, Depends(query_or_cookie_extractor)],
):
    return {"query": query}
```

### Dependency Overrides (for testing)

```python
# In tests
def override_get_db():
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db
```

### Route-Level and App-Level Dependencies

```python
# Applied to all routes in app
app = FastAPI(dependencies=[Depends(verify_token)])

# Applied to all routes in router
router = APIRouter(dependencies=[Depends(verify_token)])

# Applied to a single route
@app.get("/items/", dependencies=[Depends(verify_token)])
async def read_items():
    return []
```

---

## 12. Security

**Import:** `from fastapi.security import ...`

**Source:** `fastapi/security/`

### OAuth2PasswordBearer

```python
from fastapi.security import OAuth2PasswordBearer

oauth2_scheme = OAuth2PasswordBearer(
    tokenUrl: str,                                # Path to token endpoint
    scheme_name: str | None = None,               # Name in OpenAPI
    scopes: dict[str, str] | None = None,         # Available scopes
    description: str | None = None,               # OpenAPI description
    auto_error: bool = True,                      # 401 if no token
    refreshUrl: str | None = None,                # Token refresh URL
)

# When called, returns the bearer token string (without "Bearer " prefix)
token: str = await oauth2_scheme(request)
```

### OAuth2PasswordRequestForm

```python
from fastapi.security import OAuth2PasswordRequestForm

@app.post("/token")
async def login(
    form_data: Annotated[OAuth2PasswordRequestForm, Depends()],
):
    # form_data.username: str
    # form_data.password: str
    # form_data.scopes: list[str]      (parsed from space-separated string)
    # form_data.grant_type: str | None
    # form_data.client_id: str | None
    # form_data.client_secret: str | None
    ...
```

### OAuth2PasswordRequestFormStrict

Same as `OAuth2PasswordRequestForm` but `grant_type` is **required** (must be `"password"`).

### OAuth2AuthorizationCodeBearer

```python
from fastapi.security import OAuth2AuthorizationCodeBearer

oauth2_code = OAuth2AuthorizationCodeBearer(
    authorizationUrl: str,                        # URL for user authorization
    tokenUrl: str,                                # URL to exchange code for token
    refreshUrl: str | None = None,
    scheme_name: str | None = None,
    scopes: dict[str, str] | None = None,
    description: str | None = None,
    auto_error: bool = True,
)
```

### HTTPBasic

```python
from fastapi.security import HTTPBasic, HTTPBasicCredentials

security = HTTPBasic(
    scheme_name: str | None = None,
    realm: str | None = None,                     # Auth realm
    description: str | None = None,
    auto_error: bool = True,
)

@app.get("/users/me")
async def read_current_user(
    credentials: Annotated[HTTPBasicCredentials, Depends(security)],
):
    # credentials.username: str
    # credentials.password: str
    ...
```

### HTTPBearer

```python
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials

security = HTTPBearer(
    bearerFormat: str | None = None,              # Token format description
    scheme_name: str | None = None,
    description: str | None = None,
    auto_error: bool = True,
)

@app.get("/items/")
async def read_items(
    credentials: Annotated[HTTPAuthorizationCredentials, Depends(security)],
):
    # credentials.scheme: str        (e.g., "Bearer")
    # credentials.credentials: str   (the token)
    ...
```

### HTTPDigest

```python
from fastapi.security import HTTPDigest

security = HTTPDigest(
    scheme_name: str | None = None,
    description: str | None = None,
    auto_error: bool = True,
)
# Note: Stub implementation — needs subclassing for full Digest auth
```

### API Key Authentication

```python
from fastapi.security import APIKeyQuery, APIKeyHeader, APIKeyCookie

# API key in query parameter
api_key_query = APIKeyQuery(
    name: str,                                    # Query param name
    scheme_name: str | None = None,
    description: str | None = None,
    auto_error: bool = True,
)

# API key in header
api_key_header = APIKeyHeader(
    name: str,                                    # Header name
    scheme_name: str | None = None,
    description: str | None = None,
    auto_error: bool = True,
)

# API key in cookie
api_key_cookie = APIKeyCookie(
    name: str,                                    # Cookie name
    scheme_name: str | None = None,
    description: str | None = None,
    auto_error: bool = True,
)

# All return: str | None (the API key value)
```

### OpenID Connect

```python
from fastapi.security import OpenIdConnect

oidc = OpenIdConnect(
    openIdConnectUrl: str,                        # OpenID Connect config URL
    scheme_name: str | None = None,
    description: str | None = None,
    auto_error: bool = True,
)
# Note: Stub implementation — needs subclassing
```

### Security (with scopes)

**Import:** `from fastapi import Security`

```python
@dataclass(frozen=True)
class Security(Depends):
    scopes: Sequence[str] | None = None
```

### SecurityScopes

```python
from fastapi.security import SecurityScopes

# Injected automatically when declared as a parameter
async def get_current_user(
    token: Annotated[str, Depends(oauth2_scheme)],
    security_scopes: SecurityScopes,
):
    # security_scopes.scopes: list[str]   (all required scopes)
    # security_scopes.scope_str: str       (space-separated)
    ...

@app.get("/items/", dependencies=[Security(get_current_user, scopes=["items:read"])])
async def read_items():
    ...
```

---

## 13. Request & Response

### Request

**Import:** `from fastapi import Request`

Re-exported from Starlette. Key attributes:

```python
request.method: str                  # HTTP method
request.url: URL                     # Full URL
request.headers: Headers             # Request headers
request.query_params: QueryParams    # Query parameters
request.path_params: dict            # Path parameters
request.cookies: dict                # Cookies
request.client: Address | None       # Client host and port
request.state: State                 # Per-request state
request.app: FastAPI                 # Application instance

# Body methods
await request.body() -> bytes
await request.json() -> Any
await request.form() -> FormData
await request.stream() -> AsyncGenerator[bytes, None]
await request.close() -> None
```

### Response Classes

**Import:** `from fastapi.responses import ...`

| Class | Content-Type | Description |
|-------|-------------|-------------|
| `JSONResponse` | `application/json` | Default. Serializes with `json.dumps` |
| `HTMLResponse` | `text/html` | Returns HTML content |
| `PlainTextResponse` | `text/plain` | Returns plain text |
| `RedirectResponse` | — | HTTP redirect (301/302/307/308) |
| `StreamingResponse` | varies | Streaming body from async generator |
| `FileResponse` | varies | Serves a file from disk |
| `Response` | `text/plain` | Base response class |
| `ORJSONResponse` | `application/json` | Fast JSON with `orjson` |
| `UJSONResponse` | `application/json` | Fast JSON with `ujson` |
| `EventSourceResponse` | `text/event-stream` | Server-Sent Events |

### Response Object

```python
from fastapi import Response

@app.get("/items/")
async def read_items(response: Response):
    # Modify the response before returning
    response.headers["X-Custom"] = "value"
    response.set_cookie(key="session", value="abc123")
    response.status_code = 201
    return {"message": "created"}
```

### Returning a Response Directly

```python
from fastapi.responses import JSONResponse, RedirectResponse, StreamingResponse

@app.get("/redirect")
async def redirect():
    return RedirectResponse(url="/new-location", status_code=307)

@app.get("/custom")
async def custom():
    return JSONResponse(
        content={"message": "hello"},
        status_code=200,
        headers={"X-Custom": "value"},
    )

@app.get("/stream")
async def stream():
    async def generate():
        for i in range(10):
            yield f"chunk {i}\n"
    return StreamingResponse(generate(), media_type="text/plain")
```

### Setting the Response Class for a Route

```python
@app.get("/items/", response_class=HTMLResponse)
async def read_items():
    return "<html><body>Hello</body></html>"

@app.get("/items/", response_class=ORJSONResponse)
async def read_items():
    return [{"item": "Foo"}]
```

---

## 14. HTTPException & Error Handling

### HTTPException

**Import:** `from fastapi import HTTPException`

**Source:** `fastapi/exceptions.py`

```python
class HTTPException(StarletteHTTPException):
    def __init__(
        self,
        status_code: int,
        detail: Any = None,                        # Can be str, dict, list, etc.
        headers: Mapping[str, str] | None = None,
    ) -> None
```

```python
raise HTTPException(status_code=404, detail="Item not found")
raise HTTPException(
    status_code=403,
    detail="Not authorized",
    headers={"WWW-Authenticate": "Bearer"},
)
```

### WebSocketException

```python
from fastapi import WebSocketException

class WebSocketException(StarletteWebSocketException):
    def __init__(
        self,
        code: int,                                 # WebSocket close code
        reason: str | None = None,
    ) -> None
```

### RequestValidationError

**Import:** `from fastapi.exceptions import RequestValidationError`

```python
class RequestValidationError(ValidationException):
    def __init__(
        self,
        errors: Sequence[Any],
        *,
        body: Any = None,
    ) -> None
```

Raised automatically when request validation fails. Default handler returns 422 with:

```json
{
    "detail": [
        {
            "loc": ["body", "price"],
            "msg": "value is not a valid float",
            "type": "type_error.float"
        }
    ]
}
```

### Custom Exception Handlers

```python
from fastapi.exceptions import RequestValidationError
from fastapi.responses import PlainTextResponse

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request, exc):
    return PlainTextResponse(str(exc.errors()), status_code=422)

@app.exception_handler(HTTPException)
async def http_exception_handler(request, exc):
    return PlainTextResponse(str(exc.detail), status_code=exc.status_code)

# Custom exception
class UnicornException(Exception):
    def __init__(self, name: str):
        self.name = name

@app.exception_handler(UnicornException)
async def unicorn_exception_handler(request, exc):
    return JSONResponse(
        status_code=418,
        content={"message": f"Oops! {exc.name} did something."},
    )
```

---

## 15. Background Tasks

**Import:** `from fastapi import BackgroundTasks`

**Source:** `fastapi/background.py`

```python
class BackgroundTasks(StarletteBackgroundTasks):
    def add_task(
        self,
        func: Callable[P, Any],   # Sync or async callable
        *args: P.args,
        **kwargs: P.kwargs,
    ) -> None
```

Tasks run **after** the response is sent.

### Usage

```python
def write_log(message: str):
    with open("log.txt", "a") as f:
        f.write(message + "\n")

@app.post("/send-notification/")
async def send_notification(
    email: str,
    background_tasks: BackgroundTasks,
):
    background_tasks.add_task(write_log, f"Notification sent to {email}")
    return {"message": "Notification will be sent"}
```

### In Dependencies

```python
def write_query_log(background_tasks: BackgroundTasks, q: str | None = None):
    if q:
        background_tasks.add_task(write_log, q)

@app.post("/items/", dependencies=[Depends(write_query_log)])
async def create_item(item: Item):
    return item
```

---

## 16. WebSockets

**Import:** `from fastapi import WebSocket, WebSocketDisconnect`

**Source:** `fastapi/websockets.py` (re-exported from Starlette)

### Decorator

```python
@app.websocket(
    path: str,
    name: str | None = None,
    *,
    dependencies: Sequence[Depends] | None = None,
)
```

### WebSocket Object

```python
# Connection lifecycle
await websocket.accept(subprotocol: str | None = None, headers: Iterable | None = None)
await websocket.close(code: int = 1000, reason: str | None = None)

# Receiving data
await websocket.receive_text() -> str
await websocket.receive_bytes() -> bytes
await websocket.receive_json(mode: str = "text") -> Any

# Sending data
await websocket.send_text(data: str)
await websocket.send_bytes(data: bytes)
await websocket.send_json(data: Any, mode: str = "text")

# Iteration
async for data in websocket.iter_text():
    ...
async for data in websocket.iter_bytes():
    ...
async for data in websocket.iter_json():
    ...
```

### Usage

```python
@app.websocket("/ws")
async def websocket_endpoint(websocket: WebSocket):
    await websocket.accept()
    try:
        while True:
            data = await websocket.receive_text()
            await websocket.send_text(f"Echo: {data}")
    except WebSocketDisconnect:
        print("Client disconnected")
```

### WebSocket with Dependencies

```python
async def get_token(websocket: WebSocket, token: str | None = Query(default=None)):
    if token is None:
        raise WebSocketException(code=1008, reason="No token")
    return token

@app.websocket("/ws")
async def websocket_endpoint(
    websocket: WebSocket,
    token: Annotated[str, Depends(get_token)],
):
    await websocket.accept()
    ...
```

---

## 17. Middleware

### CORS Middleware

**Import:** `from fastapi.middleware.cors import CORSMiddleware`

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins: Sequence[str] = (),           # Origins allowed (["*"] for all)
    allow_methods: Sequence[str] = ("GET",),     # HTTP methods allowed
    allow_headers: Sequence[str] = (),           # Headers allowed
    allow_credentials: bool = False,             # Allow cookies/auth
    allow_origin_regex: str | None = None,       # Regex for origins
    expose_headers: Sequence[str] = (),          # Headers exposed to browser
    max_age: int = 600,                          # Preflight cache (seconds)
)
```

### GZip Middleware

**Import:** `from fastapi.middleware.gzip import GZipMiddleware`

```python
app.add_middleware(
    GZipMiddleware,
    minimum_size: int = 500,                     # Min bytes to compress
    compresslevel: int = 9,                      # Compression level 1-9
)
```

### Trusted Host Middleware

**Import:** `from fastapi.middleware.trustedhost import TrustedHostMiddleware`

```python
app.add_middleware(
    TrustedHostMiddleware,
    allowed_hosts: Sequence[str] = ["*"],        # e.g., ["example.com", "*.example.com"]
    www_redirect: bool = True,
)
```

### HTTPS Redirect Middleware

**Import:** `from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware`

```python
app.add_middleware(HTTPSRedirectMiddleware)
```

### Custom HTTP Middleware

```python
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
    import time
    start = time.perf_counter()
    response = await call_next(request)
    duration = time.perf_counter() - start
    response.headers["X-Process-Time"] = str(duration)
    return response
```

### Middleware via Constructor

```python
from starlette.middleware import Middleware

app = FastAPI(
    middleware=[
        Middleware(CORSMiddleware, allow_origins=["*"]),
        Middleware(GZipMiddleware, minimum_size=1000),
    ]
)
```

### Middleware Execution Order

Middleware is applied in **LIFO** order — the last middleware added wraps the outermost layer:

1. `ServerErrorMiddleware` (outermost, always present)
2. User middleware (in reverse order of addition)
3. `ExceptionMiddleware`
4. `AsyncExitStackMiddleware` (innermost)

---

## 18. Lifespan Events

### Recommended: Lifespan Context Manager

```python
from contextlib import asynccontextmanager

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Startup: runs before the application starts receiving requests
    db = await create_db_pool()
    app.state.db = db
    yield
    # Shutdown: runs when the application is shutting down
    await db.close()

app = FastAPI(lifespan=lifespan)
```

### Deprecated: Event Handlers

```python
@app.on_event("startup")
async def startup():
    ...

@app.on_event("shutdown")
async def shutdown():
    ...
```

### Via Constructor (deprecated)

```python
app = FastAPI(
    on_startup=[startup_func1, startup_func2],
    on_shutdown=[shutdown_func1],
)
```

---

## 19. Static Files & Templates

### StaticFiles

**Import:** `from fastapi.staticfiles import StaticFiles`

```python
# Mount static files directory
app.mount("/static", StaticFiles(directory="static"), name="static")

# With HTML fallback (for SPAs)
app.mount("/", StaticFiles(directory="build", html=True), name="spa")
```

### Jinja2Templates

**Import:** `from fastapi.templating import Jinja2Templates`

```python
templates = Jinja2Templates(directory="templates")

@app.get("/items/{id}", response_class=HTMLResponse)
async def read_item(request: Request, id: int):
    return templates.TemplateResponse(
        request=request,
        name="item.html",
        context={"id": id},
    )
```

Template (`templates/item.html`):
```html
<html>
<body>
    <h1>Item {{ id }}</h1>
    <a href="{{ url_for('read_item', id=id) }}">Self</a>
</body>
</html>
```

---

## 20. OpenAPI & Documentation

### Disabling Documentation

```python
# Disable all docs
app = FastAPI(openapi_url=None)

# Disable only Swagger UI
app = FastAPI(docs_url=None)

# Disable only ReDoc
app = FastAPI(redoc_url=None)
```

### OpenAPI Tags

```python
tags_metadata = [
    {"name": "users", "description": "Operations with users."},
    {"name": "items", "description": "Manage items.", "externalDocs": {
        "description": "Items docs",
        "url": "https://example.com/items",
    }},
]

app = FastAPI(openapi_tags=tags_metadata)
```

### Custom Swagger UI

```python
from fastapi.openapi.docs import get_swagger_ui_html

get_swagger_ui_html(
    *,
    openapi_url: str,                              # REQUIRED
    title: str,                                    # REQUIRED
    swagger_js_url: str = "https://cdn.jsdelivr.net/npm/swagger-ui-dist@5/swagger-ui-bundle.js",
    swagger_css_url: str = "https://cdn.jsdelivr.net/npm/swagger-ui-dist@5/swagger-ui.css",
    swagger_favicon_url: str = "https://fastapi.tiangolo.com/img/favicon.png",
    oauth2_redirect_url: str | None = None,
    init_oauth: dict[str, Any] | None = None,
    swagger_ui_parameters: dict[str, Any] | None = None,
) -> HTMLResponse
```

### Custom ReDoc

```python
from fastapi.openapi.docs import get_redoc_html

get_redoc_html(
    *,
    openapi_url: str,                              # REQUIRED
    title: str,                                    # REQUIRED
    redoc_js_url: str = "https://cdn.jsdelivr.net/npm/redoc@2/bundles/redoc.standalone.js",
    redoc_favicon_url: str = "https://fastapi.tiangolo.com/img/favicon.png",
    with_google_fonts: bool = True,
) -> HTMLResponse
```

### Self-Hosting Docs (No CDN)

```python
app = FastAPI(docs_url=None, redoc_url=None)

@app.get("/docs", include_in_schema=False)
async def custom_swagger_ui_html():
    return get_swagger_ui_html(
        openapi_url=app.openapi_url,
        title=app.title + " - Swagger UI",
        swagger_js_url="/static/swagger-ui-bundle.js",
        swagger_css_url="/static/swagger-ui.css",
    )
```

### Swagger UI Parameters

```python
app = FastAPI(
    swagger_ui_parameters={
        "deepLinking": True,
        "defaultModelsExpandDepth": -1,
        "docExpansion": "none",
        "filter": True,
        "operationsSorter": "alpha",
        "tagsSorter": "alpha",
        "tryItOutEnabled": True,
    }
)
```

### Swagger UI Default Parameters

```python
swagger_ui_default_parameters = {
    "dom_id": "#swagger-ui",
    "layout": "BaseLayout",
    "deepLinking": True,
    "showExtensions": True,
    "showCommonExtensions": True,
}
```

---

## 21. JSON Encoding

**Import:** `from fastapi.encoders import jsonable_encoder`

**Source:** `fastapi/encoders.py`

```python
def jsonable_encoder(
    obj: Any,
    include: IncEx | None = None,
    exclude: IncEx | None = None,
    by_alias: bool = True,
    exclude_unset: bool = False,
    exclude_defaults: bool = False,
    exclude_none: bool = False,
    custom_encoder: dict[Any, Callable[[Any], Any]] | None = None,
    sqlalchemy_safe: bool = True,
) -> Any
```

Converts arbitrary objects to JSON-serializable types. Handles:
- Pydantic models (via `model_dump`)
- Dataclasses
- Enums, Dates, Datetimes, Timedeltas
- Decimals, UUIDs, Paths
- IP addresses
- Sets, frozensets, deques, generators
- bytes (base64 encoded)

### Usage

```python
from fastapi.encoders import jsonable_encoder

@app.put("/items/{id}")
async def update_item(id: str, item: Item):
    json_compatible_data = jsonable_encoder(item)
    # Use with database, cache, etc.
    return json_compatible_data

# With filtering
data = jsonable_encoder(item, exclude_unset=True)
data = jsonable_encoder(item, include={"name", "price"})
data = jsonable_encoder(item, exclude={"tax"})
```

---

## 22. Sub-Applications & Mounting

### Sub-Applications (independent OpenAPI)

```python
app = FastAPI()
subapi = FastAPI()

@subapi.get("/items/")
async def read_items():
    return [{"item": "Foo"}]

# Mount creates a separate app with its own /docs
app.mount("/v2", subapi)
```

Key differences from `include_router`:
- Sub-apps have **independent** OpenAPI schemas
- Each sub-app has its own `/docs` and `/redoc`
- `root_path` is automatically handled

### Mounting Non-FastAPI Apps

```python
# WSGI app (e.g., Flask)
from fastapi.middleware.wsgi import WSGIMiddleware

flask_app = Flask(__name__)
app.mount("/flask", WSGIMiddleware(flask_app))

# Static files
app.mount("/static", StaticFiles(directory="static"), name="static")
```

---

## 23. Testing

**Import:** `from fastapi.testclient import TestClient`

Re-exported from Starlette. Uses `httpx` under the hood.

### Basic Testing

```python
from fastapi.testclient import TestClient

client = TestClient(app)

# GET
response = client.get("/items/", params={"skip": 0, "limit": 10})
assert response.status_code == 200
assert response.json() == [...]

# POST with JSON body
response = client.post("/items/", json={"name": "Foo", "price": 42.0})
assert response.status_code == 200

# POST with form data
response = client.post("/login/", data={"username": "foo", "password": "bar"})

# POST with file upload
with open("test.txt", "rb") as f:
    response = client.post("/upload/", files={"file": ("test.txt", f, "text/plain")})

# With headers
response = client.get("/items/", headers={"X-Token": "secret"})

# With cookies
response = client.get("/items/", cookies={"session": "abc123"})
```

### WebSocket Testing

```python
with client.websocket_connect("/ws") as websocket:
    websocket.send_text("hello")
    data = websocket.receive_text()
    assert data == "Echo: hello"
```

### Async Testing (with httpx)

```python
import pytest
from httpx import ASGITransport, AsyncClient

@pytest.mark.anyio
async def test_read_items():
    async with AsyncClient(
        transport=ASGITransport(app=app),
        base_url="http://test",
    ) as client:
        response = await client.get("/items/")
        assert response.status_code == 200
```

### Overriding Dependencies in Tests

```python
from fastapi.testclient import TestClient

def override_get_db():
    db = TestSessionLocal()
    try:
        yield db
    finally:
        db.close()

app.dependency_overrides[get_db] = override_get_db
client = TestClient(app)

# After tests, clean up
app.dependency_overrides.clear()
```

---

## 24. Status Codes

**Import:** `from fastapi import status`

Re-exported from Starlette. Constants follow the pattern `HTTP_{code}_{name}`:

### Informational (1xx)

| Constant | Value |
|----------|-------|
| `status.HTTP_100_CONTINUE` | 100 |
| `status.HTTP_101_SWITCHING_PROTOCOLS` | 101 |

### Success (2xx)

| Constant | Value |
|----------|-------|
| `status.HTTP_200_OK` | 200 |
| `status.HTTP_201_CREATED` | 201 |
| `status.HTTP_202_ACCEPTED` | 202 |
| `status.HTTP_204_NO_CONTENT` | 204 |

### Redirection (3xx)

| Constant | Value |
|----------|-------|
| `status.HTTP_301_MOVED_PERMANENTLY` | 301 |
| `status.HTTP_302_FOUND` | 302 |
| `status.HTTP_304_NOT_MODIFIED` | 304 |
| `status.HTTP_307_TEMPORARY_REDIRECT` | 307 |
| `status.HTTP_308_PERMANENT_REDIRECT` | 308 |

### Client Error (4xx)

| Constant | Value |
|----------|-------|
| `status.HTTP_400_BAD_REQUEST` | 400 |
| `status.HTTP_401_UNAUTHORIZED` | 401 |
| `status.HTTP_403_FORBIDDEN` | 403 |
| `status.HTTP_404_NOT_FOUND` | 404 |
| `status.HTTP_405_METHOD_NOT_ALLOWED` | 405 |
| `status.HTTP_409_CONFLICT` | 409 |
| `status.HTTP_422_UNPROCESSABLE_ENTITY` | 422 |
| `status.HTTP_429_TOO_MANY_REQUESTS` | 429 |

### Server Error (5xx)

| Constant | Value |
|----------|-------|
| `status.HTTP_500_INTERNAL_SERVER_ERROR` | 500 |
| `status.HTTP_502_BAD_GATEWAY` | 502 |
| `status.HTTP_503_SERVICE_UNAVAILABLE` | 503 |
| `status.HTTP_504_GATEWAY_TIMEOUT` | 504 |

### WebSocket

| Constant | Value |
|----------|-------|
| `status.WS_1000_NORMAL_CLOSURE` | 1000 |
| `status.WS_1001_GOING_AWAY` | 1001 |
| `status.WS_1008_POLICY_VIOLATION` | 1008 |
| `status.WS_1011_SERVER_ERROR` | 1011 |

---

## 25. Common Patterns

### Parameter Type Hierarchy

```
FieldInfo (Pydantic base)
├── Param (FastAPI base for URL/header/cookie params)
│   ├── Path    — in_: path
│   ├── Query   — in_: query
│   ├── Header  — in_: header  (+convert_underscores)
│   └── Cookie  — in_: cookie
├── Body (JSON body)
│   ├── Form    — media_type: application/x-www-form-urlencoded
│   └── File    — media_type: multipart/form-data
└── Depends (dependency injection)
    └── Security (+scopes for OAuth2)
```

### Shared Validation Parameters

All `Path`, `Query`, `Header`, `Cookie`, `Body`, `Form`, `File` share:

| Parameter | Type | Description |
|-----------|------|-------------|
| `default` | `Any` | Default value (`...` = required) |
| `alias` | `str` | Alternative name for extraction |
| `title` | `str` | Title in OpenAPI schema |
| `description` | `str` | Description in OpenAPI schema |
| `gt` | `float` | Greater than |
| `ge` | `float` | Greater than or equal |
| `lt` | `float` | Less than |
| `le` | `float` | Less than or equal |
| `min_length` | `int` | Minimum string length |
| `max_length` | `int` | Maximum string length |
| `pattern` | `str` | Regex pattern for strings |
| `strict` | `bool` | Strict type validation |
| `multiple_of` | `float` | Must be a multiple of this value |
| `examples` | `list` | Example values for OpenAPI |
| `openapi_examples` | `dict` | Named examples with descriptions |
| `deprecated` | `bool\|str` | Mark as deprecated |
| `include_in_schema` | `bool` | Include in OpenAPI schema |
| `json_schema_extra` | `dict` | Extra JSON Schema properties |

### Annotated Pattern (Preferred)

```python
from typing import Annotated
from fastapi import Depends, Path, Query

# Type + metadata together — reusable and clean
CurrentUser = Annotated[User, Depends(get_current_user)]
ItemId = Annotated[int, Path(ge=1)]
Pagination = Annotated[int, Query(ge=0, le=100)]

@app.get("/items/{item_id}")
async def read_item(
    item_id: ItemId,
    user: CurrentUser,
    skip: Pagination = 0,
    limit: Pagination = 10,
):
    ...
```

### Application Structure (Larger Apps)

```
app/
├── main.py              # FastAPI app, include routers
├── dependencies.py      # Shared dependencies
├── routers/
│   ├── __init__.py
│   ├── items.py         # router = APIRouter(prefix="/items", tags=["items"])
│   └── users.py         # router = APIRouter(prefix="/users", tags=["users"])
├── models/
│   ├── __init__.py
│   └── item.py          # Pydantic models
├── crud/
│   └── item.py          # Database operations
└── core/
    ├── config.py        # Settings (pydantic-settings)
    └── security.py      # Auth utilities
```

### Configuration with pydantic-settings

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "My API"
    admin_email: str
    database_url: str

    model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
app = FastAPI(title=settings.app_name)
```

### Server-Sent Events (SSE)

```python
from fastapi.responses import EventSourceResponse
# or
from sse_starlette.sse import EventSourceResponse

@app.get("/stream")
async def stream():
    async def event_generator():
        for i in range(10):
            yield {"data": f"message {i}"}
    return EventSourceResponse(event_generator())
```

### Key Imports Summary

```python
# Core
from fastapi import FastAPI, APIRouter, Request, Response

# Parameters
from fastapi import Path, Query, Header, Cookie, Body, Form, File

# Dependency Injection
from fastapi import Depends, Security

# Data Types
from fastapi import UploadFile, BackgroundTasks

# WebSocket
from fastapi import WebSocket, WebSocketDisconnect

# Errors
from fastapi import HTTPException, status
from fastapi.exceptions import RequestValidationError

# Responses
from fastapi.responses import (
    JSONResponse, HTMLResponse, PlainTextResponse,
    RedirectResponse, StreamingResponse, FileResponse,
    ORJSONResponse, UJSONResponse,
)

# Security
from fastapi.security import (
    OAuth2PasswordBearer, OAuth2PasswordRequestForm,
    OAuth2AuthorizationCodeBearer,
    HTTPBasic, HTTPBasicCredentials,
    HTTPBearer, HTTPAuthorizationCredentials,
    APIKeyQuery, APIKeyHeader, APIKeyCookie,
    OpenIdConnect, SecurityScopes,
)

# Middleware
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.gzip import GZipMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.middleware.httpsredirect import HTTPSRedirectMiddleware

# Utilities
from fastapi.encoders import jsonable_encoder
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from fastapi.testclient import TestClient
```
