# Robyn Framework Reference for Claude Code

> Robyn is a high-performance Python web framework with a Rust runtime. Python handles business logic; Rust handles HTTP parsing, routing, and I/O. Install: `uv pip install robyn`

## App Setup & Startup

```python
from robyn import Robyn, Request, Response

app = Robyn(__file__)

# Start options
app.start(port=8080, host="0.0.0.0")  # host defaults to 127.0.0.1
```

### CLI Flags

```bash
python app.py --dev                    # Dev mode with auto-reload
python app.py --fast                   # Auto-optimized multi-core
python app.py --processes 4 --workers 2  # Manual scaling
python app.py --disable-openapi        # Disable /docs and /openapi.json
python app.py --log-level INFO
python app.py --compile-rust-path "."  # Compile inline Rust files
python app.py --create                 # Scaffold new project via CLI
```

### Timeout Configuration

```python
app.start(
    client_timeout=30,       # Max seconds for client request processing
    keep_alive_timeout=20,   # Seconds to keep idle connections alive
)
# Override via env: ROBYN_CLIENT_TIMEOUT, ROBYN_KEEP_ALIVE_TIMEOUT
```

### Environment Config (`robyn.env`)

Place at project root. Loaded automatically.

```bash
ROBYN_PORT=8080
ROBYN_HOST=127.0.0.1
ROBYN_BROWSER_OPEN=True
ROBYN_DEV_MODE=True
ROBYN_MAX_PAYLOAD_SIZE=1000000  # bytes
ROBYN_DISABLE_OPENAPI=true      # Disable /docs and /openapi.json
```

---

## Routing

### Basic Routes (sync and async)

```python
@app.get("/")
def index(request: Request):
    return "Hello, world"

@app.post("/data")
async def create(request: Request):
    return {"created": True}

# All HTTP methods: app.get, app.post, app.put, app.patch, app.delete
```

### Path Parameters

```python
@app.get("/users/:id")
def get_user(request: Request):
    user_id = request.path_params["id"]
    return {"id": user_id}

# Multiple params
@app.get("/users/:user_id/posts/:post_id")

# Wildcard/extra path capture
@app.get("/files/*extra")
def serve(request: Request):
    filepath = request.path_params["extra"]  # "foo/bar/baz"
```

### Const Routes (cached in Rust, bypasses Python)

```python
@app.get("/health", const=True)
def health():
    return {"status": "healthy"}
```

### Redirects

```python
@app.get("/")
async def index():
    return Response(status_code=307, description="", headers={"Location": "/landing"})
```

---

## Request Object

```python
@dataclass
class Request:
    query_params: QueryParams   # NOT a dict! Use .to_dict(), .get(), .empty()
    headers: Headers            # dict-like, .get() has NO default arg, .set() to modify
    path_params: dict[str, str] # from :param in URL
    body: Union[str, bytes]     # raw body
    method: str                 # GET, POST, etc.
    url: Url                    # url.path, url.host, url.scheme (NOT a string!)
    form_data: dict[str, str]
    files: dict[str, bytes]
    ip_addr: Optional[str]
    identity: Optional[Identity]  # set by auth middleware
```

**IMPORTANT**: `QueryParams` is NOT a dict. Do NOT use `dict(request.query_params)` or
iterate it directly. Use `request.query_params.to_dict()` to convert, `.get(key, default)`
to read a single value, `.empty()` to check if empty.

**CRITICAL**: Both `QueryParams.get()` and `Headers.get()` REQUIRE a default argument.
`request.query_params.get('key')` raises `TypeError`. Always use
`request.query_params.get('key', '')` or `request.headers.get('key') or ''`.

### Parsing JSON Body

```python
data = request.json()  # Returns dict with full type preservation (null->None, etc.)
# Raises ValueError if not valid JSON
```

---

## Parameter Injection

Robyn introspects handler signatures and auto-injects parameters.

### Type-Based Injection

```python
from robyn import Request, QueryParams, Headers
from robyn.types import PathParams, RequestBody, RequestMethod, RequestURL, FormData, RequestFiles, RequestIP, RequestIdentity

@app.post("/users/:user_id")
async def handler(
    request: Request,          # Full request object
    path_params: PathParams,   # {"user_id": "123"}
    query_params: QueryParams, # ?key=value
    headers: Headers,
    body: RequestBody,         # Raw body
    method: RequestMethod,     # "POST"
):
    ...
```

