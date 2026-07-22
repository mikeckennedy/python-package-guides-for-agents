# switchlang (python-switch) Comprehensive Reference

> **Version:** 0.1.3
> **Author:** Michael Kennedy (michael@talkpython.fm)
> **License:** MIT
> **Repository:** https://github.com/mikeckennedy/python-switch
> **Documentation:** https://mkennedy.codes/docs/python-switch
> **Python:** 3.9 - 3.15
> **Dependencies:** none (zero runtime dependencies)

## Overview

`switchlang` adds an explicit **switch statement to Python without changing the language**. It is implemented as a context manager: you open a `with switch(value)` block, register cases as method calls, and read the matched case's return value from `s.result` after the block exits. The entire library is ~200 lines, ships type hints with a `py.typed` marker (PEP 561), and has no runtime dependencies.

Note the naming split: the **GitHub repo is `python-switch`**, but the **PyPI distribution and import name are `switchlang`** — `pip install switchlang`, `import switchlang`.

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/python-switch](https://mkennedy.codes/docs/python-switch). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/python-switch
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/python-switch/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/python-switch/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/python-switch/skill.md — also discoverable at `/.well-known/agent-skills/switchlang/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/python-switch/skills.html

Every documentation page also has a plain-Markdown twin — swap the `.html` extension for `.md` to get token-efficient source without the site chrome. For example https://mkennedy.codes/docs/python-switch/reference/switch.html → https://mkennedy.codes/docs/python-switch/reference/switch.md

## Installation

```bash
pip install switchlang
# or
uv add switchlang
```

## Quick Start

```python
from switchlang import switch

def main():
    num = 7
    val = input('Enter a character, a, b, c or any other: ')

    with switch(val) as s:
        s.case('a', process_a)
        s.case('b', lambda: process_with_data(val, num, 'other values still'))
        s.default(process_any)

    print(s.result)  # return value of whichever case ran

def process_a():
    print('Found A!')

def process_any():
    print('Found Default!')

def process_with_data(*value):
    print(f'Found with data: {value}')

main()
```

Key mental model: `case()` and `default()` only **register** cases; the matched function executes when the `with` block **exits** (in `__exit__`), and `s.result` is only readable after the block.

---

## Public API Reference

The package exports exactly two public names:

```python
__all__ = ['switch', 'closed_range']
```

Module-level metadata: `switchlang.__version__` (read at runtime from installed package metadata via `importlib.metadata`, falling back to `'0.0.0'` when the package is not installed) and `switchlang.__author__`.

---

### `class switch(value)`

The context manager that gives Python an explicit switch statement. Use it in a `with` block, register cases, then read `result` after the block.

#### Signature

```python
class switch:
    def __init__(self, value: Any) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `value` | `Any` | *(required)* | The value each case key is compared against (with `==`). |

#### Usage

```python
from switchlang import switch, closed_range

with switch(value) as s:
    s.case('a', process_a)                  # key == value -> run process_a
    s.case(['v', 'b'], view_bookings)       # list key: each item is a case
    s.case(range(1, 6), handler)            # range key: each item is a case
    s.case(closed_range(6, 9), handler2)    # inclusive range: 6, 7, 8, 9
    s.case(2, do_two, fallthrough=True)     # opt into running the next case too
    s.default(unknown_command)              # runs if nothing else matched
print(s.result)                             # return value of the executed case
```

#### Semantics (all pinned by the test suite)

- **Cases run on block exit, not at registration.** `case()`/`default()` only register; the matched function(s) execute in `__exit__`. So `s.result` is only valid *after* the `with` block — reading it inside the block raises `Exception`.
- **Matching is equality-based** (`key == value`). Keys are also stored in a `set`, so case keys must be **hashable**; an unhashable key (e.g. a `dict`) raises `TypeError`. Any hashable value works as a key, including `None` and arbitrary objects. Corollary of `==` plus set storage: keys that compare equal are duplicates (`1` and `True`, `1` and `1.0`).
- **`default()` is just a case** keyed on a private sentinel, and **ordering is not enforced**: a default registered *before* a matching case will also run (both functions execute; `result` is the matched case's since it runs last). **Always register `default()` last.**
- **`case()` returns `bool`** — `True` if the case (or any item of a list/range key) matched the switch value.
- **Fall-through is opt-in** per case via `fallthrough=True`; the next registered case then runs whether or not its key matches, and so on until a case without fall-through. When falling through, `result` is the return value of the **last** function executed. Falling through into `default()` runs the default too (it registers with `fallthrough=False`, so the chain stops there).
- **An exception raised inside the `with` block aborts the switch**: `__exit__` re-raises it before any case actions run, so no case functions execute. This holds for any exception — the guard is `exc_val is not None` (identity, not truthiness), so even an exception whose `__bool__` returns `False` still aborts.
- **No match and no default** raises `Exception` on block exit.

---

### `switch.case(key, func, fallthrough=False)`

Register a case for the switch block.

#### Signature

```python
def case(
    self,
    key: Any,
    func: Callable[[], Any],
    fallthrough: bool | None = False,
) -> bool
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `key` | `Any` | *(required)* | Key for the case test. If this is a `list` or `range`, each item is added as a case for `func`. Otherwise it must be hashable. |
| `func` | `Callable[[], Any]` | *(required)* | Any callable taking **no parameters**, executed on block exit if this case matches. Its return value becomes `result`. Use a lambda/closure to bind arguments. |
| `fallthrough` | `bool \| None` | `False` | Pass `True` to also run the next registered case. **`None` is reserved for internal use** (list/range expansion) — callers must pass only `True` or `False`. |

#### Returns

`bool` — `True` if this case (or any item of a list/range key) matched the switch value, otherwise `False`.

#### Raises

- `ValueError` — duplicate case key (`'Duplicate case: {key}'`).
- `ValueError` — `func` is `None` (`'Action for case cannot be None.'`) or not callable (`'Func must be callable.'`).
- `ValueError` — empty list or empty range key (`'You cannot pass an empty collection as the case. It will never match.'`); it could never match.
- `TypeError` — unhashable key (from the underlying `set` membership test).

#### Example

```python
with switch(action) as s:
    matched = s.case(['c', 'a'], create_account)   # matched is True/False
    s.case('l', log_into_account)
    s.case(range(1, 6), lambda: set_level(action))  # bind args via lambda
    s.default(unknown_command)
```

---

### `switch.default(func)`

Register the default case: the action to run when no other case matches.

#### Signature

```python
def default(self, func: Callable[[], Any]) -> None
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `func` | `Callable[[], Any]` | *(required)* | Any callable taking no parameters, executed if no other case matched. |

#### Returns

`None`.

#### Behavior

- Internally calls `case()` with a private sentinel key, so the same `func` validation applies (`ValueError` if `None`/not callable) and calling `default()` twice raises the duplicate-case `ValueError`.
- Ordering is **not** enforced: a default registered before a case that also matches will run too. Always make `default()` the last registration in the block.

---

### `switch.result` *(property)*

The value returned by the function of the matched case.

#### Signature

```python
@property
def result(self) -> Any
```

#### Returns

The value returned by the matched case's function. When cases fall through, `result` is the return value of the **last** function executed.

#### Raises

- `Exception` — if accessed before the switch block has exited and computed a result (`'No result has been computed (did you access switch.result inside the with block?)'`).

#### Behavior details

- The "no result" check is **identity-based** (`is`) against a private sentinel, not equality. A computed result with a permissive `__eq__` (e.g. a NumPy array) is therefore never mistaken for "nothing computed."
- `None` is a valid computed result and is distinct from "not computed": a case whose function returns `None` gives `s.result is None` without raising.

#### Example

```python
value = 4
with switch(value) as s:
    s.case(closed_range(1, 5), lambda: '1-to-5')
    s.default(lambda: 'default')

res = s.result  # res == '1-to-5'; reading this INSIDE the block raises
```

---

### `switch.__enter__()` / `switch.__exit__(exc_type, exc_val, exc_tb)`

The context-manager protocol — you never call these directly, but their behavior defines the library.

```python
def __enter__(self) -> switch                    # returns self; bind with `as s`
def __exit__(
    self,
    exc_type: type[BaseException] | None,
    exc_val: BaseException | None,
    exc_tb: TracebackType | None,
) -> None
```

`__exit__` is where everything happens, in this order:

1. If an exception was raised inside the block (`exc_val is not None` — identity check, deliberately not truthiness), it is re-raised and **no case actions run**.
2. If no case matched and no default was registered, raises `Exception('Value does not match any case and there is no default case: value {value}')`.
3. Otherwise runs the matched function and any fall-through functions in registration order, storing each return value; the last one wins as `result`.

---

### `closed_range(start, stop, step=1)`

Create a closed range for a case: both `start` and `stop` are included — unlike built-in `range`, where the upper bound is exclusive.

#### Signature

```python
def closed_range(start: int, stop: int, step: int = 1) -> range
```

#### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `start` | `int` | *(required)* | The inclusive lower bound. Must be strictly less than `stop`. |
| `stop` | `int` | *(required)* | The inclusive upper bound. |
| `step` | `int` | `1` | Step size between elements; must be 1 or greater. |

#### Returns

`range` — a plain `range` object (`range(start, stop + 1, step)`) with a closed (inclusive) upper bound; usable anywhere a range is, not just in `case()`.

#### Raises

- `ValueError` — `start >= stop` (`'Start must be less than stop.'`). Note this rejects one-element ranges: `closed_range(3, 3)` raises.
- `ValueError` — `step < 1` (`'Step must be 1 or greater.'`).

#### Behavior details

- `closed_range(1, 5)` -> 1, 2, 3, 4, 5.
- The range never overshoots `stop`: `closed_range(1, 6, 2)` -> 1, 3, 5. `stop` itself is included when the step lands on it exactly: `closed_range(1, 7, 2)` -> 1, 3, 5, 7.
- **Adjacent closed ranges overlap**: `closed_range(1, 5)` and `closed_range(5, 9)` both contain 5, so registering both raises the duplicate-case `ValueError`. Plain `range(1, 5)` / `range(5, 9)` do not overlap.

#### Example

```python
from switchlang import switch, closed_range

with switch(value) as s:
    s.case(closed_range(1, 5), lambda: 'one to five')  # matches 1, 2, 3, 4, 5
    s.case(closed_range(6, 9), lambda: 'six to nine')  # matches 6, 7, 8, 9
    s.default(lambda: 'something else')
```

---

## Exceptions

The library defines **no custom exception classes**. It raises built-ins:

| Exception | Raised when |
|---|---|
| `ValueError` | Duplicate case key; `func` is `None` or not callable; empty list/range key; `closed_range` with `start >= stop` or `step < 1`. Raised at **registration** time (or call time for `closed_range`). |
| `Exception` (plain) | No case matched and no `default()` was registered (raised on block exit); `result` accessed before the block has exited. |
| `TypeError` | Unhashable case key (from set storage). |

Any exception raised by user code — inside the `with` block or by a case function — propagates unchanged. An exception inside the block aborts the switch before any case function runs; an exception inside a case function propagates from `__exit__`.

---

## Internal API

Not part of `__all__` — listed for understanding, not for use.

- **`switchlang.__switchlang_impl`** — the private module (leading double underscore) holding the entire implementation. Always import via the package (`from switchlang import switch, closed_range`), never from the impl module directly.
- **`switch._switch__no_result` / `switch._switch__default`** — name-mangled class attributes assigned `uuid.uuid4()` at class-load time; unique opaque sentinels for "no result computed" and the default-case key. The `result` property compares against `__no_result` with `is` (identity) deliberately.
- **Instance attributes** — `s.value` (the switch value), `s.cases` (the `set` of registered keys); plus private state `_found`, `_falling_through`, `_func_stack` (the callables `__exit__` will run). Treat all of these as read-only implementation detail.
- **`fallthrough=None`** — the reserved internal value: during list/range expansion, `case()` recurses per item with `fallthrough=None`, meaning "leave the fall-through state unchanged."

---

## Execution model (the part agents get wrong)

`with switch(value)` does **not** dispatch as cases are registered. Each `case()` call only records whether its key matched and pushes the function onto an internal stack; everything executes when the block exits:

```python
with switch(2) as s:
    s.case(1, lambda: print('one'))
    s.case(2, lambda: print('two'))   # nothing prints here...
    s.default(lambda: print('other'))
# ...it prints as the block exits, right here

print(s.result)  # and only now is result readable
```

Consequences:

- Case functions take **zero parameters**. Bind data with lambdas or closures: `s.case('b', lambda: handle(val, num))`.
- Reading `s.result` inside the block always raises, even for a case already registered and matched.
- Registration-time errors (duplicate key, bad func, empty collection) raise immediately at the `s.case(...)` line; the no-match error raises at the block's closing line.

## Fall-through mechanics

```python
value = 2
with switch(value) as s:
    s.case(1, lambda: 'one')
    s.case(2, lambda: 'two', fallthrough=True)
    s.case(3, lambda: 'three')          # runs as well (key 3 never compared), then stops
    s.default(lambda: 'other')

print(s.result)  # 'three' — the LAST function executed wins
```

- Fall-through is per-case and opt-in; the chain continues through consecutive `fallthrough=True` cases and stops at the first case registered without it.
- A fall-through target runs **regardless of whether its own key matches**.
- If the matched case falls through and the next registration is `default()`, the default runs too.
- List/range cases support `fallthrough=True` the same way.

## Compatibility notes

- **Python 3.9+** (classifiers through 3.15). Pure Python, single module, zero runtime dependencies — safe to add to any project.
- **Typed**: ships inline type hints plus a `py.typed` marker (PEP 561), so mypy/pyright/ty read the annotations directly.
- **vs. `match` (PEP 634)**: `match` destructures by shape but has no fall-through, no runtime-built case lists, no result capture, and needs 3.10+. `switchlang` is value-equality dispatch with opt-in fall-through, list/range keys built at runtime, `s.result` capture, and 3.9 support. Also, a bare name in a `match` case pattern *captures* rather than compares — that gotcha cannot happen here.
- **vs. dict dispatch**: `{key: func}.get(value, default)()` works but has no duplicate-key detection (later keys silently win), no callable validation, no ranges/fall-through.

## Common patterns

```python
# Map several discrete values to one action
s.case(['c', 'a'], create_account)

# Map a numeric span to one action (inclusive)
s.case(closed_range(1, 5), lambda: set_level(action))

# Do nothing for a key (still counts as matched — no default runs)
s.case('', lambda: None)

# Bind arguments to the zero-arg case function
s.case('b', lambda: process(val, num))

# Use the match test at registration time
if s.case('x', exit_app):
    ...  # returns True when 'x' == value
```

## Complete working example

```python
from switchlang import switch, closed_range


def create_account():
    return 'created'


def view_bookings():
    return 'bookings'


def set_level(level):
    return f'level {level}'


def unknown_command():
    return 'unknown'


def run(action):
    with switch(action) as s:
        s.case(['c', 'a'], create_account)
        s.case(['v', 'b'], view_bookings)
        s.case('', lambda: None)
        s.case(closed_range(1, 5), lambda: set_level(action))
        s.default(unknown_command)

    return s.result


print(run('c'))   # created
print(run('b'))   # bookings
print(run(3))     # level 3
print(run('z'))   # unknown
print(run(''))    # None (matched, function returned None)
```

## Error-handling patterns

```python
# Duplicate case — raises ValueError at the second registration line
with switch(3) as s:
    s.case(closed_range(1, 5), lambda: 'low')
    s.case(closed_range(5, 9), lambda: 'high')   # ValueError: Duplicate case: 5

# No match, no default — raises Exception at block exit
with switch('nope') as s:      # <- Exception raised on this line's block exit
    s.case(1, lambda: 'one')

# result read too early — raises Exception
with switch(1) as s:
    s.case(1, lambda: 'one')
    print(s.result)            # Exception: No result has been computed ...

# Exception in the block — propagates, NO case actions run
with switch(1) as s:
    s.case(1, do_work)
    raise RuntimeError('boom')  # do_work never runs

# Exception in a case action — propagates from block exit
with switch(1) as s:
    s.case(1, lambda: 1 / 0)   # ZeroDivisionError raised at block exit
```

## API summary table

| Symbol | Purpose | Returns |
|---|---|---|
| `switch(value)` | Create a switch block testing cases against `value` | context manager (`__enter__` returns self) |
| `s.case(key, func, fallthrough=False)` | Register a case (list/range keys expand to one case per item) | `bool` — did it match |
| `s.default(func)` | Register the no-match action (always last) | `None` |
| `s.result` | Return value of the executed case (last one when falling through); post-block only | `Any` |
| `closed_range(start, stop, step=1)` | Inclusive-both-ends range for case keys | `range` |
