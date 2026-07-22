# chameleon-robyn Comprehensive Reference

> **Version:** 0.1.2
> **Author:** Michael Kennedy (michael@talkpython.fm)
> **License:** MIT
> **Repository:** https://github.com/mikeckennedy/chameleon-robyn
> **Documentation:** https://mkennedy.codes/docs/chameleon-robyn
> **Python:** 3.10 - 3.15
> **Dependencies:** `robyn>=0.60`, `Chameleon`

## Overview

`chameleon-robyn` is a lightweight integration library that brings the [Chameleon template language](https://chameleon.readthedocs.io/) (`.pt` page templates with TAL/TALES/METAL) to the [Robyn](https://robyn.tech/) web framework. It provides a decorator-based API for rendering Chameleon templates from Robyn route handlers, with full support for both synchronous and asynchronous handlers, plus a standalone `ChameleonTemplate` class matching Robyn's built-in template-engine pattern. The project is alpha — the API is not yet promised stable.

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/chameleon-robyn](https://mkennedy.codes/docs/chameleon-robyn). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/chameleon-robyn
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/chameleon-robyn/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/chameleon-robyn/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/chameleon-robyn/skill.md — also at `/.well-known/agent-skills/chameleon-robyn/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/chameleon-robyn/skills.html

Every documentation page also has a plain-Markdown twin — swap the `.html` extension for `.md` to get token-efficient source without the site chrome. For example https://mkennedy.codes/docs/chameleon-robyn/reference/template.html → https://mkennedy.codes/docs/chameleon-robyn/reference/template.md

## Installation

```bash
pip install chameleon-robyn
```

This pulls in `robyn` and `Chameleon` as dependencies. The library never imports or depends on Jinja2.

## Quick Start

```python
from pathlib import Path

from robyn import Robyn

import chameleon_robyn

app = Robyn(__file__)


@app.get('/')
@chameleon_robyn.template('home/index.pt')
async def index(request):
    return {'name': 'World', 'items': ['Robyn', 'Chameleon', 'Python']}


def main():
    # 1. Point Chameleon at your templates folder (once, at startup)
    template_folder = (Path(__file__).resolve().parent / 'templates').as_posix()
    chameleon_robyn.global_init(template_folder, auto_reload=True)  # auto_reload for dev only

    # 2. Run the app
    app.start(host='127.0.0.1', port=8000)


if __name__ == '__main__':
    main()
```

`templates/home/index.pt`:

```html
<!DOCTYPE html>
<html>
<body>
    <h1>Hello, ${name}!</h1>
    <ul>
        <li tal:repeat="item items">${item}</li>
    </ul>
</body>
</html>
```

The handler returns a `dict`, the `@template` decorator renders it through the Chameleon template and wraps the result in a Robyn `Response`.

---

## Public API Reference

The package exports nine public names:

```python
__all__ = [
    'template',
    'global_init',
    'clear',
    'not_found',
    'render',
    'response',
    'ChameleonTemplate',
    'ChameleonRobynException',
    'ChameleonRobynNotFoundException',
]
```

`chameleon_robyn.__version__` is also available (read from package metadata; `'0.0.0'` if the distribution is not installed).

---

### `global_init(template_folder, auto_reload=False, cache_init=True, restricted_namespace=True)`

Initialize the module-level Chameleon template engine. **Must be called before any rendering happens** — but thanks to lazy template-name resolution it may be called *after* routes are decorated (the common pattern is decorating at import time and calling `global_init()` from `main()`).

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
| `auto_reload` | `bool` | `False` | When `True`, Chameleon reloads templates from disk when they change. Enable in development, disable in production. |
| `cache_init` | `bool` | `True` | When `True`, calling `global_init()` again while already initialized is a silent no-op. Set to `False` to force re-initialization. |
| `restricted_namespace` | `bool` | `True` | When `True`, only TAL/METAL/i18n namespaces are allowed in templates. Set to `False` to allow attribute-based JS framework shorthand (Alpine.js `@click`, `:class`, htmx attributes, etc.). |

#### Returns

`None`.

#### Raises

- `ChameleonRobynException` if `template_folder` is falsy (empty string, `None`).
- `ChameleonRobynException` if `template_folder` is not an existing directory.

#### Behavior Details

- Internally creates a `chameleon.PageTemplateLoader` bound to the folder and stores it in a module-private global, plus the module-level `template_path`.
- `auto_reload` and `restricted_namespace` are passed straight through to `PageTemplateLoader`.

#### Example

```python
from pathlib import Path
import chameleon_robyn

template_folder = (Path(__file__).resolve().parent / 'templates').as_posix()

# Development: auto-reload on, unrestricted namespace for Alpine.js/htmx
chameleon_robyn.global_init(template_folder, auto_reload=True, restricted_namespace=False)

# Production: all defaults
chameleon_robyn.global_init(template_folder)
```

---

### `clear()`

Reset the template engine to its uninitialized state. After calling it, `global_init()` must be called again before rendering. Mostly useful in tests to isolate template configuration between cases.

#### Signature

```python
def clear() -> None
```

#### Example

```python
import chameleon_robyn

def setup_function():
    chameleon_robyn.global_init('tests/templates', cache_init=False)

def teardown_function():
    chameleon_robyn.clear()
```

---

### `@template(template_file=None, content_type='text/html', status_code=200)`

Decorator for Robyn route handlers. Renders the handler's returned `dict` through a Chameleon template and wraps it in a Robyn `Response`. Works with sync and async handlers.

#### Signature

```python
def template(
    template_file: Optional[Union[Callable, str]] = None,
    content_type: str = 'text/html',
    status_code: int = 200,
) -> Callable
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` or `None` | `None` | Template file path relative to the template folder (e.g. `'home/index.pt'`). If omitted, the path is auto-derived from module and function name (see below). The `Callable` in the type union exists only to support the bare `@template` form. |
| `content_type` | `str` | `'text/html'` | Content-Type for rendered responses; `'; charset=utf-8'` is appended automatically. Use `'application/xml'` for XML templates. |
| `status_code` | `int` | `200` | HTTP status code for rendered responses (e.g. `201` for a create action). |

#### Return Value Contract

The decorated handler **must** return one of:

1. **`dict`** — the template model. Rendered through the template into a Robyn `Response` with the given `content_type` and `status_code`. The reserved key `__response_callback__` is treated specially (see below).
2. **`robyn.Response`** — passed through completely untouched (no rendering). This is the escape hatch for redirects and custom error responses.

Any other return type raises `ChameleonRobynException` at request time.

#### Decorator Usage Forms

```python
# 1. Explicit template path (most common)
@chameleon_robyn.template('home/index.pt')
def index(request):
    return {'data': 'value'}

# 2. Auto-resolve with parentheses
@chameleon_robyn.template()
def index(request):
    return {'data': 'value'}

# 3. Auto-resolve without parentheses (bare decorator)
@chameleon_robyn.template
def index(request):
    return {'data': 'value'}
```

#### Auto-Resolution Logic (When `template_file` Is Omitted)

The template path is derived as:

```
{module_name}/{function_name}.html   (used if that file exists on disk)
{module_name}/{function_name}.pt     (fallback)
```

Where `module_name` is the last dotted segment of the handler's `__module__` (for `views.home` it uses `home`) and `function_name` is `f.__name__`.

Resolution happens **lazily, at first request** — not at decoration time — because the template folder is usually unknown when routes are decorated at import time. The `.html`-exists check uses the `global_init()` folder, falling back to the literal directory `'templates'` if `global_init()` hasn't run yet. The derived name is cached only once `global_init()` has set the template folder, so init-later apps resolve correctly.

#### Sync and Async Support

Async handlers are detected automatically via `inspect.iscoroutinefunction()` and awaited; sync handlers are called directly. Same decorator, no flag needed.

```python
@app.get('/dashboard')
@chameleon_robyn.template('dashboard.pt')
async def dashboard(request):
    stats = await fetch_stats_from_db()
    return {'stats': stats}
```

#### `__response_callback__` — Post-Render Response Hook

If the returned dict contains the reserved key `__response_callback__`, its value is popped **before** rendering (from a copy — the handler's dict is never mutated) and invoked with the freshly built `Response` after rendering. The classic use is setting a session cookie after login:

```python
from robyn import Response

@app.post('/account/login')
@chameleon_robyn.template('account/welcome.pt')
async def login(request):
    user = await authenticate(request)
    if not user:
        return chameleon_robyn.response('account/login_failed.pt', status_code=401)

    token = create_session_token(user)

    def set_session_cookie(resp: Response):
        resp.headers.append('Set-Cookie', f'session={token}; HttpOnly; Path=/')

    return {
        'user': user,
        '__response_callback__': set_session_cookie,
    }
```

Details:

- The key never reaches the template.
- Mutate the `Response` in place; the callback's return value is ignored.
- The value must be callable — anything else raises `ChameleonRobynException`.

#### 404 Handling

If the handler raises `ChameleonRobynNotFoundException` (normally via `chameleon_robyn.not_found()`), the decorator catches it and renders the exception's 404 template with status 404 and content type `text/html`, passing the exception's message to the template as the `message` variable. The handler's own `content_type` and `status_code` do not apply on this path.

#### Full Examples

```python
# Basic HTML rendering
@app.get('/episodes')
@chameleon_robyn.template('episodes/list.pt')
def episode_list(request):
    return {'episodes': get_all_episodes()}

# Custom content type and status code
@app.get('/xml/episodes')
@chameleon_robyn.template('episodes/feed.xml', content_type='application/xml')
def episodes_xml(request):
    return {'episodes': get_recent_episodes()}

# Response pass-through (redirect)
from robyn import Headers, Response

@app.get('/old-page')
@chameleon_robyn.template('home/index.pt')
def old_page(request):
    return Response(status_code=302, description='', headers=Headers({'Location': '/new-page'}))

# 404 handling with Robyn route params
@app.get('/episodes/:episode_id')
@chameleon_robyn.template('episodes/detail.pt')
async def episode_detail(request, episode_id: int):
    episode = await get_episode(episode_id)
    if not episode:
        chameleon_robyn.not_found()
    return {'episode': episode}
```

---

### `response(template_file, content_type='text/html', status_code=200, **template_data)`

Render a Chameleon template and wrap it in a fully-formed Robyn `Response` — the non-decorator form. Useful in error handlers, middleware, or branches of a decorated handler that need a different template/status.

#### Signature

```python
def response(
    template_file: str,
    content_type: str = 'text/html',
    status_code: int = 200,
    **template_data: Any,
) -> Response
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_file` | `str` | *(required)* | Template file path relative to the template folder. |
| `content_type` | `str` | `'text/html'` | Content-Type header value; `'; charset=utf-8'` is appended. |
| `status_code` | `int` | `200` | HTTP status code. |
| `**template_data` | `Any` | — | Values passed to the template as its model. |

#### Returns

A `robyn.Response` with the rendered template as its body (`description`) and a `Content-Type: <content_type>; charset=utf-8` header.

#### Raises

- `ChameleonRobynException` (from the underlying `render()`) if `global_init()` has not been called.

#### Example

```python
resp = chameleon_robyn.response('errors/500.pt', status_code=500, error=str(e))
```

---

### `render(template_file, **template_data)`

Render a Chameleon template to a string — no `Response` wrapping. The lowest-level rendering function, used internally by `response()` and `@template`. Useful outside route handlers: emails, middleware, error handlers.

#### Signature

```python
def render(template_file: str, **template_data: Any) -> str
```

#### Parameters

| Parameter | Type | Description |
|---|---|---|
| `template_file` | `str` | Template file path relative to the template folder (e.g. `'emails/welcome.pt'`). |
| `**template_data` | `Any` | Values passed to the template as its model. |

#### Returns

The rendered template as a `str` (rendered with `encoding='utf-8'`).

#### Raises

- `ChameleonRobynException` if `global_init()` has not been called.

#### Example

```python
html = chameleon_robyn.render('emails/welcome.pt', username='Michael')
send_email(recipient, subject='Welcome', body=html)
```

> **Note:** Unlike chameleon-flask, `render` here is a first-class public export (in `__all__`).

---

### `not_found(four04template_file='errors/404.pt')`

Render a friendly 404 page from within a `@template`-decorated handler. Always raises; never returns.

#### Signature

```python
def not_found(four04template_file: str = 'errors/404.pt') -> NoReturn
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `four04template_file` | `str` | `'errors/404.pt'` | The 404 template to render, relative to the template folder. If empty or whitespace-only, the default is used. |

#### Behavior

- Always raises `ChameleonRobynNotFoundException` carrying the 404 template path and the message `'The URL resulted in a 404 response.'`.
- The `@template` decorator catches the exception and renders the carried template with HTTP status 404 and content type `text/html`, passing the message to the template as the `message` variable (render it with `${message}` if desired).
- **Only works inside handlers decorated with `@template`** — elsewhere the exception propagates unhandled.

#### Example

```python
@app.get('/guests/:slug')
@chameleon_robyn.template('guests/detail.pt')
def guest_detail(request, slug: str):
    guest = GUESTS.get(slug)
    if not guest:
        chameleon_robyn.not_found()  # renders errors/404.pt with status 404
    return {'guest': guest}

# With a custom 404 template
chameleon_robyn.not_found(four04template_file='errors/custom_404.pt')
```

---

### `ChameleonTemplate` class

A standalone Chameleon template engine implementing Robyn's `TemplateInterface` pattern — the same `render_template()` shape as Robyn's built-in `JinjaTemplate`. It owns its own `PageTemplateLoader` and does **not** require (or interact with) `global_init()`.

It subclasses a local, `runtime_checkable` `Protocol` that structurally matches Robyn's ABC. This is deliberate: importing Robyn's real `TemplateInterface` would pull in a Jinja2 dependency, and this library keeps Jinja2 out of the dependency tree.

#### Constructor

```python
class ChameleonTemplate(TemplateInterface):
    def __init__(
        self,
        directory: str,
        auto_reload: bool = False,
        encoding: str = 'utf-8',
        restricted_namespace: bool = True,
    )
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `directory` | `str` | *(required)* | Path to the template directory. |
| `auto_reload` | `bool` | `False` | Reload templates on change (use `True` in development). |
| `encoding` | `str` | `'utf-8'` | Output encoding passed to Chameleon's render. |
| `restricted_namespace` | `bool` | `True` | Same meaning as in `global_init()`. |

Instance attributes: `loader` (the `chameleon.PageTemplateLoader`) and `encoding`.

#### `render_template(template_name, **kwargs)`

```python
def render_template(self, template_name: str, **kwargs: Any) -> Response
```

Renders the template and returns a Robyn `Response` with **status 200** (always) and `Content-Type: text/html; charset=utf-8`. `template_name` is relative to the constructor's `directory`; `**kwargs` become the template model.

#### Example

```python
from chameleon_robyn import ChameleonTemplate

chameleon = ChameleonTemplate('templates/', auto_reload=True)

@app.get('/page')
def page(request):
    return chameleon.render_template('page.pt', title='Hello')
```

---

## Exceptions

Defined in `chameleon_robyn.exceptions`; both are re-exported from the top-level package.

### `ChameleonRobynException`

```python
class ChameleonRobynException(Exception):
    pass
```

Base exception for all library errors. Raised when:

- `global_init()` has not been called before rendering.
- `template_folder` is invalid (empty or not a directory).
- A decorated handler returns a type other than `dict` or `robyn.Response`.
- `__response_callback__` is present in the model but not callable.

### `ChameleonRobynNotFoundException`

```python
class ChameleonRobynNotFoundException(ChameleonRobynException):
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
| `template_file` | `str` | Path to the 404 template, relative to the template folder. |
| `message` | `Optional[str]` | Human-readable 404 description. The constructor defaults it to `None`; `not_found()` passes `'The URL resulted in a 404 response.'` when it raises. The decorator hands it to the 404 template as the `message` variable. |

---

## Internal API

Not part of `__all__`, but visible in `chameleon_robyn.engine`:

### `chameleon_robyn.engine.TemplateInterface`

```python
@runtime_checkable
class TemplateInterface(Protocol):
    def render_template(self, *args, **kwargs) -> Response: ...
```

A structural (`Protocol`) stand-in for Robyn's `TemplateInterface` ABC, so the library never imports Robyn's Jinja2-backed templating module. `ChameleonTemplate` subclasses it; any object with a matching `render_template` passes an `isinstance` check against it.

### `chameleon_robyn.engine.template_path`

Module-level `Optional[str]` holding the folder passed to `global_init()` (`None` before init / after `clear()`). Read by the auto-naming logic; treat it as read-only.

---

## Template Language Reference

chameleon-robyn uses the [Chameleon](https://chameleon.readthedocs.io/) template engine, which implements TAL (Template Attribute Language), TALES, and METAL. **This is not Jinja** — there is no `{% ... %}` or `{{ ... }}`.

### File Extensions

| Extension | Description |
|---|---|
| `.pt` | Standard Chameleon page template (preferred) |
| `.html` | Also supported; checked first during auto-resolution |
| `.xml` | XML templates, use with `content_type='application/xml'` |

### Variable Interpolation

Use `${}` for expression interpolation — any Python expression is allowed, and output is HTML-escaped by default:

```html
<h1>Hello, ${username}!</h1>
<p>You have ${len(items)} items.</p>
```

### TAL Attributes

```html
<!-- Loop -->
<li tal:repeat="item items">${item.name} - ${item.price}</li>

<!-- The repeat variable exposes index/number/even/odd/first/last -->
<tr tal:repeat="row data" class="${'odd' if repeat.row.odd else 'even'}">
    <td>${repeat.row.number}. ${row.title}</td>
</tr>

<!-- Conditionals (element omitted entirely when falsy) -->
<div tal:condition="show_banner">Special offer!</div>
<div tal:condition="not items">Nothing here yet.</div>

<!-- Replace element content / whole element -->
<span tal:content="message">placeholder</span>
<span tal:replace="formatted_date">2024-01-01</span>

<!-- Set attributes (semicolon-separated) -->
<a tal:attributes="href item.url; class item.css_class">${item.name}</a>

<!-- Emit already-safe HTML without escaping -->
<div tal:replace="structure rich_html_content" />
```

### METAL Macros (Layout Inheritance)

METAL is Chameleon's equivalent of Jinja's `{% extends %}`/`{% block %}`:

```html
<!-- templates/shared/layout.pt -->
<html>
<body>
    <main metal:define-slot="content">Default content</main>
</body>
</html>

<!-- templates/home/index.pt -->
<metal:block use-macro="load: shared/layout.pt">
<div metal:fill-slot="content">
    <h1>My page content here</h1>
</div>
</metal:block>
```

### XML Templates

```xml
<?xml version="1.0" encoding="utf-8" ?>
<episodes>
    <episode tal:repeat="ep episodes">${ep['title']}</episode>
</episodes>
```

Use with `@chameleon_robyn.template('episodes/feed.xml', content_type='application/xml')`.

---

## Project Structure Conventions

### Recommended Layout

```
my_app/
├── app.py                # Robyn app + routes; global_init() called from main()
├── templates/
│   ├── home/
│   │   └── index.pt      # matches a bare-@template index() in module home
│   ├── episodes/
│   │   ├── list.pt
│   │   └── detail.pt
│   ├── errors/
│   │   └── 404.pt        # default target of not_found()
│   └── shared/
│       └── layout.pt     # METAL macros
└── static/
```

### Template Auto-Resolution Convention

For a function `index()` in module `views.home`, the bare decorator looks for:

1. `templates/home/index.html`
2. `templates/home/index.pt`

Resolution happens at first request, so `global_init()` may run after route decoration — decorate routes at import time, initialize in `main()`.

### Decorator Ordering

The `@template` decorator must sit **below** the Robyn route decorator (closest to the function):

```python
# CORRECT
@app.get('/path')
@chameleon_robyn.template('page.pt')
def view(request):
    return {}

# WRONG
@chameleon_robyn.template('page.pt')
@app.get('/path')
def view(request):
    return {}
```

---

## Robyn Compatibility Notes

- Sync and async handlers both work; async is detected automatically with `inspect.iscoroutinefunction()`.
- Handlers receive whatever Robyn passes them (typically `request`, plus typed route params like `episode_id: int` for `/:episode_id` routes); the decorator forwards `*args, **kwargs` untouched.
- Responses built by the library always use Robyn's `Headers` class (never a plain dict) with `Content-Type: <type>; charset=utf-8`.
- A handler-returned `robyn.Response` bypasses templating entirely — that's how redirects and custom errors work.
- Requires `robyn>=0.60`.

## Working with Alpine.js, htmx, and Other JS Frameworks

By default, Chameleon restricts template namespaces to TAL/METAL/i18n, which rejects attribute shorthand like `@click`, `:class`, and `x-data`. Allow them at init time:

```python
chameleon_robyn.global_init(template_folder, auto_reload=True, restricted_namespace=False)
```

```html
<div x-data="{ open: false }">
    <button @click="open = !open">Toggle</button>
    <div :class="{ 'hidden': !open }">Content</div>
</div>
```

## Working with chameleon-partials

[chameleon-partials](https://github.com/mikeckennedy/chameleon-partials) adds `render_partial()` for reusable sub-templates and works alongside chameleon-robyn with no extra configuration — initialize both at startup:

```python
import chameleon_partials
import chameleon_robyn

chameleon_robyn.global_init(template_folder, auto_reload=dev_mode)
chameleon_partials.register_extensions(template_folder, auto_reload=dev_mode)
```

Pass `render_partial` through the model, then use `tal:replace="structure ..."` in templates (the `structure` keyword prevents escaping):

```python
@app.get('/')
@chameleon_robyn.template('home/index.pt')
def index(request):
    return {'render_partial': chameleon_partials.render_partial, 'episodes': get_episodes()}
```

```xml
<div tal:repeat="ep episodes">
    <div tal:replace="structure render_partial('shared/episode_card.pt', ep=ep)" />
</div>
```

---

## Complete Working Example

```python
# app.py
import asyncio
from pathlib import Path

from robyn import Headers, Response, Robyn

import chameleon_robyn

app = Robyn(__file__)

EPISODES = [
    {'id': 1, 'title': 'Welcome to the Show', 'guest': 'Guido van Rossum'},
    {'id': 2, 'title': 'Async All the Things', 'guest': 'Andrew Godwin'},
]


@app.get('/')
@chameleon_robyn.template('home/index.pt')
async def index(request):
    await asyncio.sleep(0.001)
    return {'title': 'Chameleon + Robyn Demo', 'episode_count': len(EPISODES)}


@app.get('/episodes/:episode_id')
@chameleon_robyn.template('episodes/detail.pt')
async def episode_detail(request, episode_id: int):
    episode = next((ep for ep in EPISODES if ep['id'] == episode_id), None)
    if not episode:
        chameleon_robyn.not_found()
    return {'title': episode['title'], 'episode': episode}


@app.get('/redirect-example')
@chameleon_robyn.template('home/index.pt')
def redirect_example(request):
    # Response pass-through — no template rendering
    return Response(status_code=302, description='', headers=Headers({'Location': '/'}))


@app.get('/cookie-example')
@chameleon_robyn.template('home/index.pt')
def cookie_example(request):
    def remember_visit(resp: Response):
        resp.headers.append('Set-Cookie', 'last_visit=home; Path=/')

    return {
        'title': 'Chameleon + Robyn Demo',
        'episode_count': len(EPISODES),
        '__response_callback__': remember_visit,
    }


@app.get('/xml/episodes')
@chameleon_robyn.template('episodes/feed.xml', content_type='application/xml')
def episodes_xml(request):
    return {'episodes': EPISODES}


def main():
    template_folder = (Path(__file__).resolve().parent / 'templates').as_posix()
    chameleon_robyn.global_init(template_folder, auto_reload=True)
    app.start(host='127.0.0.1', port=5555)


if __name__ == '__main__':
    main()
```

**`templates/home/index.pt`:**

```html
<!DOCTYPE html>
<html lang="en">
<body>
<h1>${title}</h1>
<p>We have ${episode_count} episode(s). <a href="/episodes/1">See one</a>.</p>
</body>
</html>
```

**`templates/episodes/detail.pt`:**

```html
<!DOCTYPE html>
<html lang="en">
<body>
<h1>${episode['title']}</h1>
<p>Guest: ${episode['guest']}</p>
</body>
</html>
```

**`templates/episodes/feed.xml`:**

```xml
<?xml version="1.0" encoding="utf-8" ?>
<episodes>
    <episode tal:repeat="ep episodes">${ep['title']}</episode>
</episodes>
```

**`templates/errors/404.pt`:**

```html
<!DOCTYPE html>
<html lang="en">
<body>
<h1>Page not found</h1>
<p>${message}</p>
</body>
</html>
```

---

## Error-Handling Patterns

### Missing Initialization

```python
# Raises ChameleonRobynException:
# "You must call global_init() before rendering templates."
chameleon_robyn.render('home/index.pt')  # before global_init()
```

### Invalid Return Type

```python
@chameleon_robyn.template('home/index.pt')
def view(request):
    return 'just a string'  # ChameleonRobynException at request time — expected dict or Response
```

### Non-Callable `__response_callback__`

```python
@chameleon_robyn.template('home/index.pt')
def view(request):
    return {'__response_callback__': 'not callable'}  # ChameleonRobynException
```

### Missing Template File

Requesting a template file that does not exist raises `ValueError` from Chameleon's `PageTemplateLoader` at render time.

---

## API Summary Table

| Symbol | Purpose | Returns |
|---|---|---|
| `global_init(folder, ...)` | Initialize the module-level template engine | `None` |
| `clear()` | Reset engine state (mostly for tests) | `None` |
| `@template(file, ...)` | Decorator: render handler's dict through a template | `robyn.Response` |
| `response(file, ...)` | Render a template to a Robyn Response directly | `robyn.Response` |
| `render(file, ...)` | Render a template to a string | `str` |
| `not_found(file)` | Raise a 404 inside a decorated handler | `NoReturn` (always raises) |
| `ChameleonTemplate(dir, ...)` | Standalone engine matching Robyn's TemplateInterface | instance |
| `ChameleonTemplate.render_template(name, **model)` | Render via the standalone engine | `robyn.Response` (status 200) |
| `ChameleonRobynException` | Base error (config, return-type, callback misuse) | — |
| `ChameleonRobynNotFoundException` | 404 signal carrying `template_file` + `message` | — |
