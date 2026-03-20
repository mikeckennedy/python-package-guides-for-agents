# DiskCache Comprehensive Reference

> **Version:** 5.6.3
> **License:** Apache 2.0
> **Python:** 3.x
> **Backend:** SQLite + filesystem
> **Thread/Process Safe:** Yes

DiskCache is a pure-Python, disk-backed cache library. It stores data in SQLite databases with optional filesystem storage for large values. All operations are thread-safe and process-safe. No server required.

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Cache](#cache)
- [FanoutCache](#fanoutcache)
- [Deque](#deque)
- [Index](#index)
- [DjangoCache](#djangocache)
- [Disk Serialization](#disk-serialization)
- [Recipes](#recipes)
- [Settings & Configuration](#settings--configuration)
- [Eviction Policies](#eviction-policies)
- [Performance Notes](#performance-notes)
- [Caveats & Best Practices](#caveats--best-practices)

---

## Installation

```bash
pip install diskcache
```

---

## Quick Start

```python
from diskcache import Cache

cache = Cache('/tmp/my-cache')     # Persistent directory
cache['key'] = 'value'             # Set
print(cache['key'])                # Get -> 'value'
print('key' in cache)              # Membership -> True
del cache['key']                   # Delete
cache.close()                      # Cleanup

# Or use as context manager
with Cache('/tmp/my-cache') as cache:
    cache.set('key', 'value', expire=60, tag='demo')
```

---

## Cache

**Module:** `diskcache.core`
**Import:** `from diskcache import Cache`

The primary disk-backed cache. Uses a single SQLite database. For concurrent write-heavy workloads, use `FanoutCache` instead.

### Constructor

```python
Cache(directory=None, timeout=60, disk=Disk, **settings)
```

| Parameter   | Type       | Default | Description |
|-------------|------------|---------|-------------|
| `directory` | `str\|None` | `None`  | Cache directory path. `None` creates a temp directory. |
| `timeout`   | `float`    | `60`    | SQLite connection timeout in seconds. |
| `disk`      | `type`     | `Disk`  | Disk serialization class (or subclass). |
| `**settings`| —          | —       | Any key from `DEFAULT_SETTINGS` (see [Settings](#settings--configuration)). |

### Properties

| Property    | Type   | Description |
|-------------|--------|-------------|
| `directory` | `str`  | Cache directory path. |
| `timeout`   | `float`| SQLite connection timeout. |
| `disk`      | `Disk` | Disk instance used for serialization. |

---

### Core Read/Write Methods

#### `set(key, value, expire=None, read=False, tag=None, retry=False) -> bool`

Store a key-value pair.

| Parameter | Type         | Default | Description |
|-----------|--------------|---------|-------------|
| `key`     | any          | —       | Cache key. |
| `value`   | any          | —       | Value to store. |
| `expire`  | `float\|None` | `None`  | Seconds until expiration. `None` = no expiry. |
| `read`    | `bool`       | `False` | If `True`, `value` is a file-like object opened for binary reading. |
| `tag`     | `str\|None`   | `None`  | Tag for batch eviction. |
| `retry`   | `bool`       | `False` | Retry on database timeout. |

Returns `True` if stored successfully.

```python
cache.set('name', 'Alice', expire=300, tag='users')

# Store a file
with open('photo.jpg', 'rb') as f:
    cache.set('photo', f, read=True)
```

#### `get(key, default=None, read=False, expire_time=False, tag=False, retry=False)`

Retrieve a value by key.

| Parameter     | Type   | Default | Description |
|---------------|--------|---------|-------------|
| `key`         | any    | —       | Cache key. |
| `default`     | any    | `None`  | Returned if key is missing. |
| `read`        | `bool` | `False` | If `True`, return a file handle instead of the value. |
| `expire_time` | `bool` | `False` | If `True`, also return the expiration timestamp. |
| `tag`         | `bool` | `False` | If `True`, also return the tag. |
| `retry`       | `bool` | `False` | Retry on database timeout. |

Return type depends on flags:
- Default: `value` (or `default`)
- `expire_time=True`: `(value, expire_time)`
- `tag=True`: `(value, tag)`
- Both: `(value, expire_time, tag)`

```python
value = cache.get('name', default='unknown')
value, exp = cache.get('name', expire_time=True)
value, exp, tag = cache.get('name', expire_time=True, tag=True)
```

#### `add(key, value, expire=None, read=False, tag=None, retry=False) -> bool`

Store only if the key does **not** already exist. Atomic operation.

Returns `True` if stored, `False` if key already exists.

```python
if cache.add('lock', True, expire=10):
    print('Lock acquired')
```

#### `delete(key, retry=False) -> bool`

Remove a key. Returns `True` if deleted, `False` if key was missing.

#### `pop(key, default=None, expire_time=False, tag=False, retry=False)`

Remove and return a value. Atomic operation. Same return format options as `get()`.

```python
value = cache.pop('counter', default=0)
```

#### `touch(key, expire=None, retry=False) -> bool`

Update expiration time without changing the value. Returns `True` if key exists.

```python
cache.touch('session', expire=3600)  # Extend by 1 hour
```

#### `read(key, retry=False)`

Return a file handle for reading binary data. The file handle's `.name` attribute contains the full file path.

```python
with cache.read('photo') as reader:
    data = reader.read()
```

---

### Dictionary-Style Access

```python
cache[key] = value      # Calls set(key, value, retry=True)
value = cache[key]      # Calls get(key); raises KeyError if missing
del cache[key]          # Calls delete(key, retry=True); raises KeyError if missing
key in cache            # Calls __contains__(key)
```

---

### Atomic Arithmetic

#### `incr(key, delta=1, default=0, retry=False) -> int|float`

Atomically increment a value. If key is missing, initializes to `default` then increments.

| Parameter | Type        | Default | Description |
|-----------|-------------|---------|-------------|
| `key`     | any         | —       | Cache key. |
| `delta`   | `int\|float` | `1`     | Amount to increment. |
| `default` | `int\|float\|None` | `0` | Initial value if missing. `None` raises `KeyError`. |
| `retry`   | `bool`      | `False` | Retry on timeout. |

```python
cache.incr('page-views')          # 0 + 1 = 1
cache.incr('page-views')          # 1 + 1 = 2
cache.incr('total', delta=9.99)   # Supports floats
```

#### `decr(key, delta=1, default=0, retry=False) -> int|float`

Atomically decrement. Same signature as `incr()`.

---

### Queue Operations

Cache supports FIFO/LIFO queue semantics using integer keys.

#### `push(value, prefix=None, side='back', expire=None, tag=None, retry=False) -> key`

Add a value to the queue. Returns the generated key.

| Parameter | Type         | Default  | Description |
|-----------|--------------|----------|-------------|
| `value`   | any          | —        | Value to push. |
| `prefix`  | `str\|None`   | `None`   | Optional key prefix for namespaced queues. |
| `side`    | `str`        | `'back'` | `'back'` or `'front'`. |
| `expire`  | `float\|None` | `None`   | Seconds until expiration. |
| `tag`     | `str\|None`   | `None`   | Tag for eviction. |
| `retry`   | `bool`       | `False`  | Retry on timeout. |

```python
cache.push('task-1')
cache.push('task-2')
cache.push('urgent', side='front')
```

#### `pull(prefix=None, default=(None, None), side='front', expire_time=False, tag=False, retry=False)`

Remove and return the next item. Returns `(key, value)` tuple.

| Parameter | Type    | Default         | Description |
|-----------|---------|-----------------|-------------|
| `prefix`  | `str\|None` | `None`       | Filter by key prefix. |
| `default` | tuple   | `(None, None)`  | Default if queue is empty. |
| `side`    | `str`   | `'front'`       | `'front'` (FIFO) or `'back'` (LIFO). |

```python
key, value = cache.pull()         # FIFO: get oldest
key, value = cache.pull(side='back')  # LIFO: get newest
```

#### `peek(prefix=None, default=(None, None), side='front', expire_time=False, tag=False, retry=False)`

View the next item without removing it. Same signature as `pull()`.

#### `peekitem(last=True, expire_time=False, tag=False, retry=False)`

Return a `(key, value)` pair. Raises `KeyError` if empty.

| Parameter | Type   | Default | Description |
|-----------|--------|---------|-------------|
| `last`    | `bool` | `True`  | `True` for last item, `False` for first. |

---

### Iteration

```python
for key in cache:                # Keys in database order (includes expired)
    pass

for key in reversed(cache):      # Reverse order
    pass

len(cache)                        # Count of items (includes expired)
```

**Note:** Iteration includes expired items. Call `cache.expire()` first to exclude them.

---

### Memoization

#### `memoize(name=None, typed=False, expire=None, tag=None, ignore=())`

Decorator that caches function return values.

| Parameter | Type         | Default | Description |
|-----------|--------------|---------|-------------|
| `name`    | `str\|None`   | `None`  | Cache key name. Default: function's qualified name. |
| `typed`   | `bool`       | `False` | If `True`, arguments of different types cached separately. |
| `expire`  | `float\|None` | `None`  | Seconds until expiration. |
| `tag`     | `str\|None`   | `None`  | Tag for batch eviction. |
| `ignore`  | `tuple`      | `()`    | Argument names to exclude from the cache key. |

The decorated function gains:
- `__wrapped__` — the original function
- `__cache_key__(*args, **kwargs)` — returns the cache key that would be used

```python
@cache.memoize(expire=3600, tag='fib')
def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

# Access underlying function
fibonacci.__wrapped__(10)

# Get cache key
key = fibonacci.__cache_key__(10)
```

---

### Cache Management

#### `clear(retry=False) -> int`

Remove all items. Returns count of items removed.

#### `expire(now=None, retry=False) -> int`

Remove expired items. Returns count removed. Call periodically for cleanup.

#### `evict(tag, retry=False) -> int`

Remove all items with the given tag. Returns count removed. Requires tag index for best performance.

#### `cull(retry=False) -> int`

Remove items until cache size is below `size_limit`. Returns count removed. Uses the configured eviction policy.

#### `check(fix=False, retry=False) -> list`

Verify database and filesystem consistency. Returns list of warnings. Set `fix=True` to repair issues.

#### `create_tag_index()` / `drop_tag_index()`

Create or remove the tag column index. The index speeds up `evict()` calls significantly.

#### `volume() -> int`

Return estimated total cache size in bytes (database + stored files).

#### `stats(enable=True, reset=False) -> (hits, misses)`

Return `(hits, misses)` tuple. Set `enable=True` to start tracking, `reset=True` to zero counters.

```python
cache.stats(enable=True)
# ... use cache ...
hits, misses = cache.stats()
```

---

### Transactions

#### `transact(retry=False)`

Context manager for atomic multi-operation transactions. Groups operations into a single SQLite transaction for atomicity and better performance (2-5x faster than individual operations).

```python
with cache.transact():
    cache.incr('total', 123.45)
    cache.incr('count')
```

**Note:** Keep transactions short — they block other writers.

---

### Settings Access

#### `reset(key, value=ENOVAL, update=True)`

Get or set a cache setting. Supports `sqlite_*` prefixes for SQLite PRAGMAs and `disk_*` for Disk attributes.

```python
cache.reset('size_limit', 2**30)
cache.reset('cull_limit', 0)        # Disable auto-culling
current = cache.reset('size_limit')  # Get current value
```

---

### Connection Management

```python
cache.close()  # Close database connection

# Context manager (recommended)
with Cache('/tmp/cache') as cache:
    cache['key'] = 'value'
# Automatically closed
```

Closed caches auto-reopen on access, but this is slower than keeping them open.

---

## FanoutCache

**Module:** `diskcache.fanout`
**Import:** `from diskcache import FanoutCache`

Sharded cache that distributes keys across multiple `Cache` instances. Reduces write contention for concurrent workloads. Recommended for multi-threaded/multi-process applications.

### Constructor

```python
FanoutCache(directory=None, shards=8, timeout=0.010, disk=Disk, **settings)
```

| Parameter   | Type       | Default  | Description |
|-------------|------------|----------|-------------|
| `directory` | `str\|None` | `None`   | Cache directory. |
| `shards`    | `int`      | `8`      | Number of Cache shards. Use 1 shard per concurrent writer. |
| `timeout`   | `float`    | `0.010`  | SQLite timeout per shard (10ms). Short timeout avoids blocking. |
| `disk`      | `type`     | `Disk`   | Disk serialization class. |
| `**settings`| —          | —        | Applied to each shard. `size_limit` is divided among shards. |

### Key Differences from Cache

1. **Never raises `Timeout`** — operations silently fail on timeout (returns `False` for set, `default` for get).
2. **Default timeout is 10ms** (vs 60s for Cache).
3. **`retry` defaults to `False`** — set explicitly if needed.
4. **`transact()` is not available** — keys are distributed across shards, so cross-shard transactions aren't possible.

### All Cache Methods Available

FanoutCache supports all Cache methods with the same signatures:
`set()`, `get()`, `add()`, `delete()`, `pop()`, `push()`, `pull()`, `peek()`, `peekitem()`, `touch()`, `incr()`, `decr()`, `read()`, `memoize()`, `clear()`, `expire()`, `evict()`, `cull()`, `check()`, `stats()`, `volume()`, `close()`, `create_tag_index()`, `drop_tag_index()`, `reset()`, `__getitem__`, `__setitem__`, `__delitem__`, `__contains__`, `__iter__`, `__reversed__`, `__len__`.

### Additional Methods

#### `cache(name, timeout=60, disk=None, **settings) -> Cache`

Get or create a named `Cache` subdirectory within the FanoutCache directory.

```python
tutorial_cache = fanout.cache('tutorial')
```

#### `deque(name, maxlen=None) -> Deque`

Get or create a named `Deque` subdirectory.

```python
task_queue = fanout.deque('tasks')
```

#### `index(name) -> Index`

Get or create a named `Index` subdirectory.

```python
url_map = fanout.index('urls')
```

---

## Deque

**Module:** `diskcache.persistent`
**Import:** `from diskcache import Deque`

Persistent, disk-backed double-ended queue. API mirrors `collections.deque`. Cross-process and cross-thread safe. Fixed memory footprint regardless of size.

### Constructor

```python
Deque(iterable=(), directory=None, maxlen=None)
```

| Parameter   | Type         | Default | Description |
|-------------|--------------|---------|-------------|
| `iterable`  | iterable     | `()`    | Initial items. |
| `directory` | `str\|None`   | `None`  | Storage directory. `None` creates temp dir. |
| `maxlen`    | `int\|None`   | `None`  | Maximum length. Oldest items dropped on overflow. |

### Class Method

#### `Deque.fromcache(cache, iterable=(), maxlen=None) -> Deque`

Create a Deque backed by an existing Cache instance.

### Properties

| Property    | Type       | Description |
|-------------|------------|-------------|
| `cache`     | `Cache`    | Underlying Cache instance. |
| `directory` | `str`      | Directory path. |
| `maxlen`    | `int\|None` | Maximum length (settable). |

### Methods

| Method | Description |
|--------|-------------|
| `append(value)` | Add to back. |
| `appendleft(value)` | Add to front. |
| `pop() -> value` | Remove and return from back. Raises `IndexError` if empty. |
| `popleft() -> value` | Remove and return from front. Raises `IndexError` if empty. |
| `peek() -> value` | View back item without removing. Raises `IndexError` if empty. |
| `peekleft() -> value` | View front item without removing. Raises `IndexError` if empty. |
| `extend(iterable)` | Add multiple items to back. |
| `extendleft(iterable)` | Add multiple items to front (each item added individually, so order reverses). |
| `clear()` | Remove all items. |
| `copy() -> Deque` | Return a shallow copy. |
| `count(value) -> int` | Count occurrences. |
| `remove(value)` | Remove first occurrence. Raises `ValueError` if not found. |
| `reverse()` | Reverse in place. |
| `rotate(steps=1)` | Rotate right by `steps`. Negative rotates left. |
| `transact()` | Context manager for atomic operations. |

### Supported Operations

```python
deque[index]             # Get by index
deque[index] = value     # Set by index
del deque[index]         # Delete by index
len(deque)               # Length
iter(deque)              # Forward iteration
reversed(deque)          # Reverse iteration
deque += iterable        # Extend (+=)
deque == other           # Equality (and !=, <, >, <=, >=)
```

### Example

```python
from diskcache import Deque

tasks = Deque(directory='/tmp/tasks')
tasks.append('task-1')
tasks.append('task-2')
tasks.appendleft('urgent-task')

task = tasks.popleft()  # 'urgent-task'
```

---

## Index

**Module:** `diskcache.persistent`
**Import:** `from diskcache import Index`

Persistent, disk-backed ordered mapping. API mirrors `collections.OrderedDict`. Cross-process and cross-thread safe. Fixed memory footprint regardless of size.

### Constructor

```python
Index(*args, **kwargs)
```

The first positional argument, if a string, is used as the directory path. Otherwise, arguments are treated as mapping initializers.

```python
index = Index('/tmp/index')                    # Empty, persistent
index = Index({'a': 1, 'b': 2})                # From dict (temp dir)
index = Index('/tmp/index', a=1, b=2)          # Persistent with initial data
index = Index([('a', 1), ('b', 2)])            # From iterable
```

### Class Method

#### `Index.fromcache(cache, *args, **kwargs) -> Index`

Create an Index backed by an existing Cache instance.

### Properties

| Property    | Type    | Description |
|-------------|---------|-------------|
| `cache`     | `Cache` | Underlying Cache instance. |
| `directory` | `str`   | Directory path. |

### Methods

| Method | Description |
|--------|-------------|
| `get(key, default=None)` | Get value or default. |
| `pop(key, default=ENOVAL)` | Remove and return value. Raises `KeyError` if missing and no default. |
| `popitem(last=True)` | Remove and return `(key, value)`. `last=False` for first item. |
| `setdefault(key, default=None)` | Set key to `default` if not present. Return value. |
| `peekitem(last=True)` | View `(key, value)` without removing. |
| `clear()` | Remove all items. |
| `update(*args, **kwargs)` | Update from mapping or iterable. |
| `keys()` | Return `KeysView`. |
| `values()` | Return `ValuesView`. |
| `items()` | Return `ItemsView`. |
| `push(value, prefix=None, side='back') -> key` | Queue-style push. |
| `pull(prefix=None, default=(None, None), side='front')` | Queue-style pull. |
| `transact()` | Context manager for atomic operations. |
| `memoize(name=None, typed=False, ignore=())` | Memoization decorator. |

### Supported Operations

```python
index[key]               # Get (raises KeyError)
index[key] = value       # Set
del index[key]           # Delete (raises KeyError)
key in index             # Membership
iter(index)              # Keys in insertion order
reversed(index)          # Reverse key order
len(index)               # Count
index == other           # Equality (and !=)
```

### Example

```python
from diskcache import Index

config = Index('/tmp/config')
config['host'] = 'localhost'
config['port'] = 8080

for key in config:
    print(f'{key} = {config[key]}')
```

---

## DjangoCache

**Module:** `diskcache.djangocache`
**Import:** `from diskcache import DjangoCache`

Django-compatible cache backend. Wraps `FanoutCache` internally. Implements Django's `BaseCache` API.

### Django Settings Configuration

```python
CACHES = {
    'default': {
        'BACKEND': 'diskcache.DjangoCache',
        'LOCATION': '/path/to/cache/directory',
        'TIMEOUT': 300,               # Default timeout in seconds
        'SHARDS': 8,                   # Number of shards
        'DATABASE_TIMEOUT': 0.010,     # SQLite timeout (10ms)
        'OPTIONS': {
            'size_limit': 2 ** 30,     # 1 GB
            # Any DEFAULT_SETTINGS key can go here
        },
    },
}
```

### Key Differences from FanoutCache

1. **`retry` defaults to `True`** — operations retry on timeout.
2. **`version` parameter** — supports Django's key versioning.
3. **`timeout` uses Django's `DEFAULT_TIMEOUT`** convention.
4. **Never raises `Timeout`** to the caller.

### Methods

All standard Django cache methods plus DiskCache extensions:

| Method | Description |
|--------|-------------|
| `set(key, value, timeout=DEFAULT_TIMEOUT, version=None, read=False, tag=None, retry=True)` | Store value. |
| `get(key, default=None, version=None, read=False, expire_time=False, tag=False, retry=False)` | Retrieve value. |
| `add(key, value, timeout=DEFAULT_TIMEOUT, version=None, read=False, tag=None, retry=True)` | Store if absent. |
| `delete(key, version=None, retry=True)` | Remove key. |
| `has_key(key, version=None)` | Check existence. |
| `incr(key, delta=1, version=None, default=None, retry=True)` | Increment. |
| `decr(key, delta=1, version=None, default=None, retry=True)` | Decrement. |
| `touch(key, timeout=DEFAULT_TIMEOUT, version=None, retry=True)` | Update expiration. |
| `pop(key, default=None, version=None, retry=True)` | Remove and return. |
| `read(key, version=None)` | Get file handle. |
| `clear()` | Remove all. |
| `close(**kwargs)` | Close connections. |
| `expire()` | Remove expired items. |
| `evict(tag)` | Remove by tag. |
| `cull()` | Enforce size limit. |
| `stats(enable=True, reset=False)` | Hit/miss statistics. |
| `create_tag_index()` / `drop_tag_index()` | Manage tag index. |
| `cache(name)` | Get named Cache subdirectory. |
| `deque(name, maxlen=None)` | Get named Deque. |
| `index(name)` | Get named Index. |
| `memoize(name=None, timeout=DEFAULT_TIMEOUT, ...)` | Memoization decorator. |

### File Serving with X-Accel-Redirect

```python
from django.core.cache import cache

# Store file content
with open('media/photo.jpg', 'rb') as f:
    cache.set('photo', f, read=True)

# Serve via NGINX X-Accel-Redirect
def media_view(request, path):
    with cache.read(path) as reader:
        response = HttpResponse()
        response['X-Accel-Redirect'] = reader.name
        return response
```

---

## Disk Serialization

**Module:** `diskcache.core`
**Import:** `from diskcache import Disk, JSONDisk`

### Disk (Default Serializer)

```python
Disk(directory, min_file_size=0, pickle_protocol=0)
```

| Parameter          | Type  | Default | Description |
|--------------------|-------|---------|-------------|
| `directory`        | `str` | —       | Cache directory path. |
| `min_file_size`    | `int` | `0`     | Minimum bytes before storing as file (vs in database). |
| `pickle_protocol`  | `int` | `0`     | Pickle protocol version. |

**Native types** stored directly in SQLite (no pickling): `int`, `float`, `str`, `bytes`.
**All other types** are pickled.

#### Methods (for subclassing)

| Method | Description |
|--------|-------------|
| `hash(key) -> int` | Portable hash for sharding. |
| `put(key) -> (db_key, raw)` | Serialize key for database storage. |
| `get(key, raw) -> object` | Deserialize key from database. |
| `store(value, read, key=UNKNOWN) -> (size, mode, filename, db_value)` | Serialize value for storage. |
| `fetch(mode, filename, value, read) -> object` | Deserialize stored value. |
| `filename(key=UNKNOWN, value=UNKNOWN) -> (filename, full_path)` | Generate filename for file storage. |
| `remove(file_path)` | Remove a stored file (cross-process safe). |

### JSONDisk

```python
JSONDisk(directory, compress_level=1, **kwargs)
```

JSON serialization with zlib compression. Useful when you need consistent key serialization (pickle can produce different bytes for equivalent tuples).

| Parameter        | Type  | Default | Description |
|------------------|-------|---------|-------------|
| `compress_level` | `int` | `1`     | zlib compression level (0=none, 1=fast, 9=best). |

```python
from diskcache import Cache, JSONDisk

cache = Cache('/tmp/json-cache', disk=JSONDisk, disk_compress_level=6)
```

### Custom Disk Subclass Example

```python
import json, zlib
from diskcache import Disk, UNKNOWN

class MyDisk(Disk):
    def __init__(self, directory, compress_level=1, **kwargs):
        self.compress_level = compress_level
        super().__init__(directory, **kwargs)

    def store(self, value, read, key=UNKNOWN):
        if not read:
            value = zlib.compress(json.dumps(value).encode(), self.compress_level)
        return super().store(value, read, key=key)

    def fetch(self, mode, filename, value, read):
        data = super().fetch(mode, filename, value, read)
        if not read:
            data = json.loads(zlib.decompress(data))
        return data

cache = Cache('/tmp/custom', disk=MyDisk, disk_compress_level=6)
```

---

## Recipes

**Module:** `diskcache.recipes`
**Import:** `from diskcache import Lock, RLock, BoundedSemaphore, Averager, throttle, barrier, memoize_stampede`

### Lock

Cross-process/thread mutual exclusion lock.

```python
Lock(cache, key, expire=None, tag=None)
```

| Method | Description |
|--------|-------------|
| `acquire()` | Acquire lock (blocks until available). |
| `release()` | Release lock. |
| `locked() -> bool` | Check if locked. |
| Context manager | `with Lock(cache, 'my-lock'):` |

```python
from diskcache import Cache, Lock

cache = Cache('/tmp/cache')
lock = Lock(cache, 'critical-section')

with lock:
    # Exclusive access across processes
    do_work()
```

### RLock

Re-entrant lock (same thread can acquire multiple times).

```python
RLock(cache, key, expire=None, tag=None)
```

Same API as `Lock`, but the same thread can call `acquire()` multiple times without deadlocking. Must call `release()` the same number of times.

### BoundedSemaphore

Limits concurrent access to a resource.

```python
BoundedSemaphore(cache, key, value=1, expire=None, tag=None)
```

| Parameter | Type  | Default | Description |
|-----------|-------|---------|-------------|
| `value`   | `int` | `1`     | Maximum concurrent acquisitions. |

```python
from diskcache import Cache, BoundedSemaphore

cache = Cache('/tmp/cache')
sem = BoundedSemaphore(cache, 'pool', value=5)

with sem:
    # Max 5 concurrent threads/processes
    use_resource()
```

### Averager

Running average calculator.

```python
Averager(cache, key, expire=None, tag=None)
```

| Method | Description |
|--------|-------------|
| `add(value)` | Add a value to the running average. |
| `get() -> float` | Get current average. |
| `pop() -> float` | Get average and reset. |
| `__call__()` | Same as `get()`. |

```python
from diskcache import Cache, Averager

cache = Cache('/tmp/cache')
avg = Averager(cache, 'response-time')
avg.add(0.25)
avg.add(0.30)
print(avg.get())  # 0.275
```

### throttle (Decorator)

Rate-limit function calls.

```python
@throttle(cache, count, seconds, name=None, expire=None, tag=None,
          time_func=time.time, sleep_func=time.sleep)
def func():
    pass
```

| Parameter    | Type       | Default      | Description |
|--------------|------------|--------------|-------------|
| `cache`      | `Cache`    | —            | Cache instance. |
| `count`      | `int`      | —            | Max calls per interval. |
| `seconds`    | `float`    | —            | Interval duration. |
| `name`       | `str\|None` | `None`       | Cache key name. |
| `expire`     | `float\|None`| `None`      | Expiration for rate limit data. |
| `tag`        | `str\|None` | `None`       | Tag for eviction. |
| `time_func`  | callable   | `time.time`  | Time function. |
| `sleep_func` | callable   | `time.sleep` | Sleep function. |

```python
from diskcache import Cache, throttle

cache = Cache('/tmp/cache')

@throttle(cache, count=10, seconds=60)
def api_call():
    return requests.get('https://api.example.com')
```

### barrier (Decorator)

Serialize function execution using a lock.

```python
@barrier(cache, lock_factory, name=None, expire=None, tag=None)
def func():
    pass
```

| Parameter      | Type       | Default | Description |
|----------------|------------|---------|-------------|
| `cache`        | `Cache`    | —       | Cache instance. |
| `lock_factory` | type       | —       | Lock class (e.g., `Lock`). |
| `name`         | `str\|None` | `None`  | Cache key name. |

Useful for preventing cache stampedes with double-checked locking:

```python
from diskcache import Cache, Lock, barrier

cache = Cache('/tmp/cache')

@cache.memoize(expire=0)        # Optimistic lookup (no store)
@barrier(cache, Lock)            # Serialize computation
@cache.memoize(expire=60)       # Actual cache store
def expensive():
    return compute()
```

### memoize_stampede (Decorator)

Cache stampede prevention via probabilistic early recomputation.

```python
@memoize_stampede(cache, expire, name=None, typed=False, tag=None, beta=1, ignore=())
def func():
    pass
```

| Parameter | Type         | Default | Description |
|-----------|--------------|---------|-------------|
| `cache`   | `Cache`      | —       | Cache instance. |
| `expire`  | `float`      | —       | Cache TTL in seconds. |
| `name`    | `str\|None`   | `None`  | Cache key name. |
| `typed`   | `bool`       | `False` | Cache different types separately. |
| `tag`     | `str\|None`   | `None`  | Tag for eviction. |
| `beta`    | `float`      | `1`     | Recomputation probability factor (0.3-1.0 typical). Higher = more eager recomputation. |
| `ignore`  | `tuple`      | `()`    | Argument names to exclude from cache key. |

```python
from diskcache import Cache, memoize_stampede

cache = Cache('/tmp/cache')

@memoize_stampede(cache, expire=300, beta=1.0)
def landing_page():
    return render_template()
```

---

## Settings & Configuration

### DEFAULT_SETTINGS

```python
from diskcache import DEFAULT_SETTINGS
```

| Setting                | Default                  | Description |
|------------------------|--------------------------|-------------|
| `statistics`           | `0` (disabled)           | Track hit/miss statistics. |
| `tag_index`            | `0` (disabled)           | Create index on tag column. |
| `eviction_policy`      | `'least-recently-stored'`| Eviction strategy. |
| `size_limit`           | `2**30` (1 GB)           | Target cache size in bytes (soft limit). |
| `cull_limit`           | `10`                     | Max items removed per cull cycle. Set `0` to disable auto-culling. |
| `sqlite_auto_vacuum`   | `1` (FULL)               | SQLite auto-vacuum mode. |
| `sqlite_cache_size`    | `2**13` (8192 pages)     | SQLite in-memory page cache. |
| `sqlite_journal_mode`  | `'wal'`                  | Write-Ahead Logging (readers don't block writers). |
| `sqlite_mmap_size`     | `2**26` (64 MB)          | Memory-mapped I/O size. |
| `sqlite_synchronous`   | `1` (NORMAL)             | Sync mode (lower = faster, less durable). |
| `disk_min_file_size`   | `2**15` (32 KB)          | Values larger than this stored as files, not in SQLite. |
| `disk_pickle_protocol` | `HIGHEST_PROTOCOL`       | Pickle protocol version. |

### Applying Settings

```python
# At creation
cache = Cache('/tmp/cache', size_limit=int(4e9), statistics=True, tag_index=True)

# After creation
cache.reset('size_limit', int(4e9))
cache.reset('cull_limit', 0)

# Read current value
current_limit = cache.reset('size_limit')

# Disk settings use disk_ prefix
cache = Cache('/tmp/cache', disk_min_file_size=2**20)  # 1 MB threshold

# SQLite settings use sqlite_ prefix
cache = Cache('/tmp/cache', sqlite_mmap_size=2**28)    # 256 MB mmap
```

---

## Eviction Policies

```python
from diskcache import EVICTION_POLICY
```

| Policy | Key | Behavior | Performance Impact |
|--------|-----|----------|-------------------|
| No eviction | `'none'` | Cache grows without limit. | No overhead. Used by Deque/Index. |
| Least Recently Stored | `'least-recently-stored'` | Evicts items by insertion/update time. | No read overhead. Default. |
| Least Recently Used | `'least-recently-used'` | Evicts items by last access time. | Every `get()` updates the database. |
| Least Frequently Used | `'least-frequently-used'` | Evicts least-accessed items. | Every `get()` increments a counter. |

```python
cache = Cache('/tmp/cache', eviction_policy='least-recently-used')
```

**Note:** LRU and LFU have per-read write overhead (database updates on every access). LRS (default) has no read-side overhead.

---

## Performance Notes

### Latency (single process, 100K operations)

| Operation | DiskCache (Cache) | Memcached | Redis |
|-----------|-------------------|-----------|-------|
| GET       | ~12 us            | ~25 us    | ~40 us |
| SET       | ~69 us            | ~28 us    | ~44 us |
| DELETE    | ~47 us            | ~26 us    | ~42 us |

- **Reads are faster** than Memcached/Redis (no network overhead).
- **Writes are slower** (SQLite transactions), but still fast.
- FanoutCache with sharding reduces write contention significantly.

### Concurrency (8 processes, FanoutCache)

| Operation | DiskCache (FanoutCache) | Memcached |
|-----------|------------------------|-----------|
| GET       | ~16 us                 | ~84 us    |
| SET       | ~135 us                | ~85 us    |
| DELETE    | ~89 us                 | ~84 us    |

### Optimization Tips

1. **Use FanoutCache** for concurrent workloads (1 shard per writer).
2. **Batch operations in transactions** for 2-5x write throughput.
3. **WAL journal mode** (default) allows concurrent reads during writes.
4. **Increase `sqlite_mmap_size`** for read-heavy workloads.
5. **Set `cull_limit=0`** and schedule `cull()` via cron for write-heavy workloads.
6. **Use `read=True`** for large binary values to serve files directly.

---

## Caveats & Best Practices

### Important Caveats

1. **Size limit is soft.** The cache can exceed `size_limit` between cull cycles. Culling is lazy — it runs during `set()`/`add()` and removes at most `cull_limit` items.

2. **Iteration includes expired items.** Call `cache.expire()` before iterating if you need only live items.

3. **Pickle key inconsistency.** Pickle may produce different byte sequences for equivalent tuple keys. Use `JSONDisk` if you need consistent key serialization.

4. **Python's `__hash__`/`__eq__` are NOT used** for cache key lookups. Keys are compared by their serialized byte representation.

5. **NFS is not supported.** SQLite performs poorly on network filesystems.

6. **No async/await.** SQLite's Python module lacks async support. Use a thread pool executor for async contexts.

7. **Disk/database full** raises `sqlite3.OperationalError`.

8. **FanoutCache `set()` can silently fail** on timeout (returns `False`). Check return values if reliability matters.

### Best Practices

1. **Always use context managers** (`with Cache(...) as cache:`) or call `close()` explicitly.

2. **Enable tag_index** at creation time if you plan to use `evict()`:
   ```python
   cache = Cache('/tmp/cache', tag_index=True)
   ```

3. **Monitor disk usage** with `cache.volume()` and set appropriate `size_limit`.

4. **Run `cache.check(fix=True)`** periodically in maintenance scripts to repair any inconsistencies.

5. **Use `expire=` on `set()`** to prevent unbounded cache growth.

6. **Use `add()` for lock-like patterns** — it's atomic and returns `False` if the key exists.

7. **Prefer `incr()`/`decr()`** over `get()`+`set()` for counters — they're atomic.

8. **For multi-process queues**, use `Deque` or `push()`/`pull()` — they handle locking automatically.

### Common Patterns

#### Atomic Counter

```python
cache.incr('page-views')
views = cache.pop('page-views', default=0)  # Get and reset
```

#### Simple Distributed Lock

```python
if cache.add('lock:resource', True, expire=30):
    try:
        do_work()
    finally:
        cache.delete('lock:resource')
```

#### Multi-Process Work Queue

```python
from diskcache import Deque

queue = Deque(directory='/tmp/work-queue')

# Producer
queue.append({'task': 'process', 'id': 42})

# Consumer (in another process)
while True:
    try:
        task = queue.popleft()
        process(task)
    except IndexError:
        time.sleep(1)
```

#### Cache-Aside Pattern

```python
def get_user(user_id):
    result = cache.get(f'user:{user_id}')
    if result is not None:
        return result
    user = db.query(f'SELECT * FROM users WHERE id = {user_id}')
    cache.set(f'user:{user_id}', user, expire=300, tag='users')
    return user
```

---

## Sentinels & Constants

| Name     | Description |
|----------|-------------|
| `ENOVAL` | Sentinel for "no value" — distinguishes missing keys from `None` values. |
| `UNKNOWN`| Sentinel for unknown key/value in Disk methods. |
| `DBNAME` | Database filename: `'cache.db'`. |

### Storage Modes (Internal)

| Constant      | Value | Description |
|---------------|-------|-------------|
| `MODE_RAW`    | `1`   | Native SQLite type (int, float, str, bytes). |
| `MODE_BINARY` | `2`   | Binary data in database. |
| `MODE_TEXT`   | `3`   | Text data in database. |
| `MODE_PICKLE` | `4`   | Pickled Python object. |

---

## Exceptions & Warnings

| Name                 | Base           | Description |
|----------------------|----------------|-------------|
| `Timeout`            | `Exception`    | SQLite database timeout expired. |
| `UnknownFileWarning` | `UserWarning`  | Unknown files found in cache directory. |
| `EmptyDirWarning`    | `UserWarning`  | Empty subdirectories found in cache directory. |

---

## Full Import Reference

```python
from diskcache import (
    # Core
    Cache,
    FanoutCache,
    Deque,
    Index,

    # Serialization
    Disk,
    JSONDisk,

    # Recipes
    Lock,
    RLock,
    BoundedSemaphore,
    Averager,
    throttle,
    barrier,
    memoize_stampede,

    # Constants
    ENOVAL,
    UNKNOWN,
    DEFAULT_SETTINGS,
    EVICTION_POLICY,

    # Exceptions
    Timeout,
    UnknownFileWarning,
    EmptyDirWarning,

    # Django (if Django installed)
    DjangoCache,
)
```
