# jinja-partials Comprehensive Reference

> **Version:** 0.3.2
> **Author:** Michael Kennedy (michael@talkpython.fm)
> **License:** MIT
> **Repository:** https://github.com/mikeckennedy/jinja_partials
> **Documentation:** https://mkennedy.codes/docs/jinja-partials
> **Python:** 3.10 - 3.15
> **Dependencies:** `jinja2` (which brings `markupsafe`)

## Overview

`jinja_partials` (PyPI: `jinja-partials`) adds a `render_partial()` global to Jinja2 templates so HTML fragments ("partials") can be composed and nested with explicitly passed model data — `{{ render_partial('shared/partials/video_image.html', video=video) }}`. It works with any plain Jinja2 environment and ships first-class registration helpers for Flask, Quart, FastAPI, and Starlette, including transparent support for async (`enable_async=True`) environments. The package is fully typed (PEP 561 `py.typed`).

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/jinja-partials](https://mkennedy.codes/docs/jinja-partials). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/jinja-partials
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/jinja-partials/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/jinja-partials/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/jinja-partials/SKILL.md — also at `/.well-known/agent-skills/jinja-partials/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/jinja-partials/skills.html

Every documentation page also has a plain-Markdown twin — swap `.html` for `.md` to get token-efficient source without site chrome. For example https://mkennedy.codes/docs/jinja-partials/reference/render_partial.html → https://mkennedy.codes/docs/jinja-partials/reference/render_partial.md

## Installation

```bash
pip install jinja-partials
```

Note the naming: the PyPI distribution is `jinja-partials` (hyphen) while the import is `jinja_partials` (underscore). Requires Python 3.10+ (on older Pythons, pip resolves to an earlier release that predates the FastAPI/Starlette/Quart support described here).

## Quick Start

```python
# app.py — Flask
import flask
import jinja_partials

app = flask.Flask(__name__)
jinja_partials.register_extensions(app)  # one call at startup


@app.get('/')
def index():
    return flask.render_template('home/index.html', videos=get_videos())
```

```html
<!-- templates/home/index.html -->
{% for v in videos %}
    <div class="col-md-3 video">
        {{ render_partial('shared/partials/video_square.html', video=v) }}
    </div>
{% endfor %}
```

```html
<!-- templates/shared/partials/video_square.html — partials can nest further partials -->
<div>
    <a href="https://www.youtube.com/watch?v={{ video.id }}" target="_blank">
        {{ render_partial('shared/partials/video_image.html', video=video) }}
    </a>
    <div class="views">{{ "{:,}".format(video.views) }} views</div>
</div>
```

---

## Public API Reference

Everything lives in the single module `jinja_partials`:

```python
__all__ = [
    '__version__',
    'register_extensions',
    'register_quart_extensions',
    'register_fastapi_extensions',
    'register_starlette_extensions',
    'register_environment',
    'render_partial',
    'generate_render_partial',
    'PartialsException',
    'PartialsJinjaExtension',
]
```

`markupsafe.Markup` is re-exported as `jinja_partials.Markup` (not in `__all__`). `__version__` resolves via `importlib.metadata` at runtime and is `'0.0.0'` when the package is not installed as a distribution.

---

### `register_extensions(app)` — Flask

Register `render_partial` with a Flask application. Partials render through `flask.render_template`, so Flask context processors, `g`, and `request` are available inside partials — this is the **only** registration path where that is true.

#### Signature

```python
def register_extensions(app: 'Flask') -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `app` | `flask.Flask` | *(required)* | The Flask application instance. |

#### Returns

`None`. Installs `render_partial` into `app.jinja_env.globals`.

#### Raises

- `PartialsException` if Flask is not installed (`'Install Flask to use `register_extensions`'`).

#### Example

```python
import flask
import jinja_partials

app = flask.Flask(__name__)
jinja_partials.register_extensions(app)
```

---

### `register_quart_extensions(app, max_workers=4)` — Quart

Register `render_partial` with a Quart application. Quart's Jinja environment is async, so partial renders are submitted to a `ThreadPoolExecutor` (each worker runs the render in a fresh event loop via `asyncio.run`). A dedicated executor is created in `before_serving` and shut down in `after_serving`; outside a serving cycle a per-registration fallback executor is used. A stale executor left by a previously failed startup is replaced on the next start.

#### Signature

```python
def register_quart_extensions(app: 'Quart', max_workers: int = 4) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `app` | `quart.Quart` | *(required)* | The Quart application instance. |
| `max_workers` | `int` | `4` | Maximum worker threads for rendering partials (both the dedicated and fallback executors). |

#### Returns

`None`. Installs `render_partial` (with `markup=True`) into `app.jinja_env.globals`; the serving-cycle executor is stored at `app.extensions['jinja_partials_executor']`.

#### Raises

- `PartialsException` if Quart is not installed.

#### Example

```python
from quart import Quart
import jinja_partials

app = Quart(__name__)
jinja_partials.register_quart_extensions(app)
```

---

### `register_fastapi_extensions(app, templates, max_workers=4)` — FastAPI

Register `render_partial` with a FastAPI application and its `Jinja2Templates`. Wraps the app's existing lifespan: a dedicated `ThreadPoolExecutor` is created when the lifespan starts and shut down (in a `finally`) when it exits, even if startup or shutdown raised — so the app can start/stop repeatedly (e.g. multiple `TestClient` cycles). Outside a lifespan cycle, sync environments render directly and async environments use a per-registration fallback executor.

#### Signature

```python
def register_fastapi_extensions(
    app: 'FastAPI',
    templates: 'FastAPIJinja2Templates',
    max_workers: int = 4,
) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `app` | `fastapi.FastAPI` | *(required)* | The FastAPI application instance. |
| `templates` | `fastapi.templating.Jinja2Templates` | *(required)* | The Jinja2Templates instance whose environment gets `render_partial`. |
| `max_workers` | `int` | `4` | Maximum worker threads for rendering partials. |

#### Returns

`None`. Installs `render_partial` (with `markup=True`) into `templates.env.globals`; the lifespan executor is stored at `app.state.jinja_partials_executor`.

#### Raises

- `PartialsException` if FastAPI is not installed.

#### Example

```python
from fastapi import FastAPI
from fastapi.templating import Jinja2Templates
import jinja_partials

app = FastAPI()
templates = Jinja2Templates(directory='templates')
jinja_partials.register_fastapi_extensions(app, templates)
```

---

### `register_starlette_extensions(templates, app=None, max_workers=4)` — Starlette

Register `render_partial` with Starlette `Jinja2Templates`. **Argument order differs from the FastAPI helper: `templates` comes first and `app` is an optional keyword.** With an `app`, executor lifecycle is tied to the app's lifespan exactly as in the FastAPI helper. Without an `app` (backwards compatible), sync environments render directly and async environments use a shared module-level executor.

#### Signature

```python
def register_starlette_extensions(
    templates: 'Jinja2Templates',
    app: Optional['Starlette'] = None,
    max_workers: int = 4,
) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `templates` | `starlette.templating.Jinja2Templates` | *(required)* | The Jinja2Templates instance whose environment gets `render_partial`. |
| `app` | `Optional[starlette.applications.Starlette]` | `None` | Optional app for executor lifecycle management (recommended). |
| `max_workers` | `int` | `4` | Maximum worker threads. Only used when `app` is provided. |

#### Returns

`None`. Installs `render_partial` (with `markup=True`) into `templates.env.globals`.

#### Raises

- `PartialsException` if Starlette is not installed.

#### Example

```python
from starlette.applications import Starlette
from starlette.templating import Jinja2Templates
import jinja_partials

templates = Jinja2Templates(directory='templates')
app = Starlette(routes=[...])
jinja_partials.register_starlette_extensions(templates, app=app)
```

---

### `register_environment(env, markup=False)` — plain Jinja2

Register `render_partial` with any Jinja2 `Environment` — standalone Jinja2 usage or frameworks without dedicated support. Handles both sync and async (`enable_async=True`) environments; async renders go through a shared module-level executor.

#### Signature

```python
def register_environment(env: Environment, markup: bool = False) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `env` | `jinja2.Environment` | *(required)* | The environment to make `render_partial` available in. |
| `markup` | `bool` | `False` | When `True`, wrap rendered partials in `markupsafe.Markup`. **Note the default is `False` here, unlike every framework path (all `markup=True`).** Pass `markup=True` for autoescaping environments or the partial's HTML will be re-escaped when inserted. |

#### Returns

`None`. Installs `render_partial` into `env.globals`.

#### Example

```python
from jinja2 import Environment, FileSystemLoader
import jinja_partials

environment = Environment(loader=FileSystemLoader('templates'), autoescape=True)
jinja_partials.register_environment(environment, markup=True)
```

---

### `render_partial(template_name, renderer=None, markup=True, **data)`

Render a partial template and return the resulting HTML fragment. This is the core primitive — the function templates call as `{{ render_partial(...) }}` — and it can also be called directly from Python. With no `renderer`, it falls back to `flask.render_template`.

#### Signature

```python
def render_partial(
    template_name: str,
    renderer: Optional[Callable[..., Any]] = None,
    markup: bool = True,
    **data: Any,
) -> Union[Markup, str]
```

Typed `@overload`s narrow the return type: `markup=True` (the default) → `Markup`; `markup=False` (keyword-only in that overload) → `str`.

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `template_name` | `str` | *(required)* | Path of the template within the templates folder, e.g. `shared/partials/video_image.html`. |
| `renderer` | `Optional[Callable[..., Any]]` | `None` | Callable rendering the template with keyword arguments. Defaults to `flask.render_template` when Flask is installed. |
| `markup` | `bool` | `True` | When `True`, wrap the result in `markupsafe.Markup` so autoescaping does not re-escape it. |
| `**data` | `Any` | — | Model data passed to the template as keyword arguments. |

#### Returns

The rendered fragment as `Markup`, or `str` when `markup=False`.

#### Raises

- `PartialsException` (`'No renderer specified'`) if `renderer` is `None` and Flask is not installed.

#### Example

```python
# Inside a Flask request context (uses flask.render_template by default):
html = jinja_partials.render_partial('shared/partials/video_image.html', video=video)
```

---

### `generate_render_partial(renderer, markup=True)`

Create a `render_partial` function bound to a specific renderer — literally `functools.partial(render_partial, renderer=renderer, markup=markup)`. This is what every `register_*` function installs as the Jinja global; use it directly to wire a custom renderer.

#### Signature

```python
def generate_render_partial(
    renderer: Callable[..., Any],
    markup: bool = True,
) -> Callable[..., Union[Markup, str]]
```

Typed `@overload`s narrow the returned callable: `markup=True` → `Callable[..., Markup]`, `markup=False` → `Callable[..., str]`.

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `renderer` | `Callable[..., Any]` | *(required)* | Callable that renders a template name plus keyword arguments to a string, i.e. `renderer(template_name, **data) -> str`. |
| `markup` | `bool` | `True` | When `True`, results are wrapped in `markupsafe.Markup`. |

#### Returns

A callable with the same signature as `render_partial` that uses the bound renderer.

#### Example

```python
def my_renderer(template_name: str, **data) -> str:
    return my_engine.get_template(template_name).render(**data)

env.globals.update(render_partial=jinja_partials.generate_render_partial(my_renderer))
```

---

### `PartialsJinjaExtension` — declarative Jinja2 extension

Jinja2 extension class that registers `render_partial` when listed in an environment's `extensions=[...]` (or added via `env.add_extension`). Internally calls `register_environment(environment, markup=True)` — note **markup is `True` here**, unlike calling `register_environment` yourself.

#### Signature

```python
class PartialsJinjaExtension(Extension):
    def __init__(self, environment: Environment) -> None
```

Jinja2 passes `environment` automatically; you never construct it yourself.

#### Example

```python
from jinja2 import Environment, FileSystemLoader

environment = Environment(
    loader=FileSystemLoader('templates'),
    extensions=['jinja_partials.PartialsJinjaExtension'],
)
# render_partial is now available in templates — no further setup.
```

On a Flask app: `app.jinja_env.add_extension('jinja_partials.PartialsJinjaExtension')`. Caveat: this path renders partials directly through the Jinja environment, **not** `flask.render_template`, so Flask context processors and the `template_rendered` signal do not apply inside partials. Flask apps relying on those should use `register_extensions(app)` instead.

---

## Exceptions

### `PartialsException`

```python
class PartialsException(Exception):
    pass
```

The library's only exception type. Subclasses `Exception` directly, adds no constructor parameters or attributes beyond the message. Raised when:

- `render_partial` is called with no `renderer` and Flask is not installed (`'No renderer specified'`).
- A `register_*` function is called for a framework that is not installed (e.g. `'Install FastAPI to use `register_fastapi_extensions`'`).

---

## Internal API

Not in `__all__`; useful for understanding behavior and for testing, but subject to change.

### `_render_template_blocking(env, template_name, **data) -> str`

Synchronous render of a template from a (possibly async) environment. For `env.is_async` environments it submits the render to the module-level executor; for sync environments it renders directly via `template.new_context(data)` + `concat(template.root_render_func(ctx))`, routing errors through `env.handle_exception()`. This is the renderer behind `register_environment` and the app-less Starlette path.

### `_run_async_render(template, data) -> str`

Runs `template.render_async(**data)` to completion inside a fresh event loop via `asyncio.run()`. Always called from an executor worker thread, never from the serving event loop.

### `_async_render_executor`

Module-level `ThreadPoolExecutor(max_workers=4)` shared by `register_environment`, `PartialsJinjaExtension`, and app-less `register_starlette_extensions` when the environment is async. Intentionally never shut down (process-lifetime).

---

## How async rendering works (the executor model)

Jinja template expressions cannot `await`, but Quart/FastAPI/Starlette typically use `enable_async=True` environments where `template.render()` is unusable from a running event loop. The bridge: the installed renderer submits the async render to a `ThreadPoolExecutor` worker thread, which runs it in a fresh event loop (`asyncio.run`) while the caller blocks on `future.result()`. The render blocks one worker thread per partial — an accepted tradeoff for making `{{ render_partial(...) }}` work unchanged in async apps.

Three executor tiers, in order of preference:

1. **Dedicated executor** tied to the app's serving lifecycle — FastAPI/Starlette via a wrapped lifespan (stored on `app.state.jinja_partials_executor`), Quart via `before_serving`/`after_serving` (stored in `app.extensions['jinja_partials_executor']`). Created on startup, shut down cleanly on shutdown, exception-safe and re-entrant across repeated start/stop cycles.
2. **Per-registration fallback executor** (a closure variable inside each `register_*` call) used for renders outside a serving/lifespan cycle. Honors `max_workers`; never shut down.
3. **Module-level `_async_render_executor`** for `register_environment` / `PartialsJinjaExtension` / app-less Starlette. Never shut down.

Caveat: calling `register_fastapi_extensions` or `register_starlette_extensions` twice on the same app double-wraps the lifespan — there is no idempotency guard. Register once at startup.

---

## `markup` defaults at a glance

| Path | `markup` | Return in templates |
|---|---|---|
| `render_partial(...)` direct call | `True` (default) | `Markup` |
| `generate_render_partial(renderer)` | `True` (default) | `Markup` |
| `register_extensions` (Flask) | `True` (fixed) | `Markup` |
| `register_quart_extensions` | `True` (fixed) | `Markup` |
| `register_fastapi_extensions` | `True` (fixed) | `Markup` |
| `register_starlette_extensions` | `True` (fixed) | `Markup` |
| `PartialsJinjaExtension` | `True` (fixed) | `Markup` |
| `register_environment` | **`False` (default)** | `str` unless you pass `markup=True` |

With autoescaping on and `markup=False`, the partial's HTML is escaped and shows up as literal `&lt;div&gt;...` text — if you see escaped HTML where a partial should be, this is why.

---

## Project structure conventions

Partials are ordinary Jinja templates; the recommended convention is a `partials` subfolder under `templates/shared`, with template paths always given relative to the templates root:

```
├── templates
│   ├── home
│   │   ├── index.html
│   │   └── listing.html
│   └── shared
│       ├── _layout.html
│       └── partials
│           ├── video_image.html
│           └── video_square.html
```

A partial is just an HTML fragment taking its model from `render_partial`'s keyword arguments:

```html
<!-- templates/shared/partials/video_image.html -->
<img src="https://img.youtube.com/vi/{{ video.id }}/maxresdefault.jpg"
     class="img img-responsive {{ (classes | default([])) | join(' ') }}"
     alt="{{ video.title }}"
     title="{{ video.title }}">
```

Partials nest: a partial can call `render_partial` on another partial (including recursively). Data flow is always explicit — only what you pass as keyword arguments is available in the partial (plus Flask context, on the Flask `register_extensions` path only).

Why not Jinja's built-in `include` or `macro`? They are nearly the same but fall short: `include` shares the caller's whole context implicitly rather than taking an explicit model, and macros must be imported and don't compose as cleanly across folders. See the discussion at https://github.com/mikeckennedy/jinja_partials/issues/1

---

## Framework compatibility notes

- **Flask** — `register_extensions(app)`. The only path rendering through `flask.render_template`; context processors, `g`, and `request` work inside partials.
- **Quart** — `register_quart_extensions(app)`. Async environment; partials render on the executor.
- **FastAPI** — `register_fastapi_extensions(app, templates)`. Works with sync or async (`enable_async=True`) `Jinja2Templates` environments.
- **Starlette** — `register_starlette_extensions(templates, app=app)`. Same as FastAPI but note the argument order (templates first) and that `app` is optional.
- **Anything else / plain Jinja2** — `register_environment(env, markup=True)` or the `PartialsJinjaExtension`.
- **FastAPI/Starlette page rendering with `enable_async=True`:** render pages with `await template.render_async(...)` and return an `HTMLResponse`. `templates.TemplateResponse(...)` on an async environment calls `template.render()` → `asyncio.run()` inside the running event loop and crashes. Partials inside the page still render synchronously via the app-managed executor — that part needs no changes.

---

## Complete working example (FastAPI, async environment)

```python
# app.py
from pathlib import Path

import jinja2
from fastapi import FastAPI
from fastapi.responses import HTMLResponse
from fastapi.templating import Jinja2Templates

import jinja_partials

app = FastAPI()

env = jinja2.Environment(
    loader=jinja2.FileSystemLoader(Path(__file__).parent / 'templates'),
    autoescape=True,
    enable_async=True,
)
templates = Jinja2Templates(env=env)

# Ties the partial-rendering executor to the app's lifespan.
jinja_partials.register_fastapi_extensions(app, templates)


@app.get('/')
async def index() -> HTMLResponse:
    items = [
        {'title': 'Python Basics', 'author': 'Alice', 'views': 12500, 'tag': 'beginner'},
        {'title': 'Async Patterns', 'author': 'Bob', 'views': 8750, 'tag': 'advanced'},
    ]
    # Async environment: render the page via render_async, not TemplateResponse.
    template = templates.get_template('home/index.html')
    return HTMLResponse(await template.render_async(items=items))


if __name__ == '__main__':
    import uvicorn

    uvicorn.run(app, host='127.0.0.1', port=10003)
```

```html
<!-- templates/home/index.html -->
<!DOCTYPE html>
<html lang="en">
<body>
<h1>Videos</h1>
{% for item in items %}
    {{ render_partial('shared/partials/card.html', item=item) }}
{% endfor %}
</body>
</html>
```

```html
<!-- templates/shared/partials/card.html — nests another partial -->
<div class="card">
    <h2>{{ item.title }}</h2>
    <p>{{ item.author }} — {{ "{:,}".format(item.views) }} views</p>
    {{ render_partial('shared/partials/badge.html', tag=item.tag) }}
</div>
```

```html
<!-- templates/shared/partials/badge.html -->
<span class="badge">{{ tag }}</span>
```

The repository ships four parallel runnable demos in [`examples/`](https://github.com/mikeckennedy/jinja_partials/tree/main/examples): Flask (port 5001), Quart (10001), Starlette (10002), and FastAPI (10003).

---

## Error-handling patterns

### Missing framework

```python
import jinja_partials

# Flask not installed:
jinja_partials.register_extensions(app)
# PartialsException: Install Flask to use `register_extensions`
```

### No renderer available

```python
# Flask not installed and no renderer passed:
jinja_partials.render_partial('shared/partials/thing.html', model=m)
# PartialsException: No renderer specified
```

### Template errors inside partials

Missing templates raise Jinja2's `TemplateNotFound` from `env.get_template` (or from `flask.render_template` on the Flask path); undefined-variable behavior follows the environment's `undefined` policy. On the direct-render paths, render errors route through `env.handle_exception()`, preserving Jinja's normal traceback rewriting.

### Escaped HTML instead of rendered partial

The partial's output was a plain `str` in an autoescaping environment — you used `register_environment` (default `markup=False`) without `markup=True`. Fix at registration, not per-call.

---

## API summary table

| Symbol | Purpose | Returns |
|---|---|---|
| `register_extensions(app)` | Flask registration (via `flask.render_template`) | `None` |
| `register_quart_extensions(app, max_workers=4)` | Quart registration with managed executor | `None` |
| `register_fastapi_extensions(app, templates, max_workers=4)` | FastAPI registration with lifespan-managed executor | `None` |
| `register_starlette_extensions(templates, app=None, max_workers=4)` | Starlette registration (templates first!) | `None` |
| `register_environment(env, markup=False)` | Plain Jinja2 registration | `None` |
| `render_partial(name, renderer=None, markup=True, **data)` | Render one partial | `Markup` (or `str` if `markup=False`) |
| `generate_render_partial(renderer, markup=True)` | Bind a renderer; what registration installs | `Callable[..., Markup \| str]` |
| `PartialsJinjaExtension` | Declarative registration via `extensions=[...]` (markup on) | — |
| `PartialsException` | All misconfiguration / missing-framework errors | — |
| `__version__` | Installed distribution version | `str` |
