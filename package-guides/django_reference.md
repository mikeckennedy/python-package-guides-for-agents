# Django 6.1 Comprehensive Reference

> Generated from the Django source code at the `main` branch. This reference covers all major APIs, class signatures, method signatures, settings, and utilities for building with Django.

---

## Table of Contents

1. [Models & ORM](#1-models--orm)
2. [QuerySet API](#2-queryset-api)
3. [Model Fields](#3-model-fields)
4. [Expressions, Aggregates & Lookups](#4-expressions-aggregates--lookups)
5. [Constraints & Indexes](#5-constraints--indexes)
6. [Views](#6-views)
7. [URL Routing](#7-url-routing)
8. [HTTP Request & Response](#8-http-request--response)
9. [Shortcuts](#9-shortcuts)
10. [Forms](#10-forms)
11. [Form Fields & Widgets](#11-form-fields--widgets)
12. [FormSets & Model Forms](#12-formsets--model-forms)
13. [Admin](#13-admin)
14. [Authentication & Authorization](#14-authentication--authorization)
15. [Middleware](#15-middleware)
16. [Templates](#16-templates)
17. [Email](#17-email)
18. [Caching](#18-caching)
19. [Signals](#19-signals)
20. [Pagination](#20-pagination)
21. [Serialization](#21-serialization)
22. [Management Commands](#22-management-commands)
23. [Testing](#23-testing)
24. [Settings Reference](#24-settings-reference)
25. [Exceptions](#25-exceptions)
26. [Utilities](#26-utilities)
27. [View Decorators](#27-view-decorators)

---

## 1. Models & ORM

**Source:** `django/db/models/base.py`

### Model Base Class

```python
from django.db import models

class MyModel(models.Model):
    class Meta:
        # Model options
        ...
```

### Model Methods

```python
class Model:
    def save(self, *, force_insert=False, force_update=False, using=None, update_fields=None)
    async def asave(self, *, force_insert=False, force_update=False, using=None, update_fields=None)

    def delete(self, using=None, keep_parents=False)
    async def adelete(self)

    def clean(self)
    def full_clean(self, exclude=None, validate_unique=True, validate_constraints=True)
    def clean_fields(self, exclude=None)

    def refresh_from_db(self, using=None, fields=None, from_queryset=None)
    async def arefresh_from_db(self, using=None, fields=None, from_queryset=None)

    def serializable_value(self, field_name)

    @classmethod
    def from_db(cls, db, init_list, init_values, fetch_mode=FETCH_ONE)
```

### Manager

**Source:** `django/db/models/manager.py`

```python
class Manager(BaseManager):
    def get_queryset(self)
    def all(self)
    # All QuerySet methods are proxied through Manager
```

---

## 2. QuerySet API

**Source:** `django/db/models/query.py`

```python
class QuerySet:
    def __init__(self, model=None, query=None, using=None, hints=None)
```

### Filtering & Retrieval

```python
    def filter(self, *args, **kwargs)           # Return filtered QuerySet
    def exclude(self, *args, **kwargs)          # Return excluded QuerySet
    def get(self, *args, **kwargs)              # Return single object
    async def aget(self, *args, **kwargs)
    def first(self)                             # Return first object or None
    def last(self)                              # Return last object or None
    def earliest(self, *fields)                 # Return earliest by field(s)
    def latest(self, *fields)                   # Return latest by field(s)
    def exists(self)                            # Return True if any results
    async def aexists(self)
    def contains(self, obj)                     # Check if obj is in QuerySet
    def count(self)                             # Return count of results
    async def acount(self)
```

### Creation & Updates

```python
    def create(self, **kwargs)                  # Create and save an object
    async def acreate(self, **kwargs)
    def get_or_create(self, defaults=None, **kwargs)
    async def aget_or_create(self, defaults=None, **kwargs)
    def update_or_create(self, defaults=None, create_defaults=None, **kwargs)
    async def aupdate_or_create(self, defaults=None, create_defaults=None, **kwargs)
    def bulk_create(self, objs, batch_size=None, ignore_conflicts=False,
                    update_conflicts=False, update_fields=None, unique_fields=None)
    async def abulk_create(...)
    def bulk_update(self, objs, fields, batch_size=None)
    async def abulk_update(...)
    def update(self, **kwargs)                  # Update matching objects
    def delete(self)                            # Delete matching objects
```

### Annotation & Aggregation

```python
    def annotate(self, *args, **kwargs)         # Add annotation expressions
    def aggregate(self, *args, **kwargs)        # Compute aggregate values
    async def aaggregate(self, *args, **kwargs)
```

### Ordering & Slicing

```python
    def order_by(self, *field_names)            # Order results
    def reverse(self)                           # Reverse ordering
    def distinct(self, *fields)                 # Remove duplicates
```

### Related Object Loading

```python
    def select_related(self, *fields)           # Follow FK joins in SQL
    def prefetch_related(self, *lookups)         # Prefetch related objects
    def defer(self, *fields)                    # Defer field loading
    def only(self, *fields)                     # Load only specified fields
```

### Output Transformations

```python
    def values(self, *fields, **expressions)    # Return dicts
    def values_list(self, *fields, flat=False, named=False)  # Return tuples
```

### Set Operations

```python
    def union(self, *other_qs, all=False)
    def intersection(self, *other_qs)
    def difference(self, *other_qs)
```

### Database Control

```python
    def using(self, alias)                      # Select database
    def select_for_update(self, nowait=False, skip_locked=False, of=())
    def explain(self, *, prefix=None, verbose=False, analyze=False, **options)
    def iterator(self, chunk_size=None)         # Iterate without caching
    async def aiterator(self, chunk_size=2000)
    def extra(self, select=None, where=None, params=None,
              tables=None, order_by=None, select_params=None)
```

### Iterable Classes

| Class | Yields |
|---|---|
| `ModelIterable` | Model instances |
| `ValuesIterable` | Dictionaries |
| `ValuesListIterable` | Tuples |
| `NamedValuesListIterable` | Named tuples |
| `FlatValuesListIterable` | Single values |
| `RawModelIterable` | Raw query instances |

### Prefetch Object

```python
class Prefetch:
    def __init__(self, lookup, queryset=None, to_attr=None)
```

---

## 3. Model Fields

**Source:** `django/db/models/fields/__init__.py`

### Base Field

```python
class Field:
    def __init__(
        self,
        verbose_name=None,
        name=None,
        primary_key=False,
        max_length=None,
        unique=False,
        blank=False,
        null=False,
        db_index=False,
        rel=None,
        default=NOT_PROVIDED,
        editable=True,
        serialize=True,
        unique_for_date=None,
        unique_for_month=None,
        unique_for_year=None,
        choices=None,
        help_text="",
        db_column=None,
        db_tablespace=None,
        auto_created=False,
        validators=(),
        error_messages=None,
        db_comment=None,
        db_default=NOT_PROVIDED,
    )
```

### Numeric Fields

| Field | Extra Parameters |
|---|---|
| `AutoField` | — |
| `BigAutoField` | — |
| `SmallAutoField` | — |
| `IntegerField` | — |
| `BigIntegerField` | — |
| `SmallIntegerField` | — |
| `PositiveIntegerField` | — |
| `PositiveBigIntegerField` | — |
| `PositiveSmallIntegerField` | — |
| `FloatField` | — |
| `DecimalField` | `max_digits=None, decimal_places=None` |

### String Fields

| Field | Extra Parameters |
|---|---|
| `CharField` | `max_length=None` |
| `TextField` | — |
| `SlugField` | inherits CharField |
| `URLField` | inherits CharField |
| `EmailField` | inherits CharField |

### Date/Time Fields

| Field | Extra Parameters |
|---|---|
| `DateField` | `auto_now=False, auto_now_add=False` |
| `DateTimeField` | `auto_now=False, auto_now_add=False` |
| `TimeField` | `auto_now=False, auto_now_add=False` |
| `DurationField` | — |

### Other Fields

| Field | Extra Parameters |
|---|---|
| `BooleanField` | — |
| `BinaryField` | — |
| `UUIDField` | — |
| `GenericIPAddressField` | `protocol='both', unpack_ipv4=False` |
| `FilePathField` | `path='', match=None, recursive=False, allow_files=True, allow_folders=False` |

### File Fields

**Source:** `django/db/models/fields/files.py`

```python
class FileField(Field):
    def __init__(self, verbose_name=None, name=None, upload_to="", storage=None, **kwargs)

class ImageField(FileField):
    # Same signature as FileField
```

### JSON Field

**Source:** `django/db/models/fields/json.py`

```python
class JSONField(Field):
    def __init__(self, verbose_name=None, name=None, encoder=None, decoder=None, **kwargs)
```

**JSON Lookups:** `contains`, `contained_by`, `has_key`, `has_keys`, `has_any_keys`

### Related Fields

**Source:** `django/db/models/fields/related.py`

```python
class ForeignKey(ForeignObject):
    def __init__(
        self,
        to,                         # Related model
        on_delete,                  # CASCADE, PROTECT, SET_NULL, SET_DEFAULT, SET(), DO_NOTHING
        related_name=None,
        related_query_name=None,
        limit_choices_to=None,
        parent_link=False,
        to_field=None,
        db_constraint=True,
        **kwargs,
    )

class OneToOneField(ForeignKey):
    # Same signature as ForeignKey

class ManyToManyField(RelatedField):
    def __init__(
        self,
        to,
        related_name=None,
        related_query_name=None,
        limit_choices_to=None,
        symmetrical=None,
        through=None,
        through_fields=None,
        db_table=None,
        db_constraint=True,
        swappable=True,
        **kwargs,
    )
```

### on_delete Options

| Option | Behavior |
|---|---|
| `models.CASCADE` | Delete related objects |
| `models.PROTECT` | Prevent deletion |
| `models.RESTRICT` | Like PROTECT but allows cascading deletes |
| `models.SET_NULL` | Set FK to NULL (requires `null=True`) |
| `models.SET_DEFAULT` | Set FK to default value |
| `models.SET(value)` | Set FK to given value or callable |
| `models.DO_NOTHING` | Take no action |

---

## 4. Expressions, Aggregates & Lookups

### Expressions

**Source:** `django/db/models/expressions.py`

```python
from django.db.models import F, Value, Case, When, Exists, Subquery, OuterRef, ExpressionWrapper

F(name)                                             # Field reference
Value(value, output_field=None)                     # Literal value
Case(*cases, default=None, output_field=None)       # CASE/WHEN
When(condition, then)                               # WHEN clause
Subquery(queryset, output_field=None)               # Subquery
OuterRef(name)                                      # Reference to outer query
Exists(queryset)                                    # EXISTS subquery
ExpressionWrapper(expression, output_field)         # Wrap expression with output type
Func(*expressions, **extra)                         # Database function
RawSQL(sql, params)                                 # Raw SQL expression
Window(*expressions, frame=None, order_by=None, partition_by=None)
WindowFrame(start=None, end=None)
OrderBy(expression, *, descending=False, nulls_first=None, nulls_last=None)
```

### Expression Operators

```python
# F() expressions support arithmetic
F('price') + 10
F('price') - F('discount')
F('quantity') * F('unit_price')
F('total') / F('count')
F('value') % 2
F('value') ** 2
# Bitwise: bitand, bitor, bitxor, bitleftshift, bitrightshift
```

### Aggregates

**Source:** `django/db/models/aggregates.py`

```python
class Aggregate(Func):
    def __init__(self, *expressions, distinct=False, filter=None,
                 default=None, order_by=None, **extra)

# Specific aggregates:
Avg(expression, output_field=None, filter=None, default=None, **extra)
Count(expression, filter=None, **extra)
Max(expression, output_field=None, filter=None, default=None, **extra)
Min(expression, output_field=None, filter=None, default=None, **extra)
Sum(expression, output_field=None, filter=None, default=None, **extra)
StdDev(expression, sample=False, **extra)
Variance(expression, sample=False, **extra)
StringAgg(expression, delimiter, **extra)
AnyValue(expression)
```

### Lookups

**Source:** `django/db/models/lookups.py`

| Lookup | SQL Equivalent | Example |
|---|---|---|
| `exact` | `= value` | `name__exact='John'` |
| `iexact` | `ILIKE value` | `name__iexact='john'` |
| `contains` | `LIKE '%value%'` | `name__contains='oh'` |
| `icontains` | `ILIKE '%value%'` | `name__icontains='oh'` |
| `startswith` | `LIKE 'value%'` | `name__startswith='J'` |
| `istartswith` | `ILIKE 'value%'` | `name__istartswith='j'` |
| `endswith` | `LIKE '%value'` | `name__endswith='n'` |
| `iendswith` | `ILIKE '%value'` | `name__iendswith='N'` |
| `in` | `IN (...)` | `id__in=[1, 2, 3]` |
| `gt` | `>` | `age__gt=18` |
| `gte` | `>=` | `age__gte=18` |
| `lt` | `<` | `age__lt=65` |
| `lte` | `<=` | `age__lte=65` |
| `range` | `BETWEEN ... AND ...` | `age__range=(18, 65)` |
| `isnull` | `IS NULL` / `IS NOT NULL` | `email__isnull=True` |
| `regex` | `~ regex` | `name__regex=r'^J'` |
| `iregex` | `~* regex` | `name__iregex=r'^j'` |

**Date/Time Lookups:** `year`, `month`, `day`, `week_day`, `hour`, `minute`, `second`

### Q Objects

```python
from django.db.models import Q

Q(name='John')                  # Simple condition
Q(name='John') | Q(name='Jane')  # OR
Q(name='John') & Q(age=25)      # AND
~Q(name='John')                  # NOT
```

---

## 5. Constraints & Indexes

### Constraints

**Source:** `django/db/models/constraints.py`

```python
class BaseConstraint:
    def __init__(self, *, name, violation_error_code=None, violation_error_message=None)

class CheckConstraint(BaseConstraint):
    def __init__(self, *, condition, name, violation_error_code=None,
                 violation_error_message=None)

class UniqueConstraint(BaseConstraint):
    def __init__(
        self,
        *expressions,
        fields=(),
        name,
        condition=None,
        deferrable=None,           # Deferrable.DEFERRED or Deferrable.IMMEDIATE
        include=None,
        opclasses=(),
        nulls_distinct=None,
        violation_error_code=None,
        violation_error_message=None,
    )
```

### Indexes

**Source:** `django/db/models/indexes.py`

```python
class Index:
    def __init__(
        self,
        *expressions,
        fields=(),
        name=None,
        db_tablespace=None,
        opclasses=(),
        condition=None,
        include=None,
    )
```

---

## 6. Views

### Base Views

**Source:** `django/views/generic/base.py`

```python
class View:
    http_method_names = ["get", "post", "put", "patch", "delete", "head", "options", "trace"]

    def __init__(self, **kwargs)
    @classmethod
    def as_view(cls, **initkwargs)              # Returns a callable view
    def setup(self, request, *args, **kwargs)    # Initialize per-request state
    def dispatch(self, request, *args, **kwargs) # Route to method handler
    def http_method_not_allowed(self, request, *args, **kwargs)
    def options(self, request, *args, **kwargs)

class TemplateView(TemplateResponseMixin, ContextMixin, View):
    def get(self, request, *args, **kwargs)

class RedirectView(View):
    permanent = False
    url = None
    pattern_name = None
    query_string = False
    def get_redirect_url(self, *args, **kwargs)
```

### Mixins

```python
class ContextMixin:
    extra_context = None
    def get_context_data(self, **kwargs)

class TemplateResponseMixin:
    template_name = None
    template_engine = None
    response_class = TemplateResponse
    content_type = None
    def render_to_response(self, context, **response_kwargs)
    def get_template_names(self)
```

### List Views

**Source:** `django/views/generic/list.py`

```python
class MultipleObjectMixin(ContextMixin):
    allow_empty = True
    queryset = None
    model = None
    paginate_by = None
    paginate_orphans = 0
    context_object_name = None
    paginator_class = Paginator
    page_kwarg = "page"
    ordering = None

    def get_queryset(self)
    def get_ordering(self)
    def paginate_queryset(self, queryset, page_size)
    def get_paginate_by(self, queryset)
    def get_paginator(self, queryset, per_page, orphans=0, allow_empty_first_page=True, **kwargs)
    def get_context_object_name(self, object_list)
    def get_context_data(self, *, object_list=None, **kwargs)

class ListView(MultipleObjectTemplateResponseMixin, BaseListView):
    # template_name_suffix = "_list"
    pass
```

### Detail Views

**Source:** `django/views/generic/detail.py`

```python
class SingleObjectMixin(ContextMixin):
    model = None
    queryset = None
    slug_field = "slug"
    context_object_name = None
    slug_url_kwarg = "slug"
    pk_url_kwarg = "pk"
    query_pk_and_slug = False

    def get_object(self, queryset=None)
    def get_queryset(self)
    def get_slug_field(self)
    def get_context_object_name(self, obj)
    def get_context_data(self, **kwargs)

class DetailView(SingleObjectTemplateResponseMixin, BaseDetailView):
    # template_name_suffix = "_detail"
    pass
```

### Edit Views

**Source:** `django/views/generic/edit.py`

```python
class FormMixin(ContextMixin):
    initial = {}
    form_class = None
    success_url = None
    prefix = None

    def get_initial(self)
    def get_prefix(self)
    def get_form_class(self)
    def get_form(self, form_class=None)
    def get_form_kwargs(self)
    def get_success_url(self)
    def form_valid(self, form)
    def form_invalid(self, form)

class FormView(TemplateResponseMixin, BaseFormView):
    pass

class CreateView(SingleObjectTemplateResponseMixin, BaseCreateView):
    template_name_suffix = "_form"

class UpdateView(SingleObjectTemplateResponseMixin, BaseUpdateView):
    template_name_suffix = "_form"

class DeleteView(SingleObjectTemplateResponseMixin, BaseDeleteView):
    template_name_suffix = "_confirm_delete"
```

### Date-Based Views

**Source:** `django/views/generic/dates.py`

| View | Description | Template Suffix |
|---|---|---|
| `ArchiveIndexView` | Top-level archive | `_archive` |
| `YearArchiveView` | Year archive | `_archive_year` |
| `MonthArchiveView` | Month archive | `_archive_month` |
| `WeekArchiveView` | Week archive | `_archive_week` |
| `DayArchiveView` | Day archive | `_archive_day` |
| `TodayArchiveView` | Today's archive | `_archive_day` |
| `DateDetailView` | Detail by date | (inherits detail) |

**Common date mixin attributes:** `date_field`, `allow_future`, `year_format`, `month_format`, `day_format`, `week_format`

---

## 7. URL Routing

**Source:** `django/urls/`

### URL Configuration

```python
from django.urls import path, re_path, include, reverse, reverse_lazy

path(route, view, kwargs=None, name=None)
    # Returns URLPattern or URLResolver

re_path(route, view, kwargs=None, name=None)
    # Same as path() but uses regex patterns

include(arg, namespace=None)
    # Include another URLconf module
```

### URL Resolution

```python
reverse(viewname, urlconf=None, args=None, kwargs=None, current_app=None,
        *, query=None, fragment=None)
    # Returns URL string

reverse_lazy(viewname, ...)
    # Lazy version of reverse()

resolve(path, urlconf=None)
    # Returns ResolverMatch

is_valid_path(path, urlconf=None)
    # Returns bool

register_converter(converter, type_name)
    # Register a custom URL converter
```

### ResolverMatch

```python
class ResolverMatch:
    func            # The matched view function
    args            # Positional arguments
    kwargs          # Keyword arguments
    url_name        # URL pattern name
    route           # The matched route string
    app_names       # List of app names
    app_name        # Colon-separated app name string
    namespaces      # List of namespaces
    namespace       # Colon-separated namespace string
    view_name       # Full view name (namespace:name)
```

### Built-in Path Converters

| Converter | Matches |
|---|---|
| `int` | Zero or positive integer |
| `str` | Non-empty string (excluding `/`) |
| `slug` | ASCII letters, numbers, hyphens, underscores |
| `uuid` | UUID format |
| `path` | Non-empty string (including `/`) |

---

## 8. HTTP Request & Response

### HttpRequest

**Source:** `django/http/request.py`

```python
class HttpRequest:
    # Attributes
    method              # "GET", "POST", etc.
    path                # URL path (e.g., "/music/bands/")
    path_info           # Path portion used for URL resolution
    scheme              # "http" or "https"
    body                # Raw HTTP body as bytes
    encoding            # Encoding for GET/POST data
    content_type        # MIME type of the request
    content_params      # Dict of content-type parameters
    GET                 # QueryDict of GET parameters
    POST                # QueryDict of POST parameters
    COOKIES             # Dict of cookies
    FILES               # MultiValueDict of uploaded files
    META                # Dict of HTTP headers and server info
    headers             # CaseInsensitiveMapping of HTTP headers
    resolver_match      # ResolverMatch instance

    # Methods
    def get_host(self)
    def get_port(self)
    def get_full_path(self, force_append_slash=False)
    def get_full_path_info(self, force_append_slash=False)
    def build_absolute_uri(self, location=None)
    def get_signed_cookie(self, key, default=RAISE_ERROR, salt="", max_age=None)
    def is_secure(self)
    def accepts(self, media_type)
    def get_preferred_type(self, media_types)

    # Properties
    accepted_types              # List of accepted media types
    accepted_types_by_precedence  # Sorted by quality value
```

### QueryDict

```python
class QueryDict(MultiValueDict):
    def __init__(self, query_string=None, mutable=False, encoding=None)
    # Extends MultiValueDict with immutability by default
```

### HttpResponse

**Source:** `django/http/response.py`

```python
class HttpResponseBase:
    def __init__(self, content_type=None, status=None, reason=None,
                 charset=None, headers=None)
    status_code = 200
    def __setitem__(self, header, value)
    def __getitem__(self, header)
    def __delitem__(self, header)
    def has_header(self, header)
    def set_cookie(self, key, value="", max_age=None, expires=None, path="/",
                   domain=None, secure=False, httponly=False, samesite=None)
    def set_signed_cookie(self, key, value, salt="", **kwargs)
    def delete_cookie(self, key, path="/", domain=None, samesite=None)

class HttpResponse(HttpResponseBase):
    streaming = False
    content             # property: body bytes
    text                # property: body as string

class JsonResponse(HttpResponse):
    def __init__(self, data, encoder=DjangoJSONEncoder, safe=True,
                 json_dumps_params=None, **kwargs)

class StreamingHttpResponse(HttpResponseBase):
    streaming = True
    streaming_content   # property: iterator

class FileResponse(StreamingHttpResponse):
    def __init__(self, *args, as_attachment=False, filename="", **kwargs)
    block_size = 4096
```

### Response Status Classes

| Class | Status Code |
|---|---|
| `HttpResponse` | 200 |
| `HttpResponseRedirect` | 302 (307 with `preserve_request=True`) |
| `HttpResponsePermanentRedirect` | 301 (308 with `preserve_request=True`) |
| `HttpResponseNotModified` | 304 |
| `HttpResponseBadRequest` | 400 |
| `HttpResponseForbidden` | 403 |
| `HttpResponseNotFound` | 404 |
| `HttpResponseNotAllowed` | 405 |
| `HttpResponseGone` | 410 |
| `HttpResponseServerError` | 500 |

---

## 9. Shortcuts

**Source:** `django/shortcuts.py`

```python
from django.shortcuts import render, redirect, get_object_or_404, get_list_or_404

render(request, template_name, context=None, content_type=None, status=None, using=None)
    # Returns HttpResponse with rendered template

redirect(to, *args, permanent=False, preserve_request=False, **kwargs)
    # Returns HttpResponseRedirect or HttpResponsePermanentRedirect

get_object_or_404(klass, *args, **kwargs)
    # Returns object or raises Http404
async aget_object_or_404(klass, *args, **kwargs)

get_list_or_404(klass, *args, **kwargs)
    # Returns list or raises Http404
async aget_list_or_404(klass, *args, **kwargs)

resolve_url(to, *args, **kwargs)
    # Resolve a URL from a model, view name, or absolute/relative path
```

---

## 10. Forms

**Source:** `django/forms/forms.py`

### BaseForm

```python
class BaseForm:
    def __init__(
        self,
        data=None,                      # Bound data dict
        files=None,                     # Uploaded files dict
        auto_id="id_%s",               # Auto-generated field IDs
        prefix=None,                    # Form prefix for namespacing
        initial=None,                   # Initial values dict
        error_class=ErrorList,
        label_suffix=None,
        empty_permitted=False,
        field_order=None,
        use_required_attribute=None,
        renderer=None,
        bound_field_class=None,
    )

    # Validation
    def is_valid(self)                  # Returns bool
    def full_clean(self)
    def clean(self)                     # Override for cross-field validation
    def has_error(self, field, code=None)
    def add_error(self, field, error)
    def non_field_errors(self)

    # Data access
    errors                              # property: ErrorDict
    cleaned_data                        # dict of validated data
    changed_data                        # cached_property: list of changed field names
    def has_changed(self)

    # Rendering
    def __iter__(self)                  # Iterate over BoundFields
    def __getitem__(self, name)         # Get BoundField by name
    def hidden_fields(self)
    def visible_fields(self)
    def is_multipart(self)
    media                               # property

class Form(BaseForm, metaclass=DeclarativeFieldsMetaclass):
    pass
```

---

## 11. Form Fields & Widgets

### Form Fields

**Source:** `django/forms/fields.py`

#### Base Field

```python
class Field:
    def __init__(
        self,
        *,
        required=True,
        widget=None,
        label=None,
        initial=None,
        help_text="",
        error_messages=None,
        show_hidden_initial=False,
        validators=(),
        localize=False,
        disabled=False,
        label_suffix=None,
        template_name=None,
        bound_field_class=None,
    )

    def clean(self, value)          # Validate and return cleaned value
    def to_python(self, value)      # Convert to Python object
    def validate(self, value)       # Run validation
    def run_validators(self, value) # Run all validators
    def has_changed(self, initial, data)
    def widget_attrs(self, widget)
```

#### Field Types

| Field | Extra Parameters |
|---|---|
| `CharField` | `max_length=None, min_length=None, strip=True, empty_value=""` |
| `IntegerField` | `max_value=None, min_value=None, step_size=None` |
| `FloatField` | inherits IntegerField |
| `DecimalField` | `max_value=None, min_value=None, max_digits=None, decimal_places=None` |
| `DateField` | `input_formats=None` |
| `TimeField` | `input_formats=None` |
| `DateTimeField` | `input_formats=None` |
| `DurationField` | — |
| `RegexField` | `regex` |
| `EmailField` | — |
| `FileField` | `max_length=None, allow_empty_file=False` |
| `ImageField` | inherits FileField |
| `URLField` | `assume_scheme=None` |
| `BooleanField` | — |
| `NullBooleanField` | — |
| `ChoiceField` | `choices=()` |
| `TypedChoiceField` | `coerce=lambda val: val, empty_value=""` |
| `MultipleChoiceField` | inherits ChoiceField |
| `TypedMultipleChoiceField` | `coerce=lambda val: val` |
| `ComboField` | `fields` |
| `MultiValueField` | `fields, require_all_fields=True` |
| `FilePathField` | `path, match=None, recursive=False, allow_files=True, allow_folders=False` |
| `SplitDateTimeField` | `input_date_formats=None, input_time_formats=None` |
| `GenericIPAddressField` | `protocol="both", unpack_ipv4=False` |
| `SlugField` | `allow_unicode=False` |
| `UUIDField` | — |
| `JSONField` | `encoder=None, decoder=None` |

### Widgets

**Source:** `django/forms/widgets.py`

```python
class Widget:
    def __init__(self, attrs=None)
    def render(self, name, value, attrs=None, renderer=None)
    def value_from_datadict(self, data, files, name)
    def value_omitted_from_data(self, data, files, name)
    def id_for_label(self, id_)
    def build_attrs(self, base_attrs, extra_attrs=None)
```

#### Widget Types

| Widget | Extra Parameters |
|---|---|
| `TextInput` | — |
| `NumberInput` | — |
| `EmailInput` | — |
| `URLInput` | — |
| `ColorInput` | — |
| `SearchInput` | — |
| `TelInput` | — |
| `PasswordInput` | `render_value=False` |
| `HiddenInput` | — |
| `MultipleHiddenInput` | — |
| `FileInput` | — |
| `ClearableFileInput` | — |
| `Textarea` | — |
| `DateInput` | `format=None` |
| `DateTimeInput` | `format=None` |
| `TimeInput` | `format=None` |
| `CheckboxInput` | `check_test=None` |
| `Select` | `choices=()` |
| `NullBooleanSelect` | — |
| `SelectMultiple` | inherits Select |
| `RadioSelect` | `choices=()` |
| `CheckboxSelectMultiple` | inherits RadioSelect |
| `MultiWidget` | `widgets` |
| `SplitDateTimeWidget` | `date_format=None, time_format=None, date_attrs=None, time_attrs=None` |
| `SelectDateWidget` | `years=None, months=None, empty_label=None` |

---

## 12. FormSets & Model Forms

### FormSets

**Source:** `django/forms/formsets.py`

```python
class BaseFormSet:
    def __init__(
        self,
        data=None,
        files=None,
        auto_id="id_%s",
        prefix=None,
        initial=None,
        error_class=ErrorList,
        form_kwargs=None,
        error_messages=None,
    )

    # Properties
    management_form     # cached_property
    forms               # cached_property: list of form instances
    initial_forms       # property
    extra_forms         # property
    empty_form          # property
    errors              # property
    cleaned_data        # property
    deleted_forms       # property
    ordered_forms       # property

    # Methods
    def total_form_count(self)
    def initial_form_count(self)
    def is_valid(self)
    def full_clean(self)
    def clean(self)                 # Override for custom validation
    def has_changed(self)
    def add_fields(self, form, index)
    def total_error_count(self)
    def non_form_errors(self)

def formset_factory(
    form,
    formset=BaseFormSet,
    extra=1,
    can_order=False,
    can_delete=False,
    max_num=None,
    validate_max=False,
    min_num=None,
    validate_min=False,
    absolute_max=None,
    can_delete_extra=True,
    renderer=None,
)
```

### Model Forms

**Source:** `django/forms/models.py`

```python
class BaseModelForm(BaseForm):
    def __init__(
        self,
        data=None,
        files=None,
        auto_id="id_%s",
        prefix=None,
        initial=None,
        error_class=ErrorList,
        label_suffix=None,
        empty_permitted=False,
        instance=None,              # Model instance to edit
        use_required_attribute=None,
        renderer=None,
    )

    def save(self, commit=True)     # Save the model instance
    def validate_unique(self)
    def validate_constraints(self)

class ModelForm(BaseModelForm, metaclass=ModelFormMetaclass):
    class Meta:
        model = None
        fields = None               # List of fields or "__all__"
        exclude = None
        widgets = None
        labels = None
        help_texts = None
        error_messages = None
        field_classes = None
        localized_fields = None
```

### Factory Functions

```python
def modelform_factory(
    model, form=ModelForm, fields=None, exclude=None,
    formfield_callback=None, widgets=None, localized_fields=None,
    labels=None, help_texts=None, error_messages=None, field_classes=None,
)

def modelformset_factory(
    model, form=ModelForm, formfield_callback=None, formset=BaseModelFormSet,
    extra=1, can_delete=False, can_order=False, max_num=None, fields=None,
    exclude=None, widgets=None, validate_max=False, localized_fields=None,
    labels=None, help_texts=None, error_messages=None, min_num=None,
    validate_min=False, field_classes=None, absolute_max=None,
    can_delete_extra=True, renderer=None, edit_only=False,
)

def inlineformset_factory(
    parent_model, model, form=ModelForm, formset=BaseInlineFormSet,
    fk_name=None, fields=None, exclude=None, extra=3, can_order=False,
    can_delete=True, max_num=None, formfield_callback=None, widgets=None,
    validate_max=False, localized_fields=None, labels=None, help_texts=None,
    error_messages=None, min_num=None, validate_min=False, field_classes=None,
    absolute_max=None, can_delete_extra=True, renderer=None, edit_only=False,
)
```

### Model Choice Fields

```python
class ModelChoiceField(ChoiceField):
    def __init__(
        self, queryset, *, empty_label="---------", required=True,
        widget=None, label=None, initial=None, help_text="",
        to_field_name=None, limit_choices_to=None, blank=False, **kwargs,
    )
    def label_from_instance(self, obj)  # Override to customize display

class ModelMultipleChoiceField(ModelChoiceField):
    def __init__(self, queryset, **kwargs)
```

### Utility Functions

```python
model_to_dict(instance, fields=None, exclude=None)
construct_instance(form, instance, fields=None, exclude=None)
fields_for_model(model, fields=None, exclude=None, widgets=None, ...)
```

---

## 13. Admin

**Source:** `django/contrib/admin/`

### AdminSite

**Source:** `django/contrib/admin/sites.py`

```python
class AdminSite:
    site_title = "Django site admin"
    site_header = "Django administration"
    index_title = "Site administration"
    site_url = "/"
    enable_nav_sidebar = True
    empty_value_display = "-"
    login_form = None
    final_catch_all_view = True

    # Template overrides
    index_template = None
    app_index_template = None
    login_template = None
    logout_template = None
    password_change_template = None
    password_change_done_template = None

    def __init__(self, name="admin")
    def register(self, model_or_iterable, admin_class=None, **options)
    def unregister(self, model_or_iterable)
    def is_registered(self, model)
    def get_model_admin(self, model)
    def has_permission(self, request)
    def get_urls(self)
    def each_context(self, request)
```

### ModelAdmin

**Source:** `django/contrib/admin/options.py`

```python
class ModelAdmin(BaseModelAdmin):
    # List display options
    list_display = ("__str__",)
    list_display_links = ()
    list_filter = ()
    list_select_related = False
    list_per_page = 100
    list_max_show_all = 200
    list_editable = ()
    search_fields = ()
    search_help_text = None
    date_hierarchy = None
    ordering = None
    sortable_by = None
    show_facets = ShowFacets.ALLOW
    show_full_result_count = True
    paginator = Paginator
    preserve_filters = True

    # Form options
    form = forms.ModelForm
    fields = None
    exclude = None
    fieldsets = None
    readonly_fields = ()
    prepopulated_fields = {}
    formfield_overrides = {}
    filter_vertical = ()
    filter_horizontal = ()
    radio_fields = {}
    autocomplete_fields = ()
    raw_id_fields = ()
    view_on_site = True

    # Save options
    save_as = False
    save_as_continue = True
    save_on_top = False

    # Actions
    actions = ()
    action_form = helpers.ActionForm
    actions_on_top = True
    actions_on_bottom = False
    actions_selection_counter = True

    # Inlines
    inlines = ()

    # Template overrides
    add_form_template = None
    change_form_template = None
    change_list_template = None
    delete_confirmation_template = None
    delete_selected_confirmation_template = None
    object_history_template = None
    popup_response_template = None

    def __init__(self, model, admin_site)
```

#### Key ModelAdmin Methods

```python
    # QuerySet & Display
    def get_queryset(self, request)
    def get_list_display(self, request)
    def get_list_display_links(self, request, list_display)
    def get_list_filter(self, request)
    def get_list_select_related(self, request)
    def get_search_fields(self, request)
    def get_search_results(self, request, queryset, search_term)
    def get_ordering(self, request)
    def get_sortable_by(self, request)

    # Forms & Formsets
    def get_form(self, request, obj=None, change=False, **kwargs)
    def get_fields(self, request, obj=None)
    def get_fieldsets(self, request, obj=None)
    def get_exclude(self, request, obj=None)
    def get_readonly_fields(self, request, obj=None)
    def get_prepopulated_fields(self, request, obj=None)
    def get_inlines(self, request, obj)
    def get_inline_instances(self, request, obj=None)
    def get_formsets_with_inlines(self, request, obj=None)
    def get_changelist(self, request, **kwargs)
    def get_changelist_form(self, request, **kwargs)
    def get_changelist_formset(self, request, **kwargs)
    def get_paginator(self, request, queryset, per_page, orphans=0, allow_empty_first_page=True)
    def get_object(self, request, object_id, from_field=None)

    # Permissions
    def has_add_permission(self, request)
    def has_change_permission(self, request, obj=None)
    def has_delete_permission(self, request, obj=None)
    def has_view_permission(self, request, obj=None)
    def has_module_permission(self, request)

    # Save & Delete
    def save_model(self, request, obj, form, change)
    def delete_model(self, request, obj)
    def delete_queryset(self, request, queryset)
    def save_formset(self, request, form, formset, change)
    def save_related(self, request, form, formsets, change)
    def save_form(self, request, form, change)

    # Responses
    def response_add(self, request, obj, post_url_continue=None)
    def response_change(self, request, obj)
    def response_post_save_add(self, request, obj)
    def response_post_save_change(self, request, obj)

    # Views
    def changeform_view(self, request, object_id=None, form_url="", extra_context=None)
    def changelist_view(self, request, extra_context=None)
    def add_view(self, request, form_url="", extra_context=None)
    def change_view(self, request, object_id, form_url="", extra_context=None)
    def delete_view(self, request, object_id, extra_context=None)
    def history_view(self, request, object_id, extra_context=None)

    # Actions & Logging
    def get_actions(self, request)
    def get_action_choices(self, request)
    def log_addition(self, request, obj, message)
    def log_change(self, request, obj, message)
    def log_deletion(self, request, obj_display, obj_id)
    def message_user(self, request, message, level=messages.INFO, extra_tags="", fail_silently=False)
    def construct_change_message(self, request, form, formsets, add=False)
    def get_preserved_filters(self, request)
```

### Inline Admin

```python
class InlineModelAdmin(BaseModelAdmin):
    model = None
    fk_name = None
    formset = BaseInlineFormSet
    extra = 3
    min_num = None
    max_num = None
    template = None
    verbose_name = None
    verbose_name_plural = None
    can_delete = True
    show_change_link = False
    classes = None

    def __init__(self, parent_model, admin_site)
    def get_extra(self, request, obj=None, **kwargs)
    def get_min_num(self, request, obj=None, **kwargs)
    def get_max_num(self, request, obj=None, **kwargs)
    def get_formset(self, request, obj=None, **kwargs)
    def has_add_permission(self, request, obj)

class StackedInline(InlineModelAdmin):
    template = "admin/edit_inline/stacked.html"

class TabularInline(InlineModelAdmin):
    template = "admin/edit_inline/tabular.html"
```

### Register Decorator

```python
from django.contrib import admin

@admin.register(MyModel)
class MyModelAdmin(admin.ModelAdmin):
    ...

# Or functional registration:
admin.site.register(MyModel, MyModelAdmin)
```

---

## 14. Authentication & Authorization

### Auth Functions

**Source:** `django/contrib/auth/__init__.py`

```python
from django.contrib.auth import authenticate, login, logout

authenticate(request=None, **credentials)
    # Returns User or None
async aauthenticate(request=None, **credentials)

login(request, user, backend=None)
async alogin(request, user, backend=None)

logout(request)
async alogout(request)
```

### User Models

**Source:** `django/contrib/auth/models.py`

```python
class AbstractBaseUser(models.Model):
    password = models.CharField(max_length=128)
    last_login = models.DateTimeField(blank=True, null=True)
    is_active = True
    REQUIRED_FIELDS = []

    def set_password(self, raw_password)
    def check_password(self, raw_password)
    async def acheck_password(self, raw_password)
    def set_unusable_password(self)
    def has_usable_password(self)
    def get_username(self)
    def get_session_auth_hash(self)
    @classmethod
    def get_email_field_name(cls)
    @classmethod
    def normalize_username(cls, username)
    @property
    def is_anonymous                # Always False
    @property
    def is_authenticated            # Always True

class AbstractUser(AbstractBaseUser, PermissionsMixin):
    username = models.CharField(max_length=150, unique=True)
    first_name = models.CharField(max_length=150, blank=True)
    last_name = models.CharField(max_length=150, blank=True)
    email = models.EmailField(blank=True)
    is_staff = models.BooleanField(default=False)
    is_active = models.BooleanField(default=True)
    date_joined = models.DateTimeField(default=timezone.now)

    USERNAME_FIELD = "username"
    EMAIL_FIELD = "email"
    REQUIRED_FIELDS = ["email"]

    def get_full_name(self)
    def get_short_name(self)
    def email_user(self, subject, message, from_email=None, **kwargs)

class User(AbstractUser):
    # Concrete default user model (swappable via AUTH_USER_MODEL)
    pass
```

### PermissionsMixin

```python
class PermissionsMixin(models.Model):
    is_superuser = models.BooleanField(default=False)
    groups = models.ManyToManyField(Group, blank=True)
    user_permissions = models.ManyToManyField(Permission, blank=True)

    def get_user_permissions(self, obj=None)
    def get_group_permissions(self, obj=None)
    def get_all_permissions(self, obj=None)
    def has_perm(self, perm, obj=None)
    def has_perms(self, perm_list, obj=None)
    def has_module_perms(self, app_label)
    # All have async variants (aget_*, ahas_*)
```

### Permission & Group

```python
class Permission(models.Model):
    name = models.CharField(max_length=255)
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    codename = models.CharField(max_length=100)

class Group(models.Model):
    name = models.CharField(max_length=150, unique=True)
    permissions = models.ManyToManyField(Permission, blank=True)
```

### AnonymousUser

```python
class AnonymousUser:
    id = None
    pk = None
    username = ""
    is_staff = False
    is_active = False
    is_superuser = False
    is_anonymous = True         # property, always True
    is_authenticated = False    # property, always False
```

### Auth Decorators

**Source:** `django/contrib/auth/decorators.py`

```python
login_required(function=None, redirect_field_name=REDIRECT_FIELD_NAME, login_url=None)
login_not_required(view_func)
permission_required(perm, login_url=None, raise_exception=False)
user_passes_test(test_func, login_url=None, redirect_field_name=REDIRECT_FIELD_NAME)
```

### Auth Mixins

**Source:** `django/contrib/auth/mixins.py`

```python
class AccessMixin:
    login_url = None
    permission_denied_message = ""
    raise_exception = False
    redirect_field_name = REDIRECT_FIELD_NAME

    def get_login_url(self)
    def get_permission_denied_message(self)
    def get_redirect_field_name(self)
    def handle_no_permission(self)

class LoginRequiredMixin(AccessMixin):
    def dispatch(self, request, *args, **kwargs)

class PermissionRequiredMixin(AccessMixin):
    permission_required = None
    def get_permission_required(self)
    def has_permission(self)

class UserPassesTestMixin(AccessMixin):
    def test_func(self)
    def get_test_func(self)
```

### Password Utilities

**Source:** `django/contrib/auth/hashers.py`

```python
make_password(password, salt=None, hasher="default")
check_password(password, encoded, setter=None, preferred="default")
async acheck_password(password, encoded, setter=None, preferred="default")
is_password_usable(encoded)
verify_password(password, encoded, preferred="default")
```

### Auth Forms

**Source:** `django/contrib/auth/forms.py`

```python
class AuthenticationForm(forms.Form):
    def __init__(self, request=None, *args, **kwargs)
    def confirm_login_allowed(self, user)
    def get_user(self)

class UserCreationForm(BaseUserCreationForm):
    # password1, password2 fields

class UserChangeForm(forms.ModelForm):
    password = ReadOnlyPasswordHashField()

class SetPasswordForm(forms.Form):
    def __init__(self, user, *args, **kwargs)
    def save(self, commit=True)

class PasswordChangeForm(SetPasswordForm):
    old_password = forms.CharField()
    def clean_old_password(self)

class PasswordResetForm(forms.Form):
    email = forms.EmailField(max_length=254)
    def send_mail(self, subject_template_name, email_template_name, context,
                  from_email, to_email, html_email_template_name=None)
    def get_users(self, email)
    def save(self, domain_override=None, ...)

class AdminPasswordChangeForm(forms.Form):
    def __init__(self, user, *args, **kwargs)
    def save(self, commit=True)
```

---

## 15. Middleware

**Source:** `django/middleware/`

All middleware inherit from `MiddlewareMixin`:

```python
class MiddlewareMixin:
    def __init__(self, get_response)
    def __call__(self, request)
    # Hook methods (override as needed):
    def process_request(self, request)
    def process_view(self, request, view_func, view_args, view_kwargs)
    def process_exception(self, request, exception)
    def process_template_response(self, request, response)
    def process_response(self, request, response)
```

### Built-in Middleware

| Middleware | Module | Purpose |
|---|---|---|
| `SecurityMiddleware` | `django.middleware.security` | HTTPS redirects, HSTS, referrer policy |
| `CsrfViewMiddleware` | `django.middleware.csrf` | CSRF protection |
| `CommonMiddleware` | `django.middleware.common` | URL rewriting, slash appending |
| `GZipMiddleware` | `django.middleware.gzip` | GZip compression |
| `ConditionalGetMiddleware` | `django.middleware.http` | ETag/Last-Modified support |
| `LocaleMiddleware` | `django.middleware.locale` | Language detection |
| `XFrameOptionsMiddleware` | `django.middleware.clickjacking` | Clickjacking protection |
| `ContentSecurityPolicyMiddleware` | `django.middleware.csp` | CSP headers |
| `CacheMiddleware` | `django.middleware.cache` | Full-site caching |
| `UpdateCacheMiddleware` | `django.middleware.cache` | Cache update (response phase) |
| `FetchFromCacheMiddleware` | `django.middleware.cache` | Cache fetch (request phase) |
| `AuthenticationMiddleware` | `django.contrib.auth.middleware` | Attach user to request |
| `SessionMiddleware` | `django.contrib.sessions.middleware` | Session support |
| `MessageMiddleware` | `django.contrib.messages.middleware` | Messages framework |

### CacheMiddleware

```python
class CacheMiddleware(UpdateCacheMiddleware, FetchFromCacheMiddleware):
    def __init__(self, get_response, cache_timeout=None, cache_alias=None, cache_key_prefix=None)
```

---

## 16. Templates

### Template Loading

**Source:** `django/template/loader.py`

```python
get_template(template_name, using=None)
    # Returns Template object, raises TemplateDoesNotExist

select_template(template_name_list, using=None)
    # Returns first matching Template

render_to_string(template_name, context=None, request=None, using=None)
    # Returns rendered string
```

### TemplateResponse

**Source:** `django/template/response.py`

```python
class SimpleTemplateResponse(HttpResponse):
    def __init__(self, template, context=None, content_type=None,
                 status=None, charset=None, using=None, headers=None)
    rendered_content        # property
    def render(self)
    def add_post_render_callback(self, callback)
    is_rendered             # property

class TemplateResponse(SimpleTemplateResponse):
    def __init__(self, request, template, context=None, content_type=None,
                 status=None, charset=None, using=None, headers=None)
```

---

## 17. Email

**Source:** `django/core/mail/__init__.py`

```python
from django.core.mail import send_mail, send_mass_mail, EmailMessage, EmailMultiAlternatives

send_mail(
    subject,
    message,
    from_email,
    recipient_list,
    *,
    fail_silently=False,
    auth_user=None,
    auth_password=None,
    connection=None,
    html_message=None,
)

send_mass_mail(datatuple, *, fail_silently=False, auth_user=None,
               auth_password=None, connection=None)
    # datatuple: sequence of (subject, message, from_email, recipient_list)

mail_admins(subject, message, *, fail_silently=False, connection=None, html_message=None)
mail_managers(subject, message, *, fail_silently=False, connection=None, html_message=None)

get_connection(backend=None, *, fail_silently=False, **kwds)
```

### EmailMessage

```python
class EmailMessage:
    # Construct and send email messages

class EmailMultiAlternatives(EmailMessage):
    # Email with alternative content types (e.g., HTML)
```

---

## 18. Caching

**Source:** `django/core/cache/__init__.py`

```python
from django.core.cache import cache, caches

DEFAULT_CACHE_ALIAS = "default"

# cache object methods:
cache.get(key, default=None)
cache.set(key, value, timeout=DEFAULT_TIMEOUT, version=None)
cache.delete(key, version=None)
cache.clear()
cache.get_many(keys, version=None)
cache.set_many(data, timeout=DEFAULT_TIMEOUT, version=None)
cache.delete_many(keys, version=None)
cache.has_key(key, version=None)
cache.incr(key, delta=1, version=None)
cache.decr(key, delta=1, version=None)
cache.touch(key, timeout=DEFAULT_TIMEOUT, version=None)

# Access other cache backends:
caches["other_cache"].get(...)
```

---

## 19. Signals

### Signal Class

**Source:** `django/dispatch/dispatcher.py`

```python
class Signal:
    def __init__(self, use_caching=False)
    def connect(self, receiver, sender=None, weak=True, dispatch_uid=None)
    def disconnect(self, receiver=None, sender=None, dispatch_uid=None)
    def send(self, sender, **kwargs)
    def send_robust(self, sender, **kwargs)
```

### Model Signals

**Source:** `django/db/models/signals.py`

| Signal | Sent When | Key kwargs |
|---|---|---|
| `pre_init` | Before `__init__` | `args, kwargs` |
| `post_init` | After `__init__` | `instance` |
| `pre_save` | Before `save()` | `instance, raw, using, update_fields` |
| `post_save` | After `save()` | `instance, created, raw, using, update_fields` |
| `pre_delete` | Before `delete()` | `instance, using` |
| `post_delete` | After `delete()` | `instance, using` |
| `m2m_changed` | M2M relationship changes | `instance, action, reverse, model, pk_set, using` |
| `class_prepared` | Model class finalized | (none) |
| `pre_migrate` | Before migration | `app_config, verbosity, interactive, using, plan, apps` |
| `post_migrate` | After migration | `app_config, verbosity, interactive, using, plan, apps` |

### Core Signals

**Source:** `django/core/signals.py`

| Signal | Sent When |
|---|---|
| `request_started` | HTTP request begins (`environ` kwarg) |
| `request_finished` | HTTP request ends |
| `got_request_exception` | Unhandled exception (`request` kwarg) |
| `setting_changed` | Setting changed via `override_settings` (`setting, value` kwargs) |

### Usage Example

```python
from django.db.models.signals import post_save
from django.dispatch import receiver

@receiver(post_save, sender=MyModel)
def my_handler(sender, instance, created, **kwargs):
    if created:
        # Handle new object creation
        ...
```

---

## 20. Pagination

**Source:** `django/core/paginator.py`

```python
class Paginator:
    def __init__(self, object_list, per_page, orphans=0, allow_empty_first_page=True,
                 error_messages=None)

    count               # property: total number of objects
    num_pages           # property: total number of pages
    page_range          # property: range of page numbers (1-indexed)

    def page(self, number)          # Returns Page, may raise InvalidPage
    def get_page(self, number)      # Returns Page, handles errors gracefully
    def validate_number(self, number)

class Page:
    def __init__(self, object_list, number, paginator)

    object_list         # Items on this page
    number              # Page number (1-indexed)
    paginator           # Parent Paginator

    def has_next(self)
    def has_previous(self)
    def has_other_pages(self)
    def next_page_number(self)
    def previous_page_number(self)
    def start_index(self)           # 1-based index of first item
    def end_index(self)             # 1-based index of last item

# Exceptions
class InvalidPage(Exception): ...
class PageNotAnInteger(InvalidPage): ...
class EmptyPage(InvalidPage): ...
```

---

## 21. Serialization

**Source:** `django/core/serializers/__init__.py`

```python
from django.core import serializers

serializers.serialize(format, queryset, **options)
    # format: "json", "xml", "yaml", "python", "jsonl"
    # Returns: serialized string

serializers.deserialize(format, stream_or_string, **options)
    # Returns: iterator of DeserializedObject

serializers.get_serializer(format)
serializers.get_deserializer(format)
serializers.register_serializer(format, serializer_module)
serializers.get_serializer_formats()
serializers.get_public_serializer_formats()
serializers.sort_dependencies(app_list, allow_cycles=False)
```

---

## 22. Management Commands

**Source:** `django/core/management/base.py`

```python
class BaseCommand:
    help = ""                           # Short description
    output_transaction = False          # Wrap output in BEGIN/COMMIT
    requires_migrations_checks = False
    requires_system_checks = "__all__"  # List of check tags or "__all__"

    def __init__(self, stdout=None, stderr=None, no_color=False, force_color=False)
    def create_parser(self, prog_name, subcommand)
    def add_arguments(self, parser)     # Override to add arguments
    def handle(self, **options)         # Override: main command logic
    def execute(self, **options)
    def run_from_argv(self, argv)
    def check(self, app_configs=None, tags=None)
    def get_version(self)

class CommandError(Exception):
    def __init__(self, *args, returncode=1, **kwargs)

class SystemCheckError(CommandError): ...
```

---

## 23. Testing

### Test Cases

**Source:** `django/test/testcases.py`

```python
class SimpleTestCase(unittest.TestCase):
    # No database access
    # Assertions:
    def assertRedirects(self, response, expected_url, ...)
    def assertTemplateUsed(self, response, template_name)
    def assertTemplateNotUsed(self, response, template_name)
    def assertURLEqual(self, url1, url2)
    def assertJSONEqual(self, raw, expected_data)
    def assertXMLEqual(self, xml1, xml2)
    def assertContains(self, response, text, count=None, status_code=200, html=False)
    def assertNotContains(self, response, text, status_code=200, html=False)
    def assertFormError(self, form, field, errors, msg_prefix="")
    def assertHTMLEqual(self, html1, html2)
    def assertHTMLNotEqual(self, html1, html2)
    def assertInHTML(self, needle, haystack, count=None)

class TransactionTestCase(SimpleTestCase):
    # Database access with table truncation between tests
    fixtures = None
    databases = {"default"}
    serialized_rollback = False

class TestCase(TransactionTestCase):
    # Database access with transaction rollback (faster)

class LiveServerTestCase(TransactionTestCase):
    # Launches a live Django server for integration tests
    live_server_url     # class attribute: URL of the live server
```

### Test Client

**Source:** `django/test/client.py`

```python
class Client:
    def get(self, path, data=None, follow=False, **extra)
    def post(self, path, data=None, content_type=MULTIPART_CONTENT, follow=False, **extra)
    def put(self, path, data=None, content_type=..., follow=False, **extra)
    def patch(self, path, data=None, content_type=..., follow=False, **extra)
    def delete(self, path, data=None, content_type=..., follow=False, **extra)
    def head(self, path, data=None, follow=False, **extra)
    def options(self, path, data=None, content_type=..., follow=False, **extra)
    def trace(self, path, data=None, follow=False, **extra)
    def login(**credentials)                # Returns bool
    def force_login(user, backend=None)
    def logout()

    # Response object includes:
    #   .status_code, .content, .text, .json(), .context, .templates, .redirect_chain

class AsyncClient:
    # Async versions of all Client methods

class RequestFactory:
    # Creates HttpRequest objects (not responses)
    def get(self, path, data=None, **extra)
    def post(self, path, data=None, content_type=..., **extra)
    # ... same HTTP methods as Client

class AsyncRequestFactory:
    # Async version of RequestFactory
```

### Test Decorators

```python
from django.test import skipIfDBFeature, skipUnlessDBFeature

@skipIfDBFeature('feature_name')
@skipUnlessDBFeature('feature_name')
```

---

## 24. Settings Reference

**Source:** `django/conf/global_settings.py`

### Core Settings

| Setting | Default | Description |
|---|---|---|
| `DEBUG` | `False` | Debug mode |
| `SECRET_KEY` | `""` | Cryptographic signing key |
| `SECRET_KEY_FALLBACKS` | `[]` | Previous secret keys for rotation |
| `ALLOWED_HOSTS` | `[]` | Allowed host/domain names |
| `INTERNAL_IPS` | `[]` | IPs that see debug info |
| `ROOT_URLCONF` | — | Root URL configuration module |
| `WSGI_APPLICATION` | `None` | WSGI application path |
| `DEFAULT_AUTO_FIELD` | `"django.db.models.BigAutoField"` | Default primary key field type |
| `INSTALLED_APPS` | `[]` | List of enabled applications |
| `MIDDLEWARE` | `[]` | Middleware stack |

### Database Settings

| Setting | Default | Description |
|---|---|---|
| `DATABASES` | `{}` | Database configuration dict |
| `DATABASE_ROUTERS` | `[]` | Database router classes |

### URL & Request Settings

| Setting | Default | Description |
|---|---|---|
| `APPEND_SLASH` | `True` | Append trailing slash to URLs |
| `PREPEND_WWW` | `False` | Prepend "www." to URLs |
| `FORCE_SCRIPT_NAME` | `None` | Override SCRIPT_NAME |
| `DISALLOWED_USER_AGENTS` | `[]` | Blocked user agents |

### Template Settings

| Setting | Default | Description |
|---|---|---|
| `TEMPLATES` | `[]` | Template engine configuration |
| `FORM_RENDERER` | `"django.forms.renderers.DjangoTemplates"` | Form rendering engine |

### Security Settings

| Setting | Default | Description |
|---|---|---|
| `SECURE_PROXY_SSL_HEADER` | `None` | Tuple for SSL proxy detection |
| `SECURE_SSL_REDIRECT` | `False` | Redirect HTTP to HTTPS |
| `SECURE_SSL_HOST` | `None` | HTTPS redirect host |
| `SECURE_HSTS_SECONDS` | `0` | HSTS max-age |
| `SECURE_HSTS_INCLUDE_SUBDOMAINS` | `False` | HSTS include subdomains |
| `SECURE_HSTS_PRELOAD` | `False` | HSTS preload |
| `SECURE_CONTENT_TYPE_NOSNIFF` | `True` | X-Content-Type-Options |
| `SECURE_CROSS_ORIGIN_OPENER_POLICY` | `"same-origin"` | COOP header |
| `SECURE_REFERRER_POLICY` | `"same-origin"` | Referrer-Policy header |
| `SECURE_REDIRECT_EXEMPT` | `[]` | URL patterns exempt from SSL redirect |
| `X_FRAME_OPTIONS` | `"DENY"` | X-Frame-Options header |
| `SECURE_CSP` | `{}` | Content Security Policy |
| `SECURE_CSP_REPORT_ONLY` | `{}` | CSP report-only mode |

### CSRF Settings

| Setting | Default | Description |
|---|---|---|
| `CSRF_COOKIE_NAME` | `"csrftoken"` | CSRF cookie name |
| `CSRF_COOKIE_AGE` | `31449600` (1 year) | CSRF cookie age |
| `CSRF_COOKIE_DOMAIN` | `None` | CSRF cookie domain |
| `CSRF_COOKIE_PATH` | `"/"` | CSRF cookie path |
| `CSRF_COOKIE_SECURE` | `False` | HTTPS-only CSRF cookie |
| `CSRF_COOKIE_HTTPONLY` | `False` | HTTPOnly CSRF cookie |
| `CSRF_COOKIE_SAMESITE` | `"Lax"` | SameSite attribute |
| `CSRF_HEADER_NAME` | `"HTTP_X_CSRFTOKEN"` | CSRF header name |
| `CSRF_TRUSTED_ORIGINS` | `[]` | Trusted origins |
| `CSRF_USE_SESSIONS` | `False` | Store CSRF in session |
| `CSRF_FAILURE_VIEW` | `"django.views.csrf.csrf_failure"` | CSRF failure view |

### Session Settings

| Setting | Default | Description |
|---|---|---|
| `SESSION_ENGINE` | `"django.contrib.sessions.backends.db"` | Session backend |
| `SESSION_COOKIE_NAME` | `"sessionid"` | Session cookie name |
| `SESSION_COOKIE_AGE` | `1209600` (2 weeks) | Session cookie age |
| `SESSION_COOKIE_DOMAIN` | `None` | Session cookie domain |
| `SESSION_COOKIE_SECURE` | `False` | HTTPS-only session cookie |
| `SESSION_COOKIE_PATH` | `"/"` | Session cookie path |
| `SESSION_COOKIE_HTTPONLY` | `True` | HTTPOnly session cookie |
| `SESSION_COOKIE_SAMESITE` | `"Lax"` | SameSite attribute |
| `SESSION_SAVE_EVERY_REQUEST` | `False` | Save session on every request |
| `SESSION_EXPIRE_AT_BROWSER_CLOSE` | `False` | Session expires on close |
| `SESSION_FILE_PATH` | `None` | File session storage path |
| `SESSION_SERIALIZER` | `"...JSONSerializer"` | Session data serializer |
| `SESSION_CACHE_ALIAS` | `"default"` | Cache alias for sessions |

### Auth Settings

| Setting | Default | Description |
|---|---|---|
| `AUTH_USER_MODEL` | `"auth.User"` | Custom user model |
| `AUTHENTICATION_BACKENDS` | `["...ModelBackend"]` | Auth backends |
| `LOGIN_URL` | `"/accounts/login/"` | Login page URL |
| `LOGIN_REDIRECT_URL` | `"/accounts/profile/"` | Post-login redirect |
| `LOGOUT_REDIRECT_URL` | `None` | Post-logout redirect |
| `PASSWORD_RESET_TIMEOUT` | `259200` (3 days) | Password reset link timeout |
| `PASSWORD_HASHERS` | (list) | Password hashing algorithms |
| `AUTH_PASSWORD_VALIDATORS` | `[]` | Password validation rules |

### Email Settings

| Setting | Default | Description |
|---|---|---|
| `EMAIL_BACKEND` | `"...smtp.EmailBackend"` | Email sending backend |
| `EMAIL_HOST` | `"localhost"` | SMTP server |
| `EMAIL_PORT` | `25` | SMTP port |
| `EMAIL_HOST_USER` | `""` | SMTP username |
| `EMAIL_HOST_PASSWORD` | `""` | SMTP password |
| `EMAIL_USE_TLS` | `False` | Use TLS |
| `EMAIL_USE_SSL` | `False` | Use SSL |
| `EMAIL_TIMEOUT` | `None` | Connection timeout |
| `EMAIL_SSL_CERTFILE` | `None` | SSL certificate file |
| `EMAIL_SSL_KEYFILE` | `None` | SSL key file |
| `DEFAULT_FROM_EMAIL` | `"webmaster@localhost"` | Default "From" address |
| `SERVER_EMAIL` | `"root@localhost"` | Error notification sender |
| `EMAIL_SUBJECT_PREFIX` | `"[Django] "` | Subject line prefix |

### Cache Settings

| Setting | Default | Description |
|---|---|---|
| `CACHES` | `{"default": {...locmem...}}` | Cache backend configuration |
| `CACHE_MIDDLEWARE_KEY_PREFIX` | `""` | Cache key prefix |
| `CACHE_MIDDLEWARE_SECONDS` | `600` | Cache timeout |
| `CACHE_MIDDLEWARE_ALIAS` | `"default"` | Cache alias |

### Static & Media Files

| Setting | Default | Description |
|---|---|---|
| `STATIC_ROOT` | `None` | Collected static files directory |
| `STATIC_URL` | `None` | Static files URL prefix |
| `STATICFILES_DIRS` | `[]` | Additional static file directories |
| `STATICFILES_FINDERS` | (list) | Static file finder backends |
| `MEDIA_ROOT` | `""` | User-uploaded files directory |
| `MEDIA_URL` | `""` | Media files URL prefix |
| `STORAGES` | `{"default": ..., "staticfiles": ...}` | File storage backends |

### File Upload Settings

| Setting | Default | Description |
|---|---|---|
| `FILE_UPLOAD_HANDLERS` | (list) | Upload handler classes |
| `FILE_UPLOAD_MAX_MEMORY_SIZE` | `2621440` (2.5 MB) | Max in-memory upload size |
| `DATA_UPLOAD_MAX_MEMORY_SIZE` | `2621440` (2.5 MB) | Max request body size |
| `DATA_UPLOAD_MAX_NUMBER_FIELDS` | `1000` | Max GET/POST parameters |
| `DATA_UPLOAD_MAX_NUMBER_FILES` | `100` | Max uploaded files |
| `FILE_UPLOAD_TEMP_DIR` | `None` | Temporary upload directory |
| `FILE_UPLOAD_PERMISSIONS` | `0o644` | Uploaded file permissions |

### I18n Settings

| Setting | Default | Description |
|---|---|---|
| `LANGUAGE_CODE` | `"en-us"` | Default language |
| `USE_I18N` | `True` | Enable internationalization |
| `USE_TZ` | `True` | Enable timezone support |
| `TIME_ZONE` | `"America/Chicago"` | Default timezone |
| `LOCALE_PATHS` | `[]` | Additional locale directories |
| `LANGUAGE_COOKIE_NAME` | `"django_language"` | Language cookie name |

### Date/Number Formatting

| Setting | Default | Description |
|---|---|---|
| `DATE_FORMAT` | `"N j, Y"` | Default date format |
| `DATETIME_FORMAT` | `"N j, Y, P"` | Default datetime format |
| `TIME_FORMAT` | `"P"` | Default time format |
| `SHORT_DATE_FORMAT` | `"m/d/Y"` | Short date format |
| `SHORT_DATETIME_FORMAT` | `"m/d/Y P"` | Short datetime format |
| `FIRST_DAY_OF_WEEK` | `0` (Sunday) | First day of week |
| `DECIMAL_SEPARATOR` | `"."` | Decimal separator |
| `THOUSAND_SEPARATOR` | `","` | Thousands separator |
| `USE_THOUSAND_SEPARATOR` | `False` | Use thousands separator |
| `NUMBER_GROUPING` | `0` | Digit grouping |
| `DEFAULT_CHARSET` | `"utf-8"` | Default charset |

### Logging Settings

| Setting | Default | Description |
|---|---|---|
| `LOGGING_CONFIG` | `"logging.config.dictConfig"` | Logging config callable |
| `LOGGING` | `{}` | Logging configuration dict |

### Testing Settings

| Setting | Default | Description |
|---|---|---|
| `TEST_RUNNER` | `"django.test.runner.DiscoverRunner"` | Test runner class |
| `TEST_NON_SERIALIZED_APPS` | `[]` | Apps to skip serialization in tests |

---

## 25. Exceptions

**Source:** `django/core/exceptions.py`

| Exception | Description |
|---|---|
| `ObjectDoesNotExist` | Requested object doesn't exist |
| `ObjectNotUpdated` | Updated object no longer exists |
| `MultipleObjectsReturned` | Query returned multiple when one expected |
| `ValidationError(message, code=None, params=None)` | Data validation error |
| `PermissionDenied` | User lacks permission |
| `ViewDoesNotExist` | Requested view doesn't exist |
| `MiddlewareNotUsed` | Middleware not used |
| `ImproperlyConfigured` | Configuration error |
| `FieldError` | Model field problem |
| `FieldDoesNotExist` | Requested model field doesn't exist |
| `FieldFetchBlocked` | On-demand field fetch blocked |
| `SuspiciousOperation` | Suspicious user activity |
| `SuspiciousMultipartForm` | Suspicious MIME in multipart |
| `SuspiciousFileOperation` | Suspicious filesystem operation |
| `DisallowedHost` | Invalid HTTP_HOST header |
| `DisallowedRedirect` | Invalid redirect |
| `TooManyFieldsSent` | Too many GET/POST fields |
| `TooManyFilesSent` | Too many uploaded files |
| `RequestDataTooBig` | Request body too large |
| `RequestAborted` | Request closed/timed out |
| `BadRequest` | Malformed request |
| `AppRegistryNotReady` | Apps not loaded yet |
| `EmptyResultSet` | Query predicate is impossible |
| `FullResultSet` | Query predicate matches everything |
| `SynchronousOnlyOperation` | Sync-only call from async context |

**Special:** `NON_FIELD_ERRORS = "__all__"` — key for form-wide validation errors.

---

## 26. Utilities

### Timezone

**Source:** `django/utils/timezone.py`

```python
from django.utils import timezone

timezone.now()                              # Current datetime (aware if USE_TZ)
timezone.localtime(value=None, timezone=None)  # Convert to local time
timezone.localdate(value=None, timezone=None)  # Convert to local date
timezone.is_aware(value)                    # True if datetime has tzinfo
timezone.is_naive(value)                    # True if datetime has no tzinfo
timezone.make_aware(value, timezone=None)   # Make naive datetime aware
timezone.make_naive(value, timezone=None)   # Make aware datetime naive
timezone.get_default_timezone()             # Return default timezone
timezone.get_current_timezone()             # Return active timezone
timezone.activate(timezone)                 # Set timezone for current thread
timezone.deactivate()                       # Unset timezone
timezone.override(timezone)                 # Context manager/decorator
timezone.get_fixed_timezone(offset)         # Fixed offset timezone
```

### Text

**Source:** `django/utils/text.py`

```python
from django.utils.text import slugify, Truncator, capfirst, wrap

slugify(value, allow_unicode=False)         # URL-friendly slug
capfirst(x)                                 # Capitalize first letter

class Truncator(SimpleLazyObject):
    def chars(self, num, truncate=None, html=False)
    def words(self, num, truncate=None, html=False)

wrap(text, width)                           # Word-wrap preserving line breaks
```

### Translation

**Source:** `django/utils/translation/__init__.py`

```python
from django.utils.translation import gettext, gettext_lazy, ngettext, pgettext

gettext(message)                            # Translate message
gettext_lazy(message)                       # Lazy translation
gettext_noop(message)                       # Mark for translation without translating
ngettext(singular, plural, number)          # Plural-aware translation
ngettext_lazy(singular, plural, number=None)
pgettext(context, message)                  # Context-aware translation
pgettext_lazy(context, message)
npgettext(context, singular, plural, number)
npgettext_lazy(context, singular, plural, number=None)

activate(language)                          # Activate language
deactivate()                                # Deactivate translation
deactivate_all()                            # Deactivate all translation
get_language()                              # Get active language code
get_language_info(lang_code)                # Get language info dict
get_language_bidi(language_code)            # Check if RTL language
check_for_language(lang_code)               # Check language availability
to_language(locale_name)                    # Locale name → language code
to_locale(language_code)                    # Language code → locale name
override(language, deactivate=False)        # Context manager/decorator
```

### Validators

**Source:** `django/core/validators.py`

```python
from django.core.validators import (
    RegexValidator, URLValidator, EmailValidator,
    MaxValueValidator, MinValueValidator,
    MaxLengthValidator, MinLengthValidator,
    DecimalValidator, FileExtensionValidator,
    validate_email, validate_slug, validate_unicode_slug,
    validate_ipv4_address, validate_ipv6_address,
    validate_ipv46_address, validate_domain_name,
)

class RegexValidator:
    def __init__(self, regex=None, message=None, code=None,
                 inverse_match=None, flags=None)

class URLValidator(RegexValidator): ...
class EmailValidator: ...
class MaxValueValidator:
    def __init__(self, limit_value, message=None)
class MinValueValidator:
    def __init__(self, limit_value, message=None)
class MaxLengthValidator:
    def __init__(self, limit_value, message=None)
class MinLengthValidator:
    def __init__(self, limit_value, message=None)
class DecimalValidator:
    def __init__(self, max_digits, decimal_places)
class FileExtensionValidator:
    def __init__(self, allowed_extensions=None, message=None, code=None)
```

### Files

**Source:** `django/core/files/`

```python
from django.core.files.base import File, ContentFile

class File:
    def __init__(self, file, name="")

class ContentFile(File):
    def __init__(self, content, name="")

class UploadedFile(File):
    # Represents a user-uploaded file
```

---

## 27. View Decorators

### Cache Decorators

**Source:** `django/views/decorators/cache.py`

```python
cache_page(timeout, *, cache=None, key_prefix=None)
cache_control(**kwargs)         # Set Cache-Control header
never_cache(view_func)          # Add Cache-Control: max-age=0 headers
```

### CSRF Decorators

**Source:** `django/views/decorators/csrf.py`

```python
csrf_protect(view_func)         # Enforce CSRF for this view
csrf_exempt(view_func)          # Exempt this view from CSRF
requires_csrf_token(view_func)  # Ensure CSRF token is available
ensure_csrf_cookie(view_func)   # Ensure CSRF cookie is set
```

### HTTP Method Decorators

**Source:** `django/views/decorators/http.py`

```python
require_http_methods(request_method_list)   # Restrict to specific methods
require_GET                                  # Allow only GET
require_POST                                 # Allow only POST
require_safe                                 # Allow only GET and HEAD

condition(etag_func=None, last_modified_func=None)  # Conditional response
etag(etag_func)
last_modified(last_modified_func)
```

### Vary Decorators

**Source:** `django/views/decorators/vary.py`

```python
vary_on_headers(*headers)       # Set Vary header
vary_on_cookie                  # Set Vary: Cookie
```

### Other Decorators

```python
from django.views.decorators.common import no_append_slash
no_append_slash(view_func)      # Don't append slash to this view's URLs
```

---

## Quick Import Reference

```python
# Models
from django.db import models
from django.db.models import Q, F, Value, Case, When, Exists, Subquery, OuterRef
from django.db.models import Avg, Count, Max, Min, Sum, StdDev, Variance
from django.db.models import Prefetch
from django.db.models import UniqueConstraint, CheckConstraint, Index

# Views
from django.views import View
from django.views.generic import ListView, DetailView, CreateView, UpdateView, DeleteView
from django.views.generic import TemplateView, RedirectView, FormView

# URLs
from django.urls import path, re_path, include, reverse, reverse_lazy

# HTTP
from django.http import HttpRequest, HttpResponse, JsonResponse, FileResponse
from django.http import HttpResponseRedirect, HttpResponsePermanentRedirect
from django.http import Http404, HttpResponseNotFound, HttpResponseForbidden
from django.http import StreamingHttpResponse

# Shortcuts
from django.shortcuts import render, redirect, get_object_or_404, get_list_or_404

# Forms
from django.forms import Form, ModelForm, formset_factory, modelformset_factory
from django.forms import inlineformset_factory, modelform_factory
from django.forms import CharField, IntegerField, BooleanField, ChoiceField, ...
from django.forms import TextInput, Select, CheckboxInput, RadioSelect, ...

# Admin
from django.contrib import admin
from django.contrib.admin import ModelAdmin, TabularInline, StackedInline

# Auth
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.models import User, Group, Permission
from django.contrib.auth.decorators import login_required, permission_required
from django.contrib.auth.mixins import LoginRequiredMixin, PermissionRequiredMixin

# Templates
from django.template.loader import render_to_string, get_template
from django.template.response import TemplateResponse

# Email
from django.core.mail import send_mail, EmailMessage, EmailMultiAlternatives

# Cache
from django.core.cache import cache, caches

# Signals
from django.db.models.signals import pre_save, post_save, pre_delete, post_delete
from django.dispatch import receiver, Signal

# Utils
from django.utils import timezone
from django.utils.text import slugify
from django.utils.translation import gettext_lazy as _

# Testing
from django.test import TestCase, SimpleTestCase, TransactionTestCase, LiveServerTestCase
from django.test import Client, RequestFactory

# Exceptions
from django.core.exceptions import (
    ValidationError, ObjectDoesNotExist, PermissionDenied,
    ImproperlyConfigured, FieldError,
)

# Pagination
from django.core.paginator import Paginator

# Settings
from django.conf import settings

# Validators
from django.core.validators import RegexValidator, URLValidator, EmailValidator
```