### Name-Based Injection (no type annotation needed)

Reserved names: `request`/`req`/`r`, `query_params`, `headers`, `path_params`, `body`, `method`, `url`, `form_data`, `files`, `ip_addr`, `identity`

### Easy Access Parameters (auto type coercion)

```python
@app.get("/items/:id")
async def get_item(id: int, q: str, page: int = 1):
    # id coerced from path param to int, q from query string, page defaults to 1
    return {"id": id, "q": q, "page": page}

# Supported types: int, float, str, bool, Optional[T], List[T]
# Bool accepts: true/false, 1/0, yes/no, on/off
# Missing required params or coercion failure -> 400 Bad Request
```

---

## Response Types

```python
# String
return "Hello"

# Dict (auto JSON)
return {"key": "value"}

# Explicit Response
return Response(status_code=200, description="body", headers={"X-Custom": "value"})

# HTML
from robyn import serve_html, html
return serve_html("./index.html")
return html("<h1>Hello</h1>")

# File download
from robyn import serve_file
return serve_file("./report.pdf", file_name="report.pdf")

# Serve static directory (e.g. React build)
app.serve_directory(route="/static", directory_path="./build", index_file="index.html")
```

---

## Middlewares & Events

### Before/After Request

```python
@app.before_request("/")
async def before(request: Request):
    request.headers.set("X-Custom", "value")
    return request  # MUST return request (or a Response to short-circuit)

@app.after_request("/")
def after(response: Response):
    response.headers.set("X-Processed", "true")
    return response  # MUST return response

# after_request can also accept request as first param:
@app.after_request("/")
def after(request: Request, response: Response):
    response.headers.set("request_path", request.url.path)
    return response
```

**Note:** If any `before_request` returns a `Response` object, the chain stops and that response is sent immediately.

### Startup/Shutdown Events

```python
@app.startup_handler
async def on_startup():
    print("Starting up")

@app.shutdown_handler
def on_shutdown():
    print("Shutting down")
```

---

## Authentication

```python
from robyn.authentication import AuthenticationHandler, BearerGetter, Identity

class MyAuth(AuthenticationHandler):
    def authenticate(self, request: Request) -> Optional[Identity]:
        token = self.token_getter.get_token(request)
        if token == "valid":
            return Identity(claims={"user_id": "123"})
        return None  # Return None -> 401

app.configure_authentication(MyAuth(token_getter=BearerGetter()))

@app.get("/protected", auth_required=True)
def protected(request: Request):
    user = request.identity.claims["user_id"]
    return f"Hello {user}"
```

The `BearerGetter` extracts the token from `Authorization: Bearer <token>` header. Authentication is implemented as a `before_request` middleware under the hood.

---

## CORS

```python
from robyn import ALLOW_CORS

ALLOW_CORS(app, origins=["http://localhost:3000"])
```

---

## SubRouters

```python
from robyn import SubRouter

api = SubRouter(__file__, prefix="/api/v1")

@api.get("/users")
def list_users():
    return {"users": []}

@api.post("/users")
def create_user(body):
    return {"created": True}

app.include_router(api)
# Routes: /api/v1/users (GET, POST)
```

SubRouters support their own `configure_authentication()` and `auth_required=True` on individual routes.

---

## Exception Handling

```python
@app.exception
def handle_exception(error: Exception):
    return Response(status_code=500, description=f"Error: {error}", headers={})
```

---

## Dependency Injection

```python
# Global (available to all routes)
app.inject_global(DB_URL="sqlite:///app.db")

@app.get("/")
def handler(request, global_dependencies):
    return global_dependencies["DB_URL"]

# Router-level (available to routes on that router)
app.inject(cache=my_cache)

@app.get("/")
def handler(request, router_dependencies):
    return router_dependencies["cache"]
```

**Reserved param names:** `global_dependencies`, `router_dependencies` (must come after `request`).

Works with WebSockets too (handler, on_connect, on_close).

---

## WebSockets

```python
from robyn import WebSocketDisconnect

@app.websocket("/ws")
async def handler(websocket):
    try:
        while True:
            msg = await websocket.receive_text()
            await websocket.send_text(f"Echo: {msg}")
    except WebSocketDisconnect:
        print(f"Client {websocket.id} disconnected")

@handler.on_connect
def on_connect(websocket):
    return "Welcome!"  # Sent as first message to client

@handler.on_close
def on_close(websocket):
    return "Goodbye!"  # Sent as final message
```

