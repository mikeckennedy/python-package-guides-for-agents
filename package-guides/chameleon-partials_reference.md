# chameleon_partials: Comprehensive Reference

> Simple reuse of partial HTML page templates in the Chameleon template language for Python web frameworks.

**Version:** 0.1.0
**Author:** Michael Kennedy <michael@talkpython.fm>
**License:** MIT
**PyPI:** `pip install chameleon-partials`
**Source:** <https://github.com/mikeckennedy/chameleon_partials>
**Dependency:** [Chameleon](https://chameleon.readthedocs.io/) (the template engine used by Pyramid)
**Companion library:** [jinja_partials](https://github.com/mikeckennedy/jinja_partials) (Jinja2/Flask equivalent)

---

## Table of Contents

- [What It Does](#what-it-does)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [register_extensions()](#register_extensions)
  - [render_partial()](#render_partial)
  - [extend_model()](#extend_model)
  - [HTML class](#html-class)
  - [PartialsException](#partialsexception)
- [Module-Level State](#module-level-state)
- [Framework Integration](#framework-integration)
  - [Pyramid](#pyramid-integration)
  - [Other Frameworks (FastAPI, etc.)](#other-frameworks)
- [Template Authoring](#template-authoring)
  - [Basic Partials](#basic-partials)
  - [Passing Data to Partials](#passing-data-to-partials)
  - [Nesting Partials](#nesting-partials)
  - [Using Partials with Layouts (METAL macros)](#using-partials-with-layouts)
  - [Chameleon Syntax Quick Reference](#chameleon-syntax-quick-reference)
- [Recommended Project Structure](#recommended-project-structure)
- [Error Handling](#error-handling)
- [Testing](#testing)
- [Complete Example: Pyramid Video Gallery](#complete-example)

---

## What It Does

When building web apps with Chameleon templates, HTML fragments are often repeated across pages. `chameleon_partials` lets you extract those fragments into standalone `.pt` template files and render them anywhere with a single function call, including nesting partials inside other partials.

**Core concept:** Call `render_partial('path/to/fragment.pt', key=value, ...)` from any Chameleon template to inline a rendered HTML fragment, passing data as keyword arguments.

---

## Installation

```bash
pip install chameleon-partials
```

Pure Python. The only runtime dependency is `chameleon`.

---

## Quick Start

### 1. Register the template folder at app startup

```python
from pathlib import Path
import chameleon_partials

folder = (Path(__file__).parent / "templates").as_posix()
chameleon_partials.register_extensions(folder, auto_reload=True)
```

### 2. Create a partial template (`templates/shared/partials/greeting.pt`)

```html
<div class="greeting">
    <h2>Hello, ${ name }!</h2>
</div>
```

### 3. Use it from any Chameleon template

```html
${ render_partial('shared/partials/greeting.pt', name='Michael') }
```

### 4. Make `render_partial` available in templates

**Pyramid** — add a `BeforeRender` subscriber (details [below](#pyramid-integration)):
```python
@subscriber(BeforeRender)
def add_global(event):
    event['render_partial'] = chameleon_partials.render_partial
```

**Other frameworks** — extend your view model:
```python
model = dict(name='Michael')
return chameleon_partials.extend_model(model)
```

---

## API Reference

All public symbols are exported in `chameleon_partials.__all__`:

```python
__all__ = ['register_extensions', 'render_partial', 'PartialsException', 'extend_model']
```

---

### `register_extensions()`

```python
def register_extensions(
    template_folder: str,
    auto_reload: bool = False,
    cache_init: bool = True
) -> None
```

Initializes the chameleon_partials template system. **Must be called once at application startup** before any calls to `render_partial()`.

#### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `template_folder` | `str` | *(required)* | Absolute or relative path to the root templates directory. All partial template paths are resolved relative to this folder. |
| `auto_reload` | `bool` | `False` | When `True`, Chameleon will detect template file changes and reload them automatically. Enable during development, disable in production for performance. |
| `cache_init` | `bool` | `True` | When `True`, subsequent calls to `register_extensions()` are no-ops if already registered. Set to `False` to force re-registration (useful in tests). |

#### Behavior

1. If `cache_init=True` and already registered, returns immediately (no-op).
2. Sets `has_registered_extensions = True`.
3. Validates `template_folder` is not empty/falsy — raises `PartialsException` if it is.
4. Validates `template_folder` exists and is a directory — raises `PartialsException` if not.
5. Creates a `chameleon.PageTemplateLoader` instance pointed at the folder.

#### Raises

- `PartialsException("The template_folder must be specified.")` — if `template_folder` is empty or falsy.
- `PartialsException("The specified template folder must be a folder, it's not: {path}")` — if path does not exist or is not a directory.

#### Example

```python
from pathlib import Path
import chameleon_partials

# Typical Pyramid setup — resolve relative to the current package
folder = (Path(__file__).parent / "templates").as_posix()
chameleon_partials.register_extensions(folder, auto_reload=True, cache_init=True)
```

#### Important Notes

- Use `.as_posix()` on `Path` objects to ensure forward slashes on all platforms.
- The `PageTemplateLoader` caches compiled templates internally for performance.
- With `auto_reload=True`, templates are checked for modifications on each render (development convenience, slight performance cost).

---

### `render_partial()`

```python
def render_partial(
    template_file: str,
    **template_data
) -> HTML
```

Renders a Chameleon template file and returns the result as an HTML-safe object. This is the primary function you call from within templates.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `template_file` | `str` | Relative path from the registered template folder to the `.pt` file. Uses forward slashes as path separators (e.g., `'shared/partials/video_image.pt'`). |
| `**template_data` | keyword args | Arbitrary keyword arguments passed as template variables. Each key becomes a variable name accessible in the template via `${ key }` or TAL expressions. |

#### Returns

`HTML` — An object wrapping the rendered HTML string. Chameleon recognizes this via the `__html__()` protocol and outputs it without escaping.

#### Behavior

1. Checks that `register_extensions()` has been called; raises `PartialsException` if not.
2. **Automatically injects `render_partial` into `template_data`** if not already present — this is how nested partials work without explicit wiring.
3. Loads the template via the `PageTemplateLoader` using `__templates[template_file]`.
4. Renders with `encoding='utf-8'` and passes all `**template_data` as template variables.
5. Wraps the rendered string in an `HTML` object to prevent double-escaping.

#### Raises

- `PartialsException("You must call register_extensions() before this function can be used.")` — if called before registration.
- `ValueError` (from Chameleon) — if `template_file` does not exist in the registered folder.

#### Examples

**Simple partial call from a template:**
```html
${ render_partial('shared/partials/video_image.pt', video=video, classes=['thumbnail']) }
```

**Iterating and rendering partials:**
```html
<div class="video" tal:repeat="v videos">
    ${ render_partial('shared/partials/video_square.pt', video=v) }
</div>
```

**Calling from Python code (e.g., for HTMX responses or email rendering):**
```python
html = chameleon_partials.render_partial('emails/welcome.pt', user=user)
html_string = html.html_text  # Access the raw HTML string
```

#### Key Design Detail: Auto-Injection of `render_partial`

When `render_partial()` is called, it automatically adds itself to the template data:

```python
if 'render_partial' not in template_data:
    template_data['render_partial'] = render_partial
```

This means any partial template can call `render_partial()` to nest further partials without the caller needing to pass it explicitly. Nesting depth is unlimited.

---

### `extend_model()`

```python
def extend_model(
    model: Dict[str, Any]
) -> Dict[str, Any]
```

Helper function that injects `render_partial` into a view model dictionary. Use this in frameworks that don't have Pyramid's `BeforeRender` event system.

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model` | `Dict[str, Any]` or `None` | The view model dictionary that will be passed to the template renderer. If `None`, an empty dict is created. |

#### Returns

The same `model` dict, now containing `model['render_partial'] = render_partial`.

#### Raises

- `PartialsException("The model must be a dictionary.")` — if `model` is not `None` and not a `dict`.

#### Example

```python
@view_config(route_name='listing', renderer='myapp:templates/listing.pt')
def listing(request):
    videos = video_service.all_videos()
    model = dict(videos=videos)
    return chameleon_partials.extend_model(model)
```

#### When to Use

- Use `extend_model()` when you **cannot** use Pyramid's `BeforeRender` subscriber pattern.
- If using Pyramid, prefer the [middleware approach](#pyramid-integration) instead — it's cleaner and doesn't require modifying every view.

---

### `HTML` Class

```python
class HTML:
    def __init__(self, html_text: str)
    def __html__(self) -> str
```

Wrapper for rendered HTML content. Implements Chameleon's `__html__()` protocol so the output is treated as safe markup and not double-escaped.

#### Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| `html_text` | `str` | The raw rendered HTML string. |

#### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `__html__()` | `str` | Returns `self.html_text`. Chameleon calls this method to determine that the value is already safe HTML and should not be escaped. |

#### Usage

You typically don't instantiate `HTML` directly — it's returned by `render_partial()`. Access the raw HTML via `.html_text`:

```python
result = chameleon_partials.render_partial('fragment.pt', title='Hello')
raw_html = result.html_text  # '<div><h1>Hello</h1></div>'
```

---

### `PartialsException`

```python
class PartialsException(Exception):
    pass
```

Custom exception class for all chameleon_partials-specific errors. Inherits directly from `Exception` with no additional behavior.

#### When It's Raised

| Scenario | Message |
|----------|---------|
| `render_partial()` called before `register_extensions()` | `"You must call register_extensions() before this function can be used."` |
| `register_extensions()` called with empty/falsy `template_folder` | `"The template_folder must be specified."` |
| `register_extensions()` called with a path that isn't a directory | `"The specified template folder must be a folder, it's not: {path}"` |
| `extend_model()` called with non-dict, non-None value | `"The model must be a dictionary."` |

---

## Module-Level State

The library uses three module-level globals to track initialization state:

```python
has_registered_extensions: bool = False    # Gate for render_partial()
__templates: Optional[PageTemplateLoader] = None  # Cached template loader
template_path: Optional[str] = None        # Registered folder path
```

- `has_registered_extensions` is checked on every `render_partial()` call.
- `__templates` holds the Chameleon `PageTemplateLoader`, which handles template caching and optional auto-reload.
- `template_path` stores the registered directory path.
- When `cache_init=True` (default), calling `register_extensions()` a second time is a no-op.
- In tests, reset state between tests by setting `chameleon_partials.has_registered_extensions = False`.

---

## Framework Integration

### Pyramid Integration

Pyramid is the primary target framework. Integration uses Pyramid's event system to make `render_partial` globally available in all templates.

#### Step 1: Register extensions in `main()`

```python
# __init__.py (your Pyramid app entry point)
from pathlib import Path
from pyramid.config import Configurator
import chameleon_partials

def main(_, **settings):
    with Configurator(settings=settings) as config:
        config.include('pyramid_chameleon')
        config.include('.routes')
        config.scan()

        folder = (Path(__file__).parent / "templates").as_posix()
        chameleon_partials.register_extensions(folder, auto_reload=True, cache_init=True)

    return config.make_wsgi_app()
```

#### Step 2: Add BeforeRender middleware

Create a file in your `views/` folder (e.g., `views/partials_middleware.py`):

```python
from pyramid.events import subscriber, BeforeRender
import chameleon_partials

@subscriber(BeforeRender)
def add_global(event):
    event['render_partial'] = chameleon_partials.render_partial
```

The `config.scan()` call in `main()` will discover this subscriber automatically.

#### Step 3: Use in templates

View functions return plain dicts — no modification needed:

```python
@view_config(route_name='index', renderer='myapp:templates/home/index.pt')
def index(request):
    return dict(videos=video_service.top_videos())
```

All templates automatically have access to `render_partial`.

---

### Other Frameworks

For frameworks without Pyramid's event system (FastAPI, plain WSGI, etc.):

#### Step 1: Register extensions at startup

```python
import chameleon_partials
from pathlib import Path

folder = (Path(__file__).parent / "templates").as_posix()
chameleon_partials.register_extensions(folder, auto_reload=True)
```

#### Step 2: Extend the model in each view

```python
def my_view(request):
    model = dict(items=get_items())
    return chameleon_partials.extend_model(model)
```

Or manually:

```python
def my_view(request):
    return dict(
        items=get_items(),
        render_partial=chameleon_partials.render_partial,
    )
```

---

## Template Authoring

### Basic Partials

A partial is a standard Chameleon `.pt` file containing an HTML fragment — no full document structure needed.

```html
<!-- templates/shared/partials/alert.pt -->
<div class="alert alert-info">
    <strong>Notice:</strong> ${ message }
</div>
```

Call it from another template:

```html
${ render_partial('shared/partials/alert.pt', message='System update scheduled.') }
```

### Passing Data to Partials

All keyword arguments to `render_partial()` become template variables:

```html
<!-- Calling template -->
${ render_partial('shared/partials/user_card.pt', user=current_user, show_avatar=True) }

<!-- shared/partials/user_card.pt -->
<div class="card">
    <img tal:condition="show_avatar" src="${ user.avatar_url }" />
    <h3>${ user.display_name }</h3>
</div>
```

### Nesting Partials

Partials can render other partials. `render_partial` is automatically available inside every partial — no explicit passing required.

```html
<!-- shared/partials/video_square.pt -->
<div>
    <a href="https://www.youtube.com/watch?v=${ video.id }" target="_blank">
        ${ render_partial('shared/partials/video_image.pt', video=video, classes=[]) }
    </a>
    <a href="https://www.youtube.com/watch?v=${ video.id }" target="_blank"
       class="author">${ video.author }</a>
    <div class="views">${ "{:,}".format(video.views) } views</div>
</div>

<!-- shared/partials/video_image.pt (called by video_square.pt) -->
<img src="https://img.youtube.com/vi/${ video.id }/maxresdefault.jpg"
     class="img img-responsive ${ ' '.join(classes or []) }"
     alt="${ video.title }"
     title="${ video.title }">
```

Nesting depth is unlimited.

### Using Partials with Layouts

Partials work alongside Chameleon's METAL macro system for template inheritance:

```html
<!-- shared/_layout.pt -->
<!DOCTYPE html metal:define-macro="layout">
<html lang="en">
<head>
    <title>My App</title>
</head>
<body>
    <div class="main_content">
        <div metal:define-slot="content">No content</div>
    </div>
</body>
</html>

<!-- home/index.pt (page template using layout + partials) -->
<div metal:use-macro="load: ../shared/_layout.pt">
    <div metal:fill-slot="content">
        <h1>Videos</h1>
        <div class="video" tal:repeat="v videos">
            ${ render_partial('shared/partials/video_square.pt', video=v) }
        </div>
    </div>
</div>
```

### Chameleon Syntax Quick Reference

These are the Chameleon template features commonly used alongside partials:

| Syntax | Purpose | Example |
|--------|---------|---------|
| `${ expr }` | Expression interpolation | `${ video.title }` |
| `${ "{:,}".format(n) }` | Python expressions | `${ "{:,}".format(video.views) }` |
| `tal:repeat="item items"` | Iteration | `<div tal:repeat="v videos">` |
| `tal:condition="expr"` | Conditional rendering | `<span tal:condition="show_count">` |
| `tal:content="expr"` | Set element content | `<span tal:content="user.name">placeholder</span>` |
| `tal:attributes="attr expr"` | Dynamic attributes | `<div tal:attributes="class css_class">` |
| `metal:define-macro="name"` | Define a reusable layout macro | On the root element of a layout |
| `metal:use-macro="load: path"` | Use a layout macro | `metal:use-macro="load: ../shared/_layout.pt"` |
| `metal:define-slot="name"` | Define a fillable slot in a layout | `<div metal:define-slot="content">` |
| `metal:fill-slot="name"` | Fill a slot in a used layout | `<div metal:fill-slot="content">` |

---

## Recommended Project Structure

```
your_app/
├── __init__.py                    # Call register_extensions() here
├── routes.py
├── views/
│   ├── home.py                    # View functions return plain dicts
│   └── partials_middleware.py     # BeforeRender subscriber (Pyramid)
├── templates/
│   ├── home/
│   │   ├── index.pt               # Page templates
│   │   └── listing.pt
│   ├── errors/
│   │   └── 404.pt
│   └── shared/
│       ├── _layout.pt             # Master layout (METAL macro)
│       └── partials/
│           ├── video_image.pt     # Reusable HTML fragments
│           └── video_square.pt
```

**Conventions:**

- Prefix layout files with `_` (e.g., `_layout.pt`).
- Group partials under `shared/partials/`.
- Template paths in `render_partial()` are always relative to the registered folder (e.g., `'shared/partials/video_image.pt'`).

---

## Error Handling

### Error Summary

| Error | Type | Cause |
|-------|------|-------|
| Not registered | `PartialsException` | `render_partial()` called before `register_extensions()` |
| Empty folder path | `PartialsException` | `register_extensions('')` or `register_extensions(None)` |
| Invalid folder path | `PartialsException` | Path doesn't exist or is a file, not a directory |
| Non-dict model | `PartialsException` | `extend_model(42)` or `extend_model("string")` |
| Missing template | `ValueError` | Template file not found in the registered folder (raised by Chameleon's `PageTemplateLoader`) |
| Template syntax error | *(Chameleon exceptions)* | Invalid Chameleon syntax in `.pt` file |

### Catching Errors

```python
from chameleon_partials import PartialsException

try:
    chameleon_partials.register_extensions('/bad/path')
except PartialsException as e:
    print(f"Setup failed: {e}")
```

---

## Testing

### Test Fixture Pattern

```python
import pytest
from pathlib import Path
import chameleon_partials

@pytest.fixture
def registered_extension():
    folder = (Path(__file__).parent / "test_templates").as_posix()
    chameleon_partials.register_extensions(folder, auto_reload=True, cache_init=True)

    yield

    # Reset global state for the next test
    chameleon_partials.has_registered_extensions = False
```

### Test Examples

```python
def test_render_empty(registered_extension):
    html = chameleon_partials.render_partial('render/bare.pt')
    assert '<h1>This is bare HTML fragment</h1>' in html.html_text

def test_render_with_data(registered_extension):
    html = chameleon_partials.render_partial('render/with_data.pt', name='Sarah', age=32)
    assert 'Your name is Sarah and age is 32' in html.html_text

def test_render_recursive(registered_extension):
    html = chameleon_partials.render_partial(
        'render/recursive.pt',
        message="outer message",
        inner="inner message"
    )
    assert "outer message" in html.html_text
    assert "inner message" in html.html_text

def test_missing_template(registered_extension):
    with pytest.raises(ValueError):
        chameleon_partials.render_partial('no-way.pt', message=7)

def test_not_registered():
    with pytest.raises(chameleon_partials.PartialsException):
        chameleon_partials.render_partial('doesnt-matter.pt', message=7)
```

### Running Tests

```bash
python3 -m pytest tests/           # All tests
python3 -m pytest tests/ -v        # Verbose
```

---

## Complete Example

A full Pyramid video gallery application is included in the `example/` folder of the repository.

### Application Startup

```python
# example/demo_chameleon_partials/__init__.py
from pathlib import Path
from pyramid.config import Configurator
import chameleon_partials

def main(_, **settings):
    with Configurator(settings=settings) as config:
        config.include('pyramid_chameleon')
        config.include('.routes')
        config.scan()

        folder = (Path(__file__).parent / "templates").as_posix()
        chameleon_partials.register_extensions(folder, auto_reload=True, cache_init=True)

    return config.make_wsgi_app()
```

### Middleware (Pyramid-specific)

```python
# example/demo_chameleon_partials/views/partials_middleware.py
from pyramid.events import subscriber, BeforeRender
import chameleon_partials

@subscriber(BeforeRender)
def add_global(event):
    event['render_partial'] = chameleon_partials.render_partial
```

### View Functions

```python
# example/demo_chameleon_partials/views/home.py
from pyramid.view import view_config
from demo_chameleon_partials.services import video_service

@view_config(route_name='index', renderer='demo_chameleon_partials:templates/home/index.pt')
def index(_):
    row1 = video_service.top_videos()
    return dict(rows=[row1])

@view_config(route_name='listing', renderer='demo_chameleon_partials:templates/home/listing.pt')
def listing(_):
    videos = video_service.all_videos()
    return dict(videos=videos)
```

### Page Template with Partials

```html
<!-- templates/home/index.pt -->
<div metal:use-macro="load: ../shared/_layout.pt">
    <div metal:fill-slot="content">
        <h1>Demo app: chameleon_partials</h1>
        <div class="container videos category">
            <div class="row" tal:repeat="row rows">
                <div class="col-md-3 video" tal:repeat="v row">
                    ${ render_partial('shared/partials/video_square.pt', video=v) }
                </div>
            </div>
        </div>
    </div>
</div>
```

### Partial: Video Square (nests Video Image)

```html
<!-- templates/shared/partials/video_square.pt -->
<div>
    <a href="https://www.youtube.com/watch?v=${ video.id }" target="_blank">
        ${ render_partial('shared/partials/video_image.pt', video=video, classes=[]) }
    </a>
    <a href="https://www.youtube.com/watch?v=${ video.id }" target="_blank"
       class="author">${ video.author }</a>
    <div class="views">${ "{:,}".format(video.views) } views</div>
</div>
```

### Partial: Video Image (leaf partial)

```html
<!-- templates/shared/partials/video_image.pt -->
<img src="https://img.youtube.com/vi/${ video.id }/maxresdefault.jpg"
     class="img img-responsive ${ ' '.join(classes or []) }"
     alt="${ video.title }"
     title="${ video.title }">
```

### Data Flow Summary

```
View function
  └─ returns dict(rows=[[Video, Video, Video]])
      └─ Pyramid BeforeRender adds render_partial to the dict
          └─ index.pt iterates rows → calls render_partial('video_square.pt', video=v)
              └─ video_square.pt calls render_partial('video_image.pt', video=video, classes=[])
                  └─ video_image.pt renders <img> tag (leaf — no further nesting)
```

### Running the Example

```bash
cd example/
pip install -e .
pserve development.ini    # Serves on http://localhost:6543
```
