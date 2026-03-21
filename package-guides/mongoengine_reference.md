# MongoEngine Reference (v0.29.0)

A comprehensive reference for building applications with MongoEngine, the Python Object-Document Mapper (ODM) for MongoDB.

---

## Table of Contents

1. [Installation & Connection](#1-installation--connection)
2. [Defining Documents](#2-defining-documents)
3. [Field Types](#3-field-types)
4. [Document Meta Options](#4-document-meta-options)
5. [Document Instance Methods](#5-document-instance-methods)
6. [Querying (QuerySet API)](#6-querying-queryset-api)
7. [Query Operators](#7-query-operators)
8. [Update Operators](#8-update-operators)
9. [Indexes](#9-indexes)
10. [Signals](#10-signals)
11. [Context Managers](#11-context-managers)
12. [GridFS (File Storage)](#12-gridfs-file-storage)
13. [Document Inheritance](#13-document-inheritance)
14. [References & Delete Rules](#14-references--delete-rules)
15. [Text Search](#15-text-search)
16. [Geospatial Queries](#16-geospatial-queries)
17. [Transactions](#17-transactions)
18. [Errors & Exceptions](#18-errors--exceptions)

---

## 1. Installation & Connection

### Install

```bash
pip install mongoengine
```

### Connecting to MongoDB

```python
from mongoengine import connect, disconnect, disconnect_all, get_db, get_connection

# Simple connection (defaults: localhost:27017, database "test")
connect()

# Named database
connect('my_database')

# Full URI (preferred)
connect(host="mongodb://user:pass@host:27017/mydb?authSource=admin&ssl=true&replicaSet=rs0")

# Keyword arguments
connect(
    db='my_database',
    host='192.168.1.100',
    port=27017,
    username='admin',
    password='secret',
    authentication_source='admin',
    authentication_mechanism='SCRAM-SHA-256',
)

# Multiple databases via aliases
connect(db='users_db', alias='users')
connect(db='analytics_db', alias='analytics')

# Disconnect
disconnect()              # Default connection
disconnect(alias='users') # Named connection
disconnect_all()          # All connections

# Get PyMongo objects directly
db = get_db(alias='default')
client = get_connection(alias='default')
```

**Connection functions:**

| Function | Signature | Description |
|----------|-----------|-------------|
| `connect` | `connect(db=None, alias='default', **kwargs)` | Create database connection |
| `register_connection` | `register_connection(alias, db=None, host=None, port=None, ...)` | Register without connecting |
| `get_connection` | `get_connection(alias='default', reconnect=False)` | Get MongoClient object |
| `get_db` | `get_db(alias='default', reconnect=False)` | Get Database object |
| `disconnect` | `disconnect(alias='default')` | Close specific connection |
| `disconnect_all` | `disconnect_all()` | Close all connections |

**Constants:**
- `DEFAULT_CONNECTION_NAME = "default"`
- `DEFAULT_DATABASE_NAME = "test"`

---

## 2. Defining Documents

### Document Classes

```python
from mongoengine import (
    Document, EmbeddedDocument, DynamicDocument,
    DynamicEmbeddedDocument
)

# Standard document (stored in its own collection)
class User(Document):
    name = StringField(required=True)
    email = EmailField(unique=True)
    meta = {'collection': 'users'}

# Embedded document (no own collection; nested inside another document)
class Address(EmbeddedDocument):
    street = StringField()
    city = StringField()

# Dynamic document (allows undefined fields)
class Config(DynamicDocument):
    name = StringField()
    # Any additional fields accepted at runtime

# Dynamic embedded document
class Metadata(DynamicEmbeddedDocument):
    created_by = StringField()
```

### Class Hierarchy

```
BaseDocument
├── EmbeddedDocument
│   └── DynamicEmbeddedDocument
└── Document
    └── DynamicDocument
```

### Abstract Documents

```python
class BaseContent(Document):
    created_at = DateTimeField()
    updated_at = DateTimeField()
    meta = {'abstract': True}  # No collection, no _cls field

class Article(BaseContent):
    title = StringField()
    # Stored in 'article' collection with created_at, updated_at, title
```

---

## 3. Field Types

### BaseField Common Parameters

All fields accept these keyword arguments in `__init__`:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `db_field` | str | None | MongoDB field name (if different from attribute name) |
| `required` | bool | False | Field must have a value |
| `default` | value/callable | None | Default value or callable returning default |
| `unique` | bool | False | Field value must be unique |
| `unique_with` | str/list | None | Compound uniqueness with other field(s) |
| `primary_key` | bool | False | Use as primary key (`_id`) |
| `choices` | list/enum | None | Restrict to specific values |
| `null` | bool | False | Allow None to be stored in DB |
| `sparse` | bool | False | Sparse index (skip docs where field is missing) |
| `validation` | callable | None | Custom validation function |

### String Fields

```python
StringField(regex=None, max_length=None, min_length=None, **kwargs)
```
- Validates: type (str), length bounds, regex pattern match.

```python
URLField(url_regex=None, schemes=None, **kwargs)
```
- Default schemes: `["http", "https", "ftp", "ftps"]`
- Validates full URL format.

```python
EmailField(domain_whitelist=None, allow_utf8_user=False, allow_ip_domain=False, **kwargs)
```
- RFC 5322 compliant. Supports IDN encoding, IPv4/IPv6 domains.

### Numeric Fields

```python
IntField(min_value=None, max_value=None, **kwargs)
```
- Converts to `int`. Validates bounds.

```python
FloatField(min_value=None, max_value=None, **kwargs)
```
- Accepts int or float. Validates bounds.

```python
DecimalField(min_value=None, max_value=None, force_string=False, precision=2,
             rounding=decimal.ROUND_HALF_UP, **kwargs)
```
- Legacy field with potential precision issues. Prefer `Decimal128Field` for monetary data.
- `force_string=True` stores as string to avoid float precision loss.

```python
Decimal128Field(min_value=None, max_value=None, **kwargs)
```
- True 128-bit decimal storage via BSON Decimal128. Returns `decimal.Decimal` to Python.

### Boolean Field

```python
BooleanField(**kwargs)
```
- Strict type check: must be `bool`.

### Date/Time Fields

```python
DateTimeField(**kwargs)
```
- Microseconds rounded to milliseconds. Parses strings via `dateutil` if available.

```python
DateField(**kwargs)
```
- Inherits from DateTimeField. Strips time components.

```python
ComplexDateTimeField(separator=",", **kwargs)
```
- Stored as string with exact microsecond precision. Format: `YYYY,MM,DD,HH,MM,SS,NNNNNN`.

### Embedded Document Fields

```python
EmbeddedDocumentField(document_type, **kwargs)
```
- `document_type`: EmbeddedDocument class or `'self'` for recursive reference.

```python
GenericEmbeddedDocumentField(**kwargs)
```
- Accepts any EmbeddedDocument subclass. Stores `_cls` for type resolution.

### Container Fields

```python
ListField(field=None, *, max_length=None, **kwargs)
```
- `field`: Optional inner field type for validation (e.g., `ListField(StringField())`).
- Default value: `[]`.

```python
SortedListField(field, ordering=None, reverse=False, **kwargs)
```
- Auto-sorts on write. `ordering` specifies sort key for embedded documents.

```python
EmbeddedDocumentListField(document_type, **kwargs)
```
- Shorthand for `ListField(EmbeddedDocumentField(MyDoc))`.
- Returns `EmbeddedDocumentList` with `.filter()`, `.exclude()`, `.get()`, `.create()`, `.delete()`, `.update()` methods.

```python
DictField(field=None, **kwargs)
```
- `field`: Optional value field type. Keys must be strings, cannot start with `$`.
- Default value: `{}`.

```python
MapField(field=None, **kwargs)
```
- Like DictField but `field` is required. Enforces value type.

### Reference Fields

```python
ReferenceField(document_type, dbref=False, reverse_delete_rule=DO_NOTHING, **kwargs)
```
- `document_type`: Document class, `'self'`, or class name string.
- `dbref=True`: Store as DBRef instead of ObjectId.
- `reverse_delete_rule`: `DO_NOTHING`, `NULLIFY`, `CASCADE`, `DENY`, `PULL`.
- Auto-dereferences on access (eager loading).

```python
LazyReferenceField(document_type, passthrough=False, dbref=False,
                   reverse_delete_rule=DO_NOTHING, **kwargs)
```
- Returns `LazyReference` proxy. Call `.fetch()` to load.
- `passthrough=True`: Access fields directly without `.fetch()`.

```python
CachedReferenceField(document_type, fields=None, auto_sync=True, **kwargs)
```
- `fields`: List of field names to cache in the referencing document.
- `auto_sync=True`: Auto-updates cached data on save.

```python
GenericReferenceField(**kwargs)
```
- References any Document subclass. Stores `_cls` and `_ref` (DBRef).
- Less efficient than typed ReferenceField.

```python
GenericLazyReferenceField(passthrough=False, **kwargs)
```
- Lazy version of GenericReferenceField.

### Other Fields

```python
BooleanField(**kwargs)
```

```python
BinaryField(max_bytes=None, **kwargs)
```
- Stores `bytes` as BSON Binary.

```python
ObjectIdField(**kwargs)
```
- MongoDB ObjectId. Auto-generated for `_id` field.

```python
EnumField(enum, **kwargs)
```
- `enum`: Python `Enum` class. Stores `enum.value` in MongoDB.

```python
UUIDField(binary=True, **kwargs)
```
- `binary=True`: Store as BSON Binary (efficient). `False`: Store as string.

```python
SequenceField(collection_name=None, db_alias=None, sequence_name=None,
              value_decorator=None, **kwargs)
```
- Auto-incrementing counter using a separate MongoDB collection.
- Default counter collection: `"mongoengine.counters"`.
- `value_decorator`: Callable to transform counter value (default: `int`).

```python
DynamicField(**kwargs)
```
- Accepts any type. Used internally by DynamicDocument.

### File Fields

```python
FileField(db_alias=DEFAULT_CONNECTION_NAME, collection_name="fs", **kwargs)
```
- Stores files in GridFS. See [GridFS section](#12-gridfs-file-storage).

```python
ImageField(size=None, thumbnail_size=None, collection_name="images", **kwargs)
```
- `size`: `{'width': W, 'height': H, 'force': bool}` for auto-resize.
- `thumbnail_size`: Same format for thumbnail generation.
- Requires Pillow library.

### GeoJSON Fields

All GeoJSON fields inherit from `GeoJsonBaseField` and auto-create `2dsphere` indexes.

```python
PointField(**kwargs)           # {'type': 'Point', 'coordinates': [lng, lat]}
LineStringField(**kwargs)      # {'type': 'LineString', 'coordinates': [[lng, lat], ...]}
PolygonField(**kwargs)         # {'type': 'Polygon', 'coordinates': [[[lng, lat], ...]]}
MultiPointField(**kwargs)      # {'type': 'MultiPoint', 'coordinates': [[lng, lat], ...]}
MultiLineStringField(**kwargs) # {'type': 'MultiLineString', ...}
MultiPolygonField(**kwargs)    # {'type': 'MultiPolygon', ...}
```

```python
GeoPointField(**kwargs)  # DEPRECATED legacy 2D point: [x, y]
```

### Complete Field Hierarchy

```
BaseField
├── StringField
│   ├── URLField
│   └── EmailField
├── IntField
├── FloatField
├── DecimalField
├── Decimal128Field
├── BooleanField
├── DateTimeField
│   └── DateField
├── ComplexDateTimeField (extends StringField)
├── EmbeddedDocumentField
├── GenericEmbeddedDocumentField
├── DynamicField
├── ComplexBaseField
│   ├── ListField
│   │   ├── SortedListField
│   │   └── EmbeddedDocumentListField
│   └── DictField
│       └── MapField
├── ReferenceField
├── CachedReferenceField
├── LazyReferenceField
├── GenericReferenceField
│   └── GenericLazyReferenceField
├── BinaryField
├── EnumField
├── SequenceField
├── UUIDField
├── ObjectIdField
├── FileField
│   └── ImageField
├── GeoPointField (deprecated)
└── GeoJsonBaseField
    ├── PointField
    ├── LineStringField
    ├── PolygonField
    ├── MultiPointField
    ├── MultiLineStringField
    └── MultiPolygonField
```

---

## 4. Document Meta Options

Set via the `meta` dictionary on Document classes:

```python
class MyDoc(Document):
    name = StringField()
    meta = {
        'collection': 'my_collection',     # MongoDB collection name (default: snake_case class name)
        'db_alias': 'default',             # Connection alias
        'abstract': False,                 # If True, no collection created
        'allow_inheritance': False,        # Enable subclassing with _cls field
        'strict': True,                    # Reject undefined fields
        'ordering': ['-created_at'],       # Default query ordering
        'queryset_class': QuerySet,        # Custom QuerySet class

        # Indexes
        'indexes': [],                     # List of index specifications
        'auto_create_index': True,         # Auto-create on first collection access
        'auto_create_index_on_save': False, # Re-ensure on every save
        'index_background': False,         # Build indexes in background
        'index_cls': True,                 # Add _cls to indexes when allow_inheritance=True
        'index_opts': {},                  # Default options for all indexes

        # Primary key
        'id_field': 'id',                  # Name of primary key field

        # Capped collection
        'max_size': None,                  # Max bytes (creates capped collection)
        'max_documents': None,             # Max document count

        # Timeseries collection
        'timeseries': {
            'timeField': 'timestamp',
            'metaField': 'metadata',
            'granularity': 'seconds',      # 'seconds', 'minutes', 'hours'
            'expireAfterSeconds': 3600,
        },

        # Sharding
        'shard_key': ('field1', 'field2'), # Shard key (immutable after first save)

        # Cascading
        'cascade': False,                  # Auto-save referenced documents
    }
```

---

## 5. Document Instance Methods

### Creating & Saving

```python
# Create instance
doc = MyDoc(name='Alice', age=30)

# Save (insert or update)
doc.save(
    force_insert=False,       # Only insert, never update
    validate=True,            # Run validation before save
    clean=True,               # Call clean() method (requires validate=True)
    write_concern=None,       # MongoDB write concern dict, e.g. {'w': 1, 'j': True}
    cascade=None,             # Recursively save referenced documents
    cascade_kwargs=None,      # Override kwargs for cascade saves
    save_condition=None,      # Query condition for conditional save
    signal_kwargs=None,       # Extra kwargs passed to signal handlers
)
# Returns: self
```

### Updating

```python
# Atomic modify (find-and-modify)
doc.modify(query=None, **update)
# Example: doc.modify(set__name='Bob', inc__counter=1)
# Returns: True if updated, False if no match

# Convenience update via queryset
doc.update(**kwargs)
# Example: doc.update(set__name='Bob')
```

### Deleting

```python
doc.delete(signal_kwargs=None, **write_concern)
# Sends pre_delete/post_delete signals
# Also deletes associated FileField/ImageField GridFS files
```

### Reloading

```python
doc.reload(*fields, max_depth=1)
# Reload specific fields or entire document from database
# Returns: self
# Raises: DoesNotExist if document was deleted
```

### Validation

```python
doc.validate(clean=True)
# Validates all fields. Raises ValidationError with error dict.

doc.clean()
# Override for custom document-level validation/data cleaning.
# Errors raised here stored under NON_FIELD_ERRORS key.
```

### Serialization

```python
# To MongoDB dict (SON)
son = doc.to_mongo(use_db_field=True, fields=None)

# To JSON
json_str = doc.to_json()

# From JSON
doc = MyDoc.from_json(json_data, created=False)

# To DBRef
dbref = doc.to_dbref()
```

### Field Change Tracking

```python
doc._get_changed_fields()
# Returns: list of changed field names (dotted paths for nested)

doc._clear_changed_fields()
# Resets change tracking (called automatically after save)

doc._delta()
# Returns: (set_data, unset_data) tuple for MongoDB $set/$unset
```

### Database/Collection Access

```python
# Primary key
doc.pk   # Alias for doc.id

# Switch database temporarily
doc.switch_db('archive_alias', keep_created=True)

# Switch collection temporarily
doc.switch_collection('backup_collection', keep_created=True)

# Dereference references
doc.select_related(max_depth=1)
```

### Class Methods

```python
# Get PyMongo objects
MyDoc._get_db()          # PyMongo Database
MyDoc._get_collection()  # PyMongo Collection

# Drop collection
MyDoc.drop_collection()

# Indexes
MyDoc.ensure_indexes()
MyDoc.create_index(keys, background=False, **kwargs)
MyDoc.list_indexes()     # Returns list of index field tuples
MyDoc.compare_indexes()  # Returns {'missing': [...], 'extra': [...]}

# Field lookup
MyDoc._lookup_field(['address', 'city'])  # Returns [Field, Field]
MyDoc._translate_field_name('address.city')  # Returns db field path
```

---

## 6. Querying (QuerySet API)

Access via `Document.objects` (a `QuerySetManager`).

### Retrieving Documents

```python
# All documents
MyDoc.objects.all()
MyDoc.objects()           # Same as all()

# Filter with conditions
MyDoc.objects.filter(age__gte=18, status='active')
MyDoc.objects(age__gte=18, status='active')  # Shorthand

# Single document
doc = MyDoc.objects.get(id=some_id)        # Raises DoesNotExist or MultipleObjectsReturned
doc = MyDoc.objects.first()                 # First match or None
doc = MyDoc.objects.with_id(some_id)        # By primary key, no extra filters

# Bulk fetch
docs = MyDoc.objects.in_bulk([id1, id2])   # Returns {id: doc} dict

# Create
doc = MyDoc.objects.create(name='Alice')   # Create and save immediately

# Empty queryset
MyDoc.objects.none()                        # Returns no results
```

### Filtering & Excluding

```python
# Filter
MyDoc.objects.filter(name='Alice')
MyDoc.objects.filter(Q(age__gt=18) | Q(verified=True))
MyDoc.objects.exclude(status='deleted')  # Exclude matching documents

# Raw queries
MyDoc.objects(__raw__={'$or': [{'name': 'Alice'}, {'age': {'$gt': 18}}]})

# JavaScript where clause
MyDoc.objects.where('this.name.length > 10')

# Subclass filtering (with inheritance)
MyDoc.objects.no_sub_classes()  # Only exact class, not subclasses
```

### Ordering & Pagination

```python
MyDoc.objects.order_by('name')            # Ascending
MyDoc.objects.order_by('-created_at')     # Descending
MyDoc.objects.order_by('+name', '-age')   # Multiple fields

MyDoc.objects.skip(20).limit(10)          # Pagination
MyDoc.objects[20:30]                      # Slice syntax (same as skip/limit)
MyDoc.objects[0]                          # Single item by index
MyDoc.objects.batch_size(50)              # Cursor batch size
```

### Field Projection

```python
MyDoc.objects.only('name', 'email')       # Include only these fields
MyDoc.objects.exclude('password')          # Exclude these fields
MyDoc.objects.fields(slice__tags=5)        # $slice: first 5 tags
MyDoc.objects.all_fields()                 # Reset projection

# Scalar values (tuples instead of documents)
MyDoc.objects.scalar('name', 'age')
MyDoc.objects.values_list('name', 'age')  # Same as scalar
```

### Aggregation

```python
MyDoc.objects.count(with_limit_and_skip=False)
MyDoc.objects.sum('score')
MyDoc.objects.average('score')
MyDoc.objects.distinct('category')  # Returns list of unique values

# MongoDB aggregation pipeline
results = MyDoc.objects.aggregate([
    {'$match': {'status': 'active'}},
    {'$group': {'_id': '$category', 'total': {'$sum': 1}}},
    {'$sort': {'total': -1}},
])

# MapReduce
results = MyDoc.objects.map_reduce(
    map_f='function() { emit(this.category, 1); }',
    reduce_f='function(key, values) { return Array.sum(values); }',
    output='inline',
)
```

### Bulk Operations

```python
# Bulk insert
docs = [MyDoc(name='A'), MyDoc(name='B')]
MyDoc.objects.insert(docs, load_bulk=True, write_concern=None, signal_kwargs=None)
# Returns: list of inserted documents (if load_bulk=True) or list of IDs

# Bulk update
MyDoc.objects(status='draft').update(set__status='published')
# Returns: number of modified documents

# Bulk delete
MyDoc.objects(status='deleted').delete()
# Returns: number of deleted documents
```

### Update Methods

```python
# Update multiple documents (default)
MyDoc.objects(active=True).update(
    upsert=False,
    multi=True,
    write_concern=None,
    full_result=False,       # Return full MongoDB result dict
    array_filters=None,      # Array filter conditions
    **update                 # Update operators (see section 8)
)

# Update single document
MyDoc.objects(name='Alice').update_one(set__age=31)

# Upsert (update or insert)
MyDoc.objects(name='Alice').upsert_one(set__age=31)
# Returns: the updated/inserted document

# Atomic modify and return
doc = MyDoc.objects(name='Alice').modify(
    upsert=False,
    remove=False,            # If True, delete and return document
    new=False,               # If True, return modified document (not original)
    set__age=31,
)
```

### Reference Handling

```python
MyDoc.objects.select_related(max_depth=1)  # Eagerly dereference references
MyDoc.objects.no_dereference()              # Return raw references (DBRef/ObjectId)
```

### Cursor & Query Options

```python
MyDoc.objects.hint([('field', 1)])          # Specify index to use
MyDoc.objects.comment('admin dashboard')    # Add query comment
MyDoc.objects.max_time_ms(5000)             # Server-side timeout (ms)
MyDoc.objects.allow_disk_use(True)          # Allow temp files for sorting
MyDoc.objects.timeout(True)                 # Enable/disable cursor timeout
MyDoc.objects.collation({'locale': 'en'})   # Language-specific comparison
MyDoc.objects.read_preference(ReadPreference.SECONDARY)
MyDoc.objects.read_concern({'level': 'majority'})
```

### Output Formats

```python
MyDoc.objects.as_pymongo()                  # Raw dicts instead of Document instances
MyDoc.objects.to_json()                     # JSON string
MyDoc.objects.from_json(json_data)          # Documents from JSON
MyDoc.objects.explain()                     # Query explain plan
```

### QuerySet Variants

```python
# Default: caches results in memory
qs = MyDoc.objects.all()  # QuerySet (caching)

# No caching (lower memory for large result sets)
qs = MyDoc.objects.no_cache()  # QuerySetNoCache
```

### Custom Managers

```python
class ActiveQuerySet(QuerySet):
    def active(self):
        return self.filter(is_active=True)

class User(Document):
    is_active = BooleanField(default=True)
    meta = {'queryset_class': ActiveQuerySet}

# Or with decorator
class User(Document):
    is_active = BooleanField(default=True)

    @queryset_manager
    def active_users(doc_cls, queryset):
        return queryset.filter(is_active=True)

# Usage
User.active_users()  # Returns filtered queryset
```

### Q Objects (Complex Queries)

```python
from mongoengine import Q

# OR
User.objects(Q(age__lt=18) | Q(age__gt=65))

# AND
User.objects(Q(name='Alice') & Q(active=True))

# Nested
User.objects(
    (Q(role='admin') | Q(role='superuser')) & Q(active=True)
)
```

---

## 7. Query Operators

Use in `filter()`, `get()`, `exclude()`, etc., via double-underscore syntax: `field__operator=value`.

### Comparison Operators

| Operator | MongoDB | Example | Description |
|----------|---------|---------|-------------|
| (none) | `$eq` | `name='Alice'` | Exact match (default) |
| `ne` | `$ne` | `name__ne='Bob'` | Not equal |
| `gt` | `$gt` | `age__gt=18` | Greater than |
| `gte` | `$gte` | `age__gte=18` | Greater than or equal |
| `lt` | `$lt` | `age__lt=65` | Less than |
| `lte` | `$lte` | `age__lte=65` | Less than or equal |
| `in` | `$in` | `status__in=['active', 'pending']` | In list |
| `nin` | `$nin` | `status__nin=['deleted']` | Not in list |
| `mod` | `$mod` | `count__mod=[10, 0]` | Modulo (divisor, remainder) |
| `not` | `$not` | `name__not=re.compile('^A')` | Negate condition |
| `exists` | `$exists` | `email__exists=True` | Field exists/missing |
| `type` | `$type` | `value__type=2` | BSON type number |

### Array Operators

| Operator | MongoDB | Example | Description |
|----------|---------|---------|-------------|
| `all` | `$all` | `tags__all=['python', 'mongodb']` | All elements present |
| `size` | `$size` | `tags__size=3` | Exact array length |
| `elemMatch` | `$elemMatch` | `scores__elemMatch={'$gt': 90}` | Element matches criteria |

### String Operators

| Operator | MongoDB | Example | Description |
|----------|---------|---------|-------------|
| `exact` | regex | `name__exact='Alice'` | Exact match |
| `iexact` | regex(i) | `name__iexact='alice'` | Case-insensitive exact |
| `contains` | regex | `name__contains='lic'` | Substring match |
| `icontains` | regex(i) | `name__icontains='lic'` | Case-insensitive substring |
| `startswith` | regex | `name__startswith='Al'` | Starts with |
| `istartswith` | regex(i) | `name__istartswith='al'` | Case-insensitive starts with |
| `endswith` | regex | `name__endswith='ce'` | Ends with |
| `iendswith` | regex(i) | `name__iendswith='CE'` | Case-insensitive ends with |
| `regex` | `$regex` | `name__regex=r'^A.*e$'` | Regular expression |
| `iregex` | `$regex`(i) | `name__iregex=r'^a.*e$'` | Case-insensitive regex |
| `wholeword` | regex | `bio__wholeword='python'` | Whole word match |
| `iwholeword` | regex(i) | `bio__iwholeword='python'` | Case-insensitive whole word |

### Geo Operators (Legacy 2D)

| Operator | Description |
|----------|-------------|
| `near` | Near a point |
| `near_sphere` | Near on sphere |
| `within_distance` | Within distance from point |
| `within_spherical_distance` | Within spherical distance |
| `within_box` | Within bounding box |
| `within_polygon` | Within polygon |
| `max_distance` | Maximum distance constraint |
| `min_distance` | Minimum distance constraint |

### Geo Operators (GeoJSON)

| Operator | Description |
|----------|-------------|
| `geo_within` | Within geometry |
| `geo_within_box` | Within bounding box |
| `geo_within_polygon` | Within polygon |
| `geo_within_center` | Within circle center |
| `geo_within_sphere` | Within sphere |
| `geo_intersects` | Intersects geometry |

### Nested Field Access

Use double underscores for nested fields:

```python
# Embedded document field
User.objects(address__city='Portland')

# List element by index
User.objects(tags__0='python')

# Nested embedded document
User.objects(profile__settings__theme='dark')
```

---

## 8. Update Operators

Use in `update()`, `update_one()`, `modify()`, etc., via double-underscore prefix: `operator__field=value`.

| Operator | MongoDB | Example | Description |
|----------|---------|---------|-------------|
| `set` | `$set` | `set__name='Alice'` | Set field value (default if no operator given) |
| `unset` | `$unset` | `unset__email=1` | Remove field |
| `inc` | `$inc` | `inc__counter=1` | Increment (negative for decrement) |
| `dec` | `$inc` | `dec__counter=1` | Decrement (converts to negative inc) |
| `mul` | `$mul` | `mul__price=1.1` | Multiply |
| `min` | `$min` | `min__low_score=50` | Set if less than current |
| `max` | `$max` | `max__high_score=100` | Set if greater than current |
| `rename` | `$rename` | `rename__old_name='new_name'` | Rename field |
| `push` | `$push` | `push__tags='python'` | Append to array |
| `push_all` | `$push` | `push_all__tags=['a', 'b']` | Append multiple to array |
| `pop` | `$pop` | `pop__tags=1` | Remove last (1) or first (-1) |
| `pull` | `$pull` | `pull__tags='java'` | Remove matching elements |
| `pull_all` | `$pullAll` | `pull_all__tags=['a', 'b']` | Remove all matching |
| `add_to_set` | `$addToSet` | `add_to_set__tags='python'` | Add if not present |
| `set_on_insert` | `$setOnInsert` | `set_on_insert__created=now` | Set only on upsert insert |

### Examples

```python
# Simple update
User.objects(id=user_id).update(set__name='Alice', inc__login_count=1)

# Array operations
User.objects(id=user_id).update(push__tags='python')
User.objects(id=user_id).update(pull__tags='java')
User.objects(id=user_id).update(add_to_set__tags='mongodb')

# Nested field update
User.objects(id=user_id).update(set__profile__bio='New bio')

# Upsert with set_on_insert
User.objects(email='alice@example.com').update(
    upsert=True,
    set__name='Alice',
    set_on_insert__created_at=datetime.utcnow(),
)

# Multiple operators combined
Post.objects(id=post_id).update(
    inc__view_count=1,
    set__last_viewed=datetime.utcnow(),
    push__viewers=user_id,
)
```

---

## 9. Indexes

### Defining Indexes

```python
class MyDoc(Document):
    title = StringField()
    rating = FloatField()
    created = DateTimeField()
    location = PointField()
    tags = ListField(StringField())

    meta = {
        'indexes': [
            'title',                          # Single field ascending
            '-rating',                        # Single field descending
            ('title', '-rating'),             # Compound index
            '$title',                         # Text index
            '#title',                         # Hashed index

            # Full specification dict
            {
                'fields': ['created'],
                'expireAfterSeconds': 3600,   # TTL index
            },
            {
                'fields': ['title', 'rating'],
                'unique': True,
                'sparse': True,
                'name': 'title_rating_unique',
                'collation': {'locale': 'en', 'strength': 2},  # Case-insensitive
                'cls': False,                 # Don't include _cls
            },
        ],
    }
```

### Index Direction Prefixes

| Prefix | Type | PyMongo Constant |
|--------|------|------------------|
| `+` or none | Ascending | `ASCENDING` |
| `-` | Descending | `DESCENDING` |
| `$` | Text | `TEXT` |
| `#` | Hashed | `HASHED` |
| `(` | Geosphere | `GEOSPHERE` |
| `)` | GeoHaystack | `GEOHAYSTACK` |
| `*` | Geo2D | `GEO2D` |

### Field-Level Unique Constraints

```python
class User(Document):
    email = StringField(unique=True)
    username = StringField(unique_with='tenant_id')  # Compound unique
    tenant_id = StringField()
```

### Index Management

```python
MyDoc.ensure_indexes()         # Create all defined indexes
MyDoc.create_index(['title'])  # Create specific index
MyDoc.list_indexes()           # List expected index specs
MyDoc.compare_indexes()        # {'missing': [...], 'extra': [...]}

# Drop index (via PyMongo)
MyDoc._get_collection().drop_index('index_name')
```

### Text Indexes with Weights

```python
class Article(Document):
    title = StringField()
    content = StringField()
    meta = {
        'indexes': [{
            'fields': ['$title', '$content'],
            'weights': {'title': 10, 'content': 2},
            'default_language': 'english',
        }]
    }
```

---

## 10. Signals

Requires the `blinker` library (`pip install blinker`).

### Available Signals

| Signal | Sender | Extra kwargs | When |
|--------|--------|-------------|------|
| `pre_init` | Document class | `document`, `values` | Before `__init__` completes |
| `post_init` | Document class | `document` | After `__init__` completes |
| `pre_save` | Document class | `document` | Before `save()` |
| `pre_save_post_validation` | Document class | `document`, `created` | After validation, before DB write |
| `post_save` | Document class | `document`, `created` | After successful save |
| `pre_delete` | Document class | `document` | Before `delete()` |
| `post_delete` | Document class | `document` | After successful delete |
| `pre_bulk_insert` | Document class | `documents` | Before bulk insert |
| `post_bulk_insert` | Document class | `documents`, `loaded` | After bulk insert |

### Connecting Handlers

```python
from mongoengine import signals

# Method 1: Decorator
@signals.post_save.connect_via(User)
def user_saved(sender, document, created, **kwargs):
    if created:
        send_welcome_email(document.email)

# Method 2: Class method in document
class User(Document):
    name = StringField()

    @classmethod
    def post_save(cls, sender, document, created, **kwargs):
        if created:
            send_welcome_email(document.email)

# Register class method handler
signals.post_save.connect(User.post_save, sender=User)

# Method 3: Direct connect
def on_save(sender, document, **kwargs):
    pass

signals.pre_save.connect(on_save)  # All documents
signals.pre_save.connect(on_save, sender=User)  # Only User
```

---

## 11. Context Managers

```python
from mongoengine.context_managers import (
    switch_db, switch_collection, no_dereference,
    no_sub_classes, query_counter, set_write_concern,
    set_read_write_concern, run_in_transaction,
)
```

### switch_db

```python
with switch_db(User, 'archive-db') as User:
    User(name='archived').save()  # Saves to archive-db database
```

### switch_collection

```python
with switch_collection(Group, 'group_backup') as Group:
    Group(name='backup').save()  # Saves to group_backup collection
```

### no_dereference

```python
with no_dereference(Post):
    posts = Post.objects()  # ReferenceFields return raw ObjectId/DBRef
```

### no_sub_classes

```python
with no_sub_classes(Page):
    pages = Page.objects()  # Only Page instances, no DatedPage etc.
```

### query_counter

```python
with query_counter() as q:
    User.objects.count()
    print(int(q))  # Number of queries executed
    assert q == 1
    # Supports: ==, !=, <, <=, >, >=
```

### set_write_concern / set_read_write_concern

```python
collection = User._get_collection()

with set_write_concern(collection, {'w': 'majority', 'j': True}):
    # All writes use this concern
    pass

with set_read_write_concern(collection,
                           write_concerns={'w': 1},
                           read_concerns={'level': 'majority'}):
    pass
```

### run_in_transaction

```python
with run_in_transaction():
    user = User(name='Alice')
    user.save()
    Profile(user=user).save()
    # Auto-commits on successful exit; auto-aborts on exception
```

---

## 12. GridFS (File Storage)

### FileField Usage

```python
class Animal(Document):
    name = StringField()
    photo = FileField(collection_name='fs')  # Default GridFS collection

animal = Animal(name='Marmot')
animal.save()

# Store file
with open('photo.jpg', 'rb') as f:
    animal.photo.put(f, content_type='image/jpeg')
animal.save()  # Important: save after put

# Read file
content = animal.photo.read()
animal.photo.seek(0)  # Rewind for re-reading

# Stream write
animal.photo.new_file()
animal.photo.write(b'chunk1')
animal.photo.write(b'chunk2')
animal.photo.close()
animal.save()

# Replace file
with open('new_photo.png', 'rb') as f:
    animal.photo.replace(f, content_type='image/png')
animal.save()

# Delete file (must save afterward)
animal.photo.delete()
animal.save()
```

### GridFSProxy Properties

| Property | Description |
|----------|-------------|
| `grid_id` | ObjectId of the GridFS file |
| `content_type` | MIME type |
| `filename` | Original filename |
| `length` | File size in bytes |
| `upload_date` | When uploaded |

### ImageField Usage

```python
class Photo(Document):
    image = ImageField(
        size={'width': 800, 'height': 600, 'force': True},
        thumbnail_size={'width': 150, 'height': 150, 'force': True},
        collection_name='images',
    )

# Access thumbnail
thumb = photo.image.thumbnail  # Returns GridFS file for thumbnail
width = photo.image.width
height = photo.image.height
fmt = photo.image.format
```

> **Note:** Deleting a document does NOT automatically delete its GridFS files. Delete files explicitly before deleting the document.

---

## 13. Document Inheritance

### Enabling Inheritance

```python
class Page(Document):
    title = StringField()
    meta = {'allow_inheritance': True}

class DatedPage(Page):
    date = DateTimeField()

class BlogPost(DatedPage):
    body = StringField()
```

**Behavior:**
- All stored in the same collection (`page` by default).
- MongoDB stores `_cls` field: `"Page"`, `"Page.DatedPage"`, `"Page.DatedPage.BlogPost"`.
- `Page.objects()` returns Page, DatedPage, and BlogPost instances.
- `DatedPage.objects()` returns DatedPage and BlogPost instances.
- `BlogPost.objects()` returns only BlogPost instances.

### Querying Exact Class (No Subclasses)

```python
# Context manager
with no_sub_classes(Page):
    pages = Page.objects()  # Only Page, not DatedPage or BlogPost

# QuerySet method
Page.objects.no_sub_classes()
```

### Abstract vs. Inheritance

| Feature | `abstract=True` | `allow_inheritance=True` |
|---------|-----------------|--------------------------|
| Own collection | No | Yes (shared with subclasses) |
| `_cls` field | No | Yes |
| Polymorphic queries | No | Yes |
| Collection per subclass | Yes (each gets own) | No (shared) |
| Use case | Code reuse | Polymorphism |

---

## 14. References & Delete Rules

### Delete Rule Constants

```python
from mongoengine import DO_NOTHING, NULLIFY, CASCADE, DENY, PULL
```

| Rule | Value | Behavior |
|------|-------|----------|
| `DO_NOTHING` | 0 | No action (may leave dangling references) |
| `NULLIFY` | 1 | Set reference to None |
| `CASCADE` | 2 | Delete referencing documents |
| `DENY` | 3 | Prevent deletion if references exist |
| `PULL` | 4 | Remove from ListField containing reference |

### Example

```python
class Author(Document):
    name = StringField()

class Post(Document):
    title = StringField()
    author = ReferenceField(Author, reverse_delete_rule=NULLIFY)
    # When Author is deleted, Post.author becomes None

class Comment(Document):
    post = ReferenceField(Post, reverse_delete_rule=CASCADE)
    # When Post is deleted, Comment is also deleted

class Tag(Document):
    posts = ListField(ReferenceField(Post, reverse_delete_rule=PULL))
    # When Post is deleted, it's removed from Tag.posts list
```

> **Important:** Delete rules are enforced in MongoEngine application memory only, not at the database level. The module declaring the relationship must be imported before delete operations execute.

---

## 15. Text Search

### Defining Text Indexes

```python
class Article(Document):
    title = StringField()
    content = StringField()
    meta = {
        'indexes': [{
            'fields': ['$title', '$content'],
            'weights': {'title': 10, 'content': 2},
            'default_language': 'english',
        }]
    }
```

### Querying

```python
# Basic text search
results = Article.objects.search_text('mongodb python')

# Order by relevance score
results = Article.objects.search_text('mongodb').order_by('$text_score')

# Get text score
doc = results.first()
score = doc.get_text_score()

# Combine with other filters
results = Article.objects.search_text('mongodb').filter(status='published')
```

---

## 16. Geospatial Queries

### Defining Geo Fields

```python
class Venue(Document):
    name = StringField()
    location = PointField()  # Auto-creates 2dsphere index
```

### Querying Near a Point

```python
# Near (sorted by distance)
Venue.objects(location__near=[lng, lat])
Venue.objects(location__near={'type': 'Point', 'coordinates': [lng, lat]})

# With distance constraints
Venue.objects(
    location__near=[lng, lat],
    location__max_distance=5000,   # meters
    location__min_distance=100,
)

# Near on sphere
Venue.objects(location__near_sphere=[lng, lat])
```

### Querying Within Geometry

```python
# Within polygon
polygon = {'type': 'Polygon', 'coordinates': [[[...]]]}
Venue.objects(location__geo_within=polygon)

# Within box
Venue.objects(location__geo_within_box=[(sw_lng, sw_lat), (ne_lng, ne_lat)])

# Within circle/sphere
Venue.objects(location__geo_within_center=[(lng, lat), radius])
Venue.objects(location__geo_within_sphere=[(lng, lat), radius])

# Intersects
line = {'type': 'LineString', 'coordinates': [[...], [...]]}
Venue.objects(location__geo_intersects=line)
```

---

## 17. Transactions

Requires MongoDB 4.0+ with replica set or sharded cluster.

```python
from mongoengine.context_managers import run_in_transaction

# Basic transaction
with run_in_transaction():
    account_a = Account.objects.get(id=a_id)
    account_b = Account.objects.get(id=b_id)
    account_a.update(dec__balance=100)
    account_b.update(inc__balance=100)
    # Auto-commits on exit; auto-aborts on exception

# With custom options
with run_in_transaction(
    db_alias='default',
    max_commit_time_ms=5000,
):
    # Operations here
    pass
```

---

## 18. Errors & Exceptions

All importable from `mongoengine` or `mongoengine.errors`:

| Exception | Description |
|-----------|-------------|
| `ValidationError` | Field validation failed. Has `.errors` dict and `.message` |
| `DoesNotExist` | `get()` found no matching document. Subclassed per Document |
| `MultipleObjectsReturned` | `get()` found more than one match. Subclassed per Document |
| `OperationError` | Database operation failed |
| `NotUniqueError` | Unique constraint violated (subclass of OperationError) |
| `BulkWriteError` | Bulk write operation failed |
| `SaveConditionError` | `save_condition` was not met |
| `InvalidQueryError` | Invalid query syntax |
| `InvalidDocumentError` | Invalid document configuration |
| `LookUpError` | Field lookup failed |
| `NotRegistered` | Document class not registered |
| `FieldDoesNotExist` | Undefined field accessed on strict document |
| `ConnectionFailure` | Database connection failed |
| `InvalidCollectionError` | Invalid collection configuration |
| `DeprecatedError` | Deprecated feature used |

### ValidationError Usage

```python
try:
    doc.validate()
except ValidationError as e:
    print(e.errors)          # {'field_name': ValidationError, ...}
    print(e.to_dict())       # Nested dict of error messages
    print(e.message)         # Summary message

# Custom document-level validation
class User(Document):
    start_date = DateTimeField()
    end_date = DateTimeField()

    def clean(self):
        if self.end_date and self.start_date and self.end_date < self.start_date:
            raise ValidationError('end_date must be after start_date')
```

### Per-Document Exceptions

```python
try:
    user = User.objects.get(id='nonexistent')
except User.DoesNotExist:
    print('User not found')
except User.MultipleObjectsReturned:
    print('Multiple users matched')
```

---

## Quick Reference: Common Patterns

### CRUD Operations

```python
# Create
user = User(name='Alice', email='alice@example.com').save()

# Read
user = User.objects.get(email='alice@example.com')
users = User.objects(age__gte=18).order_by('-created_at')[:10]

# Update
user.update(set__name='Alice Smith', inc__login_count=1)
# or
user.name = 'Alice Smith'
user.save()

# Delete
user.delete()
```

### Pagination

```python
page = 2
per_page = 20
users = User.objects.skip((page - 1) * per_page).limit(per_page)
total = User.objects.count()
```

### Conditional Save (Optimistic Locking)

```python
user.save(save_condition={'version': user.version})
# Raises SaveConditionError if version changed since read
```

### Atomic Counter

```python
User.objects(id=user_id).update_one(inc__view_count=1)
```

### Embedded Document CRUD

```python
class Comment(EmbeddedDocument):
    text = StringField()
    author = StringField()

class Post(Document):
    comments = EmbeddedDocumentListField(Comment)

# Add
post.comments.create(text='Great!', author='Alice')
post.save()

# Query embedded list
post.comments.filter(author='Alice')
post.comments.get(author='Alice')
post.comments.first()
post.comments.count()

# Update all in list
post.comments.update(approved=True)

# Delete all
post.comments.delete()
post.save()
```

### Multi-Database

```python
connect(db='primary', alias='default')
connect(db='analytics', alias='analytics')

class User(Document):
    meta = {'db_alias': 'default'}

class Event(Document):
    meta = {'db_alias': 'analytics'}

# Or switch at runtime
with switch_db(User, 'analytics'):
    User.objects.count()
```

### Raw PyMongo Access

```python
collection = User._get_collection()
result = collection.find_one({'_id': some_id})
collection.create_index([('field', 1)])
```
