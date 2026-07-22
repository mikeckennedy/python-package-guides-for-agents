# fastapi-chameleon Comprehensive Reference

> **Version:** 0.1.18
> **Author:** Michael Kennedy (michael@talkpython.fm)
> **License:** MIT
> **Repository:** https://github.com/mikeckennedy/fastapi-chameleon
> **Documentation:** https://mkennedy.codes/docs/fastapi-chameleon
> **Python:** 3.10 - 3.15
> **Dependencies:** `fastapi`, `chameleon`

## Overview

`fastapi-chameleon` is a lightweight integration library that brings the [Chameleon template language](https://chameleon.readthedocs.io/) to FastAPI. It provides a decorator-based API for rendering Chameleon templates (`.pt` and `.html` files) from view functions: return a `dict` from a decorated view and it becomes the template's variables, rendered into a `fastapi.Response`. Both synchronous and asynchronous views are fully supported, and the package ships inline type hints with a `py.typed` marker.

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/fastapi-chameleon](https://mkennedy.codes/docs/fastapi-chameleon). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/fastapi-chameleon
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/fastapi-chameleon/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/fastapi-chameleon/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/fastapi-chameleon/SKILL.md — also discoverable at `/.well-known/agent-skills/fastapi-chameleon/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/fastapi-chameleon/skills.html

Every documentation page also has a plain-Markdown twin — swap the `.html` extension for `.md` to get token-efficient source without the site chrome. For example https://mkennedy.codes/docs/fastapi-chameleon/reference/template.html → https://mkennedy.codes/docs/fastapi-chameleon/reference/template.md

## Installation

```bash
pip install fastapi-chameleon
```

## Quick Start

```python
# main.py
from pathlib import Path

import fastapi
import uvicorn

import fastapi_chameleon

app = fastapi.FastAPI()

# 1. Initialize the template engine at startup (before views are registered)
BASE_DIR = Path(__file__).resolve().parent
fastapi_chameleon.global_init(str(BASE_DIR / 'templates'), auto_reload=True)


# 2. Decorate view functions — route decorator on the outside
@app.get('/')
@fastapi_chameleon.template('index.pt')
def hello_world():
    return {'message': "Let's go Chameleon and FastAPI!"}


# 3. Run the app
if __name__ == '__main__':
    uvicorn.run(app, host='127.0.0.1', port=8000)
```

---

## Public API Reference

The library exports five public functions from the `fastapi_chameleon` package:

```python
__all__ = ['template', 'global_init', 'not_found', 'response', 'generic_error']
```

---

### `global_init(template_folder, auto_reload=False, cache_init=True)`

Initialize the Chameleon template engine. **Must be called before any template rendering.**

#### Signature

```python
def global_init(template_folder: str, auto_reload: bool = False, cache_init: bool = True) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_folder` | `str` | *(required)* | Absolute or relative path to the directory containing template files. |
| `auto_reload` | `bool` | `False` | When `True`, Chameleon reloads templates from disk when they change. Enable in development, disable in production. |
| `cache_init` | `bool` | `True` | When `True`, calling `global_init()` a second time is a no-op (the engine is already initialized). Set to `False` to force re-initialization (e.g., to switch template folders in tests). |

#### Raises

- `FastAPIChameleonException` if `template_folder` is falsy (empty string, `None`).
- `FastAPIChameleonException` if `template_folder` does not point to an existing directory.

#### Behavior Details

- Internally creates a `chameleon.PageTemplateLoader` bound to the template folder and stores it (plus the folder path) as module-global state — one engine per process.
- If `cache_init=True` and the engine is already initialized, the call returns immediately *before* validating the arguments.
- Unlike the sister project `chameleon-flask`, there is **no** `restricted_namespace` parameter.

#### Example

```python
from pathlib import Path
import fastapi_chameleon

dev_mode = True

BASE_DIR = Path(__file__).resolve().parent
template_folder = str(BASE_DIR / 'templates')

fastapi_chameleon.global_init(template_folder, auto_reload=dev_mode)
```

---

### `@template(template_file=None, mimetype='text/html')`

Decorator for FastAPI view functions. Renders the return value through a Chameleon template and wraps it in a `fastapi.Response`.

#### Signature

```python
def template(
    template_file: Optional[Union[Callable[..., R], str]] = None,
    mimetype: str = 'text/html',
)
```

The runtime signature above is accompanied by `ParamSpec`-based `@overload`s so the decorated view keeps its exact parameter signature for FastAPI's dependency injection and type checkers.

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` or `None` | `None` | Path to the template file, relative to the template folder (e.g., `'home/index.pt'`). Supports `.pt` and `.html` extensions. If omitted, the template path is auto-resolved from the module and function name (see below). In the bare `@template` form this argument receives the view function itself. |
| `mimetype` | `str` | `'text/html'` | MIME type for the response. Use `'application/xml'` for XML templates, etc. |

Note: the keyword is `mimetype` here, not `content_type`, and there is **no** `status_code` parameter on this decorator (use `response()` for a custom status code).

#### Return Value Contract

The decorated view function **must** return one of:

1. **`dict`** — The dictionary is unpacked as keyword arguments to the Chameleon template. The template is rendered and returned as a `fastapi.Response` with the specified `mimetype` and status 200.

2. **`fastapi.Response`** (any subclass, e.g. `RedirectResponse`, `JSONResponse`) — Passed through unchanged. The template is **not** rendered. This is the escape hatch for redirects, custom error responses, or any response that should bypass templating.

Returning any other type raises `FastAPIChameleonException` at request time.

#### Decorator Usage Forms

The decorator supports three equivalent usage patterns:

```python
# 1. Explicit template path (most common)
@fastapi_chameleon.template('home/index.pt')
def index():
    return {'data': 'value'}

# 2. Auto-resolve with parentheses
@fastapi_chameleon.template()
def index():
    return {'data': 'value'}

# 3. Auto-resolve without parentheses
@fastapi_chameleon.template
def index():
    return {'data': 'value'}
```

#### Auto-Resolution Logic (When `template_file` Is Omitted)

When no template file is specified, the decorator resolves the template path as:

```
{module_name}/{function_name}.html   (checked first)
{module_name}/{function_name}.pt     (fallback if the .html file does not exist)
```

Where:
- `module_name` is the last segment of the view function's `__module__` (e.g., for `views.home`, it uses `home`).
- `function_name` is the view function's `__name__`.

**Example:** A function named `index` in a module `views.home` resolves to `home/index.html`, falling back to `home/index.pt`.

This resolution (including the `.html`-exists check on disk) happens **once at decoration time**, i.e. at import — not per request. Two consequences:

- If `global_init()` has not run yet when the view is decorated, the template folder silently defaults to `'templates'` relative to the current working directory; a later `global_init()` does not re-resolve already-derived names. Call `global_init()` before importing/registering view modules, or always pass explicit template names.
- If neither derived file exists, the failure surfaces at request time as a `ValueError` from Chameleon's loader.

#### Sync and Async Support

The decorator detects async view functions once, at decoration time, using `inspect.iscoroutinefunction()` and picks the matching wrapper:

```python
# Sync view - works
@app.get('/')
@fastapi_chameleon.template('index.pt')
def index():
    return {'message': 'Hello'}

# Async view - also works, detected automatically
@app.get('/async')
@fastapi_chameleon.template('async.pt')
async def async_index():
    await asyncio.sleep(0.01)
    return {'message': 'Hello async'}
```

#### Error-Page Handling

Both wrappers catch the library's control-flow exceptions raised anywhere below the view:

- `FastAPIChameleonNotFoundException` (raised by `not_found()`) → renders the exception's `template_file` with an **empty** model at HTTP 404.
- `FastAPIChameleonGenericException` (raised by `generic_error()`) → renders the exception's `template_file` with its `template_data` (or `{}` if `None`) at the exception's `status_code`.

Error pages are always rendered as `text/html`, regardless of the `mimetype` passed to the decorator.

#### Full Examples

```python
# Basic HTML rendering
@app.get('/')
@fastapi_chameleon.template('home/index.pt')
def home():
    return {'title': 'Home', 'items': get_items()}

# Custom mimetype (XML sitemap)
@app.get('/sitemap.xml')
@fastapi_chameleon.template('seo/sitemap.pt', mimetype='application/xml')
def sitemap():
    return {'urls': get_site_urls()}

# Response pass-through (redirect)
@router.post('/account/login')
@fastapi_chameleon.template('account/login.pt')
async def login(request: Request):
    user = await try_login(request)
    if user:
        return fastapi.responses.RedirectResponse('/account', status_code=302)
    return {'error': 'Invalid login'}  # re-render the form with an error

# 404 handling
@router.get('/catalog/item/{item_id}')
@fastapi_chameleon.template('catalog/item.pt')
async def item(item_id: int):
    item = service.get_item_by_id(item_id)
    if not item:
        fastapi_chameleon.not_found()
    return item.dict()
```

---

### `response(template_file, mimetype='text/html', status_code=200, **template_data)`

Render a template and return a `fastapi.Response` directly (non-decorator form). Useful when you need full manual control — a non-200 status code or a non-HTML mimetype — without going through the decorator.

#### Signature

```python
def response(
    template_file: str,
    mimetype: str = 'text/html',
    status_code: int = 200,
    **template_data,
) -> fastapi.Response
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` | *(required)* | Path to the template file, relative to the template folder. |
| `mimetype` | `str` | `'text/html'` | MIME type for the response. |
| `status_code` | `int` | `200` | HTTP status code. |
| `**template_data` | — | — | Arbitrary keyword arguments passed as template variables. |

#### Returns

`fastapi.Response` with the rendered template as the body.

#### Raises

- `FastAPIChameleonException` if `global_init()` has not been called.

#### Example

```python
@router.get('/report')
def report():
    data = generate_report()
    return fastapi_chameleon.response(
        'reports/summary.pt',
        status_code=202,
        report=data,
        title='Monthly summary',
    )
```

---

### `not_found(four04template_file='errors/404.pt')`

Short-circuit the current view and render a friendly 404 page. Call this from within (or anywhere below) a view function decorated with `@template`. This is especially useful in FastAPI, which returns JSON for 404s by default.

#### Signature

```python
def not_found(four04template_file: str = 'errors/404.pt') -> NoReturn
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `four04template_file` | `str` | `'errors/404.pt'` | Path to the 404 template file, relative to the template folder. If empty or whitespace, falls back to `'errors/404.pt'`. |

#### Behavior

- **Always raises** `FastAPIChameleonNotFoundException` — it never returns normally (`-> NoReturn`), so execution stops at the call site.
- The `@template` decorator catches this exception and renders the specified 404 template with an **empty** model (`{}`), HTTP status 404, and `text/html` (custom data cannot be passed to the 404 template — use `generic_error()` if you need that).
- Because the decorator does the catching, `not_found()` can be called from deep in a service or data-access layer, as long as a decorated view is above it on the call stack. Calling it from an undecorated route (or from middleware/dependencies) leaves the exception unhandled and FastAPI returns a 500.

#### Example

```python
@router.get('/user/{username}')
@fastapi_chameleon.template('user/profile.pt')
def user_profile(username: str):
    user = find_user(username)
    if not user:
        fastapi_chameleon.not_found()  # Uses default errors/404.pt

    return {'user': user}

# With a custom 404 template
@router.get('/product/{product_id}')
@fastapi_chameleon.template('product/detail.pt')
def product_detail(product_id: int):
    product = find_product(product_id)
    if not product:
        fastapi_chameleon.not_found('errors/product_not_found.pt')

    return {'product': product}
```

---

### `generic_error(template_file, status_code, template_data=None)`

Short-circuit the current view and render an error page with a custom status code. Like `not_found()`, but for any error — and it can pass data into the error template.

#### Signature

```python
def generic_error(template_file: str, status_code: int, template_data: Optional[dict] = None) -> NoReturn
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` | *(required)* | Path to the error template, relative to the template folder. |
| `status_code` | `int` | *(required)* | The HTTP status code to return (e.g., `fastapi.status.HTTP_401_UNAUTHORIZED`). |
| `template_data` | `Optional[dict]` | `None` | Optional variables passed to the error template. `None` renders with an empty context. |

#### Behavior

- **Always raises** `FastAPIChameleonGenericException` — it never returns normally (`-> NoReturn`).
- The `@template` decorator catches the exception and renders `template_file` with `template_data` (or `{}`) at `status_code`, always as `text/html`.
- The same call-stack rule as `not_found()` applies: a decorated view must be above the call site.

#### Example

```python
@router.get('/catalog/item/{item_id}')
@fastapi_chameleon.template('catalog/item.pt')
async def item(item_id: int):
    item = service.get_item_by_id(item_id)
    if not item:
        fastapi_chameleon.generic_error('errors/unauthorized.pt',
                                        fastapi.status.HTTP_401_UNAUTHORIZED)

    return item.dict()

# With data for the error template
fastapi_chameleon.generic_error('errors/500.pt', 500,
                                template_data={'detail': 'Something went sideways.'})
```

---

## Exceptions

Defined in `fastapi_chameleon.exceptions`. They are **not** re-exported at the package top level — import them from the submodule.

```python
from fastapi_chameleon.exceptions import (
    FastAPIChameleonException,
    FastAPIChameleonNotFoundException,
    FastAPIChameleonGenericException,
)
```

### `FastAPIChameleonException`

```python
class FastAPIChameleonException(Exception):
    pass
```

Base exception for all library errors — the control-flow exceptions below subclass it, so catching this type catches them all. Raised directly when:

- `global_init()` is called with a falsy or non-directory `template_folder`.
- Rendering is attempted before `global_init()` has been called.
- A decorated view function returns a type other than `dict` or `fastapi.Response`.

### `FastAPIChameleonNotFoundException`

```python
class FastAPIChameleonNotFoundException(FastAPIChameleonException):
    def __init__(self, message: Optional[str] = None, four04template_file: str = 'errors/404.pt'):
        ...
```

Raised by `not_found()` and caught by the `@template` decorator to render a 404 page.

#### Attributes

| Attribute | Type | Description |
|---|---|---|
| `template_file` | `str` | Path to the 404 template (the constructor accepts it as `four04template_file`). |
| `message` | `Optional[str]` | Error message. The constructor defaults it to `None`; `not_found()` passes `'The URL resulted in a 404 response.'` when it raises. |

### `FastAPIChameleonGenericException`

```python
class FastAPIChameleonGenericException(FastAPIChameleonException):
    def __init__(
        self,
        template_file: str,
        status_code: int,
        message: Optional[str] = None,
        template_data: Optional[dict] = None,
    ):
        ...
```

Raised by `generic_error()` and caught by the `@template` decorator to render an arbitrary error page.

#### Attributes

| Attribute | Type | Description |
|---|---|---|
| `template_file` | `str` | Path to the error template. |
| `status_code` | `int` | The HTTP status code to return. |
| `message` | `Optional[str]` | Error message. `generic_error()` passes `'The URL resulted in an error.'` when it raises. |
| `template_data` | `Optional[dict]` | Variables passed to the template when rendering (`None` → empty context). |

---

## Internal API

These are not part of the public `__all__` but may be useful for testing or advanced usage. Access them via the `fastapi_chameleon.engine` module.

### `fastapi_chameleon.engine.render(template_file, **template_data)`

```python
def render(template_file: str, **template_data) -> str
```

Render a template to a string using the configured engine — the lowest-level rendering function, used internally by `response()` and the decorator. Useful when you need raw HTML/XML output without a `fastapi.Response` wrapper (e.g., rendering an email body). Raises `FastAPIChameleonException` if `global_init()` has not been called.

```python
html = fastapi_chameleon.engine.render('email/welcome.pt', name='Michael')
send_email(recipient, subject='Welcome', body=html)
```

### `fastapi_chameleon.engine.clear()`

```python
def clear() -> None
```

Resets the engine: clears the cached `PageTemplateLoader` and template path. Primarily a test-isolation hook — call it between tests (or before a fresh `global_init()`) so engine state does not leak across them.

---

## Template Language Reference

fastapi-chameleon uses the [Chameleon](https://chameleon.readthedocs.io/) template engine, which implements TAL (Template Attribute Language), TALES, and METAL. This is **not** Jinja — there is no `{% ... %}` or `{{ ... }}` syntax.

### File Extensions

| Extension | Description |
|---|---|
| `.pt` | Standard Chameleon page template (preferred) |
| `.html` | Also supported; checked first during auto-resolution |
| Anything else (e.g. `.xml`) | Works with an explicit `template_file`; pair with a matching `mimetype` |

### Variable Interpolation

Use `${}` for expression interpolation (any Python expression):

```html
<h1>Hello, ${username}!</h1>
<p>You have ${len(items)} items.</p>
```

### TAL Attributes

#### `tal:repeat` - Looping

```html
<ul>
    <li tal:repeat="item items">${item.name} - ${item.price}</li>
</ul>
```

#### `tal:condition` - Conditionals

```html
<div tal:condition="user">Welcome, ${user.name}!</div>
<div tal:condition="not user">Please log in.</div>
```

#### `tal:content` - Replace element content

```html
<span tal:content="message">placeholder</span>
```

#### `tal:attributes` - Set HTML attributes

```html
<a tal:attributes="href item.url; class item.css_class">${item.name}</a>
```

#### `tal:replace` - Replace entire element

```html
<span tal:replace="formatted_date">2024-01-01</span>
```

### METAL Macros (Shared Layouts)

METAL is Chameleon's answer to template inheritance:

```html
<!-- templates/shared/layout.pt -->
<html metal:define-macro="layout">
  <head><title>${title}</title></head>
  <body>
    <main metal:define-slot="content">default content</main>
  </body>
</html>
```

```html
<!-- templates/home/index.pt -->
<div metal:use-macro="load: ../shared/layout.pt">
  <div metal:fill-slot="content">
    <h1>${title}</h1>
  </div>
</div>
```

---

## Project Structure Conventions

### Recommended Layout

```
my_app/
├── main.py               # FastAPI app creation + fastapi_chameleon.global_init()
├── views/
│   ├── home.py           # View functions
│   └── catalog.py
├── templates/
│   ├── home/
│   │   └── index.pt      # Matches home.index view function
│   ├── catalog/
│   │   ├── list.pt
│   │   └── item.pt
│   ├── errors/
│   │   └── 404.pt        # Default 404 template
│   └── shared/
│       └── layout.pt     # Shared layout (via METAL macros)
└── static/
    └── site.css
```

### Template Auto-Resolution Convention

When using `@fastapi_chameleon.template()` without an explicit path, templates are resolved as:

```
templates/{module_name}/{function_name}.html  →  checked first
templates/{module_name}/{function_name}.pt    →  fallback
```

For a function `index()` in module `views.home`, the decorator looks for:
1. `templates/home/index.html`
2. `templates/home/index.pt`

---

## Decorator Ordering

The `@template` decorator must be applied **after** the FastAPI route decorator (i.e., closer to the function):

```python
# CORRECT
@app.get('/path')
@fastapi_chameleon.template('page.pt')
def view():
    return {}

# WRONG - will not work as expected
@fastapi_chameleon.template('page.pt')
@app.get('/path')
def view():
    return {}
```

---

## Sync/Async Compatibility

Both sync and async view functions work with the same decorator — async is detected automatically at decoration time, so no separate import or flag is needed. The decorator is signature-transparent (ParamSpec overloads + `functools.wraps`), so FastAPI's dependency injection (`Request`, path/query params, `Depends(...)`) keeps working on decorated views.

```python
@router.post('/')
@fastapi_chameleon.template('home/index.pt')
async def home_post(request: Request):
    form = await request.form()
    vm = PersonViewModel(**form)
    return vm.dict()
```

---

## Common Patterns

### Dev mode vs. production

`auto_reload` defaults to `False`: Chameleon caches compiled templates for production performance. Set `auto_reload=True` during development to pick up template edits without restarting.

```python
fastapi_chameleon.global_init(template_folder, auto_reload=dev_mode)
```

### Testing decorated views without an HTTP layer

Decorated views remain plain callables. Call them directly (or via `asyncio.run()` for async views) and inspect the returned `fastapi.Response`:

```python
# conftest.py
from pathlib import Path

import pytest
import fastapi_chameleon as fc

@pytest.fixture
def setup_global_template(pytestconfig):
    fc.global_init(str(Path(pytestconfig.rootdir, 'tests', 'templates')))
    yield
    fc.engine.clear()  # don't leak engine state between tests
```

```python
# test_views.py
def test_index_renders(setup_global_template):
    resp = index_view()
    assert resp.status_code == 200
    assert 'Hello' in resp.body.decode('utf-8')
```

---

## Complete Working Example

```python
# example_app.py
import asyncio
from pathlib import Path

import fastapi
import uvicorn

import fastapi_chameleon

app = fastapi.FastAPI()


@app.get('/')
@fastapi_chameleon.template('index.pt')
def hello_world():
    return {'message': "Let's go Chameleon and FastAPI!"}


@app.get('/async')
@fastapi_chameleon.template('async.pt')
async def async_world():
    await asyncio.sleep(0.01)
    return {'message': "Let's go async Chameleon and FastAPI!"}


def main():
    dev_mode = True
    BASE_DIR = Path(__file__).resolve().parent
    template_folder = (BASE_DIR / 'templates').as_posix()
    fastapi_chameleon.global_init(template_folder, auto_reload=dev_mode)
    uvicorn.run(app, host='127.0.0.1', port=8000)


if __name__ == '__main__':
    main()
```

Note: this example calls `global_init()` inside `main()` (after the views are decorated with *explicit* template names, which is safe) — run it with `python example_app.py`, not via the `uvicorn` CLI, or `global_init()` never executes.

### Example Templates

**`templates/index.pt`:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <link rel="stylesheet" href="/static/site.css">
</head>
<body>
<h1>Hello world</h1>
<p>Your message is <strong>${message}</strong>
    <a href="/async">See the async example</a> too.</p>
</body>
</html>
```

**`templates/errors/404.pt`:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Page Not Found</title>
</head>
<body>
<h1>This is a pretty 404 page.</h1>
</body>
</html>
```

---

## Error Handling Patterns

### Missing Initialization

```python
# Raises FastAPIChameleonException at request time:
# "You must call global_init() before rendering templates."
@fastapi_chameleon.template('index.pt')
def view():
    return {}

view()  # Error - global_init() not called
```

### Invalid Return Type

```python
@fastapi_chameleon.template('index.pt')
def view():
    return "just a string"  # FastAPIChameleonException - expected dict or fastapi.Response

view()
```

### Missing Template File

```python
@fastapi_chameleon.template('nonexistent.pt')
def view():
    return {}

view()  # ValueError from Chameleon's loader - template not found
```

### Late `global_init()` with inferred template names

If a view module using the bare `@template` form is imported **before** `global_init()` runs, the template folder silently defaults to `'templates'` (relative to the CWD) at decoration time, and a later `global_init()` will not fix the already-derived names. Either initialize first or use explicit template paths.

---

## API Summary Table

| Function | Purpose | Returns |
|---|---|---|
| `global_init(folder, ...)` | Initialize template engine | `None` |
| `@template(file, mimetype)` | Decorator: render view result through template | `fastapi.Response` |
| `response(file, ...)` | Render template to Response directly (custom status/mimetype) | `fastapi.Response` |
| `not_found(file)` | Raise 404 inside a decorated view | `NoReturn` (always raises) |
| `generic_error(file, status, data)` | Raise any error status inside a decorated view | `NoReturn` (always raises) |
| `engine.render(file, ...)` | Render template to string | `str` |
| `engine.clear()` | Reset engine state (for testing) | `None` |
