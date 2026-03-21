# Pyramid Web Framework — Comprehensive Reference

> **Version:** 2.1 / 2.2.dev0
> **Generated from source:** `src/pyramid/`
> **Purpose:** Detailed API reference for humans and AI building with Pyramid.

---

## Table of Contents

1. [Quick Start & Core Concepts](#1-quick-start--core-concepts)
2. [Application Configuration (Configurator)](#2-application-configuration-configurator)
3. [Route Configuration](#3-route-configuration)
4. [View Configuration](#4-view-configuration)
5. [Request Object](#5-request-object)
6. [Response Object](#6-response-object)
7. [URL Generation](#7-url-generation)
8. [Renderers](#8-renderers)
9. [Security](#9-security)
10. [Authentication Helpers](#10-authentication-helpers)
11. [Authorization & ACLs](#11-authorization--acls)
12. [CSRF Protection](#12-csrf-protection)
13. [Sessions](#13-sessions)
14. [Events](#14-events)
15. [Traversal](#15-traversal)
16. [HTTP Exceptions](#16-http-exceptions)
17. [Tweens](#17-tweens)
18. [Static Assets](#18-static-assets)
19. [Internationalization (i18n)](#19-internationalization-i18n)
20. [Testing Utilities](#20-testing-utilities)
21. [Scripting & CLI](#21-scripting--cli)
22. [Settings & Environment](#22-settings--environment)
23. [View Derivers](#23-view-derivers)
24. [Utility Functions & Classes](#24-utility-functions--classes)
25. [WSGI Helpers](#25-wsgi-helpers)
26. [Extending Pyramid Applications](#26-extending-pyramid-applications)
27. [Deployment & PasteDeploy](#27-deployment--pastedeploy)
28. [Common Patterns & Best Practices](#28-common-patterns--best-practices)

---

## 1. Quick Start & Core Concepts

### Minimal Application

```python
from wsgiref.simple_server import make_server
from pyramid.config import Configurator
from pyramid.response import Response

def hello_world(request):
    return Response('Hello World!')

if __name__ == '__main__':
    with Configurator() as config:
        config.add_route('hello', '/')
        config.add_view(hello_world, route_name='hello')
        app = config.make_wsgi_app()
    server = make_server('0.0.0.0', 6543, app)
    server.serve_forever()
```

### Declarative Style (Preferred)

```python
from pyramid.config import Configurator
from pyramid.response import Response
from pyramid.view import view_config

@view_config(route_name='hello', renderer='json')
def hello_world(request):
    return {'message': 'Hello World!'}

def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.add_route('hello', '/')
    config.scan()  # Finds @view_config decorators
    return config.make_wsgi_app()
```

### Core Architecture

- **Configurator** — Central API for wiring up the application.
- **Router** — The WSGI application that processes requests.
- **Registry** — Component registry holding all configuration (Zope Component Architecture).
- **Request** — WebOb-based request object, enriched with Pyramid methods.
- **Response** — WebOb-based response object.
- **View callable** — Function or class that handles a request and returns a response.
- **Route** — Maps a URL pattern to a view via URL dispatch.
- **Traversal** — Alternative to URL dispatch; walks a resource tree.
- **Tween** — Middleware-like wrapper around the request handler.
- **Renderer** — Converts a view's return value into a Response.
- **Event** — Broadcast notifications at key points in request processing.

### Two Configuration Styles

| Style | Mechanism | When Applied |
|-------|-----------|-------------|
| **Imperative** | `config.add_view(...)`, `config.add_route(...)` | Immediately (or at commit) |
| **Declarative** | `@view_config(...)` + `config.scan()` | When `scan()` discovers decorators |

Decorators do nothing until `config.scan()` is called.

---

## 2. Application Configuration (Configurator)

**Module:** `pyramid.config`

### Constructor

```python
Configurator(
    registry=None,                  # Registry instance; None creates a new one
    package=None,                   # Python package for relative path resolution
    settings=None,                  # dict of deployment settings
    root_factory=None,              # Default root factory callable or dotted name
    security_policy=None,           # ISecurityPolicy instance or dotted name
    authentication_policy=None,     # DEPRECATED 2.0 — use security_policy
    authorization_policy=None,      # DEPRECATED 2.0 — use security_policy
    renderers=None,                 # List of (name, factory) tuples; None = defaults
    debug_logger=None,              # Logger instance or name string
    locale_negotiator=None,         # Locale negotiator callable or dotted name
    request_factory=None,           # Request factory or dotted name
    response_factory=None,          # Response factory or dotted name
    default_permission=None,        # Permission string for all views by default
    session_factory=None,           # Session factory callable
    default_view_mapper=None,       # IViewMapperFactory or dotted name
    autocommit=False,               # True = immediate actions, no conflict detection
    exceptionresponse_view=default_exceptionresponse_view,
    route_prefix=None,              # String prepended to all route patterns
    introspection=True,             # Enable/disable introspection
    root_package=None,              # Root package for resource location
)
```

Supports context manager usage:

```python
with Configurator(settings=settings) as config:
    config.add_route('home', '/')
    config.scan()
    app = config.make_wsgi_app()
```

### Key Methods

#### `include(callable, route_prefix=None)`

Include external configuration. `callable` can be a function, dotted name, or module (searches for `includeme` function).

```python
# In myapp/__init__.py:
config.include('myapp.views')
config.include('myapp.api', route_prefix='api/v1')

# In myapp/views.py:
def includeme(config):
    config.add_route('home', '/')
    config.scan('.views')
```

#### `scan(package=None, categories=('pyramid',), ignore=None)`

Scan a package for configuration decorators (`@view_config`, `@subscriber`, etc.).

```python
config.scan()                     # Scan caller's package
config.scan('myapp.views')        # Scan specific package
config.scan(ignore='myapp.tests') # Skip test modules
```

#### `make_wsgi_app()`

Commit configuration, fire `ApplicationCreated` event, return the WSGI application (Router).

#### `add_directive(name, directive, action_wrap=True)`

Add a custom configuration method to the Configurator.

```python
def add_custom_route(config, name, pattern):
    config.add_route(name, pattern)
    config.add_view(route_name=name, renderer='json')

config.add_directive('add_custom_route', add_custom_route)
config.add_custom_route('api_users', '/api/users')
```

#### `with_package(package)`

Return a new Configurator with the same registry but a different package context.

#### `maybe_dotted(dotted)`

Resolve a dotted Python name to an object. Returns non-strings as-is.

#### `begin(request=None)` / `end()`

Push/pop the registry and request onto the thread-local stack.

---

## 3. Route Configuration

**Module:** `pyramid.config` (RoutesConfiguratorMixin)

### `add_route(...)`

```python
config.add_route(
    name,                       # REQUIRED — unique route name
    pattern=None,               # URL pattern, e.g. '/items/{id}'
    factory=None,               # Root factory override for this route
    header=None,                # Header predicate: string or iterable
    xhr=None,                   # True = require X-Requested-With header
    accept=None,                # Media type string or list for Accept matching
    path_info=None,             # Regex tested against PATH_INFO
    request_method=None,        # HTTP method string or tuple ('GET', 'POST', ...)
    request_param=None,         # Query/body param: 'key' or 'key=value'
    traverse=None,              # Traversal path pattern using match variables
    use_global_views=False,     # Fall back to views without route_name
    pregenerator=None,          # URL generation pre-processor
    static=False,               # True = URL generation only, never matches
    inherit_slash=None,         # Inherit trailing slash from prefix (for empty pattern)
    is_authenticated=None,      # True/False — match auth state
    **predicates,               # Custom predicates
)
```

### Pattern Syntax

```python
# Simple replacement markers
config.add_route('user', '/users/{id}')

# With regex constraint
config.add_route('user', '/users/{id:\d+}')

# Remainder/stararg (captured as tuple)
config.add_route('pages', '/pages/*subpath')

# Compound segments
config.add_route('file', '/files/{name}.{ext}')

# External route (auto-static, for URL generation only)
config.add_route('external', 'https://example.com/{path}')
```

### Route Prefix Context

```python
with config.route_prefix_context('api/v1'):
    config.add_route('users', '/users')      # pattern = 'api/v1/users'
    config.add_route('items', '/items')       # pattern = 'api/v1/items'
```

### Matched Route Data in Views

```python
@view_config(route_name='user')
def user_view(request):
    user_id = request.matchdict['id']         # Route variables
    page = request.matched_route.name         # Route name: 'user'
```

### Custom Route Predicates

```python
config.add_route_predicate('is_api', ApiPredicate)
config.add_route('api', '/api', is_api=True)
```

### Important: Route Order Matters

Routes are matched in the order they are added. More specific routes must be added before more general ones.

---

## 4. View Configuration

**Module:** `pyramid.config` (ViewsConfiguratorMixin), `pyramid.view`

### `add_view(...)`

```python
config.add_view(
    view=None,                  # View callable, class, or dotted name
    name="",                    # View name (for traversal)
    permission=None,            # Permission string; NO_PERMISSION_REQUIRED for anonymous
    route_name=None,            # Must match a route name from add_route
    request_method=None,        # HTTP method string or tuple
    request_param=None,         # Query/body param matching
    containment=None,           # Class/interface in context lineage
    attr=None,                  # Method name on class-based views
    renderer=None,              # Renderer name or template path
    wrapper=None,               # View name of wrapping view
    xhr=None,                   # True = require XMLHttpRequest
    accept=None,                # Single media type string
    header=None,                # Header predicate
    path_info=None,             # Regex against PATH_INFO
    context=None,               # Class/interface the context must match
    decorator=None,             # Callable or iterable wrapping the view
    mapper=None,                # IViewMapperFactory
    http_cache=None,            # int seconds, timedelta, 0, or (value, dict)
    match_param=None,           # 'key=value' for matchdict matching
    require_csrf=None,          # True/False/None — CSRF check control
    exception_only=False,       # True = register only as exception view
    is_authenticated=None,      # True/False — match auth state
    physical_path=None,         # Physical traversal path match
    **view_options,             # Custom predicates
)
```

### `@view_config` Decorator

```python
from pyramid.view import view_config

@view_config(route_name='home', renderer='json', request_method='GET')
def home(request):
    return {'status': 'ok'}
```

Accepts all the same arguments as `add_view`. Requires `config.scan()` to take effect.

### `@view_defaults` Class Decorator

```python
from pyramid.view import view_defaults, view_config

@view_defaults(route_name='users', renderer='json')
class UserViews:
    def __init__(self, request):
        self.request = request

    @view_config(request_method='GET')
    def list(self):
        return {'users': [...]}

    @view_config(request_method='POST')
    def create(self):
        return {'created': True}
```

### View Callable Signatures

```python
# Function view (request only) — most common
def my_view(request):
    return Response('Hello')

# Function view (context, request) — for traversal
def my_view(context, request):
    return Response('Hello')

# Class-based view
class MyView:
    def __init__(self, request):
        self.request = request

    def __call__(self):
        return Response('Hello')

    # Or use attr= to call a specific method:
    @view_config(route_name='list', attr='list_items')
    def list_items(self):
        return Response('Items')
```

### Special View Registrations

#### Not Found (404) View

```python
from pyramid.view import notfound_view_config

@notfound_view_config(renderer='templates/404.jinja2')
def notfound(request):
    request.response.status = 404
    return {}

# Or with slash-appending redirect:
@notfound_view_config(append_slash=True)
def notfound(request):
    return Response('Not Found', status='404 Not Found')
```

#### Forbidden (403) View

```python
from pyramid.view import forbidden_view_config

@forbidden_view_config(renderer='templates/403.jinja2')
def forbidden(request):
    return {}
```

#### Exception Views

```python
from pyramid.view import exception_view_config

@exception_view_config(ValueError, renderer='json')
def value_error_view(exc, request):
    request.response.status = 400
    return {'error': str(exc)}
```

### `http_cache` Option

```python
@view_config(http_cache=3600)                          # Cache 1 hour
@view_config(http_cache=0)                             # No cache
@view_config(http_cache=(3600, {'public': True}))      # Cache 1hr, public
@view_config(http_cache=None)                          # No cache headers
```

### View Decorators

```python
def timing_decorator(view):
    def wrapper(context, request):
        import time
        start = time.time()
        response = view(context, request)
        response.headers['X-Time'] = str(time.time() - start)
        return response
    return wrapper

@view_config(route_name='home', decorator=timing_decorator)
def home(request):
    return Response('Hello')

# Multiple decorators (applied in order; first is outermost):
@view_config(decorator=(outer_decorator, inner_decorator))
```

### Static Views

```python
# Local static files
config.add_static_view('static', 'myapp:static', cache_max_age=3600)

# CDN-hosted static files
config.add_static_view('https://cdn.example.com/static', 'myapp:static')
```

---

## 5. Request Object

**Module:** `pyramid.request`
**Base:** `webob.BaseRequest`

The Request object is the primary interface to the incoming HTTP request. It extends WebOb's `BaseRequest` with Pyramid-specific functionality via mixins.

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `matchdict` | `dict` or `None` | Matched URL dispatch route variables |
| `matched_route` | `IRoute` or `None` | The matched route object |
| `context` | object | The traversal context resource |
| `root` | object | The root resource |
| `subpath` | tuple | Remaining path segments after traversal |
| `exception` | `Exception` or `None` | Set during exception view processing |
| `exc_info` | tuple or `None` | Exception info during exception view processing |
| `response` | `Response` | Response object (reified; for setting cookies/headers in views) |
| `session` | `ISession` | Session object (reified; requires session factory) |
| `registry` | `Registry` | The application registry |
| `localizer` | `Localizer` | i18n localizer (reified) |
| `locale_name` | `str` | Current locale name (reified) |

### WebOb Request Attributes (inherited)

| Attribute | Type | Description |
|-----------|------|-------------|
| `method` | `str` | HTTP method: `'GET'`, `'POST'`, etc. |
| `url` | `str` | Full request URL |
| `path` | `str` | URL path portion |
| `path_info` | `str` | PATH_INFO |
| `script_name` | `str` | SCRIPT_NAME |
| `host` | `str` | Host header value |
| `host_url` | `str` | Scheme + host |
| `application_url` | `str` | Scheme + host + script_name |
| `path_url` | `str` | URL without query string |
| `params` | `NestedMultiDict` | Combined GET + POST parameters |
| `GET` | `MultiDict` | Query string parameters |
| `POST` | `MultiDict` | Request body parameters |
| `json_body` | parsed | Parsed JSON request body |
| `json` | parsed | Alias for `json_body` |
| `text` | `str` | Request body as text |
| `body` | `bytes` | Raw request body |
| `cookies` | `dict` | Cookie values |
| `headers` | `EnvironHeaders` | Request headers |
| `content_type` | `str` | Content-Type header |
| `accept` | `AcceptValidHeader` | Accept header |
| `remote_addr` | `str` | Client IP address |
| `environ` | `dict` | Raw WSGI environ |
| `charset` | `str` | Request charset (default UTF-8) |

### Security Properties (from SecurityAPIMixin)

| Property/Method | Description |
|-----------------|-------------|
| `identity` | Opaque identity object from security policy, or `None` |
| `authenticated_userid` | User ID string, or `None` |
| `is_authenticated` | `True` if `authenticated_userid is not None` |
| `has_permission(permission, context=None)` | Returns `Allowed` or `Denied` |

### Callback Methods (from CallbackMethodsMixin)

```python
# Called after response is created (FIFO):
request.add_response_callback(lambda request, response: ...)

# Called unconditionally at end of request (FIFO, even on exceptions):
request.add_finished_callback(lambda request: ...)
```

### URL Generation Methods (from URLMethodsMixin)

See [URL Generation](#7-url-generation).

### Invoke Subrequest

```python
from pyramid.request import Request

def view_one(request):
    subreq = Request.blank('/other_view')
    response = request.invoke_subrequest(subreq, use_tweens=False)
    return response
```

### `RequestLocalCache`

Per-request caching utility (added 2.0):

```python
from pyramid.request import RequestLocalCache

@RequestLocalCache()
def get_current_user(request):
    return request.dbsession.query(User).get(request.authenticated_userid)

# In views:
user = get_current_user.get_or_create(request)
```

### Adding Methods/Properties to Request

```python
# As a reified property (computed once per request):
def db(request):
    conn = SomeDBConnection()
    request.add_finished_callback(lambda req: conn.close())
    return conn

config.add_request_method(db, reify=True)
# Now: request.db

# As a regular method:
config.add_request_method(some_callable, name='do_thing')
# Now: request.do_thing(...)
```

---

## 6. Response Object

**Module:** `pyramid.response`
**Base:** `webob.Response`

### Class: `Response`

Standard WebOb Response. Implements `IResponse`.

```python
from pyramid.response import Response

response = Response(body='Hello', status='200 OK', content_type='text/plain')
response.set_cookie('key', 'value', max_age=3600)
response.headers['X-Custom'] = 'value'
```

### Key Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `status` | `str` | Status line: `'200 OK'` |
| `status_int` | `int` | Status code: `200` |
| `body` | `bytes` | Response body |
| `text` | `str` | Response body as text |
| `json_body` | parsed | Response body as parsed JSON |
| `content_type` | `str` | Content-Type |
| `charset` | `str` | Response charset |
| `headers` | `ResponseHeaders` | Response headers |
| `headerlist` | `list` | Headers as list of tuples |
| `content_length` | `int` | Content-Length |
| `cache_control` | `CacheControl` | Cache-Control header object |

### `FileResponse`

```python
from pyramid.response import FileResponse

response = FileResponse(
    '/path/to/file.pdf',
    request=request,            # Needed for wsgi.file_wrapper
    cache_max_age=3600,
    content_type='application/pdf',
)
```

### `response_adapter` Decorator

```python
from pyramid.response import response_adapter

@response_adapter(str)
def string_adapter(s):
    return Response(s)
```

---

## 7. URL Generation

**Module:** `pyramid.url` (mixed into Request via `URLMethodsMixin`)

### `request.route_url(route_name, *elements, **kw)`

Generate a fully qualified URL for a named route.

```python
# Given: config.add_route('user', '/users/{id}')
url = request.route_url('user', id=42)
# -> 'http://example.com/users/42'

url = request.route_url('user', id=42, _query={'tab': 'profile'})
# -> 'http://example.com/users/42?tab=profile'

url = request.route_url('user', id=42, _anchor='top')
# -> 'http://example.com/users/42#top'

url = request.route_url('user', id=42, _scheme='https', _host='api.example.com')
# -> 'https://api.example.com/users/42'
```

**Special keyword arguments (with underscore prefix):**

| Key | Description |
|-----|-------------|
| `_query` | Query string: string, dict, or sequence of 2-tuples |
| `_anchor` | URL fragment |
| `_app_url` | Override protocol/host/port/path prefix |
| `_scheme` | Override URL scheme |
| `_host` | Override hostname |
| `_port` | Override port |

### `request.route_path(route_name, *elements, **kw)`

Like `route_url` but returns a relative path (no scheme/host/port).

```python
path = request.route_path('user', id=42)
# -> '/users/42'
```

### `request.resource_url(resource, *elements, **kw)`

Generate a URL for a traversal resource. Note: keyword arguments do **not** have underscore prefixes (unlike `route_url`).

```python
url = request.resource_url(resource)
# -> 'http://example.com/path/to/resource/'

url = request.resource_url(resource, 'edit')
# -> 'http://example.com/path/to/resource/edit'

url = request.resource_url(resource, query={'a': '1'}, anchor='top')
```

| Key | Description |
|-----|-------------|
| `query` | Query string data |
| `anchor` | URL fragment |
| `app_url` | Override prefix (pass `''` for relative) |
| `scheme` | Override URL scheme |
| `host` | Override hostname |
| `port` | Override port |
| `route_name` | Delegate URL generation to a named route |
| `route_kw` | Extra kwargs for `route_url` (with `route_name`) |

### `request.static_url(path, **kw)`

Generate a URL for a static asset.

```python
url = request.static_url('myapp:static/style.css')
# -> 'http://example.com/static/style.css'
```

### `request.static_path(path, **kw)`

Like `static_url` but returns a relative path.

### `request.current_route_url(**kw)`

Generate URL for the currently matched route, merging `request.matchdict` with provided kwargs.

### `request.current_route_path(**kw)`

Like `current_route_url` but returns a relative path.

---

## 8. Renderers

**Module:** `pyramid.renderers`

### Built-in Renderers

| Name | Content-Type | Description |
|------|-------------|-------------|
| `'json'` | `application/json` | JSON serialization |
| `'string'` | `text/plain` | String conversion |
| `'.pt'` | varies | Chameleon PageTemplate (requires `pyramid_chameleon`) |
| `'.jinja2'` | varies | Jinja2 template (requires `pyramid_jinja2`) |
| `'.mak'` / `'.mako'` | varies | Mako template (requires `pyramid_mako`) |

### Using Renderers

```python
# View returns dict; renderer converts to response
@view_config(route_name='api', renderer='json')
def api_view(request):
    return {'key': 'value'}

# Template renderer
@view_config(route_name='home', renderer='templates/home.jinja2')
def home(request):
    return {'title': 'Welcome'}
```

If a view returns a `Response` object, the renderer is **bypassed entirely**.

### Modifying Response Attributes with Renderers

```python
@view_config(renderer='templates/page.jinja2')
def myview(request):
    request.response.status = '404 Not Found'
    request.response.cache_control.max_age = 0
    return {'content': 'Not here'}
```

### JSON Renderer Customization

```python
from pyramid.renderers import JSON

json_renderer = JSON()

# Custom adapter for types you don't own:
import datetime
json_renderer.add_adapter(datetime.datetime, lambda obj, req: obj.isoformat())

config.add_renderer('json', json_renderer)
```

Objects can implement `__json__` for self-serialization:

```python
class MyModel:
    def __json__(self, request):
        return {'id': self.id, 'name': self.name}
```

### JSONP Renderer

```python
from pyramid.renderers import JSONP

config.add_renderer('jsonp', JSONP(param_name='callback'))

@view_config(renderer='jsonp')
def api(request):
    return {'data': 'value'}
# GET /api?callback=handleData -> handleData({"data": "value"})
```

### Programmatic Rendering

```python
from pyramid.renderers import render, render_to_response, get_renderer

# Render to string:
html = render('templates/email.jinja2', {'user': user}, request=request)

# Render to response:
response = render_to_response('templates/page.jinja2', {'title': 'Hi'}, request=request)

# Get renderer object:
renderer = get_renderer('templates/base.pt')
```

### Custom Renderer Factory

```python
class YAMLRendererFactory:
    def __init__(self, info):
        """info is a RendererHelper instance with .name, .package, .settings, .registry"""
        pass

    def __call__(self, value, system):
        """value = view return value; system = dict with request, context, view, renderer_name"""
        import yaml
        request = system.get('request')
        if request is not None:
            request.response.content_type = 'application/yaml'
        return yaml.dump(value)

config.add_renderer('.yaml', YAMLRendererFactory)
```

### Template System Values

Templates receive these system variables automatically:

| Variable | Description |
|----------|-------------|
| `request` | The current request |
| `context` | The traversal context |
| `view` | The view callable instance (for class views) |
| `renderer_name` | Name of the renderer |
| `renderer_info` | RendererHelper instance |
| `get_csrf_token()` | Function returning the current CSRF token |

### `config.add_renderer(name, factory)`

Register a renderer factory. Use `None` or `''` for the default renderer.

---

## 9. Security

**Module:** `pyramid.security`

### Security Policy (Pyramid 2.0+)

The recommended approach. Implement `ISecurityPolicy`:

```python
from pyramid.authentication import SessionAuthenticationHelper
from pyramid.authorization import ACLHelper, Allowed, Denied

class MySecurityPolicy:
    def __init__(self):
        self.authn_helper = SessionAuthenticationHelper()
        self.acl_helper = ACLHelper()

    def identity(self, request):
        """Return an opaque identity object or None."""
        userid = self.authn_helper.authenticated_userid(request)
        if userid is None:
            return None
        return request.dbsession.query(User).get(userid)

    def authenticated_userid(self, request):
        """Return the authenticated user ID string or None."""
        identity = self.identity(request)
        if identity is not None:
            return str(identity.id)
        return None

    def permits(self, request, context, permission):
        """Return Allowed or Denied."""
        principals = self._get_principals(request)
        return self.acl_helper.permits(context, principals, permission)

    def remember(self, request, userid, **kw):
        """Return headers to remember the user."""
        return self.authn_helper.remember(request, userid, **kw)

    def forget(self, request, **kw):
        """Return headers to forget the user."""
        return self.authn_helper.forget(request, **kw)

    def _get_principals(self, request):
        principals = ['system.Everyone']
        identity = self.identity(request)
        if identity is not None:
            principals.append('system.Authenticated')
            principals.append(str(identity.id))
            principals.extend(identity.groups)
        return principals

# Register:
config.set_security_policy(MySecurityPolicy())
```

### Protecting Views with Permissions

```python
# Require 'edit' permission:
@view_config(route_name='edit', permission='edit')
def edit_view(request): ...

# No permission required (bypass default permission):
from pyramid.security import NO_PERMISSION_REQUIRED

@view_config(route_name='login', permission=NO_PERMISSION_REQUIRED)
def login_view(request): ...

# Set a default permission for all views:
config.set_default_permission('view')
```

### Security Helpers

```python
from pyramid.security import remember, forget

def login_view(request):
    # ... validate credentials ...
    headers = remember(request, userid)
    return HTTPFound(location='/', headers=headers)

def logout_view(request):
    headers = forget(request)
    return HTTPFound(location='/', headers=headers)
```

### Request Security Methods

```python
request.identity                    # Opaque identity object or None
request.authenticated_userid       # User ID string or None
request.is_authenticated            # True/False
request.has_permission('edit')      # Returns Allowed or Denied
request.has_permission('edit', context)  # Against specific context
```

### Security Check Results

```python
from pyramid.security import Allowed, Denied

result = request.has_permission('edit')
if result:                          # Allowed is truthy, Denied is falsy
    print(result.msg)               # Explanation message
```

---

## 10. Authentication Helpers

**Module:** `pyramid.authentication`

### `SessionAuthenticationHelper`

Stores user identity in the session.

```python
from pyramid.authentication import SessionAuthenticationHelper

helper = SessionAuthenticationHelper(prefix='auth.')
helper.remember(request, userid)           # -> [] (stores in session)
helper.forget(request)                     # -> [] (removes from session)
helper.authenticated_userid(request)       # -> userid or None
```

### `AuthTktCookieHelper`

Signed cookie-based authentication tickets.

```python
from pyramid.authentication import AuthTktCookieHelper

helper = AuthTktCookieHelper(
    secret='my-secret-key',
    cookie_name='auth_tkt',
    secure=True,                # HTTPS only
    include_ip=False,           # Don't bind to IP
    timeout=86400,              # 24 hour ticket lifetime
    reissue_time=3600,          # Re-issue after 1 hour
    max_age=86400,              # Cookie max age
    http_only=True,
    hashalg='sha512',
    samesite='Lax',
    domain=None,                # Auto-detect domain
)

identity = helper.identify(request)   # -> {timestamp, userid, tokens, userdata} or None
headers = helper.remember(request, 'user123', max_age=86400, tokens=('admin',))
headers = helper.forget(request)
```

### `extract_http_basic_credentials(request)`

```python
from pyramid.authentication import extract_http_basic_credentials

creds = extract_http_basic_credentials(request)
if creds:
    username = creds.username
    password = creds.password
```

### Legacy Authentication Policies (Deprecated 2.0)

These implement `IAuthenticationPolicy` and are bridged via `LegacySecurityPolicy`:

| Policy | Storage | Constructor |
|--------|---------|------------|
| `AuthTktAuthenticationPolicy` | Signed cookie | `(secret, callback=None, cookie_name='auth_tkt', ...)` |
| `SessionAuthenticationPolicy` | Session | `(prefix='auth.', callback=None)` |
| `BasicAuthAuthenticationPolicy` | HTTP Basic | `(check, realm='Realm')` |
| `RemoteUserAuthenticationPolicy` | REMOTE_USER | `(environ_key='REMOTE_USER', callback=None)` |
| `RepozeWho1AuthenticationPolicy` | repoze.who | `(identifier_name='auth_tkt', callback=None)` |

---

## 11. Authorization & ACLs

**Module:** `pyramid.authorization`

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `Allow` | `'Allow'` | Grant access |
| `Deny` | `'Deny'` | Deny access |
| `Everyone` | `'system.Everyone'` | All users (authenticated or not) |
| `Authenticated` | `'system.Authenticated'` | Any authenticated user |
| `ALL_PERMISSIONS` | sentinel | Matches any permission |
| `DENY_ALL` | `(Deny, Everyone, ALL_PERMISSIONS)` | Deny everything |

### ACL (Access Control List)

ACLs are lists of ACE (Access Control Entry) tuples: `(action, principal, permission)`.

```python
from pyramid.authorization import Allow, Deny, Everyone, Authenticated, ALL_PERMISSIONS, DENY_ALL

class Blog:
    __acl__ = [
        (Allow, Everyone, 'view'),
        (Allow, Authenticated, 'comment'),
        (Allow, 'group:editors', ('add', 'edit')),
        (Allow, 'group:admins', ALL_PERMISSIONS),
        DENY_ALL,                                   # Stop ACL inheritance
    ]
```

Dynamic ACLs (callable):

```python
class Document:
    def __acl__(self):
        return [
            (Allow, self.owner_id, ('view', 'edit', 'delete')),
            (Allow, Everyone, 'view'),
        ]
```

### `ACLHelper`

Used in custom security policies for ACL-based authorization:

```python
from pyramid.authorization import ACLHelper

class MySecurityPolicy:
    def __init__(self):
        self.acl = ACLHelper()

    def permits(self, request, context, permission):
        principals = self._get_principals(request)
        return self.acl.permits(context, principals, permission)
```

`ACLHelper.permits(context, principals, permission)`:
- Walks the resource lineage (context -> parent -> ... -> root)
- Checks `__acl__` on each resource (supports callable `__acl__`)
- Returns `ACLAllowed` on first `Allow` match, `ACLDenied` on first `Deny` match
- Returns `ACLDenied` if no match found (default deny)

---

## 12. CSRF Protection

**Module:** `pyramid.csrf`

### Configuration

```python
# Enable CSRF checking globally:
config.set_default_csrf_options(
    require_csrf=True,
    token='csrf_token',                                  # POST body param name
    header='X-CSRF-Token',                               # Header name
    safe_methods=('GET', 'HEAD', 'OPTIONS', 'TRACE'),    # Exempt methods
    check_origin=True,                                   # Validate Origin/Referer
    allow_no_origin=False,                               # Reject missing Origin+Referer
    callback=None,                                       # callable(request) -> bool
)

# Per-view override:
@view_config(route_name='api', require_csrf=False)
def api_view(request): ...
```

### CSRF Token Functions

```python
from pyramid.csrf import get_csrf_token, new_csrf_token, check_csrf_token

token = get_csrf_token(request)         # Get or generate
token = new_csrf_token(request)         # Force new token

# Manual check:
check_csrf_token(request, token='csrf_token', header='X-CSRF-Token', raises=True)
```

### CSRF Storage Policies

```python
from pyramid.csrf import SessionCSRFStoragePolicy, CookieCSRFStoragePolicy

# Session-based (default):
config.set_csrf_storage_policy(SessionCSRFStoragePolicy(key='_csrft_'))

# Cookie-based (Double Submit Cookie pattern):
config.set_csrf_storage_policy(CookieCSRFStoragePolicy(
    cookie_name='csrf_token',
    secure=True,
    httponly=False,         # Must be False so JavaScript can read the cookie
    samesite='Lax',
))
```

### In Templates

```html
<!-- Jinja2 -->
<form method="POST">
    <input type="hidden" name="csrf_token" value="{{ get_csrf_token() }}">
    ...
</form>

<!-- AJAX -->
<script>
fetch('/api', {
    method: 'POST',
    headers: {'X-CSRF-Token': '{{ get_csrf_token() }}'},
    body: JSON.stringify(data)
});
</script>
```

---

## 13. Sessions

**Module:** `pyramid.session`

### `SignedCookieSessionFactory` (Recommended)

```python
from pyramid.session import SignedCookieSessionFactory

session_factory = SignedCookieSessionFactory(
    secret='my-secret-key-at-least-64-chars-for-sha512',
    cookie_name='session',
    max_age=None,               # Cookie max age (None = browser session)
    path='/',
    domain=None,
    secure=False,               # True for HTTPS
    httponly=True,
    samesite='Lax',
    timeout=1200,               # Inactivity timeout: 20 minutes (None = never)
    reissue_time=0,             # Auto-reissue interval (0 = every request)
    hashalg='sha512',
    salt='pyramid.session.',
    serializer=None,            # Default: JSONSerializer
)

config.set_session_factory(session_factory)
```

**Cookie payload limit: 4064 bytes.** The session data is stored in the cookie, not server-side.

### Session API

```python
def my_view(request):
    session = request.session     # dict-like interface

    session['key'] = 'value'      # Store data
    value = session['key']        # Retrieve data
    value = session.get('key')    # Safe retrieve

    # IMPORTANT: call changed() after mutating mutable values
    session['items'] = [1, 2, 3]
    session['items'].append(4)
    session.changed()             # Notify that session data changed

    session.invalidate()          # Clear all session data
```

### Flash Messages

```python
# Add messages:
request.session.flash('Item saved successfully')
request.session.flash('Error: invalid input', queue='errors')
request.session.flash('Warning: low disk', queue='warnings')

# Retrieve and remove messages:
messages = request.session.pop_flash()            # Default queue
errors = request.session.pop_flash('errors')

# Peek without removing:
messages = request.session.peek_flash()
```

### Session CSRF Token Methods

```python
token = request.session.get_csrf_token()
token = request.session.new_csrf_token()
```

---

## 14. Events

**Module:** `pyramid.events`

### Built-in Event Types

| Event | Interface | Fired When | Key Attributes |
|-------|-----------|-----------|----------------|
| `ApplicationCreated` | `IApplicationCreated` | `make_wsgi_app()` completes | `.app` |
| `NewRequest` | `INewRequest` | Start of request processing | `.request` |
| `BeforeTraversal` | `IBeforeTraversal` | After route matching, before traversal | `.request` |
| `ContextFound` | `IContextFound` | After traversal finds context | `.request` |
| `NewResponse` | `INewResponse` | When view returns a response | `.request`, `.response` |
| `BeforeRender` | `IBeforeRender` | Just before renderer invocation | dict-like; `.rendering_val` |

### Subscribing to Events

```python
# Decorator style (requires config.scan()):
from pyramid.events import subscriber, NewRequest, NewResponse

@subscriber(NewRequest)
def on_new_request(event):
    event.request.start_time = time.time()

@subscriber(NewResponse)
def on_new_response(event):
    elapsed = time.time() - event.request.start_time
    event.response.headers['X-Elapsed'] = str(elapsed)

# Imperative style:
config.add_subscriber(on_new_request, NewRequest)
```

### `BeforeRender` — Adding Template Globals

```python
@subscriber(BeforeRender)
def add_globals(event):
    event['project_name'] = 'My App'
    event['current_year'] = datetime.now().year
    # These are now available in all templates
```

### Custom Events

```python
class UserCreated:
    def __init__(self, user, request):
        self.user = user
        self.request = request

# Fire it:
request.registry.notify(UserCreated(new_user, request))

# Subscribe:
@subscriber(UserCreated)
def on_user_created(event):
    send_welcome_email(event.user)
```

### Subscriber Predicates

```python
config.add_subscriber_predicate('route_name', RouteNamePredicate)
config.add_subscriber(handler, NewRequest, route_name='api')
```

---

## 15. Traversal

**Module:** `pyramid.traversal`

### How Traversal Works

Traversal walks a resource tree using URL path segments, calling `__getitem__` on each resource:

```
URL: /foo/bar/baz
     Root.__getitem__('foo') -> Foo
         Foo.__getitem__('bar') -> Bar
             Bar.__getitem__('baz') -> KeyError!
     context = Bar, view_name = 'baz'
```

### Resource Tree Example

```python
class Root(dict):
    def __init__(self, request):
        self['users'] = UserContainer(parent=self)

class UserContainer(dict):
    def __init__(self, parent):
        self.__parent__ = parent
        self.__name__ = 'users'

    def __getitem__(self, key):
        user = db.get_user(key)
        if user is None:
            raise KeyError(key)
        user.__parent__ = self
        user.__name__ = key
        return user

config = Configurator(root_factory=Root)
```

### Location-Aware Resources

Resources must have `__parent__` and `__name__` attributes:

```python
class MyResource:
    __parent__ = None    # Parent resource (None for root)
    __name__ = None      # Name of this resource in the parent
```

### Traversal Helper Functions

```python
from pyramid.traversal import (
    find_root,                # Walk to root of tree
    find_resource,            # Find resource by path string or tuple
    find_interface,           # Find ancestor implementing interface/class
    resource_path,            # '/path/to/resource' string
    resource_path_tuple,      # ('', 'path', 'to', 'resource') tuple
    lineage,                  # Generator: resource, parent, grandparent, ...
    traverse,                 # Full traversal returning dict
    virtual_root,             # Virtual root for vhosting
)

root = find_root(some_resource)
user = find_resource(root, '/users/42')
container = find_interface(some_resource, IContainer)
path = resource_path(some_resource)  # -> '/users/42'
```

### `traverse(resource, path)`

```python
result = traverse(root, '/users/42/edit')
# result = {
#     'context': <User 42>,
#     'root': <Root>,
#     'view_name': 'edit',
#     'subpath': (),
#     'traversed': ('', 'users', '42'),
#     'virtual_root': <Root>,
#     'virtual_root_path': (),
# }
```

### Hybrid: URL Dispatch + Traversal

```python
config.add_route('user', '/users/{id}/*traverse', factory=UserFactory)

class UserFactory:
    def __init__(self, request):
        self.user = db.get_user(request.matchdict['id'])
    def __getitem__(self, key):
        # Further traversal within user context
        ...
```

### `@@` View Name Prefix

Use `@@viewname` in URLs to unambiguously specify a view name (vs. a resource child):

```
/users/@@list     -> view_name='list' on /users resource
/users/list       -> tries __getitem__('list') first
```

---

## 16. HTTP Exceptions

**Module:** `pyramid.httpexceptions`

HTTP exceptions are both `Response` and `Exception` subclasses. They can be **raised or returned**.

### Common Exceptions

```python
from pyramid.httpexceptions import (
    HTTPOk,                     # 200
    HTTPCreated,                # 201
    HTTPNoContent,              # 204
    HTTPMovedPermanently,       # 301
    HTTPFound,                  # 302 (common redirect)
    HTTPSeeOther,               # 303
    HTTPNotModified,            # 304
    HTTPTemporaryRedirect,      # 307
    HTTPPermanentRedirect,      # 308
    HTTPBadRequest,             # 400
    HTTPUnauthorized,           # 401
    HTTPForbidden,              # 403 (has .result attribute)
    HTTPNotFound,               # 404
    HTTPMethodNotAllowed,       # 405
    HTTPConflict,               # 409
    HTTPGone,                   # 410
    HTTPUnprocessableEntity,    # 422
    HTTPTooManyRequests,        # 429
    HTTPInternalServerError,    # 500
    HTTPServiceUnavailable,     # 503
)
```

### Usage

```python
# Raise as exception:
raise HTTPNotFound('Resource not found')

# Return as response (equivalent):
return HTTPFound(location=request.route_url('home'))

# Redirects require location:
raise HTTPFound(location='/new-location')
raise HTTPMovedPermanently(location='/permanent-new-location')

# With JSON body:
raise HTTPBadRequest(json_body={'error': 'Invalid input'})

# Constructor signature:
HTTPException(
    detail=None,            # Detailed error message
    headers=None,           # Extra headers list
    comment=None,           # Comment (not sent to client)
    body_template=None,     # Custom body template
    json_formatter=None,    # Custom JSON formatter
)
```

### `exception_response(status_code, **kw)`

Create an exception by status code:

```python
from pyramid.httpexceptions import exception_response
raise exception_response(404, detail='Not found')
```

### Framework Exceptions

```python
from pyramid.exceptions import (
    BadCSRFOrigin,              # CSRF origin check failed (subclass of HTTPBadRequest)
    BadCSRFToken,               # CSRF token check failed (subclass of HTTPBadRequest)
    PredicateMismatch,          # No view matched predicates (subclass of HTTPNotFound)
    ConfigurationError,         # Invalid configuration
    ConfigurationConflictError, # Conflicting config directives
)
```

---

## 17. Tweens

**Module:** `pyramid.tweens`, `pyramid.config`

Tweens are like WSGI middleware but operate at the Pyramid level (with request/response objects instead of raw WSGI environ).

### Writing a Tween

```python
def timing_tween_factory(handler, registry):
    """handler is the next tween or the Pyramid router."""
    def timing_tween(request):
        start = time.time()
        try:
            response = handler(request)
        finally:
            elapsed = time.time() - start
            log.info('Request took %.3f seconds', elapsed)
        return response
    return timing_tween
```

### Registering Tweens

```python
config.add_tween('myapp.tweens.timing_tween_factory')

# With ordering:
config.add_tween(
    'myapp.tweens.timing_tween_factory',
    under='pyramid.tweens.excview_tween_factory',  # Closer to app
    over='INGRESS',                                 # Farther from app
)
```

### Ordering Constants

| Constant | Position |
|----------|----------|
| `INGRESS` | Outermost (closest to WSGI) |
| `EXCVIEW` | `pyramid.tweens.excview_tween_factory` — handles exception views |
| `MAIN` | Innermost (the actual request handler) |

### Default Tween Chain

```
INGRESS -> excview_tween -> MAIN (handle_request)
```

### Built-in Tween

`pyramid.tweens.excview_tween_factory` — Catches exceptions raised during request handling and invokes registered exception views.

### Explicit Tween Ordering (in `.ini`)

```ini
pyramid.tweens =
    myapp.tweens.timing_tween_factory
    pyramid.tweens.excview_tween_factory
```

---

## 18. Static Assets

**Module:** `pyramid.static`

### Serving Static Files

```python
config.add_static_view(
    name='static',                  # URL prefix
    path='myapp:static',           # Asset spec or filesystem path
    cache_max_age=3600,            # Cache-Control max-age
    content_encodings=('br', 'gzip'),  # Pre-compressed alternatives
)
```

### `static_view` Class

```python
from pyramid.static import static_view

view = static_view(
    root_dir='myapp:static',
    cache_max_age=3600,
    use_subpath=False,          # True for traversal-based serving
    index='index.html',
    reload=False,               # Watch for changes
    content_encodings=(),       # e.g., ('br', 'gzip')
)
```

### Cache Busting

```python
from pyramid.static import ManifestCacheBuster, QueryStringConstantCacheBuster

# Manifest-based (uses JSON mapping file):
config.add_cache_buster('myapp:static', ManifestCacheBuster('myapp:static/manifest.json'))

# Constant query string:
config.add_cache_buster('myapp:static', QueryStringConstantCacheBuster('v1.2.3'))
```

Custom cache buster:

```python
from pyramid.static import QueryStringCacheBuster

class GitHashCacheBuster(QueryStringCacheBuster):
    def __init__(self):
        super().__init__(param='h')
    def tokenize(self, request, pathspec, kw):
        return get_git_hash()

config.add_cache_buster('myapp:static', GitHashCacheBuster())
```

### Asset Overrides

```python
# Override a single file:
config.override_asset('myapp:templates/base.pt', 'newapp:templates/custom_base.pt')

# Override an entire directory:
config.override_asset('myapp:templates/', 'newapp:templates/')
```

---

## 19. Internationalization (i18n)

**Module:** `pyramid.i18n`

### Translation Strings

```python
from pyramid.i18n import TranslationStringFactory

_ = TranslationStringFactory('myapp')  # 'myapp' is the gettext domain

# Simple:
ts = _('welcome')

# With default and mapping:
ts = _('greeting', default='Hello ${name}', mapping={'name': 'World'})
```

### Localizer

```python
# In views (via request.localizer):
def my_view(request):
    localizer = request.localizer
    translated = localizer.translate(_('welcome'))
    plural = localizer.pluralize('${n} item', '${n} items', n, mapping={'n': n})
```

### Configuration

```python
config.add_translation_dirs('myapp:locale')

# With override priority:
config.add_translation_dirs('myapp:locale', override=True)

# Custom locale negotiator:
def my_negotiator(request):
    return request.params.get('lang') or request.cookies.get('lang') or 'en'

config.set_locale_negotiator(my_negotiator)
```

### Default Locale Negotiator

Checks (in order):
1. `request._LOCALE_` attribute
2. `request.params['_LOCALE_']`
3. `request.cookies['_LOCALE_']`
4. Falls back to `pyramid.default_locale_name` setting (default: `'en'`)

---

## 20. Testing Utilities

**Module:** `pyramid.testing`

### Setup/Teardown

```python
import unittest
from pyramid import testing

class ViewTests(unittest.TestCase):
    def setUp(self):
        self.config = testing.setUp()

    def tearDown(self):
        testing.tearDown()
```

#### `testing.setUp(registry=None, request=None, hook_zca=True, autocommit=True, settings=None)`

Returns a `Configurator` in autocommit mode.

#### `testing.tearDown(unhook_zca=True)`

Cleans up thread-local state.

### Context Manager

```python
with testing.testConfig() as config:
    config.add_route('home', '/')
    # Test code here...
```

### `DummyRequest`

```python
request = testing.DummyRequest(
    params={'key': 'value'},
    post={'form_field': 'data'},        # Sets method='POST'
    cookies={'session': 'abc'},
    headers={'X-Custom': 'test'},
    path='/test',
    accept='application/json',
    matchdict={'id': '42'},             # Via **kw
    dbsession=mock_session,             # Via **kw — arbitrary attributes
)

request.method          # 'POST' (because post was provided)
request.application_url # 'http://example.com'
request.host            # 'example.com:80'
```

### `DummyResource`

```python
root = testing.DummyResource()
root['child'] = testing.DummyResource()
root['child'].__name__      # 'child'
root['child'].__parent__    # root

# Clone:
clone = root.clone(__name__='new_name', extra_attr='value')
```

### `DummySecurityPolicy`

```python
self.config.testing_securitypolicy(
    userid='user1',
    identity=user_object,
    permissive=True,                    # True = all permissions allowed
    remember_result=[('Set-Cookie', '...')],
    forget_result=[],
)
```

### `DummySession`

Implements `ISession` with flash messages and CSRF:

```python
session = testing.DummySession()
session['key'] = 'value'
session.flash('message')
session.pop_flash()        # -> ['message']
```

### Testing Security

```python
# Test view requires permission:
def test_forbidden(self):
    self.config.testing_securitypolicy(userid='user1', permissive=False)
    request = testing.DummyRequest()
    from pyramid.httpexceptions import HTTPForbidden
    self.assertRaises(HTTPForbidden, my_view, request)
```

### Testing with Renderers

```python
# Override a renderer to capture template variables:
renderer = self.config.testing_add_renderer('templates/home.jinja2')

my_view(request)

renderer.assert_(title='Expected Title')
```

### Functional Testing with WebTest

```python
from webtest import TestApp

class FunctionalTests(unittest.TestCase):
    def setUp(self):
        from myapp import main
        app = main({}, **{'setting': 'value'})
        self.testapp = TestApp(app)

    def test_home(self):
        res = self.testapp.get('/', status=200)
        assert 'Welcome' in res.text

    def test_post(self):
        res = self.testapp.post_json('/api', {'key': 'value'}, status=200)
        assert res.json['status'] == 'ok'
```

---

## 21. Scripting & CLI

**Module:** `pyramid.scripting`

### `prepare(request=None, registry=None)`

Set up a scripting environment outside of a web request:

```python
from pyramid.scripting import prepare
from pyramid.paster import get_appsettings, get_app

# Option 1: Using prepare()
app = get_app('development.ini')
with prepare(registry=app.registry) as env:
    request = env['request']
    root = env['root']
    # Use request.route_url(), dbsession, etc.
```

### `get_root(app, request=None)`

```python
from pyramid.scripting import get_root

root, closer = get_root(app)
try:
    # Work with root resource
    pass
finally:
    closer()
```

---

## 22. Settings & Environment

### Settings in `.ini` File

```ini
[app:main]
use = egg:myapp

# Pyramid settings:
pyramid.reload_templates = true
pyramid.debug_authorization = false
pyramid.debug_notfound = false
pyramid.debug_routematch = false
pyramid.default_locale_name = en
pyramid.prevent_http_cache = false

# Custom settings:
myapp.database_url = sqlite:///myapp.db
myapp.secret_key = changeme
```

### Accessing Settings

```python
# In main():
def main(global_config, **settings):
    db_url = settings['myapp.database_url']
    config = Configurator(settings=settings)
    ...

# In views:
def my_view(request):
    db_url = request.registry.settings['myapp.database_url']
```

### Pyramid Settings Reference

| Setting | Env Variable | Type | Default | Description |
|---------|-------------|------|---------|-------------|
| `pyramid.debug_all` | `PYRAMID_DEBUG_ALL` | bool | `false` | Enable all debug flags |
| `pyramid.debug_authorization` | `PYRAMID_DEBUG_AUTHORIZATION` | bool | `false` | Print auth decisions |
| `pyramid.debug_notfound` | `PYRAMID_DEBUG_NOTFOUND` | bool | `false` | Print 404 debug info |
| `pyramid.debug_routematch` | `PYRAMID_DEBUG_ROUTEMATCH` | bool | `false` | Print route matching |
| `pyramid.debug_templates` | `PYRAMID_DEBUG_TEMPLATES` | bool | `false` | Template debug info |
| `pyramid.reload_all` | `PYRAMID_RELOAD_ALL` | bool | `false` | Enable all reload flags |
| `pyramid.reload_templates` | `PYRAMID_RELOAD_TEMPLATES` | bool | `false` | Auto-reload templates |
| `pyramid.reload_assets` | `PYRAMID_RELOAD_ASSETS` | bool | `false` | Auto-reload assets |
| `pyramid.default_locale_name` | `PYRAMID_DEFAULT_LOCALE_NAME` | str | `'en'` | Default locale |
| `pyramid.prevent_http_cache` | `PYRAMID_PREVENT_HTTP_CACHE` | bool | `false` | Disable caching |
| `pyramid.prevent_cachebust` | `PYRAMID_PREVENT_CACHEBUST` | bool | `false` | Disable cache busting |
| `pyramid.csrf_trusted_origins` | `PYRAMID_CSRF_TRUSTED_ORIGINS` | list | `[]` | Trusted CSRF origins |
| `pyramid.includes` | — | list | `[]` | Auto-include packages |
| `pyramid.tweens` | — | list | — | Explicit tween ordering |

Environment variables override `.ini` settings. `debug_all` sets all `debug_*` flags. `reload_all` sets all `reload_*` flags.

### Settings Utility Functions

```python
from pyramid.settings import asbool, aslist

asbool('true')          # True  (truthy: t, true, y, yes, on, 1)
asbool('false')         # False (falsey: f, false, n, no, off, 0)
aslist('a b\nc d')      # ['a', 'b', 'c', 'd']
```

---

## 23. View Derivers

**Module:** `pyramid.viewderivers`

View derivers form a pipeline that wraps view callables. Each deriver can add behavior (permission checking, CSRF validation, rendering, caching, etc.).

### Default Pipeline (INGRESS to VIEW)

```
INGRESS
  -> secured_view         (permission checking)
  -> csrf_view            (CSRF token validation)
  -> owrapped_view        (view wrapping)
  -> http_cached_view     (HTTP caching headers)
  -> decorated_view       (user-provided decorators)
  -> rendered_view        (renderer invocation)
  -> mapped_view          (view mapper / signature adaptation)
VIEW
```

### Custom View Deriver

```python
def my_deriver(view, info):
    """
    view: the next callable in the chain
    info: ViewDeriverInfo with .registry, .package, .settings, .options, .exception_only, .predicates
    """
    if info.options.get('my_flag'):
        def wrapper(context, request):
            # Pre-processing
            response = view(context, request)
            # Post-processing
            return response
        return wrapper
    return view

my_deriver.options = ('my_flag',)  # Declare accepted view options

config.add_view_deriver(my_deriver, under='decorated_view', over='rendered_view')
```

---

## 24. Utility Functions & Classes

### `pyramid.decorator.reify`

Property decorator that computes once and caches on the instance:

```python
from pyramid.decorator import reify

class MyClass:
    @reify
    def expensive_computation(self):
        return compute_something()
```

### `pyramid.path.DottedNameResolver`

Resolve dotted Python names to objects:

```python
from pyramid.path import DottedNameResolver

resolver = DottedNameResolver()
cls = resolver.resolve('myapp.models.User')
cls = resolver.maybe_resolve('myapp.models.User')  # Returns non-strings as-is
```

### `pyramid.path.AssetResolver`

Resolve asset specifications:

```python
from pyramid.path import AssetResolver

resolver = AssetResolver('myapp')
descriptor = resolver.resolve('templates/home.pt')
path = descriptor.abspath()
exists = descriptor.exists()
```

### `pyramid.location`

```python
from pyramid.location import lineage, inside

# Walk the resource tree upward:
for resource in lineage(context):
    print(resource)

# Check containment:
if inside(child_resource, parent_resource):
    print('child is inside parent')
```

### `pyramid.encode`

```python
from pyramid.encode import urlencode, url_quote

qs = urlencode([('key', 'value'), ('a', 'b')])  # 'key=value&a=b'
quoted = url_quote('/path with spaces')
```

### `pyramid.asset`

```python
from pyramid.asset import resolve_asset_spec

pkg, path = resolve_asset_spec('myapp:templates/foo.pt')
# -> ('myapp', 'templates/foo.pt')
```

### Predicate Negation

```python
from pyramid.config import not_

config.add_view(my_view, request_method=not_('POST'))  # Match anything except POST
```

---

## 25. WSGI Helpers

**Module:** `pyramid.wsgi`

### `@wsgiapp`

Wrap a WSGI app as a Pyramid view (no PATH_INFO fixup):

```python
from pyramid.wsgi import wsgiapp

@wsgiapp
def my_wsgi_app(environ, start_response):
    start_response('200 OK', [('Content-Type', 'text/plain')])
    return [b'Hello from WSGI']

config.add_view(my_wsgi_app, route_name='legacy')
```

### `@wsgiapp2`

Like `wsgiapp` but fixes PATH_INFO from `request.subpath` and adjusts SCRIPT_NAME:

```python
from pyramid.wsgi import wsgiapp2

@wsgiapp2
def downstream_app(environ, start_response):
    # PATH_INFO is set from request subpath
    ...

config.add_view(downstream_app, route_name='downstream')
```

---

## 26. Extending Pyramid Applications

### `config.include()`

The primary extension mechanism:

```python
# In your package's __init__.py:
def includeme(config):
    """Called by config.include('mypackage')"""
    config.add_route('api', '/api')
    config.scan()

# Usage:
config.include('mypackage')
config.include('mypackage', route_prefix='v1')
```

### Overriding Views

```python
config.include('original_app')
# Later registrations override earlier ones for the same view conditions:
config.add_view('myapp.views.custom_home', route_name='home')
```

### Overriding Assets/Templates

```python
config.override_asset(
    'original_app:templates/base.pt',
    'myapp:templates/custom_base.pt'
)
```

### Custom Directives

```python
def setup_database(config, url):
    settings = config.get_settings()
    settings['db_url'] = url
    engine = create_engine(url)
    config.registry.dbmaker = sessionmaker(bind=engine)

config.add_directive('setup_database', setup_database)
config.setup_database('sqlite:///mydb.sqlite')
```

### Response Adapters

```python
config.add_response_adapter(string_response_adapter, str)

def string_response_adapter(s):
    response = Response(s)
    response.content_type = 'text/plain'
    return response
```

---

## 27. Deployment & PasteDeploy

### `.ini` File Structure

```ini
[app:main]
use = egg:myproject
pyramid.reload_templates = false
pyramid.debug_authorization = false
myapp.secret = production-secret

[server:main]
use = egg:waitress#main
listen = 0.0.0.0:6543

[loggers]
keys = root, myapp

[handlers]
keys = console

[formatters]
keys = generic

[logger_root]
level = WARN
handlers = console

[logger_myapp]
level = WARN
handlers =
qualname = myapp

[handler_console]
class = StreamHandler
args = (sys.stderr,)
level = NOTSET
formatter = generic

[formatter_generic]
format = %(asctime)s %(levelname)-5.5s [%(name)s:%(lineno)s] %(message)s
```

### Entry Point

```python
# In pyproject.toml:
[project.entry-points."paste.app_factory"]
main = "myapp:main"

# In myapp/__init__.py:
def main(global_config, **settings):
    config = Configurator(settings=settings)
    config.include('.routes')
    config.scan()
    return config.make_wsgi_app()
```

### Pipeline Configuration

```ini
[pipeline:main]
pipeline =
    egg:WebError#evalerror
    myapp

[app:myapp]
use = egg:myproject
```

### Running

```bash
# Development:
pserve development.ini --reload

# Production (with waitress):
pserve production.ini
```

### Logging in Application Code

```python
import logging
log = logging.getLogger(__name__)

def my_view(request):
    log.debug('Processing request for %s', request.path)
    log.info('User %s accessed resource', request.authenticated_userid)
    log.warning('Deprecated API called')
```

---

## 28. Common Patterns & Best Practices

### Application Factory Pattern

```python
def main(global_config, **settings):
    config = Configurator(settings=settings)

    # Include packages
    config.include('pyramid_jinja2')
    config.include('pyramid_tm')

    # Security
    config.set_security_policy(MySecurityPolicy())
    config.set_default_csrf_options(require_csrf=True)
    config.set_default_permission('view')

    # Sessions
    config.set_session_factory(SignedCookieSessionFactory(settings['session.secret']))

    # Routes
    config.include('.routes')

    # Static assets
    config.add_static_view('static', 'static', cache_max_age=3600)

    # Scan for view configs
    config.scan()

    return config.make_wsgi_app()
```

### Database Integration Pattern

```python
def db(request):
    session = request.registry.dbmaker()
    def cleanup(request):
        if request.exception is not None:
            session.rollback()
        else:
            session.commit()
        session.close()
    request.add_finished_callback(cleanup)
    return session

config.add_request_method(db, reify=True)

# In views:
def my_view(request):
    users = request.db.query(User).all()
```

### Modular Route Configuration

```python
# routes.py
def includeme(config):
    config.add_route('home', '/')
    config.add_route('login', '/login')
    config.add_route('logout', '/logout')

    config.include('.api', route_prefix='api/v1')

# api.py
def includeme(config):
    config.add_route('api_users', '/users')
    config.add_route('api_user', '/users/{id:\d+}')
```

### Request Method Dispatch with Classes

```python
@view_defaults(route_name='items', renderer='json')
class ItemViews:
    def __init__(self, request):
        self.request = request

    @view_config(request_method='GET')
    def list(self):
        return {'items': [...]}

    @view_config(request_method='POST')
    def create(self):
        data = self.request.json_body
        return {'created': data}

    @view_config(route_name='item', request_method='GET')
    def get(self):
        return {'item': ...}

    @view_config(route_name='item', request_method='PUT')
    def update(self):
        return {'updated': True}

    @view_config(route_name='item', request_method='DELETE')
    def delete(self):
        return {'deleted': True}
```

### Login/Logout Pattern

```python
from pyramid.httpexceptions import HTTPFound
from pyramid.security import remember, forget, NO_PERMISSION_REQUIRED

@view_config(route_name='login', renderer='templates/login.jinja2',
             permission=NO_PERMISSION_REQUIRED)
def login(request):
    if request.method == 'POST':
        username = request.POST['username']
        password = request.POST['password']
        user = authenticate(username, password)
        if user:
            headers = remember(request, user.id)
            return HTTPFound(location=request.route_url('home'), headers=headers)
        request.session.flash('Invalid credentials', queue='errors')
    return {}

@view_config(route_name='logout')
def logout(request):
    headers = forget(request)
    return HTTPFound(location=request.route_url('home'), headers=headers)
```

### Exception Handling Pattern

```python
@exception_view_config(Exception, renderer='json')
def error_view(exc, request):
    log.exception('Unhandled exception')
    request.response.status_int = 500
    return {'error': 'Internal server error'}

@exception_view_config(HTTPNotFound, renderer='templates/404.jinja2')
def not_found(exc, request):
    request.response.status_int = 404
    return {}

@exception_view_config(HTTPForbidden, renderer='templates/403.jinja2')
def forbidden(exc, request):
    request.response.status_int = 403
    return {}
```

### Subrequest Pattern

```python
from pyramid.request import Request

def composite_view(request):
    subreq = Request.blank('/api/data')
    subreq.registry = request.registry
    response = request.invoke_subrequest(subreq, use_tweens=True)
    data = response.json
    return {'api_data': data}
```

### Custom Predicates

```python
class ApiVersionPredicate:
    def __init__(self, val, config):
        self.val = val

    def text(self):
        return f'api_version = {self.val}'

    def phash(self):
        return self.text()

    def __call__(self, context, request):
        return request.headers.get('X-API-Version') == self.val

config.add_view_predicate('api_version', ApiVersionPredicate)

@view_config(route_name='api', api_version='2')
def api_v2(request): ...
```

### Transaction Management (pyramid_tm)

```python
config.include('pyramid_tm')
# Transactions are auto-committed on success, auto-rolled-back on exceptions

# Manual control:
import transaction

def my_view(request):
    # ... do work ...
    transaction.doom()     # Force rollback after view returns
    # ... or ...
    transaction.commit()   # Explicit commit mid-view (rare)
```

### Conflict Detection

Pyramid raises `ConfigurationConflictError` at startup if two directives conflict (e.g., two `add_view` calls with identical predicates). Later `config.include()` calls can override earlier ones because includes get a different "discriminator context." Within the same `include`, conflicts are errors.

### No Global State

Pyramid uses no module-level singletons. Multiple Pyramid applications can coexist in the same Python process, each with its own registry and configuration.

---

## Appendix: Module Index

| Module | Primary Contents |
|--------|-----------------|
| `pyramid.config` | `Configurator`, all `add_*`/`set_*` methods |
| `pyramid.request` | `Request`, `RequestLocalCache`, callback mixins |
| `pyramid.response` | `Response`, `FileResponse`, `response_adapter` |
| `pyramid.view` | `view_config`, `view_defaults`, `notfound_view_config`, `forbidden_view_config`, `exception_view_config` |
| `pyramid.url` | URL generation methods (mixed into Request) |
| `pyramid.router` | `Router` — the WSGI application |
| `pyramid.renderers` | `render`, `render_to_response`, `JSON`, `JSONP`, `get_renderer` |
| `pyramid.security` | `remember`, `forget`, `Allowed`, `Denied`, `NO_PERMISSION_REQUIRED`, security mixins |
| `pyramid.authentication` | `SessionAuthenticationHelper`, `AuthTktCookieHelper`, `extract_http_basic_credentials`, legacy policies |
| `pyramid.authorization` | `ACLHelper`, `Allow`, `Deny`, `Everyone`, `Authenticated`, `ALL_PERMISSIONS`, `DENY_ALL` |
| `pyramid.csrf` | `get_csrf_token`, `new_csrf_token`, `check_csrf_token`, CSRF storage policies |
| `pyramid.session` | `SignedCookieSessionFactory`, `BaseCookieSessionFactory` |
| `pyramid.events` | `subscriber`, `NewRequest`, `NewResponse`, `BeforeRender`, `ContextFound`, `ApplicationCreated` |
| `pyramid.traversal` | `find_root`, `find_resource`, `traverse`, `resource_path`, `lineage`, `ResourceTreeTraverser` |
| `pyramid.httpexceptions` | All HTTP exception classes (200–507), `exception_response` |
| `pyramid.exceptions` | `BadCSRFOrigin`, `BadCSRFToken`, `ConfigurationError`, `PredicateMismatch` |
| `pyramid.static` | `static_view`, cache buster classes |
| `pyramid.i18n` | `TranslationStringFactory`, `Localizer`, locale negotiation |
| `pyramid.testing` | `setUp`, `tearDown`, `DummyRequest`, `DummyResource`, `DummySecurityPolicy`, `DummySession` |
| `pyramid.scripting` | `prepare`, `get_root` |
| `pyramid.tweens` | `excview_tween_factory` |
| `pyramid.wsgi` | `wsgiapp`, `wsgiapp2` |
| `pyramid.decorator` | `reify` |
| `pyramid.predicates` | Built-in predicate classes |
| `pyramid.path` | `DottedNameResolver`, `AssetResolver`, `caller_package` |
| `pyramid.encode` | `urlencode`, `url_quote` |
| `pyramid.location` | `lineage`, `inside` |
| `pyramid.settings` | `asbool`, `aslist` |
| `pyramid.asset` | `resolve_asset_spec` |
| `pyramid.viewderivers` | Default view deriver pipeline |
| `pyramid.registry` | `Registry` (Zope component registry subclass) |