### WebSocket API

| Method/Property | Description |
|---|---|
| `await websocket.receive_text()` | Block until next message; raises `WebSocketDisconnect` on close |
| `await websocket.receive_bytes()` | Binary message variant |
| `await websocket.receive_json()` | JSON-decoded variant |
| `await websocket.send_text(data)` | Send string to this client |
| `await websocket.send_bytes(data)` | Send binary to this client |
| `await websocket.send_json(data)` | Send JSON to this client |
| `await websocket.broadcast(data)` | Send to ALL clients on this endpoint |
| `await websocket.close()` | Close connection server-side |
| `websocket.id` | Connection UUID string |
| `websocket.query_params` | Query params from connection URL |

### WebSocket Easy Access Params

```python
@app.websocket("/ws")
async def handler(websocket, room: str = "default", page: int = 1):
    ...  # room and page auto-resolved from query params
```

---

## Server-Sent Events (SSE)

```python
from robyn import SSEResponse, SSEMessage

@app.get("/events")
def stream(request):
    def gen():
        for i in range(10):
            yield SSEMessage(f"Event {i}", event="update", id=str(i))
            time.sleep(1)
    return SSEResponse(gen())

# Async generators also supported
@app.get("/events/async")
async def stream_async(request):
    async def gen():
        for i in range(5):
            await asyncio.sleep(0.5)
            yield SSEMessage(json.dumps({"i": i}), event="data", id=str(i))
    return SSEResponse(gen())
```

---

## Pydantic Integration

Install: `pip install "robyn[pydantic]"` or `pip install "robyn[all]"`

```python
from pydantic import BaseModel

class UserCreate(BaseModel):
    name: str
    email: str
    age: int
    active: bool = True

@app.post("/users")
def create_user(user: UserCreate):
    return {"name": user.name, "email": user.email}

# Validation failure -> 422 with structured error details
# Nested models supported
# Can combine with Request: def handler(request: Request, user: UserCreate)
# Can return Pydantic models directly (auto-serialized to JSON)
# Auto-generates rich OpenAPI schema
```

**Pydantic vs Body:** Use Pydantic for automatic validation + rich OpenAPI. Use `Body` (built-in) for basic OpenAPI schema without validation overhead.

---

## OpenAPI / Swagger

Built-in, enabled by default:
- `/docs` - Swagger UI
- `/openapi.json` - JSON spec

```python
from robyn import Robyn
from robyn.robyn import QueryParams

app = Robyn(
    file_object=__file__,
    openapi=OpenAPI(
        info=OpenAPIInfo(title="My API", version="1.0.0", description="..."),
    ),
)

class SearchParams(QueryParams):
    q: str
    page: int

@app.get("/search", openapi_name="Search", openapi_tags=["Search"])
def search(r: Request, query_params: SearchParams):
    """Search endpoint"""  # Docstring becomes description
    return r.query_params
```

### Request/Response Body Typing for OpenAPI

```python
from robyn.types import JSONResponse, Body

class CreateItemBody(Body):
    name: str
    price: float

class CreateResponse(JSONResponse):
    success: bool

@app.post("/items")
def create(request: Request, body: CreateItemBody) -> CreateResponse:
    return CreateResponse(success=True)
```

---

## Templating (Jinja2)

```python
from robyn.templating import JinjaTemplate
import pathlib, os

current_dir = pathlib.Path(__file__).parent.resolve()
jinja = JinjaTemplate(os.path.join(current_dir, "templates"))

@app.get("/page")
def page():
    return jinja.render_template("index.html", framework="Robyn", version="1.0")
```

Custom engines: subclass `TemplateInterface` and implement `render_template()`.

---

## File Uploads

```python
# Raw body upload
@app.post("/upload")
async def upload(request: Request):
    file = bytearray(request.body)
    with open("uploaded.bin", "wb") as f:
        f.write(file)
    return {"status": "ok"}

# Multipart form data
@app.post("/upload-multi")
def upload_multi(request: Request):
    files = request.files       # dict[FILENAME, bytes] — key is the FILENAME, not form field name!
    form = request.form_data    # dict[str, str]
    return {"files": list(files.keys())}
# IMPORTANT: request.files keys are FILENAMES (e.g. 'photo.jpg'), NOT form field names.
# To get the first uploaded file: filename = next(iter(request.files)); data = request.files[filename]
```

