# PyMongo Comprehensive Reference

> A complete API reference for PyMongo — the official Python driver for MongoDB.
> Covers synchronous and asynchronous APIs, BSON types, GridFS, and configuration.
> Based on PyMongo 4.x source code.

---

## Table of Contents

- [Installation & Quick Start](#installation--quick-start)
- [MongoClient](#mongoclient)
- [AsyncMongoClient](#asyncmongoclient)
- [Database](#database)
- [Collection](#collection)
- [CRUD Operations](#crud-operations)
- [Cursor](#cursor)
- [CommandCursor](#commandcursor)
- [Result Classes](#result-classes)
- [ClientSession & Transactions](#clientsession--transactions)
- [Change Streams](#change-streams)
- [Aggregation Pipelines](#aggregation-pipelines)
- [Index Management](#index-management)
- [Search Indexes (Atlas)](#search-indexes-atlas)
- [Bulk Write Operations](#bulk-write-operations)
- [Read Preferences](#read-preferences)
- [Write Concern](#write-concern)
- [Read Concern](#read-concern)
- [Collation](#collation)
- [Server API](#server-api)
- [BSON Types](#bson-types)
  - [ObjectId](#objectid)
  - [Binary](#binary)
  - [Decimal128](#decimal128)
  - [Int64](#int64)
  - [Timestamp](#timestamp)
  - [Code](#code)
  - [Regex](#regex)
  - [DBRef](#dbref)
  - [SON](#son)
  - [MinKey / MaxKey](#minkey--maxkey)
  - [DatetimeMS](#datetimems)
  - [RawBSONDocument](#rawbsondocument)
- [BSON Encoding & Decoding](#bson-encoding--decoding)
- [JSON Utilities (bson.json_util)](#json-utilities-bsonjson_util)
- [Codec Options & Custom Types](#codec-options--custom-types)
- [GridFS](#gridfs)
- [Client-Side Encryption](#client-side-encryption)
- [Monitoring & Events](#monitoring--events)
- [Errors & Exceptions](#errors--exceptions)
- [Connection Configuration Constants](#connection-configuration-constants)
- [PyMongo Module Exports](#pymongo-module-exports)
- [Migration Notes (PyMongo 3 → 4)](#migration-notes-pymongo-3--4)

---

## Installation & Quick Start

```bash
pip install pymongo
```

```python
from pymongo import MongoClient

# Connect (returns immediately, connects in background)
client = MongoClient("mongodb://localhost:27017")

# Access database and collection
db = client["mydb"]
collection = db["mycollection"]

# Insert
result = collection.insert_one({"name": "Alice", "age": 30})
print(result.inserted_id)

# Query
doc = collection.find_one({"name": "Alice"})
for doc in collection.find({"age": {"$gte": 25}}):
    print(doc)

# Update
collection.update_one({"name": "Alice"}, {"$set": {"age": 31}})

# Delete
collection.delete_one({"name": "Alice"})

# Always close or use context manager
client.close()
```

**Context manager pattern (recommended):**

```python
with MongoClient("mongodb://localhost:27017") as client:
    db = client.mydb
    db.mycollection.insert_one({"key": "value"})
```

**Async quick start:**

```python
import asyncio
from pymongo import AsyncMongoClient

async def main():
    async with AsyncMongoClient("mongodb://localhost:27017") as client:
        db = client.mydb
        await db.mycollection.insert_one({"key": "value"})
        doc = await db.mycollection.find_one({"key": "value"})
        print(doc)

asyncio.run(main())
```

---

## MongoClient

The primary entry point for synchronous MongoDB connections. Thread-safe with built-in connection pooling.

**Source:** `pymongo/synchronous/mongo_client.py`

### Constructor

```python
MongoClient(
    host: Optional[Union[str, Sequence[str]]] = None,  # hostname, IP, URI, or list
    port: Optional[int] = None,                          # default 27017
    document_class: Optional[Type[_DocumentType]] = None, # default dict
    tz_aware: Optional[bool] = None,                     # timezone-aware datetimes
    connect: Optional[bool] = None,                      # connect immediately (default True)
    type_registry: Optional[TypeRegistry] = None,        # custom type encoding/decoding
    **kwargs: Any,                                       # additional connection options
)
```

**Key `**kwargs` options:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `directConnection` | bool | False | Connect directly to a single server |
| `replicaSet` | str | None | Replica set name |
| `maxPoolSize` | int | 100 | Max connections per server |
| `minPoolSize` | int | 0 | Min connections per server |
| `maxIdleTimeMS` | int | None | Max idle time before connection is removed |
| `connectTimeoutMS` | int | 20000 | Connection timeout in ms |
| `socketTimeoutMS` | int | None | Socket operation timeout in ms |
| `serverSelectionTimeoutMS` | int | 30000 | Server selection timeout in ms |
| `heartbeatFrequencyMS` | int | 10000 | Heartbeat interval in ms |
| `appname` | str | None | Application name for server logs |
| `retryWrites` | bool | True | Enable retryable writes |
| `retryReads` | bool | True | Enable retryable reads |
| `w` | int/str | None | Write concern `w` value |
| `wTimeoutMS` | int | None | Write concern timeout |
| `journal` | bool | None | Write concern journaling |
| `readPreference` | str | "primary" | Read preference mode |
| `readPreferenceTags` | list | None | Tag sets for read preference |
| `maxStalenessSeconds` | int | -1 | Max staleness for secondaries |
| `readConcernLevel` | str | None | Read concern level |
| `authSource` | str | None | Authentication database |
| `authMechanism` | str | None | Auth mechanism (SCRAM-SHA-256, etc.) |
| `username` | str | None | Authentication username |
| `password` | str | None | Authentication password |
| `tls` | bool | None | Enable TLS/SSL |
| `tlsCAFile` | str | None | CA certificate file path |
| `tlsCertificateKeyFile` | str | None | Client certificate file |
| `tlsAllowInvalidCertificates` | bool | False | Skip cert validation |
| `tlsAllowInvalidHostnames` | bool | False | Skip hostname verification |
| `compressors` | str | None | Compression algorithms (snappy, zlib, zstd) |
| `timeoutMS` | int | None | Client-side operation timeout (CSOT) |
| `server_api` | ServerApi | None | Stable API version |
| `auto_encryption_opts` | AutoEncryptionOpts | None | Client-side encryption |
| `event_listeners` | list | None | Monitoring event listeners |

### Properties

```python
client.address          # -> Optional[tuple[str, int]]  Current server (host, port)
client.primary          # -> Optional[tuple[str, int]]  Replica set primary
client.secondaries      # -> set[tuple[str, int]]       Replica set secondaries
client.arbiters         # -> set[tuple[str, int]]       Replica set arbiters
client.nodes            # -> FrozenSet[tuple[str, int]] All connected servers
client.is_primary       # -> bool                       Connected to writable server
client.is_mongos        # -> bool                       Connected to mongos
client.options          # -> ClientOptions               All parsed client options
client.topology_description  # -> TopologyDescription    Current topology state
client.codec_options    # -> CodecOptions                BSON codec options
client.read_preference  # -> _ServerMode                 Read preference
client.write_concern    # -> WriteConcern                Write concern
client.read_concern     # -> ReadConcern                 Read concern
```

### Methods

```python
# Database access
client.database_name              # attribute access → Database
client["database-name"]           # subscript access → Database
client.get_database(
    name=None,                    # database name (None = default from URI)
    codec_options=None,
    read_preference=None,
    write_concern=None,
    read_concern=None,
) -> Database

client.get_default_database(
    default=None,                 # fallback name if no DB in URI
    codec_options=None,
    read_preference=None,
    write_concern=None,
    read_concern=None,
) -> Database

# Server operations
client.server_info(session=None) -> dict[str, Any]
client.list_databases(session=None, comment=None, **kwargs) -> CommandCursor
client.list_database_names(session=None, comment=None) -> list[str]
client.drop_database(name_or_database, session=None, comment=None) -> None

# Sessions
client.start_session(
    causal_consistency=None,
    default_transaction_options=None,
    snapshot=False,
) -> ClientSession

# Client-level bulk write (MongoDB 8.0+)
client.bulk_write(
    models: Sequence[_WriteOp],
    session=None,
    ordered=True,
    verbose_results=False,
    bypass_document_validation=None,
    comment=None,
    let=None,
    write_concern=None,
) -> ClientBulkWriteResult

# Change streams (cluster-wide, MongoDB 4.0+)
client.watch(
    pipeline=None,
    full_document=None,
    resume_after=None,
    max_await_time_ms=None,
    batch_size=None,
    collation=None,
    start_at_operation_time=None,
    session=None,
    start_after=None,
    comment=None,
    full_document_before_change=None,
    show_expanded_events=None,
) -> ChangeStream

# Connection lifecycle
client.close() -> None
```

---

## AsyncMongoClient

Async version of MongoClient with identical API. All I/O methods are coroutines.

**Source:** `pymongo/asynchronous/mongo_client.py`

```python
from pymongo import AsyncMongoClient

async with AsyncMongoClient("mongodb://localhost:27017") as client:
    db = client.mydb
    await db.mycollection.insert_one({"key": "value"})
    names = await client.list_database_names()
```

All methods have the same signatures as `MongoClient` but must be awaited. Cursors return `AsyncCursor`, `AsyncCommandCursor`, etc.

---

## Database

Represents a MongoDB database. Obtained from a `MongoClient`.

**Source:** `pymongo/synchronous/database.py`

### Constructor

```python
Database(
    client: MongoClient[_DocumentType],
    name: str,
    codec_options: Optional[CodecOptions] = None,
    read_preference: Optional[_ServerMode] = None,
    write_concern: Optional[WriteConcern] = None,
    read_concern: Optional[ReadConcern] = None,
)
```

### Properties

```python
db.client       # -> MongoClient     The client for this database
db.name         # -> str             Database name
db.codec_options
db.read_preference
db.write_concern
db.read_concern
```

### Methods

```python
# Collection access
db.collection_name           # attribute access → Collection
db["collection-name"]        # subscript access → Collection
db.get_collection(
    name,
    codec_options=None,
    read_preference=None,
    write_concern=None,
    read_concern=None,
) -> Collection

# Clone with different options
db.with_options(
    codec_options=None,
    read_preference=None,
    write_concern=None,
    read_concern=None,
) -> Database

# Collection management
db.create_collection(
    name,
    codec_options=None,
    read_preference=None,
    write_concern=None,
    read_concern=None,
    session=None,
    check_exists=True,
    **kwargs,                 # capped, size, max, validator, etc.
) -> Collection

db.drop_collection(
    name_or_collection,
    session=None,
    comment=None,
    encrypted_fields=None,
) -> dict

db.list_collections(session=None, filter=None, comment=None, **kwargs) -> CommandCursor
db.list_collection_names(session=None, filter=None, comment=None, **kwargs) -> list[str]

# Commands
db.command(
    command,                  # str or dict
    value=1,
    check=True,
    allowable_errors=None,
    read_preference=None,
    codec_options=None,
    session=None,
    comment=None,
    **kwargs,
) -> dict

db.cursor_command(
    command,
    value=1,
    read_preference=None,
    codec_options=None,
    session=None,
    comment=None,
    max_await_time_ms=None,
    **kwargs,
) -> CommandCursor

# Aggregation (database-level)
db.aggregate(pipeline, session=None, **kwargs) -> CommandCursor

# Change streams (database-level)
db.watch(
    pipeline=None,
    full_document=None,
    resume_after=None,
    max_await_time_ms=None,
    batch_size=None,
    collation=None,
    start_at_operation_time=None,
    session=None,
    start_after=None,
    comment=None,
    full_document_before_change=None,
    show_expanded_events=None,
) -> DatabaseChangeStream

# Other
db.validate_collection(
    name_or_collection,
    scandata=False,
    full=False,
    session=None,
    background=False,
    comment=None,
) -> dict

db.dereference(dbref, session=None, comment=None, **kwargs) -> Optional[_DocumentType]
```

---

## Collection

Represents a MongoDB collection. The main interface for CRUD operations.

**Source:** `pymongo/synchronous/collection.py`

### Constructor

```python
Collection(
    database: Database[_DocumentType],
    name: str,
    create: Optional[bool] = False,
    codec_options: Optional[CodecOptions] = None,
    read_preference: Optional[_ServerMode] = None,
    write_concern: Optional[WriteConcern] = None,
    read_concern: Optional[ReadConcern] = None,
    session: Optional[ClientSession] = None,
    **kwargs: Any,
)
```

### Properties

```python
collection.full_name   # -> str          "database.collection"
collection.name        # -> str          Collection name
collection.database    # -> Database     Parent database
collection.codec_options
collection.read_preference
collection.write_concern
collection.read_concern
```

### Methods — Full Signatures

See the [CRUD Operations](#crud-operations) section for detailed usage examples.

```python
# Clone with options
collection.with_options(
    codec_options=None,
    read_preference=None,
    write_concern=None,
    read_concern=None,
) -> Collection

# Get collection options
collection.options(session=None, comment=None) -> MutableMapping

# Rename
collection.rename(new_name, session=None, comment=None, **kwargs) -> MutableMapping

# Drop
collection.drop(session=None, comment=None, encrypted_fields=None) -> None

# Sub-collection access
collection.subcollection    # attribute access
collection["subcollection"] # subscript access
```

---

## CRUD Operations

### Insert

```python
# Insert one document
collection.insert_one(
    document: _DocumentType,
    bypass_document_validation: bool = False,
    session: Optional[ClientSession] = None,
    comment: Optional[Any] = None,
) -> InsertOneResult

# Insert multiple documents
collection.insert_many(
    documents: Iterable[_DocumentType],
    ordered: bool = True,
    bypass_document_validation: bool = False,
    session: Optional[ClientSession] = None,
    comment: Optional[Any] = None,
) -> InsertManyResult
```

### Find / Query

```python
# Find one document
collection.find_one(
    filter: Optional[Mapping[str, Any]] = None,
    *args: Any,
    **kwargs: Any,
) -> Optional[_DocumentType]

# Find multiple documents (returns Cursor)
collection.find(
    filter: Optional[Mapping[str, Any]] = None,
    projection: Optional[Union[Mapping[str, Any], Iterable[str]]] = None,
    session: Optional[ClientSession] = None,
    skip: int = 0,
    limit: int = 0,
    no_cursor_timeout: bool = False,
    cursor_type: int = CursorType.NON_TAILABLE,
    sort: Optional[_Sort] = None,
    allow_partial_results: bool = False,
    batch_size: int = 0,
    collation: Optional[_CollationIn] = None,
    return_key: Optional[bool] = None,
    show_record_id: Optional[bool] = None,
    hint: Optional[_Hint] = None,
    max_time_ms: Optional[int] = None,
    min: Optional[_Sort] = None,
    max: Optional[_Sort] = None,
    comment: Optional[Any] = None,
    allow_disk_use: Optional[bool] = None,
    **kwargs: Any,
) -> Cursor[_DocumentType]
```

**Common query patterns:**

```python
# Equality
collection.find({"status": "A"})

# Comparison operators
collection.find({"age": {"$gt": 25}})
collection.find({"age": {"$gte": 20, "$lte": 40}})

# Logical operators
collection.find({"$or": [{"status": "A"}, {"qty": {"$lt": 30}}]})
collection.find({"$and": [{"price": {"$ne": 1.99}}, {"price": {"$exists": True}}]})

# Array queries
collection.find({"tags": "red"})                          # array contains "red"
collection.find({"tags": {"$all": ["red", "blank"]}})     # contains all
collection.find({"tags": {"$size": 3}})                   # array has 3 elements
collection.find({"results": {"$elemMatch": {"$gte": 80}}})

# Nested documents
collection.find({"address.city": "New York"})

# Regex
collection.find({"name": {"$regex": "^Al"}})

# Null / exists
collection.find({"email": None})
collection.find({"email": {"$exists": True}})

# Projections (include/exclude fields)
collection.find({"status": "A"}, {"item": 1, "status": 1, "_id": 0})

# Sort, skip, limit
collection.find().sort("age", pymongo.ASCENDING).skip(10).limit(5)
collection.find().sort([("age", pymongo.DESCENDING), ("name", pymongo.ASCENDING)])

# Count
collection.count_documents({"status": "A"})
collection.estimated_document_count()

# Distinct values
collection.distinct("status")
collection.distinct("status", {"age": {"$gt": 25}})
```

### Update

```python
# Update one document
collection.update_one(
    filter: Mapping[str, Any],
    update: Union[Mapping[str, Any], _Pipeline],
    upsert: bool = False,
    bypass_document_validation: bool = False,
    collation: Optional[_CollationIn] = None,
    array_filters: Optional[list[Mapping[str, Any]]] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    sort: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
) -> UpdateResult

# Update many documents
collection.update_many(
    filter: Mapping[str, Any],
    update: Union[Mapping[str, Any], _Pipeline],
    upsert: bool = False,
    array_filters: Optional[list[Mapping[str, Any]]] = None,
    bypass_document_validation: bool = False,
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
) -> UpdateResult

# Replace one document entirely
collection.replace_one(
    filter: Mapping[str, Any],
    replacement: _DocumentType,
    upsert: bool = False,
    bypass_document_validation: bool = False,
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    sort: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
) -> UpdateResult
```

**Common update operators:**

```python
# Set fields
collection.update_one({"_id": id}, {"$set": {"status": "D"}})

# Increment
collection.update_one({"_id": id}, {"$inc": {"quantity": 5}})

# Unset (remove field)
collection.update_one({"_id": id}, {"$unset": {"temp_field": ""}})

# Push to array
collection.update_one({"_id": id}, {"$push": {"tags": "new_tag"}})

# Pull from array
collection.update_one({"_id": id}, {"$pull": {"tags": "old_tag"}})

# Upsert (insert if not found)
collection.update_one({"name": "Bob"}, {"$set": {"age": 25}}, upsert=True)

# Array filters
collection.update_one(
    {"_id": id},
    {"$set": {"grades.$[elem].mean": 100}},
    array_filters=[{"elem.grade": {"$gte": 85}}]
)

# Update with aggregation pipeline
collection.update_one({"_id": id}, [{"$set": {"total": {"$sum": "$items.price"}}}])
```

### Delete

```python
# Delete one document
collection.delete_one(
    filter: Mapping[str, Any],
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
) -> DeleteResult

# Delete many documents
collection.delete_many(
    filter: Mapping[str, Any],
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
) -> DeleteResult
```

### Find and Modify (Atomic)

```python
# Find one and delete (returns the deleted document)
collection.find_one_and_delete(
    filter: Mapping[str, Any],
    projection: Optional[Mapping[str, Any]] = None,
    sort: Optional[_Sort] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
    **kwargs: Any,
) -> Optional[_DocumentType]

# Find one and replace (returns document before or after replacement)
collection.find_one_and_replace(
    filter: Mapping[str, Any],
    replacement: Mapping[str, Any],
    projection: Optional[Mapping[str, Any]] = None,
    sort: Optional[_Sort] = None,
    upsert: bool = False,
    return_document: ReturnDocument = ReturnDocument.BEFORE,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
    **kwargs: Any,
) -> Optional[_DocumentType]

# Find one and update (returns document before or after update)
collection.find_one_and_update(
    filter: Mapping[str, Any],
    update: Union[Mapping[str, Any], _Pipeline],
    projection: Optional[Mapping[str, Any]] = None,
    sort: Optional[_Sort] = None,
    upsert: bool = False,
    return_document: ReturnDocument = ReturnDocument.BEFORE,
    array_filters: Optional[list[Mapping[str, Any]]] = None,
    hint: Optional[_IndexKeyHint] = None,
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
    **kwargs: Any,
) -> Optional[_DocumentType]
```

**ReturnDocument constants:**

```python
from pymongo import ReturnDocument

ReturnDocument.BEFORE  # Return document before modification (default)
ReturnDocument.AFTER   # Return document after modification
```

### Count & Distinct

```python
# Count documents matching a filter (accurate, uses aggregation)
collection.count_documents(
    filter: Mapping[str, Any],
    session: Optional[ClientSession] = None,
    comment: Optional[Any] = None,
    **kwargs: Any,   # hint, limit, skip, maxTimeMS, collation
) -> int

# Estimate total document count from collection metadata (fast)
collection.estimated_document_count(
    comment: Optional[Any] = None,
    **kwargs: Any,   # maxTimeMS
) -> int

# Get distinct values for a field
collection.distinct(
    key: str,
    filter: Optional[Mapping[str, Any]] = None,
    session: Optional[ClientSession] = None,
    comment: Optional[Any] = None,
    hint: Optional[_IndexKeyHint] = None,
    **kwargs: Any,
) -> list[Any]
```

---

## Cursor

Iterates over query results. Returned by `collection.find()`.

**Source:** `pymongo/synchronous/cursor.py`

### Properties

```python
cursor.collection   # -> Collection    The collection being queried
cursor.retrieved    # -> int           Number of documents retrieved so far
cursor.session      # -> Optional[ClientSession]
cursor.alive        # -> bool          Whether cursor can return more data
cursor.cursor_id    # -> int           Server-side cursor ID
```

### Chaining Methods (return self)

All these methods return the cursor itself, enabling method chaining. They must be called before iterating.

```python
cursor.sort(key_or_list, direction=None)   # Sort results
cursor.skip(skip: int)                     # Skip first N results
cursor.limit(limit: int)                   # Limit to N results
cursor.batch_size(batch_size: int)         # Set batch size
cursor.hint(index)                         # Force specific index
cursor.comment(comment: Any)               # Attach a comment
cursor.where(code: str)                    # Add $where JS filter
cursor.collation(collation)                # Set collation
cursor.max_time_ms(max_time_ms: int)       # Max execution time
cursor.max_await_time_ms(ms: int)          # Max getMore wait (tailable)
cursor.min(spec: _Sort)                    # Lower index bound
cursor.max(spec: _Sort)                    # Upper index bound
cursor.allow_disk_use(allow: bool)         # Allow disk for sort
cursor.add_option(mask: int)               # Set query flags
cursor.remove_option(mask: int)            # Unset query flags
```

### Iteration Methods

```python
cursor.next()                  # Get next document
cursor.__next__()              # Same as next() (for-loop support)
cursor.__getitem__(index)      # Get by index or slice
cursor.explain()               # Get query execution plan
cursor.distinct(key: str)      # Distinct values from cursor results
cursor.rewind()                # Reset cursor to beginning
cursor.close()                 # Close cursor
cursor.clone()                 # Clone the cursor (resets iteration)
cursor.to_list(length=None)    # Get all results as a list
cursor.try_next()              # Get next without blocking (tailable)
```

### CursorType Constants

```python
from pymongo import CursorType

CursorType.NON_TAILABLE      # Standard cursor (default)
CursorType.TAILABLE           # Tailable cursor for capped collections
CursorType.TAILABLE_AWAIT     # Tailable cursor that blocks for new data
CursorType.EXHAUST            # Exhaust cursor (deprecated)
```

### Usage Patterns

```python
# Basic iteration
for doc in collection.find({"status": "A"}):
    print(doc)

# With chaining
cursor = collection.find({"status": "A"}).sort("name").limit(10).skip(5)
for doc in cursor:
    print(doc)

# As a list
docs = list(collection.find({"status": "A"}))
docs = collection.find({"status": "A"}).to_list()

# Context manager
with collection.find({"status": "A"}) as cursor:
    for doc in cursor:
        print(doc)

# Tailable cursor (capped collection)
cursor = collection.find(cursor_type=CursorType.TAILABLE_AWAIT)
while cursor.alive:
    doc = cursor.try_next()
    if doc is not None:
        print(doc)
```

---

## CommandCursor

Iterates over results from database commands (e.g., `aggregate`, `list_collections`).

**Source:** `pymongo/synchronous/command_cursor.py`

### Methods

```python
cursor.batch_size(batch_size: int)   # Set batch size
cursor.close()                       # Close cursor
cursor.next()                        # Get next document
cursor.try_next()                    # Get next without blocking
cursor.to_list(length=None)          # Get all results as list
cursor.alive                         # Whether cursor can return more
```

---

## Result Classes

**Source:** `pymongo/results.py`

### InsertOneResult

```python
result.acknowledged   # -> bool   Whether write was acknowledged
result.inserted_id    # -> Any    The _id of the inserted document
```

### InsertManyResult

```python
result.acknowledged   # -> bool       Whether write was acknowledged
result.inserted_ids   # -> list[Any]  List of _ids of inserted documents
```

### UpdateResult

```python
result.acknowledged   # -> bool          Whether write was acknowledged
result.matched_count  # -> int           Documents matched by filter
result.modified_count # -> int           Documents actually modified
result.upserted_id    # -> Optional[Any] The _id if an upsert occurred
result.did_upsert     # -> bool          Whether an upsert occurred
result.raw_result     # -> dict          Raw server response
```

### DeleteResult

```python
result.acknowledged   # -> bool   Whether write was acknowledged
result.deleted_count  # -> int    Number of documents deleted
result.raw_result     # -> dict   Raw server response
```

### BulkWriteResult

```python
result.acknowledged    # -> bool            Whether write was acknowledged
result.inserted_count  # -> int             Documents inserted
result.matched_count   # -> int             Documents matched
result.modified_count  # -> int             Documents modified
result.deleted_count   # -> int             Documents deleted
result.upserted_count  # -> int             Documents upserted
result.upserted_ids    # -> dict[int, Any]  Map of index → upserted _id
result.bulk_api_result # -> dict            Raw bulk result
```

### ClientBulkWriteResult

```python
result.acknowledged      # -> bool   Whether write was acknowledged
result.has_verbose_results  # -> bool
result.inserted_count    # -> int
result.matched_count     # -> int
result.modified_count    # -> int
result.deleted_count     # -> int
result.upserted_count    # -> int
result.insert_results    # -> dict[int, InsertOneResult]
result.update_results    # -> dict[int, UpdateResult]
result.delete_results    # -> dict[int, DeleteResult]
```

---

## ClientSession & Transactions

**Source:** `pymongo/synchronous/client_session.py`

### Creating a Session

```python
with client.start_session(
    causal_consistency=None,          # Enable causal consistency
    default_transaction_options=None, # TransactionOptions
    snapshot=False,                   # Enable snapshot reads
) as session:
    # Use session with operations
    collection.insert_one({"x": 1}, session=session)
```

### Session Properties

```python
session.client          # -> MongoClient
session.options         # -> SessionOptions
session.session_id      # -> dict            Server session ID
session.cluster_time    # -> Optional[dict]
session.operation_time  # -> Optional[Timestamp]
session.has_ended       # -> bool
session.in_transaction  # -> bool
```

### Transactions

```python
# Method 1: with_transaction (recommended — handles retries automatically)
def callback(session):
    collection.insert_one({"x": 1}, session=session)
    collection.update_one({"x": 1}, {"$set": {"y": 2}}, session=session)

with client.start_session() as session:
    session.with_transaction(
        callback,
        read_concern=None,
        write_concern=None,
        read_preference=None,
        max_commit_time_ms=None,
    )

# Method 2: Manual transaction control
with client.start_session() as session:
    session.start_transaction(
        read_concern=None,
        write_concern=None,
        read_preference=None,
        max_commit_time_ms=None,
    )
    try:
        collection.insert_one({"x": 1}, session=session)
        collection.update_one({"x": 1}, {"$set": {"y": 2}}, session=session)
        session.commit_transaction()
    except Exception:
        session.abort_transaction()
        raise
```

### TransactionOptions

```python
from pymongo.client_session import TransactionOptions

opts = TransactionOptions(
    read_concern=ReadConcern("snapshot"),
    write_concern=WriteConcern(w="majority"),
    read_preference=ReadPreference.PRIMARY,
    max_commit_time_ms=5000,
)
```

### Session Methods

```python
session.start_transaction(read_concern=None, write_concern=None,
                          read_preference=None, max_commit_time_ms=None)
session.commit_transaction()
session.abort_transaction()
session.with_transaction(callback, read_concern=None, write_concern=None,
                         read_preference=None, max_commit_time_ms=None)
session.end_session()
session.advance_cluster_time(cluster_time)
session.advance_operation_time(operation_time)
```

---

## Change Streams

Watch real-time changes on collections, databases, or the entire cluster.

**Source:** `pymongo/synchronous/change_stream.py`

### Opening Change Streams

```python
# Collection-level
with collection.watch(
    pipeline=None,                        # Additional aggregation stages
    full_document=None,                   # "updateLookup", "whenAvailable", "required"
    full_document_before_change=None,     # "whenAvailable", "required"
    resume_after=None,                    # Resume token
    start_after=None,                     # Start after invalidate
    max_await_time_ms=None,               # Max wait for getMore
    batch_size=None,
    collation=None,
    start_at_operation_time=None,         # Timestamp
    session=None,
    comment=None,
    show_expanded_events=None,            # Include DDL events
) as stream:
    for change in stream:
        print(change)

# Database-level
with db.watch() as stream:
    for change in stream:
        print(change)

# Cluster-level (MongoDB 4.0+)
with client.watch() as stream:
    for change in stream:
        print(change)
```

### ChangeStream Properties & Methods

```python
stream.resume_token  # -> dict   Cached resume token for resuming
stream.alive         # -> bool   Whether stream can return more data
stream.next()        # -> dict   Block until next change document
stream.try_next()    # -> Optional[dict]   Non-blocking next
stream.close()       # Close the stream
```

### Change Document Format

```python
{
    "_id": {"_data": "..."},          # Resume token
    "operationType": "insert",         # insert, update, replace, delete, drop, rename, ...
    "fullDocument": {...},             # The document (for insert/replace/update with lookup)
    "fullDocumentBeforeChange": {...}, # Pre-image (if configured)
    "ns": {"db": "mydb", "coll": "mycoll"},
    "documentKey": {"_id": ObjectId("...")},
    "updateDescription": {             # For update operations
        "updatedFields": {"field": "new_value"},
        "removedFields": ["old_field"],
        "truncatedArrays": [],
    },
    "clusterTime": Timestamp(1234, 1),
}
```

---

## Aggregation Pipelines

```python
# Collection-level aggregation
cursor = collection.aggregate(
    pipeline: list[dict],
    session: Optional[ClientSession] = None,
    let: Optional[Mapping[str, Any]] = None,
    comment: Optional[Any] = None,
    **kwargs: Any,
) -> CommandCursor

# Database-level aggregation
cursor = db.aggregate(pipeline, session=None, **kwargs)
```

### Common Pipeline Stages

```python
pipeline = [
    {"$match": {"status": "A"}},
    {"$group": {"_id": "$cust_id", "total": {"$sum": "$amount"}}},
    {"$sort": {"total": -1}},
    {"$limit": 10},
    {"$skip": 5},
    {"$project": {"_id": 0, "customer": "$_id", "total": 1}},
    {"$unwind": "$items"},
    {"$lookup": {
        "from": "orders",
        "localField": "order_id",
        "foreignField": "_id",
        "as": "order_details"
    }},
    {"$addFields": {"total_with_tax": {"$multiply": ["$total", 1.1]}}},
    {"$out": "output_collection"},   # Write to collection
    {"$merge": {"into": "target"}},  # Merge into collection
]
result = collection.aggregate(pipeline)
```

### Raw Batch Aggregation

```python
# Returns raw BSON batches (for high-performance processing)
cursor = collection.aggregate_raw_batches(pipeline, session=None, comment=None, **kwargs)
cursor = collection.find_raw_batches(*args, **kwargs)
```

---

## Index Management

```python
# Create a single index
collection.create_index(
    keys,            # Field name or list of (field, direction) pairs
    session=None,
    comment=None,
    **kwargs,        # name, unique, background, sparse, expireAfterSeconds, etc.
) -> str             # Returns index name

# Create multiple indexes
collection.create_indexes(
    indexes: list[IndexModel],
    session=None,
    comment=None,
    **kwargs,
) -> list[str]

# Drop an index
collection.drop_index(index_or_name, session=None, comment=None, **kwargs) -> None

# Drop all indexes (except _id)
collection.drop_indexes(session=None, comment=None, **kwargs) -> None

# List indexes
collection.list_indexes(session=None, comment=None) -> CommandCursor

# Get index information as dict
collection.index_information(session=None, comment=None) -> MutableMapping
```

### IndexModel

```python
from pymongo import IndexModel, ASCENDING, DESCENDING, TEXT, HASHED, GEO2D, GEOSPHERE

# Simple index
IndexModel([("field", ASCENDING)])

# Compound index
IndexModel([("field1", ASCENDING), ("field2", DESCENDING)])

# With options
IndexModel(
    [("field", ASCENDING)],
    name="my_index",
    unique=True,
    sparse=True,
    expireAfterSeconds=3600,
    background=True,
    collation=Collation(locale="en", strength=2),
)

# Text index
IndexModel([("content", TEXT)])

# Geospatial
IndexModel([("location", GEO2D)])
IndexModel([("location", GEOSPHERE)])

# Hashed
IndexModel([("field", HASHED)])
```

### Direction Constants

```python
import pymongo

pymongo.ASCENDING   # 1
pymongo.DESCENDING  # -1
pymongo.GEO2D       # "2d"
pymongo.GEOSPHERE   # "2dsphere"
pymongo.HASHED      # "hashed"
pymongo.TEXT         # "text"
```

---

## Search Indexes (Atlas)

For MongoDB Atlas Search and Vector Search indexes.

```python
# Create a search index
collection.create_search_index(
    model: SearchIndexModel,
    session=None,
    comment=None,
    **kwargs,
) -> str

# Create multiple search indexes
collection.create_search_indexes(
    models: list[SearchIndexModel],
    session=None,
    comment=None,
    **kwargs,
) -> list[str]

# List search indexes
collection.list_search_indexes(
    name=None,
    session=None,
    comment=None,
    **kwargs,
) -> CommandCursor

# Update a search index
collection.update_search_index(
    name: str,
    definition: Mapping[str, Any],
    session=None,
    comment=None,
    **kwargs,
) -> None

# Drop a search index
collection.drop_search_index(name: str, session=None, comment=None, **kwargs) -> None
```

---

## Bulk Write Operations

### Collection-Level Bulk Write

```python
from pymongo import InsertOne, UpdateOne, UpdateMany, DeleteOne, DeleteMany, ReplaceOne

requests = [
    InsertOne({"x": 1}),
    UpdateOne({"x": 1}, {"$set": {"y": 2}}),
    UpdateMany({"x": 1}, {"$inc": {"y": 1}}),
    ReplaceOne({"x": 1}, {"x": 2, "y": 3}),
    DeleteOne({"x": 2}),
    DeleteMany({"y": {"$gt": 5}}),
]

result = collection.bulk_write(
    requests: Sequence[_WriteOp],
    ordered: bool = True,         # Stop on first error if True
    bypass_document_validation: bool = False,
    session: Optional[ClientSession] = None,
    comment: Optional[Any] = None,
    let: Optional[Mapping[str, Any]] = None,
) -> BulkWriteResult
```

### Client-Level Bulk Write (MongoDB 8.0+)

```python
# Operations across multiple namespaces
from pymongo import InsertOne, UpdateOne, DeleteOne

models = [
    InsertOne(namespace="db.coll1", document={"x": 1}),
    UpdateOne(namespace="db.coll2", filter={"x": 1}, update={"$set": {"y": 2}}),
    DeleteOne(namespace="db.coll1", filter={"x": 1}),
]

result = client.bulk_write(
    models,
    ordered=True,
    verbose_results=False,
    bypass_document_validation=None,
    comment=None,
    let=None,
    write_concern=None,
) -> ClientBulkWriteResult
```

### Operation Classes

```python
InsertOne(
    document: _DocumentType,
    namespace: Optional[str] = None,  # Only for client.bulk_write
)

UpdateOne(
    filter: Mapping[str, Any],
    update: Union[Mapping[str, Any], _Pipeline],
    upsert: Optional[bool] = None,
    collation: Optional[_CollationIn] = None,
    array_filters: Optional[list] = None,
    hint: Optional[_IndexKeyHint] = None,
    namespace: Optional[str] = None,
    sort: Optional[Mapping[str, Any]] = None,
)

UpdateMany(
    filter: Mapping[str, Any],
    update: Union[Mapping[str, Any], _Pipeline],
    upsert: Optional[bool] = None,
    collation: Optional[_CollationIn] = None,
    array_filters: Optional[list] = None,
    hint: Optional[_IndexKeyHint] = None,
    namespace: Optional[str] = None,
)

ReplaceOne(
    filter: Mapping[str, Any],
    replacement: _DocumentType,
    upsert: Optional[bool] = None,
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    namespace: Optional[str] = None,
    sort: Optional[Mapping[str, Any]] = None,
)

DeleteOne(
    filter: Mapping[str, Any],
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    namespace: Optional[str] = None,
)

DeleteMany(
    filter: Mapping[str, Any],
    collation: Optional[_CollationIn] = None,
    hint: Optional[_IndexKeyHint] = None,
    namespace: Optional[str] = None,
)
```

---

## Read Preferences

Control which replica set members receive read operations.

**Source:** `pymongo/read_preferences.py`

```python
from pymongo import ReadPreference

ReadPreference.PRIMARY              # Only primary (default)
ReadPreference.PRIMARY_PREFERRED    # Primary preferred, fall back to secondary
ReadPreference.SECONDARY            # Only secondaries
ReadPreference.SECONDARY_PREFERRED  # Secondary preferred, fall back to primary
ReadPreference.NEAREST              # Lowest latency member

# With tag sets and max staleness
from pymongo.read_preferences import Secondary, Nearest

pref = Secondary(
    tag_sets=[{"region": "us-east"}],
    max_staleness=120,  # seconds
)

pref = Nearest(
    tag_sets=[{"dc": "east"}, {}],  # Fallback to any if no match
    max_staleness=300,
)

# Apply to client, database, or collection
client = MongoClient(readPreference="secondaryPreferred")
db = client.get_database("mydb", read_preference=ReadPreference.SECONDARY)
coll = db.get_collection("mycoll", read_preference=ReadPreference.NEAREST)
```

### Read Preference Properties

```python
pref.name            # -> str   "Primary", "Secondary", etc.
pref.mode            # -> int   Numeric mode value
pref.tag_sets        # -> list  Tag set list
pref.max_staleness   # -> int   Max staleness in seconds (-1 = no max)
pref.document        # -> dict  Wire protocol representation
```

---

## Write Concern

Controls acknowledgment of write operations.

**Source:** `pymongo/write_concern.py`

```python
from pymongo import WriteConcern

# Default (acknowledged, no journaling requirement)
wc = WriteConcern()

# Majority acknowledgment
wc = WriteConcern(w="majority")

# Specific number of nodes
wc = WriteConcern(w=2)

# With journaling
wc = WriteConcern(w="majority", j=True)

# With timeout
wc = WriteConcern(w="majority", wtimeout=5000)  # 5 seconds

# Unacknowledged writes (fire-and-forget)
wc = WriteConcern(w=0)
```

### WriteConcern Properties

```python
wc.acknowledged      # -> bool   Whether writes are acknowledged
wc.is_server_default # -> bool   Whether this is the default
wc.document          # -> dict   Wire protocol representation
```

### Apply to client, database, or collection

```python
client = MongoClient(w="majority", journal=True)
db = client.get_database("mydb", write_concern=WriteConcern(w=2))
coll = db.get_collection("mycoll", write_concern=WriteConcern(w="majority", j=True))
```

---

## Read Concern

Controls consistency and isolation for read operations.

**Source:** `pymongo/read_concern.py`

```python
from pymongo import ReadConcern

ReadConcern()             # Default (no read concern specified)
ReadConcern("local")      # Return the most recent data (default behavior)
ReadConcern("available")  # Like local, but for sharded clusters
ReadConcern("majority")   # Data acknowledged by majority
ReadConcern("linearizable") # Linearizable reads (single-document)
ReadConcern("snapshot")   # Snapshot reads (for transactions)
```

### ReadConcern Properties

```python
rc.level     # -> Optional[str]  The level string
rc.document  # -> dict           Wire protocol representation
```

---

## Collation

Configure string comparison rules for operations.

**Source:** `pymongo/collation.py`

```python
from pymongo.collation import Collation, CollationStrength

collation = Collation(
    locale="en",                                    # Required: ICU locale
    caseLevel=None,                                 # bool
    caseFirst=None,                                 # "upper", "lower", "off"
    strength=CollationStrength.SECONDARY,            # 1-5
    numericOrdering=None,                           # bool (sort "10" after "2")
    alternate=None,                                 # "non-ignorable", "shifted"
    maxVariable=None,                               # "punct", "space"
    normalization=None,                             # bool
    backwards=None,                                 # bool
)

# Use with operations
collection.find({"name": "cafe"}).collation(Collation(locale="fr"))
collection.create_index("name", collation=Collation(locale="en", strength=2))
collection.update_one({"x": 1}, {"$set": {"y": 2}}, collation=collation)
```

### CollationStrength Constants

```python
CollationStrength.PRIMARY     # 1 - Base characters only
CollationStrength.SECONDARY   # 2 - + Accents
CollationStrength.TERTIARY    # 3 - + Case (default)
CollationStrength.QUATERNARY  # 4 - + Punctuation
CollationStrength.IDENTICAL   # 5 - All differences significant
```

---

## Server API

Pin the driver to a specific MongoDB Stable API version.

**Source:** `pymongo/server_api.py`

```python
from pymongo.server_api import ServerApi, ServerApiVersion

api = ServerApi(
    version=ServerApiVersion.V1,   # Currently only "1"
    strict=True,                   # Error on non-stable commands
    deprecation_errors=True,       # Error on deprecated commands
)

client = MongoClient("mongodb://localhost", server_api=api)
```

---

## BSON Types

### ObjectId

12-byte unique identifier for MongoDB documents.

**Source:** `bson/objectid.py`

```python
from bson import ObjectId

oid = ObjectId()                          # Generate new ObjectId
oid = ObjectId("507f1f77bcf86cd799439011") # From hex string
oid = ObjectId(b'\x50\x7f...')            # From bytes

# Properties
oid.binary           # -> bytes              Raw 12-byte value
oid.generation_time  # -> datetime.datetime  When this ObjectId was generated

# Class methods
ObjectId.from_datetime(dt)    # Create ObjectId from datetime
ObjectId.is_valid("507f...")  # Check if string is valid ObjectId hex

# Comparison and hashing
oid1 == oid2
oid1 < oid2   # ObjectIds are time-ordered
str(oid)       # -> "507f1f77bcf86cd799439011"
```

### Binary

BSON binary data with subtype.

**Source:** `bson/binary.py`

```python
from bson.binary import Binary, UuidRepresentation, BinaryVector, BinaryVectorDtype
from bson import BINARY_SUBTYPE, UUID_SUBTYPE

# Create from bytes
b = Binary(b"\x00\x01\x02", subtype=0)

# Properties
b.subtype  # -> int  Binary subtype

# UUID support
import uuid
b = Binary.from_uuid(uuid.uuid4(), UuidRepresentation.STANDARD)
u = b.as_uuid(UuidRepresentation.STANDARD)

# Vector support (MongoDB Atlas Vector Search)
v = Binary.from_vector([1.0, 2.0, 3.0], BinaryVectorDtype.FLOAT32)
vec = v.as_vector()  # -> BinaryVector
```

**Binary Subtypes:**

```python
BINARY_SUBTYPE       = 0   # Generic binary (default)
FUNCTION_SUBTYPE     = 1   # Function
OLD_BINARY_SUBTYPE   = 2   # Old binary (deprecated)
OLD_UUID_SUBTYPE     = 3   # Old UUID
UUID_SUBTYPE         = 4   # UUID (RFC 4122)
MD5_SUBTYPE          = 5   # MD5
COLUMN_SUBTYPE       = 7   # Column
SENSITIVE_SUBTYPE    = 8   # Sensitive
VECTOR_SUBTYPE       = 9   # Vector
USER_DEFINED_SUBTYPE = 128 # User-defined
```

**UuidRepresentation:**

```python
UuidRepresentation.UNSPECIFIED     # 0 - Raises error (default)
UuidRepresentation.STANDARD       # 4 - RFC 4122 / subtype 4
UuidRepresentation.PYTHON_LEGACY  # 3 - Python legacy subtype 3
UuidRepresentation.JAVA_LEGACY    # 5 - Java driver legacy
UuidRepresentation.CSHARP_LEGACY  # 6 - C# driver legacy
```

**BinaryVectorDtype:**

```python
BinaryVectorDtype.INT8        # 8-bit integer
BinaryVectorDtype.FLOAT32     # 32-bit float
BinaryVectorDtype.PACKED_BIT  # Packed bit
```

### Decimal128

128-bit decimal floating point (IEEE 754-2008).

**Source:** `bson/decimal128.py`

```python
from bson.decimal128 import Decimal128
from decimal import Decimal

d = Decimal128("3.14159")
d = Decimal128(Decimal("3.14159"))
d = Decimal128(3.14)

d.to_decimal()  # -> decimal.Decimal
```

### Int64

Explicitly store a value as BSON int64 (64-bit integer).

**Source:** `bson/int64.py`

```python
from bson.int64 import Int64

val = Int64(9999999999999)
# Behaves like int, but encoded as BSON int64
```

### Timestamp

BSON internal timestamp type (used for replication oplog).

**Source:** `bson/timestamp.py`

```python
from bson.timestamp import Timestamp
import datetime

ts = Timestamp(time=1234567890, inc=1)    # From unix timestamp + increment
ts = Timestamp(time=datetime.datetime.now(), inc=0)

ts.time           # -> int               Unix timestamp
ts.inc            # -> int               Increment
ts.as_datetime()  # -> datetime.datetime
```

### Code

BSON JavaScript code type.

**Source:** `bson/code.py`

```python
from bson.code import Code

code = Code("function() { return 1; }")
code = Code("function(x) { return x; }", scope={"x": 42})

code.scope  # -> Optional[dict]  Variable scope
```

### Regex

BSON regular expression.

**Source:** `bson/regex.py`

```python
from bson.regex import Regex
import re

r = Regex("^test", "i")           # Pattern + flags as string
r = Regex.from_native(re.compile("^test", re.IGNORECASE))

r.pattern       # -> str
r.flags         # -> int
r.try_compile() # -> re.Pattern
```

### DBRef

Database reference.

**Source:** `bson/dbref.py`

```python
from bson.dbref import DBRef

ref = DBRef("collection_name", ObjectId("..."))
ref = DBRef("collection_name", ObjectId("..."), database="mydb")

ref.collection  # -> str
ref.id          # -> Any (usually ObjectId)
ref.database    # -> Optional[str]
ref.as_doc()    # -> SON

# Dereference
doc = db.dereference(ref)
```

### SON

Ordered dict that preserves insertion order (Serialized Ocument Notation). Used internally for BSON document ordering.

**Source:** `bson/son.py`

```python
from bson.son import SON

doc = SON([("name", "Alice"), ("age", 30)])
doc = SON({"name": "Alice", "age": 30})

doc.to_dict()  # -> dict (standard Python dict)

# Behaves like OrderedDict
doc["name"]     # "Alice"
doc.keys()
doc.values()
doc.items()
```

### MinKey / MaxKey

Boundary values that compare lower/higher than all other BSON values.

```python
from bson.min_key import MinKey
from bson.max_key import MaxKey

MinKey()  # Compares less than all other values
MaxKey()  # Compares greater than all other values

# Useful for range queries on mixed-type fields
```

### DatetimeMS

Extended range datetime stored as int64 milliseconds since epoch.

**Source:** `bson/datetime_ms.py`

```python
from bson.datetime_ms import DatetimeMS
import datetime

d = DatetimeMS(1234567890000)                    # From milliseconds
d = DatetimeMS(datetime.datetime(2024, 1, 1))    # From datetime

d.as_datetime()  # -> datetime.datetime
int(d)           # -> int (milliseconds)
```

### RawBSONDocument

Lazily decoded BSON document. Useful for high-performance scenarios.

**Source:** `bson/raw_bson.py`

```python
from bson.raw_bson import RawBSONDocument

# Use as document_class for lazy decoding
coll = db.get_collection("mycoll", codec_options=CodecOptions(document_class=RawBSONDocument))
doc = coll.find_one()       # Returns RawBSONDocument
doc.raw                     # -> bytes (raw BSON)
doc["field"]                # Decoded on access
```

---

## BSON Encoding & Decoding

**Source:** `bson/__init__.py`

```python
import bson

# Encode a document to BSON bytes
data = bson.encode({"name": "Alice", "age": 30})   # -> bytes
data = bson.encode(document, codec_options=None, check_keys=True)

# Decode BSON bytes to a document
doc = bson.decode(data)                             # -> dict
doc = bson.decode(data, codec_options=None)

# Decode multiple documents concatenated in one bytes object
docs = bson.decode_all(data)                        # -> list[dict]

# Decode as iterator
for doc in bson.decode_iter(data):
    print(doc)

# Decode from file
with open("data.bson", "rb") as f:
    for doc in bson.decode_file_iter(f):
        print(doc)

# Validate BSON
bson.is_valid(data)  # -> bool
```

---

## JSON Utilities (bson.json_util)

Serialize/deserialize BSON-extended documents as JSON.

**Source:** `bson/json_util.py`

```python
from bson.json_util import dumps, loads, JSONOptions, JSONMode, DatetimeRepresentation

# Serialize (handles ObjectId, datetime, Binary, etc.)
json_str = dumps({"_id": ObjectId(), "date": datetime.datetime.now()})

# Deserialize (reconstructs BSON types)
doc = loads(json_str)

# Custom JSON options
opts = JSONOptions(
    json_mode=JSONMode.RELAXED,          # LEGACY, RELAXED, or CANONICAL
    strict_number_long=False,
    datetime_representation=DatetimeRepresentation.ISO8601,
    strict_uuid=False,
)
json_str = dumps(doc, json_options=opts)
```

### JSONMode Constants

```python
JSONMode.LEGACY     # 0 - Legacy PyMongo format
JSONMode.RELAXED    # 1 - Relaxed Extended JSON (default, human-readable)
JSONMode.CANONICAL  # 2 - Canonical Extended JSON (type-preserving)
```

### Pre-built JSONOptions

```python
from bson.json_util import (
    DEFAULT_JSON_OPTIONS,       # Relaxed mode (default)
    RELAXED_JSON_OPTIONS,       # Relaxed Extended JSON
    CANONICAL_JSON_OPTIONS,     # Canonical Extended JSON
    LEGACY_JSON_OPTIONS,        # Legacy format
)
```

---

## Codec Options & Custom Types

Configure how BSON types map to Python types.

**Source:** `bson/codec_options.py`

```python
from bson.codec_options import CodecOptions, TypeRegistry, TypeEncoder, TypeDecoder, TypeCodec
from bson.codec_options import DatetimeConversion
from bson.binary import UuidRepresentation

options = CodecOptions(
    document_class=dict,                              # dict, SON, RawBSONDocument, etc.
    tz_aware=False,                                   # Timezone-aware datetimes
    uuid_representation=UuidRepresentation.UNSPECIFIED,
    unicode_decode_error_handler="strict",
    tzinfo=None,                                      # Default timezone
    type_registry=None,                               # Custom type codecs
    datetime_conversion=DatetimeConversion.DATETIME,
)

# Clone with overrides
new_options = options.with_options(tz_aware=True)
```

### DatetimeConversion

```python
DatetimeConversion.DATETIME       # 1 - Standard datetime (default, may raise on out-of-range)
DatetimeConversion.DATETIME_CLAMP # 2 - Clamp to datetime.min/max
DatetimeConversion.DATETIME_MS    # 3 - Return DatetimeMS instead of datetime
DatetimeConversion.DATETIME_AUTO  # 4 - datetime if in range, else DatetimeMS
```

### Custom Type Codecs

```python
from bson.codec_options import TypeEncoder, TypeDecoder, TypeCodec, TypeRegistry

# Encode custom Python type → BSON type
class DecimalEncoder(TypeEncoder):
    python_type = Decimal
    def transform_python(self, value):
        return Decimal128(value)

# Decode BSON type → custom Python type
class DecimalDecoder(TypeDecoder):
    bson_type = Decimal128
    def transform_bson(self, value):
        return value.to_decimal()

# Combined encoder + decoder
class DecimalCodec(TypeCodec):
    python_type = Decimal
    bson_type = Decimal128
    def transform_python(self, value):
        return Decimal128(value)
    def transform_bson(self, value):
        return value.to_decimal()

# Register and use
registry = TypeRegistry([DecimalCodec()])
options = CodecOptions(type_registry=registry)
coll = db.get_collection("mycoll", codec_options=options)
```

---

## GridFS

Store and retrieve files larger than the 16MB BSON document limit.

**Source:** `gridfs/synchronous/grid_file.py`

### GridFSBucket (Recommended API)

```python
from gridfs import GridFSBucket

bucket = GridFSBucket(
    db,                           # Database instance
    bucket_name="fs",             # Collection prefix (default "fs")
    chunk_size_bytes=255 * 1024,  # 255KB default chunk size
    write_concern=None,
    read_preference=None,
)
```

**Upload:**

```python
# Stream upload
with bucket.open_upload_stream("my_file.txt", metadata={"type": "text"}) as grid_in:
    grid_in.write(b"Hello, World!")

# Upload from source
with open("local_file.pdf", "rb") as f:
    file_id = bucket.upload_from_stream("remote_name.pdf", f, metadata={"author": "Alice"})

# With explicit file_id
bucket.upload_from_stream_with_id(
    file_id=ObjectId(),
    filename="my_file.txt",
    source=b"content",
)
```

**Download:**

```python
# Stream download
grid_out = bucket.open_download_stream(file_id)
data = grid_out.read()

# Download to destination
with open("output.pdf", "wb") as f:
    bucket.download_to_stream(file_id, f)

# By filename (latest version by default)
grid_out = bucket.open_download_stream_by_name("my_file.txt", revision=-1)

# Download by name to destination
with open("output.txt", "wb") as f:
    bucket.download_to_stream_by_name("my_file.txt", f, revision=-1)
```

**Management:**

```python
bucket.delete(file_id)                           # Delete a file
bucket.delete_by_name("my_file.txt")             # Delete by filename
bucket.rename(file_id, "new_name.txt")           # Rename a file
bucket.rename_by_name("old.txt", "new.txt")      # Rename by name
bucket.find({"filename": "test.txt"})            # Query files → GridOutCursor
```

### GridIn (File Writer) Properties

```python
grid_in.closed      # -> bool
grid_in.writeable   # -> bool
grid_in.write(data)
grid_in.writelines(sequence)
grid_in.close()
grid_in.abort()     # Discard chunks and metadata
```

### GridOut (File Reader) Properties

```python
grid_out.read(size=-1)    # Read bytes
grid_out.readchunk()      # Read one chunk
grid_out.readline(size=-1)
grid_out.seek(pos, whence=0)
grid_out.tell             # Current position
grid_out.seekable         # -> bool (True)
grid_out.readable         # -> bool (True)
grid_out.close()
```

### Legacy GridFS API

```python
from gridfs import GridFS

fs = GridFS(db, collection="fs")

# Store
file_id = fs.put(b"data", filename="test.txt")
file_id = fs.put(open("file.txt", "rb"), filename="test.txt")

# Retrieve
grid_out = fs.get(file_id)
data = grid_out.read()

# By filename
grid_out = fs.get_last_version("test.txt")
grid_out = fs.get_version("test.txt", version=-1)

# Delete
fs.delete(file_id)

# List and find
filenames = fs.list()
grid_out = fs.find_one({"filename": "test.txt"})
for grid_out in fs.find({"filename": "test.txt"}):
    print(grid_out.read())

# Check existence
fs.exists(file_id)
fs.exists(filename="test.txt")
```

---

## Client-Side Encryption

### AutoEncryptionOpts

Configure automatic client-side field-level encryption.

**Source:** `pymongo/encryption_options.py`

```python
from pymongo.encryption_options import AutoEncryptionOpts

opts = AutoEncryptionOpts(
    kms_providers={                     # KMS provider credentials
        "local": {"key": local_master_key},
        # or "aws": {"accessKeyId": "...", "secretAccessKey": "..."}
        # or "azure": {"tenantId": "...", "clientId": "...", "clientSecret": "..."}
        # or "gcp": {"email": "...", "privateKey": "..."}
        # or "kmip": {"endpoint": "..."}
    },
    key_vault_namespace="mydb.__keyVault",  # namespace for data keys
    key_vault_client=None,                  # Separate client for key vault
    schema_map=None,                        # JSON schema for encryption
    bypass_auto_encryption=False,
    mongocryptd_uri="mongodb://localhost:27020",
    mongocryptd_bypass_spawn=False,
    mongocryptd_spawn_path="mongocryptd",
    mongocryptd_spawn_args=None,
    kms_tls_options=None,
    crypt_shared_lib_path=None,             # Path to crypt_shared library
    crypt_shared_lib_required=False,
    bypass_query_analysis=False,
    encrypted_fields_map=None,
    key_expiration_ms=None,
)

client = MongoClient(auto_encryption_opts=opts)
```

### ClientEncryption

Explicit encryption/decryption and key management.

**Source:** `pymongo/synchronous/encryption.py`

```python
from pymongo.encryption import ClientEncryption, Algorithm, QueryType

enc = ClientEncryption(
    kms_providers={"local": {"key": master_key}},
    key_vault_namespace="mydb.__keyVault",
    key_vault_client=client,
    codec_options=CodecOptions(),
    kms_tls_options=None,
    key_expiration_ms=None,
)

# Data key management
key_id = enc.create_data_key("local", key_alt_names=["my_key"])
enc.get_key(key_id)
enc.get_keys()                        # -> Cursor of all keys
enc.get_key_by_alt_name("my_key")
enc.add_key_alt_name(key_id, "alias")
enc.remove_key_alt_name(key_id, "alias")
enc.delete_key(key_id)
enc.rewrap_many_data_key(filter={}, provider="local", master_key=None)

# Encrypt / decrypt
encrypted = enc.encrypt(
    value="sensitive data",
    algorithm=Algorithm.AEAD_AES_256_CBC_HMAC_SHA_512_Deterministic,
    key_id=key_id,
    # or key_alt_name="my_key",
)
decrypted = enc.decrypt(encrypted)

# Create encrypted collection (Queryable Encryption)
enc.create_encrypted_collection(
    database=db,
    name="encrypted_coll",
    encrypted_fields={...},
    kms_provider="local",
    master_key=None,
)

enc.close()
```

### Algorithm Constants

```python
Algorithm.AEAD_AES_256_CBC_HMAC_SHA_512_Deterministic  # Same input → same output
Algorithm.AEAD_AES_256_CBC_HMAC_SHA_512_Random          # Random encryption
Algorithm.INDEXED     # Queryable Encryption indexed
Algorithm.UNINDEXED   # Queryable Encryption unindexed
Algorithm.RANGE       # Range queries on encrypted fields
```

### QueryType Constants

```python
QueryType.EQUALITY   # Equality queries on encrypted fields
QueryType.RANGE      # Range queries on encrypted fields
```

---

## Monitoring & Events

Register listeners to observe driver behavior.

**Source:** `pymongo/monitoring.py`

### Listener Base Classes

```python
from pymongo import monitoring

# Command monitoring
class MyCommandListener(monitoring.CommandListener):
    def started(self, event: monitoring.CommandStartedEvent):
        print(f"Command {event.command_name} started on {event.database_name}")

    def succeeded(self, event: monitoring.CommandSucceededEvent):
        print(f"Command {event.command_name} succeeded in {event.duration_micros}µs")

    def failed(self, event: monitoring.CommandFailedEvent):
        print(f"Command {event.command_name} failed: {event.failure}")

# Connection pool monitoring
class MyPoolListener(monitoring.ConnectionPoolListener):
    def pool_created(self, event): ...
    def pool_ready(self, event): ...
    def pool_cleared(self, event): ...
    def pool_closed(self, event): ...
    def connection_created(self, event): ...
    def connection_ready(self, event): ...
    def connection_closed(self, event): ...
    def connection_check_out_started(self, event): ...
    def connection_check_out_failed(self, event): ...
    def connection_checked_out(self, event): ...
    def connection_checked_in(self, event): ...

# Server heartbeat monitoring
class MyHeartbeatListener(monitoring.ServerHeartbeatListener):
    def started(self, event): ...
    def succeeded(self, event): ...
    def failed(self, event): ...

# Topology monitoring
class MyTopologyListener(monitoring.TopologyListener):
    def opened(self, event): ...
    def description_changed(self, event): ...
    def closed(self, event): ...

# Server monitoring
class MyServerListener(monitoring.ServerListener):
    def opened(self, event): ...
    def description_changed(self, event): ...
    def closed(self, event): ...
```

### Registering Listeners

```python
# Via MongoClient constructor
client = MongoClient(
    event_listeners=[MyCommandListener(), MyPoolListener()]
)

# Global registration (before creating clients)
monitoring.register(MyCommandListener())
```

### Event Properties

**CommandStartedEvent:**
- `command` — the command document
- `command_name` — e.g. "find", "insert"
- `database_name`
- `request_id`
- `connection_id`
- `operation_id`
- `service_id`
- `server_connection_id`

**CommandSucceededEvent** (adds):
- `duration_micros`
- `reply`

**CommandFailedEvent** (adds):
- `duration_micros`
- `failure`

**Connection events** have `address`, `connection_id`, `duration_micros`, `reason` as applicable.

---

## Errors & Exceptions

**Source:** `pymongo/errors.py`

### Exception Hierarchy

```
PyMongoError (base)
├── ProtocolError
├── ConnectionFailure
│   ├── WaitQueueTimeoutError
│   └── AutoReconnect
│       ├── NetworkTimeout
│       ├── NotPrimaryError
│       └── ServerSelectionTimeoutError
├── ConfigurationError
│   └── InvalidURI
├── OperationFailure
│   ├── CursorNotFound
│   ├── ExecutionTimeout
│   ├── WriteConcernError
│   │   └── WTimeoutError
│   ├── WriteError
│   │   └── DuplicateKeyError
│   ├── BulkWriteError
│   └── ClientBulkWriteException
├── InvalidOperation
├── InvalidName
├── CollectionInvalid
├── DocumentTooLarge
└── EncryptionError
```

### Common Exception Usage

```python
from pymongo.errors import (
    PyMongoError,
    ConnectionFailure,
    ServerSelectionTimeoutError,
    OperationFailure,
    DuplicateKeyError,
    BulkWriteError,
    ConfigurationError,
    InvalidOperation,
    InvalidURI,
    WriteConcernError,
    NetworkTimeout,
    AutoReconnect,
    ExecutionTimeout,
)

# Check for duplicate key
try:
    collection.insert_one({"_id": 1})
    collection.insert_one({"_id": 1})
except DuplicateKeyError as e:
    print(f"Duplicate key: {e.details}")

# Check for connection issues
try:
    client.admin.command("ping")
except ConnectionFailure:
    print("Cannot connect to MongoDB")

# Check for timeout
try:
    collection.find_one({"$where": "sleep(10000)"}, max_time_ms=1000)
except ExecutionTimeout:
    print("Query timed out")

# Error labels (for transaction retry logic)
try:
    session.commit_transaction()
except PyMongoError as e:
    if e.has_error_label("TransientTransactionError"):
        # Retry the whole transaction
        pass
    elif e.has_error_label("UnknownTransactionCommitResult"):
        # Retry commit
        pass
```

### PyMongoError Properties

```python
error.timeout              # -> bool   Whether this was a timeout error
error.has_error_label(label)  # -> bool   Check for specific error label
```

### OperationFailure Properties

```python
error.code     # -> Optional[int]   MongoDB error code
error.details  # -> Optional[dict]  Full error document
```

### BulkWriteError Properties

```python
error.details  # -> dict with keys:
               #    "writeErrors": list of individual write errors
               #    "writeConcernErrors": list of write concern errors
               #    "nInserted", "nMatched", "nModified", "nRemoved", "nUpserted"
```

---

## Connection Configuration Constants

**Source:** `pymongo/common.py`

| Constant | Default | Description |
|----------|---------|-------------|
| `MAX_BSON_SIZE` | 16 MB | Maximum BSON document size |
| `MAX_MESSAGE_SIZE` | 48 MB | Maximum wire protocol message size |
| `MAX_WRITE_BATCH_SIZE` | 100,000 | Maximum operations per bulk write |
| `MAX_POOL_SIZE` | 100 | Default max connections per server |
| `MIN_POOL_SIZE` | 0 | Default min connections per server |
| `MAX_CONNECTING` | 2 | Max simultaneous connection establishments |
| `CONNECT_TIMEOUT` | 20.0s | Default connection timeout |
| `SERVER_SELECTION_TIMEOUT` | 30s | Default server selection timeout |
| `HEARTBEAT_FREQUENCY` | 10s | Default heartbeat interval |
| `MIN_HEARTBEAT_INTERVAL` | 0.5s | Minimum heartbeat interval |
| `LOCAL_THRESHOLD_MS` | 15ms | Default local threshold for nearest reads |
| `RETRY_WRITES` | True | Default retryable writes setting |
| `RETRY_READS` | True | Default retryable reads setting |
| `MIN_SUPPORTED_SERVER_VERSION` | "4.2" | Minimum supported MongoDB version |

---

## PyMongo Module Exports

**Source:** `pymongo/__init__.py`

```python
import pymongo

# Version
pymongo.__version__       # e.g. "4.17.0"
pymongo.version           # Same as __version__
pymongo.version_tuple     # e.g. (4, 17, 0)

# Direction constants
pymongo.ASCENDING         # 1
pymongo.DESCENDING        # -1
pymongo.GEO2D             # "2d"
pymongo.GEOSPHERE         # "2dsphere"
pymongo.HASHED            # "hashed"
pymongo.TEXT              # "text"

# C extension check
pymongo.has_c             # bool - whether C extensions are available

# Timeout context manager
with pymongo.timeout(seconds=5.0):
    collection.insert_one({"x": 1})
    collection.find_one({"x": 1})  # Combined timeout for both operations

# Wire version support
pymongo.MAX_SUPPORTED_WIRE_VERSION
pymongo.MIN_SUPPORTED_WIRE_VERSION
```

---

## Migration Notes (PyMongo 3 → 4)

### Key Breaking Changes

**MongoClient:**
- `directConnection` defaults to `False`
- Cannot execute operations after `close()`
- Removed: `fsync()`, `unlock()`, `is_locked`, `database_names()`, `max_bson_size`, `max_message_size`
- SSL options renamed: `ssl_*` → `tls*` (e.g., `ssl_certfile` → `tlsCertificateKeyFile`)

**Database:**
- Removed: `authenticate()`, `logout()` — use MongoClient credentials
- Removed: `collection_names()` → use `list_collection_names()`
- Removed: `add_user()`, `remove_user()` → use `createUser`/`dropUser` commands
- `__bool__` now raises `NotImplementedError`

**Collection:**
- Removed: `insert()` → `insert_one()` / `insert_many()`
- Removed: `save()` → `insert_one()` / `update_one()`
- Removed: `update()` → `update_one()` / `update_many()`
- Removed: `remove()` → `delete_one()` / `delete_many()`
- Removed: `find_and_modify()` → `find_one_and_update()` / `find_one_and_replace()` / `find_one_and_delete()`
- Removed: `count()` → `count_documents()` / `estimated_document_count()`
- Removed: `ensure_index()` → `create_index()`
- Removed: `group()`, `map_reduce()`, `inline_map_reduce()`
- `hint` now required with `min`/`max` queries
- Empty projections return entire document (not just `_id`)

**BSON:**
- Default UUID representation changed to `UNSPECIFIED` (raises error by default — must explicitly choose)
- Default JSON mode changed from `LEGACY` to `RELAXED`
- `SON.items()` returns `dict_items` (not list)

**Removed entirely:**
- `CursorManager`, `MongoClient.close_cursor()`, `MongoClient.kill_cursors()`
- `Database.eval()`, `Database.system_js`
- `Collection.parallel_scan()`
- `IsMaster` class (→ `Hello`)
- `NotMasterError` (→ `NotPrimaryError`)

### Migration Pattern Examples

```python
# OLD (PyMongo 3)
collection.insert({"x": 1})
collection.save(doc)
collection.update({"x": 1}, {"$set": {"y": 2}})
collection.remove({"x": 1})
collection.find_and_modify({"x": 1}, {"$set": {"y": 2}})
count = collection.count()
db.authenticate("user", "pass")
collection.ensure_index("field")

# NEW (PyMongo 4)
collection.insert_one({"x": 1})
collection.insert_one(doc)  # or update_one with upsert
collection.update_one({"x": 1}, {"$set": {"y": 2}})
collection.delete_one({"x": 1})
collection.find_one_and_update({"x": 1}, {"$set": {"y": 2}})
count = collection.count_documents({})
# Pass credentials to MongoClient constructor
client = MongoClient(username="user", password="pass")
collection.create_index("field")
```
