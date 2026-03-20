# Quart Framework — Comprehensive Reference

Quart is a fast async Python web microframework. It is an asyncio reimplementation of Flask's API, supporting HTTP, WebSockets, HTTP/2 server push, and streaming responses. Quart runs on Python 3.9+ and is fully ASGI-compatible.

**Key dependencies:** aiofiles, blinker, click, hypercorn, jinja2, werkzeug, itsdangerous

---

## Table of Contents

- [Getting Started](#getting-started)
- [Application (`Quart`)](#application-quart)
- [Routing](#routing)
- [Request Object](#request-object)
- [Response Object](#response-object)
- [WebSockets](#websockets)
- [Templates](#templates)
- [Sessions](#sessions)
- [Blueprints](#blueprints)
- [Context & Globals](#context--globals)
- [Helper Functions](#helper-functions)
- [Configuration](#configuration)
- [Error Handling](#error-handling)
- [Lifecycle Hooks](#lifecycle-hooks)
- [Background Tasks](#background-tasks)
- [Streaming Responses](#streaming-responses)
- [Server-Sent Events (SSE)](#server-sent-events-sse)
- [Testing](#testing)
- [Class-Based Views](#class-based-views)
- [JSON Handling](#json-handling)
- [Signals](#signals)
- [Logging](#logging)
- [Middleware (ASGI)](#middleware-asgi)
- [Running Synchronous Code](#running-synchronous-code)
- [CLI Commands](#cli-commands)
- [HTTP/2 & Server Push](#http2--server-push)
- [Deployment](#deployment)
- [Flask Migration Guide](#flask-migration-guide)
- [DOS Mitigations](#dos-mitigations)
- [Extensions Ecosystem](#extensions-ecosystem)
- [Type Reference](#type-reference)
- [Public API Exports](#public-api-exports)

---

## Getting Started

### Installation

```bash
pip install quart
pip install quart[dotenv]   # optional dotenv support
```

### Minimal Application

```python
from quart import Quart

app = Quart(__name__)

@app.route("/")
async def hello():
    return "Hello, World!"

app.run()
```

### Running

```bash
# Development
python hello.py
# OR
QUART_APP=hello:app quart run

# Production (use an ASGI server)
hypercorn hello:app
```

---

## Application (`Quart`)

The central object. One instance per application. Created by passing the import name of the module.

### Constructor

```python
Quart(
    import_name: str,
    static_url_path: str | None = None,
    static_folder: str | None = "static",
    static_host: str | None = None,
    host_matching: bool = False,
    subdomain_matching: bool = False,
    template_folder: str | None = "templates",
    instance_path: str | None = None,
    instance_relative_config: bool = False,
    root_path: str | None = None,
)
```

### Running the App

```python
# Development server (not for production)
app.run(
    host: str | None = None,
    port: int | None = None,
    debug: bool | None = None,
    use_reloader: bool = True,
    loop: asyncio.AbstractEventLoop | None = None,
    ca_certs: str | None = None,
    certfile: str | None = None,
    keyfile: str | None = None,
    **kwargs,
) -> None

# Returns a coroutine for custom event loop control
app.run_task(
    host: str = "127.0.0.1",
    port: int = 5000,
    debug: bool | None = None,
    ca_certs: str | None = None,
    certfile: str | None = None,
    keyfile: str | None = None,
    shutdown_trigger: Callable[..., Awaitable[None]] | None = None,
) -> Coroutine[None, None, None]
```

### ASGI Interface

```python
# Quart is a standard ASGI app
await app(scope, receive, send)

# The actual ASGI handler (wrap this for middleware)
await app.asgi_app(scope, receive, send)
```

### Server Lifecycle

```python
await app.startup()   # Called when the server starts
await app.shutdown()  # Called when the server stops
```

### Context Factories

```python
app.app_context() -> AppContext
app.request_context(request: Request) -> RequestContext
app.websocket_context(websocket: Websocket) -> WebsocketContext
```

### Async/Sync Conversion

```python
# Ensure a function is async; wraps sync functions in an executor
app.ensure_async(func: Callable) -> Callable[..., Awaitable]

# Explicitly convert sync → async
app.sync_to_async(func: Callable[P, T]) -> Callable[P, Awaitable[T]]
```

### Resource Access

```python
await app.open_resource(path: FilePath, mode: str = "rb") -> AiofilesContextManager
await app.open_instance_resource(path: FilePath, mode: str = "rb") -> AiofilesContextManager
await app.send_static_file(filename: str) -> Response
app.get_send_file_max_age(filename: str | None) -> int | None
```

### Shell & Testing Utilities

```python
app.make_shell_context() -> dict
app.test_client(use_cookies: bool = True, **kwargs) -> TestClientProtocol
app.test_cli_runner(**kwargs) -> QuartCliRunner
app.test_app() -> TestAppProtocol

app.test_request_context(
    path: str,
    *,
    method: str = "GET",
    headers: dict | Headers | None = None,
    query_string: dict | None = None,
    scheme: str = "http",
    send_push_promise: Callable = no_op_push,
    data: AnyStr | None = None,
    form: dict | None = None,
    json: Any = sentinel,
    root_path: str = "",
    http_version: str = "1.1",
    scope_base: dict | None = None,
    auth: Authorization | tuple[str, str] | None = None,
    subdomain: str | None = None,
) -> RequestContext
```

### Request/Response Processing (Internal Pipeline)

```python
await app.full_dispatch_request(request_context=None) -> ResponseTypes
await app.preprocess_request(request_context=None) -> ResponseReturnValue | None
await app.dispatch_request(request_context=None) -> ResponseReturnValue
await app.finalize_request(result, request_context=None, from_error_handler=False) -> ResponseTypes
await app.process_response(response, request_context=None) -> ResponseTypes

# WebSocket equivalents
await app.full_dispatch_websocket(websocket_context=None) -> ResponseTypes | None
await app.preprocess_websocket(websocket_context=None) -> ResponseReturnValue | None
await app.dispatch_websocket(websocket_context=None) -> ResponseReturnValue | None
await app.finalize_websocket(result, websocket_context=None, from_error_handler=False) -> ResponseTypes | None
await app.postprocess_websocket(response, websocket_context=None) -> ResponseTypes

# End-to-end handlers
await app.handle_request(request: Request) -> ResponseTypes
await app.handle_websocket(websocket: Websocket) -> ResponseTypes | None
```

### Error Handling Methods

```python
await app.handle_http_exception(error: HTTPException) -> HTTPException | ResponseReturnValue
await app.handle_user_exception(error: Exception) -> HTTPException | ResponseReturnValue
await app.handle_exception(error: Exception) -> ResponseTypes
await app.handle_websocket_exception(error: Exception) -> ResponseTypes | None
await app.handle_background_exception(error: Exception) -> None
app.log_exception(exception_info) -> None
```

### Teardown

```python
await app.do_teardown_request(exc, request_context=None) -> None
await app.do_teardown_websocket(exc, websocket_context=None) -> None
await app.do_teardown_appcontext(exc) -> None
```

### URL & Template

```python
app.create_url_adapter(request: BaseRequestWebsocket | None) -> MapAdapter | None
app.url_for(endpoint, *, _anchor=None, _external=None, _method=None, _scheme=None, **values) -> str
app.create_jinja_environment() -> Environment
await app.update_template_context(context: dict) -> None
await app.make_default_options_response() -> Response
await app.make_response(result: ResponseReturnValue | HTTPException) -> ResponseTypes
```

---

## Routing

### Decorators

```python
# HTTP routes
@app.route("/path")
@app.route("/path", methods=["GET", "POST"])
@app.route("/path/<variable>")
@app.route("/path/<int:variable>")

# WebSocket routes
@app.websocket("/ws")
```

### URL Variable Converters

| Converter | Description | Example |
|-----------|-------------|---------|
| `string`  | (default) Any text without a slash | `<name>` |
| `int`     | Positive integers | `<int:page>` |
| `float`   | Positive floating point | `<float:value>` |
| `uuid`    | UUID strings | `<uuid:id>` |
| `path`    | Like string but includes slashes | `<path:subpath>` |

### Programmatic Route Registration

```python
app.add_url_rule(rule: str, endpoint=None, view_func=None, **options)
app.add_websocket(rule: str, endpoint=None, view_func=None, **options)
```

### Advanced Routing

```python
# Optional parameters with defaults
@app.route("/page/")
@app.route("/page/<int:page_no>", defaults={"page_no": 1})
async def page(page_no):
    ...

# Host matching (requires SERVER_NAME config)
@app.route("/", host="example.com")
async def index():
    ...

# Subdomain matching
@app.route("/", subdomain="api")
async def api_index():
    ...
```

### QuartRule & QuartMap

```python
# Custom rule class used internally
QuartRule(
    string: str,
    defaults: dict | None = None,
    subdomain: str | None = None,
    methods: Iterable[str] | None = None,
    endpoint: str | None = None,
    strict_slashes: bool | None = None,
    merge_slashes: bool | None = None,
    host: str | None = None,
    websocket: bool = False,
    provide_automatic_options: bool = False,
)

# QuartMap adds ASGI request binding
QuartMap.bind_to_request(request, subdomain, server_name) -> MapAdapter
```

---

## Request Object

Available as the `request` global within route handlers.

### Class: `Request`

```python
Request(
    method: str,
    scheme: str,
    path: str,
    query_string: bytes,
    headers: Headers,
    root_path: str,
    http_version: str,
    scope: HTTPScope,
    *,
    max_content_length: int | None = None,
    body_timeout: int | None = None,
    send_push_promise: Callable[[str, Headers], Awaitable[None]],
)
```

Inherits from `BaseRequestWebsocket` → Werkzeug's `SansIORequest`.

### Common Properties (sync — no await needed)

```python
request.method        # "GET", "POST", etc.
request.url           # Full URL string
request.path          # URL path
request.scheme        # "http" or "https"
request.host          # Host header value
request.headers       # Headers dict-like object
request.args          # Query string parameters (MultiDict)
request.cookies       # Cookie dict
request.content_type  # Content-Type header value
request.content_length # Content-Length as int
request.http_version  # "1.1", "2"
request.endpoint      # Matched endpoint name or None
request.blueprint     # Blueprint name or None
request.blueprints    # List of blueprint names (nested)
request.url_rule      # Matched URL rule
request.view_args     # Dict of URL variable values
request.script_root   # Script root path
request.url_root      # URL root
request.is_json       # True if content type is JSON
request.max_content_length  # Get/set max body size
request.max_form_memory_size  # Get/set max form field size
request.max_form_parts  # Get/set max multipart parts
```

### Async Properties (must await)

```python
data: bytes = await request.data                           # Raw body bytes
form: MultiDict = await request.form                       # Parsed form data
files: MultiDict[str, FileStorage] = await request.files   # Uploaded files
json_data: Any = await request.json                        # Parsed JSON body
values: CombinedMultiDict = await request.values           # Combined args + form
```

### Methods

```python
await request.get_data(
    cache: bool = True,
    as_text: bool = False,
    parse_form_data: bool = False,
) -> str | bytes

await request.get_json(
    force: bool = False,    # Ignore content type
    silent: bool = False,   # Return None on error
    cache: bool = True,
) -> Any

# HTTP/2 server push
await request.send_push_promise(path: str) -> None

await request.close() -> None
```

### Streaming the Request Body

```python
# Consume body incrementally (avoids loading all into memory)
async for chunk in request.body:
    process(chunk)
```

**Important:** Iterating `request.body` consumes it. If you need the data again, save it.

### Body Class

```python
Body(expected_content_length: int | None, max_content_length: int | None)
# Async iterable and awaitable
data = await body           # Get all data at once
async for chunk in body:    # Stream chunks
    ...
body.append(data: bytes)    # Add data (used by ASGI layer)
body.set_complete()         # Mark body as fully received
body.set_result(data: bytes)  # Set complete data (testing)
body.clear()                # Reset
```

### BaseRequestWebsocket (shared by Request and Websocket)

```python
# Properties available on both Request and Websocket
.endpoint -> str | None
.blueprint -> str | None
.blueprints -> list[str]
.script_root -> str
.url_root -> str
.routing_exception -> Exception | None
.url_rule -> QuartRule | None
.view_args -> dict[str, Any] | None
```

### FileStorage

```python
FileStorage(
    stream: IO[bytes] | None = None,
    filename: str | None = None,
    name: str | None = None,
    content_type: str | None = None,
    content_length: int | None = None,
    headers: Headers | None = None,
)

await file_storage.save(destination: PathLike, buffer_size: int = 16384) -> None
await file_storage.load(source: PathLike, buffer_size: int = 16384) -> None
```

---

## Response Object

### Class: `Response`

```python
Response(
    response: ResponseBody | str | bytes | Iterable | AsyncIterable | None = None,
    status: int | None = None,
    headers: dict | Headers | None = None,
    mimetype: str | None = None,
    content_type: str | None = None,
)
```

**Class attributes:**
- `automatically_set_content_length = True`
- `default_mimetype = "text/html"`
- `implicit_sequence_conversion = True`

### Properties

```python
response.status_code    # HTTP status code (int)
response.headers        # Response headers
response.content_type   # Content-Type header
response.max_cookie_size -> int

# Async
data: bytes = await response.data
json_data: Any = await response.json
```

### Methods

```python
response.set_data(data: str | bytes) -> None
await response.get_data(as_text: bool = False) -> str | bytes
await response.get_json(force: bool = False, silent: bool = False) -> Any

# Cookies
response.set_cookie(key, value="", max_age=None, expires=None, path="/",
                     domain=None, secure=False, httponly=False, samesite=None)
response.delete_cookie(key, path="/", domain=None)

# Conditional responses (ETag, Range, etc.)
await response.make_conditional(
    request: Request,
    accept_ranges: bool | str = False,
    complete_length: int | None = None,
) -> Response

await response.add_etag(overwrite: bool = False, weak: bool = False) -> None
await response.make_sequence() -> None
await response.freeze() -> None   # Freeze for pickling
await response.iter_encode() -> AsyncGenerator[bytes, None]
```

### Response Return Value Types

From a route handler, you can return:

| Return Type | Behavior |
|-------------|----------|
| `str` | Rendered as `text/html` |
| `dict` | Auto-serialized to JSON (`application/json`) |
| `list` | Auto-serialized to JSON (`application/json`) |
| `bytes` | Sent as-is |
| `Response` | Returned directly |
| `AsyncGenerator[bytes, None]` | Streamed response |
| `(value, status_code)` | Tuple with status |
| `(value, headers_dict)` | Tuple with headers |
| `(value, status_code, headers_dict)` | Tuple with both |

### Response Body Types (Internal)

```python
DataBody(data: bytes)         # For in-memory data
FileBody(file_path, *, buffer_size=None)  # For file serving
IOBody(io_stream: BytesIO, *, buffer_size=None)  # For BytesIO
IterableBody(iterable)        # For async/sync iterables
```

Each supports `async with` context manager and async iteration.

---

## WebSockets

### Declaring WebSocket Routes

```python
@app.websocket("/ws")
async def ws():
    while True:
        data = await websocket.receive()
        await websocket.send(f"Echo: {data}")
```

### Websocket Object

Available as the `websocket` global within websocket route handlers.

```python
Websocket(
    path: str,
    query_string: bytes,
    scheme: str,
    headers: Headers,
    root_path: str,
    http_version: str,
    subprotocols: list[str],
    receive: Callable,
    send: Callable,
    accept: Callable,
    close: Callable,
    scope: WebsocketScope,
)
```

### Properties

```python
websocket.requested_subprotocols -> list[str]
# Plus all BaseRequestWebsocket properties: .endpoint, .blueprint, .headers, etc.
```

### Methods

```python
data: AnyStr = await websocket.receive()        # Receive text or bytes
await websocket.send(data: AnyStr)              # Send text or bytes
json_data: Any = await websocket.receive_json()  # Receive and parse JSON
await websocket.send_json(*args, **kwargs)       # Serialize and send JSON

# Manual connection control
await websocket.accept(
    headers: dict | Headers | None = None,
    subprotocol: str | None = None,
) -> None

await websocket.close(code: int, reason: str = "") -> None
```

### Patterns

**Independent send/receive:**
```python
@app.websocket("/ws")
async def ws():
    producer = asyncio.create_task(send_updates())
    consumer = asyncio.create_task(receive_commands())
    await asyncio.gather(producer, consumer)
```

**Detecting disconnection:**
```python
@app.websocket("/ws")
async def ws():
    try:
        while True:
            data = await websocket.receive()
            await websocket.send(data)
    except asyncio.CancelledError:
        # Client disconnected — MUST re-raise for WebSockets
        raise
```

**Rejecting a connection** (return an HTTP response instead of accepting):
```python
@app.websocket("/ws")
async def ws():
    if not authorized():
        return "Forbidden", 403
    # Otherwise, proceed with WebSocket
    ...
```

---

## Templates

Uses Jinja2. Templates are loaded from the `templates/` folder by default.

### Rendering

```python
from quart import render_template, render_template_string, stream_template, stream_template_string

# Render to string
html: str = await render_template("index.html", name="World")
html: str = await render_template_string("<p>Hello {{ name }}</p>", name="World")

# Stream (returns async iterator for large templates)
async for chunk in await stream_template("large.html", items=items):
    ...
async for chunk in await stream_template_string(source, **context):
    ...
```

### Default Template Context

Available in every template automatically:
- `config` — The app config object
- `request` — The current request
- `session` — The session data
- `g` — The request-scoped namespace
- `url_for()` — URL builder function
- `get_flashed_messages()` — Flash message retrieval

### Custom Filters, Tests, and Globals

```python
@app.template_filter("reverse")
def reverse_filter(s):
    return s[::-1]

@app.template_test("even")
def is_even(n):
    return n % 2 == 0

@app.template_global()
def my_global():
    return "value"
```

### Context Processors

```python
@app.context_processor
async def inject_user():
    return dict(user=get_current_user())

# Blueprint-specific
@blueprint.context_processor
async def inject_blueprint_data():
    return dict(data="value")
```

---

## Sessions

Secure cookie-based sessions by default. Requires `app.secret_key` to be set.

### Usage

```python
from quart import session

@app.route("/login", methods=["POST"])
async def login():
    session["username"] = (await request.form)["username"]
    return redirect(url_for("index"))

@app.route("/")
async def index():
    username = session.get("username")
    return f"Hello {username}"
```

### Making Sessions Permanent

```python
session.permanent = True  # Must set BEFORE modifying session data
session["key"] = "value"
# Expiry controlled by PERMANENT_SESSION_LIFETIME config
```

### SessionInterface

Override for custom session storage (e.g., Redis, database):

```python
class SessionInterface:
    async open_session(self, app: Quart, request: BaseRequestWebsocket) -> SessionMixin | None
    async save_session(self, app: Quart, session: SessionMixin, response: Response | None) -> None
    def is_null_session(self, instance: object) -> bool
    def get_cookie_name(self, app: Quart) -> str
    def get_cookie_domain(self, app: Quart) -> str | None
    def get_cookie_path(self, app: Quart) -> str
    def get_cookie_httponly(self, app: Quart) -> bool
    def get_cookie_secure(self, app: Quart) -> bool
    def get_cookie_samesite(self, app: Quart) -> str
    def get_cookie_partitioned(self, app: Quart) -> bool
    def get_expiration_time(self, app: Quart, session: SessionMixin) -> datetime | None
    def should_set_cookie(self, app: Quart, session: SessionMixin) -> bool
```

### SecureCookieSessionInterface (Default)

```python
class SecureCookieSessionInterface(SessionInterface):
    salt = "cookie-session"
    digest_method = staticmethod(hashlib.sha1)
    key_derivation = "hmac"
    session_class = SecureCookieSession

    def get_signing_serializer(self, app: Quart) -> URLSafeTimedSerializer | None
    async open_session(self, app, request) -> SecureCookieSession | None
    async save_session(self, app, session, response) -> None
```

**Note:** WebSocket connections cannot set cookies after the connection has been accepted.

---

## Blueprints

Modular code organization for multi-file applications.

### Creating and Registering

```python
from quart import Blueprint

bp = Blueprint("auth", __name__, url_prefix="/auth")

@bp.route("/login")
async def login():
    ...

# Register with the app
app.register_blueprint(bp)
```

### Blueprint Constructor

Inherits from Flask's `SansIOBlueprint`. Key parameters:
- `name: str` — Unique identifier
- `import_name: str` — Module name (usually `__name__`)
- `url_prefix: str | None` — Prefix for all routes
- `template_folder: str | None` — Blueprint-specific templates
- `static_folder: str | None` — Blueprint-specific static files

### Blueprint-Specific Methods

```python
# WebSocket routes on blueprints
@bp.websocket("/ws")
async def ws():
    ...

bp.add_websocket(rule, endpoint=None, view_func=None, **options)

# Lifecycle hooks (blueprint-scoped)
@bp.before_websocket
@bp.after_websocket
@bp.teardown_websocket

# App-wide hooks registered from a blueprint
@bp.before_app_websocket
@bp.after_app_websocket
@bp.teardown_app_websocket
@bp.before_app_serving
@bp.after_app_serving
@bp.while_app_serving
```

### Resource Access

```python
await bp.open_resource(path: FilePath, mode: str = "rb") -> AiofilesContextManager
await bp.send_static_file(filename: str) -> Response
bp.get_send_file_max_age(filename: str | None) -> int | None
```

### Nested Blueprints

```python
parent = Blueprint("parent", __name__, url_prefix="/parent")
child = Blueprint("child", __name__, url_prefix="/child")
parent.register_blueprint(child)
app.register_blueprint(parent)
# child routes accessible at /parent/child/...
# Endpoint: "parent.child.view_name"
```

---

## Context & Globals

Quart uses task-local context variables (via `ContextVar`) so each concurrent request has its own isolated globals.

### Available Globals

```python
from quart import current_app, g, request, session, websocket

current_app  # The current Quart application instance
g            # Request-scoped namespace (resets per request)
request      # The current HTTP Request object
session      # Session data (dict-like)
websocket    # The current Websocket object (in websocket handlers)
```

### Additional Context Proxies

```python
from quart.ctx import (
    app_ctx,          # The AppContext object itself
    request_ctx,      # The RequestContext object itself
    websocket_ctx,    # The WebsocketContext object itself
)
```

### Context Types

**AppContext:**
```python
class AppContext:
    app: Quart
    url_adapter: MapAdapter | None
    g: AppCtxGlobals

    def copy(self) -> AppContext
    async push(self) -> None
    async pop(self, exc=None) -> None
```

**RequestContext:**
```python
class RequestContext:
    app: Quart
    request: Request
    session: SessionMixin | None
    flashes: None

    def copy(self) -> RequestContext
    def match_request(self) -> None
    async push(self) -> None
    async pop(self, exc=None) -> None
```

**WebsocketContext:**
```python
class WebsocketContext:
    app: Quart
    websocket: Websocket
    session: SessionMixin | None

    async push(self) -> None
    async pop(self, exc=None) -> None
```

### Context Checking

```python
from quart import has_app_context, has_request_context, has_websocket_context

if has_request_context():
    # Safe to use `request`
    ...
```

### Context Copying (for spawned tasks)

Spawned tasks do NOT inherit context. Use these decorators:

```python
from quart import copy_current_app_context, copy_current_request_context, copy_current_websocket_context

@app.route("/")
async def index():
    @copy_current_request_context
    async def background():
        # `request` is available here
        ...
    asyncio.create_task(background())
    return "OK"
```

### `after_this_request`

Register a function to run after the current request only:

```python
from quart.ctx import after_this_request

@app.route("/")
async def index():
    @after_this_request
    async def add_header(response):
        response.headers["X-Custom"] = "value"
        return response
    return "Hello"
```

---

## Helper Functions

Top-level convenience functions importable from `quart`:

### URL Building

```python
from quart import url_for

url_for(
    endpoint: str,
    *,
    _anchor: str | None = None,
    _external: bool | None = None,
    _method: str | None = None,
    _scheme: str | None = None,
    **values,
) -> str
```

### Response Creation

```python
from quart import make_response, redirect, abort, jsonify

# Wrap return values into a Response
response: Response = await make_response("body", 200, {"X-Header": "val"})

# Redirect
redirect(location: str, code: int = 302) -> Response

# Abort with HTTP error
abort(code: int | Response, *args, **kwargs) -> NoReturn

# Create JSON response
jsonify(*args, **kwargs) -> Response
```

### File Serving

```python
from quart import send_file, send_from_directory

await send_file(
    filename_or_io: FilePath | BytesIO,
    mimetype: str | None = None,
    as_attachment: bool = False,
    attachment_filename: str | None = None,
    add_etags: bool = True,
    cache_timeout: int | None = None,
    conditional: bool = False,
    last_modified: datetime | None = None,
) -> Response

await send_from_directory(
    directory: FilePath,
    file_name: str,
    *,
    mimetype: str | None = None,
    as_attachment: bool = False,
    attachment_filename: str | None = None,
    add_etags: bool = True,
    cache_timeout: int | None = None,
    conditional: bool = True,
    last_modified: datetime | None = None,
) -> Response
```

### Flash Messages

```python
from quart import flash, get_flashed_messages

await flash(message: str, category: str = "message") -> None

get_flashed_messages(
    with_categories: bool = False,
    category_filter: Iterable[str] = (),
) -> list[str] | list[tuple[str, str]]
```

### HTTP/2 Push Promise

```python
from quart import make_push_promise

await make_push_promise(path: str) -> None
```

### Stream with Context

```python
from quart import stream_with_context

@app.route("/stream")
async def stream():
    @stream_with_context
    async def generate():
        # `request` is available inside here
        yield b"data"
    return generate(), 200, {"Content-Type": "text/plain"}
```

### Template Utilities

```python
from quart import get_template_attribute

get_template_attribute(template_name: str, attribute: str) -> Any
```

---

## Configuration

### Setting Values

```python
app.config["SECRET_KEY"] = "your-secret"
app.config["DEBUG"] = True

# From environment variables (strips QUART_ prefix)
app.config.from_prefixed_env()  # QUART_SECRET_KEY → SECRET_KEY

# From file
app.config.from_file("config.toml", tomllib.load)

# From object
app.config.from_object("config.ProductionConfig")
```

### Class-Based Configuration

```python
class Config:
    SECRET_KEY = "default-secret"

class DevelopmentConfig(Config):
    DEBUG = True

class ProductionConfig(Config):
    DEBUG = False

app.config.from_object(DevelopmentConfig)
```

### Key Configuration Values

| Key | Default | Description |
|-----|---------|-------------|
| `SECRET_KEY` | `None` | Secret for signing sessions/cookies |
| `DEBUG` | `False` | Enable debug mode |
| `TESTING` | `False` | Enable testing mode |
| `MAX_CONTENT_LENGTH` | `16 * 1024 * 1024` | Max request body size (16 MB) |
| `MAX_FORM_MEMORY_SIZE` | `500000` | Max form field size in memory |
| `MAX_FORM_PARTS` | `1000` | Max multipart form parts |
| `BODY_TIMEOUT` | `60` | Seconds to wait for request body |
| `RESPONSE_TIMEOUT` | `60` | Seconds before closing slow response |
| `BACKGROUND_TASK_SHUTDOWN_TIMEOUT` | `None` | Seconds to wait for bg tasks on shutdown |
| `PERMANENT_SESSION_LIFETIME` | `timedelta(days=31)` | Session lifetime |
| `SESSION_COOKIE_NAME` | `"session"` | Session cookie name |
| `SESSION_COOKIE_SECURE` | `False` | HTTPS-only cookies |
| `SESSION_COOKIE_HTTPONLY` | `True` | No JS access to cookie |
| `SESSION_COOKIE_SAMESITE` | `"Lax"` | SameSite cookie policy |
| `SERVER_NAME` | `None` | Server name for URL building & host matching |
| `PREFERRED_URL_SCHEME` | `"http"` | URL scheme for external URLs |
| `SERVER_PUSH_HEADERS_TO_COPY` | `...` | Headers to copy for HTTP/2 push |

### Instance Folders

```python
app = Quart(__name__, instance_relative_config=True)
app.config.from_file("config.toml", tomllib.load)  # Loads from instance/
```

---

## Error Handling

### Using `abort()`

```python
from quart import abort

@app.route("/item/<int:id>")
async def get_item(id):
    item = await db.get(id)
    if item is None:
        abort(404)
    return item
```

### Custom Error Handlers

```python
@app.errorhandler(404)
async def not_found(error):
    return {"error": "Not found"}, 404

@app.errorhandler(500)
async def server_error(error):
    return {"error": "Internal error"}, 500

# Handle specific exception types
@app.errorhandler(ValueError)
async def handle_value_error(error):
    return {"error": str(error)}, 400
```

---

## Lifecycle Hooks

### Request Hooks

```python
@app.before_request
async def before():
    # Runs before every request
    # Return a value to short-circuit (skip the route handler)
    ...

@app.after_request
async def after(response: Response) -> Response:
    # Runs after every request; must return the response
    response.headers["X-Custom"] = "value"
    return response

@app.teardown_request
async def teardown(exc: BaseException | None) -> None:
    # Runs after response is sent; for cleanup
    ...
```

### WebSocket Hooks

```python
@app.before_websocket
async def before_ws():
    ...

@app.after_websocket
async def after_ws(response: Response | None) -> Response | None:
    ...

@app.teardown_websocket
async def teardown_ws(exc: BaseException | None) -> None:
    ...
```

### Serving Hooks (Startup/Shutdown)

```python
@app.before_serving
async def startup():
    # Run once when the server starts
    app.db_pool = await create_pool(...)

@app.after_serving
async def shutdown():
    # Run once when the server stops
    await app.db_pool.close()

# Combined startup + shutdown
@app.while_serving
async def lifespan():
    app.db_pool = await create_pool(...)
    yield  # App runs here
    await app.db_pool.close()
```

**Important:** `g` is reset after startup. Store long-lived resources on `app` directly (e.g., `app.db_pool`).

---

## Background Tasks

```python
@app.route("/submit")
async def submit():
    data = await request.get_json()
    app.add_background_task(process_data, data)
    return {"status": "accepted"}, 202

async def process_data(data):
    # Runs in the background; has access to app context
    # Must complete before BACKGROUND_TASK_SHUTDOWN_TIMEOUT on shutdown
    await heavy_processing(data)
```

- Supports both async and sync functions (sync runs in a thread)
- Background tasks have access to the application context
- Errors are logged but do not crash the app
- All tasks are awaited during shutdown

---

## Streaming Responses

### Returning a Stream

```python
@app.route("/stream")
async def stream():
    async def generate():
        for i in range(100):
            yield f"data: {i}\n".encode()
            await asyncio.sleep(0.1)
    return generate(), 200, {"Content-Type": "text/plain"}
```

### Preserving Request Context in Generators

```python
from quart import stream_with_context

@app.route("/stream")
async def stream():
    @stream_with_context
    async def generate():
        name = request.args.get("name")  # request is accessible
        yield f"Hello {name}".encode()
    return generate()
```

### Disabling Timeout for Long Streams

```python
@app.route("/stream")
async def stream():
    response = await make_response(generate())
    response.timeout = None  # Disable RESPONSE_TIMEOUT
    return response
```

### Testing Streams

```python
async with client.request("/stream") as connection:
    data = await connection.receive()  # Receive chunks incrementally
```

---

## Server-Sent Events (SSE)

```python
@app.route("/events")
async def events():
    if "text/event-stream" not in request.accept_mimetypes:
        abort(400)

    async def send_events():
        while True:
            data = await get_next_event()
            yield f"data: {data}\nevent: update\nid: {uuid4()}\n\n".encode()

    response = await make_response(
        send_events(),
        {"Content-Type": "text/event-stream", "Cache-Control": "no-cache"},
    )
    response.timeout = None  # Disable timeout for SSE
    return response
```

**SSE Event format:**
```
data: message content
event: event_type
id: unique_id
retry: 5000

```

---

## Testing

### Test Client

```python
import pytest
from quart import Quart

@pytest.fixture
def app():
    app = Quart(__name__)
    # ... configure app ...
    return app

@pytest.fixture
def client(app):
    return app.test_client()
```

### QuartClient Methods

```python
QuartClient(app: Quart, use_cookies: bool = True)

# HTTP methods (all async)
response = await client.get(path, **kwargs)
response = await client.post(path, **kwargs)
response = await client.put(path, **kwargs)
response = await client.patch(path, **kwargs)
response = await client.delete(path, **kwargs)
response = await client.head(path, **kwargs)
response = await client.options(path, **kwargs)
response = await client.trace(path, **kwargs)

# Generic request
response = await client.open(
    path: str,
    *,
    method: str = "GET",
    headers: dict | Headers | None = None,
    data: AnyStr | None = None,
    form: dict | None = None,
    files: dict[str, FileStorage] | None = None,
    query_string: dict | None = None,
    json: Any = sentinel,
    scheme: str = "http",
    follow_redirects: bool = False,
    root_path: str = "",
    http_version: str = "1.1",
    scope_base: dict | None = None,
    auth: Authorization | tuple[str, str] | None = None,
    subdomain: str | None = None,
) -> Response
```

### Streaming Test Client

```python
# For streaming request/response
async with client.request("/path", method="POST") as connection:
    await connection.send(b"chunk1")
    await connection.send(b"chunk2")
    await connection.send_complete()
    data = await connection.receive()
    response = await connection.as_response()
```

### WebSocket Testing

```python
async with client.websocket("/ws") as ws:
    await ws.send("hello")
    data = await ws.receive()
    json_data = await ws.receive_json()
    await ws.send_json({"key": "value"})
```

### Session Testing

```python
async with client.session_transaction("/", method="GET") as session:
    session["key"] = "value"
# Subsequent requests will see the session data
```

### Cookie Management

```python
client.set_cookie("localhost", "key", "value", max_age=3600)
client.delete_cookie("localhost", "key")
```

### Context Testing

```python
# App context
async with app.app_context():
    # current_app and g are available
    ...

# Request context (note: before_request hooks are NOT called)
async with app.test_request_context("/path", method="GET"):
    # request, session, g are available
    assert request.path == "/path"
    # To run before_request hooks:
    await app.preprocess_request()
```

### TestApp (for startup/shutdown hooks)

```python
from quart.testing import TestApp

async with app.test_app() as test_app:
    # before_serving hooks have run
    client = test_app.test_client()
    response = await client.get("/")
    # after_serving hooks will run on exit
```

```python
TestApp(
    app: Quart,
    startup_timeout: int = DEFAULT_TIMEOUT,
    shutdown_timeout: int = DEFAULT_TIMEOUT,
)
```

### Test HTTP Connection

```python
TestHTTPConnection(app: Quart, scope: HTTPScope)

async connection.send(data: bytes) -> None
async connection.send_complete() -> None
async connection.receive() -> bytes
async connection.disconnect() -> None
async connection.as_response() -> Response
```

### Test WebSocket Connection

```python
TestWebsocketConnection(app: Quart, scope: WebsocketScope)

async ws.receive() -> AnyStr
async ws.send(data: AnyStr) -> None
async ws.receive_json() -> Any
async ws.send_json(data: Any) -> None
async ws.close(code: int) -> None
async ws.disconnect() -> None
```

### CLI Testing

```python
runner = app.test_cli_runner()
result = runner.invoke(args=["command_name"])
```

---

## Class-Based Views

### View

```python
from quart.views import View

class MyView(View):
    methods = ["GET", "POST"]
    decorators = [login_required]
    init_every_request = True  # Default: new instance per request

    async def dispatch_request(self, **kwargs) -> ResponseReturnValue:
        if request.method == "POST":
            ...
        return await render_template("view.html")

app.add_url_rule("/path", view_func=MyView.as_view("my_view"))
```

### MethodView

```python
from quart.views import MethodView

class ItemAPI(MethodView):
    async def get(self, id):
        ...

    async def post(self):
        ...

    async def put(self, id):
        ...

    async def delete(self, id):
        ...

app.add_url_rule("/items/<int:id>", view_func=ItemAPI.as_view("item_api"))
```

`MethodView` automatically sets `methods` based on which HTTP method handlers are defined.

---

## JSON Handling

### Returning JSON

```python
from quart import jsonify

@app.route("/api/data")
async def data():
    return jsonify(name="World", count=42)
    # OR simply return a dict:
    return {"name": "World", "count": 42}
```

### Custom JSON Provider

```python
from flask.json.provider import DefaultJSONProvider

class CustomJSONProvider(DefaultJSONProvider):
    def default(self, obj):
        if isinstance(obj, Decimal):
            return str(obj)
        if isinstance(obj, MyCustomType):
            return obj.to_dict()
        return super().default(obj)

app.json_provider_class = CustomJSONProvider
app.json = CustomJSONProvider(app)
```

### Request/Response JSON

```python
# Request
data = await request.get_json(force=False, silent=False, cache=True)

# Response
data = await response.get_json(force=False, silent=False)
```

---

## Signals

Signals allow decoupled notifications using the `blinker` library.

```python
from quart import (
    request_started, request_finished, request_tearing_down,
    websocket_started, websocket_finished, websocket_tearing_down,
    before_render_template, template_rendered,
    got_request_exception, got_websocket_exception,
    appcontext_pushed, appcontext_popped, appcontext_tearing_down,
    message_flashed,
)
```

### All Available Signals

| Signal | Fired When |
|--------|------------|
| `request_started` | Before request processing begins |
| `request_finished` | After response is created |
| `request_tearing_down` | During request teardown |
| `websocket_started` | Before WebSocket processing |
| `websocket_received` | When a WebSocket message is received |
| `websocket_sent` | When a WebSocket message is sent |
| `websocket_finished` | After WebSocket handling completes |
| `websocket_tearing_down` | During WebSocket teardown |
| `got_request_exception` | When a request raises an exception |
| `got_websocket_exception` | When a WebSocket raises an exception |
| `got_background_exception` | When a background task raises an exception |
| `got_serving_exception` | When a serving error occurs |
| `before_render_template` | Before a template is rendered |
| `template_rendered` | After a template is rendered |
| `appcontext_pushed` | When an app context is pushed |
| `appcontext_popped` | When an app context is popped |
| `appcontext_tearing_down` | During app context teardown |
| `message_flashed` | When a message is flashed |

### Subscribing

```python
from quart import request_started

def log_request(sender, **extra):
    app.logger.info("Request started")

request_started.connect(log_request, app)
```

---

## Logging

```python
# Access the app logger
app.logger.info("Something happened")
app.logger.warning("Warning!")
app.logger.error("Error occurred")
```

The logger name matches `app.name`. Configure using Python's standard `logging` module:

```python
from logging.config import dictConfig

dictConfig({
    "version": 1,
    "handlers": {
        "file": {
            "class": "logging.FileHandler",
            "filename": "app.log",
        }
    },
    "root": {"level": "INFO", "handlers": ["file"]},
})
```

---

## Middleware (ASGI)

Quart is an ASGI app and can use any ASGI middleware:

```python
from some_middleware import SomeMiddleware

# Wrap asgi_app so middleware is also active during testing
app.asgi_app = SomeMiddleware(app.asgi_app)
```

### Example: Custom Header Validation Middleware

```python
class RequireHeaderMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            headers = dict(scope["headers"])
            if b"x-api-key" not in headers:
                await send({"type": "http.response.start", "status": 401, "headers": []})
                await send({"type": "http.response.body", "body": b"Unauthorized"})
                return
        await self.app(scope, receive, send)

app.asgi_app = RequireHeaderMiddleware(app.asgi_app)
```

---

## Running Synchronous Code

Quart can run sync code in a thread executor, preserving context:

### Sync Route Handlers

```python
@app.route("/sync")
def sync_route():
    # Automatically run in an executor
    # request, session, g are all accessible
    data = request.args.get("key")
    return f"Hello {data}"
```

### Sync Before/After Hooks

```python
@app.before_request
def sync_before():
    g.start_time = time.time()

@app.after_request
def sync_after(response):
    duration = time.time() - g.start_time
    response.headers["X-Duration"] = str(duration)
    return response
```

### Converting Between Sync and Async

```python
# Ensure a callable is async (Quart handles this internally)
async_func = app.ensure_async(maybe_sync_func)

# Explicitly wrap sync → async
async_func = app.sync_to_async(sync_func)
```

### Utility: `run_sync`

```python
from quart.utils import run_sync

# Low-level: wraps a sync function to run in executor
async_callable = run_sync(sync_func)
result = await async_callable(*args)
```

---

## CLI Commands

### Built-in Commands

```bash
quart run            # Run the development server
quart shell          # Open interactive Python shell with app context
quart routes         # Display registered routes
```

### Custom Commands

```python
@app.cli.command()
def seed_db():
    """Seed the database with initial data."""
    import asyncio
    asyncio.get_event_loop().run_until_complete(_seed())
    print("Database seeded!")

async def _seed():
    async with app.app_context():
        ...
```

### Command Groups

```python
@app.cli.group()
def users():
    """User management commands."""
    pass

@users.command()
@click.argument("name")
def create(name):
    """Create a new user."""
    ...
```

### `with_appcontext` Decorator

```python
from quart.cli import with_appcontext

@app.cli.command()
@with_appcontext
def my_command():
    # current_app is available
    ...
```

### ScriptInfo

```python
ScriptInfo(
    app_import_path: str | None = None,
    create_app: Callable[..., Quart] | None = None,
    set_debug_flag: bool = True,
)
# .load_app() -> Quart
```

### Environment Variables

| Variable | Description |
|----------|-------------|
| `QUART_APP` | Import path to the app (e.g., `myapp:app`) |
| `QUART_ENV` | Environment name |
| `QUART_DEBUG` | Enable debug mode (`1` / `true`) |
| `QUART_SKIP_DOTENV` | Skip loading `.env` files |

---

## HTTP/2 & Server Push

Requires an HTTP/2-capable ASGI server (e.g., Hypercorn with TLS).

### Push Promises

```python
from quart import make_push_promise, url_for

@app.route("/")
async def index():
    await make_push_promise(url_for("static", filename="css/style.css"))
    await make_push_promise(url_for("static", filename="js/app.js"))
    return await render_template("index.html")
```

### Running with TLS

```bash
hypercorn app:app --certfile cert.pem --keyfile key.pem
```

### Config for Push

```python
# Headers to copy from the original request to push promise requests
app.config["SERVER_PUSH_HEADERS_TO_COPY"] = [
    "Accept", "Accept-Language", "Cache-Control", "Cookie",
]
```

**Note:** Browser support for HTTP/2 server push is being deprecated.

---

## Deployment

### Production ASGI Servers

**Do NOT use `app.run()` in production.** Use a proper ASGI server:

```bash
# Hypercorn (ships with Quart)
hypercorn myapp:app
hypercorn myapp:app --bind 0.0.0.0:8000
hypercorn myapp:app --workers 4

# Uvicorn
uvicorn myapp:app --host 0.0.0.0 --port 8000

# Daphne
daphne myapp:app
```

### Serverless (AWS Lambda)

```python
from mangum import Mangum
from myapp import app

handler = Mangum(app)
```

---

## Flask Migration Guide

### Import Changes

```python
# Before (Flask)
from flask import Flask, request, render_template
# After (Quart)
from quart import Quart, request, render_template
```

### Add `async`/`await`

```python
# Before (Flask)
@app.route("/")
def index():
    return render_template("index.html")

# After (Quart)
@app.route("/")
async def index():
    return await render_template("index.html")
```

### Properties That Require `await` in Quart

| Flask (sync) | Quart (async) |
|-------------|---------------|
| `request.data` | `await request.data` |
| `request.get_data()` | `await request.get_data()` |
| `request.json` | `await request.json` |
| `request.get_json()` | `await request.get_json()` |
| `request.form` | `await request.form` |
| `request.files` | `await request.files` |
| `render_template(...)` | `await render_template(...)` |
| `render_template_string(...)` | `await render_template_string(...)` |

### Test Client Changes

```python
# Flask
response = client.get("/")
# Quart
response = await client.get("/")
```

### Common Gotcha with `await` and Attribute Access

```python
# WRONG — tries to access .get() on a coroutine
form_value = await request.form.get("key")

# CORRECT — await first, then access
form_value = (await request.form).get("key")
```

### Flask Extensions

Many Flask extensions work via `Quart-Flask-Patch`. Quart also has native alternatives:
- Quart-Auth, Quart-CORS, Quart-DB, Quart-Schema, Quart-Login, etc.

---

## DOS Mitigations

| Attack Vector | Mitigation | Config |
|--------------|------------|--------|
| Large request body | Size limit | `MAX_CONTENT_LENGTH` (default 16 MB) |
| Slow request body | Timeout | `BODY_TIMEOUT` (default 60s) |
| Slow response consumption | Timeout | `RESPONSE_TIMEOUT` (default 60s) |
| Inactive connection | Server timeout | Configured in ASGI server |
| Large WebSocket message | Size limit | Configured in ASGI server |

---

## Extensions Ecosystem

### Quart-Native Extensions

| Extension | Purpose |
|-----------|---------|
| Quart-Auth | Authentication & authorization |
| Quart-CORS | Cross-Origin Resource Sharing |
| Quart-DB | Database connection management |
| Quart-Schema | Request/response validation + OpenAPI docs |
| Quart-Login | User session management |
| Quart-WTF | Form handling with WTForms |
| Quart-SQLAlchemy | SQLAlchemy integration |

### Flask Extensions via Compatibility

Use `Quart-Flask-Patch` to enable Flask extensions that expect synchronous code:

```python
import quart_flask_patch  # Must be first import
from flask_sqlalchemy import SQLAlchemy
```

### Writing Extensions

```python
# Use ensure_async for sync function support
current_app.ensure_async(sync_function)
```

---

## Type Reference

### Response-Related Types

```python
FilePath = Union[bytes, str, os.PathLike]

ResponseValue = Union[
    Response, WerkzeugResponse, bytes, str,
    Mapping[str, Any], list[Any],
    Iterator[bytes], Iterator[str],
]

ResponseReturnValue = Union[
    ResponseValue,
    tuple[ResponseValue, HeadersValue],
    tuple[ResponseValue, StatusCode],
    tuple[ResponseValue, StatusCode, HeadersValue],
]

ResponseTypes = Union[Response, WerkzeugResponse]
StatusCode = int
```

### Headers Types

```python
HeaderName = str
HeaderValue = Union[str, list[str], tuple[str, ...]]
HeadersValue = Union[
    Headers,
    Mapping[HeaderName, HeaderValue],
    Sequence[tuple[HeaderName, HeaderValue]],
]
```

### Callable Types

```python
RouteCallable = Union[
    Callable[..., ResponseReturnValue],
    Callable[..., Awaitable[ResponseReturnValue]],
]

WebsocketCallable = Union[
    Callable[..., Optional[ResponseReturnValue]],
    Callable[..., Awaitable[Optional[ResponseReturnValue]]],
]

BeforeRequestCallable = Union[
    Callable[[], Optional[ResponseReturnValue]],
    Callable[[], Awaitable[Optional[ResponseReturnValue]]],
]

AfterRequestCallable = Union[
    Callable[[ResponseTypes], ResponseTypes],
    Callable[[ResponseTypes], Awaitable[ResponseTypes]],
]

TeardownCallable = Union[
    Callable[[Optional[BaseException]], None],
    Callable[[Optional[BaseException]], Awaitable[None]],
]

ErrorHandlerCallable = Union[
    Callable[[Any], ResponseReturnValue],
    Callable[[Any], Awaitable[ResponseReturnValue]],
]

TemplateContextProcessorCallable = Union[
    Callable[[], dict[str, Any]],
    Callable[[], Awaitable[dict[str, Any]]],
]

BeforeServingCallable = Union[Callable[[], None], Callable[[], Awaitable[None]]]
AfterServingCallable = Union[Callable[[], None], Callable[[], Awaitable[None]]]
WhileServingCallable = Callable[[], AsyncGenerator[None, None]]

BeforeWebsocketCallable = Union[
    Callable[[], Optional[ResponseReturnValue]],
    Callable[[], Awaitable[Optional[ResponseReturnValue]]],
]

AfterWebsocketCallable = Union[
    Callable[[Optional[ResponseTypes]], Optional[ResponseTypes]],
    Callable[[Optional[ResponseTypes]], Awaitable[Optional[ResponseTypes]]],
]
```

### ASGI Protocols

```python
class ASGIHTTPProtocol(Protocol):
    def __init__(self, app: Quart, scope: HTTPScope) -> None: ...
    async def __call__(self, receive: ASGIReceiveCallable, send: ASGISendCallable) -> None: ...

class ASGIWebsocketProtocol(Protocol):
    def __init__(self, app: Quart, scope: WebsocketScope) -> None: ...
    async def __call__(self, receive: ASGIReceiveCallable, send: ASGISendCallable) -> None: ...

class ASGILifespanProtocol(Protocol):
    def __init__(self, app: Quart, scope: LifespanScope) -> None: ...
    async def __call__(self, receive: ASGIReceiveCallable, send: ASGISendCallable) -> None: ...
```

---

## Public API Exports

Everything importable from `from quart import ...`:

### Classes
- `Quart` — Main application class
- `Blueprint` — Modular application component
- `Config` — Configuration management
- `Request` — HTTP request wrapper
- `Response` — HTTP response wrapper
- `Websocket` — WebSocket connection wrapper

### Context Globals
- `current_app` — Current application instance
- `g` — Request-scoped namespace
- `request` — Current HTTP request
- `session` — Session data
- `websocket` — Current WebSocket connection

### Functions
- `abort` — Raise HTTP exception
- `after_this_request` — Register post-request callback
- `copy_current_app_context` — Copy app context for spawned tasks
- `copy_current_request_context` — Copy request context for spawned tasks
- `copy_current_websocket_context` — Copy WebSocket context for spawned tasks
- `flash` — Flash a message
- `get_flashed_messages` — Retrieve flashed messages
- `get_template_attribute` — Get attribute from template
- `has_app_context` — Check for active app context
- `has_request_context` — Check for active request context
- `has_websocket_context` — Check for active WebSocket context
- `jsonify` — Create JSON response
- `make_push_promise` — HTTP/2 server push
- `make_response` — Create response from values
- `redirect` — Create redirect response
- `render_template` — Render Jinja2 template
- `render_template_string` — Render template from string
- `send_file` — Send file as response
- `send_from_directory` — Send file from directory
- `stream_template` — Stream template rendering
- `stream_template_string` — Stream template string rendering
- `stream_with_context` — Preserve context in generators
- `url_for` — Build URL for endpoint

### Signals
- `appcontext_popped`, `appcontext_pushed`, `appcontext_tearing_down`
- `before_render_template`, `template_rendered`
- `got_request_exception`, `got_websocket_exception`
- `got_background_exception`, `got_serving_exception`
- `message_flashed`
- `request_started`, `request_finished`, `request_tearing_down`
- `websocket_started`, `websocket_finished`, `websocket_tearing_down`
- `websocket_received`, `websocket_sent`
- `signals_available`

### Markup
- `escape` — HTML escape
- `Markup` — Safe HTML markup string

### Types
- `ResponseReturnValue` — Valid route return types
