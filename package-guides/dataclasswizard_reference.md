# Dataclass Wizard - Comprehensive Reference

> **Version:** 0.39.1 | **License:** Apache 2.0 | **Python:** 3.9 - 3.14
> **Install:** `pip install dataclass-wizard`

Lightning-fast JSON/TOML/YAML/ENV serialization for Python dataclasses. Converts between dataclass instances and JSON-compatible dicts with automatic key casing, type coercion, nested dataclass support, and more.

---

## Table of Contents

- [Quick Start](#quick-start)
- [Core Classes](#core-classes)
  - [JSONWizard / JSONSerializable](#jsonwizard--jsonserializable)
  - [DataclassWizard](#dataclasswizard)
  - [JSONPyWizard](#jsonpywizard)
- [Mixin Classes](#mixin-classes)
  - [JSONListWizard](#jsonlistwizard)
  - [JSONFileWizard](#jsonfilewizard)
  - [TOMLWizard](#tomlwizard)
  - [YAMLWizard](#yamlwizard)
  - [EnvWizard](#envwizard)
- [Standalone Functions](#standalone-functions)
- [Meta Configuration](#meta-configuration)
  - [v0 Meta Attributes](#v0-meta-attributes)
  - [v1 Meta Attributes](#v1-meta-attributes)
  - [EnvWizard Meta Attributes](#envwizard-meta-attributes)
  - [LoadMeta / DumpMeta / EnvMeta](#loadmeta--dumpmeta--envmeta)
- [Field Configuration](#field-configuration)
  - [json_field](#json_field)
  - [json_key](#json_key)
  - [KeyPath](#keypath)
  - [path_field](#path_field)
  - [Alias (v1)](#alias-v1)
  - [AliasPath (v1)](#aliaspath-v1)
  - [env_field](#env_field)
  - [skip_if_field](#skip_if_field)
- [Conditional Skipping](#conditional-skipping)
- [Date/Time Patterns](#datetime-patterns)
  - [v0 Patterns](#v0-patterns)
  - [v1 Patterns](#v1-patterns)
- [Custom Type Hooks](#custom-type-hooks)
- [Serializer Hooks](#serializer-hooks)
- [Union Types & Tagging](#union-types--tagging)
- [Recursive / Cyclic Dataclasses](#recursive--cyclic-dataclasses)
- [Unknown JSON Keys](#unknown-json-keys)
- [Key Transforms & Casing](#key-transforms--casing)
- [Container Class](#container-class)
- [property_wizard Metaclass](#property_wizard-metaclass)
- [Enums Reference](#enums-reference)
- [Errors & Exceptions](#errors--exceptions)
- [Supported Types](#supported-types)
- [v0 vs v1 Engine](#v0-vs-v1-engine)
- [CLI Tool (wiz)](#cli-tool-wiz)
- [Optional Dependencies](#optional-dependencies)

---

## Quick Start

```python
from dataclasses import dataclass
from dataclass_wizard import JSONWizard

@dataclass
class User(JSONWizard):
    first_name: str
    last_name: str
    age: int

# Deserialize from JSON string
user = User.from_json('{"firstName": "John", "lastName": "Doe", "age": 30}')

# Deserialize from dict
user = User.from_dict({"firstName": "John", "lastName": "Doe", "age": 30})

# Serialize to dict (keys become camelCase by default)
d = user.to_dict()   # {"firstName": "John", "lastName": "Doe", "age": 30}

# Serialize to JSON string
s = user.to_json()    # '{"firstName": "John", "lastName": "Doe", "age": 30}'

# Deserialize a list of dicts
users = User.from_list([{"firstName": "A", "lastName": "B", "age": 1}])

# Serialize a list of instances to JSON
json_str = User.list_to_json([user])

# __str__ returns prettified JSON
print(user)
```

### Without Inheritance (Functional API)

```python
from dataclasses import dataclass
from dataclass_wizard import fromdict, fromlist, asdict, LoadMeta, DumpMeta

@dataclass
class User:
    first_name: str
    age: int

# Optional: configure key transform
LoadMeta(key_transform='CAMEL').bind_to(User)

user = fromdict(User, {"firstName": "John", "age": 30})
d = asdict(user)
users = fromlist(User, [{"firstName": "John", "age": 30}])
```

---

## Core Classes

### JSONWizard / JSONSerializable

`JSONWizard` is an alias for `JSONSerializable`. This is the most commonly used base class.

```python
from dataclass_wizard import JSONWizard  # or JSONSerializable

@dataclass
class MyClass(JSONWizard):
    ...
```

**Subclass parameters:**

```python
@dataclass
class MyClass(JSONWizard, str=True, debug=False, case=None, dump_case=None, load_case=None):
    ...
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `str` | `bool` | `True` | Add `__str__` method that returns prettified JSON |
| `debug` | `bool \| int` | `False` | Enable debug logging. `True` = DEBUG level, or pass a `logging` level int |
| `case` | `str \| None` | `None` | Key casing for both load/dump (v1). Sets `v1=True` automatically |
| `load_case` | `str \| None` | `None` | Key casing for load only (v1). Sets `v1=True` automatically |
| `dump_case` | `str \| None` | `None` | Key casing for dump only (v1). Sets `v1=True` automatically |

#### Instance & Class Methods

**`from_json(string, *, decoder=json.loads, **decoder_kwargs)`** (classmethod)
Deserialize a JSON string to a dataclass instance (if JSON is an object) or list of instances (if JSON is an array).

```python
user = User.from_json('{"firstName": "John", "age": 30}')
users = User.from_json('[{"firstName": "A", "age": 1}]')
```

**`from_dict(d)`** (classmethod)
Deserialize a dict to a dataclass instance.

```python
user = User.from_dict({"firstName": "John", "age": 30})
```

**`from_list(list_of_dict)`** (classmethod)
Deserialize a list of dicts to a list of dataclass instances.

```python
users = User.from_list([{"firstName": "A", "age": 1}])
```

**`to_dict()`** (instance method)
Serialize a dataclass instance to a dict.

```python
d = user.to_dict()
```

Note: `to_dict` also accepts keyword arguments when called via `asdict()`:
```python
from dataclass_wizard import asdict
d = asdict(user, exclude={'password'})
```

**`to_json(*, encoder=json.dumps, **encoder_kwargs)`** (instance method)
Serialize a dataclass instance to a JSON string.

```python
s = user.to_json()
s = user.to_json(indent=2)
```

**`list_to_json(instances, encoder=json.dumps, **encoder_kwargs)`** (classmethod)
Serialize a list of instances to a JSON string.

```python
s = User.list_to_json([user1, user2], indent=2)
```

**`register_type(tp, *, load=None, dump=None, mode=None)`** (classmethod)
Register custom load/dump hooks for a type. See [Custom Type Hooks](#custom-type-hooks).

### DataclassWizard

Lower-level base class. Same API as `JSONWizard` but with `str=False` and `_v1_default=True` by default. Also auto-applies `@dataclass` decorator to subclasses.

```python
from dataclass_wizard import DataclassWizard

class MyClass(DataclassWizard):  # No need for @dataclass
    name: str
    age: int
```

### JSONPyWizard

Variant of `JSONWizard` that keeps dict keys as-is (no camelCase transform on dump). Uses `pprint.pformat` for `__str__` instead of JSON.

```python
from dataclass_wizard import JSONPyWizard

@dataclass
class MyClass(JSONPyWizard):
    my_field: str

obj = MyClass(my_field="hello")
obj.to_dict()  # {"my_field": "hello"}  -- keys preserved as snake_case
```

---

## Mixin Classes

### JSONListWizard

Extends `JSONWizard`. `from_json()` and `from_list()` return a `Container` (list subclass with helper methods) instead of a plain list.

```python
from dataclass_wizard import JSONListWizard

@dataclass
class Item(JSONListWizard):
    name: str

items = Item.from_json('[{"name": "a"}, {"name": "b"}]')
# items is a Container[Item]
print(items.prettify())       # prettified JSON
json_str = items.to_json()    # JSON string
items.to_json_file("out.json")
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `from_json` | `(cls, string, *, decoder=json.loads, **decoder_kwargs)` | Returns instance or `Container[cls]` |
| `from_list` | `(cls, o)` | Returns `Container[cls]` |

### JSONFileWizard

Adds JSON file I/O. Can be combined with `JSONWizard`.

```python
from dataclass_wizard import JSONWizard, JSONFileWizard

@dataclass
class Config(JSONWizard, JSONFileWizard):
    name: str

config = Config.from_json_file("config.json")
config.to_json_file("output.json")
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `from_json_file` | `(cls, file, *, decoder=json.load, **decoder_kwargs)` | Read JSON file to instance or list |
| `to_json_file` | `(self, file, mode='w', encoder=json.dump, **encoder_kwargs)` | Write instance to JSON file |

### TOMLWizard

TOML serialization/deserialization. **Default dump key transform: NONE** (snake_case preserved). Requires `pip install dataclass-wizard[toml]`.

```python
from dataclasses import dataclass
from dataclass_wizard import TOMLWizard

@dataclass
class Config(TOMLWizard):
    server_name: str
    port: int

# Or with custom key transform:
@dataclass
class Config(TOMLWizard, key_transform='CAMEL'):
    server_name: str
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `from_toml` | `(cls, string_or_stream, *, decoder=None, header='items', parse_float=float)` | Parse TOML string/stream |
| `from_toml_file` | `(cls, file, *, decoder=None, header='items', parse_float=float)` | Parse TOML file |
| `to_toml` | `(self, /, *encoder_args, encoder=None, multiline_strings=False, indent=4)` | Serialize to TOML string |
| `to_toml_file` | `(self, file, mode='wb', encoder=None, multiline_strings=False, indent=4)` | Write to TOML file |
| `list_to_toml` | `(cls, instances, header='items', encoder=None, **encoder_kwargs)` | Serialize list to TOML |

The `header` parameter specifies which key contains the list when deserializing arrays from TOML.

### YAMLWizard

YAML serialization/deserialization. **Default dump key transform: LISP** (kebab-case). Requires `pip install dataclass-wizard[yaml]`.

```python
from dataclasses import dataclass
from dataclass_wizard import YAMLWizard

@dataclass
class Config(YAMLWizard):
    server_name: str

# Custom key transform:
@dataclass
class Config(YAMLWizard, key_transform='CAMEL'):
    server_name: str
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `from_yaml` | `(cls, string_or_stream, *, decoder=None, **decoder_kwargs)` | Parse YAML string/stream |
| `from_yaml_file` | `(cls, file, *, decoder=None, **decoder_kwargs)` | Parse YAML file |
| `to_yaml` | `(self, *, encoder=None, **encoder_kwargs)` | Serialize to YAML string |
| `to_yaml_file` | `(self, file, mode='w', encoder=None, **encoder_kwargs)` | Write to YAML file |
| `list_to_yaml` | `(cls, instances, encoder=None, **encoder_kwargs)` | Serialize list to YAML |

### EnvWizard

Load configuration from environment variables, `.env` files, and secrets directories.

```python
from dataclass_wizard import EnvWizard

class Config(EnvWizard):
    class _(EnvWizard.Meta):
        env_file = True        # Load from .env file
        env_prefix = 'APP_'   # Prefix for env var lookup

    database_url: str
    debug: bool = False
    port: int = 8000
```

**Subclass parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `reload_env` | `bool` | `False` | Reload env vars on each instantiation |
| `debug` | `bool` | `False` | Enable debug logging |
| `key_transform` | `LetterCase` | `NONE` | Key transform for dump |

**Constructor:**
```python
config = Config(
    _env_file='.env.local',    # Override env file path
    _reload=True,              # Force reload
    _env_prefix='MY_',         # Override prefix
    _secrets_dir='/run/secrets',
    # Can also pass field values directly:
    port=9000,
)
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `dict()` | `(self) -> dict` | Return field values as dict |
| `to_dict()` | `(self) -> dict` | Alias for `asdict(self)` |
| `to_json()` | `(self, *, encoder=json.dumps, **encoder_kwargs)` | Serialize to JSON string |

**EnvWizard v1 features (opt-in with `v1 = True` in Meta):**
- `v1_env_precedence`: Control lookup order (`EnvPrecedence.SECRETS_ENV_DOTENV` default)
- `v1_field_to_env_load`: Map field names to specific env var names
- `v1_field_to_alias_dump`: Map field names to dump aliases
- `v1_load_case`: `EnvKeyStrategy.ENV` (default), `FIELD_FIRST`, or `STRICT`
- Nested dataclass support

---

## Standalone Functions

These work without inheriting from `JSONWizard`:

### `fromdict(cls, d)`

```python
from dataclass_wizard import fromdict

user = fromdict(User, {"firstName": "John", "age": 30})
```

### `fromlist(cls, list_of_dict)`

```python
from dataclass_wizard import fromlist

users = fromlist(User, [{"firstName": "A", "age": 1}])
```

### `asdict(o, *, cls=None, dict_factory=dict, exclude=None, **kwargs)`

```python
from dataclass_wizard import asdict

d = asdict(user)
d = asdict(user, exclude={'password', 'secret'})
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `o` | dataclass instance | required | The instance to serialize |
| `cls` | `type \| None` | `None` | Class to use for dumper lookup |
| `dict_factory` | `type` | `dict` | Factory for creating dicts |
| `exclude` | `Collection[str] \| None` | `None` | Field names to exclude |

### `register_type(cls, tp, *, load=None, dump=None, mode=None)`

Register custom type hooks globally or per-class. See [Custom Type Hooks](#custom-type-hooks).

---

## Meta Configuration

Configure serialization behavior via an inner `Meta` class or via `LoadMeta()`/`DumpMeta()`.

```python
@dataclass
class MyClass(JSONWizard):
    class Meta(JSONWizard.Meta):
        key_transform_with_dump = 'SNAKE'
        skip_defaults = True

    name: str
    age: int = 0
```

### v0 Meta Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `debug_enabled` | `bool \| int \| str` | `False` | Enable debug logging. `True` = DEBUG, or pass logging level |
| `recursive` | `bool` | `True` | Apply Meta config to nested dataclasses |
| `recursive_classes` | `bool` | `False` | Support self-referential/cyclic dataclasses |
| `raise_on_unknown_json_key` | `bool` | `False` | Raise `UnknownKeysError` on unknown keys |
| `json_key_to_field` | `dict[str, str]` | `None` | Map JSON keys to field names: `{'jsonKey': 'field_name'}` |
| `marshal_date_time_as` | `DateTimeTo \| str` | `None` | DateTime serialization format: `'ISO_FORMAT'` or `'TIMESTAMP'` |
| `key_transform_with_load` | `LetterCase \| str` | `'PASCAL'`* | Key transform for deserialization |
| `key_transform_with_dump` | `LetterCase \| str` | `'CAMEL'`* | Key transform for serialization |
| `tag` | `str` | `None` | Class tag for Union type discrimination |
| `tag_key` | `str` | `'__tag__'` | JSON key name for the tag field |
| `auto_assign_tags` | `bool` | `False` | Auto-assign class name as tag |
| `skip_defaults` | `bool` | `False` | Omit fields with default values during serialization |
| `skip_if` | `Condition` | `None` | Global skip condition for all fields |
| `skip_defaults_if` | `Condition` | `None` | Skip condition for fields with defaults only |

*\* Default key transforms: load expects PascalCase/camelCase JSON keys mapping to snake_case fields; dump outputs camelCase.*

### v1 Meta Attributes

Enable v1 with `v1 = True` in Meta (or use `case`/`load_case`/`dump_case` subclass params which auto-enable v1).

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `v1` | `bool` | `False` | Enable v1 codegen engine |
| `v1_debug` | `bool \| int \| str` | `False` | V1 debug output |
| `v1_case` | `KeyCase \| str \| None` | `None` | Key casing for both load and dump |
| `v1_load_case` | `KeyCase \| str \| None` | `None` | Key casing for load only |
| `v1_dump_case` | `KeyCase \| str \| None` | `None` | Key casing for dump only |
| `v1_field_to_alias` | `Mapping[str, str \| Sequence[str]]` | `None` | Field-to-alias mapping (both directions) |
| `v1_field_to_alias_load` | `Mapping[str, str \| Sequence[str]]` | `None` | Field-to-alias for load only |
| `v1_field_to_alias_dump` | `Mapping[str, str \| Sequence[str]]` | `None` | Field-to-alias for dump only |
| `v1_on_unknown_key` | `KeyAction` | `None` | Handle unknown keys: `IGNORE`, `WARN`, `RAISE` |
| `v1_type_to_load_hook` | `dict[type, Callable]` | `None` | Custom load hooks by type |
| `v1_type_to_dump_hook` | `dict[type, Callable]` | `None` | Custom dump hooks by type |
| `v1_pre_decoder` | `Callable` | `None` | Pre-decoder transformation |
| `v1_unsafe_parse_dataclass_in_union` | `bool` | `False` | Allow dataclass parsing in unions without tags |
| `v1_dump_date_time_as` | `V1DateTimeTo \| str` | `None` | DateTime dump format |
| `v1_assume_naive_datetime_tz` | `tzinfo \| None` | `None` | Timezone for naive datetimes |
| `v1_namedtuple_as_dict` | `bool` | `None` | Dump NamedTuples as dicts |
| `v1_coerce_none_to_empty_str` | `bool` | `None` | Convert None to "" for str fields |
| `v1_leaf_handling` | `'exact' \| 'issubclass'` | `None` | Type matching strategy for leaf types |

### EnvWizard Meta Attributes

| Attribute | Type | Default | Description |
|-----------|------|---------|-------------|
| `debug_enabled` | `bool` | `False` | Enable debug logging |
| `recursive` | `bool` | `True` | Apply config to nested dataclasses |
| `env_file` | `str \| Path \| bool \| list` | `None` | Path to .env file(s), or `True` for auto |
| `env_prefix` | `str` | `None` | Prefix for env var lookup (e.g., `'APP_'`) |
| `secrets_dir` | `str \| Path \| list` | `None` | Directory for Docker/file-based secrets |
| `field_to_env_var` | `dict[str, str]` | `None` | Map field names to specific env var names |
| `key_lookup_with_load` | `LetterCasePriority \| str` | `'SCREAMING_SNAKE'` | Env var name lookup strategy |
| `key_transform_with_dump` | `LetterCase \| str` | `'SNAKE'` | Key transform for dump |
| `skip_defaults` | `bool` | `False` | Skip default values on dump |
| `skip_if` | `Condition` | `None` | Skip condition |
| `skip_defaults_if` | `Condition` | `None` | Skip defaults condition |
| `v1` | `bool` | `False` | Enable v1 env features |
| `v1_env_precedence` | `EnvPrecedence` | `None` | Lookup order for env values |
| `v1_load_case` | `EnvKeyStrategy \| str` | `None` | Env key resolution strategy |
| `v1_field_to_env_load` | `Mapping[str, str \| Sequence[str]]` | `None` | Field-to-env-var mapping for loading |
| `v1_field_to_alias_dump` | `Mapping[str, str \| Sequence[str]]` | `None` | Field-to-alias mapping for dump |

### LoadMeta / DumpMeta / EnvMeta

Functional API for applying Meta configuration without class inheritance.

```python
from dataclass_wizard import LoadMeta, DumpMeta, EnvMeta

# Apply load config
LoadMeta(key_transform='CAMEL', v1=True).bind_to(MyClass)

# Apply dump config
DumpMeta(key_transform='SNAKE', skip_defaults=True).bind_to(MyClass)

# Apply env config
EnvMeta(env_file=True, env_prefix='APP_').bind_to(MyEnvClass)
```

**`LoadMeta(**kwargs)`** - Creates a Meta class with the given attributes and returns it. Call `.bind_to(cls)` to attach.

**`DumpMeta(**kwargs)`** - Same as LoadMeta but for dump configuration.

**`EnvMeta(**kwargs)`** - Same but for EnvWizard configuration.

**`.bind_to(cls, create=True, is_default=True)`:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `cls` | `type` | required | Dataclass to bind to |
| `create` | `bool` | `True` | Create loader/dumper if not exists |
| `is_default` | `bool` | `True` | Set as default loader/dumper for class |

---

## Field Configuration

### json_field

Creates a `dataclasses.field()` with JSON key mapping metadata.

```python
from dataclass_wizard import json_field

def json_field(
    keys: str | tuple[str, ...],
    *,
    all: bool = False,
    dump: bool = True,
    default = MISSING,
    default_factory = MISSING,
    init: bool = True,
    repr: bool = True,
    hash: bool | None = None,
    compare: bool = True,
    metadata: dict | None = None
) -> Field
```

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `keys` | `str \| tuple` | required | JSON key name(s) to map to this field |
| `all` | `bool` | `False` | If True, all keys are valid for deserialization |
| `dump` | `bool` | `True` | If False, exclude this field from serialization |
| `default` | any | `MISSING` | Default value |
| `default_factory` | callable | `MISSING` | Factory for default value |

```python
@dataclass
class User(JSONWizard):
    name: str = json_field('UserName')
    age: int = json_field(('user_age', 'userAge'), all=True, default=0)
    password: str = json_field('pass', dump=False, default='')
```

### json_key

Lighter-weight alternative to `json_field` for use with `Annotated`.

```python
from dataclass_wizard import json_key

def json_key(*keys: str, all: bool = False, dump: bool = True)
```

```python
from typing import Annotated

@dataclass
class User(JSONWizard):
    name: Annotated[str, json_key('UserName')]
    age: Annotated[int, json_key('user_age', 'userAge', all=True)]
```

### KeyPath

Maps a field to a nested JSON path (dot notation or bracket syntax).

```python
from dataclass_wizard import KeyPath

def KeyPath(keys: str | tuple[str, ...], all: bool = True, dump: bool = True)
```

```python
from typing import Annotated

@dataclass
class User(JSONWizard):
    # Maps to data["user"]["name"] in JSON
    name: Annotated[str, KeyPath('data.user.name')]
    # Maps to items[0]["id"]
    first_id: Annotated[int, KeyPath('items[0].id')]
```

**Path syntax:**
- Dots (`.`) separate keys: `a.b.c` → `["a"]["b"]["c"]`
- Brackets for indices: `items[0]` → `["items"][0]`
- Quoted keys for special chars: `data["my key"]` → `["data"]["my key"]`
- Integers are parsed as indices, quoted integers as string keys

### path_field

Like `json_field` but with `all=True` by default and path interpretation enabled.

```python
def path_field(
    keys: str | tuple[str, ...],
    *,
    all: bool = True,
    dump: bool = True,
    default = MISSING,
    default_factory = MISSING,
    init: bool = True,
    repr: bool = True,
    hash: bool | None = None,
    compare: bool = True,
    metadata: dict | None = None
) -> Field
```

```python
@dataclass
class Config(JSONWizard):
    db_host: str = path_field('database.host', default='localhost')
```

### Alias (v1)

V1-only field alias for load and/or dump. Import from `dataclass_wizard.v1`.

```python
from dataclass_wizard.v1 import Alias

def Alias(
    *all: str,
    load: str | Sequence[str] | None = None,
    dump: str | None = None,
    env: str | Sequence[str] | None = None,
    skip: bool = False,
    default = MISSING,
    default_factory = MISSING,
    init: bool = True,
    repr: bool = True,
    hash: bool | None = None,
    compare: bool = True,
    metadata: dict | None = None,
    kw_only: bool = False
) -> Field
```

```python
from dataclass_wizard.v1 import Alias

@dataclass
class User(JSONWizard):
    class Meta(JSONWizard.Meta):
        v1 = True

    # Same alias for both load and dump
    name: str = Alias('userName')

    # Separate load/dump aliases
    email: str = Alias(load='email_address', dump='email')

    # Skip on dump
    internal_id: str = Alias('id', skip=True, default='')

    # With Annotated
    age: Annotated[int, Alias('userAge')] = 0
```

### AliasPath (v1)

V1-only nested path alias. Import from `dataclass_wizard.v1`.

```python
from dataclass_wizard.v1 import AliasPath

def AliasPath(
    *all: PathType | str,
    load: PathType | str | None = None,
    dump: PathType | str | None = None,
    env: PathType | str | bool | None = None,
    skip: bool = False,
    default = MISSING,
    default_factory = MISSING,
    init: bool = True,
    repr: bool = True,
    hash: bool | None = None,
    compare: bool = True,
    metadata: dict | None = None,
    kw_only: bool = False
) -> Field
```

```python
from dataclass_wizard.v1 import AliasPath

@dataclass
class Config(JSONWizard):
    class Meta(JSONWizard.Meta):
        v1 = True

    # Map to nested path with multiple fallbacks
    name: str = AliasPath('a.b.c.1', 'x.y.z', default="fallback")
```

### env_field

Alias for `json_field`. Used with `EnvWizard` for clarity.

```python
from dataclass_wizard import env_field

class Config(EnvWizard):
    db_url: str = env_field('DATABASE_URL')
```

### skip_if_field

Creates a field with a built-in skip condition for serialization.

```python
from dataclass_wizard import skip_if_field, IS

@dataclass
class User(JSONWizard):
    name: str
    nickname: str = skip_if_field(IS(None), default=None)
```

---

## Conditional Skipping

Control which fields are included during serialization.

### Global Skip (Meta)

```python
from dataclass_wizard import JSONWizard, IS

@dataclass
class MyClass(JSONWizard):
    class Meta(JSONWizard.Meta):
        skip_defaults = True            # Skip all fields at their default value
        skip_if = IS(None)             # Skip any field whose value is None
        skip_defaults_if = IS(None)    # Skip default fields only if None
```

### Per-Field Skip (Annotated)

```python
from typing import Annotated
from dataclass_wizard import JSONWizard, SkipIf, SkipIfNone, IS, EQ, LT

@dataclass
class MyClass(JSONWizard):
    # Skip if value is None
    optional_name: Annotated[str | None, SkipIfNone] = None

    # Skip if value equals empty string
    tag: Annotated[str, SkipIf(EQ(''))] = ''

    # Skip if value less than 0
    score: Annotated[int, SkipIf(LT(0))] = -1
```

### Condition Operators

| Function | Description | Example |
|----------|-------------|---------|
| `EQ(value)` | Equal to | `EQ(0)` |
| `NE(value)` | Not equal to | `NE('')` |
| `LT(value)` | Less than | `LT(0)` |
| `LE(value)` | Less than or equal | `LE(0)` |
| `GT(value)` | Greater than | `GT(100)` |
| `GE(value)` | Greater than or equal | `GE(1)` |
| `IS(value)` | Identity check (`is`) | `IS(None)` |
| `IS_NOT(value)` | Not identity (`is not`) | `IS_NOT(None)` |
| `IS_TRUTHY()` | Truthy check | `IS_TRUTHY()` |
| `IS_FALSY()` | Falsy check | `IS_FALSY()` |

`SkipIfNone` is a convenience alias for `SkipIf(IS(None))`.

---

## Date/Time Patterns

### v0 Patterns

Use `Annotated` with `DatePattern`, `TimePattern`, or `DateTimePattern`:

```python
from typing import Annotated
from dataclass_wizard import JSONWizard, DatePattern, TimePattern, DateTimePattern

@dataclass
class Event(JSONWizard):
    start_date: Annotated[date, DatePattern('%Y-%m-%d')]
    start_time: Annotated[time, TimePattern('%H:%M')]
    created_at: Annotated[datetime, DateTimePattern('%Y/%m/%d %H:%M:%S')]
```

For containers of date/time values, use `Pattern()`:

```python
from dataclass_wizard import Pattern

@dataclass
class Events(JSONWizard):
    dates: Annotated[list[date], Pattern('%Y-%m-%d')]
```

### v1 Patterns

V1 adds timezone-aware and UTC patterns. Import from `dataclass_wizard.v1`:

```python
from dataclass_wizard.v1 import (
    DatePattern, TimePattern, DateTimePattern,
    AwareDateTimePattern, AwareTimePattern,
    UTCDateTimePattern, UTCTimePattern,
)
```

| Pattern Class | Description | Subscript Syntax |
|---------------|-------------|-----------------|
| `DatePattern` | Naive date | `DatePattern['%Y-%m-%d']` |
| `TimePattern` | Naive time | `TimePattern['%H:%M:%S']` |
| `DateTimePattern` | Naive datetime | `DateTimePattern['%Y-%m-%d %H:%M']` |
| `AwareDateTimePattern` | Timezone-aware datetime | `AwareDateTimePattern['US/Eastern', '%Y-%m-%d %H:%M']` |
| `AwareTimePattern` | Timezone-aware time | `AwareTimePattern['UTC', '%H:%M']` |
| `UTCDateTimePattern` | UTC datetime | `UTCDateTimePattern['%Y-%m-%d %H:%M']` |
| `UTCTimePattern` | UTC time | `UTCTimePattern['%H:%M']` |

Multiple patterns (try in order):
```python
my_field: DatePattern['%Y-%m-%d', '%m/%d/%Y']
```

---

## Custom Type Hooks

Register custom serialization/deserialization for types not natively supported.

### Quick Registration

```python
from ipaddress import IPv4Address

@dataclass
class Server(JSONWizard):
    host: IPv4Address
    port: int

# Default: load=IPv4Address (constructor), dump=str
Server.register_type(IPv4Address)

# Custom functions:
Server.register_type(
    IPv4Address,
    load=lambda s: IPv4Address(s),
    dump=lambda ip: str(ip)
)
```

### Global Registration (without inheritance)

```python
from dataclass_wizard import register_type, LoadMeta

# Register globally
register_type(None, IPv4Address, load=IPv4Address, dump=str)
# The first arg `None` means global (all classes)

# Or register for a specific class
register_type(MyClass, IPv4Address, load=IPv4Address, dump=str)
```

### V1 Codegen Hooks

For the v1 engine, hooks can return code strings for compilation:

```python
# Via Meta:
class Meta(JSONWizard.Meta):
    v1 = True
    v1_type_to_load_hook = {
        IPv4Address: lambda tp, extras: f"IPv4Address({{val}})"
    }
    v1_type_to_dump_hook = {
        IPv4Address: lambda tp, extras: f"str({{val}})"
    }
```

### Enum by Name (Common Pattern)

```python
from enum import Enum
from dataclass_wizard import JSONWizard

class Color(Enum):
    RED = 1
    BLUE = 2

@dataclass
class Item(JSONWizard):
    color: Color

# Load/dump by name instead of value:
Item.register_type(
    Color,
    load=lambda name: Color[name],
    dump=lambda c: c.name
)
```

---

## Serializer Hooks

Lifecycle hooks for customizing the load/dump process.

### Pre-Load Hook

Called before `from_dict` processes the input dict. Must return a dict.

```python
@dataclass
class MyClass(JSONWizard):
    name: str

    @classmethod
    def _pre_from_dict(cls, o: dict) -> dict:
        # Transform input before loading
        o['name'] = o.get('name', '').strip()
        return o
```

### Post-Load Hook

Standard dataclass `__post_init__`:

```python
@dataclass
class MyClass(JSONWizard):
    name: str
    name_upper: str = ''

    def __post_init__(self):
        self.name_upper = self.name.upper()
```

### Pre-Dump Hook

Called before `to_dict` processes the instance:

```python
@dataclass
class MyClass(JSONWizard):
    name: str
    _cache: dict = field(default_factory=dict, repr=False)

    def _pre_dict(self):
        # Side effects or transformations before dump
        pass
```

---

## Union Types & Tagging

When a field's type is a Union of multiple dataclasses, use tagging to disambiguate.

### Auto-Assign Tags

```python
@dataclass
class Pet(JSONWizard):
    class Meta(JSONWizard.Meta):
        tag_key = '__type__'        # JSON key for the tag (default: '__tag__')
        auto_assign_tags = True     # Use class name as tag

@dataclass
class Cat(Pet):
    meow_volume: int

@dataclass
class Dog(Pet):
    bark_volume: int

@dataclass
class Owner(JSONWizard):
    pet: Cat | Dog

owner = Owner.from_dict({
    'pet': {'__type__': 'Cat', 'meowVolume': 9}
})
```

### Manual Tags

```python
@dataclass
class Cat(JSONWizard):
    class Meta(JSONWizard.Meta):
        tag = 'cat'
    meow_volume: int

@dataclass
class Dog(JSONWizard):
    class Meta(JSONWizard.Meta):
        tag = 'dog'
    bark_volume: int
```

### Functional API

```python
LoadMeta(tag_key='type', auto_assign_tags=True).bind_to(Pet)
```

---

## Recursive / Cyclic Dataclasses

For self-referential or mutually recursive dataclasses:

```python
from __future__ import annotations

@dataclass
class Node(JSONWizard):
    class Meta(JSONWizard.Meta):
        recursive_classes = True

    value: int
    children: list[Node] = field(default_factory=list)
```

Or via functional API:
```python
LoadMeta(recursive_classes=True).bind_to(Node)
```

---

## Unknown JSON Keys

### Default Behavior

Unknown keys are silently ignored (or warned in debug mode).

### Raise on Unknown Keys (v0)

```python
class Meta(JSONWizard.Meta):
    raise_on_unknown_json_key = True
```

### Handle Unknown Keys (v1)

```python
class Meta(JSONWizard.Meta):
    v1 = True
    v1_on_unknown_key = 'RAISE'  # or 'WARN' or 'IGNORE'
```

### Capture Unknown Keys with CatchAll

```python
from dataclass_wizard import JSONWizard, CatchAll

@dataclass
class Flexible(JSONWizard):
    name: str
    extras: CatchAll     # Captures all unmapped keys as a dict
```

---

## Key Transforms & Casing

### v0 Key Transforms (LetterCase)

| Value | Transform | Example |
|-------|-----------|---------|
| `'CAMEL'` | camelCase | `my_field` → `myField` |
| `'PASCAL'` | PascalCase | `my_field` → `MyField` |
| `'SNAKE'` | snake_case | `myField` → `my_field` |
| `'LISP'` | lisp-case (kebab) | `my_field` → `my-field` |
| `'NONE'` | No transform | `my_field` → `my_field` |

```python
# Via Meta:
class Meta(JSONWizard.Meta):
    key_transform_with_load = 'SNAKE'
    key_transform_with_dump = 'CAMEL'

# Via functional API:
LoadMeta(key_transform='CAMEL').bind_to(MyClass)
DumpMeta(key_transform='SNAKE').bind_to(MyClass)
```

### v1 Key Transforms (KeyCase)

| Value | Transform | Example |
|-------|-----------|---------|
| `'CAMEL'` / `'C'` | camelCase | `my_field` → `myField` |
| `'PASCAL'` / `'P'` | PascalCase | `my_field` → `MyField` |
| `'KEBAB'` / `'K'` | kebab-case | `my_field` → `my-field` |
| `'SNAKE'` / `'S'` | snake_case | `myField` → `my_field` |
| `'AUTO'` / `'A'` | Auto-detect | Tries all transforms at runtime |

```python
# Via subclass params (auto-enables v1):
@dataclass
class MyClass(JSONWizard, case='CAMEL'):
    my_field: str

# Via Meta:
class Meta(JSONWizard.Meta):
    v1 = True
    v1_case = 'CAMEL'          # both directions
    v1_load_case = 'AUTO'      # load only
    v1_dump_case = 'SNAKE'     # dump only
```

---

## Container Class

`Container[T]` is a `list` subclass returned by `JSONListWizard`. It adds helper methods:

```python
from dataclass_wizard import Container

items: Container[MyClass]

# Pretty-print as JSON
print(items.prettify())

# Serialize to JSON string
json_str = items.to_json()

# Write to file
items.to_json_file("output.json")

# Access model class
print(items.__model__)  # MyClass
```

**Methods:**

| Method | Signature | Description |
|--------|-----------|-------------|
| `prettify` | `(encoder=json.dumps, ensure_ascii=False, **kwargs)` | Pretty JSON string |
| `to_json` | `(encoder=json.dumps, **kwargs)` | JSON string |
| `to_json_file` | `(file, mode='w', encoder=json.dump, **kwargs)` | Write to file |

---

## property_wizard Metaclass

Enables `@property` decorators on dataclass fields.

```python
from dataclasses import dataclass
from dataclass_wizard import JSONWizard, property_wizard

@dataclass
class User(JSONWizard, metaclass=property_wizard):
    _name: str = ''

    @property
    def name(self) -> str:
        return self._name

    @name.setter
    def name(self, value: str):
        self._name = value.strip().title()

user = User.from_dict({"name": "  john doe  "})
print(user.name)  # "John Doe"
```

- Works with both `_field` (private) and `field` (public) naming patterns
- Automatically infers defaults from annotations
- Handles mutable defaults via `default_factory`

---

## Enums Reference

### `LetterCase` (v0 key transforms)

```python
from dataclass_wizard.enums import LetterCase

LetterCase.CAMEL    # camelCase
LetterCase.PASCAL   # PascalCase
LetterCase.SNAKE    # snake_case
LetterCase.LISP     # lisp-case
LetterCase.NONE     # no transform
```

### `DateTimeTo` (v0 datetime serialization)

```python
from dataclass_wizard.enums import DateTimeTo

DateTimeTo.ISO_FORMAT   # ISO 8601 string (default)
DateTimeTo.TIMESTAMP    # Unix timestamp (seconds)
```

### `LetterCasePriority` (EnvWizard lookup)

```python
from dataclass_wizard.enums import LetterCasePriority

LetterCasePriority.SCREAMING_SNAKE  # MY_FIELD_NAME (default)
LetterCasePriority.SNAKE            # my_field_name
LetterCasePriority.CAMEL            # myFieldName
LetterCasePriority.PASCAL           # MyFieldName
```

### `KeyCase` (v1 key transforms)

```python
from dataclass_wizard.v1.enums import KeyCase

KeyCase.CAMEL   # or KeyCase.C
KeyCase.PASCAL  # or KeyCase.P
KeyCase.KEBAB   # or KeyCase.K
KeyCase.SNAKE   # or KeyCase.S
KeyCase.AUTO    # or KeyCase.A
```

### `KeyAction` (v1 unknown key handling)

```python
from dataclass_wizard.v1.enums import KeyAction

KeyAction.IGNORE  # Skip silently
KeyAction.RAISE   # Raise exception
KeyAction.WARN    # Log warning
```

### `EnvKeyStrategy` (v1 env key resolution)

```python
from dataclass_wizard.v1.enums import EnvKeyStrategy

EnvKeyStrategy.ENV          # MY_FIELD > my_field (default)
EnvKeyStrategy.FIELD_FIRST  # myField > MY_FIELD > my_field
EnvKeyStrategy.STRICT       # Explicit keys only (kwargs + aliases)
```

### `EnvPrecedence` (v1 env value precedence)

```python
from dataclass_wizard.v1.enums import EnvPrecedence

EnvPrecedence.SECRETS_ENV_DOTENV  # secrets > env > dotenv (default)
EnvPrecedence.SECRETS_DOTENV_ENV  # secrets > dotenv > env
EnvPrecedence.ENV_ONLY            # env vars only
```

### `V1DateTimeTo` (v1 datetime serialization)

```python
from dataclass_wizard.v1.enums import DateTimeTo as V1DateTimeTo

V1DateTimeTo.ISO        # ISO 8601 string (default)
V1DateTimeTo.TIMESTAMP  # Unix timestamp
```

---

## Errors & Exceptions

All exceptions inherit from `JSONWizardError`.

| Exception | When Raised |
|-----------|-------------|
| `ParseError` | Type conversion/parsing failure |
| `MissingFields` | Required fields not provided in input |
| `MissingData` | Input data is missing or empty |
| `UnknownKeysError` | Unknown JSON keys (when `raise_on_unknown_json_key=True` or `v1_on_unknown_key='RAISE'`) |
| `ExtraData` | Unexpected extra data in input |
| `RecursiveClassError` | Recursive class definition without `recursive_classes=True` |

```python
from dataclass_wizard.errors import ParseError, MissingFields, UnknownKeysError
```

All errors include contextual information: class name, field name, expected type, and actual value.

---

## Supported Types

### Primitives
`str`, `int`, `float`, `bool`, `None`, `bytes`, `bytearray`

### Numeric
`Decimal`

### Collections
`list`, `tuple`, `set`, `frozenset`, `dict`, `defaultdict`, `deque`, `OrderedDict`

### ABC Collections
`Sequence`, `MutableSequence`, `Collection`

### Typed Structures
`TypedDict`, `NamedTuple`, `namedtuple`

### Date/Time
`datetime`, `date`, `time`, `timedelta`

### Identity
`UUID`

### Path
`pathlib.Path`

### Enums
`Enum`, `StrEnum`, `IntEnum`

### Typing Constructs
`Union`, `Optional`, `Literal`, `LiteralString`, `Annotated`, `Any`, `Required`, `NotRequired`, `ReadOnly`

### Nested
Nested dataclasses (including recursive/cyclic with `recursive_classes=True`)

### Special
`CatchAll` (captures unmapped keys)

### Custom
Any type via `register_type()` hooks

---

## v0 vs v1 Engine

| Feature | v0 (Default) | v1 (Opt-in) |
|---------|-------------|-------------|
| Mechanism | Runtime dispatch | Code generation (compiled) |
| Performance | Good | Faster |
| Enable | Default | `v1 = True` in Meta or `case=` param |
| Key transforms | `LetterCase` | `KeyCase` (adds AUTO, KEBAB) |
| Unknown keys | `raise_on_unknown_json_key` | `v1_on_unknown_key` (IGNORE/WARN/RAISE) |
| Field aliases | `json_key`/`json_field` | `Alias()`/`AliasPath()` + bulk `v1_field_to_alias` |
| Date patterns | `DatePattern`/`TimePattern`/`DateTimePattern` | Adds `Aware*Pattern`, `UTC*Pattern` |
| Env features | Basic | `EnvPrecedence`, `EnvKeyStrategy`, nested dataclass support |
| Custom hooks | Runtime callables | Codegen hooks (return code strings) |

**Enabling v1:**
```python
# Option 1: Meta class
class Meta(JSONWizard.Meta):
    v1 = True

# Option 2: Subclass parameter (auto-enables v1)
class MyClass(JSONWizard, case='CAMEL'):
    ...

# Option 3: Functional API
LoadMeta(v1=True).bind_to(MyClass)
```

---

## CLI Tool (wiz)

The `wiz` command generates dataclass schemas from JSON.

```bash
# Install
pip install dataclass-wizard[cli]

# Generate dataclass from JSON
echo '{"firstName": "John", "age": 30}' | wiz gs

# From file
wiz gs -f input.json

# With options
wiz gs -f input.json --snake-case --wizard-mixin
```

---

## Optional Dependencies

Install extras with `pip install dataclass-wizard[extra_name]`:

| Extra | Package | For |
|-------|---------|-----|
| `dotenv` | python-dotenv | `.env` file support in EnvWizard |
| `timedelta` | pytimeparse | `timedelta` string parsing |
| `toml` | tomli / tomli-w | TOML support |
| `tz` | pytz / tzdata | Timezone data |
| `yaml` | PyYAML | YAML support |

---

## Common Patterns

### Disable `__str__` Override

```python
@dataclass
class MyClass(JSONWizard, str=False):
    name: str
```

### Global Meta for All Classes

```python
# Set default for ALL JSONWizard subclasses
class _(JSONWizard.Meta):
    key_transform_with_dump = 'SNAKE'
    skip_defaults = True
```

### Exclude Fields from Serialization

```python
# Via json_field
name: str = json_field('name', dump=False)

# Via Alias (v1)
name: str = Alias('name', skip=True)

# Via asdict exclude parameter
d = asdict(obj, exclude={'password', 'secret'})
```

### Multiple Key Mappings

```python
# Accept multiple JSON key names for the same field
name: str = json_field(('name', 'username', 'user_name'), all=True)
```

### Custom JSON Key Mapping via Meta

```python
class Meta(JSONWizard.Meta):
    json_key_to_field = {
        'SomeCrazyKey': 'my_field'
    }
```

### DateTime as Timestamps

```python
class Meta(JSONWizard.Meta):
    marshal_date_time_as = 'TIMESTAMP'     # v0
    # or
    v1 = True
    v1_dump_date_time_as = 'TIMESTAMP'     # v1
```

### Combining Multiple Mixins

```python
@dataclass
class Config(JSONWizard, JSONFileWizard, TOMLWizard, YAMLWizard):
    name: str
    port: int
```