---

## Logging

```python
from robyn import Logger

logger = Logger(app)

@app.before_request()
async def log_req(request: Request):
    logger.info("Received: %s", request)
```

---

## MCP (Model Context Protocol) - Experimental

```python
@app.mcp.resource("user://{user_id}/profile")
def user_profile(user_id: str) -> str:
    return f"Profile for {user_id}"

@app.mcp.tool()
def greet(name: str, formal: bool = False) -> str:
    return f"Good day, {name}." if formal else f"Hi {name}!"

@app.mcp.prompt()
def code_review(code: str, language: str = "python") -> str:
    return f"Review this {language}: {code}"
```

MCP endpoint: `/mcp` (JSON-RPC 2.0). Connect Claude Desktop to `http://localhost:8080/mcp`.

---

## AI Agent & Memory

```python
from robyn.ai import agent, memory

mem = memory(provider="inmemory", user_id="user123")
chat_agent = agent(runner="simple", memory=mem)

@app.post("/chat")
async def chat(request):
    data = request.json()
    result = await chat_agent.run(data["query"], history=True)
    return result
```

Memory API: `add(message, metadata)`, `get(query)`, `clear()`
Custom providers: subclass `MemoryProvider`. Custom runners: subclass `AgentRunner`.

---

## Scaling & Deployment

### Multi-Process Architecture

- Shared-nothing model: each process has own Python interpreter + GIL
- Workers within each process provide I/O concurrency
- `--fast` flag: auto-optimized for your hardware

```bash
# CPU-heavy: processes = cores, workers = 1
python app.py --processes 8 --workers 1

# I/O-heavy: processes = cores/2, workers = 2-4
python app.py --processes 4 --workers 3

# Balanced
python app.py --processes 4 --workers 2
```

### Multiprocess Shared State

```python
from multiprocessing import Value
count = Value("i", 0)  # Thread-safe shared counter

@app.get("/count")
def get_count(request):
    return str(count.value)
```

### Docker

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8080
HEALTHCHECK --interval=30s CMD curl -f http://localhost:8080/health || exit 1
CMD ["python", "app.py", "--fast", "--host", "0.0.0.0", "--port", "8080"]
```

### Hosting

- **Railway**: Use `main.py` as entry point, `host="0.0.0.0"`, port from `$PORT` env var
- **Docker/K8s**: Use `--fast` or explicit `--processes`/`--workers`
- **Nginx**: Reverse proxy to multiple Robyn instances with `upstream` block

---

## Using Rust Directly

```bash
python -m robyn --create-rust-file hello_world
```

```rust
// hello_world.rs
// rustimport:pyo3
use pyo3::prelude::*;

#[pyfunction]
fn square(n: i32) -> i32 {
    n * n
}
```

```python
from hello_world import square
print(square(5))  # 25
```

Run with: `python -m robyn --compile-rust-path "." --dev`

---

## GraphQL (Strawberry)

```python
pip install robyn strawberry-graphql

import strawberry
from robyn import Robyn, jsonify

@strawberry.type
class Query:
    @strawberry.field
    def hello(self) -> str:
        return "world"

schema = strawberry.Schema(Query)

@app.get("/", const=True)
async def graphiql():
    return strawberry.utils.graphiql.get_graphiql_html()

@app.post("/")
async def graphql(request):
    body = request.json()
    data = await schema.execute(body["query"], body.get("variables"))
    return jsonify({"data": data.data})
```

---

## Key Architecture Notes

1. **Python layer**: Developer API, business logic, decorators
2. **Rust layer**: HTTP parsing, routing (matchit crate), WebSocket connections, static file serving, response serialization
3. **PyO3 bridge**: Rust calls Python handlers via PyO3; route registration passes `FunctionInfo` to Rust
4. **Request flow**: HTTP arrives -> Rust parses -> Rust routes -> Python handler (via PyO3) -> Rust serializes response
5. **Sync handlers**: Run in thread pool (won't block async runtime)
6. **Async handlers**: Run directly in async runtime
7. **Const routes**: Response cached in Rust memory, Python never called after first execution
8. **Zero-copy**: Request bodies parsed once, strings referenced not copied, response buffers reused
