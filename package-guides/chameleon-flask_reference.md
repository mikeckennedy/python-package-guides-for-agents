# chameleon-flask Comprehensive Reference

> **Version:** 0.6.1
> **Author:** Michael Kennedy (michael@talkpython.fm)
> **License:** MIT
> **Repository:** https://github.com/mikeckennedy/chameleon-flask
> **Documentation:** https://mkennedy.codes/docs/chameleon-flask
> **Python:** 3.10 - 3.15
> **Dependencies:** `flask[async]`, `Chameleon`

## Overview

`chameleon-flask` is a lightweight integration library that brings the [Chameleon template language](https://chameleon.readthedocs.io/) to Flask and Quart applications. It provides a decorator-based API for rendering Chameleon templates (`.pt` and `.html` files) from view functions, with full support for both synchronous and asynchronous views.

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/chameleon-flask](https://mkennedy.codes/docs/chameleon-flask). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/chameleon-flask
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/chameleon-flask/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/chameleon-flask/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/chameleon-flask/skill.md — also discoverable at `/.well-known/agent-skills/chameleon-flask/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/chameleon-flask/skills.html

Every documentation page also has a plain-Markdown twin — swap the `.html` extension for `.md` to get token-efficient source without the site chrome. For example https://mkennedy.codes/docs/chameleon-flask/reference/template.html → https://mkennedy.codes/docs/chameleon-flask/reference/template.md

## Installation

```bash
pip install chameleon_flask
```

## Quick Start

```python
import os
from pathlib import Path

import flask
import chameleon_flask

app = flask.Flask(__name__)

# 1. Initialize the template engine at startup
BASE_DIR = Path(__file__).resolve().parent
template_folder = str(BASE_DIR / 'templates')
chameleon_flask.global_init(template_folder, auto_reload=True)

# 2. Decorate view functions
@app.get('/')
@chameleon_flask.template('home/index.pt')
def index():
    return {'title': 'Welcome', 'message': 'Hello from Chameleon!'}

# 3. Run the app
app.run()
```

---

## Public API Reference

The library exports four public functions from the `chameleon_flask` package:

```python
__all__ = ['template', 'global_init', 'not_found', 'response']
```

---

### `global_init(template_folder, auto_reload=False, cache_init=True, restricted_namespace=True)`

Initialize the Chameleon template engine. **Must be called before any template rendering.**

#### Signature

```python
def global_init(
    template_folder: str,
    auto_reload: bool = False,
    cache_init: bool = True,
    restricted_namespace: bool = True,
) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_folder` | `str` | *(required)* | Absolute or relative path to the directory containing template files. |
| `auto_reload` | `bool` | `False` | When `True`, Chameleon reloads templates from disk on each render if they have changed. Enable in development, disable in production. |
| `cache_init` | `bool` | `True` | When `True`, calling `global_init()` a second time is a no-op (the engine is already initialized). Set to `False` to force re-initialization. |
| `restricted_namespace` | `bool` | `True` | When `True`, only standard TAL/METAL/i18n namespaces are permitted in templates. When `False`, allows attribute-based JS framework shorthand syntax (e.g., Alpine.js `@click`, `:class`, `x-data`). *Added in v0.6.0.* |

#### Raises

- `FlaskChameleonException` if `template_folder` is falsy (empty string, `None`).
- `FlaskChameleonException` if `template_folder` does not point to an existing directory.

#### Behavior Details

- Internally creates a `chameleon.PageTemplateLoader` bound to the template folder.
- If `cache_init=True` and the engine is already initialized, the call returns immediately.
- The `restricted_namespace` parameter is passed directly to Chameleon's `PageTemplateLoader`.

#### Example

```python
from pathlib import Path
import chameleon_flask

BASE_DIR = Path(__file__).resolve().parent
template_folder = str(BASE_DIR / 'templates')

# Development: auto-reload on, restricted_namespace off for Alpine.js
chameleon_flask.global_init(
    template_folder,
    auto_reload=True,
    restricted_namespace=False,
)

# Production: all defaults
chameleon_flask.global_init(template_folder)
```

---

### `@template(template_file=None, content_type='text/html', status_code=200)`

Decorator for Flask/Quart view functions. Renders the return value through a Chameleon template and wraps it in a `flask.Response`.

#### Signature

```python
def template(
    template_file: Optional[Union[Callable, str]] = None,
    content_type: str = 'text/html',
    status_code: int = 200,
)
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` or `None` | `None` | Path to the template file, relative to the template folder (e.g., `'home/index.pt'`). Supports `.pt` and `.html` extensions. If omitted, the template path is auto-generated from the module and function name (see below). |
| `content_type` | `str` | `'text/html'` | MIME type for the response. Use `'application/xml'` for XML templates, etc. |
| `status_code` | `int` | `200` | HTTP status code for the response. Useful for e.g., `201` on a POST/create action. |

#### Return Value Contract

The decorated view function **must** return one of:

1. **`dict`** - The dictionary is unpacked as keyword arguments to the Chameleon template. The template is rendered and returned as a `flask.Response` with the specified `content_type` and `status_code`.

2. **`flask.Response` or `quart.Response`** - Passed through unchanged. The template is **not** rendered. This is the escape hatch for redirects, custom error responses, or any response that should bypass templating.

Returning any other type raises `FlaskChameleonException`.

#### Decorator Usage Forms

The decorator supports three equivalent usage patterns:

```python
# 1. Explicit template path (most common)
@chameleon_flask.template('home/index.pt')
def index():
    return {'data': 'value'}

# 2. Auto-resolve with parentheses
@chameleon_flask.template()
def index():
    return {'data': 'value'}

# 3. Auto-resolve without parentheses
@chameleon_flask.template
def index():
    return {'data': 'value'}
```

#### Auto-Resolution Logic (When `template_file` Is Omitted)

When no template file is specified, the decorator resolves the template path as:

```
{module_name}/{function_name}.html   (checked first)
{module_name}/{function_name}.pt     (fallback)
```

Where:
- `module_name` is the last segment of the view function's `__module__` (e.g., for `views.home`, it uses `home`).
- `function_name` is `f.__name__`.

**Example:** A function named `details` in a module `app.views` would resolve to `views/details.html`, falling back to `views/details.pt`.

If neither file exists, a `ValueError` is raised at call time.

#### Sync and Async Support

The decorator automatically detects async view functions using `inspect.iscoroutinefunction()` and wraps them appropriately:

```python
# Sync view - works
@app.get('/')
@chameleon_flask.template('index.pt')
def index():
    return {'message': 'Hello'}

# Async view - also works, detected automatically
@app.get('/async')
@chameleon_flask.template('async.pt')
async def async_index():
    await asyncio.sleep(0.01)
    return {'message': 'Hello async'}
```

#### 404 Handling

If the view function raises `FlaskChameleonNotFoundException` (via `chameleon_flask.not_found()`), the decorator catches it and renders the specified 404 template with a 404 status code instead of the normal template.

#### Full Examples

```python
# Basic HTML rendering
@app.get('/')
@chameleon_flask.template('home/index.pt')
def home():
    return {'title': 'Home', 'items': get_items()}

# Custom content type and status code
@app.get('/feed')
@chameleon_flask.template('feed.xml', content_type='application/xml', status_code=201)
def xml_feed():
    return {'items': ['flask', 'pyramid', 'fastapi']}

# Response pass-through (redirect)
@app.get('/old-page')
@chameleon_flask.template('home/index.pt')
def old_page():
    return flask.redirect('/new-page')

# 404 handling
@app.get('/item/<int:item_id>')
@chameleon_flask.template('catalog/item.pt')
def item(item_id):
    item = get_item(item_id)
    if not item:
        chameleon_flask.not_found()
    return {'item': item}
```

---

### `response(template_file, content_type='text/html', status_code=200, **template_data)`

Render a template and return a `flask.Response` directly (non-decorator form). Useful when you need to render a template outside of the decorator pattern.

#### Signature

```python
def response(
    template_file: str,
    content_type: str = 'text/html',
    status_code: int = 200,
    **template_data,
) -> flask.Response
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` | *(required)* | Path to the template file, relative to the template folder. |
| `content_type` | `str` | `'text/html'` | MIME type for the response. |
| `status_code` | `int` | `200` | HTTP status code. |
| `**template_data` | `dict` | — | Arbitrary keyword arguments passed as template variables. |

#### Returns

`flask.Response` with the rendered template as the body.

#### Example

```python
@app.get('/report')
def report():
    data = generate_report()
    return chameleon_flask.response(
        'reports/summary.pt',
        content_type='text/html',
        status_code=200,
        report=data,
        generated_at=datetime.now(),
    )
```

---

### `not_found(four04template_file='errors/404.pt')`

Trigger a 404 response with an optional custom error template. Call this from within a view function decorated with `@template`.

#### Signature

```python
def not_found(four04template_file: str = 'errors/404.pt') -> NoReturn
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `four04template_file` | `str` | `'errors/404.pt'` | Path to the 404 template file, relative to the template folder. If empty or whitespace, defaults to `'errors/404.pt'`. |

#### Behavior

- Raises `FlaskChameleonNotFoundException` internally.
- The `@template` decorator catches this exception and renders the specified 404 template with an empty dict (`{}`) and HTTP status 404.
- **Must be called inside a view function that is decorated with `@template`**, otherwise the exception will propagate unhandled.

#### Example

```python
@app.get('/user/<username>')
@chameleon_flask.template('user/profile.pt')
def user_profile(username):
    user = find_user(username)
    if not user:
        chameleon_flask.not_found()  # Uses default errors/404.pt

    return {'user': user}

# With a custom 404 template
@app.get('/product/<int:product_id>')
@chameleon_flask.template('product/detail.pt')
def product_detail(product_id):
    product = find_product(product_id)
    if not product:
        chameleon_flask.not_found('errors/product_not_found.pt')

    return {'product': product}
```

---

### `render(template_file, **template_data)` *(low-level)*

Render a template to a string. This is the lowest-level rendering function, used internally by `response()` and `@template`. Useful when you need raw HTML/XML output without wrapping it in a `flask.Response`.

#### Signature

```python
def render(template_file: str, **template_data: Any) -> str
```

#### Parameters

| Parameter | Type | Description |
|---|---|---|
| `template_file` | `str` | Path to the template file, relative to the template folder. |
| `**template_data` | `dict` | Template variables. |

#### Returns

Rendered template content as a `str`.

#### Raises

- `FlaskChameleonException` if `global_init()` has not been called.

#### Example

```python
html = chameleon_flask.engine.render('email/welcome.pt', name='Michael')
send_email(recipient, subject='Welcome', body=html)
```

> **Note:** `render` is accessible via `chameleon_flask.engine.render()` but is not part of the top-level `__all__` exports.

---

## Exceptions

Defined in `chameleon_flask.exceptions`.

### `FlaskChameleonException`

```python
class FlaskChameleonException(Exception):
    pass
```

Base exception for all library errors. Raised when:

- `global_init()` has not been called before rendering.
- `template_folder` is invalid (empty or not a directory).
- A decorated view function returns a type other than `dict` or `flask.Response`.

### `FlaskChameleonNotFoundException`

```python
class FlaskChameleonNotFoundException(FlaskChameleonException):
    def __init__(
        self,
        message: Optional[str] = None,
        four04template_file: str = 'errors/404.pt',
    ):
        ...
```

Raised by `not_found()` and caught by the `@template` decorator to render a 404 page.

#### Attributes

| Attribute | Type | Description |
|---|---|---|
| `template_file` | `str` | Path to the 404 template. |
| `message` | `Optional[str]` | Error message. The constructor defaults it to `None`; `not_found()` passes `'The URL resulted in a 404 response.'` when it raises. |

---

## Internal API

These are not part of the public `__all__` but may be useful for testing or advanced usage.

### `chameleon_flask.engine.clear()`

```python
def clear() -> None
```

Resets the template engine by clearing the `PageTemplateLoader` and `template_path`. Used in tests to isolate state between test cases.

### `chameleon_flask.engine.response_classes`

```python
response_classes = {
    "<class 'flask.wrappers.Response'>",
    "<class 'quart.wrappers.response.Response'>",
    "<class 'flask.Response'>",
    "<class 'quart.response.Response'>",
    "<class 'quart.Response'>",
}
```

Legacy fallback set of type-name strings. Response pass-through is detected **primarily** via `isinstance(value, werkzeug.sansio.response.Response)` — the shared base class of `flask.Response`, `quart.Response`, and bare werkzeug responses — so the library never imports Quart. This string set only catches Response implementations that predate that shared werkzeug base.

---

## Template Language Reference

chameleon-flask uses the [Chameleon](https://chameleon.readthedocs.io/) template engine, which implements TAL (Template Attribute Language), TALES, and METAL.

### File Extensions

| Extension | Description |
|---|---|
| `.pt` | Standard Chameleon page template (preferred) |
| `.html` | Also supported; checked first during auto-resolution |
| `.xml` | XML templates, use with `content_type='application/xml'` |

### Variable Interpolation

Use `${}` for expression interpolation:

```html
<h1>Hello, ${username}!</h1>
<p>You have ${len(items)} items.</p>
<p>Literal: ${'some string'}</p>
```

### TAL Attributes

#### `tal:repeat` - Looping

```html
<ul>
    <li tal:repeat="item items">${item.name} - ${item.price}</li>
</ul>

<!-- Repeat with multiple variables -->
<table>
    <tr tal:repeat="row rows">
        <td tal:repeat="col columns">${row} ${col}</td>
    </tr>
</table>

<!-- Tuple literal -->
<tr tal:repeat="fruit 'apple', 'banana', 'pineapple'">
    <td>${fruit.capitalize()}</td>
</tr>
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

### XML Templates

```xml
<?xml version="1.0" encoding="utf-8" ?>
<root-node>
    <item tal:repeat="i items">${i}</item>
</root-node>
```

Use with:

```python
@chameleon_flask.template('feed.xml', content_type='application/xml')
def xml_feed():
    return {'items': ['a', 'b', 'c']}
```

---

## Project Structure Conventions

### Recommended Layout

```
my_app/
├── app.py                # Flask app creation + chameleon_flask.global_init()
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

When using `@chameleon_flask.template()` without an explicit path, templates are resolved as:

```
templates/{module_name}/{function_name}.html  →  checked first
templates/{module_name}/{function_name}.pt    →  fallback
```

For a function `index()` in module `views.home`, the decorator looks for:
1. `templates/home/index.html`
2. `templates/home/index.pt`

---

## Decorator Ordering

The `@template` decorator must be applied **after** the Flask route decorator (i.e., closer to the function):

```python
# CORRECT
@app.get('/path')
@chameleon_flask.template('page.pt')
def view():
    return {}

# WRONG - will not work as expected
@chameleon_flask.template('page.pt')
@app.get('/path')
def view():
    return {}
```

---

## Flask/Quart Compatibility

The library works with both Flask and Quart. Response pass-through detection supports response classes from both frameworks. Async view functions are automatically detected and handled correctly with Quart or Flask's async support (`flask[async]`).

```python
# Flask
import flask
app = flask.Flask(__name__)

# Quart
import quart
app = quart.Quart(__name__)

# Same decorator API for both
@app.get('/')
@chameleon_flask.template('index.pt')
async def index():
    return {'message': 'Works with Flask[async] and Quart'}
```

---

## Working with Alpine.js and Other JS Frameworks (v0.6.0+)

By default, Chameleon restricts template namespaces to TAL/METAL/i18n. This conflicts with attribute-based JS frameworks like Alpine.js that use `@click`, `:class`, `x-data`, etc.

To allow these, set `restricted_namespace=False` during initialization:

```python
chameleon_flask.global_init(
    template_folder,
    auto_reload=True,
    restricted_namespace=False,  # Allow Alpine.js syntax
)
```

Then in templates:

```html
<div x-data="{ open: false }">
    <button @click="open = !open">Toggle</button>
    <div :class="{ 'hidden': !open }">Content</div>
</div>
```

---

## Complete Working Example

```python
# example_app.py
import asyncio
from pathlib import Path

import flask
import chameleon_flask

app = flask.Flask(__name__)


@app.get('/')
@chameleon_flask.template('index.pt')
def hello_world():
    return {'message': "Let's go Chameleon!"}


@app.get('/async')
@chameleon_flask.template('async.pt')
async def async_world():
    await asyncio.sleep(0.01)
    return {'message': "Let's go async Chameleon!"}


@app.get('/xml')
@chameleon_flask.template('sample.xml', content_type='application/xml', status_code=201)
def xml_response():
    return {
        'items': ['pyramid', 'flask', 'fastapi'],
    }


def main():
    BASE_DIR = Path(__file__).resolve().parent
    template_folder = (BASE_DIR / 'templates').as_posix()
    chameleon_flask.global_init(template_folder, auto_reload=True)
    app.run(debug=True, host='127.0.0.1', port=5555)


if __name__ == '__main__':
    main()
```

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
    <a href="/async">See the async example</a>
    or even <a href="/xml">the xml example</a>!</p>
</body>
</html>
```

**`templates/async.pt`:**
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
    <link rel="stylesheet" href="/static/site.css">
</head>
<body>
<h1>Hello async world</h1>
<p>Your async message is <strong>${message}</strong></p>
</body>
</html>
```

**`templates/sample.xml`:**
```xml
<?xml version="1.0" encoding="utf-8" ?>
<root-node>
    <item tal:repeat="i items">${i}</item>
</root-node>
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
# Raises FlaskChameleonException:
# "You must call global_init() before rendering templates."
@chameleon_flask.template('index.pt')
def view():
    return {}

view()  # Error - global_init() not called
```

### Invalid Return Type

```python
@chameleon_flask.template('index.pt')
def view():
    return "just a string"  # FlaskChameleonException - expected dict or Response

view()
```

### Missing Template File

```python
@chameleon_flask.template('nonexistent.pt')
def view():
    return {}

view()  # ValueError - template not found
```

---

## API Summary Table

| Function | Purpose | Returns |
|---|---|---|
| `global_init(folder, ...)` | Initialize template engine | `None` |
| `@template(file, ...)` | Decorator: render view result through template | `flask.Response` |
| `response(file, ...)` | Render template to Response directly | `flask.Response` |
| `not_found(file)` | Raise 404 inside a decorated view | `NoReturn` (always raises) |
| `engine.render(file, ...)` | Render template to string | `str` |
| `engine.clear()` | Reset engine state (for testing) | `None` |
