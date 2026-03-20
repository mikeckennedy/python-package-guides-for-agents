# NiceGUI Comprehensive Reference

> A detailed reference for building web applications with NiceGUI — covering every element, function signature, pattern, and deployment option.

NiceGUI is a Python-based UI framework that lets you build web applications with a simple, declarative API. It is built on top of FastAPI, Vue 3, and Quasar, and uses WebSockets for real-time communication between the Python backend and the browser frontend.

---

## Table of Contents

- [Getting Started](#getting-started)
- [Running Your App](#running-your-app)
- [Pages and Routing](#pages-and-routing)
- [Layout and Containers](#layout-and-containers)
- [Text Elements](#text-elements)
- [Input Controls](#input-controls)
- [Buttons and Actions](#buttons-and-actions)
- [Data Display](#data-display)
- [Charts and Visualization](#charts-and-visualization)
- [Media Elements](#media-elements)
- [Feedback and Overlays](#feedback-and-overlays)
- [Navigation Elements](#navigation-elements)
- [Styling and Appearance](#styling-and-appearance)
- [Data Binding](#data-binding)
- [Events and Handlers](#events-and-handlers)
- [Refreshable UI](#refreshable-ui)
- [Storage](#storage)
- [Background Tasks](#background-tasks)
- [JavaScript Interop](#javascript-interop)
- [Navigation API](#navigation-api)
- [Notifications](#notifications)
- [Downloads and Clipboard](#downloads-and-clipboard)
- [Timers](#timers)
- [Keyboard](#keyboard)
- [3D Scenes](#3d-scenes)
- [Maps](#maps)
- [Native Desktop Mode](#native-desktop-mode)
- [Deployment](#deployment)
- [Testing](#testing)
- [Element Base Class and Mixins](#element-base-class-and-mixins)

---

## Getting Started

### Minimal App

```python
from nicegui import ui

ui.label('Hello, NiceGUI!')
ui.button('Click me', on_click=lambda: ui.notify('Button clicked!'))

ui.run()
```

This creates a web page at `http://localhost:8080` with a label and a button. NiceGUI supports two modes:

- **Script mode**: UI elements at the top level are shared across all clients.
- **Page mode**: Use `@ui.page('/path')` to create per-client pages.

### Installation

```bash
pip install nicegui
```

---

## Running Your App

### `ui.run()`

Start the NiceGUI server as a standalone application.

```python
ui.run(
    host='0.0.0.0',                  # Bind address ('127.0.0.1' in native mode)
    port=8080,                        # Server port (auto in native mode)
    title='NiceGUI',                  # Browser tab title
    viewport='width=device-width, initial-scale=1',
    favicon=None,                     # File path, URL, or emoji (e.g. '🚀')
    dark=False,                       # True, False, or None (auto)
    language='en-US',                 # Page language
    binding_refresh_interval=0.1,     # Seconds between binding updates (None to disable)
    reconnect_timeout=3.0,            # Seconds before showing disconnect overlay
    message_history_length=1000,      # Messages buffered for reconnect (0 to disable)
    storage_secret=None,              # Required for app.storage.user and .browser
    show=True,                        # Open browser on start (or path string)
    reload=True,                      # Auto-reload on code changes
    native=False,                     # Open in native desktop window
    window_size=(800, 600),           # Activates native mode
    fullscreen=False,                 # Activates native mode
    frameless=False,                  # Activates native mode
    on_air=None,                      # True for random URL, or token string
    tailwind=True,                    # Enable Tailwind CSS
    unocss=None,                      # 'mini', 'wind3', 'wind4' (alternative to Tailwind)
    prod_js=True,                     # Use production JavaScript bundles
    fastapi_docs=False,               # Enable Swagger UI (or config dict)
    endpoint_documentation='none',    # 'none', 'internal', 'page', 'all'
    uvicorn_logging_level='warning',
    uvicorn_reload_dirs='.',
    uvicorn_reload_includes='*.py',
    uvicorn_reload_excludes='.*, .py[cod], .sw.*, ~*',
    ssl_certfile=None,                # Path to SSL certificate
    ssl_keyfile=None,                 # Path to SSL key
    show_welcome_message=True,
)
```

### `ui.run_with()`

Mount NiceGUI onto an existing FastAPI (or Starlette) application.

```python
from fastapi import FastAPI

app = FastAPI()

@app.get('/api/data')
def get_data():
    return {'value': 42}

@ui.page('/')
def index():
    ui.label('Hello from NiceGUI on FastAPI!')

ui.run_with(
    app,
    mount_path='/',          # Where to mount NiceGUI
    title='My App',
    storage_secret='secret',
    # ... same parameters as ui.run() minus host/port/reload
)
```

---

## Pages and Routing

### `@ui.page()`

Create a page function that runs for each client connection.

```python
@ui.page(
    path='/my-page',
    title='Page Title',
    viewport='width=device-width, initial-scale=1',
    favicon=None,
    dark=None,                   # True, False, None (auto), or inherit
    language='en-US',
    response_timeout=3.0,        # Seconds to wait for page build
    reconnect_timeout=None,      # Override global reconnect_timeout
    api_router=None,             # FastAPI APIRouter for grouping
)
def my_page():
    ui.label('This page is unique per client.')
```

### Path Parameters

FastAPI-style path parameters with automatic type conversion:

```python
@ui.page('/user/{user_id}')
def user_page(user_id: int):
    ui.label(f'User ID: {user_id}')

@ui.page('/greet/{name}')
def greet(name: str, greeting: str = 'Hello'):  # greeting is a query param
    ui.label(f'{greeting}, {name}!')
```

### Special Parameter Injection

```python
from fastapi import Request
from nicegui import Client

@ui.page('/info')
def info_page(request: Request, client: Client):
    ui.label(f'IP: {request.client.host}')
    ui.label(f'Client ID: {client.id}')
```

### Wait for Client Connection

Some features (like JavaScript execution) require a connected client:

```python
@ui.page('/connected')
async def page():
    await ui.context.client.connected()
    result = await ui.run_javascript('navigator.userAgent')
    ui.label(f'Browser: {result}')
```

### APIRouter

Group pages with FastAPI's APIRouter:

```python
from fastapi import APIRouter

router = APIRouter(prefix='/admin')

@ui.page('/dashboard', api_router=router)
def dashboard():
    ui.label('Admin Dashboard')
# Accessible at /admin/dashboard
```

### Multicasting

Send updates to all clients on a page:

```python
for client in app.clients('/dashboard'):
    with client:
        ui.notify('Server update!')
```

### Page Layout

Standard page structure with header, drawers, and footer:

```python
@ui.page('/app')
def app_page():
    with ui.header():
        ui.label('My App')
    with ui.left_drawer():
        ui.label('Sidebar')
    ui.label('Main content')
    with ui.footer():
        ui.label('Footer')
```

### Sub-Pages

```python
@ui.page('/docs')
def docs():
    ui.sub_pages()  # Renders matched sub-page content here

@ui.page('/docs/guide')
def guide():
    ui.label('Guide content')
```

### Static and Media Files

```python
from nicegui import app

# Serve a directory of static files
app.add_static_files('/static', 'local/static/dir')

# Serve a single static file (returns URL)
url = app.add_static_file(local_file='report.pdf')

# Media files (supports range requests for streaming)
app.add_media_files('/media', 'local/media/dir')
url = app.add_media_file(local_file='video.mp4')
```

### Custom HTML in Head/Body

```python
ui.add_head_html('<link rel="stylesheet" href="/static/style.css">')
ui.add_body_html('<script>console.log("loaded")</script>')
ui.add_css('.my-class { color: red; }')
ui.add_scss('$color: red; .my-class { color: $color; }')
ui.add_sass('.my-class\n  color: red')
```

---

## Layout and Containers

### `ui.row()`

Horizontal flex container.

```python
with ui.row(wrap=True, align_items=None):
    ui.label('Left')
    ui.label('Right')
```

### `ui.column()`

Vertical flex container.

```python
with ui.column(wrap=False, align_items=None):
    ui.label('Top')
    ui.label('Bottom')
```

### `ui.grid()`

CSS grid layout.

```python
with ui.grid(columns=3):       # or columns='1fr 2fr 1fr'
    for i in range(9):
        ui.label(f'Cell {i}')

with ui.grid(rows=2, columns=2):
    ...
```

### `ui.card()`

Material design card container.

```python
with ui.card(align_items=None):
    ui.label('Card content')

# Tight card (no padding)
with ui.card().tight():
    ui.image('image.png')
    ui.label('Caption')
```

- `ui.card_section()` — Section divider inside a card.
- `ui.card_actions()` — Action area inside a card.

### `ui.expansion()`

Collapsible section.

```python
with ui.expansion(
    text='Click to expand',
    caption=None,
    icon=None,
    group=None,          # Only one in group open at a time
    value=False,         # Initially open/closed
    on_value_change=None,
):
    ui.label('Hidden content')
```

### `ui.dialog()`

Modal dialog (can be awaited for result).

```python
with ui.dialog() as dialog, ui.card():
    ui.label('Are you sure?')
    with ui.row():
        ui.button('Yes', on_click=lambda: dialog.submit('Yes'))
        ui.button('No', on_click=lambda: dialog.submit('No'))

async def ask():
    result = await dialog
    ui.notify(f'You chose: {result}')

ui.button('Open Dialog', on_click=ask)
```

### `ui.tabs()` / `ui.tab_panels()`

Tabbed interface.

```python
with ui.tabs() as tabs:
    one = ui.tab('One')
    two = ui.tab('Two', icon='home')

with ui.tab_panels(tabs, value=one):
    with ui.tab_panel(one):
        ui.label('First tab')
    with ui.tab_panel(two):
        ui.label('Second tab')
```

### `ui.stepper()`

Step-by-step wizard.

```python
with ui.stepper() as stepper:
    with ui.step('Step 1'):
        ui.label('First step')
        with ui.stepper_navigation():
            ui.button('Next', on_click=stepper.next)
    with ui.step('Step 2'):
        ui.label('Second step')
        with ui.stepper_navigation():
            ui.button('Back', on_click=stepper.previous)
            ui.button('Done', on_click=lambda: ui.notify('Done!'))
```

### `ui.carousel()`

Image/content carousel.

```python
with ui.carousel(animated=True, arrows=True, navigation=True):
    with ui.carousel_slide():
        ui.label('Slide 1')
    with ui.carousel_slide():
        ui.label('Slide 2')
```

### `ui.scroll_area()`

Custom scroll container with events.

```python
with ui.scroll_area(on_scroll=lambda e: print(e.vertical_percentage)):
    for i in range(100):
        ui.label(f'Line {i}')
```

### `ui.splitter()`

Resizable split panels.

```python
with ui.splitter(horizontal=False, value=50) as splitter:
    with splitter.before:
        ui.label('Left pane')
    with splitter.after:
        ui.label('Right pane')
```

### `ui.header()` / `ui.footer()`

Page-level header and footer.

```python
with ui.header():
    ui.label('App Header').classes('text-h5')

with ui.footer():
    ui.label('© 2024')
```

### `ui.left_drawer()` / `ui.right_drawer()`

Slide-out navigation drawers.

```python
with ui.left_drawer(value=True, fixed=True, bordered=True):
    ui.label('Navigation')
```

### Other Layout Utilities

| Element | Description |
|---------|-------------|
| `ui.separator()` | Horizontal divider line |
| `ui.space()` | Flexbox spacer (fills available space) |
| `ui.skeleton(type='rect')` | Loading placeholder skeleton |
| `ui.fullscreen()` | Fullscreen container |
| `ui.teleport(to='selector')` | Vue teleport to DOM location |
| `ui.page_sticky(position='bottom-right')` | Fixed position element |
| `ui.list()` | List container for items |
| `ui.item()` | Individual list item |
| `ui.timeline()` | Timeline display |

---

## Text Elements

### `ui.label()`

Simple text display.

```python
ui.label('Hello World')
label = ui.label()
label.set_text('Updated text')
label.bind_text_from(obj, 'property')
```

### `ui.link()`

Hyperlink to URL, page function, or element.

```python
ui.link('NiceGUI', 'https://nicegui.io', new_tab=True)
ui.link('Go to page', target=my_page_function)
ui.link('Jump to section', target=some_element)
```

### `ui.markdown()`

Render Markdown content.

```python
ui.markdown(
    content='# Hello\n**Bold** text',
    extras=['fenced-code-blocks', 'tables'],
    sanitize=True,  # True, False, or custom callable
)
```

### `ui.html()`

Render raw HTML.

```python
ui.html(
    content='<strong>Bold</strong>',
    sanitize=True,   # True, False, or custom callable
    tag='div',
)
```

### `ui.code()`

Syntax-highlighted code block with copy button.

```python
ui.code('print("hello")', language='python')
```

### `ui.mermaid()`

Render Mermaid diagrams.

```python
ui.mermaid('''
graph LR
    A --> B --> C
''', on_node_click=lambda e: print(e))
```

### `ui.restructured_text()`

Render reStructuredText.

```python
ui.restructured_text('**Bold** and *italic*')
```

### `ui.chat_message()`

Chat-style message bubble.

```python
ui.chat_message(
    text='Hello!',
    name='User',
    stamp='10:30 AM',
    avatar='https://example.com/avatar.png',
    side='left',    # 'left' or 'right'
)
```

---

## Input Controls

### `ui.input()`

Text input field.

```python
ui.input(
    label='Username',
    placeholder='Enter name...',
    value='',
    password=False,
    password_toggle_button=False,
    prefix=None,
    suffix=None,
    on_change=None,
    autocomplete=['option1', 'option2'],
    validation={'Too short': lambda v: len(v) >= 3},
)
```

Validation can be a dict mapping error messages to boolean functions, or a single callable returning an optional error string:

```python
ui.input(validation=lambda v: 'Too short' if len(v) < 3 else None)
```

### `ui.textarea()`

Multi-line text input.

```python
ui.textarea(
    label='Description',
    placeholder='Enter text...',
    value='',
    on_change=None,
    validation=None,
)
```

### `ui.number()`

Numeric input.

```python
ui.number(
    label='Age',
    placeholder=None,
    value=None,
    min=0,
    max=150,
    precision=0,      # Decimal places
    step=1,
    prefix=None,
    suffix=None,
    format=None,       # Display format string
    on_change=None,
    validation=None,
)
```

### `ui.select()`

Dropdown selection.

```python
# Simple list
ui.select(['Option A', 'Option B'], value='Option A')

# Dict mapping values to labels
ui.select({1: 'One', 2: 'Two', 3: 'Three'}, value=1)

# With search/filter
ui.select(options, with_input=True)

# Multi-select
ui.select(options, multiple=True, value=[])

# Allow new values
ui.select(options, new_value_mode='add-unique')

# Full signature
ui.select(
    options=[],
    label=None,
    value=None,
    on_change=None,
    with_input=False,
    new_value_mode=None,    # 'add', 'add-unique', 'toggle'
    multiple=False,
    clearable=False,
    validation=None,
    key_generator=None,     # For generating keys for new items
)
```

### `ui.checkbox()`

```python
ui.checkbox('Accept terms', value=False, on_change=None)
```

### `ui.switch()`

```python
ui.switch('Enable feature', value=False, on_change=None)
```

### `ui.toggle()`

Button toggle group.

```python
ui.toggle(
    options=['A', 'B', 'C'],    # List or dict
    value=None,
    on_change=None,
    clearable=False,
)
```

### `ui.radio()`

Radio button group.

```python
ui.radio(
    options=['Small', 'Medium', 'Large'],
    value=None,
    on_change=None,
)
```

### `ui.slider()`

Range slider.

```python
ui.slider(
    min=0,
    max=100,
    step=1,
    value=50,
    on_change=None,
)
```

### `ui.range()`

Dual-handle range selector.

```python
ui.range(
    min=0,
    max=100,
    step=1,
    value={'min': 20, 'max': 80},
    on_change=None,
)
```

### `ui.knob()`

Rotary knob.

```python
ui.knob(
    value=0.5,
    min=0.0,
    max=1.0,
    step=0.01,
    color='primary',
    center_color=None,
    track_color=None,
    size=None,
    show_value=False,
    on_change=None,
)
```

### `ui.rating()`

Star/icon rating.

```python
ui.rating(
    value=3,
    max=5,
    icon=None,              # Custom icon name
    icon_selected=None,
    icon_half=None,
    color='primary',
    size=None,
    on_change=None,
)
```

### `ui.joystick()`

Virtual joystick input.

```python
ui.joystick(
    on_start=None,
    on_move=None,
    on_end=None,
    throttle=0.05,
)
```

### `ui.color_input()`

Color input with picker popup.

```python
ui.color_input(
    label='Color',
    placeholder=None,
    value='#ff0000',
    on_change=None,
    preview=False,         # Show color preview swatch
)
```

### `ui.color_picker()`

Standalone color picker menu.

```python
with ui.button(icon='palette') as button:
    ui.color_picker(on_pick=lambda e: button.style(f'background-color: {e.color}'))
```

### `ui.date()` / `ui.date_input()`

Date picker and input.

```python
# Standalone calendar picker
ui.date(
    value='2024-01-15',
    mask='YYYY-MM-DD',
    on_change=None,
)

# Input field with date picker popup
ui.date_input(
    label='Date',
    range_input=False,
    placeholder=None,
    value='',
    on_change=None,
)
```

### `ui.time()` / `ui.time_input()`

Time picker and input.

```python
ui.time(value='12:30', mask='HH:mm', on_change=None)

ui.time_input(
    label='Time',
    placeholder=None,
    value='',
    on_change=None,
)
```

### `ui.upload()`

File upload component.

```python
ui.upload(
    multiple=False,
    max_file_size=None,         # Bytes
    max_total_size=None,
    max_files=None,
    on_begin_upload=None,
    on_upload=None,             # Called per file with UploadEventArguments
    on_multi_upload=None,       # Called once with all files
    on_rejected=None,
    label='Upload',
    auto_upload=False,
)
```

### `ui.input_chips()`

Tag/chip input for entering multiple values.

```python
ui.input_chips(
    label='Tags',
    value=['python', 'nicegui'],
    on_change=None,
    new_value_mode='toggle',
    clearable=False,
    validation=None,
)
```

### `ui.pagination()`

Page navigation.

```python
ui.pagination(
    min=1,
    max=10,
    direction_links=True,
    value=1,
    on_change=None,
)
```

---

## Buttons and Actions

### `ui.button()`

```python
ui.button(
    text='Click Me',
    on_click=None,
    color='primary',
    icon=None,
)
```

### `ui.button_group()`

Container for grouped buttons.

```python
with ui.button_group():
    ui.button('A', on_click=...)
    ui.button('B', on_click=...)
```

### `ui.button_dropdown()`

Button with a dropdown menu.

```python
with ui.button_dropdown(
    text='Menu',
    value=False,
    on_value_change=None,
    on_click=None,
    color='primary',
    icon=None,
    auto_close=False,
    split=False,          # Separate click from dropdown
):
    ui.item('Option 1', on_click=...)
    ui.item('Option 2', on_click=...)
```

### `ui.fab()` / `ui.fab_action()`

Floating Action Button.

```python
with ui.fab(icon='add', color='primary', direction='right'):
    ui.fab_action(icon='edit', on_click=...)
    ui.fab_action(icon='delete', on_click=...)
```

### `ui.badge()`

Small badge label.

```python
ui.badge('3', color='red', text_color='white', outline=False)
```

### `ui.chip()`

Selectable/removable chip.

```python
ui.chip(
    text='Python',
    icon='code',
    color='primary',
    text_color=None,
    on_click=None,
    selectable=False,
    selected=False,
    on_selection_change=None,
    removable=False,
    on_value_change=None,
)
```

---

## Data Display

### `ui.table()`

Full-featured data table.

```python
columns = [
    {'name': 'name', 'label': 'Name', 'field': 'name', 'required': True, 'align': 'left', 'sortable': True},
    {'name': 'age', 'label': 'Age', 'field': 'age', 'sortable': True},
]
rows = [
    {'id': 1, 'name': 'Alice', 'age': 30},
    {'id': 2, 'name': 'Bob', 'age': 25},
]

table = ui.table(
    rows=rows,
    columns=columns,
    column_defaults=None,     # Default column options dict
    row_key='id',
    title=None,
    selection=None,           # 'single', 'multiple', or None
    pagination=None,          # int or {'rowsPerPage': 10, 'sortBy': 'name', 'descending': False, 'page': 1}
    on_select=None,
    on_pagination_change=None,
)
```

**Key methods:**
- `table.add_rows(*rows)` / `table.remove_rows(*rows)` — Modify rows.
- `table.update_rows(*rows)` — Update specific rows by key.
- `table.selected` — List of selected row dicts.
- `table.toggle_fullscreen()` / `table.is_fullscreen`
- `await table.get_filtered_sorted_rows()` — Get rows after filter/sort.
- `await table.get_computed_rows()` — Get rows after all processing.

**From DataFrames:**
```python
ui.table.from_pandas(df)
ui.table.from_polars(df)
```

**Scoped Slots (Vue templates):**
```python
table.add_slot('body-cell-name', '''
    <q-td :props="props">
        <strong>{{ props.value }}</strong>
    </q-td>
''')
```

### `ui.aggrid()`

AG Grid integration.

```python
ui.aggrid(
    options={
        'columnDefs': [
            {'headerName': 'Name', 'field': 'name'},
            {'headerName': 'Age', 'field': 'age'},
        ],
        'rowData': [
            {'name': 'Alice', 'age': 30},
        ],
    },
    html_columns=[],
    theme=None,
    auto_size_columns=True,
)
```

**Key methods:**
- `await grid.get_selected_rows()` — Get selected rows.
- `await grid.get_selected_row()` — Get single selected row.
- `grid.update()` — Refresh grid data.

### `ui.tree()`

Hierarchical tree view.

```python
ui.tree(
    nodes=[
        {'id': 'root', 'label': 'Root', 'children': [
            {'id': 'child1', 'label': 'Child 1'},
            {'id': 'child2', 'label': 'Child 2'},
        ]},
    ],
    node_key='id',
    label_key='label',
    children_key='children',
    on_select=None,
    on_expand=None,
    on_tick=None,
    tick_strategy=None,   # 'leaf', 'leaf-filtered', 'strict', 'none'
)
```

### `ui.log()`

Scrollable log output.

```python
log = ui.log(max_lines=100)
log.push('New log entry')
log.clear()
```

### `ui.json_editor()`

Visual JSON editor.

```python
ui.json_editor(
    properties={'key': 'value', 'nested': {'a': 1}},
    on_select=None,
    on_change=None,
    schema=None,
)
```

### `ui.editor()`

Rich text editor (WYSIWYG).

```python
ui.editor(value='<b>Hello</b>', on_change=None)
```

### `ui.code()`

Syntax highlighted code display.

```python
ui.code('def hello():\n    pass', language='python')
```

### `ui.codemirror()`

Full-featured code editor.

```python
ui.codemirror(
    code='',
    language='python',
    on_change=None,
    theme='default',
)
```

---

## Charts and Visualization

### `ui.echart()`

Apache ECharts.

```python
ui.echart(
    options={
        'xAxis': {'type': 'category', 'data': ['A', 'B', 'C']},
        'yAxis': {'type': 'value'},
        'series': [{'data': [10, 20, 30], 'type': 'bar'}],
    },
    on_point_click=None,
    on_click=None,
    enable_3d=False,
    renderer='canvas',
    theme=None,
)
```

### `ui.highchart()`

Highcharts (requires `nicegui[highcharts]`).

```python
ui.highchart(options={...})
```

### `ui.plotly()`

Plotly charts.

```python
import plotly.graph_objects as go
fig = go.Figure(data=go.Bar(x=['A', 'B'], y=[10, 20]))
ui.plotly(fig)
```

### `ui.pyplot()` / `ui.matplotlib()`

Matplotlib figures.

```python
with ui.pyplot(figsize=(6, 4)):
    import matplotlib.pyplot as plt
    plt.plot([1, 2, 3], [1, 4, 9])
    plt.title('My Plot')

# Or from existing figure
ui.matplotlib(fig)
```

### `ui.altair()`

Altair/Vega-Lite charts.

```python
import altair as alt
chart = alt.Chart(data).mark_bar().encode(x='category', y='value')
ui.altair(chart)
```

### `ui.line_plot()`

Real-time line plot.

```python
plot = ui.line_plot(n=2, limit=100, update_every=1)
# Later:
plot.push([x], [[y1], [y2]])
```

### Progress Indicators

```python
# Linear progress bar
ui.linear_progress(value=0.5, show_value=True, color='primary')

# Circular progress
ui.circular_progress(value=0.7, min=0.0, max=1.0, show_value=True)

# Spinner (many types available)
ui.spinner(type='default', size='1em', color='primary')
# Types: 'default', 'audio', 'ball', 'bars', 'box', 'clock', 'comment',
#         'cube', 'dots', 'facebook', 'gears', 'grid', 'hearts', 'hourglass',
#         'infinity', 'ios', 'orbit', 'oval', 'pie', 'puff', 'radio', 'rings',
#         'tail'
```

---

## Media Elements

### `ui.image()`

Display image from URL, file path, base64, or PIL Image.

```python
ui.image('https://example.com/photo.jpg')
ui.image(Path('local/image.png'))
ui.image(pil_image_object)
```

### `ui.interactive_image()`

Image with SVG overlay and mouse events.

```python
ui.interactive_image(
    source='photo.jpg',
    content='<circle cx="100" cy="100" r="10" fill="red" />',   # SVG overlay
    size=None,
    on_mouse=None,
    events=['click'],
    cross=False,         # Show crosshair (True, or color string)
    sanitize=True,
)
```

### `ui.video()`

HTML5 video player.

```python
ui.video(
    src='video.mp4',     # URL, file path
    controls=True,
    autoplay=False,
    muted=False,
    loop=False,
)
```

### `ui.audio()`

HTML5 audio player.

```python
ui.audio(
    src='audio.mp3',
    controls=True,
    autoplay=False,
    muted=False,
    loop=False,
)
```

### `ui.icon()`

Material Design icon.

```python
ui.icon('home', size='2em', color='primary')
```

### `ui.avatar()`

User avatar.

```python
ui.avatar(
    icon='person',
    color='primary',
    text_color='white',
    size=None,
    font_size=None,
    square=False,
    rounded=False,
)
```

### `ui.parallax()`

Parallax scrolling image.

```python
ui.parallax(source='background.jpg', height=500, speed=1.0)
```

---

## Feedback and Overlays

### `ui.notify()`

Toast-style notification.

```python
ui.notify(
    message='Action completed',
    position='bottom',       # 'top-left', 'top-right', 'bottom-left', 'bottom-right',
                             # 'top', 'bottom', 'left', 'right', 'center'
    close_button=False,      # True or custom label string
    type=None,               # 'positive', 'negative', 'warning', 'info', 'ongoing'
    color=None,
    multi_line=False,
    icon=None,
    spinner=False,
    timeout=5.0,             # Seconds (0 for persistent)
)
```

### `ui.notification()`

Notification as a context manager (updatable).

```python
with ui.notification(message='Processing...', spinner=True, timeout=None) as n:
    # Do work...
    n.message = 'Done!'
    n.spinner = False
    n.close_button = 'OK'
```

### `ui.tooltip()`

Show tooltip on hover.

```python
with ui.button('Hover me'):
    ui.tooltip('More info here')
```

### `ui.menu()` / `ui.context_menu()`

```python
with ui.button('Menu'):
    with ui.menu():
        ui.menu_item('Option 1', on_click=...)
        ui.menu_item('Option 2', on_click=...)
        ui.separator()
        ui.menu_item('Option 3', auto_close=True)

# Right-click context menu
with ui.label('Right-click me'):
    with ui.context_menu():
        ui.menu_item('Copy', on_click=...)
        ui.menu_item('Paste', on_click=...)
```

---

## Navigation Elements

### `ui.navigate`

Browser navigation.

```python
ui.navigate.to('/other-page')
ui.navigate.to('https://example.com', new_tab=True)
ui.navigate.to(page_function)            # Navigate to @ui.page function
ui.navigate.back()
ui.navigate.forward()
ui.navigate.reload()

# History API (no page reload)
ui.navigate.history.push('/new-url')
ui.navigate.history.replace('/replacement-url')
```

### `ui.page_title()`

Dynamically set the browser tab title.

```python
ui.page_title('New Title')
```

---

## Styling and Appearance

NiceGUI provides three styling approaches: **Quasar props**, **Tailwind CSS classes**, and **plain CSS**.

### Element Styling Methods

Every element supports these chainable methods:

```python
element.classes('text-xl font-bold text-blue-500')    # Tailwind classes
element.classes(add='mt-4', remove='mb-2')
element.style('color: red; font-size: 20px')          # Inline CSS
element.style(add='border: 1px solid', remove='color: red')
element.props('outlined rounded')                       # Quasar props
element.props(add='dense', remove='outlined')
```

### `ui.colors()`

Set brand/theme colors globally.

```python
ui.colors(
    primary='#1976D2',
    secondary='#26A69A',
    accent='#9C27B0',
    dark='#1D1D1D',
    positive='#21BA45',
    negative='#C10015',
    info='#31CCEC',
    warning='#F2C037',
)
```

### `ui.dark_mode()`

Control dark mode.

```python
dark = ui.dark_mode(value=None)   # None = auto, True = dark, False = light
dark.enable()
dark.disable()
dark.toggle()
dark.auto()
```

### `ui.query()`

Apply styles to existing DOM elements.

```python
ui.query('body').classes('bg-gray-100')
ui.query('.q-page').style('padding: 20px')
```

### CSS Variables

NiceGUI respects these CSS variables:

```css
--nicegui-default-padding: 12px;
--nicegui-default-gap: 8px;
```

### CSS Layers

From lowest to highest priority:
1. `theme` — Base theming
2. `base` — Browser reset
3. `quasar` — Quasar framework
4. `nicegui` — NiceGUI defaults
5. `components` — Custom component styles
6. `utilities` — Utility classes
7. `overrides` — Final overrides
8. `quasar_importants` — Quasar `!important` overrides

### Tailwind CSS

Tailwind is enabled by default. Use standard Tailwind utility classes:

```python
ui.label('Styled').classes('text-2xl font-bold text-blue-600 bg-gray-100 p-4 rounded-lg shadow')
```

### UnoCSS (Alternative)

```python
ui.run(tailwind=False, unocss='wind4')  # 'mini', 'wind3', 'wind4'
```

---

## Data Binding

NiceGUI supports reactive data binding between UI elements and Python objects.

### Two-Way Binding

```python
class Model:
    text = ''
    number = 0

model = Model()

# Two-way binding
ui.input('Name').bind_value(model, 'text')
ui.slider(min=0, max=100).bind_value(model, 'number')
ui.label().bind_text_from(model, 'text')          # One-way: model → label
```

### Binding Variants

Every bindable property has three methods:

```python
element.bind_value(target, 'prop')       # Two-way binding
element.bind_value_to(target, 'prop')    # Element → target (one-way)
element.bind_value_from(target, 'prop')  # Target → element (one-way)
```

### Transformation Functions

```python
ui.label().bind_text_from(model, 'name', backward=lambda n: n.upper())
ui.input().bind_value(model, 'text', forward=str.strip, backward=str.strip)
```

### Bind to Dictionary

```python
data = {'name': 'Alice'}
ui.input('Name').bind_value(data, 'name')
ui.label().bind_text_from(data, 'name')
```

### Bind to Storage

```python
ui.input('Name').bind_value(app.storage.user, 'name')
```

### Bindable Properties

Available on most elements:

| Property | Elements |
|----------|----------|
| `value` | Input, Select, Slider, Checkbox, Switch, etc. |
| `text` | Label, Button, etc. |
| `icon` | Button, Icon, etc. |
| `label` | Input, Select, etc. |
| `content` | Markdown, Html, etc. |
| `visibility` | All elements |
| `enabled` | All disableable elements |

### `BindableProperty` and `@binding.bindable_dataclass`

For maximum performance, use `BindableProperty` instead of relying on active polling:

```python
from nicegui import binding

@binding.bindable_dataclass
class User:
    name: str = ''
    age: int = 0
```

---

## Events and Handlers

### Element Events

All elements support the `.on()` method for listening to DOM events:

```python
ui.button('Click').on('click', lambda: print('clicked'))

# With event arguments
ui.input().on('keydown.enter', lambda e: print(f'Enter pressed'))

# Generic events (all HTML events)
ui.label('Hover me').on('mouseover', lambda: print('hovered'))
```

### Common Event Arguments

```python
# Click events
on_click=lambda e: print(e.sender, e.client)

# Value change events
on_change=lambda e: print(e.value, e.previous_value)

# Key events
on_key=lambda e: print(e.key, e.action, e.modifiers)

# Mouse events (interactive_image)
on_mouse=lambda e: print(e.image_x, e.image_y, e.type)
```

### Async Event Handlers

```python
async def handle_click():
    await asyncio.sleep(1)
    ui.notify('Done!')

ui.button('Async', on_click=handle_click)
```

### Global Events

```python
ui.on('click', handler)          # Any click on the page
ui.on('keydown', handler)        # Any keydown
```

### App Lifecycle Events

```python
from nicegui import app

app.on_startup(handler)          # Server starts (before ui.run returns)
app.on_shutdown(handler)         # Server stops
app.on_connect(handler)          # Client connects (optional Client param)
app.on_disconnect(handler)       # Client disconnects
app.on_delete(handler)           # Client is deleted/cleaned up
```

### Exception Handling

```python
# Global exception handler
app.on_exception(lambda e: print(f'Error: {e}'))

# Custom error page (before HTML sent)
@app.on_page_exception(Exception)
def handle_error(exception):
    ui.label(f'Error: {exception}').classes('text-red')

# In-page exception handler (after HTML sent)
ui.on_exception(lambda e: ui.notify(str(e), type='negative'))
```

### Custom Event System

```python
from nicegui import Event

event = Event()
event.subscribe(my_callback)
event.emit(data)                  # Fire-and-forget
await event.call(data)            # Wait for all handlers
await event.emitted(timeout=10)   # Wait for next emission
event.unsubscribe(my_callback)
```

---

## Refreshable UI

### `@ui.refreshable`

Decorator for re-renderable UI sections.

```python
items = ['Apple', 'Banana']

@ui.refreshable
def item_list():
    for item in items:
        ui.label(item)

item_list()                          # Initial render
ui.button('Refresh', on_click=item_list.refresh)
```

### With Parameters

```python
@ui.refreshable
def show_data(filter_text=''):
    for item in items:
        if filter_text in item:
            ui.label(item)

show_data()
ui.button('Show A', on_click=lambda: show_data.refresh(filter_text='A'))
```

### `@ui.refreshable_method`

For class-based components with independent instances:

```python
class MyComponent:
    def __init__(self, title):
        self.title = title

    @ui.refreshable_method
    def build(self):
        ui.label(self.title)
        ui.button('Refresh', on_click=self.build.refresh)
```

### `ui.state()`

Reactive state that auto-triggers refresh:

```python
@ui.refreshable
def counter():
    count, set_count = ui.state(0)
    ui.label(f'Count: {count}')
    ui.button('Increment', on_click=lambda: set_count(count + 1))

counter()
```

---

## Storage

NiceGUI provides five storage scopes. Set `storage_secret` in `ui.run()` for `user` and `browser` storage.

### `app.storage.general`

Server-side, shared by all users. Persists across restarts.

```python
app.storage.general['visit_count'] = app.storage.general.get('visit_count', 0) + 1
```

### `app.storage.user`

Server-side, per-user (identified by session cookie). Persists across restarts.

```python
@ui.page('/')
def index():
    app.storage.user['visits'] = app.storage.user.get('visits', 0) + 1
    ui.label(f'Visits: {app.storage.user["visits"]}')
```

### `app.storage.browser`

Client-side, stored in encrypted browser cookie. Shared across tabs. Can only be modified before the initial response.

```python
@ui.page('/')
def index():
    app.storage.browser['theme'] = 'dark'
```

### `app.storage.tab`

Server-side in-memory, unique per browser tab. Survives page navigation within the tab.

```python
@ui.page('/')
async def index():
    await ui.context.client.connected()  # Required for tab storage
    app.storage.tab['search'] = ''
```

### `app.storage.client`

Server-side in-memory, unique per connection. Lost on page reload or navigation.

```python
@ui.page('/')
def index():
    app.storage.client['temp_data'] = compute_expensive_thing()
```

### Storage Configuration

```python
from nicegui.storage import Storage

Storage.path = '.nicegui'                      # Local file path (env: NICEGUI_STORAGE_PATH)
Storage.redis_url = None                       # Redis URL (env: NICEGUI_REDIS_URL)
Storage.redis_key_prefix = 'nicegui:'          # Redis key prefix (env: NICEGUI_REDIS_KEY_PREFIX)
Storage.max_tab_storage_age = 30 * 86400       # Tab storage TTL (30 days default)
```

---

## Background Tasks

### `run.cpu_bound()`

Run CPU-intensive work in a separate process (avoids blocking the event loop).

```python
from nicegui import run

async def handle():
    result = await run.cpu_bound(heavy_computation, arg1, arg2)
    ui.label(f'Result: {result}')
```

**Note:** Functions must be picklable (free functions with simple parameters).

### `run.io_bound()`

Run blocking I/O in a thread pool.

```python
result = await run.io_bound(requests.get, 'https://api.example.com/data')
```

### Background Task Creation

```python
from nicegui import background_tasks

background_tasks.create(my_coroutine(), name='my-task')
```

### Await on Shutdown

```python
@background_tasks.await_on_shutdown
async def important_cleanup():
    await save_data()
```

---

## JavaScript Interop

### `ui.run_javascript()`

Execute JavaScript in the browser.

```python
# Fire and forget
ui.run_javascript('alert("Hello!")')

# Await result
result = await ui.run_javascript('navigator.userAgent', timeout=1.0)

# Access Vue element
await ui.run_javascript(f'getElement({element.id}).innerText')
```

### Element-Level JavaScript

```python
# Run method on Quasar component
await element.run_method('scrollTo', 5)

# Get element's HTML element
await ui.run_javascript(f'getHtmlElement({element.id}).getBoundingClientRect()')
```

---

## Notifications

See [Feedback and Overlays > `ui.notify()`](#uinotify) above.

---

## Downloads and Clipboard

### `ui.download()`

Trigger a file download in the browser.

```python
# Download from file path
ui.download('report.pdf')
ui.download.file('report.pdf', filename='my-report.pdf')

# Download from URL
ui.download.from_url('https://example.com/file.zip')

# Download generated content
ui.download.content(b'Hello World', filename='hello.txt', media_type='text/plain')
```

### `ui.clipboard`

Read/write browser clipboard (requires HTTPS or localhost).

```python
ui.clipboard.write('Copied text!')
text = await ui.clipboard.read()
image = await ui.clipboard.read_image()   # Returns PIL Image (requires Pillow)
```

---

## Timers

### `ui.timer()`

Periodic or one-shot async timer.

```python
ui.timer(
    interval=1.0,       # Seconds between callbacks
    callback=my_func,   # Sync or async callable
    active=True,        # Bindable: start/stop dynamically
    once=False,         # Execute only once after delay
    immediate=True,     # Run callback immediately (ignored if once=True)
)
```

**Methods:**
- `timer.activate()` / `timer.deactivate()` — Start/stop.
- `timer.cancel()` — Stop permanently.

**Bindable properties:** `interval`, `active`.

```python
# Auto-updating clock
ui.label().bind_text_from(globals(), 'current_time')
ui.timer(1.0, lambda: globals().__setitem__('current_time', datetime.now().strftime('%H:%M:%S')))
```

---

## Keyboard

### `ui.keyboard()`

Listen for keyboard events.

```python
ui.keyboard(
    on_key=lambda e: print(e.key, e.action),   # KeyEventArguments
    active=True,
    repeating=True,
    ignore=['input', 'select', 'button', 'textarea'],
)
```

**KeyEventArguments:**
- `e.key` — Key name (e.g., `'Enter'`, `'a'`, `'ArrowUp'`)
- `e.action` — `KeyAction` (`.keydown`, `.keyup`, `.keypress`)
- `e.modifiers` — `KeyboardModifiers` (`.alt`, `.ctrl`, `.meta`, `.shift`)

---

## 3D Scenes

### `ui.scene()`

Three.js 3D scene.

```python
with ui.scene(
    width=400,
    height=300,
    grid=True,
    on_click=None,
    drag_constraints='',
    background_color='#eee',
    control_type='orbit',       # 'orbit', 'fly', 'trackball'
    fps=20,
) as scene:
    scene.sphere(0.5).move(0, 0, 1).material('#ff0000')
    scene.box(1, 1, 1).move(2, 0, 0.5)
    scene.cylinder(0.3, 0.3, 1).move(-2, 0, 0.5)
    scene.extrusion([[0, 0], [1, 0], [0.5, 1]], 0.5)
    scene.text('Hello', style='font-size: 0.5em')
    scene.point_cloud(points, colors, size=0.1)
    scene.line([0, 0, 0], [1, 1, 1])
    scene.curve([p1, p2, p3], num_points=20)
    with scene.group() as group:
        scene.sphere(0.2)
        group.move(1, 0, 0)
```

**Camera control:**
```python
scene.move_camera(x=0, y=-3, z=5, look_at_x=0, look_at_y=0, look_at_z=0)
```

---

## Maps

### `ui.leaflet()`

Interactive Leaflet map.

```python
m = ui.leaflet(
    center=(51.505, -0.09),
    zoom=13,
    options={},
    draw_control=False,
    hide_drawn_items=False,
)

m.tile_layer(url_template='https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png')
m.marker(latlng=(51.505, -0.09))
m.generic_layer(name='L.circle', args=[[51.505, -0.09], {'radius': 200}])
```

---

## Native Desktop Mode

Open your app in a native window using [pywebview](https://pywebview.flowrl.com/).

```python
ui.run(native=True, window_size=(1024, 768))
# or
ui.run(native=True, fullscreen=True)
# or
ui.run(native=True, frameless=True)
```

### Native Configuration

```python
from nicegui import app

app.native.window_args['resizable'] = True
app.native.start_args['debug'] = False
```

**Important:** Native config must be set *outside* the `if __name__ == '__main__':` guard.

### Native Window Events

```python
app.native.on('shown', lambda: print('Window shown'))
app.native.on('resized', lambda e: print(f'Size: {e.width}x{e.height}'))
# Events: 'shown', 'loaded', 'minimized', 'maximized', 'restored',
#          'resized', 'moved', 'closed', 'drop'
```

---

## Deployment

### Docker

```dockerfile
FROM zauberzeug/nicegui:latest

COPY . /app
RUN pip install -r requirements.txt

CMD ["python", "main.py"]
```

Or from scratch:

```dockerfile
FROM python:3.12-slim
RUN pip install nicegui
COPY . /app
WORKDIR /app
EXPOSE 8080
CMD ["python", "main.py"]
```

### Docker Compose

```yaml
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - NICEGUI_STORAGE_PATH=/data/.nicegui
    volumes:
      - app_data:/data
```

### HTTPS / SSL

```python
ui.run(ssl_certfile='cert.pem', ssl_keyfile='key.pem')
```

Or use a reverse proxy (Traefik, NGINX, Caddy) for SSL termination.

### PyInstaller Packaging

```bash
pip install nicegui[native]    # Includes pywebview
nicegui-pack main.py           # Creates standalone executable
```

```python
from multiprocessing import freeze_support

# Native config BEFORE freeze_support
app.native.window_args['resizable'] = True

if __name__ == '__main__':
    freeze_support()
    ui.run(native=True, reload=False)
```

### NiceGUI On Air

Share your local app via a public URL:

```python
ui.run(on_air=True)               # Random URL, valid ~1 hour
ui.run(on_air='your-token')       # Fixed URL with your token
```

---

## Testing

NiceGUI provides two testing fixtures for pytest.

### Screen Fixture (Real Browser)

Uses a real headless browser (Selenium). Slower but tests actual rendering.

```python
from nicegui.testing import Screen

def test_label(screen: Screen):
    ui.label('Hello')
    screen.open('/')
    screen.should_contain('Hello')
    screen.click('Button Text')
    screen.wait(0.5)
```

### User Fixture (Fast Simulation)

Simulates a user without a real browser. Much faster.

```python
from nicegui.testing import User

async def test_button(user: User):
    @ui.page('/')
    def page():
        ui.button('Click', on_click=lambda: ui.label('Clicked'))

    await user.open('/')
    user.find(ui.button).click()
    await user.should_see('Clicked')
```

### Test Configuration

```python
# conftest.py
import pytest
from nicegui.testing import Screen

@pytest.fixture
def screen(request):
    return Screen(request)
```

---

## Element Base Class and Mixins

All NiceGUI elements inherit from the `Element` base class. Understanding the base API helps with advanced usage.

### Base Element

```python
element = ui.element('div')   # Generic HTML element
```

**Key properties:**
- `element.id` — Unique ID within client.
- `element.client` — The `Client` this element belongs to.
- `element.tag` — HTML tag name.
- `element.parent_slot` — Parent slot (or `None` for root).
- `element.visible` — Visibility (bindable).
- `element.slots` — Dict of named slots.

**Styling methods (all chainable):**
```python
element.classes('text-xl font-bold')
element.classes(add='mt-4', remove='mb-2')
element.style('color: red')
element.style(add='border: 1px solid')
element.props('outlined dense')
element.props(add='rounded')
```

**Content methods:**
```python
element.set_text('Hello')
element.text                    # Get current text
element.set_visibility(True)
element.visible = False
element.tooltip('Hint text')
```

**Event methods:**
```python
element.on('click', handler, args=['value'], throttle=0.1)
element.on('mouseover', handler)
```

**DOM methods:**
```python
element.move(target_container)
element.remove(child_element)
element.delete()
element.clear()                 # Remove all children
```

**Context manager:**
```python
with ui.card() as card:
    ui.label('Inside card')    # Automatically added as child
```

### Element Mixins

Mixins provide common behavior patterns:

#### ValueElement
For elements with a `value` property (inputs, selects, sliders, etc.):
```python
element.bind_value(target, 'prop')
element.bind_value_to(target, 'prop', forward=transform)
element.bind_value_from(target, 'prop', backward=transform)
element.set_value(new_value)
element.on_value_change(callback)
```

#### TextElement
For elements with text content:
```python
element.bind_text(target, 'prop')
element.bind_text_from(target, 'prop')
element.set_text('new text')
```

#### DisableableElement
For elements that can be disabled:
```python
element.enable()
element.disable()
element.set_enabled(True)
element.bind_enabled(target, 'prop')
```

#### ValidationElement
For input elements with validation:
```python
# Dict-based: {error_message: validator_function}
ui.input(validation={'Required': lambda v: bool(v)})

# Function-based: returns error string or None
ui.input(validation=lambda v: 'Required' if not v else None)

# Async validation supported
async def validate(v):
    await check_server(v)
    return None

element.validate()                # Trigger validation manually
element.without_auto_validation() # Disable automatic validation
element.error                     # Current error message or None
```

#### Visibility Mixin
```python
element.bind_visibility_from(model, 'show_flag')
element.bind_visibility(model, 'show_flag', value='specific_value')  # Show only when prop == value
```

---

## App Object Reference

### `app` Properties and Methods

```python
from nicegui import app

# State
app.is_started          # bool
app.is_starting         # bool
app.is_stopping         # bool
app.is_stopped          # bool

# URLs (available after startup)
app.urls                # List of server URLs
app.urls.on_change(cb)  # Callback when URLs change

# Configuration
app.config.title = 'My App'
app.config.dark = True
app.config.binding_refresh_interval = 0.1

# Lifecycle hooks
app.on_startup(handler)
app.on_shutdown(handler)
app.on_connect(handler)        # Optional Client param
app.on_disconnect(handler)     # Optional Client param
app.on_delete(handler)         # Optional Client param
app.on_exception(handler)      # Optional Exception param

# Custom error pages
@app.on_page_exception(ValueError)
def handle_value_error(exc):
    ui.label(f'Bad value: {exc}')

# Theming
app.colors(primary='#1976D2', secondary='#26A69A')

# Client iteration
for client in app.clients('/page'):
    with client:
        ui.notify('Update!')

# Storage
app.storage.general     # Shared persistent dict
app.storage.user        # Per-user persistent dict
app.storage.browser     # Browser cookie dict
app.storage.tab         # Per-tab volatile dict
app.storage.client      # Per-connection volatile dict

# Static files
app.add_static_files('/static', 'local/dir')
url = app.add_static_file('file.pdf')
app.add_media_files('/media', 'media/dir')
url = app.add_media_file('video.mp4')

# Shutdown
app.shutdown()

# Native mode
app.native.window_args['resizable'] = True
app.native.start_args['debug'] = False
app.native.on('shown', callback)
```

---

## Environment Variables

| Variable | Purpose |
|----------|---------|
| `NICEGUI_STORAGE_PATH` | Local file storage directory (default: `.nicegui`) |
| `NICEGUI_REDIS_URL` | Redis URL for shared storage across instances |
| `NICEGUI_REDIS_KEY_PREFIX` | Redis key prefix (default: `nicegui:`) |
| `MATPLOTLIB` | Matplotlib backend setting |
| `MARKDOWN_CONTENT_CACHE_SIZE` | Cache size for Markdown rendering |
| `RST_CONTENT_CACHE_SIZE` | Cache size for reStructuredText rendering |

---

## Quick Patterns

### Login Page with Storage

```python
@ui.page('/login')
def login():
    def try_login():
        if username.value == 'admin' and password.value == 'secret':
            app.storage.user['authenticated'] = True
            ui.navigate.to('/')
        else:
            ui.notify('Wrong credentials', type='negative')

    username = ui.input('Username')
    password = ui.input('Password', password=True)
    ui.button('Login', on_click=try_login)

@ui.page('/')
def index():
    if not app.storage.user.get('authenticated'):
        return ui.navigate.to('/login')
    ui.label('Welcome!')
```

### Auto-Refreshing Dashboard

```python
@ui.page('/')
def dashboard():
    @ui.refreshable
    def stats():
        data = fetch_stats()
        ui.label(f'Users: {data["users"]}')
        ui.label(f'Revenue: ${data["revenue"]}')

    stats()
    ui.timer(30, stats.refresh)
```

### Chat Application

```python
messages = []

@ui.page('/')
def chat():
    @ui.refreshable
    def message_list():
        for msg in messages:
            ui.chat_message(text=msg['text'], name=msg['name'], stamp=msg['time'])

    message_list()

    with ui.row().classes('w-full'):
        inp = ui.input(placeholder='Type a message...').classes('flex-grow')
        ui.button('Send', on_click=lambda: (
            messages.append({'text': inp.value, 'name': 'User', 'time': datetime.now().strftime('%H:%M')}),
            message_list.refresh(),
            inp.set_value(''),
        ))
```

### File Upload with Preview

```python
@ui.page('/')
def upload_page():
    def handle_upload(e):
        content = e.content.read()
        ui.image(f'data:image/png;base64,{base64.b64encode(content).decode()}')
        ui.notify(f'Uploaded: {e.name}')

    ui.upload(on_upload=handle_upload, auto_upload=True).props('accept=".png,.jpg"')
```

### Custom FastAPI + NiceGUI

```python
from fastapi import FastAPI
from nicegui import ui

app = FastAPI()

@app.get('/api/health')
def health():
    return {'status': 'ok'}

@ui.page('/')
def index():
    async def check():
        import httpx
        async with httpx.AsyncClient() as client:
            r = await client.get(f'{app.urls[0]}/api/health')
            ui.notify(f'API: {r.json()["status"]}')

    ui.button('Check API', on_click=check)

ui.run_with(app)
```

---

## Complete Element List

Below is every UI element available in NiceGUI, organized alphabetically:

| Element | Description |
|---------|-------------|
| `ui.aggrid()` | AG Grid data table |
| `ui.altair()` | Altair/Vega chart |
| `ui.anywidget()` | anywidget wrapper |
| `ui.audio()` | HTML5 audio player |
| `ui.avatar()` | User avatar icon |
| `ui.badge()` | Small badge label |
| `ui.button()` | Clickable button |
| `ui.button_dropdown()` | Button with dropdown menu |
| `ui.button_group()` | Container for grouped buttons |
| `ui.card()` | Material card container |
| `ui.card_actions()` | Card action area |
| `ui.card_section()` | Card section divider |
| `ui.carousel()` | Content carousel |
| `ui.carousel_slide()` | Carousel slide |
| `ui.chat_message()` | Chat bubble message |
| `ui.checkbox()` | Boolean checkbox |
| `ui.chip()` | Chip/tag element |
| `ui.circular_progress()` | Circular progress indicator |
| `ui.code()` | Syntax highlighted code |
| `ui.codemirror()` | Code editor (CodeMirror) |
| `ui.color_input()` | Color input with picker |
| `ui.color_picker()` | Color picker menu |
| `ui.colors()` | Set theme colors |
| `ui.column()` | Vertical flex container |
| `ui.context_menu()` | Right-click menu |
| `ui.dark_mode()` | Dark mode control |
| `ui.date()` | Date calendar picker |
| `ui.date_input()` | Date input with picker |
| `ui.dialog()` | Modal dialog |
| `ui.echart()` | Apache ECharts |
| `ui.editor()` | Rich text editor |
| `ui.element()` | Generic HTML element |
| `ui.expansion()` | Collapsible section |
| `ui.fab()` | Floating action button |
| `ui.fab_action()` | FAB action item |
| `ui.footer()` | Page footer |
| `ui.fullscreen()` | Fullscreen container |
| `ui.grid()` | CSS grid layout |
| `ui.header()` | Page header |
| `ui.highchart()` | Highcharts visualization |
| `ui.html()` | Raw HTML content |
| `ui.icon()` | Material icon |
| `ui.image()` | Image display |
| `ui.input()` | Text input field |
| `ui.input_chips()` | Tag/chip input |
| `ui.interactive_image()` | Image with SVG overlay |
| `ui.joystick()` | Virtual joystick |
| `ui.json_editor()` | Visual JSON editor |
| `ui.keyboard()` | Keyboard listener |
| `ui.knob()` | Rotary knob |
| `ui.label()` | Text display |
| `ui.leaflet()` | Interactive map |
| `ui.left_drawer()` | Left slide drawer |
| `ui.line_plot()` | Real-time line chart |
| `ui.linear_progress()` | Progress bar |
| `ui.link()` | Hyperlink |
| `ui.list()` | List container |
| `ui.log()` | Scrollable log |
| `ui.markdown()` | Markdown content |
| `ui.matplotlib()` | Matplotlib figure |
| `ui.menu()` | Popup menu |
| `ui.menu_item()` | Menu item |
| `ui.mermaid()` | Mermaid diagram |
| `ui.notification()` | Updatable notification |
| `ui.notify()` | Toast notification |
| `ui.number()` | Number input |
| `ui.page_title()` | Dynamic page title |
| `ui.pagination()` | Page navigation |
| `ui.parallax()` | Parallax image |
| `ui.plotly()` | Plotly chart |
| `ui.pyplot()` | Matplotlib plot |
| `ui.query()` | DOM element selector |
| `ui.radio()` | Radio button group |
| `ui.range()` | Dual-handle range |
| `ui.rating()` | Star rating |
| `ui.restructured_text()` | reStructuredText |
| `ui.right_drawer()` | Right slide drawer |
| `ui.row()` | Horizontal flex container |
| `ui.scene()` | 3D scene (Three.js) |
| `ui.scroll_area()` | Custom scroll container |
| `ui.select()` | Dropdown select |
| `ui.separator()` | Horizontal divider |
| `ui.skeleton()` | Loading placeholder |
| `ui.slider()` | Range slider |
| `ui.space()` | Flexbox spacer |
| `ui.spinner()` | Loading spinner |
| `ui.splitter()` | Resizable split panes |
| `ui.step()` | Stepper step |
| `ui.stepper()` | Step wizard |
| `ui.stepper_navigation()` | Step navigation |
| `ui.sub_pages()` | Sub-page routing |
| `ui.switch()` | Toggle switch |
| `ui.tab()` | Tab header |
| `ui.tab_panel()` | Tab content panel |
| `ui.tab_panels()` | Tab panel container |
| `ui.tabs()` | Tab container |
| `ui.table()` | Data table |
| `ui.teleport()` | Vue teleport |
| `ui.textarea()` | Multi-line text input |
| `ui.time()` | Time picker |
| `ui.time_input()` | Time input with picker |
| `ui.timeline()` | Timeline display |
| `ui.timer()` | Async timer |
| `ui.toggle()` | Button toggle |
| `ui.tooltip()` | Hover tooltip |
| `ui.tree()` | Hierarchical tree |
| `ui.upload()` | File upload |
| `ui.video()` | HTML5 video player |
| `ui.xterm()` | Terminal emulator |

---

*Generated from the NiceGUI source code. For the latest documentation, visit [nicegui.io](https://nicegui.io).*
