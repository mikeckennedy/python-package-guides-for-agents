# Granian Reference

> **Version:** 2.7.2 | **Python:** 3.10+ (including PyPy, free-threaded 3.13+) | **License:** BSD

Granian is a Rust HTTP server for Python applications, built on [Hyper](https://github.com/hyperium/hyper) and [Tokio](https://github.com/tokio-rs/tokio). It supports ASGI/3, RSGI, and WSGI interfaces with HTTP/1.1, HTTP/2, WebSockets, HTTPS/mTLS, and direct static file serving.

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Python API](#python-api)
  - [Granian (Server)](#granian-server)
  - [Constructor Parameters](#constructor-parameters)
  - [Server Methods](#server-methods)
  - [Embedded Server](#embedded-server)
- [Enumerations](#enumerations)
- [Configuration Data Classes](#configuration-data-classes)
- [ASGI Interface](#asgi-interface)
- [RSGI Interface](#rsgi-interface)
  - [RSGI Scope](#rsgi-scope)
  - [RSGI HTTP Protocol](#rsgi-http-protocol)
  - [RSGI HTTP Stream Transport](#rsgi-http-stream-transport)
  - [RSGI WebSocket Protocol](#rsgi-websocket-protocol)
  - [RSGI WebSocket Transport](#rsgi-websocket-transport)
  - [RSGI Application Lifecycle](#rsgi-application-lifecycle)
- [WSGI Interface](#wsgi-interface)
- [CLI Reference](#cli-reference)
- [Logging](#logging)
- [Access Log Format](#access-log-format)
- [Proxy Headers](#proxy-headers)
- [Static Files](#static-files)
- [Metrics (Prometheus)](#metrics-prometheus)
- [Event Loop Customization](#event-loop-customization)
- [Lifecycle Hooks](#lifecycle-hooks)
- [Workers and Threading Model](#workers-and-threading-model)
- [Backpressure](#backpressure)
- [Free-Threaded Python](#free-threaded-python)
- [Errors](#errors)
- [Environment Variables](#environment-variables)

---

## Installation

```bash
pip install granian
```

### Optional extras

```bash
pip install granian[dotenv]    # .env file loading
pip install granian[pname]     # Custom process names
pip install granian[reload]    # Auto-reload on code changes
pip install granian[rloop]     # Rust-based event loop
pip install granian[uvloop]    # uvloop event loop
pip install granian[winloop]   # Windows event loop
```

Combine as needed: `pip install granian[pname,uvloop,reload]`

---

## Quick Start

### ASGI

```python
# main.py
async def app(scope, receive, send):
    assert scope['type'] == 'http'
    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [[b'content-type', b'text/plain']],
    })
    await send({
        'type': 'http.response.body',
        'body': b'Hello, world!',
    })
```

```bash
granian --interface asgi main:app
```

### RSGI

```python
# main.py
async def app(scope, proto):
    assert scope.proto == 'http'
    proto.response_str(
        status=200,
        headers=[('content-type', 'text/plain')],
        body="Hello, world!"
    )
```

```bash
granian --interface rsgi main:app
```

### WSGI

```python
# main.py
def app(environ, start_response):
    start_response('200 OK', [('content-type', 'text/plain')])
    return [b"Hello, world!"]
```

```bash
granian --interface wsgi main:app
```

### Programmatic Usage

```python
from granian import Granian

server = Granian(
    target="main:app",
    address="0.0.0.0",
    port=8000,
    interface="asgi",
    workers=4,
)
server.serve()
```

---

## Python API

### Granian (Server)

```python
from granian import Granian
```

`Granian` is an alias for the `Server` class. Under the hood, the actual class used depends on the Python build:

- **GIL build (standard):** `MPServer` — multi-process, each worker is a separate process
- **Free-threaded build:** `MTServer` — multi-thread, each worker is a thread in a single process

Both inherit from `AbstractServer` and expose the same API.

### Constructor Parameters

```python
Granian(
    target: str,                                        # Required. Import path "module:attribute"
    address: str = "127.0.0.1",                         # Host address to bind
    port: int = 8000,                                   # Port to bind
    uds: Path | None = None,                            # Unix domain socket path
    uds_permissions: int | None = None,                 # UDS file permissions (octal)
    interface: Interfaces = Interfaces.RSGI,             # "asgi", "asginl", "rsgi", or "wsgi"
    workers: int = 1,                                   # Number of worker processes/threads
    blocking_threads: int | None = None,                # Blocking threads per worker
    blocking_threads_idle_timeout: int = 30,            # Idle timeout in seconds (5-600)
    runtime_threads: int = 1,                           # Rust runtime threads per worker
    runtime_blocking_threads: int | None = None,        # Rust I/O blocking threads (default: 512)
    runtime_mode: RuntimeModes = RuntimeModes.auto,     # "auto", "mt", or "st"
    loop: Loops = Loops.auto,                           # Event loop: "auto","asyncio","rloop","uvloop","winloop"
    task_impl: TaskImpl = TaskImpl.asyncio,              # Async tasks: "asyncio" or "rust" (< 3.12 only)
    http: HTTPModes = HTTPModes.auto,                   # HTTP version: "auto", "1", or "2"
    websockets: bool = True,                            # Enable WebSocket support
    backlog: int = 1024,                                # Socket backlog (min 128)
    backpressure: int | None = None,                    # Max concurrent requests per worker
    http1_settings: HTTP1Settings | None = None,        # HTTP/1 tuning
    http2_settings: HTTP2Settings | None = None,        # HTTP/2 tuning
    log_enabled: bool = True,                           # Enable logging
    log_level: LogLevels = LogLevels.info,              # Log level
    log_dictconfig: dict[str, Any] | None = None,       # Python logging dictConfig
    log_access: bool = False,                           # Enable access logs
    log_access_format: str | None = None,               # Access log format string
    ssl_cert: Path | None = None,                       # SSL certificate file
    ssl_key: Path | None = None,                        # SSL key file (PKCS#8 format)
    ssl_key_password: str | None = None,                # SSL key password
    ssl_protocol_min: SSLProtocols = SSLProtocols.tls13,# Minimum TLS version
    ssl_ca: Path | None = None,                         # CA cert for client verification
    ssl_crl: list[Path] | None = None,                  # Certificate revocation lists
    ssl_client_verify: bool = False,                    # Verify client certificates (mTLS)
    url_path_prefix: str | None = None,                 # URL path prefix the app is mounted on
    respawn_failed_workers: bool = False,               # Auto-respawn crashed workers
    respawn_interval: float = 3.5,                      # Seconds between respawn attempts
    rss_sample_interval: int = 30,                      # Memory sampling interval (seconds, 1-300)
    rss_samples: int = 1,                               # Consecutive samples over limit before respawn
    workers_lifetime: int | None = None,                # Max worker lifetime (seconds, min 60)
    workers_max_rss: int | None = None,                 # Max RSS per worker (MiB)
    workers_kill_timeout: int | None = None,            # Graceful shutdown timeout (seconds, 1-1800)
    factory: bool = False,                              # Treat target as factory function
    working_dir: Path | None = None,                    # Working directory
    env_files: Sequence[Path] | None = None,            # .env files to load (requires dotenv extra)
    static_path_route: Sequence[str] | None = None,     # Routes for static file serving
    static_path_mount: Sequence[Path] | None = None,    # Filesystem paths for static files
    static_path_dir_to_file: str | None = None,         # Index file for directory listings
    static_path_expires: int = 86400,                   # Cache expiration in seconds (0 to disable)
    metrics_enabled: bool = False,                      # Enable Prometheus metrics
    metrics_scrape_interval: int = 15,                  # Metrics collection interval (1-60)
    metrics_address: str = "127.0.0.1",                 # Metrics endpoint host
    metrics_port: int = 9090,                           # Metrics endpoint port
    reload: bool = False,                               # Auto-reload on changes (requires reload extra)
    reload_paths: Sequence[Path] | None = None,         # Paths to watch (default: cwd)
    reload_ignore_dirs: Sequence[str] | None = None,    # Directory names to ignore
    reload_ignore_patterns: Sequence[str] | None = None,# Regex patterns to ignore
    reload_ignore_paths: Sequence[Path] | None = None,  # Absolute paths to ignore
    reload_filter: type | None = None,                  # Custom watchfiles.BaseFilter subclass
    reload_tick: int = 50,                              # Watch frequency in ms (50-5000)
    reload_ignore_worker_failure: bool = False,         # Ignore worker failures during reload
    process_name: str | None = None,                    # Custom process name (requires pname extra)
    pid_file: Path | None = None,                       # PID file path
)
```

### Server Methods

```python
server = Granian(...)

# Start serving (blocking)
server.serve() -> None

# Register a startup hook (called before workers spawn)
@server.on_startup
def my_startup():
    print("Starting up!")

# Register a reload hook (called on HUP signal or file changes)
@server.on_reload
def my_reload():
    print("Reloading!")

# Register a shutdown hook (called after workers stop)
@server.on_shutdown
def my_shutdown():
    print("Shutting down!")
```

Hook methods can also be called directly (not as decorators):

```python
server.on_startup(my_startup_fn)
server.on_reload(my_reload_fn)
server.on_shutdown(my_shutdown_fn)
```

### Embedded Server

For advanced lifecycle management or custom process management. Runs as AsyncIO tasks instead of separate processes/threads.

> **Limitations:** Async protocols only (no WSGI), single worker, experimental.

```python
from granian.server.embed import Server

# Note: accepts the application object directly, not an import string
server = Server(my_app, interface="asgi")

async def main():
    # Start serving (async, non-blocking when used as a task)
    server_task = asyncio.create_task(server.serve())

    # ... your logic ...

    # Trigger a reload
    server.reload()

    # Stop the server
    server.stop()
    await server_task
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `serve` | `async serve(spawn_target=None)` | Start serving asynchronously |
| `stop` | `stop() -> None` | Signal the server to stop |
| `reload` | `reload() -> None` | Trigger a worker reload |

---

## Enumerations

All enums are importable from `granian.constants`.

### `Interfaces`

| Value | String | Description |
|-------|--------|-------------|
| `ASGI` | `"asgi"` | ASGI/3 with lifespan protocol support |
| `ASGINL` | `"asginl"` | ASGI/3 without lifespan protocol |
| `RSGI` | `"rsgi"` | Granian's native RSGI protocol |
| `WSGI` | `"wsgi"` | WSGI 1.0 (synchronous) |

### `HTTPModes`

| Value | String | Description |
|-------|--------|-------------|
| `auto` | `"auto"` | Auto-negotiate (ALPN for HTTPS, HTTP/1 for plain) |
| `http1` | `"1"` | HTTP/1.1 only |
| `http2` | `"2"` | HTTP/2 only |

### `RuntimeModes`

| Value | String | Description |
|-------|--------|-------------|
| `auto` | `"auto"` | Auto-select based on configuration |
| `mt` | `"mt"` | Single multi-threaded Tokio runtime |
| `st` | `"st"` | N single-threaded Tokio runtimes |

Auto-selection logic: uses `st` for RSGI with 1 runtime thread and HTTP/1; otherwise uses `mt`.

### `Loops`

| Value | String | Extra Required | Description |
|-------|--------|----------------|-------------|
| `auto` | `"auto"` | — | Auto-select best available |
| `asyncio` | `"asyncio"` | — | stdlib asyncio |
| `rloop` | `"rloop"` | `granian[rloop]` | Rust-based event loop |
| `uvloop` | `"uvloop"` | `granian[uvloop]` | libuv-based event loop |
| `winloop` | `"winloop"` | `granian[winloop]` | Windows event loop |

### `TaskImpl`

| Value | String | Description |
|-------|--------|-------------|
| `asyncio` | `"asyncio"` | Standard Python asyncio tasks |
| `rust` | `"rust"` | Rust-based async tasks (Python < 3.12 only, experimental) |

### `SSLProtocols`

| Value | String |
|-------|--------|
| `tls12` | `"tls1.2"` |
| `tls13` | `"tls1.3"` |

### `LogLevels`

`critical`, `error`, `warning`, `warn`, `info`, `debug`, `notset`

Importable from `granian.log`.

---

## Configuration Data Classes

Import from `granian.http`.

### `HTTP1Settings`

```python
from granian.http import HTTP1Settings

@dataclass
class HTTP1Settings:
    header_read_timeout: int = 30_000       # Milliseconds to wait for headers (1-60000)
    keep_alive: bool = True                 # Enable HTTP/1.1 keep-alive
    max_buffer_size: int = 417_792          # Max buffer size in bytes (min 8192)
    pipeline_flush: bool = False            # Aggregate flushes for pipelined responses (experimental)
```

### `HTTP2Settings`

```python
from granian.http import HTTP2Settings

@dataclass
class HTTP2Settings:
    adaptive_window: bool = False                       # Adaptive flow control
    initial_connection_window_size: int = 1_048_576     # Connection flow control window (min 1024)
    initial_stream_window_size: int = 1_048_576         # Stream flow control window (min 1024)
    keep_alive_interval: int | None = None              # Ping interval in ms (1-60000, None=disabled)
    keep_alive_timeout: int = 20                        # Ping ack timeout in seconds (min 1)
    max_concurrent_streams: int = 200                   # SETTINGS_MAX_CONCURRENT_STREAMS (min 10)
    max_frame_size: int = 16_384                        # Max frame size in bytes (min 1024)
    max_headers_size: int = 16_777_216                  # Max header frame size in bytes (min 1)
    max_send_buffer_size: int = 409_600                 # Max send buffer per stream (min 1024)
```

---

## ASGI Interface

Granian implements the ASGI/3 specification. Use `interface="asgi"` for full lifespan support, or `interface="asginl"` to skip lifespan handling.

### Standard ASGI application

```python
async def app(scope, receive, send):
    if scope['type'] == 'http':
        body = b''
        while True:
            message = await receive()
            body += message.get('body', b'')
            if not message.get('more_body', False):
                break

        await send({
            'type': 'http.response.start',
            'status': 200,
            'headers': [[b'content-type', b'text/plain']],
        })
        await send({
            'type': 'http.response.body',
            'body': b'Hello, world!',
        })
```

### ASGI Lifespan

When using `interface="asgi"`, Granian sends lifespan events:

```python
async def app(scope, receive, send):
    if scope['type'] == 'lifespan':
        while True:
            message = await receive()
            if message['type'] == 'lifespan.startup':
                # ... perform startup ...
                await send({'type': 'lifespan.startup.complete'})
            elif message['type'] == 'lifespan.shutdown':
                # ... perform cleanup ...
                await send({'type': 'lifespan.shutdown.complete'})
                return
    elif scope['type'] == 'http':
        # ... handle request ...
```

### ASGI Extensions

Granian supports the ASGI [pathsend](https://asgi.readthedocs.io/en/latest/extensions.html#path-send) extension for efficient file responses.

### ASGI WebSocket

```python
async def app(scope, receive, send):
    if scope['type'] == 'websocket':
        # Accept the connection
        await send({'type': 'websocket.accept'})

        while True:
            message = await receive()
            if message['type'] == 'websocket.receive':
                data = message.get('text', message.get('bytes', ''))
                await send({'type': 'websocket.send', 'text': f'Echo: {data}'})
            elif message['type'] == 'websocket.disconnect':
                break
```

---

## RSGI Interface

RSGI (Rust Server Gateway Interface) is Granian's native protocol, designed to minimize Python-Rust boundary overhead. **RSGI Specification Version: 1.6**

### RSGI Scope

The scope object is passed as the first argument to your RSGI application. It is a Rust-backed object (not a dict) with attribute access.

**HTTP Scope attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `proto` | `str` | `"http"` for HTTP, `"ws"` for WebSocket |
| `rsgi_version` | `str` | RSGI specification version |
| `http_version` | `str` | `"1"`, `"1.1"`, or `"2"` |
| `server` | `str` | `"address:port"` of the listening server |
| `client` | `str` | `"address:port"` of the remote client |
| `scheme` | `str` | `"http"` or `"https"` |
| `method` | `str` | HTTP method, uppercased (e.g. `"GET"`) |
| `path` | `str` | Request path without query string |
| `query_string` | `str` | URL portion after `?` |
| `authority` | `str \| None` | HTTP/2 `:authority` pseudo-header (empty on HTTP/1) |
| `headers` | `RSGIHeaders` | Mapping-like object for request headers |

### RSGIHeaders

A dict-like object for accessing request headers. Header names are always lowercase.

```python
scope.headers['content-type']           # Get header value
'content-type' in scope.headers         # Check existence
scope.headers.get('x-custom', 'default')# Get with default
scope.headers.keys()                    # list[str]
scope.headers.values()                  # list[str]
scope.headers.items()                   # list[tuple[str, str]]
scope.headers.get_all('set-cookie')     # list[str] — all values for a key
len(scope.headers)                      # Number of headers
for key in scope.headers:               # Iterate over header names
    ...
```

### RSGI HTTP Protocol

The protocol object is the second argument to your RSGI application. It provides methods to read the request body and send responses.

#### Reading the request body

```python
# Read entire body at once
async def app(scope, proto):
    body: bytes = await proto()

# Read body in chunks (async iteration)
async def app(scope, proto):
    async for chunk in proto:
        process(chunk)  # chunk is bytes
```

#### Watching for client disconnection

```python
async def app(scope, proto):
    # Returns a future that resolves when client disconnects
    await proto.client_disconnect()
```

> **Note:** On keep-alive connections, `client_disconnect` may not resolve when a single request ends. Cancel the watcher after sending your response.

#### Sending responses

All response methods accept `status: int` and `headers: list[tuple[str, str]]`.

```python
# Empty response (no body)
proto.response_empty(status: int, headers: list[tuple[str, str]]) -> None

# String body
proto.response_str(status: int, headers: list[tuple[str, str]], body: str) -> None

# Bytes body
proto.response_bytes(status: int, headers: list[tuple[str, str]], body: bytes) -> None

# File response (entire file)
proto.response_file(status: int, headers: list[tuple[str, str]], file: str) -> None

# File range response (start inclusive, end exclusive)
proto.response_file_range(
    status: int, headers: list[tuple[str, str]],
    file: str, start: int, end: int
) -> None

# Streaming response — returns a transport object
transport = proto.response_stream(status: int, headers: list[tuple[str, str]]) -> RSGIHTTPStreamTransport
```

> **Note:** For `response_file` and `response_file_range`, the application is responsible for setting correct headers (Content-Type, Content-Length, etc.).

#### Complete RSGI HTTP example

```python
async def app(scope, proto):
    # Read body
    body = await proto()

    # Send response
    proto.response_str(
        status=200,
        headers=[
            ('content-type', 'application/json'),
            ('x-custom', 'value'),
        ],
        body='{"message": "Hello!"}'
    )
```

### RSGI HTTP Stream Transport

Returned by `proto.response_stream()`. Use for chunked/streaming responses.

```python
async def app(scope, proto):
    transport = proto.response_stream(
        200,
        [('content-type', 'text/event-stream')]
    )
    await transport.send_str("data: chunk 1\n\n")
    await transport.send_str("data: chunk 2\n\n")
    await transport.send_bytes(b"binary data")
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `send_bytes` | `async send_bytes(data: bytes) -> None` | Send binary data |
| `send_str` | `async send_str(data: str) -> None` | Send string data |

### RSGI WebSocket Protocol

When `scope.proto == "ws"`, the protocol object provides WebSocket methods.

```python
async def app(scope, proto):
    if scope.proto == 'ws':
        # Accept the connection — returns a transport
        transport = await proto.accept()

        while True:
            message = await transport.receive()
            if message.kind == 0:  # close
                break
            elif message.kind == 1:  # bytes
                await transport.send_bytes(message.data)
            elif message.kind == 2:  # string
                await transport.send_str(message.data)

    # OR reject the connection
    # proto.close(403)  # status is optional
```

**Protocol methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `accept` | `async accept() -> RSGIWebsocketTransport` | Accept and return transport |
| `close` | `close(status: int \| None) -> tuple[int, bool]` | Reject/close the connection |

### RSGI WebSocket Transport

Returned by `await proto.accept()`.

| Method | Signature | Description |
|--------|-----------|-------------|
| `receive` | `async receive() -> WebsocketMessage` | Receive one message |
| `send_bytes` | `async send_bytes(data: bytes) -> None` | Send binary message |
| `send_str` | `async send_str(data: str) -> None` | Send text message |

### WebsocketMessage

| Attribute | Type | Description |
|-----------|------|-------------|
| `kind` | `int` | `0` = close, `1` = bytes, `2` = string |
| `data` | `bytes \| str \| None` | Message payload (None for close) |

### RSGI Application Lifecycle

RSGI apps can implement lifecycle methods for init/cleanup:

```python
class App:
    def __rsgi_init__(self, loop):
        """Called during server startup. Event loop is NOT running yet."""
        # Synchronous init
        self.db = setup_db()
        # Async init via run_until_complete
        loop.run_until_complete(self.async_init())

    async def __rsgi__(self, scope, protocol):
        """Handle each request."""
        protocol.response_str(200, [], "OK")

    def __rsgi_del__(self, loop):
        """Called during server shutdown. Event loop is NOT running."""
        loop.run_until_complete(self.cleanup())
```

**Dual ASGI/RSGI support:**

```python
class App:
    async def __call__(self, scope, receive, send):
        """ASGI handler — used by other servers"""
        ...

    async def __rsgi__(self, scope, protocol):
        """RSGI handler — preferred by Granian"""
        ...
```

RSGI-compliant servers prefer `__rsgi__` over `__call__` when both are present.

---

## WSGI Interface

Standard WSGI 1.0 support. WebSockets are not available with WSGI.

```python
def app(environ, start_response):
    start_response('200 OK', [
        ('Content-Type', 'text/plain'),
    ])
    return [b'Hello, world!']
```

```bash
granian --interface wsgi main:app
```

### Factory pattern

```python
def create_app():
    """Factory function that returns the WSGI app."""
    app = MyFramework()
    return app
```

```bash
granian --interface wsgi --factory main:create_app
```

---

## CLI Reference

```
granian [OPTIONS] APP
```

`APP` is required: the application target in `module.path:attribute` format.

### Core Options

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--host TEXT` | `GRANIAN_HOST` | `127.0.0.1` | Host address |
| `--port INTEGER` | `GRANIAN_PORT` | `8000` | Port number |
| `--uds PATH` | `GRANIAN_UDS` | — | Unix domain socket path |
| `--uds-permissions OCTAL` | `GRANIAN_UDS_PERMISSIONS` | — | UDS file permissions |
| `--interface [asgi\|asginl\|rsgi\|wsgi]` | `GRANIAN_INTERFACE` | `rsgi` | Application interface |
| `--http [auto\|1\|2]` | `GRANIAN_HTTP` | `auto` | HTTP version |
| `--ws / --no-ws` | `GRANIAN_WEBSOCKETS` | enabled | WebSocket support |
| `--factory / --no-factory` | `GRANIAN_FACTORY` | disabled | Target is a factory function |
| `--working-dir DIRECTORY` | `GRANIAN_WORKING_DIR` | — | Working directory |
| `--url-path-prefix TEXT` | `GRANIAN_URL_PATH_PREFIX` | — | URL path prefix |

### Workers & Threading

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--workers INTEGER` | `GRANIAN_WORKERS` | `1` | Worker processes (min 1) |
| `--blocking-threads INTEGER` | `GRANIAN_BLOCKING_THREADS` | auto | Blocking threads per worker |
| `--blocking-threads-idle-timeout DURATION` | `GRANIAN_BLOCKING_THREADS_IDLE_TIMEOUT` | `30` | Idle thread timeout (5-600s) |
| `--runtime-threads INTEGER` | `GRANIAN_RUNTIME_THREADS` | `1` | Rust runtime threads per worker |
| `--runtime-blocking-threads INTEGER` | `GRANIAN_RUNTIME_BLOCKING_THREADS` | `512` | Rust I/O blocking threads |
| `--runtime-mode [auto\|mt\|st]` | `GRANIAN_RUNTIME_MODE` | `auto` | Runtime threading mode |
| `--loop [auto\|asyncio\|rloop\|uvloop\|winloop]` | `GRANIAN_LOOP` | `auto` | Event loop |
| `--task-impl [asyncio\|rust]` | `GRANIAN_TASK_IMPL` | `asyncio` | Async task implementation |

### Backlog & Backpressure

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--backlog INTEGER` | `GRANIAN_BACKLOG` | `1024` | Socket backlog (min 128) |
| `--backpressure INTEGER` | `GRANIAN_BACKPRESSURE` | `backlog/workers` | Max concurrent requests per worker |

### HTTP/1 Settings

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--http1-buffer-size INTEGER` | `GRANIAN_HTTP1_BUFFER_SIZE` | `417792` | Max buffer size (min 8192) |
| `--http1-header-read-timeout INTEGER` | `GRANIAN_HTTP1_HEADER_READ_TIMEOUT` | `30000` | Header read timeout in ms (1-60000) |
| `--http1-keep-alive / --no-http1-keep-alive` | `GRANIAN_HTTP1_KEEP_ALIVE` | enabled | Keep-alive |
| `--http1-pipeline-flush / --no-http1-pipeline-flush` | `GRANIAN_HTTP1_PIPELINE_FLUSH` | disabled | Pipelined flush (experimental) |

### HTTP/2 Settings

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--http2-adaptive-window / --no-...` | `GRANIAN_HTTP2_ADAPTIVE_WINDOW` | disabled | Adaptive flow control |
| `--http2-initial-connection-window-size INTEGER` | `GRANIAN_HTTP2_INITIAL_CONNECTION_WINDOW_SIZE` | `1048576` | Connection window (min 1024) |
| `--http2-initial-stream-window-size INTEGER` | `GRANIAN_HTTP2_INITIAL_STREAM_WINDOW_SIZE` | `1048576` | Stream window (min 1024) |
| `--http2-keep-alive-interval INTEGER` | `GRANIAN_HTTP2_KEEP_ALIVE_INTERVAL` | — | Ping interval in ms (1-60000) |
| `--http2-keep-alive-timeout DURATION` | `GRANIAN_HTTP2_KEEP_ALIVE_TIMEOUT` | `20` | Ping ack timeout (min 1s) |
| `--http2-max-concurrent-streams INTEGER` | `GRANIAN_HTTP2_MAX_CONCURRENT_STREAMS` | `200` | Max streams (min 10) |
| `--http2-max-frame-size INTEGER` | `GRANIAN_HTTP2_MAX_FRAME_SIZE` | `16384` | Max frame size (min 1024) |
| `--http2-max-headers-size INTEGER` | `GRANIAN_HTTP2_MAX_HEADERS_SIZE` | `16777216` | Max header size (min 1) |
| `--http2-max-send-buffer-size INTEGER` | `GRANIAN_HTTP2_MAX_SEND_BUFFER_SIZE` | `409600` | Send buffer per stream (min 1024) |

### SSL/TLS

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--ssl-certificate FILE` | `GRANIAN_SSL_CERTIFICATE` | — | Certificate file |
| `--ssl-keyfile FILE` | `GRANIAN_SSL_KEYFILE` | — | Key file (PKCS#8 only) |
| `--ssl-keyfile-password TEXT` | `GRANIAN_SSL_KEYFILE_PASSWORD` | — | Key password |
| `--ssl-protocol-min [tls1.2\|tls1.3]` | `GRANIAN_SSL_PROTOCOL_MIN` | `tls1.3` | Minimum TLS version |
| `--ssl-ca FILE` | `GRANIAN_SSL_CA` | — | CA cert for client verification |
| `--ssl-crl FILE` | `GRANIAN_SSL_CRL` | — | CRL file(s), repeatable |
| `--ssl-client-verify / --no-...` | `GRANIAN_SSL_CLIENT_VERIFY` | disabled | Client cert verification (mTLS) |

### Logging

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--log / --no-log` | `GRANIAN_LOG_ENABLED` | enabled | Enable logging |
| `--log-level [critical\|error\|warning\|warn\|info\|debug\|notset]` | `GRANIAN_LOG_LEVEL` | `info` | Log level |
| `--log-config FILE` | `GRANIAN_LOG_CONFIG` | — | Logging config file (JSON) |
| `--access-log / --no-access-log` | `GRANIAN_LOG_ACCESS_ENABLED` | disabled | Access log |
| `--access-log-fmt TEXT` | `GRANIAN_LOG_ACCESS_FMT` | see below | Access log format |

### Worker Management

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--respawn-failed-workers / --no-...` | `GRANIAN_RESPAWN_FAILED_WORKERS` | disabled | Auto-respawn crashed workers |
| `--respawn-interval FLOAT` | `GRANIAN_RESPAWN_INTERVAL` | `3.5` | Seconds between respawns |
| `--rss-sample-interval DURATION` | `GRANIAN_RSS_SAMPLE_INTERVAL` | `30` | Memory sampling interval (1-300s) |
| `--rss-samples INTEGER` | `GRANIAN_RSS_SAMPLES` | `1` | Samples before respawn (min 1) |
| `--workers-lifetime DURATION` | `GRANIAN_WORKERS_LIFETIME` | — | Max worker lifetime (min 60s) |
| `--workers-max-rss INTEGER` | `GRANIAN_WORKERS_MAX_RSS` | — | Max RSS per worker in MiB |
| `--workers-kill-timeout DURATION` | `GRANIAN_WORKERS_KILL_TIMEOUT` | disabled | Graceful shutdown timeout (1-1800s) |

### Static Files

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--static-path-route TEXT` | `GRANIAN_STATIC_PATH_ROUTE` | `/static` | URL route(s), repeatable |
| `--static-path-mount DIRECTORY` | `GRANIAN_STATIC_PATH_MOUNT` | — | Filesystem path(s), repeatable |
| `--static-path-dir-to-file TEXT` | `GRANIAN_STATIC_PATH_DIR_TO_FILE` | — | Index file for directories |
| `--static-path-expires DURATION` | `GRANIAN_STATIC_PATH_EXPIRES` | `86400` | Cache expiration (0=disabled) |

### Metrics

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--metrics / --no-metrics` | `GRANIAN_METRICS_ENABLED` | disabled | Prometheus metrics |
| `--metrics-scrape-interval DURATION` | `GRANIAN_METRICS_SCRAPE_INTERVAL` | `15` | Collection interval (1-60s) |
| `--metrics-address TEXT` | `GRANIAN_METRICS_ADDRESS` | `127.0.0.1` | Metrics endpoint host |
| `--metrics-port INTEGER` | `GRANIAN_METRICS_PORT` | `9090` | Metrics endpoint port |

### Reload

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--reload / --no-reload` | `GRANIAN_RELOAD` | disabled | Auto-reload (requires reload extra) |
| `--reload-paths PATH` | `GRANIAN_RELOAD_PATHS` | cwd | Paths to watch, repeatable |
| `--reload-ignore-dirs TEXT` | `GRANIAN_RELOAD_IGNORE_DIRS` | — | Directory names to ignore |
| `--reload-ignore-patterns TEXT` | `GRANIAN_RELOAD_IGNORE_PATTERNS` | — | Regex patterns to ignore |
| `--reload-ignore-paths PATH` | `GRANIAN_RELOAD_IGNORE_PATHS` | — | Absolute paths to ignore |
| `--reload-tick INTEGER` | `GRANIAN_RELOAD_TICK` | `50` | Watch frequency in ms (50-5000) |
| `--reload-ignore-worker-failure / --no-...` | `GRANIAN_RELOAD_IGNORE_WORKER_FAILURE` | disabled | Ignore worker failures |

### Other

| Option | Env Var | Default | Description |
|--------|---------|---------|-------------|
| `--env-files FILE` | `GRANIAN_ENV_FILES` | — | .env file(s) (requires dotenv extra) |
| `--process-name TEXT` | `GRANIAN_PROCESS_NAME` | — | Process name (requires pname extra) |
| `--pid-file FILE` | `GRANIAN_PID_FILE` | — | PID file path |
| `--version` | — | — | Show version |
| `--help` | — | — | Show help |

### Human-Readable Durations

Any option marked `DURATION` accepts seconds or suffixed values:

| Suffix | Meaning | Example |
|--------|---------|---------|
| (none) | seconds | `30` |
| `s` | seconds | `30s` |
| `m` | minutes | `5m` |
| `h` | hours | `2h` |
| `d` | days | `1d` |

Combinations work: `1h30m` = 5400 seconds.

---

## Logging

Granian uses Python's standard `logging` module with two loggers:

| Logger | Name | Purpose |
|--------|------|---------|
| Runtime | `_granian` | Server lifecycle messages |
| Access | `granian.access` | Per-request access logs |

### Programmatic logging configuration

```python
from granian.log import configure_logging, LogLevels

configure_logging(
    level=LogLevels.debug,
    config=None,        # Optional: Python logging dictConfig dict
    enabled=True,       # Set False to disable all logging
)
```

Or pass a JSON logging config file via CLI: `--log-config logging.json`

---

## Access Log Format

Default format:
```
[%(time)s] %(addr)s - "%(method)s %(path)s %(protocol)s" %(status)d %(dt_ms).3f
```

### Available atoms

| Atom | Description | Format |
|------|-------------|--------|
| `%(addr)s` | Client remote address | string |
| `%(time)s` | Request timestamp | string |
| `%(dt_ms).3f` | Request duration in milliseconds | float |
| `%(status)d` | HTTP response status code | integer |
| `%(path)s` | Request path (no query string) | string |
| `%(query_string)s` | Query string | string |
| `%(method)s` | HTTP method | string |
| `%(scheme)s` | Request scheme (http/https) | string |
| `%(protocol)s` | HTTP protocol version | string |

Example custom format:
```bash
granian --access-log --access-log-fmt '%(method)s %(path)s -> %(status)d in %(dt_ms).3fms' main:app
```

---

## Proxy Headers

For running behind a reverse proxy. Import from `granian.utils.proxies`.

### `wrap_asgi_with_proxy_headers`

```python
from granian.utils.proxies import wrap_asgi_with_proxy_headers

app = wrap_asgi_with_proxy_headers(
    app,                                    # Your ASGI application
    trusted_hosts: list[str] | str = "127.0.0.1"  # Trusted proxy hosts
)
```

### `wrap_wsgi_with_proxy_headers`

```python
from granian.utils.proxies import wrap_wsgi_with_proxy_headers

app = wrap_wsgi_with_proxy_headers(
    app,                                    # Your WSGI application
    trusted_hosts: list[str] | str = "127.0.0.1"  # Trusted proxy hosts
)
```

**Headers used:**
- `X-Forwarded-For` → replaces client IP
- `X-Forwarded-Proto` → replaces scheme

**`trusted_hosts` accepts:**
- IP addresses: `"192.0.2.1"`, `"fd12:3456:789a::1"`
- CIDR ranges: `"192.0.2.0/24"`, `"2001:db8:abcd::/48"`
- Wildcard: `"*"` (trust all — disables security check)
- A list: `["10.0.0.1", "10.0.0.0/24"]`

---

## Static Files

Offload static file serving to Rust, bypassing your Python application entirely.

### Basic setup

```bash
granian --static-path-mount ./assets/static main:app
# Serves files from ./assets/static at /static (default route)
```

### Multiple mounts

```bash
granian \
    --static-path-route /static --static-path-mount assets/static \
    --static-path-route /media --static-path-mount assets/media \
    main:app
```

The number of `--static-path-route` and `--static-path-mount` values must match.

### Directory index file

```bash
granian \
    --static-path-route /docs --static-path-mount generated/docs \
    --static-path-dir-to-file index.html \
    main:app
```

Requests to `/docs/somefolder` and `/docs/somefolder/index.html` both serve `index.html`.

### Cache control

```bash
# Set cache expiration to 1 hour
granian --static-path-mount ./static --static-path-expires 3600 main:app

# Disable caching
granian --static-path-mount ./static --static-path-expires 0 main:app
```

### Programmatic

```python
Granian(
    "main:app",
    static_path_route=["/static", "/media"],
    static_path_mount=[Path("assets/static"), Path("assets/media")],
    static_path_dir_to_file="index.html",
    static_path_expires=3600,
)
```

---

## Metrics (Prometheus)

Enable with `--metrics` or `metrics_enabled=True`. Exposed at `http://{metrics_address}:{metrics_port}/` in Prometheus text format. All metrics prefixed with `granian_`.

### Global metrics

| Metric | Type | Description |
|--------|------|-------------|
| `granian_workers_spawns` | counter | Total worker spawns |
| `granian_workers_respawns_for_err` | counter | Respawns due to errors |
| `granian_workers_respawns_for_lifetime` | counter | Respawns due to TTL |
| `granian_workers_respawns_for_rss` | counter | Respawns due to memory limit |

### Per-worker metrics (labeled `worker={id}`)

| Metric | Type | Unit | Description |
|--------|------|------|-------------|
| `granian_worker_lifetime` | counter | seconds | Current worker uptime |
| `granian_connections_active` | gauge | count | Active connections |
| `granian_connections_handled` | counter | count | Total accepted connections |
| `granian_connections_err` | gauge | count | Failed connections |
| `granian_requests_handled` | counter | count | Total processed requests |
| `granian_static_requests_handled` | counter | count | Static file requests |
| `granian_static_requests_err` | counter | count | Failed static requests |
| `granian_blocking_threads` | gauge | count | Active blocking threads (always 1 for async) |
| `granian_blocking_queue` | gauge | count | Pending blocking tasks |
| `granian_blocking_idle_cumulative` | counter | μs | Cumulative idle time |
| `granian_blocking_busy_cumulative` | counter | μs | Cumulative busy time |
| `granian_py_wait_cumulative` | counter | μs | Cumulative GIL wait time (0 on free-threaded) |

> **Note:** Metrics are not available when `--reload` is enabled.

---

## Event Loop Customization

Override the `auto` event loop policy when running Granian programmatically:

```python
import asyncio
from granian import Granian, loops

@loops.register('auto')
def build_loop():
    """Replace the default auto loop selection."""
    asyncio.set_event_loop_policy(asyncio.WindowsSelectorEventLoopPolicy())
    return asyncio.new_event_loop()

Granian("main:app").serve()
```

Available loop registry entries: `auto`, `asyncio`, `uvloop`, `rloop`, `winloop`.

---

## Lifecycle Hooks

Register callbacks for server lifecycle events. Hooks accept callables with no arguments.

```python
from granian import Granian

server = Granian("main:app")

# As decorators
@server.on_startup
def startup():
    print("Server starting, workers about to spawn")

@server.on_reload
def on_reload():
    print("Workers reloading (HUP signal or file change)")

@server.on_shutdown
def shutdown():
    print("Server shutting down, workers stopped")
```

**Execution order:**
1. `on_startup` hooks → workers spawn
2. (on reload signal) → `on_reload` hooks → workers respawn
3. (on shutdown) → workers stop → `on_shutdown` hooks

---

## Workers and Threading Model

### Architecture

```
Main Process
├── Worker 1 (process or thread)
│   ├── Tokio Runtime (N runtime threads)
│   ├── Blocking Thread Pool (M blocking threads)
│   └── AsyncIO Event Loop (1 per worker)
├── Worker 2
│   └── ...
└── Worker N
```

### Thread types

| Thread Type | Role | Tuning |
|-------------|------|--------|
| **Workers** | Each holds a Python interpreter and event loop | `--workers` |
| **Blocking threads** | Interact with Python/GIL for request processing | `--blocking-threads` |
| **Runtime threads** | Rust threads for network I/O (Tokio) | `--runtime-threads` |
| **Runtime blocking threads** | Rust threads for blocking I/O (filesystem) | `--runtime-blocking-threads` |

### Guidelines

- **Workers:** Start with CPU core count. In containers (Docker/K8s), use 1 worker per container and scale horizontally.
- **Blocking threads:** Fixed to 1 for async protocols (ASGI/RSGI). For WSGI, auto-set to `backpressure / 2`. Usually best left at default — tune backpressure instead.
- **Runtime threads:** Default of 1 is fine for most apps. Increase for heavy WebSocket or HTTP/2 concurrency.
- **Runtime blocking threads:** Default 512. Lower only if serving the same few files repeatedly.
- Do **not** use worker/thread numbers recommended for Gunicorn or Uvicorn — Granian's architecture is fundamentally different.

### Runtime mode

| Mode | Behavior | Best for |
|------|----------|----------|
| `st` | N single-threaded Tokio runtimes | Small process counts |
| `mt` | 1 multi-threaded Tokio runtime with N threads | Large CPU counts |
| `auto` | `st` for RSGI + 1 thread + HTTP/1; `mt` otherwise | Most cases |

---

## Backpressure

Backpressure limits how many concurrent connections a worker will accept before pausing the accept loop. Think of it as a secondary backlog managed by Granian.

**Default:** `backlog / workers`

### Guidelines by protocol

| Protocol | Recommendation |
|----------|---------------|
| **ASGI/RSGI** | Default is usually fine — AsyncIO handles suspension naturally |
| **WSGI (no I/O)** | Values above 2 won't help if your app never releases the GIL |
| **WSGI (external I/O)** | Higher values help — requests yield the GIL during network calls |
| **WSGI (database)** | Set to your connection pool size |

> **Warning:** Backpressure limits *connections*, not requests. Keep-alive connections handle multiple requests within one connection. If you have many long-lived keep-alive connections (e.g., behind a reverse proxy), set backpressure higher than the expected keep-alive count, or tune blocking threads instead.

---

## Free-Threaded Python

> **Experimental.** Not recommended for production.

Granian supports free-threaded Python (3.13t+). Key differences:

- Workers are **threads** (not processes) in a single interpreter
- Application is loaded **once** and shared between workers
- Each worker still runs its own AsyncIO event loop (in its thread)
- If the GIL gets re-enabled at runtime, Granian refuses to start

For WSGI on free-threaded Python, GIL limitations are theoretically absent, enabling true parallel execution of blocking Python code.

---

## Errors

Import from `granian.errors`.

```python
from granian.errors import FatalError, ConfigurationError, PidFileError
```

| Exception | Parent | Description |
|-----------|--------|-------------|
| `FatalError` | `Exception` | Base class for fatal errors |
| `ConfigurationError` | `FatalError` | Invalid configuration |
| `PidFileError` | `FatalError` | PID file conflict |

---

## Environment Variables

All CLI options can be set via environment variables with the `GRANIAN_` prefix. The CLI reads env vars automatically. See the [CLI Reference](#cli-reference) tables for the specific env var name for each option.

Example:
```bash
export GRANIAN_HOST=0.0.0.0
export GRANIAN_PORT=8080
export GRANIAN_WORKERS=4
export GRANIAN_INTERFACE=asgi
granian main:app
```
