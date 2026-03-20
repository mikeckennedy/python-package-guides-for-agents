# Flask Comprehensive Reference

> Generated from the Flask source code repository. This reference covers the full public
> API with function signatures, configuration options, patterns, and lifecycle details.

---

## Table of Contents

1. [Public API (Top-Level Imports)](#1-public-api-top-level-imports)
2. [Flask Application Class](#2-flask-application-class)
3. [Configuration](#3-configuration)
4. [Routing and URL Rules](#4-routing-and-url-rules)
5. [Request Object](#5-request-object)
6. [Response Object](#6-response-object)
7. [Context Locals and Proxies](#7-context-locals-and-proxies)
8. [Sessions](#8-sessions)
9. [Blueprints](#9-blueprints)
10. [Templates](#10-templates)
11. [JSON Handling](#11-json-handling)
12. [Error Handling](#12-error-handling)
13. [Signals](#13-signals)
14. [Class-Based Views](#14-class-based-views)
15. [Helper Functions](#15-helper-functions)
16. [Testing](#16-testing)
17. [CLI Integration](#17-cli-integration)
18. [Request/Response Lifecycle](#18-requestresponse-lifecycle)
19. [Common Patterns](#19-common-patterns)
20. [Security Considerations](#20-security-considerations)
21. [Deployment](#21-deployment)

---

## 1. Public API (Top-Level Imports)

Everything you need is importable directly from `flask`:

```python
from flask import (
    # Core classes
    Flask, Blueprint, Config, Request, Response,

    # Context proxies
    current_app, g, request, session,

    # Template rendering
    render_template, render_template_string,
    stream_template, stream_template_string,

    # Helpers
    url_for, redirect, abort, make_response,
    jsonify, flash, get_flashed_messages,
    send_file, send_from_directory,
    stream_with_context, get_template_attribute,

    # Context utilities
    after_this_request, copy_current_request_context,
    has_app_context, has_request_context,

    # Signals
    appcontext_pushed, appcontext_popped, appcontext_tearing_down,
    request_started, request_finished, request_tearing_down,
    got_request_exception, message_flashed,
    before_render_template, template_rendered,

    # JSON module
    json,
)
```

---

## 2. Flask Application Class

### Constructor

```python
Flask(
    import_name: str,
    static_url_path: str | None = None,
    static_folder: str | os.PathLike | None = "static",
    static_host: str | None = None,
    host_matching: bool = False,
    subdomain_matching: bool = False,
    template_folder: str | os.PathLike | None = "templates",
    instance_path: str | None = None,
    instance_relative_config: bool = False,
    root_path: str | None = None,
)
```

**Parameters:**
- `import_name` — Name of the application package. Use `__name__` for single modules.
- `static_url_path` — URL path for static files. Defaults to the `static_folder` name.
- `static_folder` — Directory for static files, relative to `root_path`. Default `"static"`.
- `static_host` — Host for static route when using `host_matching=True`.
- `host_matching` — Match routes by host instead of subdomain.
- `subdomain_matching` — Match routes by subdomain relative to `SERVER_NAME`.
- `template_folder` — Directory for templates. Default `"templates"`.
- `instance_path` — Path to the instance folder. Default: `instance/` next to the package.
- `instance_relative_config` — Load config files relative to instance path instead of root.
- `root_path` — Path to application files root. Auto-detected in most cases.

### Class Attributes

| Attribute | Default | Description |
|---|---|---|
| `request_class` | `flask.Request` | Class used for request objects |
| `response_class` | `flask.Response` | Class used for response objects |
| `session_interface` | `SecureCookieSessionInterface()` | Session backend |
| `json_provider_class` | `DefaultJSONProvider` | JSON serialization provider |
| `jinja_environment` | `Environment` | Jinja2 environment class |
| `jinja_options` | `{}` | Options passed to Jinja environment |
| `url_rule_class` | `werkzeug.routing.Rule` | Class for URL rules |
| `url_map_class` | `werkzeug.routing.Map` | Class for URL map |
| `test_client_class` | `None` (uses `FlaskClient`) | Custom test client class |
| `test_cli_runner_class` | `None` (uses `FlaskCliRunner`) | Custom CLI test runner class |
| `aborter_class` | `werkzeug.exceptions.Aborter` | Class for the `abort()` handler |
| `config_class` | `Config` | Class for the config dict |
| `app_ctx_globals_class` | `_AppCtxGlobals` | Class for `flask.g` |

### Instance Attributes

| Attribute | Type | Description |
|---|---|---|
| `config` | `Config` | Configuration dictionary |
| `url_map` | `Map` | Werkzeug URL routing map |
| `view_functions` | `dict[str, callable]` | Map of endpoint → view function |
| `error_handler_spec` | `dict` | Registered error handlers |
| `before_request_funcs` | `dict` | Before-request hook registry |
| `after_request_funcs` | `dict` | After-request hook registry |
| `teardown_request_funcs` | `dict` | Teardown hook registry |
| `teardown_appcontext_funcs` | `list` | App context teardown hooks |
| `url_value_preprocessors` | `dict` | URL value preprocessor registry |
| `url_default_functions` | `dict` | URL defaults registry |
| `template_context_processors` | `dict` | Template context processor registry |
| `shell_context_processors` | `list` | Shell context processor list |
| `blueprints` | `dict[str, Blueprint]` | Registered blueprints |
| `extensions` | `dict` | Namespace for extension data |
| `name` | `str` | Application name (from import_name) |
| `logger` | `logging.Logger` | Application logger |
| `jinja_env` | `Environment` | The Jinja2 environment (lazy-created) |
| `debug` | `bool` | Debug mode flag |
| `testing` | `bool` | Testing mode flag (ConfigAttribute) |
| `secret_key` | `str \| bytes \| None` | Secret key for signing (ConfigAttribute) |
| `permanent_session_lifetime` | `timedelta` | Permanent session lifetime (ConfigAttribute, default 31 days) |

### Routing Methods

#### `app.route(rule, **options)`
Decorator to register a URL rule and view function.

```python
@app.route("/users/<int:user_id>", methods=["GET", "POST"])
def user_detail(user_id):
    ...
```

**Parameters:**
- `rule` — URL rule string with optional converters (e.g., `<int:id>`, `<path:filename>`)
- `**options` — Passed to `add_url_rule()`: `methods`, `endpoint`, `defaults`, `subdomain`, `host`, `provide_automatic_options`, etc.

#### `app.add_url_rule(rule, endpoint=None, view_func=None, provide_automatic_options=None, **options)`
Register a URL rule without using a decorator.

```python
app.add_url_rule("/about", endpoint="about", view_func=about_page)
```

#### HTTP Method Shortcuts

```python
@app.get("/items")           # GET only
@app.post("/items")          # POST only
@app.put("/items/<int:id>")  # PUT only
@app.delete("/items/<int:id>")  # DELETE only
@app.patch("/items/<int:id>")   # PATCH only
```

### Request Hook Decorators

#### `@app.before_request`
Run before each request. If it returns a non-`None` value, that value is used as the response and the view function is skipped.

```python
@app.before_request
def load_user():
    g.user = get_user_from_session()
```

#### `@app.after_request`
Run after each request. Receives and must return the `Response` object. Not called if an unhandled exception occurred.

```python
@app.after_request
def add_header(response):
    response.headers["X-Custom"] = "value"
    return response
```

#### `@app.teardown_request`
Run at the end of every request, even if an exception occurred. Receives the exception (or `None`).

```python
@app.teardown_request
def close_db(exception):
    db = g.pop("db", None)
    if db is not None:
        db.close()
```

#### `@app.teardown_appcontext`
Run when the application context is popped. Similar to `teardown_request` but at the app context level.

```python
@app.teardown_appcontext
def close_connection(exception):
    db = g.pop("db", None)
    if db is not None:
        db.close()
```

#### `after_this_request(f)`
Register a function to run after the *current* request only. Must be called within a request context.

```python
from flask import after_this_request

@app.route("/set-cookie")
def set_cookie():
    @after_this_request
    def remember(response):
        response.set_cookie("seen", "1")
        return response
    return "Cookie set!"
```

### URL Value Preprocessors

#### `@app.url_value_preprocessor`
Modify URL values before they reach the view. Receives `endpoint` and `values` dict.

```python
@app.url_value_preprocessor
def pull_lang_code(endpoint, values):
    g.lang_code = values.pop("lang_code", None)
```

#### `@app.url_defaults`
Inject default URL values for `url_for()`. Receives `endpoint` and `values` dict.

```python
@app.url_defaults
def add_lang_code(endpoint, values):
    values.setdefault("lang_code", g.lang_code)
```

### Template Methods

#### `@app.context_processor`
Register a function that returns a dict of variables to inject into all templates.

```python
@app.context_processor
def inject_user():
    return dict(current_user=g.user)
```

#### `@app.template_filter(name=None)`
Register a custom Jinja2 filter.

```python
@app.template_filter("reverse")
def reverse_filter(s):
    return s[::-1]
```

Usage in template: `{{ "hello" | reverse }}`

#### `@app.template_global(name=None)`
Register a function available in all templates.

```python
@app.template_global()
def format_price(amount):
    return f"${amount:.2f}"
```

#### `@app.template_test(name=None)`
Register a custom Jinja2 test.

```python
@app.template_test("even")
def is_even(n):
    return n % 2 == 0
```

Usage in template: `{% if value is even %}`

### Error Handler Registration

#### `@app.errorhandler(code_or_exception)`
Register a handler for an HTTP status code or exception class.

```python
@app.errorhandler(404)
def not_found(error):
    return render_template("404.html"), 404

@app.errorhandler(ValueError)
def handle_value_error(error):
    return {"error": str(error)}, 400
```

#### `app.register_error_handler(code_or_exception, f)`
Non-decorator form of error handler registration.

### Context Methods

#### `app.app_context()`
Create an application context. Use as a context manager.

```python
with app.app_context():
    db.create_all()
    print(current_app.config["DEBUG"])
```

#### `app.test_request_context(*args, **kwargs)`
Create a request context for testing without running a request. Arguments are passed to `EnvironBuilder`.

```python
with app.test_request_context("/users?page=2", method="GET"):
    assert request.args["page"] == "2"
```

#### `app.test_client(use_cookies=True, **kwargs)`
Create a `FlaskClient` for testing.

```python
client = app.test_client()
response = client.get("/")
```

#### `app.test_cli_runner(**kwargs)`
Create a `FlaskCliRunner` for testing CLI commands.

### Server and WSGI

#### `app.run(host=None, port=None, debug=None, load_dotenv=True, **options)`
Run the development server. **Never use in production.**

```python
if __name__ == "__main__":
    app.run(debug=True, port=5000)
```

#### `app.wsgi_app(environ, start_response)`
The actual WSGI application. Useful for wrapping with middleware:

```python
from werkzeug.middleware.proxy_fix import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1)
```

### Response Conversion

#### `app.make_response(rv)`
Convert a view function return value to a `Response` object. Handles:
- `str` → `Response` with `text/html`
- `dict` or `list` → JSON `Response`
- `tuple` of `(body, status)`, `(body, headers)`, or `(body, status, headers)`
- `Response` → passed through
- WSGI callable → wrapped

### Shell Context

#### `@app.shell_context_processor`
Register a function that returns a dict to add to `flask shell` context.

```python
@app.shell_context_processor
def make_shell_context():
    return {"db": db, "User": User}
```

### Resource Methods

#### `app.open_resource(resource, mode="rb", encoding=None)`
Open a resource file relative to the application root.

#### `app.open_instance_resource(resource, mode="rb", encoding=None)`
Open a resource file relative to the instance folder.

#### `app.send_static_file(filename)`
Serve a static file from the static folder.

---

## 3. Configuration

### Config Class

`app.config` is a `Config` instance (subclass of `dict`) that stores all configuration.

```python
# Direct setting
app.config["SECRET_KEY"] = "dev-key"
app.config["DEBUG"] = True

# Batch update
app.config.update(
    TESTING=True,
    SECRET_KEY="test-key",
)
```

### Loading Configuration

#### `config.from_object(obj)`
Load config from a Python object (module or class). Only uppercase attributes are loaded.

```python
class ProductionConfig:
    SECRET_KEY = "prod-secret"
    SQLALCHEMY_DATABASE_URI = "postgresql://..."

app.config.from_object(ProductionConfig)
# Or from a module path:
app.config.from_object("myapp.config.ProductionConfig")
```

#### `config.from_pyfile(filename, silent=False)`
Load config from a Python file (executes it).

```python
app.config.from_pyfile("config.py")
# With instance_relative_config=True, loads from instance folder
```

#### `config.from_envvar(variable_name, silent=False)`
Load config from a file path stored in an environment variable.

```python
# Set: export APP_CONFIG=/path/to/config.py
app.config.from_envvar("APP_CONFIG")
```

#### `config.from_prefixed_env(prefix="FLASK", *, loads=json.loads)`
Load config from environment variables with a prefix. Double underscores create nested keys.

```python
# FLASK_SECRET_KEY=abc → config["SECRET_KEY"] = "abc"
# FLASK_SQLALCHEMY__DATABASE_URI=x → config["SQLALCHEMY"]["DATABASE_URI"] = "x"
app.config.from_prefixed_env()
```

#### `config.from_file(filename, load, silent=False, text=True)`
Load config from a file using a custom loader.

```python
import tomllib
app.config.from_file("config.toml", load=tomllib.load, text=False)

import json
app.config.from_file("config.json", load=json.load)
```

### Built-in Configuration Keys

| Key | Default | Description |
|---|---|---|
| `DEBUG` | `None` | Enable debug mode. Use `--debug` CLI flag. |
| `TESTING` | `False` | Enable testing mode. Disables error catching. |
| `PROPAGATE_EXCEPTIONS` | `None` | Re-raise exceptions instead of handling. `None` = `True` if `TESTING` or `DEBUG`. |
| `SECRET_KEY` | `None` | Key for signing sessions and security tokens. **Required for sessions.** |
| `SECRET_KEY_FALLBACKS` | `None` | List of old secret keys that can still unsign but won't be used to sign new data. |
| `PERMANENT_SESSION_LIFETIME` | `timedelta(days=31)` | Lifetime of permanent sessions. |
| `USE_X_SENDFILE` | `False` | Use server's `X-Sendfile` header for `send_file()`. |
| `TRUSTED_HOSTS` | `None` | List of trusted host/domain names for Host header validation. |
| `SERVER_NAME` | `None` | Server name for URL generation outside requests. |
| `APPLICATION_ROOT` | `"/"` | Path prefix for the application. |
| `PREFERRED_URL_SCHEME` | `"http"` | URL scheme for external URL generation. |
| `MAX_CONTENT_LENGTH` | `None` | Max request body size in bytes. `413` if exceeded. |
| `MAX_FORM_MEMORY_SIZE` | `500_000` | Max size of a single form field in bytes. |
| `MAX_FORM_PARTS` | `1_000` | Max number of multipart form fields. |
| `SEND_FILE_MAX_AGE_DEFAULT` | `None` | Default `Cache-Control` max-age for `send_file()`. |
| `TRAP_BAD_REQUEST_ERRORS` | `None` | Don't catch `BadRequest` (makes `KeyError` on `request.form` easier to debug). |
| `TRAP_HTTP_EXCEPTIONS` | `False` | Don't catch any `HTTPException`. |
| `EXPLAIN_TEMPLATE_LOADING` | `False` | Log details of template loading for debugging. |
| `TEMPLATES_AUTO_RELOAD` | `None` | Auto-reload templates on change. `None` = `True` in debug. |
| `MAX_COOKIE_SIZE` | `4093` | Warn if cookies exceed this size. |
| `PROVIDE_AUTOMATIC_OPTIONS` | `True` | Auto-generate `OPTIONS` responses. |

### Session Cookie Configuration

| Key | Default | Description |
|---|---|---|
| `SESSION_COOKIE_NAME` | `"session"` | Cookie name for the session. |
| `SESSION_COOKIE_DOMAIN` | `None` | Domain for the session cookie. |
| `SESSION_COOKIE_PATH` | `None` | Path for the session cookie. Defaults to `APPLICATION_ROOT`. |
| `SESSION_COOKIE_HTTPONLY` | `True` | Prevent JavaScript access to the cookie. |
| `SESSION_COOKIE_SECURE` | `False` | Only send cookie over HTTPS. |
| `SESSION_COOKIE_SAMESITE` | `None` | Restrict cookie to same-site requests (`"Lax"`, `"Strict"`, `"None"`). |
| `SESSION_COOKIE_PARTITIONED` | `False` | Isolate cookie in third-party context (CHIPS). |
| `SESSION_REFRESH_EACH_REQUEST` | `True` | Re-send cookie on every response to reset expiration. |

---

## 4. Routing and URL Rules

### URL Variable Converters

| Converter | Example | Matches |
|---|---|---|
| `string` (default) | `<username>` | Any text without a slash |
| `int` | `<int:id>` | Positive integers |
| `float` | `<float:weight>` | Positive floating-point numbers |
| `path` | `<path:filename>` | Like `string` but includes slashes |
| `uuid` | `<uuid:item_id>` | UUID strings |

### Route Examples

```python
@app.route("/")
def index():
    return "Home"

@app.route("/user/<username>")
def user_profile(username):
    return f"User: {escape(username)}"

@app.route("/post/<int:post_id>")
def show_post(post_id):
    return f"Post #{post_id}"

@app.route("/files/<path:filepath>")
def serve_file(filepath):
    return send_from_directory("/uploads", filepath)
```

### Trailing Slash Behavior

- `/projects/` — accessible as `/projects/` only; `/projects` redirects to `/projects/`
- `/about` — accessible as `/about` only; `/about/` returns 404

### HTTP Methods

```python
# Multiple methods on one route
@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "POST":
        return do_login()
    return show_form()

# Method-specific decorators
@app.get("/items")
def list_items(): ...

@app.post("/items")
def create_item(): ...

@app.put("/items/<int:id>")
def replace_item(id): ...

@app.patch("/items/<int:id>")
def update_item(id): ...

@app.delete("/items/<int:id>")
def delete_item(id): ...
```

### URL Building

```python
url_for("index")                          # "/"
url_for("user_profile", username="john")  # "/user/john"
url_for("show_post", post_id=5)           # "/post/5"
url_for("static", filename="style.css")   # "/static/style.css"
url_for(".local_view")                    # Relative to current blueprint
url_for("admin.dashboard")               # Blueprint-prefixed endpoint
```

#### `url_for()` Signature

```python
url_for(
    endpoint: str,
    *,
    _anchor: str | None = None,      # Append #anchor
    _method: str | None = None,      # Match specific HTTP method
    _scheme: str | None = None,      # Force URL scheme (http/https)
    _external: bool | None = None,   # Generate absolute URL
    **values: Any,                   # URL variable values; extras become query params
) -> str
```

---

## 5. Request Object

The `request` proxy provides access to the current HTTP request. It extends Werkzeug's `Request`.

### Common Attributes

| Attribute | Type | Description |
|---|---|---|
| `method` | `str` | HTTP method: `"GET"`, `"POST"`, etc. |
| `url` | `str` | Full request URL |
| `base_url` | `str` | URL without query string |
| `path` | `str` | URL path portion |
| `full_path` | `str` | Path with query string |
| `host` | `str` | Host with port |
| `host_url` | `str` | Scheme + host |
| `scheme` | `str` | `"http"` or `"https"` |
| `is_secure` | `bool` | `True` if HTTPS |
| `endpoint` | `str \| None` | Matched endpoint name |
| `url_rule` | `Rule \| None` | Matched URL rule object |
| `view_args` | `dict \| None` | URL variable values |
| `blueprint` | `str \| None` | Current blueprint name |
| `blueprints` | `list[str]` | Blueprint hierarchy |
| `routing_exception` | `HTTPException \| None` | Exception from URL matching |
| `content_type` | `str` | Request `Content-Type` header |
| `content_length` | `int \| None` | Request body length |
| `mimetype` | `str` | MIME type without options |
| `remote_addr` | `str \| None` | Client IP address |
| `referrer` | `str \| None` | Referer header |
| `user_agent` | `UserAgent` | Parsed User-Agent |

### Request Data

| Attribute / Method | Type | Description |
|---|---|---|
| `args` | `ImmutableMultiDict` | Query string parameters (`?key=value`) |
| `form` | `ImmutableMultiDict` | Form body data (POST) |
| `files` | `ImmutableMultiDict` | Uploaded `FileStorage` objects |
| `values` | `CombinedMultiDict` | Combined `args` and `form` |
| `json` | `Any \| None` | Parsed JSON body (or `None`) |
| `data` | `bytes` | Raw request body |
| `get_data(as_text=False)` | `bytes \| str` | Raw body, optionally decoded |
| `get_json(force=False, silent=False)` | `Any \| None` | Parse JSON with options |
| `cookies` | `ImmutableMultiDict` | Request cookies |
| `headers` | `EnvironHeaders` | Request headers |
| `authorization` | `Authorization \| None` | Parsed `Authorization` header |
| `accept_mimetypes` | `MIMEAccept` | Parsed `Accept` header |
| `accept_languages` | `LanguageAccept` | Parsed `Accept-Language` header |
| `is_json` | `bool` | `True` if `Content-Type` is JSON |
| `max_content_length` | `int \| None` | From app config |
| `max_form_memory_size` | `int \| None` | From app config |
| `max_form_parts` | `int \| None` | From app config |

### Accessing Data

```python
# Query params: /search?q=flask&page=2
query = request.args.get("q", "")
page = request.args.get("page", 1, type=int)

# Form data (POST)
username = request.form["username"]
remember = request.form.get("remember", False)

# JSON body
data = request.get_json()
# or simply:
data = request.json

# File uploads
file = request.files["avatar"]
file.save("/uploads/" + secure_filename(file.filename))

# Headers
auth_token = request.headers.get("Authorization")

# Cookies
theme = request.cookies.get("theme", "light")
```

---

## 6. Response Object

Flask's `Response` extends Werkzeug's `Response` with:
- `default_mimetype = "text/html"` (Werkzeug default is `"text/plain"`)
- `max_cookie_size` read from app config

### Creating Responses

```python
# String → 200 text/html
return "Hello"

# Dict or list → JSON response
return {"key": "value"}

# Tuple: (body, status), (body, headers), or (body, status, headers)
return "Created", 201
return "OK", {"X-Custom": "value"}
return "Created", 201, {"Location": "/item/1"}

# Explicit Response
from flask import make_response
resp = make_response(render_template("page.html"))
resp.status_code = 200
resp.headers["X-Custom"] = "value"
resp.set_cookie("user", "john", max_age=3600)
resp.delete_cookie("old_cookie")
return resp
```

### Response Attributes

| Attribute | Type | Description |
|---|---|---|
| `status_code` | `int` | HTTP status code |
| `status` | `str` | Status string (e.g., `"200 OK"`) |
| `headers` | `Headers` | Response headers (mutable) |
| `data` | `bytes` | Response body |
| `mimetype` | `str` | Content MIME type |
| `content_type` | `str` | Full `Content-Type` header |
| `content_length` | `int \| None` | `Content-Length` header |
| `is_json` | `bool` | Whether response is JSON |

### Response Methods

```python
response.set_cookie(
    key: str,
    value: str = "",
    max_age: int | timedelta | None = None,
    expires: datetime | int | float | None = None,
    path: str = "/",
    domain: str | None = None,
    secure: bool = False,
    httponly: bool = False,
    samesite: str | None = None,
)

response.delete_cookie(key, path="/", domain=None)
response.get_json(force=False, silent=False)
```

---

## 7. Context Locals and Proxies

Flask uses context-local proxies to provide thread-safe and async-safe access to application and request data without passing objects through every function.

### Available Proxies

| Proxy | Type | Available In | Description |
|---|---|---|---|
| `current_app` | `Flask` | App or request context | The active application instance |
| `g` | `_AppCtxGlobals` | App or request context | Per-request namespace for custom data |
| `request` | `Request` | Request context only | The current HTTP request |
| `session` | `SecureCookieSession` | Request context only | The current user session |
| `app_ctx` | `AppContext` | App or request context | The active app context object |

### Context Functions

```python
from flask import has_app_context, has_request_context

if has_app_context():
    # current_app and g are available
    ...

if has_request_context():
    # request and session are also available
    ...
```

### The `g` Object

`g` is a per-request namespace. It's reset between requests and is the right place for request-scoped data like database connections or the current user.

```python
from flask import g

@app.before_request
def load_user():
    user_id = session.get("user_id")
    g.user = User.query.get(user_id) if user_id else None

@app.route("/profile")
def profile():
    if g.user is None:
        return redirect(url_for("login"))
    return render_template("profile.html", user=g.user)
```

#### `g` Methods

```python
g.get(name, default=None)       # Like dict.get()
g.pop(name, default=_sentinel)  # Like dict.pop()
g.setdefault(name, default=None)  # Like dict.setdefault()
"key" in g                      # Check if attribute exists
iter(g)                         # Iterate over attribute names
```

### Manual Context Management

```python
# Application context (for CLI scripts, background tasks, etc.)
with app.app_context():
    db.create_all()
    print(current_app.config["DEBUG"])

# Request context (for testing)
with app.test_request_context("/path", method="POST", data={"key": "val"}):
    assert request.path == "/path"
    assert request.method == "POST"
```

### Copying Context for Background Tasks

```python
from flask import copy_current_request_context

@app.route("/process")
def start_process():
    @copy_current_request_context
    def background_task():
        # request, session, g are available here
        process_data(request.json)

    thread = Thread(target=background_task)
    thread.start()
    return "Processing..."
```

---

## 8. Sessions

Sessions store data across requests in a signed cookie (by default). Requires `SECRET_KEY` to be set.

### Basic Usage

```python
from flask import session

app.secret_key = "your-secret-key-here"  # Required!

@app.route("/login", methods=["POST"])
def login():
    session["username"] = request.form["username"]
    session.permanent = True  # Use PERMANENT_SESSION_LIFETIME
    return redirect(url_for("index"))

@app.route("/")
def index():
    username = session.get("username")
    return f"Hello, {username or 'Guest'}"

@app.route("/logout")
def logout():
    session.pop("username", None)
    # Or clear everything:
    session.clear()
    return redirect(url_for("index"))
```

### Session Properties

| Property | Description |
|---|---|
| `session.permanent` | If `True`, session uses `PERMANENT_SESSION_LIFETIME` (default `False`). |
| `session.modified` | Set automatically when session dict changes. Set manually for nested mutable data. |
| `session.accessed` | `True` if the session was read (triggers `Vary: Cookie` header). |
| `session.new` | `True` if the session was newly created (not all backends support this). |

### SessionInterface

To implement a custom session backend (e.g., Redis, database), subclass `SessionInterface`:

```python
from flask.sessions import SessionInterface, SessionMixin

class RedisSession(dict, SessionMixin):
    ...

class RedisSessionInterface(SessionInterface):
    def open_session(self, app: Flask, request: Request) -> SessionMixin | None:
        """Load session data from the request. Return None for a null session."""
        ...

    def save_session(self, app: Flask, session: SessionMixin, response: Response) -> None:
        """Save session data to the response."""
        ...

app.session_interface = RedisSessionInterface()
```

---

## 9. Blueprints

Blueprints organize an application into reusable modules with their own routes, templates, static files, and error handlers.

### Creating and Registering

```python
from flask import Blueprint

# Create
auth = Blueprint(
    "auth",                          # Blueprint name (prefixes endpoints)
    __name__,                        # import_name for resource resolution
    url_prefix="/auth",              # Optional URL prefix for all routes
    template_folder="templates",     # Blueprint-specific templates
    static_folder="static",         # Blueprint-specific static files
    static_url_path="/auth/static", # URL path for static files
)

# Register routes
@auth.route("/login")
def login():
    return render_template("auth/login.html")

# Register with app
app.register_blueprint(auth)
# Or with a URL prefix override:
app.register_blueprint(auth, url_prefix="/authentication")
```

### Blueprint Constructor

```python
Blueprint(
    name: str,
    import_name: str,
    static_folder: str | None = None,
    static_url_path: str | None = None,
    template_folder: str | None = None,
    url_prefix: str | None = None,
    subdomain: str | None = None,
    url_defaults: dict | None = None,
    root_path: str | None = None,
    cli_group: str | None = ...,
)
```

### Blueprint Methods

Blueprints support all the same decorators as the app, scoped to the blueprint:

```python
@auth.route("/login")                # Register route
@auth.before_request                 # Before blueprint requests only
@auth.after_request                  # After blueprint requests only
@auth.teardown_request               # Teardown for blueprint requests
@auth.errorhandler(404)              # Handle errors in this blueprint
@auth.context_processor              # Template context for blueprint templates
@auth.url_value_preprocessor         # Preprocess URL values
@auth.url_defaults                   # Inject URL defaults

# App-wide hooks from a blueprint
@auth.before_app_request             # Before ALL requests
@auth.after_app_request              # After ALL requests
@auth.teardown_app_request           # Teardown for ALL requests
@auth.app_context_processor          # Template context for all templates
@auth.app_errorhandler(404)          # Handle errors app-wide
@auth.app_url_value_preprocessor     # Preprocess ALL URL values
@auth.app_url_defaults               # URL defaults for ALL routes
```

### Blueprint URL Building

```python
# From within the auth blueprint:
url_for(".login")          # → /auth/login (relative to current blueprint)
url_for("auth.login")     # → /auth/login (fully qualified)

# From outside the blueprint:
url_for("auth.login")     # → /auth/login
```

### Nested Blueprints

```python
parent = Blueprint("parent", __name__, url_prefix="/parent")
child = Blueprint("child", __name__, url_prefix="/child")

@child.route("/")
def child_index():
    return "Child index"

parent.register_blueprint(child)
app.register_blueprint(parent)
# URL: /parent/child/
# Endpoint: parent.child.child_index
```

---

## 10. Templates

Flask uses Jinja2 as its template engine. Templates are loaded from the `templates/` folder by default.

### Rendering

```python
from flask import render_template, render_template_string
from flask import stream_template, stream_template_string

# Render from file
@app.route("/")
def index():
    return render_template("index.html", title="Home", items=get_items())

# Render from string
html = render_template_string("<h1>Hello {{ name }}</h1>", name="World")

# Streaming (for large templates, returns Iterator[str])
@app.route("/feed")
def feed():
    return stream_template("feed.html", items=get_many_items())
```

### Function Signatures

```python
render_template(
    template_name_or_list: str | list[str],  # Template name or list of fallback names
    **context: Any,                          # Variables passed to the template
) -> str

render_template_string(
    source: str,       # Jinja2 template source string
    **context: Any,
) -> str

stream_template(
    template_name_or_list: str | list[str],
    **context: Any,
) -> Iterator[str]

stream_template_string(
    source: str,
    **context: Any,
) -> Iterator[str]
```

### Auto-Available Template Variables

These are available in every template without explicitly passing them:

| Variable | Description |
|---|---|
| `config` | The app's configuration (`app.config`) |
| `request` | The current request object |
| `session` | The current session |
| `g` | The request-scoped `g` object |
| `url_for()` | URL building function |
| `get_flashed_messages()` | Flash message retrieval function |

### Jinja2 Autoescaping

Autoescaping is enabled by default for these file extensions:
- `.html`, `.htm`, `.xml`, `.xhtml`, `.svg`

Use the `| safe` filter or `Markup()` only for content you trust completely.

### Template Inheritance

```html
{# base.html #}
<!DOCTYPE html>
<html>
<head><title>{% block title %}{% endblock %}</title></head>
<body>
  {% block content %}{% endblock %}
</body>
</html>

{# page.html #}
{% extends "base.html" %}
{% block title %}My Page{% endblock %}
{% block content %}
  <h1>Hello!</h1>
{% endblock %}
```

### Context Processors

```python
@app.context_processor
def inject_globals():
    return dict(
        current_user=g.get("user"),
        app_version="1.0",
    )
```

### Custom Filters, Tests, and Globals

```python
# Filter: {{ value | myfilter }}
@app.template_filter("myfilter")
def my_filter(value):
    return value.upper()

# Test: {% if value is mytest %}
@app.template_test("mytest")
def my_test(value):
    return isinstance(value, int)

# Global function: {{ myfunc() }}
@app.template_global("myfunc")
def my_func():
    return datetime.now()
```

### Loading Macros from Templates

```python
from flask import get_template_attribute

# If _helpers.html has: {% macro render_nav() %}...{% endmacro %}
render_nav = get_template_attribute("_helpers.html", "render_nav")
html = render_nav()
```

---

## 11. JSON Handling

### jsonify

```python
from flask import jsonify

@app.route("/api/users")
def list_users():
    return jsonify({"users": [{"name": "Alice"}, {"name": "Bob"}]})
    # Returns Response with Content-Type: application/json
```

Note: `dict` and `list` return values from views are automatically converted to JSON responses, so `jsonify` is often unnecessary:

```python
@app.route("/api/users")
def list_users():
    return {"users": [{"name": "Alice"}]}  # Automatic JSON conversion
```

### JSON Module Functions

```python
from flask import json

json.dumps(obj, **kwargs) -> str    # Serialize to JSON string
json.dump(obj, fp, **kwargs)        # Serialize to file
json.loads(s, **kwargs) -> Any      # Deserialize from string
json.load(fp, **kwargs) -> Any      # Deserialize from file
```

### DefaultJSONProvider

The default JSON provider handles these types beyond standard JSON:
- `datetime` → ISO 8601 string
- `date` → ISO 8601 string
- `uuid.UUID` → string
- `decimal.Decimal` → string

### Custom JSON Provider

```python
from flask.json.provider import DefaultJSONProvider

class CustomJSONProvider(DefaultJSONProvider):
    def default(self, o):
        if isinstance(o, MyCustomType):
            return o.to_dict()
        return super().default(o)

app.json_provider_class = CustomJSONProvider
app.json = CustomJSONProvider(app)
```

---

## 12. Error Handling

### Registering Error Handlers

```python
# By HTTP status code
@app.errorhandler(404)
def not_found(error):
    return render_template("errors/404.html"), 404

@app.errorhandler(500)
def server_error(error):
    return render_template("errors/500.html"), 500

# By exception class
@app.errorhandler(ValueError)
def handle_value_error(error):
    return {"error": str(error)}, 400

@app.errorhandler(PermissionError)
def handle_permission(error):
    return {"error": "Forbidden"}, 403

# Catch-all for HTTPException subclasses
from werkzeug.exceptions import HTTPException

@app.errorhandler(HTTPException)
def handle_http_exception(error):
    return {
        "code": error.code,
        "name": error.name,
        "description": error.description,
    }, error.code
```

### Using abort()

```python
from flask import abort

@app.route("/user/<int:id>")
def get_user(id):
    user = User.query.get(id)
    if user is None:
        abort(404, description="User not found")
    return render_template("user.html", user=user)
```

`abort(code)` raises an `HTTPException` with the given status code. Common codes:
- `400` Bad Request
- `401` Unauthorized
- `403` Forbidden
- `404` Not Found
- `405` Method Not Allowed
- `409` Conflict
- `413` Request Entity Too Large
- `422` Unprocessable Entity
- `429` Too Many Requests
- `500` Internal Server Error

### Custom Exception Classes

```python
class APIError(Exception):
    def __init__(self, message, status_code=400, payload=None):
        super().__init__()
        self.message = message
        self.status_code = status_code
        self.payload = payload

    def to_dict(self):
        rv = dict(self.payload or ())
        rv["error"] = self.message
        return rv

@app.errorhandler(APIError)
def handle_api_error(error):
    return jsonify(error.to_dict()), error.status_code

# Usage
@app.route("/api/action")
def action():
    if not valid:
        raise APIError("Invalid input", status_code=422, payload={"field": "name"})
```

### Error Handler Priority

1. Blueprint-specific handlers take precedence over app-level handlers
2. More specific exception classes are matched first
3. 404/405 errors at the routing level are handled by the app, not blueprints (unless registered with `app_errorhandler`)

---

## 13. Signals

Flask uses the `blinker` library for signals. Signals notify subscribers when certain events occur but do **not** modify behavior.

### Built-in Signals

| Signal | Sent When | Sender | Extra Data |
|---|---|---|---|
| `request_started` | Before request dispatch | app | — |
| `request_finished` | After response is generated | app | `response=response` |
| `request_tearing_down` | During request teardown | app | `exc=exception` |
| `got_request_exception` | When exception occurs | app | `exception=exc` |
| `appcontext_pushed` | When app context is pushed | app | — |
| `appcontext_popped` | When app context is popped | app | — |
| `appcontext_tearing_down` | During app context teardown | app | `exc=exception` |
| `before_render_template` | Before template render | app | `template=template, context=context` |
| `template_rendered` | After template render | app | `template=template, context=context` |
| `message_flashed` | When `flash()` is called | app | `message=message, category=category` |

### Subscribing to Signals

```python
from flask import template_rendered

def log_template(sender, template, context, **extra):
    print(f"Rendered {template.name}")

template_rendered.connect(log_template, app)
```

### Creating Custom Signals

```python
from blinker import Namespace

my_signals = Namespace()
user_logged_in = my_signals.signal("user-logged-in")

# Send
user_logged_in.send(app, user=user)

# Subscribe
@user_logged_in.connect_via(app)
def on_login(sender, user, **kwargs):
    log_login(user)
```

### Signals in Testing

```python
from flask import template_rendered

def test_template(app, client):
    recorded = []

    def record(sender, template, context, **extra):
        recorded.append((template, context))

    template_rendered.connect(record, app)
    try:
        client.get("/")
        assert len(recorded) == 1
        assert recorded[0][0].name == "index.html"
    finally:
        template_rendered.disconnect(record, app)
```

---

## 14. Class-Based Views

### View (Generic)

Base class for class-based views. Override `dispatch_request()`.

```python
from flask.views import View

class ShowUsers(View):
    methods = ["GET"]
    decorators = [login_required]
    init_every_request = False  # Reuse instance for efficiency

    def __init__(self, db):
        self.db = db

    def dispatch_request(self):
        users = self.db.query(User).all()
        return render_template("users.html", users=users)

app.add_url_rule("/users", view_func=ShowUsers.as_view("show_users", db=db))
```

#### View Class Attributes

| Attribute | Default | Description |
|---|---|---|
| `methods` | `None` | Allowed HTTP methods. `None` defaults to `["GET", "HEAD", "OPTIONS"]` |
| `decorators` | `[]` | List of decorators applied to the generated view function |
| `provide_automatic_options` | `None` | Whether to auto-handle `OPTIONS` |
| `init_every_request` | `True` | Create new instance per request. Set `False` for efficiency |

#### `View.as_view(name, *class_args, **class_kwargs) -> RouteCallable`
Convert the class into a view function for `add_url_rule()`. Extra args are passed to `__init__`.

### MethodView (REST-style)

Dispatches to methods named after HTTP verbs: `get()`, `post()`, `put()`, `patch()`, `delete()`, `head()`, `options()`, `trace()`.

`methods` is automatically set based on which methods are defined.

```python
from flask.views import MethodView

class ItemAPI(MethodView):
    def get(self, id=None):
        if id is None:
            return jsonify(get_all_items())
        return jsonify(get_item(id))

    def post(self):
        item = create_item(request.json)
        return jsonify(item), 201

    def put(self, id):
        item = replace_item(id, request.json)
        return jsonify(item)

    def patch(self, id):
        item = update_item(id, request.json)
        return jsonify(item)

    def delete(self, id):
        delete_item(id)
        return "", 204

# Register with separate rules for collection and item
view = ItemAPI.as_view("items")
app.add_url_rule("/items/", defaults={"id": None}, view_func=view, methods=["GET"])
app.add_url_rule("/items/", view_func=view, methods=["POST"])
app.add_url_rule("/items/<int:id>", view_func=view, methods=["GET", "PUT", "PATCH", "DELETE"])
```

---

## 15. Helper Functions

### `redirect(location, code=303)`
Create a redirect response.

```python
from flask import redirect, url_for

@app.route("/old-page")
def old_page():
    return redirect(url_for("new_page"))

# With custom status code
return redirect(url_for("login"), code=302)
```

Note: Default redirect code is `303` (changed from `302` in Flask 3.2).

### `abort(code, *args, **kwargs)`
Raise an HTTP exception.

```python
from flask import abort
abort(404)
abort(404, description="Not found")
```

### `make_response(*args)`
Convert view return value to a `Response` object for further modification.

```python
from flask import make_response

resp = make_response(render_template("page.html"))
resp.set_cookie("visited", "yes")
return resp

# With status code
resp = make_response("Not Found", 404)
```

### `flash(message, category="message")`
Flash a message to the next request's template.

```python
from flask import flash

flash("Account created successfully!")
flash("Invalid password.", "error")
flash("Your session will expire soon.", "warning")
```

### `get_flashed_messages(with_categories=False, category_filter=())`
Retrieve flashed messages (consumed on read).

```python
# In Python
messages = get_flashed_messages()
messages = get_flashed_messages(with_categories=True)  # [(category, message), ...]
messages = get_flashed_messages(category_filter=["error"])
```

```html
{# In Jinja2 template #}
{% with messages = get_flashed_messages(with_categories=true) %}
  {% for category, message in messages %}
    <div class="alert alert-{{ category }}">{{ message }}</div>
  {% endfor %}
{% endwith %}
```

### `send_file(...)`

```python
send_file(
    path_or_file: PathLike | str | IO[bytes],
    mimetype: str | None = None,
    as_attachment: bool = False,
    download_name: str | None = None,
    conditional: bool = True,
    etag: bool | str = True,
    last_modified: datetime | int | float | None = None,
    max_age: int | Callable | None = None,
) -> Response
```

Send a file to the client. Supports conditional responses and range requests.

```python
from flask import send_file

@app.route("/download/<filename>")
def download(filename):
    return send_file(f"/data/files/{filename}", as_attachment=True)
```

**Warning:** Never pass user-supplied paths directly. Use `send_from_directory()` instead.

### `send_from_directory(directory, path, **kwargs)`
Safely serve a file from a directory. Prevents directory traversal attacks.

```python
from flask import send_from_directory

@app.route("/uploads/<path:filename>")
def uploaded_file(filename):
    return send_from_directory(app.config["UPLOAD_FOLDER"], filename)
```

### `stream_with_context(generator_or_function)`
Keep the request context alive during streaming responses.

```python
from flask import stream_with_context, Response

@app.route("/stream")
def stream():
    @stream_with_context
    def generate():
        yield "Hello "
        yield request.args["name"]
    return Response(generate(), content_type="text/plain")
```

### `get_template_attribute(template_name, attribute)`
Load a macro or variable from a template.

```python
# If _macros.html has: {% macro hello(name) %}Hello {{ name }}{% endmacro %}
hello = get_template_attribute("_macros.html", "hello")
result = hello("World")  # "Hello World"
```

---

## 16. Testing

### Test Setup with pytest

```python
import pytest
from myapp import create_app

@pytest.fixture()
def app():
    app = create_app()
    app.config.update({
        "TESTING": True,
        "SECRET_KEY": "test",
    })
    yield app

@pytest.fixture()
def client(app):
    return app.test_client()

@pytest.fixture()
def runner(app):
    return app.test_cli_runner()
```

### FlaskClient

The test client simulates requests without running a server.

```python
def test_home(client):
    response = client.get("/")
    assert response.status_code == 200
    assert b"Welcome" in response.data

def test_login(client):
    response = client.post("/login", data={
        "username": "admin",
        "password": "secret",
    })
    assert response.status_code == 303  # Redirect

def test_json_api(client):
    response = client.post("/api/items", json={
        "name": "Widget",
        "price": 9.99,
    })
    assert response.status_code == 201
    data = response.get_json()
    assert data["name"] == "Widget"

def test_headers(client):
    response = client.get("/api/data", headers={
        "Authorization": "Bearer token123",
        "Accept": "application/json",
    })
    assert response.is_json
```

### Session Manipulation in Tests

```python
def test_session(client):
    with client.session_transaction() as sess:
        sess["user_id"] = 42

    response = client.get("/profile")
    assert response.status_code == 200
```

### FlaskCliRunner

```python
def test_cli_command(runner):
    result = runner.invoke(args=["init-db"])
    assert "Initialized" in result.output
    assert result.exit_code == 0
```

### Test Request Context

```python
def test_context(app):
    with app.test_request_context("/users?page=2"):
        assert request.path == "/users"
        assert request.args["page"] == "2"

    with app.app_context():
        assert current_app.config["TESTING"] is True
```

---

## 17. CLI Integration

### Application Discovery

```bash
# Auto-discover app in module
flask --app myapp run

# Specify app instance
flask --app "myapp:create_app('dev')" run

# Use specific object
flask --app myapp:my_app run
```

Environment variable: `FLASK_APP=myapp`

### Development Server

```bash
flask run                        # Default: 127.0.0.1:5000
flask run --debug                # Enable debugger + reloader
flask run --host 0.0.0.0         # Bind to all interfaces
flask run --port 8080            # Custom port
flask run --extra-files file.txt # Watch extra files for reload
flask run --cert adhoc            # HTTPS with auto-generated cert
```

### Custom CLI Commands

```python
import click

@app.cli.command("init-db")
@click.option("--drop", is_flag=True, help="Drop existing tables.")
def init_db(drop):
    """Initialize the database."""
    if drop:
        db.drop_all()
    db.create_all()
    click.echo("Database initialized.")
```

```bash
flask init-db --drop
```

### Command Groups

```python
@app.cli.group()
def user():
    """User management commands."""

@user.command()
@click.argument("name")
def create(name):
    """Create a new user."""
    ...
```

```bash
flask user create admin
```

### Shell Context

```python
@app.shell_context_processor
def make_context():
    return {"db": db, "User": User, "Post": Post}
```

```bash
flask shell
>>> User.query.all()  # db, User, Post are pre-imported
```

---

## 18. Request/Response Lifecycle

### Complete Request Flow

```
Client Request
    │
    ▼
Flask.__call__(environ, start_response)
    │
    ▼
Flask.wsgi_app(environ, start_response)
    │
    ▼
AppContext created from environ (wraps Request)
    │
    ▼
ctx.push()
    ├── Sets current_app, g, request, session proxies
    └── Signal: appcontext_pushed
    │
    ▼
ctx.match_request()  ── URL routing
    │
    ▼
full_dispatch_request(ctx)
    │
    ├── Signal: request_started
    │
    ├── url_value_preprocessors called
    │
    ├── preprocess_request(ctx)
    │   └── Calls before_request handlers
    │       (if any returns non-None, skip view)
    │
    ├── dispatch_request(ctx)
    │   └── Calls the matched view function
    │
    ├── finalize_request(ctx, rv)
    │   ├── make_response(rv)  ── Convert return value to Response
    │   ├── process_response(ctx, response)
    │   │   ├── Calls after_this_request handlers
    │   │   ├── Calls after_request handlers
    │   │   └── session_interface.save_session()
    │   └── Signal: request_finished
    │
    ▼
ctx.pop(exc)
    ├── do_teardown_request(ctx, exc)
    │   ├── Calls teardown_request handlers
    │   └── Signal: request_tearing_down
    ├── do_teardown_appcontext(ctx, exc)
    │   ├── Calls teardown_appcontext handlers
    │   └── Signal: appcontext_tearing_down
    ├── Signal: appcontext_popped
    └── Resets context variables
    │
    ▼
WSGI Response returned to server
```

### Exception Handling Flow

```
Exception in view function
    │
    ▼
handle_user_exception(ctx, e)
    ├── If PassException: re-raise
    ├── If HTTPException: handle_http_exception(ctx, e)
    │   ├── Check TRAP_HTTP_EXCEPTIONS
    │   ├── Look up errorhandler for status code
    │   └── Return error response
    └── Else: handle_exception(ctx, e)
        ├── Signal: got_request_exception
        ├── Check PROPAGATE_EXCEPTIONS / TESTING / DEBUG
        ├── Log exception
        ├── Look up errorhandler for exception class
        └── Return 500 error response
```

---

## 19. Common Patterns

### Application Factory

```python
# myapp/__init__.py
from flask import Flask

def create_app(config_name="default"):
    app = Flask(__name__)
    app.config.from_object(f"myapp.config.{config_name}")

    # Initialize extensions
    from myapp.extensions import db, migrate
    db.init_app(app)
    migrate.init_app(app, db)

    # Register blueprints
    from myapp.views.main import main_bp
    from myapp.views.auth import auth_bp
    from myapp.views.api import api_bp
    app.register_blueprint(main_bp)
    app.register_blueprint(auth_bp, url_prefix="/auth")
    app.register_blueprint(api_bp, url_prefix="/api/v1")

    return app
```

### Database Connection Pattern (SQLite)

```python
import sqlite3
from flask import g

def get_db():
    if "db" not in g:
        g.db = sqlite3.connect(current_app.config["DATABASE"])
        g.db.row_factory = sqlite3.Row
    return g.db

@app.teardown_appcontext
def close_db(exception):
    db = g.pop("db", None)
    if db is not None:
        db.close()
```

### Login Required Decorator

```python
from functools import wraps
from flask import g, redirect, url_for, request

def login_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        if g.user is None:
            return redirect(url_for("auth.login", next=request.url))
        return f(*args, **kwargs)
    return decorated

@app.route("/dashboard")
@login_required
def dashboard():
    return render_template("dashboard.html")
```

### File Upload

```python
from werkzeug.utils import secure_filename

ALLOWED_EXTENSIONS = {"png", "jpg", "jpeg", "gif"}

def allowed_file(filename):
    return "." in filename and filename.rsplit(".", 1)[1].lower() in ALLOWED_EXTENSIONS

@app.route("/upload", methods=["POST"])
def upload():
    if "file" not in request.files:
        flash("No file selected")
        return redirect(request.url)

    file = request.files["file"]
    if file.filename == "":
        flash("No file selected")
        return redirect(request.url)

    if file and allowed_file(file.filename):
        filename = secure_filename(file.filename)
        file.save(os.path.join(app.config["UPLOAD_FOLDER"], filename))
        return redirect(url_for("uploaded", filename=filename))
```

### Streaming Response

```python
from flask import Response, stream_with_context

@app.route("/large-data")
def stream_data():
    @stream_with_context
    def generate():
        for row in query_large_dataset():
            yield f"{row}\n"
    return Response(generate(), mimetype="text/plain")
```

### WSGI Middleware

```python
from werkzeug.middleware.proxy_fix import ProxyFix

app = Flask(__name__)
# Fix headers when behind a reverse proxy (nginx, etc.)
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1)
```

### Logging Configuration

```python
from logging.config import dictConfig

# Configure BEFORE creating app
dictConfig({
    "version": 1,
    "formatters": {
        "default": {
            "format": "[%(asctime)s] %(levelname)s in %(module)s: %(message)s",
        }
    },
    "handlers": {
        "wsgi": {
            "class": "logging.StreamHandler",
            "stream": "ext://flask.logging.wsgi_errors_stream",
            "formatter": "default",
        }
    },
    "root": {
        "level": "INFO",
        "handlers": ["wsgi"],
    },
})

app = Flask(__name__)
app.logger.info("App started")
```

### Async Views

Flask supports async views natively:

```python
@app.route("/async-data")
async def async_data():
    data = await fetch_data_async()
    return jsonify(data)
```

Async `before_request`, `after_request`, `teardown_request`, and `errorhandler` functions are also supported.

---

## 20. Security Considerations

### Cross-Site Scripting (XSS)

- Jinja2 auto-escapes HTML in templates for `.html`, `.htm`, `.xml`, `.xhtml`, `.svg` files
- Only use `| safe` filter or `Markup()` for trusted content
- Always quote HTML attributes: `<input value="{{ value }}">`
- Use `markupsafe.escape()` when generating HTML in Python code

### Cross-Site Request Forgery (CSRF)

- Flask does not include CSRF protection by default
- Use Flask-WTF or similar extension for form CSRF tokens
- Session cookies with `SameSite=Lax` or `Strict` provide additional protection

### Session Security

```python
app.config.update(
    SECRET_KEY="use-a-long-random-string-here",
    SESSION_COOKIE_HTTPONLY=True,     # Prevent JS access (default)
    SESSION_COOKIE_SECURE=True,       # HTTPS only
    SESSION_COOKIE_SAMESITE="Lax",    # CSRF mitigation
)
```

### Request Size Limits

```python
app.config.update(
    MAX_CONTENT_LENGTH=16 * 1024 * 1024,  # 16 MB max upload
    MAX_FORM_MEMORY_SIZE=500_000,          # 500 KB per form field
    MAX_FORM_PARTS=1_000,                  # 1000 form fields max
)
```

### Host Header Validation

```python
app.config["TRUSTED_HOSTS"] = ["example.com", ".example.com"]
```

### Safe File Serving

- Never pass user-supplied paths to `send_file()` — use `send_from_directory()` instead
- Always use `werkzeug.utils.secure_filename()` for uploaded filenames

---

## 21. Deployment

### Production WSGI Servers

**Never use `flask run` or `app.run()` in production.** Use a production WSGI server:

```bash
# Gunicorn (Linux/macOS)
pip install gunicorn
gunicorn -w 4 -b 0.0.0.0:8000 "myapp:create_app()"

# Waitress (Windows/cross-platform)
pip install waitress
waitress-serve --port=8000 --call "myapp:create_app"

# uWSGI
pip install uwsgi
uwsgi --http 0.0.0.0:8000 --master -p 4 -w "myapp:create_app()"
```

### Reverse Proxy Setup

When behind nginx/Apache, use `ProxyFix` to trust forwarded headers:

```python
from werkzeug.middleware.proxy_fix import ProxyFix
app.wsgi_app = ProxyFix(app.wsgi_app, x_for=1, x_proto=1, x_host=1, x_prefix=1)
```

### Environment Variables

```bash
export FLASK_APP=myapp
export FLASK_DEBUG=0           # Never enable debug in production
export FLASK_SECRET_KEY="..."  # Set via environment, not code
```

### `.flaskenv` and `.env` Files

Flask auto-loads `.flaskenv` (for Flask-specific vars) and `.env` (for other vars) if `python-dotenv` is installed:

```ini
# .flaskenv (commit to VCS)
FLASK_APP=myapp

# .env (do NOT commit)
SECRET_KEY=super-secret-key
DATABASE_URL=postgresql://...
```

---

## Appendix: Type Aliases (from `flask.typing`)

| Type | Definition |
|---|---|
| `ResponseValue` | `Response \| str \| bytes \| list[Any] \| Mapping[str, Any] \| Iterator[str] \| Iterator[bytes]` |
| `ResponseReturnValue` | `ResponseValue \| tuple[ResponseValue, HeadersValue] \| tuple[ResponseValue, int] \| tuple[ResponseValue, int, HeadersValue] \| WSGIApplication` |
| `HeadersValue` | `Mapping[str, str] \| Headers \| Sequence[tuple[str, str]]` |
| `RouteCallable` | `Callable[..., ResponseReturnValue]` |
| `BeforeRequestCallable` | `Callable[[], ResponseReturnValue \| None]` |
| `AfterRequestCallable` | `Callable[[Response], Response]` |
| `TeardownCallable` | `Callable[[BaseException \| None], None]` |
| `ErrorHandlerCallable` | `Callable[[Any], ResponseReturnValue]` |
| `TemplateContextProcessorCallable` | `Callable[[], dict[str, Any]]` |
| `ShellContextProcessorCallable` | `Callable[[], dict[str, Any]]` |
