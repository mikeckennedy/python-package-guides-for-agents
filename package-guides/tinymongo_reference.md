# TinyMongo — Comprehensive API Reference

> TinyMongo gives you a PyMongo-shaped document API backed by embedded storage instead of a MongoDB server. The default backend is [TinyDB](https://tinydb.readthedocs.io/) JSON files, with optional memory, SQLite, DuckDB, Parquet, PostgreSQL, and MariaDB/MySQL backends. It implements a practical *subset* of PyMongo — enough for small apps, prototypes, CLI tools, and test suites that would otherwise need a running `mongod`.

> Version: 1.2.0 | License: MIT | Python: 3.9+ | Built from the source code of the TinyMongo 1.2.0 repository (`schapman1974/tinymongo`), with every signature, default, and behavior below cross-checked against the implementation and verified against a live install.

---

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [What Is and Is Not Supported](#what-is-and-is-not-supported)
- [Package Exports](#package-exports)
- [TinyMongoClient](#tinymongoclient)
  - [Constructor](#constructor)
  - [Database Access](#database-access)
  - [Client Methods](#client-methods)
  - [Capabilities](#capabilities)
- [MongoClient (PyMongo Drop-In)](#mongoclient-pymongo-drop-in)
- [TinyMongoDatabase](#tinymongodatabase)
- [TinyMongoCollection](#tinymongocollection)
  - [Inserts](#inserts)
  - [Queries](#queries)
  - [Updates](#updates)
  - [Find and Modify](#find-and-modify)
  - [Deletes](#deletes)
  - [Counting and Distinct](#counting-and-distinct)
  - [Collection Management](#collection-management)
  - [Unsupported Collection Methods](#unsupported-collection-methods)
- [Query Operators](#query-operators)
- [Update Operators](#update-operators)
- [Projections](#projections)
- [TinyMongoCursor](#tinymongocursor)
- [Indexes](#indexes)
- [Result Classes](#result-classes)
- [Exceptions](#exceptions)
- [Async API](#async-api)
  - [AsyncMongoClient / AsyncTinyMongoClient](#asyncmongoclient--asynctinymongoclient)
  - [AsyncTinyMongoDatabase](#asynctinymongodatabase)
  - [AsyncTinyMongoCollection](#asynctinymongocollection)
  - [AsyncTinyMongoCursor](#asynctinymongocursor)
- [Storage Backends](#storage-backends)
  - [Backend Selection](#backend-selection)
  - [Memory Backend](#memory-backend)
  - [Table-Native Backends](#table-native-backends)
  - [Object Storage (Parquet)](#object-storage-parquet)
  - [Remote SQL Backends](#remote-sql-backends)
- [Patching PyMongo in Tests](#patching-pymongo-in-tests)
- [BSON Types: ObjectId and datetime](#bson-types-objectid-and-datetime)
- [Concurrency and Locking](#concurrency-and-locking)
- [Command Line Interface](#command-line-interface)
- [Environment Variables](#environment-variables)
- [Gotchas and Behavioral Differences](#gotchas-and-behavioral-differences)
- [Common Patterns](#common-patterns)

---

## Installation

```bash
pip install tinymongo
```

The base install pulls only `tinydb>=3.2.1,<4` and `portalocker>=2.7.0`. Everything else is an extra:

| Extra | Installs | Needed for |
|-------|----------|------------|
| `tinymongo[serialization]` | `tinydb-serialization>=1.0.4,<2` | `tinymongo.serializers.DateTimeSerializer` |
| `tinymongo[bson]` | `pymongo>=4.9` | `ObjectId` values in documents |
| `tinymongo[pymongo]` | `pymongo>=4.9` | `tinymongo.patch()`, PyMongo exception inheritance |
| `tinymongo[duckdb]` | `duckdb>=1.0.0` | `backend="duckdb"` |
| `tinymongo[parquet]` | `duckdb>=1.0.0`, `pyarrow>=10.0.0` | `backend="parquet"` |
| `tinymongo[postgres]` | `psycopg[binary]>=3.1` | `backend="postgres"` |
| `tinymongo[mysql]` / `tinymongo[mariadb]` | `PyMySQL>=1.1` | `backend="mysql"` / `backend="mariadb"` |
| `tinymongo[remote-sql]` | `psycopg[binary]`, `PyMySQL` | both remote SQL backends |
| `tinymongo[all]` | all of the above | everything |

Selecting a backend whose driver is missing raises `ImportError` with the exact `pip install` command to run. **PyMongo is never a runtime dependency** — it is only used by the optional patch helper, `ObjectId` support, and exception-class inheritance.

The package also installs a `tinymongo` console script (see [Command Line Interface](#command-line-interface)).

---

## Quick Start

```python
from tinymongo import TinyMongoClient

client = TinyMongoClient("./tinydb")        # folder, not a connection string
db = client.shop                            # or client["shop"]
users = db.users                            # or db["users"]

users.insert_one({"name": "Ada", "email": "ada@example.com", "score": 7})
users.insert_many([
    {"_id": 2, "name": "Grace", "email": "grace@example.com", "score": 9},
    {"_id": 3, "name": "Linus", "email": "linus@example.com", "score": 5},
])

users.create_index("email", unique=True)
users.update_one({"name": "Ada"}, {"$inc": {"score": 1}})

for doc in users.find({"score": {"$gte": 8}}).sort("score", -1):
    print(doc)

print(users.count_documents({}))            # 3
print(users.find_one({"name": "Grace"}, {"name": 1, "_id": 0}))
client.close()
```

`TinyMongoClient` supports the context-manager protocol, which is the safest way to guarantee storage is closed:

```python
with TinyMongoClient("./tinydb") as client:
    client.shop.users.insert_one({"name": "Ada"})
```

---

## What Is and Is Not Supported

This is the single most important table in this document. TinyMongo is a subset — assuming full PyMongo parity is the primary source of bugs.

### Supported

| Area | Details |
|------|---------|
| CRUD | `insert_one`, `insert_many`, `find`, `find_one`, `update_one`, `update_many`, `replace_one`, `delete_one`, `delete_many` |
| Find & modify | `find_one_and_update`, `find_one_and_replace`, `find_one_and_delete` |
| Query operators | `$eq`, `$gt`, `$gte`, `$lt`, `$lte`, `$ne`, `$in`, `$nin`, `$all`, `$and`, `$or`, `$nor`, `$not`, `$regex`, `$options`, `$exists` |
| Update operators | `$set`, `$unset`, `$inc`, `$push`, `$pull`, `$addToSet` (plus `upsert=True`) |
| Projections | Inclusion / exclusion, dotted paths, MongoDB `_id` rules |
| Cursors | `sort` (single and multi-key), `skip`, `limit`, `clone`, `rewind`, `close`, `to_list`, `count` |
| Indexes | Durable single-field ascending indexes, `unique=True`, `create_indexes`, `drop_index`, `list_indexes`, `index_information` |
| Counting | `count_documents`, `estimated_document_count`, `distinct` |
| Admin | `list_database_names`, `list_databases`, `drop_database`, `list_collection_names`, `drop_collection` |
| Async | Full non-blocking client/database/collection/cursor API |
| Types | `datetime` everywhere; `ObjectId` with `tinymongo[bson]` |

### Not supported — raises `TinyMongoNotSupportedError`

| Call | Notes |
|------|-------|
| `collection.aggregate(...)` | No aggregation pipelines at all |
| `collection.bulk_write(...)` | No bulk write API |
| `collection.watch(...)`, `db.watch(...)`, `client.watch(...)` | No change streams |
| `client.start_session(...)` | No sessions or transactions |
| `db.command(...)` | No database commands |
| Any method passed `session=<not None>` | Rejected by an internal guard |
| `collection.with_options(...)` with non-default read/write concerns | Default concerns are accepted and ignored |
| `create_index` with descending / compound / `sparse` / TTL options | See [Indexes](#indexes) for what degrades vs. what fails |

### Not supported — **fails silently**

These are the dangerous ones. Unknown query operators do not raise; they simply match nothing:

```python
col.find({"tags": {"$size": 2}})        # -> [] (not an error)
col.find({"score": {"$type": "int"}})   # -> []
col.find({"tags": {"$elemMatch": {...}}})  # -> []
col.find({"score": {"$mod": [2, 1]}})   # -> []
col.find({"$where": "..."})             # -> []
```

Also absent: GridFS (`TinyGridFS` is a stub with no implementation), text search, geospatial queries, `$expr`, `$jsonSchema`, collations, read preferences, and authentication.

---

## Package Exports

```python
from tinymongo import (
    TinyMongoClient,        # native client (folder-based)
    MongoClient,            # PyMongo-shaped client (accepts and ignores URIs)
    TinyMongoDatabase,
    TinyMongoCollection,
    TinyMongoCursor,
    TinyGridFS,             # stub, not implemented
    ASCENDING,              # 1
    DESCENDING,             # -1
    generate_id,            # uuid4 hex string generator
    patch,                  # PyMongo patching context manager / decorator
    TinyMongoUnsupportedWarning,
    AsyncMongoClient,
    AsyncTinyMongoClient,
    AsyncTinyMongoDatabase, AsyncDatabase,
    AsyncTinyMongoCollection, AsyncCollection,
    AsyncTinyMongoCursor, AsyncCursor,
)
```

**Source:** `tinymongo/__init__.py`. Errors live in `tinymongo.errors`; result classes in `tinymongo.results`.

Because `ASCENDING`/`DESCENDING`/`MongoClient` are exported, the drop-in import trick works:

```python
import tinymongo as pymongo

client = pymongo.MongoClient("mongodb://localhost:27017", tinymongo_folder="./tinydb")
rows = list(client.app.users.find({}).sort("score", pymongo.DESCENDING))
```

---

## TinyMongoClient

**Source:** `tinymongo/tinymongo.py`

A client owns a **folder**, not a connection. Each database is one file (or one set of tables) inside that folder.

### Constructor

```python
TinyMongoClient(
    foldername: str = "tinydb",
    backend: str = "tinydb",
    *,
    threads: int | None = None,
    storage_uri: str | None = None,
    duckdb_config: dict | None = None,
    dsn: str | None = None,
)
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `foldername` | `str` | `"tinydb"` | Directory that holds database files. Created with `os.makedirs(..., exist_ok=True)`. For `backend="memory"` this is instead a `memory://NAME` address (or omitted for an isolated namespace). |
| `backend` | `str` | `"tinydb"` | One of `memory`, `tinydb`, `json`, `sqlite`, `duckdb`, `parquet`, `parquetv2`, `postgres`, `postgresql`, `mysql`, `mariadb`. A `None` value falls back to `"tinydb"`. |
| `threads` | `int \| None` | `None` | Passed to DuckDB-backed engines. |
| `storage_uri` | `str \| None` | `os.environ["TINYMONGO_STORAGE_URI"]` | Object-storage prefix for the Parquet backend (`s3://`, `gs://`, `az://`, …). |
| `duckdb_config` | `dict \| None` | `None` | Extra DuckDB connection settings. |
| `dsn` | `str \| None` | env fallback | Connection string for `postgres`/`mysql` backends. Falls back to backend-specific environment variables. |

Unknown backend names raise `ValueError` listing the supported set.

### Database Access

Three equivalent forms; none of them touch storage until a collection is used:

```python
db = client.shop
db = client["shop"]
db = client.get_database("shop")     # extra args accepted and ignored
```

Attribute names starting with `_` raise `AttributeError` rather than creating a database.

### Client Methods

```python
close() -> None
__enter__() / __exit__()                  # context manager
server_info() -> dict
capabilities() -> dict
supports(feature: str) -> bool
list_database_names() -> list[str]
database_names() -> list[str]             # legacy alias
list_databases(*args, **kwargs) -> TinyMongoCursor
drop_database(name_or_database, *args, **kwargs) -> None
get_database(name, *args, **kwargs) -> TinyMongoDatabase
start_session(*args, **kwargs)            # raises TinyMongoNotSupportedError
watch(*args, **kwargs)                    # raises TinyMongoNotSupportedError
```

| Method | Behavior |
|--------|----------|
| `close()` | Closes every opened database. Idempotent. Using the client afterward raises `InvalidOperation("Cannot use a closed TinyMongoClient")`. For unnamed memory clients, also discards the in-memory namespace. |
| `server_info()` | Returns local metadata, **not** MongoDB server info: `{"version": "tinymongo", "storage": <backend>, "localPath": <folder>, "storageUri": ..., "dsnConfigured": bool, "tinymongo": True}`. |
| `list_database_names()` | Scans the folder for files with the backend's extension and returns sorted stems. For remote SQL it queries the metadata table; for memory it lists the namespace. |
| `list_databases()` | Returns a `TinyMongoCursor` over dicts shaped `{"name", "sizeOnDisk", "empty"}`. |
| `drop_database(name)` | Accepts a name or a `TinyMongoDatabase`. Drops every collection, closes the database, and removes the file/directory. Returns `None`; a no-op for unknown names. |

```python
print(client.server_info())
# {'version': 'tinymongo', 'storage': 'tinydb', 'localPath': './tinydb',
#  'storageUri': None, 'dsnConfigured': False, 'tinymongo': True}
```

### Capabilities

`capabilities()` reports what the *configured backend* can actually honor — useful for writing code that adapts to TinyMongo vs. real MongoDB.

```python
{
    "backend": "sqlite",          # normalized backend name
    "persistent": True,           # False only for memory
    "remote_storage": False,      # postgres / mysql / mariadb
    "object_storage": False,      # parquet + storage_uri
    "table_native": True,         # sqlite/duckdb/parquet/postgres/mysql
    "multiprocess_writes": True,  # False for memory and object storage
    "native_indexes": True,       # sqlite, postgres, mysql/mariadb
    "projections": True,
    "bulk_writes": False,
    "aggregation": False,
    "sessions": False,
    "transactions": False,
    "change_streams": False,
    "bson_types": False,          # True when bson/pymongo is importable
}
```

`supports(feature)` returns the boolean for one key and raises `ValueError` for an unknown key or for `"backend"`.

---

## MongoClient (PyMongo Drop-In)

```python
MongoClient(
    host=None,
    port=None,
    document_class=None,
    tz_aware=None,
    connect=None,
    type_registry=None,
    **kwargs,
)
```

`MongoClient` subclasses `TinyMongoClient` and exists so PyMongo-shaped code runs unchanged. **Network arguments are accepted and ignored** — hosts, ports, MongoDB URIs, `serverSelectionTimeoutMS`, and friends never open a socket.

Storage folder resolution (`_folder_from_mongo_client_args`), in precedence order:

1. `tinymongo_folder=` kwarg
2. `tinymongo_path=` kwarg
3. `foldername=` kwarg
4. `TINYMONGO_HOME` environment variable
5. `"tinydb"` if `host` is `None` or looks like a network target
6. otherwise, `host` itself is treated as a local folder path

A host "looks like a network target" if a `port` is given, if `host` is a list/tuple, if it contains `://` or `,`, if it is `localhost`/`127.0.0.1`/`::1`, if it is a bracketed IPv6 literal, or if it contains `:` and is not an absolute path.

```python
from tinymongo import MongoClient

client = MongoClient(
    "mongodb://localhost:27017",     # ignored
    serverSelectionTimeoutMS=2000,   # ignored
    tinymongo_folder="/path/to/data",
    backend="sqlite",                # TinyMongo-specific
)
client.app.users.insert_one({"email": "ada@example.com"})
```

TinyMongo-specific kwargs recognized here: `backend`, `storage_uri`, `threads`, `duckdb_config`, `dsn`.

---

## TinyMongoDatabase

**Source:** `tinymongo/tinymongo.py`

```python
db.name                      # property -> str
db.database                  # same value (legacy attribute)
db.engine                    # table backend engine, or None for JSON/memory
db.tinydb                    # underlying TinyDB instance, or None for engine backends

db.collection_name           # attribute access -> TinyMongoCollection
db["collection_name"]        # item access -> TinyMongoCollection
db.get_collection(name, *args, **kwargs) -> TinyMongoCollection

db.collection_names() -> list[str]
db.list_collection_names() -> list[str]
db.drop_collection(name_or_collection, *args, **kwargs) -> bool
db.close() -> None
db.command(*args, **kwargs)  # raises TinyMongoNotSupportedError
db.watch(*args, **kwargs)    # raises TinyMongoNotSupportedError
```

| Method | Notes |
|--------|-------|
| `collection_names()` | All table names except internal ones prefixed `__tinymongo_`. **Includes TinyDB's `_default` table.** |
| `list_collection_names()` | Same list with `_default` filtered out. Prefer this one. |
| `drop_collection(name)` | Accepts a name or a `TinyMongoCollection`; delegates to `collection.drop()` and returns its `bool`. |

> **Gotcha:** `db.<anything>` always returns a collection handle. There is no such thing as an unknown attribute on a database — `db.typo` silently gives you a collection named `typo`.

---

## TinyMongoCollection

**Source:** `tinymongo/tinymongo.py`

```python
col.name                     # property -> collection name
col.tablename                # same value
col.database                 # parent TinyMongoDatabase
col.full_name                # "dbname.collectionname"
col.write_concern            # compatibility stub
col.read_concern             # compatibility stub
repr(col)                    # -> the collection name
```

### Inserts

```python
insert_one(doc, *args, **kwargs) -> InsertOneResult
insert_many(docs, *args, **kwargs) -> InsertManyResult
insert(docs, *args, **kwargs)                  # legacy: dispatches on list vs dict
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `doc` / `docs` | `dict` / `list[dict]` | — | `insert_one` raises `ValueError('"doc" must be a dict')` for non-dicts; `insert_many` raises `ValueError('"insert_many" requires a list input')` for non-lists. |
| `bypass_document_validation` | `bool` | `False` | Skips user-defined checks on table backends. **Never** bypasses the built-in unique `_id` constraint. |
| `session` | — | `None` | Any non-`None` value raises `TinyMongoNotSupportedError`. |

`_id` handling: if the document has no `_id`, or `_id is None`, one is generated with `generate_id()` — a 32-character `uuid4().hex`-style string, **not** an `ObjectId`. Explicit falsy ids (`0`, `False`, `""`) are respected. Reusing an existing `_id` raises `DuplicateKeyError`.

```python
result = users.insert_one({"name": "Ada"})
result.inserted_id      # 'e391a1949006412cad37fd33a163208d'
result.eid              # underlying TinyDB element id
result.acknowledged     # True

result = users.insert_many([{"_id": 1}, {"_id": 2}])
result.inserted_ids     # [1, 2]
result.eids             # [1, 2]
```

> **The caller's dict is mutated.** `insert_one({"name": "Ada"})` adds `_id` to the dict you passed in. Deep-copy first if that matters.

### Queries

```python
find(
    _filter: dict | None = None,
    projection: dict | list | None = None,
    skip: int | None = None,
    limit: int | None = None,
    *args,
    **kwargs,          # sort=..., filter=..., session=...
) -> TinyMongoCursor

find_one(
    _filter: dict | None = None,
    projection: dict | list | None = None,
    *args,
    **kwargs,          # sort=..., filter=..., session=...
) -> dict | None
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `_filter` | `dict \| None` | `None` | MongoDB-style filter. `None` or `{}` matches everything. Also accepted as the `filter=` keyword. |
| `projection` | `dict \| list \| None` | `None` | Inclusion/exclusion document, or a list of field names to include. See [Projections](#projections). |
| `skip` | `int \| None` | `None` | Non-negative. `TypeError` for non-ints (including `bool`), `ValueError` if negative. |
| `limit` | `int \| None` | `None` | `0` means no limit; negatives are treated as `abs(limit)`. |
| `sort` | keyword | `None` | Same spec as `cursor.sort()` — a `(key, direction)` list or a field name. |

> **Breaking change from older TinyMongo:** the *second positional argument is the projection*, matching PyMongo. Older releases treated it as sort. Pass ordering as `sort=` or call `.sort()` on the cursor.

```python
users.find({"score": {"$gte": 8}})
users.find({}, {"name": 1, "_id": 0})
users.find({}, None, 1, 2)                       # skip 1, limit 2
users.find({}, sort=[("score", -1), ("name", 1)])
users.find_one({"email": "ada@example.com"}, {"password_hash": 0})
```

`find_one` is `find(..., limit=1)` plus `cursor.next()`, returning `None` when nothing matches.

Two internal fast paths are worth knowing: a single-field scalar equality filter on an indexed field is answered from the in-memory index; other filters either compile to a TinyDB query or fall back to a full-scan Python matcher (`requires_python_filter` decides). Both produce the same results — the difference is only speed.

### Updates

```python
update_one(query, doc, *args, **kwargs) -> UpdateResult
update_many(query, doc, *args, **kwargs) -> UpdateResult
replace_one(query, replacement, *args, **kwargs) -> UpdateResult
update(query, doc, *args, **kwargs)        # legacy alias
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `query` | `dict` | — | Filter selecting documents to modify. |
| `doc` | `dict` | — | Update document. **Must** consist entirely of `$` operators. |
| `replacement` | `dict` | — | Full replacement document (`replace_one` only). Its `_id` is forced to the matched document's `_id`. |
| `upsert` | `bool` | `False` | Insert a document built from the filter's equality terms plus the update when nothing matches. |
| `session` | — | `None` | Non-`None` raises `TinyMongoNotSupportedError`. |

As in PyMongo, a non-operator update document is rejected — but with a plain `ValueError`, not `WriteError`:

```python
users.update_one({"name": "Ada"}, {"name": "Ada B"})
# ValueError: update only works with $ operators; use replace_one instead
```

Legacy `update()` dispatches oddly and is best avoided: a list `doc` maps `update_one` over each item, while a dict `doc` calls **`update_many`** (not `update_one`).

```python
r = users.update_one({"name": "Ada"}, {"$inc": {"score": 1}})
r.matched_count      # 1
r.modified_count     # 1 if the document actually changed, else 0
r.upserted_id        # None
r.raw_result         # {'n': 1, 'nModified': 1, 'updatedExisting': True}

r = users.update_one({"name": "Nobody"}, {"$set": {"score": 1}}, upsert=True)
r.matched_count      # 0
r.modified_count     # 0
r.upserted_id        # generated _id
```

Upsert document construction (`_document_for_upsert`): non-`$` keys from the filter are copied in (dotted keys expand into nested documents, and `{"field": {"$eq": v}}` contributes `v`); other operator-valued keys are skipped. Then the update operators are applied.

### Find and Modify

```python
find_one_and_update(query, update, *args, **kwargs) -> dict | None
find_one_and_replace(query, replacement, *args, **kwargs) -> dict | None
find_one_and_delete(query, *args, **kwargs) -> dict | None
```

| Keyword | Type | Default | Description |
|---------|------|---------|-------------|
| `return_document` | `bool` | `False` | `False` = before image (PyMongo's `ReturnDocument.BEFORE`), `True` = after image (`ReturnDocument.AFTER`). A non-bool raises `ValueError`. Silently ignored by `find_one_and_delete`, which always returns the pre-delete document. |
| `projection` | `dict \| list \| None` | `None` | Applied to the returned document. |
| `sort` | list/str | `None` | Chooses which document is acted on when several match. |
| `upsert` | `bool` | `False` | Forwarded to the underlying write (`find_one_and_update` / `find_one_and_replace`). |

Because PyMongo's `ReturnDocument.AFTER` *is* `True` and `BEFORE` *is* `False`, `from pymongo import ReturnDocument` keeps working.

```python
before = users.find_one_and_update({"name": "Grace"}, {"$inc": {"score": 1}})
after = users.find_one_and_update(
    {"name": "Grace"}, {"$inc": {"score": 1}}, return_document=True
)
removed = users.find_one_and_delete({"name": "Temp"}, projection={"_id": 0})
```

When nothing matches and `upsert` is not set, all three return `None`. With `upsert=True` and `return_document=False`, `find_one_and_update` also returns `None` (matching MongoDB).

### Deletes

```python
delete_one(query, *args, **kwargs) -> DeleteResult
delete_many(query, *args, **kwargs) -> DeleteResult
remove(spec_or_id, multi=True, *args, **kwargs)   # legacy alias
```

```python
users.delete_one({"name": "Linus"}).deleted_count    # 0 or 1
users.delete_many({}).deleted_count                  # everything
```

`delete_many({})` also resets the underlying TinyDB element counter so ids stay consistent. Legacy `remove()` maps to `delete_many` when `multi=True` (the default) and `delete_one` otherwise.

### Counting and Distinct

```python
count() -> int                                       # legacy, whole collection
count_documents(filter=None, *args, **kwargs) -> int
estimated_document_count(*args, **kwargs) -> int     # == count_documents({})
distinct(key, filter=None, *args, **kwargs) -> list
```

`distinct` walks matching documents, resolves the dotted `key`, flattens list values, and de-duplicates using **type-sensitive** comparison (`type(a) is type(b) and a == b`), so `1` and `True` are kept as separate values. Results are returned in first-seen order, not sorted.

```python
users.count_documents({"score": {"$gte": 8}})
users.distinct("tags")                    # flattens list-valued fields
users.distinct("profile.city", {"active": True})
```

### Collection Management

```python
drop(**kwargs) -> bool
```

Drops the table and removes its index catalog entries. Returns `True` on success, `False` if the collection does not exist. `writeConcern` and similar kwargs are accepted and ignored; `session` is rejected.

### Unsupported Collection Methods

```python
col.aggregate(...)     # TinyMongoNotSupportedError: Aggregation pipelines are not supported
col.bulk_write(...)    # TinyMongoNotSupportedError: Bulk writes are not supported
col.watch(...)         # TinyMongoNotSupportedError: Change streams are not supported
col.with_options(...)  # returns self; raises only for non-default read/write concerns
```

> **Serious gotcha:** any *other* unknown attribute on a collection returns the collection itself, because `TinyMongoCollection.__getattr__` is used to lazily build the table. A misspelled method does not raise `AttributeError` — you get `TypeError: 'TinyMongoCollection' object is not callable` at call time, or silent nonsense if you only read the attribute.

```python
users.no_such_method is users      # True
users.no_such_method({"a": 1})     # TypeError: 'TinyMongoCollection' object is not callable
```

---

## Query Operators

**Source:** `tinymongo/table_backends.py` (`matches_filter`, `_field_matches`) and `TinyMongoCollection.parse_condition`

| Operator | Supported | Behavior notes |
|----------|-----------|----------------|
| implicit equality | yes | Also matches **members** of array fields: `{"tags": "b"}` matches `{"tags": ["a", "b"]}`. Exact array equality (`{"tags": ["a","b"]}`) also works. |
| `$eq` | yes | Same array-member semantics as implicit equality. |
| `$gt`, `$gte`, `$lt`, `$lte` | yes | Applied to each member for array fields; incomparable types are skipped rather than raising. Missing fields never match. |
| `$ne` | yes | Missing fields **do** match `$ne` (MongoDB semantics). |
| `$in` | yes | Value or list; matches array members too. |
| `$nin` | yes | Value or list. |
| `$all` | yes | Requires the field to be a list containing every value. |
| `$exists` | yes | Truthy/falsy operand. |
| `$regex` | yes | Python `re.search` semantics, applied to string values (including array members). |
| `$options` | yes | `i`, `m`, `s`, `x` flags. Only valid alongside `$regex`; alone it matches nothing. |
| `$not` | yes | Negates a nested operator document, or a scalar as `$eq`. |
| `$and`, `$or`, `$nor` | yes | Top-level list operators. |
| dotted paths | yes | `{"profile.city": "NYC"}` |
| embedded document equality | yes | `{"meta": {"k": 1}}` compares the whole subdocument. |
| `$size`, `$type`, `$elemMatch`, `$mod`, `$where`, `$expr`, `$text`, `$near`, `$jsonSchema` | **no** | **Match nothing and raise nothing.** |

```python
col.find({"score": {"$gt": 6}})
col.find({"score": {"$in": [5, 9]}})
col.find({"tags": {"$all": ["a", "b"]}})
col.find({"name": {"$regex": "^a", "$options": "i"}})
col.find({"$or": [{"score": 5}, {"score": 9}]})
col.find({"$and": [{"score": {"$gte": 5}}, {"score": {"$lte": 7}}]})
col.find({"missing": {"$exists": False}})
col.find({"score": {"$not": {"$gt": 6}}})
col.find({"profile.city": "NYC"})
```

---

## Update Operators

**Source:** `_apply_update_document` in `tinymongo/tinymongo.py`

| Operator | Behavior |
|----------|----------|
| `$set` | Sets a value at a dotted path, creating intermediate dicts as needed. |
| `$unset` | Removes a key at a dotted path. **See the warning below.** |
| `$inc` | Adds to the current value; treats a missing field as `0`. |
| `$push` | Appends to a list; treats a missing field as `[]`. `ValueError` if the target exists and is not a list. |
| `$pull` | Removes every element equal to the value. `ValueError` for non-list targets. |
| `$addToSet` | Appends only if not already present. `ValueError` for non-list targets. |
| anything else | `ValueError: Unsupported update operator: $rename` |

Every operator value must be a dict, or `ValueError("<op> update requires a dict")` is raised. `_id` is always preserved from the original document.

> ### `$unset` does not remove fields on the JSON and memory backends
>
> This is a real, verified defect worth designing around. On the default `tinydb`/`json` backend and on the `memory` backend, `$unset` reports `modified_count == 1` but **the field is still present**, before and after reopening the database. The cause is TinyDB's `Table.update(fields, cond)`, which merges keys via `dict.update()` and cannot delete them.
>
> ```python
> col.update_one({"_id": 1}, {"$unset": {"score": ""}})
> col.find_one({"_id": 1})     # tinydb/memory: {'_id': 1, 'name': 'Ada', 'score': 7}  <-- still there
>                              # sqlite/duckdb/parquet/postgres/mysql: field is gone
> ```
>
> Workarounds: use a table-native backend, or replace the whole document:
>
> ```python
> doc = col.find_one({"_id": 1})
> doc.pop("score", None)
> col.replace_one({"_id": 1}, doc)
> ```

`replace_one`, by contrast, genuinely drops absent fields on every backend, since the document is removed and reinserted.

```python
col.update_one({"_id": 1}, {"$set": {"profile.city": "NYC"}})
col.update_one({"_id": 1}, {"$inc": {"score": 1}})
col.update_one({"_id": 1}, {"$push": {"tags": "c"}})
col.update_one({"_id": 1}, {"$addToSet": {"tags": "c"}})     # no-op if present
col.update_one({"_id": 1}, {"$pull": {"tags": "a"}})
```

---

## Projections

**Source:** `tinymongo/projection.py`

Projections are normalized once (`normalize_projection`) and applied after filtering and sorting (`project_document`). Both a mapping and a plain list of field names are accepted.

| Rule | Behavior |
|------|----------|
| Inclusion | `{"name": 1, "email": 1}` returns only those fields plus `_id`. |
| Exclusion | `{"password_hash": 0}` returns everything else. |
| Mixing | `OperationFailure` (codes 31253/31254) — except `_id`, which may always be toggled. |
| `_id` | Included by default in inclusion projections; `{"_id": 0}` removes it. |
| Dotted paths | `{"profile.email": 1}` projects nested fields; nested exclusion works too. |
| Nested mappings | `{"profile": {"email": 1}}` is flattened to `profile.email`. |
| Arrays | Inclusion recurses into list members that are dicts/lists; scalars in arrays are dropped. |
| Path collisions | `{"a": 1, "a.b": 1}` raises `OperationFailure` (codes 31249/31250). |
| `$` expressions, `$meta`, positional `$`, numeric array indexes | `TinyMongoNotSupportedError`. |
| Non-numeric flags | `TinyMongoNotSupportedError`. |
| Empty projection `{}` | Treated as no projection. |

```python
users.find({"active": True}, {"profile.email": 1, "_id": 0})
users.find_one({"email": "ada@example.com"}, {"password_hash": 0})
users.find({}, ["name", "email"])                 # list form = inclusion
```

---

## TinyMongoCursor

**Source:** `tinymongo/tinymongo.py`

Cursors are **eager**: the query has already run and results are held in memory. Sorting, skipping, and limiting operate on that in-memory list.

```python
cursor.sort(key_or_list, direction=None) -> TinyMongoCursor
cursor.skip(n: int) -> TinyMongoCursor
cursor.limit(n: int) -> TinyMongoCursor
cursor.paginate(skip, limit) -> TinyMongoCursor
cursor.clone() -> TinyMongoCursor
cursor.rewind() -> TinyMongoCursor
cursor.close() -> None
cursor.to_list(length=None) -> list[dict]
cursor.next() -> dict            # raises StopIteration when exhausted
cursor.has_next() -> bool
cursor.hasNext() -> bool
cursor.count(with_limit_and_skip=False) -> int
cursor.alive -> bool             # property
cursor[i]                        # index into the current window
iter(cursor) / next(cursor)
```

| Method | Notes |
|--------|-------|
| `sort` | `sort("field")` defaults to ascending; `sort("field", -1)` for descending; `sort([("a", 1), ("b", -1)])` for multi-key. Passing both a list *and* a direction raises `ValueError`. Malformed pairs raise `TypeError`. Sorting re-sorts the full source data, then reapplies the skip/limit window. |
| `skip` / `limit` | `TypeError` for non-int (booleans rejected), `ValueError` for negative skip. `limit(0)` means **no limit**; negative limits use `abs(n)`. |
| `count` | Returns the length of the current window. **The `with_limit_and_skip` argument is ignored** — skip/limit are always reflected. |
| `to_list` | Consumes from the current position and advances it. `to_list(2)` then `to_list()` returns the remainder. Returns `[]` on a closed cursor. |
| `clone` | Re-runs the original query against the collection when possible, preserving projection, sort spec, skip, and limit. Independent iteration position. |
| `rewind` | Resets position to the start; raises `InvalidOperation` on a closed cursor. |
| `close` | Marks the cursor closed: `alive` becomes `False`, iteration yields nothing, `to_list()` returns `[]`. |

```python
cursor = users.find({}).sort("score", -1).skip(1).limit(2)
print(cursor.count())          # 2 — the windowed count
for doc in cursor:
    print(doc)

page = users.find({}).sort("_id", 1).to_list(20)
```

### Sort ordering across mixed types

`_order` assigns a type rank so heterogeneous values remain sortable:

| Rank | Types |
|------|-------|
| 0 | `None`, missing fields, and any unsupported/unsortable type |
| 1 | `int`, `float` |
| 2 | `str` |
| 3 | `dict` (compared key-by-key) |
| 4 | nested `list` (only when comparing inside another container) |
| 5 | `bool` |

Arrays compare by their **smallest** member ascending and their **largest** member descending, matching MongoDB. Missing fields sort as `None`, i.e. first ascending. Note that `bool` sorts *after* everything else here — MongoDB's exact BSON ordering differs in the details, so do not rely on cross-type ordering being identical to a real server.

---

## Indexes

**Source:** `tinymongo/indexes.py`

TinyMongo supports **durable single-field ascending equality indexes**, optionally unique. Metadata is stored in a `__tinymongo_indexes` catalog and survives restarts on persistent backends. SQLite, PostgreSQL, and MariaDB/MySQL additionally create real database indexes.

```python
create_index(key, *args, **kwargs) -> str          # returns the effective index name
create_indexes(indexes, *args, **kwargs) -> list[str]
drop_index(key) -> None
list_indexes() -> list[dict]
index_information() -> dict
```

### `create_index`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `key` | `str \| tuple \| list` | — | `"email"`, `("email", 1)`, or `[("email", 1)]`. Only **one** field, only direction `1`. |
| `name` | `str` | `"<field>_1"` | Index name. Reserved names `_id` / `_id_` rejected for other fields. |
| `unique` | `bool` | `False` | Enforced on every write via a full post-image validation. |

Returns the effective index name (PyMongo-compatible). Passing extra positional args, a descending/compound key, or any option other than `name`/`unique` raises `TinyMongoNotSupportedError`.

```python
users.create_index("email")                                   # -> 'email_1'
users.create_index("email", unique=True, name="login_email")  # -> 'login_email'
users.create_index("_id")                                     # -> '_id_' (no-op)
users.create_index("_id", unique=True)                        # OperationFailure (code 197)
users.create_index([("score", -1)])                           # TinyMongoNotSupportedError
users.create_index("a", sparse=True)                          # TinyMongoNotSupportedError
users.drop_index("login_email")                               # by name or by field
users.drop_index("_id_")                                      # OperationFailure
```

Re-creating an identical index returns the existing name. Re-creating the same *name* with different options raises `OperationFailure("An index with the same name or key has different options")`.

### `create_indexes` and degradation

`create_indexes` accepts PyMongo `IndexModel` objects (duck-typed via `.document`) or plain mappings, and returns the requested names in order. Declarations that TinyMongo cannot honor exactly are **degraded with a `TinyMongoUnsupportedWarning`** instead of failing:

| Requested | Outcome | Warning text fragment |
|-----------|---------|-----------------------|
| compound key | leading ascending field only | "compound indexing is reduced to its ascending leading field" |
| descending (`-1`) | treated as ascending | "descending direction is treated as ascending" |
| `hashed` | ascending equality index | "hashed indexing is replaced by ascending equality indexing" |
| `text` | **skipped entirely** (no index created) | "text indexing is ignored because `$text` queries are not supported" |
| `sparse=True` | index created, sparseness ignored | "sparse membership is not honored" |
| `expireAfterSeconds` | index created, no TTL expiry | "TTL expiration is not performed" |
| `background=True` | ignored | "background creation is ignored" |

Crucially, a **unique** index combined with compound/hashed/sparse/text/TTL raises `TinyMongoNotSupportedError` before anything is created — degrading it would silently weaken a correctness constraint.

```python
from pymongo import IndexModel, ASCENDING

users.create_indexes([IndexModel([("email", ASCENDING)], unique=True, name="email_uq")])
# -> ['email_uq']

import warnings
with warnings.catch_warnings(record=True) as caught:
    warnings.simplefilter("always")
    users.create_indexes([
        {"key": [("score", 1)]},
        {"key": [("name", -1)], "name": "name_desc"},     # warns: descending
        {"key": [("a", 1), ("b", 1)], "name": "compound"},# warns: compound
    ])
```

### Reading index metadata

```python
users.list_indexes()
# [{'name': '_id_', 'key': [('_id', 1)]},
#  {'name': 'email_uq', 'key': [('email', 1)], 'unique': True}]

users.index_information()
# {'_id_': {'key': [('_id', 1)]},
#  'email_uq': {'key': [('email', 1)], 'unique': True}}
```

Note `list_indexes()` returns a plain **list**, not a `CommandCursor` as in PyMongo. (The async version returns an `AsyncTinyMongoCursor`.)

### Unique constraint semantics

Uniqueness is validated over the whole post-write image using deterministic tokens (`index_tokens`): a missing field and an explicit `None` both produce the same `null:` token, list values produce one token per member (multikey), and `1` / `1.0` / `True` are distinguished by type. Values that cannot be tokenized — dicts, nested arrays, non-finite floats — raise `TinyMongoNotSupportedError`.

> **Trap:** creating a unique index on a field that several existing documents lack raises `DuplicateKeyError`, because they all tokenize to `null:`. This matches MongoDB, but is easy to hit when adding a unique index to an existing collection.

```python
users.create_index("email", unique=True)
users.insert_one({"email": "ada@example.com"})
users.insert_one({"email": "ada@example.com"})
# DuplicateKeyError: duplicate key for unique index email_1: documents 1 and 2
```

On PostgreSQL and MariaDB/MySQL, unique indexes **reject array-valued keys** (`TinyMongoNotSupportedError`) because native scalar constraints cannot guarantee multikey uniqueness across processes.

---

## Result Classes

**Source:** `tinymongo/results.py`

```python
class InsertOneResult:
    inserted_id      # the document's _id
    eid              # underlying TinyDB element id
    acknowledged     # always True

class InsertManyResult:
    inserted_ids     # list of _id values, in input order
    eids             # list of TinyDB element ids
    acknowledged

class UpdateResult:
    raw_result       # dict for matched updates, list of ids otherwise
    matched_count
    modified_count
    upserted_id      # None unless an upsert inserted a document
    acknowledged

class DeleteResult:
    raw_result
    deleted_count
    acknowledged
```

`matched_count` / `modified_count` fall back to `len(raw_result)` when not set explicitly, and `deleted_count` is `len(raw_result)` for list results. `acknowledged` is a compatibility constant.

---

## Exceptions

**Source:** `tinymongo/errors.py`

When PyMongo is installed, every TinyMongo exception **also inherits from the matching PyMongo class**, so existing `except PyMongoError:` handlers keep working. Without PyMongo, dependency-free stand-ins are used.

```
TinyMongoError                     (-> pymongo.errors.PyMongoError)
├── ConnectionFailure              (-> pymongo.errors.ConnectionFailure)
├── ConfigurationError             (-> pymongo.errors.ConfigurationError)
├── OperationFailure               (-> pymongo.errors.OperationFailure)
│   ├── CursorNotFound
│   └── WriteError
│       └── DuplicateKeyError      (-> pymongo.errors.DuplicateKeyError)
├── InvalidOperation               (-> pymongo.errors.InvalidOperation)
├── TinyMongoNotSupportedError     (also a NotImplementedError)
└── StorageError
    ├── StorageCorruptionError
    └── LockError
```

```python
from pymongo.errors import PyMongoError
from tinymongo.errors import DuplicateKeyError, TinyMongoNotSupportedError

try:
    users.insert_one({"_id": 1})
except PyMongoError as exc:       # catches TinyMongo's DuplicateKeyError
    ...
```

> Not every failure is a TinyMongo exception. Invalid update documents and bad cursor arguments raise plain `ValueError` / `TypeError`.

| Exception | Raised when |
|-----------|-------------|
| `DuplicateKeyError` | Reused `_id`, or a unique index violation. |
| `OperationFailure` | Projection conflicts, incompatible index redefinition, dropping `_id_`, unique on `_id`. |
| `InvalidOperation` | Using a closed client or cursor. |
| `TinyMongoNotSupportedError` | Sessions, aggregation, bulk writes, change streams, DB commands, unsupported index/projection features. |
| `StorageCorruptionError` | An existing database file cannot be decoded. |
| `ValueError` | Non-operator update documents, unknown update operators, bad sort specs, unknown backends. |

---

## Async API

**Source:** `tinymongo/asyncio.py`

The async layer wraps the synchronous implementation and runs every potentially blocking call through `asyncio.to_thread`. Database and collection selection stay immediate (as in PyMongo), and `find()` returns a **lazy** cursor — no storage work happens until it is awaited or iterated.

### AsyncMongoClient / AsyncTinyMongoClient

```python
AsyncMongoClient(*args, **kwargs)        # PyMongo-shaped (host/URI accepted, ignored)
AsyncTinyMongoClient(*args, **kwargs)    # native (folder-first)
```

Constructor arguments are forwarded verbatim to `MongoClient` / `TinyMongoClient`, and the underlying sync client is created lazily on first use.

```python
client[name] / client.name / client.get_database(name)   -> AsyncTinyMongoDatabase   (sync, immediate)

await client.list_database_names() -> list[str]
await client.list_databases(...)   -> AsyncTinyMongoCursor
await client.database_names()      -> list[str]
await client.drop_database(name_or_database, ...) -> None
await client.server_info()         -> dict
await client.capabilities()        -> dict
await client.supports(feature)     -> bool
await client.watch(...)            # raises TinyMongoNotSupportedError
client.start_session(...)          # raises synchronously
await client.close()
async with client: ...             # __aexit__ shields close() from cancellation
```

```python
import asyncio
from tinymongo import AsyncMongoClient

async def main():
    async with AsyncMongoClient(tinymongo_folder="./tinydb", backend="sqlite") as client:
        users = client.app.users
        await users.insert_one({"_id": 1, "name": "Ada", "score": 9})
        return await users.find({}).sort("score", -1).to_list(length=None)

asyncio.run(main())
```

Using a closed async client raises `InvalidOperation("Cannot use a closed AsyncTinyMongoClient")`. `close()` waits for in-flight calls before shutting storage down.

### AsyncTinyMongoDatabase

Aliased as `AsyncDatabase`.

```python
db.name / db.database                        # str
db[name] / db.name / db.get_collection(name) -> AsyncTinyMongoCollection

await db.collection_names() -> list[str]
await db.list_collection_names() -> list[str]
await db.drop_collection(name_or_collection, ...) -> bool
await db.command(...)                        # raises TinyMongoNotSupportedError
await db.watch(...)                          # raises TinyMongoNotSupportedError
await db.close()
async with db: ...
```

### AsyncTinyMongoCollection

Aliased as `AsyncCollection`. Every method below is a coroutine except `find` and `with_options`.

```python
col.find(filter=None, projection=None, skip=None, limit=None, *, sort=None) -> AsyncTinyMongoCursor  # NOT awaited

await col.find_one(...)                await col.insert_one(...)
await col.insert_many(...)             await col.insert(...)
await col.update_one(...)              await col.update_many(...)
await col.update(...)                  await col.replace_one(...)
await col.find_one_and_update(...)     await col.find_one_and_replace(...)
await col.find_one_and_delete(...)     await col.delete_one(...)
await col.delete_many(...)             await col.remove(...)
await col.count(...)                   await col.count_documents(...)
await col.estimated_document_count()   await col.distinct(key, filter=None)
await col.create_index(...)            await col.create_indexes(...)
await col.drop_index(...)              await col.list_indexes()   -> AsyncTinyMongoCursor
await col.index_information()          await col.drop(...)
await col.aggregate(...)               # raises TinyMongoNotSupportedError
await col.bulk_write(...)              # raises TinyMongoNotSupportedError
await col.watch(...)                   # raises TinyMongoNotSupportedError
col.with_options(...)                  # sync; returns self

col.name / col.tablename / col.full_name / col.database
col.write_concern / col.read_concern
```

Note that `list_indexes()` here returns an `AsyncTinyMongoCursor`, so it takes a double await to get a list:

```python
indexes = await (await users.list_indexes()).to_list()
```

### AsyncTinyMongoCursor

Aliased as `AsyncCursor`. Configuration methods are synchronous and chainable; consumption is asynchronous.

```python
cursor.sort(key_or_list, direction=None) -> AsyncTinyMongoCursor    # sync
cursor.skip(count) -> AsyncTinyMongoCursor                          # sync
cursor.limit(count) -> AsyncTinyMongoCursor                         # sync
cursor.paginate(skip, limit) -> AsyncTinyMongoCursor                # sync
cursor.clone() -> AsyncTinyMongoCursor                              # sync
cursor.alive -> bool                                                # property

await cursor.to_list(length=None) -> list[dict]
await cursor.next() -> dict                # raises StopAsyncIteration
await cursor.try_next() -> dict | None
await cursor.has_next() / await cursor.hasNext() -> bool
await cursor.count(with_limit_and_skip=False) -> int
await cursor.rewind() -> AsyncTinyMongoCursor
await cursor.close() -> None
async for doc in cursor: ...
```

Sort specifications are validated eagerly at configuration time (by reusing the sync cursor's validation), so malformed specs fail immediately. Mutating a cursor after iteration begins raises `InvalidOperation("Cannot modify a cursor after it has started")`. The load task is shielded, so cancelling one waiter never duplicates or cancels the underlying query.

```python
async for doc in users.find({"score": {"$gte": 8}}).sort("score", -1).limit(10):
    print(doc)

docs = await users.find({}).skip(20).limit(20).to_list(length=None)
```

---

## Storage Backends

### Backend Selection

**Source:** `tinymongo/storage_backends.py`

| Backend name(s) | Dependency | File extension | Table-native | Native indexes |
|-----------------|------------|----------------|--------------|----------------|
| `memory` | none | *(none)* | no | no |
| `tinydb`, `json` *(default)* | TinyDB | `.json` | no | no |
| `sqlite` | stdlib | `.sqlite` | yes | yes |
| `duckdb` | `duckdb` | `.duckdb` | yes | no |
| `parquet`, `parquetv2` | `duckdb`, `pyarrow` | `.parquet` (a directory) | yes | no |
| `postgres`, `postgresql` | `psycopg` | *(none — server tables)* | yes | yes |
| `mysql`, `mariadb` | `PyMySQL` | *(none — server tables)* | yes | yes |

```python
TinyMongoClient(backend="memory")
TinyMongoClient("/path/to/folder")                       # tinydb JSON, the default
TinyMongoClient("/path/to/folder", backend="sqlite")
TinyMongoClient("/path/to/folder", backend="duckdb")
TinyMongoClient("/path/to/folder", backend="parquet")
TinyMongoClient(backend="postgres", dsn="postgresql://user:pw@localhost:5432/tinymongo")
TinyMongoClient(backend="mariadb", dsn="mysql://user:pw@localhost:3306/tinymongo")
```

An unsupported name raises `ValueError`; a supported name with a missing driver raises `ImportError` carrying the right `pip install` command.

### Memory Backend

Process-local storage that creates **no files** — ideal for tests and scratch data.

```python
from tinymongo import TinyMongoClient

client = TinyMongoClient(backend="memory")       # isolated, anonymous namespace
client.app.users.insert_one({"name": "Ada"})
assert client.app.users.count_documents({}) == 1
client.close()                                   # namespace discarded
```

Named namespaces let clients in the **same process** share data, and survive client close until the process exits:

```python
writer = TinyMongoClient("memory://shared-test", backend="memory")
writer.app.users.insert_one({"name": "Grace"})
writer.close()

reader = TinyMongoClient("memory://shared-test", backend="memory")
assert reader.app.users.find_one({"name": "Grace"}) is not None
```

Address rules: no `://` means an isolated anonymous namespace; a `memory://NAME` address must use only alphanumerics, `.`, `_`, and `-` (otherwise `ValueError`). Memory data never persists across restarts and is never safe to share between processes. The CLI intentionally omits this backend.

### Table-Native Backends

`sqlite`, `duckdb`, and `parquet` store **one real table or file per collection**, instead of one serialized blob. Supported Mongo-style filters compile to SQL over the `_id` column and the JSON document payload; anything the compiler cannot express falls back to the Python matcher, so results are identical either way. Older blob-format SQLite and DuckDB files are migrated to collection tables automatically when opened.

### Object Storage (Parquet)

The Parquet backend can write to S3-compatible object storage:

```python
client = TinyMongoClient(
    "/local/fallback-folder",
    backend="parquet",
    storage_uri="s3://my-bucket/tinymongo",
)
```

Recognized URI schemes: `s3`, `gs`, `gcs`, `az`, `azure`, `abfs`, `abfss`. This is **experimental in 1.2.0** and uses one Parquet file per collection, so updates and deletes rewrite that whole file. `capabilities()["multiprocess_writes"]` is `False` in this mode.

### Remote SQL Backends

PostgreSQL and MariaDB/MySQL keep the same collection API against server-side tables named `<database>__<collection>`, each with a `_id` primary key and a `data` JSON/JSONB payload, plus a `__tinymongo_collections` metadata table. The `path`/`foldername` argument is ignored.

```python
client = TinyMongoClient(
    backend="postgres",
    dsn="postgresql://user:password@localhost:5432/tinymongo",
)
client.app.users.insert_one({"_id": "ada", "name": "Ada"})
```

`_id` lookups use the SQL primary key; other filters currently read documents and apply the Python matcher. Unique indexes become native database indexes but reject array-valued keys.

---

## Patching PyMongo in Tests

**Source:** `tinymongo/patching.py`

```python
patch(folder: str | None = None, backend: str = "memory")
```

`tinymongo.patch()` temporarily replaces `pymongo.MongoClient` and `pymongo.AsyncMongoClient` so application code under test writes to TinyMongo instead. It works as a context manager *and* as a decorator. Requires `pip install "tinymongo[pymongo]"`.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `folder` | `str \| None` | `None` | Storage folder. When `None` and `backend="memory"`, a unique anonymous namespace is created and discarded on exit. When `None` with any other backend, `"tinydb"` is used. |
| `backend` | `str` | `"memory"` | Any supported backend name. |

```python
import pymongo
import tinymongo

with tinymongo.patch():
    writer = pymongo.MongoClient("mongodb://ignored")
    reader = pymongo.MongoClient()
    writer.app.users.insert_one({"name": "Ada"})
    assert reader.app.users.count_documents({}) == 1
# pymongo.MongoClient is restored, even if the block raised
```

As a decorator, including on `unittest` methods:

```python
@tinymongo.patch(folder="./test-data", backend="sqlite")
def test_application_code():
    client = pymongo.MongoClient()
    assert client.server_info()["storage"] == "sqlite"


class ApplicationTest(unittest.TestCase):
    @tinymongo.patch()
    def test_application_code(self):
        client = pymongo.MongoClient()
        client.app.users.insert_one({"name": "Ada"})
        self.assertEqual(client.app.users.count_documents({}), 1)
```

Rules and limits:

- All clients created **inside** one scope share the same storage; clients are closed automatically on exit.
- Code must look up `pymongo.MongoClient` while the scope is active. A name imported directly (`from pymongo import MongoClient`) before patching cannot be replaced.
- Scopes may nest but must exit in nested order (`RuntimeError` otherwise), and cannot overlap across threads or asyncio tasks — PyMongo's module attribute is process-global.
- **Decorating an async function raises `TypeError`.** Put a `with tinymongo.patch():` block *inside* the coroutine, or prefer `async with tinymongo.patch():` so async clients can be awaited during cleanup.

```python
async def test_async_code():
    async with tinymongo.patch():
        client = pymongo.AsyncMongoClient()
        await client.app.users.insert_one({"name": "Async"})
        assert await client.app.users.count_documents({}) == 1
```

---

## BSON Types: ObjectId and datetime

**Source:** `tinymongo/bson_codec.py`

Storage stays plain JSON. Values JSON cannot represent are tagged at the persistence boundary and restored on read, so callers keep working with the original Python objects.

| Type | Requirement | Storage form |
|------|-------------|--------------|
| `datetime` | none | `{"__tinymongo_type_v1__": "datetime", "value": "<isoformat>"}` |
| `bson.ObjectId` | `pip install "tinymongo[bson]"` | `{"__tinymongo_type_v1__": "objectid", "value": "<hex>"}` |

```python
from datetime import datetime
from bson import ObjectId

oid = ObjectId()
users.insert_one({"_id": oid, "created": datetime(2026, 7, 28, 12, 30)})

doc = users.find_one({"_id": oid})
type(doc["_id"])       # <class 'bson.objectid.ObjectId'>
type(doc["created"])   # <class 'datetime.datetime'>

users.count_documents({"created": {"$gt": datetime(2020, 1, 1)}})   # works
```

A user mapping that happens to have exactly the two marker keys is escaped so it cannot be misdecoded. Reading a stored `ObjectId` without `bson` installed raises `ImportError` with the install command. Filters containing `datetime`/`ObjectId` values automatically take the Python-matcher path so comparisons stay correct.

`tinymongo.serializers.DateTimeSerializer` is a legacy TinyDB serializer (requires `tinymongo[serialization]`); it is not needed for the tagging above.

---

## Concurrency and Locking

- Writes use **atomic temp-file replace** plus `fsync` on the file and its directory.
- Cross-process advisory locking uses `portalocker` (a required dependency) via a `.tinymongo.lock` file in the storage directory, with a 30-second timeout; thread-level re-entrancy is handled by a per-path `RLock`.
- File-backed deletes and find-and-modify operations hold the collection lock across the entire read/write transaction; unique-constraint preflight and the write are covered by the same lock.
- The JSON storage layer merges concurrent writers' tables by `_id` so a second process's inserts are not clobbered.
- The memory backend serializes on a per-namespace lock and bumps a revision counter that invalidates other clients' cached tables.
- Integration stress tests are opt-in: `pytest -m integration`, tunable with `TINYMONGO_INTEGRATION_PROCS`, `TINYMONGO_INTEGRATION_WRITES_PER_PROC`, and `TINYMONGO_INTEGRATION_BACKEND`.

None of this makes TinyMongo transactional. There are no sessions, no multi-document atomicity, and no isolation levels.

---

## Command Line Interface

**Source:** `tinymongo/cli.py`. Installed as the `tinymongo` console script.

```bash
tinymongo inspect PATH [--backend B] [--storage-uri URI] [--dsn DSN] [-o OUT]
tinymongo list-dbs PATH [--backend B] [--storage-uri URI] [--dsn DSN]
tinymongo list-collections PATH DATABASE [--backend B] ...
tinymongo export PATH DATABASE COLLECTION [--backend B] [-o OUT]
tinymongo import PATH DATABASE COLLECTION INPUT [--mode append|replace] [--backend B]
tinymongo migrate SOURCE TARGET --to-backend B [--from-backend B]
                  [--source-uri URI] [--target-uri URI]
                  [--source-dsn DSN] [--target-dsn DSN] [--database NAME] [-o OUT]
```

| Command | Behavior |
|---------|----------|
| `inspect` | Prints JSON with every database, its collections, and document counts. |
| `list-dbs` | One database name per line. |
| `list-collections` | One collection name per line (uses `collection_names()`, so `_default` may appear). |
| `export` | Writes a JSON array of documents. `-o -` (the default) writes to stdout. |
| `import` | Reads a JSON array; `--mode replace` clears the collection first, `append` (default) adds. Input `-` reads stdin. Non-array or non-object input exits with an error. |
| `migrate` | Copies every collection (or just `--database NAME`) between backends, **replacing** target collections, then prints a JSON summary. |

`--backend` choices are `tinydb`, `json`, `parquet`, `parquetv2`, `sqlite`, `duckdb`, `postgres`, `postgresql`, `mysql`, `mariadb`. The `memory` backend is deliberately excluded because each CLI invocation exits immediately.

```bash
tinymongo inspect ./tinydb
tinymongo export ./tinydb my_db users -o users.json
tinymongo import ./tinydb my_db users users.json --mode replace
tinymongo migrate ./tinydb ./sqlite-db --to-backend sqlite
tinymongo inspect ./local-cache --backend parquet --storage-uri s3://my-bucket/tinymongo
tinymongo migrate ./tinydb ./unused --to-backend postgres --target-dsn "$TINYMONGO_POSTGRES_DSN"
```

---

## Environment Variables

| Variable | Used for |
|----------|----------|
| `TINYMONGO_HOME` | Default storage folder for `MongoClient` when no explicit folder kwarg is given. |
| `TINYMONGO_STORAGE_URI` | Default `storage_uri` for the Parquet backend. |
| `TINYMONGO_POSTGRES_DSN`, `TINYMONGO_POSTGRESQL_DSN`, `DATABASE_URL` | PostgreSQL DSN fallbacks, in that order. |
| `TINYMONGO_MYSQL_DSN`, `TINYMONGO_MARIADB_DSN`, `MYSQL_URL`, `MARIADB_URL` | MariaDB/MySQL DSN fallbacks, in that order. |
| `TINYMONGO_INTEGRATION_PROCS`, `TINYMONGO_INTEGRATION_WRITES_PER_PROC`, `TINYMONGO_INTEGRATION_SINGLE_PROCS`, `TINYMONGO_INTEGRATION_SINGLE_WRITES_PER_PROC`, `TINYMONGO_INTEGRATION_BACKEND` | Concurrency stress-test tuning. |
| `TINYMONGO_MONGODB_URI` | Selects a real MongoDB server for the development compatibility suite. |

---

## Gotchas and Behavioral Differences

Ranked roughly by how likely each one is to bite.

1. **Unknown query operators match nothing and raise nothing.** `$size`, `$type`, `$elemMatch`, `$mod`, `$where`, `$expr`, `$text` all return an empty result set. There is no error to catch.
2. **`$unset` does not remove fields on the `tinydb`/`json` and `memory` backends** even though `modified_count` reports success. Use `replace_one` or a table-native backend.
3. **Misspelled collection methods do not raise `AttributeError`.** `TinyMongoCollection.__getattr__` returns `self`, so a typo surfaces later as `TypeError: 'TinyMongoCollection' object is not callable`.
4. **`find()`'s second positional argument is the projection**, not sort. Code written against older TinyMongo that passed a sort spec there is now silently wrong (or raises a projection error).
5. **Generated `_id` values are 32-char UUID hex strings, not `ObjectId`s.** Code doing `ObjectId(doc["_id"])` will fail.
6. **`insert_one`/`insert_many` mutate the dicts you pass in**, adding `_id`.
7. **No aggregation, no bulk writes, no transactions, no change streams, no GridFS.** `TinyGridFS` exists but is an empty stub.
8. **`cursor.count()` ignores `with_limit_and_skip`** — skip and limit are always applied.
9. **Cursors are eager and in-memory.** `find()` materializes results; there is no server-side streaming and no memory protection on large collections.
10. **`collection_names()` includes TinyDB's `_default` table**; `list_collection_names()` does not. Prefer the latter.
11. **Only single-field ascending indexes are real.** `create_index` rejects anything else outright; `create_indexes` degrades it with a warning — a "compound unique" index you asked for may not exist as you expect (and unique + compound is rejected outright).
12. **A unique index on a field several documents lack raises `DuplicateKeyError`**, because missing and `null` share one token.
13. **`update()` (legacy) calls `update_many` for a dict update**, not `update_one`.
14. **Errors are not always PyMongo-shaped.** Invalid update documents raise `ValueError`, not `WriteError`.
15. **Cross-type sort ordering approximates but does not match BSON ordering** — notably, booleans sort last.
16. **`server_info()` returns TinyMongo metadata**, not MongoDB version info.
17. **`tinydb<4` is pinned.** TinyMongo will not work with TinyDB 4.x.

---

## Common Patterns

### Swapping between MongoDB and TinyMongo by configuration

```python
import os

if os.environ.get("USE_LOCAL_DB"):
    from tinymongo import MongoClient
    client = MongoClient(tinymongo_folder="./data", backend="sqlite")
else:
    from pymongo import MongoClient
    client = MongoClient(os.environ["MONGO_URI"])

users = client.app.users        # identical from here on, within the supported subset
```

Guard anything outside the subset with `capabilities()`:

```python
if getattr(client, "capabilities", None) and not client.supports("aggregation"):
    totals = compute_totals_in_python(users.find({}))
else:
    totals = list(users.aggregate(PIPELINE))
```

### Isolated per-test database

```python
import pytest
from tinymongo import TinyMongoClient


@pytest.fixture
def db():
    client = TinyMongoClient(backend="memory")   # no files, fully isolated
    try:
        yield client.testdb
    finally:
        client.close()


def test_user_signup(db):
    db.users.insert_one({"email": "ada@example.com"})
    assert db.users.count_documents({}) == 1
```

### Testing existing PyMongo application code untouched

```python
import pymongo
import tinymongo


def test_repository_layer():
    with tinymongo.patch():                 # in-memory by default
        from myapp.repository import UserRepository

        repo = UserRepository(pymongo.MongoClient())
        repo.create("ada@example.com")
        assert repo.find_by_email("ada@example.com") is not None
```

### Paginated listing

```python
def page(collection, filter=None, page=1, per_page=20, sort_field="_id", direction=1):
    skip = (page - 1) * per_page
    cursor = collection.find(filter or {}).sort(sort_field, direction).skip(skip).limit(per_page)
    return {
        "items": cursor.to_list(),
        "total": collection.count_documents(filter or {}),
        "page": page,
    }
```

### Unique constraint with a friendly error

```python
from tinymongo.errors import DuplicateKeyError

users.create_index("email", unique=True, name="email_uq")

def register(email, name):
    try:
        return users.insert_one({"email": email, "name": name}).inserted_id
    except DuplicateKeyError:
        raise ValueError(f"{email} is already registered") from None
```

### Atomic-ish counter increment

```python
def bump_and_read(collection, doc_id, field="views"):
    return collection.find_one_and_update(
        {"_id": doc_id},
        {"$inc": {field: 1}},
        return_document=True,          # ReturnDocument.AFTER
        upsert=True,
    )
```

### Removing a field portably

Because `$unset` is unreliable on the JSON and memory backends, do it by replacement:

```python
def unset_field(collection, doc_id, field):
    doc = collection.find_one({"_id": doc_id})
    if doc is None or field not in doc:
        return False
    doc.pop(field)
    collection.replace_one({"_id": doc_id}, doc)
    return True
```

### Async repository

```python
from tinymongo import AsyncMongoClient


class UserRepo:
    def __init__(self, client: AsyncMongoClient):
        self._users = client.app.users

    async def create(self, email: str, name: str) -> str:
        result = await self._users.insert_one({"email": email, "name": name, "score": 0})
        return result.inserted_id

    async def top(self, limit: int = 10) -> list[dict]:
        return await self._users.find({}).sort("score", -1).limit(limit).to_list(length=None)

    async def bump(self, email: str) -> dict | None:
        return await self._users.find_one_and_update(
            {"email": email}, {"$inc": {"score": 1}}, return_document=True
        )
```

### Migrating a prototype to another backend

```bash
tinymongo migrate ./tinydb ./sqlite-db --to-backend sqlite
```

```python
from tinymongo import TinyMongoClient

client = TinyMongoClient("./sqlite-db", backend="sqlite")
# Recreate indexes: pre-1.2 index metadata was handle-scoped and cannot be migrated.
client.app.users.create_index("email", unique=True)
```
