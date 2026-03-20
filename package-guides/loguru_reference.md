# Loguru Comprehensive Reference

> Version: 0.7.3 | License: MIT | Python: 3.5+ and PyPy

Loguru is a Python logging library that aims to bring enjoyable logging. It provides a single
pre-configured `logger` object with sensible defaults — no Handler, no Formatter, no Filter
boilerplate.

```python
from loguru import logger
```

That's the only import you'll ever need. The `logger` is a singleton instance of the `Logger` class
with a default handler writing to `sys.stderr`.

---

## Table of Contents

- [Quick Start](#quick-start)
- [The Logger Object](#the-logger-object)
- [Logging Methods](#logging-methods)
- [Adding Sinks (Handlers)](#adding-sinks-handlers)
  - [`add()` — Full Signature](#add--full-signature)
  - [Sink Types](#sink-types)
  - [Common Parameters](#common-parameters)
  - [File-Specific Parameters](#file-specific-parameters)
  - [Async-Specific Parameters](#async-specific-parameters)
- [Removing Handlers](#removing-handlers)
- [Log Levels](#log-levels)
  - [Built-in Levels](#built-in-levels)
  - [Custom Levels](#custom-levels)
- [Format Strings](#format-strings)
  - [Record Fields](#record-fields)
  - [The Record Dict](#the-record-dict)
  - [The Message Object](#the-message-object)
  - [Color Markup Tags](#color-markup-tags)
  - [Datetime Format Tokens](#datetime-format-tokens)
- [Filtering](#filtering)
- [Context and Binding](#context-and-binding)
  - [`bind()`](#bind)
  - [`contextualize()`](#contextualize)
  - [`patch()`](#patch)
- [Logging Options — `opt()`](#logging-options--opt)
- [Exception Handling — `catch()`](#exception-handling--catch)
- [Configuration — `configure()`](#configuration--configure)
- [Module Activation — `enable()` / `disable()`](#module-activation--enable--disable)
- [File Rotation](#file-rotation)
- [File Retention](#file-retention)
- [File Compression](#file-compression)
- [Serialization (JSON Logging)](#serialization-json-logging)
- [Async / Enqueue / Thread Safety](#async--enqueue--thread-safety)
- [Completing Async Tasks — `complete()`](#completing-async-tasks--complete)
- [Multiprocessing — `reinstall()`](#multiprocessing--reinstall)
- [Parsing Log Files — `parse()`](#parsing-log-files--parse)
- [Environment Variables](#environment-variables)
- [Type Definitions](#type-definitions)
- [Common Patterns and Recipes](#common-patterns-and-recipes)
- [Standard Logging Interop](#standard-logging-interop)

---

## Quick Start

```python
from loguru import logger

# Logs to stderr with default format
logger.info("Hello, world!")

# String formatting with positional/keyword args
logger.info("User {} logged in from {ip}", "Alice", ip="192.168.1.1")

# Remove default handler and add a file sink
logger.remove()
logger.add("app.log", rotation="10 MB", retention="7 days", compression="zip")

# Exception logging
try:
    result = 1 / 0
except ZeroDivisionError:
    logger.exception("Division failed")

# Decorator for auto-catching exceptions
@logger.catch
def risky():
    return 1 / 0
```

---

## The Logger Object

The `logger` object is the sole public interface. It is a pre-instantiated singleton — you never
create `Logger` instances yourself. Every method either logs a message, configures the logger, or
returns a new logger with modified behavior.

Key design principles:
- **One logger, many sinks**: Add multiple outputs with `add()`, each with independent level, format, and filter.
- **Immutable chaining**: Methods like `bind()`, `opt()`, and `patch()` return new logger instances; they do not mutate the original.
- **Thread-safe**: All handler operations use internal locks.

---

## Logging Methods

All logging methods use Python's `str.format()` syntax — **not** `%` formatting and **not**
f-strings (f-strings cause double interpolation).

### Signature

```python
logger.trace(message, *args, **kwargs) -> None
logger.debug(message, *args, **kwargs) -> None
logger.info(message, *args, **kwargs) -> None
logger.success(message, *args, **kwargs) -> None
logger.warning(message, *args, **kwargs) -> None
logger.error(message, *args, **kwargs) -> None
logger.critical(message, *args, **kwargs) -> None
logger.exception(message, *args, **kwargs) -> None  # Logs at ERROR + includes current exception
logger.log(level, message, *args, **kwargs) -> None  # Log at arbitrary level (str or int)
```

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `message` | `str` or any | The message template. If not a `str`, it is converted to `str` and `*args` / `**kwargs` are ignored. |
| `*args` | `Any` | Positional arguments for `str.format()`. |
| `**kwargs` | `Any` | Keyword arguments for `str.format()`. By default, extra `**kwargs` are also captured into `record["extra"]` (see `opt(capture=...)`). |

### Examples

```python
logger.info("Processing item {}", item_id)
logger.debug("User {name} at {ip}", name="Alice", ip="10.0.0.1")
logger.log("CUSTOM_LEVEL", "Custom level message")
logger.log(25, "Anonymous level by number")

# Non-string messages are converted and logged as-is
logger.info({"key": "value"})
logger.info(42)
```

---

## Adding Sinks (Handlers)

### `add()` — Full Signature

```python
logger.add(
    sink,
    *,
    level="DEBUG",
    format="<default>",
    filter=None,
    colorize=None,
    serialize=False,
    backtrace=True,
    diagnose=True,
    enqueue=False,
    context=None,
    catch=True,
    # File sink only:
    rotation=None,
    retention=None,
    compression=None,
    delay=False,
    watch=False,
    mode="a",
    buffering=1,
    encoding="utf8",
    errors=None,
    newline=None,
    closefd=True,
    opener=None,
    # Async sink only:
    loop=None,
) -> int  # Returns handler ID
```

### Sink Types

| Sink Type | Example | Description |
|-----------|---------|-------------|
| **File path** (`str` / `Path`) | `logger.add("app.log")` | Writes to a file. Supports rotation, retention, compression. Path can contain `{time}` placeholder. |
| **File-like object** (`TextIO`) | `logger.add(sys.stderr)` | Any object with a `.write()` method. |
| **Callable** | `logger.add(lambda msg: ...)` | Called with each `Message` object. |
| **Coroutine function** | `logger.add(async_func)` | Async callable; requires `enqueue=True` or an event `loop`. |
| **`logging.Handler`** | `logger.add(logging.StreamHandler())` | Bridges to the standard `logging` module. |
| **Writable protocol** | `logger.add(custom_obj)` | Any object implementing `write(self, message: Message)`. |

### Common Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `level` | `str \| int` | `"DEBUG"` | Minimum severity level for this sink. Messages below this level are discarded. |
| `format` | `str \| Callable` | *(see default below)* | Format template string or a function `(record) -> str`. |
| `filter` | `str \| Callable \| dict \| None` | `None` | Controls which messages reach this sink. See [Filtering](#filtering). |
| `colorize` | `bool \| None` | `None` | Whether to convert color markup tags to ANSI codes. `None` auto-detects based on whether the sink is a TTY. |
| `serialize` | `bool` | `False` | If `True`, the record is serialized to JSON before being sent to the sink. |
| `backtrace` | `bool` | `True` | Whether to show the full stacktrace (frames beyond the exception raise point). |
| `diagnose` | `bool` | `True` | Whether to show variable values in exception tracebacks. **WARNING: Set to `False` in production — can leak sensitive data.** |
| `enqueue` | `bool` | `False` | If `True`, messages are put into a thread-safe queue and processed in a background thread. Ensures thread/process safety. |
| `context` | `str \| BaseContext \| None` | `None` | Multiprocessing context for the queue when `enqueue=True`. |
| `catch` | `bool` | `True` | Whether to catch and log exceptions that occur inside the sink itself (prevents logging from crashing your app). |

### File-Specific Parameters

These parameters are only valid when `sink` is a file path (`str` or `PathLike`):

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `rotation` | `str \| int \| time \| timedelta \| Callable \| list \| None` | `None` | When to rotate the log file. See [File Rotation](#file-rotation). |
| `retention` | `str \| int \| timedelta \| Callable \| None` | `None` | How long to keep old log files. See [File Retention](#file-retention). |
| `compression` | `str \| Callable \| None` | `None` | How to compress rotated files. See [File Compression](#file-compression). |
| `delay` | `bool` | `False` | If `True`, defer file creation until the first log message is written. |
| `watch` | `bool` | `False` | If `True`, check if the file has been moved/deleted before each write and reopen if needed (useful with `logrotate`). |
| `mode` | `str` | `"a"` | File open mode (`"a"` = append, `"w"` = overwrite). |
| `buffering` | `int` | `1` | Buffering policy (0 = unbuffered, 1 = line-buffered, >1 = buffer size). |
| `encoding` | `str` | `"utf8"` | File encoding. |
| `errors` | `str \| None` | `None` | Encoding error handling mode (e.g., `"strict"`, `"replace"`). |
| `newline` | `str \| None` | `None` | Line ending character(s). |
| `closefd` | `bool` | `True` | Whether to close the file descriptor on removal. |
| `opener` | `Callable[[str, int], int] \| None` | `None` | Custom file opener function (e.g., for setting permissions). |

**Dynamic file paths**: The file path can include `{time}` which is replaced with the current
datetime when the file is created/rotated:

```python
logger.add("logs/{time:YYYY-MM-DD}/app.log", rotation="00:00")
```

### Async-Specific Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `loop` | `AbstractEventLoop \| None` | `None` | Asyncio event loop for coroutine sinks. Deprecated — prefer `enqueue=True`. |

---

## Removing Handlers

```python
logger.remove(handler_id=None) -> None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `handler_id` | `int \| None` | `None` | The ID returned by `add()`. If `None`, **all** handlers are removed. |

```python
# Remove the default stderr handler
logger.remove()

# Remove a specific handler
handler_id = logger.add("app.log")
logger.remove(handler_id)
```

Raises `ValueError` if the handler ID doesn't exist.

---

## Log Levels

### Built-in Levels

| Level | Severity (`no`) | Color | Icon | Description |
|-------|-----------------|-------|------|-------------|
| `TRACE` | 5 | `<cyan><bold>` | Pencil | Granular, step-by-step details |
| `DEBUG` | 10 | `<blue><bold>` | Lady Beetle | Diagnostic information |
| `INFO` | 20 | `<bold>` | Information | General informational messages |
| `SUCCESS` | 25 | `<green><bold>` | Check Mark | Confirmation of successful operations |
| `WARNING` | 30 | `<yellow><bold>` | Warning | Potential issues or unexpected behavior |
| `ERROR` | 40 | `<red><bold>` | Cross Mark | Errors that don't halt execution |
| `CRITICAL` | 50 | `<RED><bold>` | Skull | Severe errors, application may be unable to continue |

### Custom Levels

```python
logger.level(name, no=None, color=None, icon=None) -> Level
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | *(required)* | The level name. |
| `no` | `int \| None` | `None` | Severity number. Required when creating a new level. |
| `color` | `str \| None` | `None` | Color markup string. |
| `icon` | `str \| None` | `None` | Icon string. |

**Returns**: `Level` named tuple: `(name, no, color, icon)`

**Behavior**:
- **Retrieve**: `logger.level("DEBUG")` — returns existing level info.
- **Create**: `logger.level("NOTICE", no=35, color="<yellow>", icon="!")` — creates new level.
- **Update**: `logger.level("DEBUG", icon="DBG")` — updates existing level. Pass only the fields to change.

```python
# Create a custom level
logger.level("NOTICE", no=35, color="<yellow>", icon="!")

# Use it
logger.log("NOTICE", "This is a notice")
```

---

## Format Strings

The default format string is:

```
<green>{time:YYYY-MM-DD HH:mm:ss.SSS Z}</green> | <level>{level: <8}</level> | <cyan>{name}</cyan>:<cyan>{function}</cyan>:<cyan>{line}</cyan> - <level>{message}</level>
```

Format strings use Python's `str.format()` syntax. The format can also be a callable
`(record: Record) -> str` that returns a format template string (the returned string is then
formatted with the record).

### Record Fields

These are the fields available in format strings via `{field_name}`:

| Field | Type | Description |
|-------|------|-------------|
| `{elapsed}` | `timedelta` | Time elapsed since the program started. |
| `{exception}` | `RecordException \| None` | Exception info (type, value, traceback) if an exception was logged. Automatically appended to string formats. |
| `{extra}` | `dict` | User-bound context from `bind()`, `contextualize()`, `patch()`, or `**kwargs`. Access with `{extra[key]}`. |
| `{file}` | `RecordFile` | Source file. Default format is the filename. Access `.name` and `.path` attributes. |
| `{file.name}` | `str` | Source filename (e.g., `"app.py"`). |
| `{file.path}` | `str` | Full file path (e.g., `"/home/user/app.py"`). |
| `{function}` | `str` | Function name where the log call was made. |
| `{level}` | `RecordLevel` | Log level. Default format is the level name. Has `.name`, `.no`, `.icon` attributes. |
| `{level.name}` | `str` | Level name (e.g., `"INFO"`). |
| `{level.no}` | `int` | Level severity number (e.g., `20`). |
| `{level.icon}` | `str` | Level icon. |
| `{line}` | `int` | Line number in source file. |
| `{message}` | `str` | The formatted log message (after `str.format()` with args/kwargs). |
| `{module}` | `str` | Module name (filename without extension). |
| `{name}` | `str \| None` | The `__name__` of the module where the log call was made. |
| `{process}` | `RecordProcess` | Process info. Default format is the process ID. Has `.id` and `.name` attributes. |
| `{process.id}` | `int` | Process ID. |
| `{process.name}` | `str` | Process name. |
| `{thread}` | `RecordThread` | Thread info. Default format is the thread ID. Has `.id` and `.name` attributes. |
| `{thread.id}` | `int` | Thread ID. |
| `{thread.name}` | `str` | Thread name (e.g., `"MainThread"`). |
| `{time}` | `datetime` | Timezone-aware timestamp of the log event. See [Datetime Format Tokens](#datetime-format-tokens). |

### The Record Dict

The `Record` is a `TypedDict` containing all log event data. It is passed to filter functions,
format functions, and patcher functions:

```python
class Record(TypedDict):
    elapsed: timedelta
    exception: Optional[RecordException]
    extra: Dict[Any, Any]
    file: RecordFile
    function: str
    level: RecordLevel
    line: int
    message: str
    module: str
    name: Optional[str]
    process: RecordProcess
    thread: RecordThread
    time: datetime
```

### The Message Object

The `Message` is a `str` subclass with an attached `.record` attribute:

```python
class Message(str):
    record: Record
```

This is what sinks receive. You can access the full record through `message.record`.

### Color Markup Tags

Color tags are used in format strings and log messages (when `colorize=True` or
`opt(colors=True)`). They are converted to ANSI escape codes.

**Foreground colors** (lowercase):

| Tag | Short | Description |
|-----|-------|-------------|
| `<black>` | `<k>` | Black text |
| `<red>` | `<r>` | Red text |
| `<green>` | `<g>` | Green text |
| `<yellow>` | `<y>` | Yellow text |
| `<blue>` | `<e>` | Blue text |
| `<magenta>` | `<m>` | Magenta text |
| `<cyan>` | `<c>` | Cyan text |
| `<white>` | `<w>` | White text |

Light variants: `<light-red>` / `<lr>`, `<light-green>` / `<lg>`, etc.

**Background colors** (UPPERCASE):

| Tag | Short | Description |
|-----|-------|-------------|
| `<BLACK>` | `<K>` | Black background |
| `<RED>` | `<R>` | Red background |
| `<GREEN>` | `<G>` | Green background |
| ... | ... | Same pattern for all colors |

Light variants: `<LIGHT-RED>` / `<LR>`, etc.

**Style tags**:

| Tag | Short | Description |
|-----|-------|-------------|
| `<bold>` | `<b>` | Bold text |
| `<dim>` | `<d>` | Dim text |
| `<italic>` | `<i>` | Italic text |
| `<underline>` | `<u>` | Underlined text |
| `<strike>` | `<s>` | Strikethrough text |
| `<blink>` | `<l>` | Blinking text |
| `<reverse>` | `<v>` | Reversed colors |
| `<hide>` | `<h>` | Hidden text |
| `<normal>` | `<n>` | Normal weight |

**Special tag**:
- `<level>` — Applies the color associated with the current log level.
- `</>` or `</tag>` — Closing tags to end a color/style scope.

Tags can be combined: `<green><bold>text</bold></green>`.

### Datetime Format Tokens

The `{time}` field supports custom formatting via `{time:FORMAT}`. Loguru uses its own token system
(not `strftime` — though `strftime` `%` codes are also supported as a fallback):

| Token | Output | Example |
|-------|--------|---------|
| `YYYY` | 4-digit year | `2024` |
| `YY` | 2-digit year | `24` |
| `Q` | Quarter | `1` to `4` |
| `MMMM` | Full month name | `January` |
| `MMM` | Abbreviated month | `Jan` |
| `MM` | Zero-padded month | `01` to `12` |
| `M` | Month number | `1` to `12` |
| `DDDD` | Zero-padded day of year | `001` to `366` |
| `DDD` | Day of year | `1` to `366` |
| `DD` | Zero-padded day of month | `01` to `31` |
| `D` | Day of month | `1` to `31` |
| `dddd` | Full weekday name | `Monday` |
| `ddd` | Abbreviated weekday | `Mon` |
| `d` | Weekday number | `0` (Mon) to `6` (Sun) |
| `E` | ISO weekday number | `1` (Mon) to `7` (Sun) |
| `HH` | 24h hour, zero-padded | `00` to `23` |
| `H` | 24h hour | `0` to `23` |
| `hh` | 12h hour, zero-padded | `01` to `12` |
| `h` | 12h hour | `1` to `12` |
| `mm` | Minutes, zero-padded | `00` to `59` |
| `m` | Minutes | `0` to `59` |
| `ss` | Seconds, zero-padded | `00` to `59` |
| `s` | Seconds | `0` to `59` |
| `S` | Deciseconds (1/10 sec) | `0` to `9` |
| `SS` | Centiseconds (1/100 sec) | `00` to `99` |
| `SSS` | Milliseconds | `000` to `999` |
| `SSSS` | Ten-microseconds | `0000` to `9999` |
| `SSSSS` | Hundred-microseconds | `00000` to `99999` |
| `SSSSSS` | Microseconds | `000000` to `999999` |
| `A` | AM/PM | `AM` or `PM` |
| `Z` | UTC offset (with colon) | `+02:00` |
| `ZZ` | UTC offset (no colon) | `+0200` |
| `zz` | Timezone name | `UTC`, `EST` |
| `X` | Unix timestamp (seconds) | `1234567890` |
| `x` | Unix timestamp (microseconds) | `1234567890000000` |

**Escaping**: Use square brackets to include literal characters: `[at] HH:mm` produces `at 14:30`.

**UTC conversion**: Append `!UTC` to the end of the format to convert to UTC:
`{time:YYYY-MM-DD HH:mm:ss!UTC}`.

**strftime fallback**: If the format contains `%`, it is treated as a `strftime` format string:
`{time:%Y-%m-%d %H:%M:%S}`.

---

## Filtering

The `filter` parameter in `add()` controls which log messages reach a sink.

### Filter as a string (module name)

```python
# Only messages from "myapp.api" and its submodules
logger.add("api.log", filter="myapp.api")
```

### Filter as a callable

```python
# Only messages with level >= WARNING
logger.add("errors.log", filter=lambda record: record["level"].no >= 30)

# Only messages with a specific extra key
logger.add("special.log", filter=lambda record: record["extra"].get("special"))
```

### Filter as a dict

```python
# Dict maps module names to minimum levels
logger.add("file.log", filter={"myapp.api": "WARNING", "myapp.db": "DEBUG"})

# Use None key for the default level, False to disable
logger.add("file.log", filter={"myapp.api": "WARNING", "": False})
```

---

## Context and Binding

### `bind()`

```python
logger.bind(**kwargs) -> Logger
```

Returns a child logger with extra key-value pairs added to `record["extra"]`. Does not affect the
parent logger.

```python
ctx_logger = logger.bind(user="Alice", request_id="abc-123")
ctx_logger.info("Processing request")
# record["extra"]["user"] == "Alice"
# record["extra"]["request_id"] == "abc-123"

# Use in format strings
logger.add(sys.stderr, format="{extra[user]} | {message}")
```

### `contextualize()`

```python
logger.contextualize(**kwargs) -> ContextManager
```

Context manager (also usable as a decorator) that binds extra values using `contextvars`. This is
**thread-safe** and **async-safe** — each thread/task gets its own copy.

```python
with logger.contextualize(request_id="abc-123"):
    logger.info("Start")    # request_id is in extra
    do_work()               # All log calls within this scope have request_id
    logger.info("Done")     # request_id is in extra
# request_id is gone here
```

### `patch()`

```python
logger.patch(patcher: Callable[[Record], None]) -> Logger
```

Returns a child logger that runs the `patcher` function on every record before it is propagated to
sinks. The patcher modifies the record dict **in place**.

```python
logger = logger.patch(lambda record: record["extra"].update(utc=datetime.utcnow()))
logger.info("Message")  # record["extra"]["utc"] is set

# Multiple patches are applied in order
logger = logger.patch(first_patch).patch(second_patch)
```

---

## Logging Options — `opt()`

```python
logger.opt(
    *,
    exception=None,
    record=False,
    lazy=False,
    colors=False,
    raw=False,
    capture=True,
    depth=0,
    ansi=False,       # Deprecated, use `colors`
) -> Logger
```

Returns a child logger with modified options for the **next** logging call.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `exception` | `bool \| Exception \| ExcInfo \| None` | `None` | If `True`, capture current exception (`sys.exc_info()`). Can also pass an exception instance or `(type, value, traceback)` tuple directly. |
| `record` | `bool` | `False` | If `True`, the format string can access `{record}` as a field (the full record dict). |
| `lazy` | `bool` | `False` | If `True`, positional/keyword args should be callables (zero-arg functions). They are only evaluated if the message will actually be logged (passes level and filter). |
| `colors` | `bool` | `False` | If `True`, color markup tags in the **message** are processed. (Format string tags are always processed based on the `colorize` parameter of the sink.) |
| `raw` | `bool` | `False` | If `True`, the message is sent to the sink **without formatting** — the sink's format string is bypassed. Useful for writing pre-formatted content. |
| `capture` | `bool` | `True` | If `True`, `**kwargs` passed to the logging call are added to `record["extra"]`. Set to `False` to prevent keyword args from polluting extra. |
| `depth` | `int` | `0` | Adjusts the stack depth for determining the caller. Increase this when wrapping loguru in your own utility functions. |
| `ansi` | `bool` | `False` | **Deprecated** — use `colors` instead. |

### Examples

```python
# Lazy evaluation — function is only called if DEBUG level passes
logger.opt(lazy=True).debug("Result: {x}", x=expensive_computation)

# Exception logging outside of except block
try:
    risky()
except Exception as e:
    logger.opt(exception=e).error("Something failed")

# Raw output (bypass format string)
logger.opt(raw=True).info("Pre-formatted line\n")

# Color markup in the message itself
logger.opt(colors=True).info("This is <red>red</red> and <green>green</green>")

# Access the full record in the message
logger.opt(record=True).info("Logged from {record[file]}")

# Adjust stack depth when wrapping loguru
def my_log(msg):
    logger.opt(depth=1).info(msg)  # Caller shows as the function calling my_log()
```

---

## Exception Handling — `catch()`

```python
logger.catch(
    exception=Exception,
    *,
    level="ERROR",
    reraise=False,
    onerror=None,
    exclude=None,
    default=None,
    message="An error has been caught in function '{record[function]}', process '{record[process].name}' ({record[process].id}), thread '{record[thread].name}' ({record[thread].id}):",
) -> Catcher
```

Can be used as a **decorator** or a **context manager**.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `exception` | `type \| tuple[type, ...]` | `Exception` | Exception type(s) to catch. |
| `level` | `str \| int` | `"ERROR"` | Log level for the caught exception. |
| `reraise` | `bool` | `False` | If `True`, re-raise the exception after logging it. |
| `onerror` | `Callable \| None` | `None` | Callback `(exception) -> None` called when an exception is caught. |
| `exclude` | `type \| tuple[type, ...] \| None` | `None` | Exception type(s) to **not** catch (they propagate normally). |
| `default` | `Any` | `None` | Return value of the decorated function if an exception is caught (only when `reraise=False`). |
| `message` | `str` | *(see above)* | Custom log message template. Supports `{record}` placeholders. |

### Examples

```python
# As a decorator
@logger.catch
def process():
    return 1 / 0  # Logged, returns None

# As a decorator with options
@logger.catch(reraise=True, exclude=KeyboardInterrupt)
def server():
    run_forever()

# As a context manager
with logger.catch(message="Batch processing failed"):
    process_batch()

# With default return value
@logger.catch(default=42)
def get_value():
    raise ValueError  # Returns 42 instead
```

---

## Configuration — `configure()`

```python
logger.configure(
    *,
    handlers=None,
    levels=None,
    extra=None,
    patcher=None,
    activation=None,
) -> list[int]
```

Bulk configuration method. Returns a list of handler IDs for the added handlers.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `handlers` | `list[HandlerConfig] \| None` | `None` | List of handler config dicts (same params as `add()`). **If provided, all existing handlers are removed first.** |
| `levels` | `list[LevelConfig] \| None` | `None` | List of level config dicts (same params as `level()`). |
| `extra` | `dict \| None` | `None` | Default extra dict available to all log messages. Merged with (does not replace) existing extra. |
| `patcher` | `Callable \| None` | `None` | Global patcher function applied to all records. |
| `activation` | `list[tuple[str \| None, bool]] \| None` | `None` | List of `(module_name, enabled)` tuples for `enable()`/`disable()`. |

```python
logger.configure(
    handlers=[
        {"sink": sys.stderr, "level": "WARNING"},
        {"sink": "app.log", "level": "DEBUG", "rotation": "10 MB"},
    ],
    levels=[
        {"name": "NOTICE", "no": 35, "color": "<yellow>", "icon": "!"},
    ],
    extra={"app": "myapp", "version": "1.0"},
    activation=[
        ("myapp", True),
        ("third_party_lib", False),
    ],
)
```

---

## Module Activation — `enable()` / `disable()`

```python
logger.disable(name: str | None) -> None
logger.enable(name: str | None) -> None
```

Controls whether log messages from specific modules are processed. Primarily designed for **library
authors** to silence their logs by default and let users opt-in.

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `str \| None` | Module name (matches the module and all submodules). Empty string `""` matches all modules. `None` disables all. |

```python
# Library code (in __init__.py)
from loguru import logger
logger.disable("my_library")

# User code (opt-in to library logs)
from loguru import logger
logger.enable("my_library")
```

Activation rules are evaluated from most-specific to least-specific. More specific module names
take precedence:

```python
logger.disable("myapp")           # Disable all of myapp
logger.enable("myapp.api")        # But enable myapp.api
logger.disable("myapp.api.v2")    # But disable myapp.api.v2
```

---

## File Rotation

The `rotation` parameter in `add()` determines when to rotate (create a new file). Rotated files
are renamed with a timestamp suffix.

### Accepted Values

| Type | Example | Description |
|------|---------|-------------|
| **Size string** | `"100 MB"`, `"1 GB"`, `"500KB"` | Rotate when file exceeds this size. Supports units: B, KB, MB, GB, TB, etc. (with `i` for binary: KiB, MiB, etc.) |
| **int / float** | `1000000` | Size in bytes. |
| **Time string** | `"12:00"`, `"00:00"`, `"13:30:00"` | Rotate daily at the specified time. |
| **Weekday + time** | `"monday at 00:00"`, `"friday"` | Rotate weekly on the specified day (and optional time). |
| **Duration string** | `"1 hour"`, `"7 days"`, `"2 weeks"` | Rotate after this much time has elapsed since file creation. |
| **Frequency string** | `"hourly"`, `"daily"`, `"weekly"`, `"monthly"`, `"yearly"` | Rotate at natural frequency boundaries. |
| `datetime.time` | `time(0, 0)` | Rotate daily at this time. |
| `datetime.timedelta` | `timedelta(hours=6)` | Rotate after this interval. |
| **Callable** | `func(message, file) -> bool` | Custom function returning `True` to trigger rotation. |
| **List** | `["500 MB", "daily"]` | Rotate when **any** condition is met (OR logic). |

### Duration String Units

The duration parser supports: `y`/`years`, `months`, `w`/`weeks`, `d`/`days`, `h`/`hours`,
`min`/`minutes`, `s`/`seconds`, `ms`/`milliseconds`, `us`/`microseconds`.

```python
logger.add("app.log", rotation="500 MB")
logger.add("app.log", rotation="12:00")
logger.add("app.log", rotation="1 week")
logger.add("app.log", rotation="monday at 00:00")
logger.add("app.log", rotation=timedelta(hours=6))
logger.add("app.log", rotation=["100 MB", "daily"])  # Whichever comes first
```

---

## File Retention

The `retention` parameter determines how long old (rotated) log files are kept.

| Type | Example | Description |
|------|---------|-------------|
| **Duration string** | `"10 days"`, `"2 weeks"`, `"1 month"` | Delete rotated files older than this age (by modification time). |
| `datetime.timedelta` | `timedelta(days=30)` | Same, using a timedelta object. |
| **int** | `5` | Keep only the N most recent rotated files. |
| **Callable** | `func(list[str]) -> None` | Custom function receiving list of log file paths to manage. |

```python
logger.add("app.log", rotation="10 MB", retention="7 days")
logger.add("app.log", rotation="daily", retention=5)  # Keep 5 most recent
```

---

## File Compression

The `compression` parameter specifies how to compress rotated files.

| Type | Example | Description |
|------|---------|-------------|
| **String** | `"gz"`, `"bz2"`, `"xz"`, `"lzma"`, `"tar"`, `"tar.gz"`, `"tar.bz2"`, `"tar.xz"`, `"zip"` | Compression format (can include leading dot: `".gz"`). |
| **Callable** | `func(filepath: str) -> None` | Custom compression function that receives the file path to compress. |

```python
logger.add("app.log", rotation="10 MB", compression="zip")
logger.add("app.log", rotation="daily", retention="30 days", compression="gz")
```

---

## Serialization (JSON Logging)

```python
logger.add("app.log", serialize=True)
```

When `serialize=True`, each log record is converted to a JSON string before being sent to the sink.
The JSON includes all record fields: time, level, message, file, function, line, exception, extra,
etc.

For custom serialization, use a format function:

```python
import json

def json_formatter(record):
    subset = {
        "timestamp": record["time"].isoformat(),
        "level": record["level"].name,
        "message": record["message"],
        "extra": record["extra"],
    }
    record["extra"]["serialized"] = json.dumps(subset)
    return "{extra[serialized]}\n"

logger.add("app.jsonl", format=json_formatter)
```

---

## Async / Enqueue / Thread Safety

```python
logger.add(sink, enqueue=True)
```

When `enqueue=True`, log messages are placed in a thread-safe `multiprocessing.SimpleQueue` and
processed by a background thread. This provides:

- **Thread safety**: Safe to use from multiple threads without locking.
- **Process safety**: Safe to use across multiple processes (with appropriate context).
- **Non-blocking**: Log calls return immediately, the actual write happens asynchronously.

For coroutine sinks, `enqueue=True` is required (or provide a `loop`):

```python
async def async_sink(message):
    await write_to_database(message)

logger.add(async_sink, enqueue=True)
```

The `context` parameter controls the multiprocessing context:

```python
logger.add(sink, enqueue=True, context="fork")     # Use fork context
logger.add(sink, enqueue=True, context="spawn")    # Use spawn context
logger.add(sink, enqueue=True, context=mp.get_context("forkserver"))  # Custom context
```

---

## Completing Async Tasks — `complete()`

```python
await logger.complete()
```

Waits for all enqueued messages to be processed and all async sinks to finish. This is an
**awaitable** — use it in async code:

```python
async def shutdown():
    await logger.complete()
```

In synchronous code, you can still use it if needed via:

```python
import asyncio
asyncio.run(logger.complete())
```

---

## Multiprocessing — `reinstall()`

```python
logger.reinstall() -> None
```

Re-initializes the logger's internal state for use in child processes created with the `spawn`
start method. Call this in the child process after receiving the logger via Process args.

```python
from multiprocessing import Process
from loguru import logger

def child(logger):
    logger.reinstall()
    logger.info("Hello from child!")

p = Process(target=child, args=(logger,))
p.start()
```

---

## Parsing Log Files — `parse()`

```python
logger.parse(
    file,
    pattern,
    *,
    cast={},
    chunk=65536,
) -> Generator[dict[str, Any], None, None]
```

Static method that parses log files using regex with named groups.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `str \| Path \| TextIO \| BinaryIO` | *(required)* | Log file path or file-like object. |
| `pattern` | `str \| Pattern` | *(required)* | Regex with named groups `(?P<name>...)` for extracting fields. |
| `cast` | `dict[str, Callable] \| Callable` | `{}` | Type converter(s). Dict maps group names to converters, or a single callable that receives the full match dict. |
| `chunk` | `int` | `65536` | Bytes to read per iteration. |

**Yields**: `dict[str, Any]` — one dict per match, keys from named groups.

```python
pattern = r"(?P<time>[\d-]+ [\d:]+) \| (?P<level>\w+) \| (?P<message>.*)"
cast = {"time": lambda t: datetime.strptime(t, "%Y-%m-%d %H:%M:%S")}

for entry in logger.parse("app.log", pattern, cast=cast):
    print(entry["time"], entry["level"], entry["message"])
```

---

## Environment Variables

All defaults for `add()` and built-in level properties can be customized via environment variables.

### Handler Defaults

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `LOGURU_AUTOINIT` | `bool` | `True` | Whether to auto-add the default stderr handler on import. |
| `LOGURU_FORMAT` | `str` | *(default format)* | Default format string for the auto-added handler. |
| `LOGURU_FILTER` | `str` | `None` | Default filter for the auto-added handler. |
| `LOGURU_LEVEL` | `str` | `"DEBUG"` | Default level for the auto-added handler. |
| `LOGURU_COLORIZE` | `bool` | `None` (auto) | Default colorize setting. |
| `LOGURU_SERIALIZE` | `bool` | `False` | Default serialize setting. |
| `LOGURU_BACKTRACE` | `bool` | `True` | Default backtrace setting. |
| `LOGURU_DIAGNOSE` | `bool` | `True` | Default diagnose setting. |
| `LOGURU_ENQUEUE` | `bool` | `False` | Default enqueue setting. |
| `LOGURU_CONTEXT` | `str` | `None` | Default multiprocessing context. |
| `LOGURU_CATCH` | `bool` | `True` | Default catch setting. |

### Level Customization

For each built-in level (`TRACE`, `DEBUG`, `INFO`, `SUCCESS`, `WARNING`, `ERROR`, `CRITICAL`):

| Variable Pattern | Type | Description |
|------------------|------|-------------|
| `LOGURU_{LEVEL}_NO` | `int` | Severity number |
| `LOGURU_{LEVEL}_COLOR` | `str` | Color markup string |
| `LOGURU_{LEVEL}_ICON` | `str` | Icon/emoji string |

### Color Control

| Variable | Description |
|----------|-------------|
| `NO_COLOR` | If set, disables all colors (honors the [NO_COLOR standard](https://no-color.org)). |
| `FORCE_COLOR` | If set, forces colors even when output is not a TTY. |

### Boolean Parsing

Environment variable booleans accept: `1`, `true`, `yes`, `y`, `ok`, `on` (truthy) or `0`,
`false`, `no`, `n`, `nok`, `off` (falsy). Case-insensitive.

---

## Type Definitions

### Core Types

```python
# Level named tuple
class Level(NamedTuple):
    name: str
    no: int
    color: str
    icon: str

# Exception info tuple
ExcInfo = Tuple[Optional[Type[BaseException]], Optional[BaseException], Optional[TracebackType]]
```

### Record Attribute Types

```python
class RecordFile:
    name: str   # Filename (e.g., "app.py")
    path: str   # Full file path

class RecordLevel:
    name: str   # Level name (e.g., "INFO")
    no: int     # Severity number (e.g., 20)
    icon: str   # Icon string

class RecordThread:
    id: int     # Thread ID
    name: str   # Thread name

class RecordProcess:
    id: int     # Process ID
    name: str   # Process name

class RecordException(NamedTuple):
    type: Optional[Type[BaseException]]
    value: Optional[BaseException]
    traceback: Optional[TracebackType]
```

All record attribute classes support `__format__()` for direct use in format strings. They format
as their primary field: `RecordFile` as `name`, `RecordLevel` as `name`, `RecordThread` as `id`,
`RecordProcess` as `id`.

### Function Types

```python
FilterDict = Dict[Optional[str], Union[str, int, bool]]
FilterFunction = Callable[[Record], bool]
FormatFunction = Callable[[Record], str]
PatcherFunction = Callable[[Record], None]
RotationFunction = Callable[[Message, TextIO], bool]
RetentionFunction = Callable[[List[str]], None]
CompressionFunction = Callable[[str], None]
StandardOpener = Callable[[str, int], int]
```

### Handler Config Types

```python
# For TextIO, callable, or logging.Handler sinks
class BasicHandlerConfig(TypedDict, total=False):
    sink: Union[TextIO, Writable, Callable[[Message], None], Handler]
    level: Union[str, int]
    format: Union[str, FormatFunction]
    filter: Optional[Union[str, FilterFunction, FilterDict]]
    colorize: Optional[bool]
    serialize: bool
    backtrace: bool
    diagnose: bool
    enqueue: bool
    catch: bool

# For file path sinks
class FileHandlerConfig(TypedDict, total=False):
    sink: Union[str, PathLikeStr]
    # ... all BasicHandlerConfig fields plus:
    rotation: Optional[Union[str, int, time, timedelta, RotationFunction, list]]
    retention: Optional[Union[str, int, timedelta, RetentionFunction]]
    compression: Optional[Union[str, CompressionFunction]]
    delay: bool
    watch: bool
    mode: str
    buffering: int
    encoding: str
    errors: Optional[str]
    newline: Optional[str]
    closefd: bool
    opener: Optional[StandardOpener]

# For async coroutine sinks
class AsyncHandlerConfig(TypedDict, total=False):
    sink: Callable[[Message], Awaitable[None]]
    # ... all BasicHandlerConfig fields plus:
    context: Optional[Union[str, BaseContext]]
    loop: Optional[AbstractEventLoop]
```

---

## Common Patterns and Recipes

### Remove Default Handler and Add Custom Ones

```python
logger.remove()  # Remove default stderr handler
logger.add(sys.stderr, level="WARNING")
logger.add("debug.log", level="DEBUG")
```

### Structured Logging with Extra Context

```python
logger.add("app.log", format="{time} | {extra[request_id]} | {message}")
request_logger = logger.bind(request_id="req-123")
request_logger.info("Handling request")
```

### Library Logging (Silent by Default)

```python
# In your library's __init__.py:
from loguru import logger
logger.disable("my_library")

# Users enable it explicitly:
logger.enable("my_library")
```

### Intercept Standard Logging

```python
import logging

class InterceptHandler(logging.Handler):
    def emit(self, record):
        try:
            level = logger.level(record.levelname).name
        except ValueError:
            level = record.levelno
        frame, depth = logging.currentframe(), 2
        while frame.f_code.co_filename == logging.__file__:
            frame = frame.f_back
            depth += 1
        logger.opt(depth=depth, exception=record.exc_info).log(level, record.getMessage())

logging.basicConfig(handlers=[InterceptHandler()], level=0)
```

### Send Loguru to Standard Logging Handler

```python
import logging
handler = logging.handlers.SysLogHandler(address="/dev/log")
logger.add(handler, format="{message}")
```

### Log to Multiple Files by Level

```python
logger.add("info.log", level="INFO", filter=lambda r: r["level"].no < 30)
logger.add("warning.log", level="WARNING")
```

### Rotating File with Retention and Compression

```python
logger.add(
    "app.log",
    rotation="100 MB",
    retention="30 days",
    compression="gz",
    encoding="utf8",
)
```

### JSON Lines Logging

```python
logger.add("app.jsonl", serialize=True)
```

### Custom Format Function

```python
def my_format(record):
    if record["exception"]:
        return "{time} | {level} | {message}\n{exception}"
    return "{time} | {level} | {message}\n"

logger.add(sys.stderr, format=my_format)
```

### Exception Catching with Decorator

```python
@logger.catch(reraise=True, exclude=(KeyboardInterrupt, SystemExit))
def main():
    run_app()
```

### Testing — Capture Logs

```python
output = []
logger.add(output.append, format="{message}")
logger.info("Test message")
assert output[-1] == "Test message"
```

### Custom File Permissions

```python
import os

def opener(file, flags):
    return os.open(file, flags, 0o600)

logger.add("secure.log", opener=opener)
```

### Lazy Evaluation for Expensive Operations

```python
logger.opt(lazy=True).debug(
    "Data dump: {data}",
    data=lambda: expensive_serialize(big_object),
)
```

### Async-Safe Logging in Web Apps

```python
logger.add("app.log", enqueue=True)

# In an async framework (FastAPI, etc.)
@app.on_event("shutdown")
async def shutdown():
    await logger.complete()
```

---

## Standard Logging Interop

### Key Differences from `logging`

| Feature | `logging` | `loguru` |
|---------|-----------|----------|
| Setup | Per-module `getLogger()` + Handler/Formatter/Filter | Single `logger` + `add()` |
| Formatting | `%` style (`%s`, `%d`) | `str.format()` style (`{}`) |
| Levels | DEBUG, INFO, WARNING, ERROR, CRITICAL | + TRACE, SUCCESS |
| Context | LoggerAdapter / extra dict | `bind()`, `contextualize()`, `patch()` |
| Exception display | Basic traceback | Colorized traceback with variable values |
| File rotation | `RotatingFileHandler` | Built-in `rotation=` parameter |
| Thread safety | Manual locking | Built-in `enqueue=True` |
| Async support | QueueHandler/QueueListener | Built-in with coroutine sinks |
| Configuration | `dictConfig` / `fileConfig` / `basicConfig` | `configure()` / `add()` |
