# listmonk — Comprehensive API Reference

> **Version:** 0.4.3
> **Author:** Michael Kennedy (michael@talkpython.fm)
> **License:** MIT
> **Repository:** https://github.com/mikeckennedy/listmonk
> **Documentation:** https://mkennedy.codes/docs/listmonk/
> **Python:** 3.10 - 3.15
> **Dependencies:** `httpx2>=2.0`, `pydantic`, `strenum`

## Overview

`listmonk` is a Python client for the open-source, self-hosted [Listmonk](https://listmonk.app) email platform. It wraps the subset of the Listmonk REST API that most SaaS apps need — subscriber management, mailing lists, campaigns, templates, media uploads, and transactional email — behind a flat, function-based module with Pydantic models for every request and response. It is built on [httpx2](https://github.com/pydantic/httpx2) (a fork of httpx with a near-identical API) and is fully type-annotated (ships `py.typed`). The client is synchronous only; async is planned but not yet implemented.

## Agent & documentation resources

The project publishes machine-readable docs at [mkennedy.codes/docs/listmonk](https://mkennedy.codes/docs/listmonk/). If you can fetch URLs, these are authoritative and stay in sync with the released package:

- **Full docs site:** https://mkennedy.codes/docs/listmonk/
- **llms.txt** (indexed API map): https://mkennedy.codes/docs/listmonk/llms.txt
- **llms-full.txt** (full text for LLM ingestion): https://mkennedy.codes/docs/listmonk/llms-full.txt
- **Agent skill (Markdown):** https://mkennedy.codes/docs/listmonk/SKILL.md — also discoverable at `/.well-known/agent-skills/listmonk/SKILL.md`
- **Skills page (install commands):** https://mkennedy.codes/docs/listmonk/skills.html

Every documentation page also has a plain-Markdown twin — swap the `.html` extension for `.md` to get token-efficient source without the site chrome. For example https://mkennedy.codes/docs/listmonk/reference/subscribers.html → https://mkennedy.codes/docs/listmonk/reference/subscribers.md

## Table of Contents

- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Authentication](#authentication)
- [Health Check](#health-check)
- [Subscribers](#subscribers)
- [Lists](#lists)
- [Campaigns](#campaigns)
- [Media](#media)
- [Templates](#templates)
- [Transactional Email](#transactional-email)
- [Models Reference](#models-reference)
- [Errors](#errors)
- [Error-Handling Patterns](#error-handling-patterns)
- [API Endpoints](#api-endpoints)
- [FAQ / Troubleshooting](#faq--troubleshooting)
- [Complete Public API](#complete-public-api)

---

## Installation

```bash
pip install listmonk
```

Dependencies (installed automatically): `httpx2>=2.0`, `pydantic`, `strenum`.

> **Note on `httpx2`:** the client depends on `httpx2`, a fork of `httpx` with a near-identical public API. Timeout configuration uses `httpx2.Timeout`, and HTTP errors surface as `httpx2.HTTPStatusError`. Import `httpx2` (not `httpx`) when constructing a custom timeout.

---

## Quick Start

```python
import listmonk

# 1. Point at your Listmonk instance (scheme required, no /api path)
listmonk.set_url_base('https://listmonk.yourdomain.com')

# 2. Authenticate (returns False if rejected/unreachable — check it)
if not listmonk.login('your_username', 'your_password'):
    raise SystemExit('Login failed: check credentials and base URL.')

# 3. Use the API
for sub in listmonk.subscribers(list_id=3):
    print(sub.email, sub.name)
```

---

## Configuration

### `set_url_base(url: str) -> None`

Set the base URL of your Listmonk instance. Must be called before any other operation.

```python
listmonk.set_url_base('https://listmonk.yourdomain.com')
```

- URL must start with `http://` or `https://`.
- A trailing slash is stripped automatically.
- Do **not** include `/api` — the library appends the API path itself.

Raises `ValidationError` if the URL is empty/whitespace or is missing the `http://`/`https://` scheme.

### `get_base_url() -> Optional[str]`

Returns the currently configured base URL (without a trailing slash), or `None` if `set_url_base()` has not been called.

```python
url = listmonk.get_base_url()
```

---

## Authentication

The library uses HTTP Basic Auth. Credentials are stored in module-level state for the lifetime of your application, so you authenticate once.

### `login(user_name: str, pw: str, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Authenticate against the instance's health endpoint and cache the credentials. `set_url_base()` must be called first.

```python
success = listmonk.login('admin', 'secretpassword')
```

- Validates credentials against the server immediately.
- Returns `True` if authentication succeeded; returns `False` if the server rejected the credentials **or could not be reached** — no HTTP error is raised in that case, so check the boolean.
- Raises `OperationNotAllowedError` if `set_url_base()` has not been called.
- Raises `ValidationError` if `user_name` or `pw` is empty.

### `verify_login(timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Verify that the stored credentials are still valid. This is a thin alias for `is_healthy()` — any error is reported as `False` rather than raised.

```python
still_valid = listmonk.verify_login()
```

---

## Health Check

### `is_healthy(timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Check whether the instance is reachable and the stored credentials are valid. This call is defensive: **any** failure (URL not set, not logged in, network error, non-2xx response, unparseable body) is caught and reported as `False`, so it never raises.

```python
if listmonk.is_healthy():
    print('Listmonk is up!')
```

---

## Subscribers

### Retrieving Subscribers

#### `subscribers(query_text: Optional[str] = None, list_id: Optional[int] = None, timeout_config: Optional[httpx2.Timeout] = None) -> list[Subscriber]`

Get subscribers matching the given criteria, or all subscribers if none are given. Results are fetched page by page (500 per page) and combined, ordered by `updated_at` descending, so a broad query may issue several HTTP requests and return a large list. Returns an empty list (never `None`) if nothing matches.

```python
# All subscribers
all_subs = listmonk.subscribers()

# Subscribers on a specific list
list_subs = listmonk.subscribers(list_id=3)

# SQL-like query filtering (Listmonk query syntax)
results = listmonk.subscribers(query_text="subscribers.email = 'user@example.com'")
results = listmonk.subscribers(query_text="subscribers.attribs->>'city' = 'Portland'")
```

See the [Listmonk querying docs](https://listmonk.app/docs/querying-and-segmentation/) for the full query syntax. Using `query_text` requires the `subscribers:sql_query` permission on the authenticated user's role, or the server returns HTTP 403.

#### `subscriber_by_email(email: str, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Subscriber]`

Look up a single subscriber by exact email match (a `+` in the address is URL-encoded automatically). Returns `None` if not found.

```python
sub = listmonk.subscriber_by_email('user@example.com')
if sub:
    print(f'{sub.name} (ID: {sub.id})')
```

#### `subscriber_by_id(subscriber_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Subscriber]`

Look up a single subscriber by numeric ID. Returns `None` if not found.

```python
sub = listmonk.subscriber_by_id(2001)
```

#### `subscriber_by_uuid(subscriber_uuid: str, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Subscriber]`

Look up a single subscriber by UUID. Returns `None` if not found.

```python
sub = listmonk.subscriber_by_uuid('f6668cf0-1c2e-...')
```

### Creating Subscribers

#### `create_subscriber(email: str, name: Optional[str] = None, list_ids: Optional[set[int]] = None, pre_confirm: bool = False, attribs: Optional[dict[str, Any]] = None, timeout_config: Optional[httpx2.Timeout] = None) -> Subscriber`

Create a new subscriber. Only `email` is required; the email is lowercased and stripped, the name is stripped, and the subscriber is always created with a global status of `enabled`.

```python
new_sub = listmonk.create_subscriber(
    email='user@example.com',
    name='Jane Doe',
    list_ids={1, 7, 9},
    pre_confirm=True,        # Mark subscriptions confirmed (skip double opt-in email)
    attribs={'city': 'Portland', 'plan': 'pro'},
)
print(f'Created with ID: {new_sub.id}')
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `email` | `str` | required | Subscriber's email address (must be non-empty after stripping) |
| `name` | `Optional[str]` | `None` | Full name; stored as an empty string when omitted |
| `list_ids` | `Optional[set[int]]` | `None` | Set of list IDs to subscribe to (no lists when omitted) |
| `pre_confirm` | `bool` | `False` | If `True`, marks new subscriptions confirmed (skips double opt-in email) |
| `attribs` | `Optional[dict[str, Any]]` | `None` | Custom attributes stored on the subscriber (queryable) |

Returns the newly created `Subscriber`, populated by the server with its `id`, `uuid`, and more. Raises `ValueError` if `email` is empty.

### Updating Subscribers

#### `update_subscriber(subscriber: Subscriber, add_to_lists: Optional[set[int]] = None, remove_from_lists: Optional[set[int]] = None, status: SubscriberStatuses = SubscriberStatuses.enabled, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Subscriber]`

Update a subscriber's email/name/attributes, list memberships, and status. Final list membership is computed as the subscriber's **existing lists minus `remove_from_lists` plus `add_to_lists`**, and the affected subscriptions are preconfirmed. After the update, the subscriber is re-fetched from the server and returned (or `None` if it can no longer be found).

```python
sub = listmonk.subscriber_by_email('user@example.com')

# Modify fields on the object
sub.email = 'newemail@example.com'
sub.name = 'Updated Name'
sub.attribs['rating'] = 7

# Save changes, add to list 4, remove from list 5
updated = listmonk.update_subscriber(sub, add_to_lists={4}, remove_from_lists={5})
```

Raises `ValueError` if `subscriber` is `None` or has no `id`. For status-only changes, prefer the dedicated `enable_subscriber` / `disable_subscriber` / `block_subscriber` wrappers.

### Subscriber Status Management

Each of these is a convenience wrapper around `update_subscriber` that changes only the status. All three take `(subscriber: Subscriber, timeout_config=None)`, return the refreshed `Optional[Subscriber]`, and raise `ValueError` if the subscriber is `None` or has no `id`.

#### `disable_subscriber(subscriber, timeout_config=None) -> Optional[Subscriber]`

Set a subscriber's status to `disabled` (paused; will not receive campaigns).

#### `enable_subscriber(subscriber, timeout_config=None) -> Optional[Subscriber]`

Set a subscriber's status back to `enabled`.

#### `block_subscriber(subscriber, timeout_config=None) -> Optional[Subscriber]`

Blocklist (unsubscribe) a subscriber. Sets status to `blocklisted` — they stay in the system but receive no mail. Use `delete_subscriber` to erase the record entirely.

```python
listmonk.block_subscriber(subscriber)
```

### Deleting Subscribers

#### `delete_subscriber(email: Optional[str] = None, overriding_subscriber_id: Optional[int] = None, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Permanently delete a subscriber. To unsubscribe instead, use `block_subscriber`. The email is normalized (lowercased/stripped) before lookup; when both arguments are given, `overriding_subscriber_id` takes precedence.

```python
# Delete by email
deleted = listmonk.delete_subscriber('user@example.com')

# Delete by ID (takes precedence over email)
deleted = listmonk.delete_subscriber(overriding_subscriber_id=2001)
```

Returns `True` on success, `False` if no subscriber matched. Raises `ValueError` if neither `email` nor `overriding_subscriber_id` is provided.

### Opt-in Confirmation

#### `confirm_optin(subscriber_uuid: str, list_uuid: str, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Confirm a subscriber's opt-in to a double opt-in list via the API. Use this only when the subscriber is genuinely opting in (for example, from an opt-in form you handle in your own code). Internally this submits the public opt-in form on the subscription endpoint rather than calling a JSON API, so HTTP errors are reported as a `False` return value rather than raised.

```python
confirmed = listmonk.confirm_optin(subscriber.uuid, mailing_list.uuid)
```

Returns `True` if the opt-in succeeded, `False` on a non-success status. Raises `ValueError` if `subscriber_uuid` or `list_uuid` is empty.

### Bulk List Management

#### `add_subscribers_to_lists(subscriber_ids: Iterable[int], list_ids: Iterable[int], timeout_config: Optional[httpx2.Timeout] = None, status: str = 'confirmed') -> bool`

Add multiple subscribers to multiple lists in a single bulk operation. IDs are de-duplicated.

```python
success = listmonk.add_subscribers_to_lists(
    subscriber_ids=[101, 102, 103],
    list_ids=[5, 6],
    status='confirmed',  # or 'unconfirmed'
)
```

Returns `True` on success. Returns `False` if either ID set is empty (or contains only `0`), or if the server responds with an error status.

---

## Lists

### `lists(timeout_config: Optional[httpx2.Timeout] = None) -> list[MailingList]`

Get all mailing lists (fetched in a single large-page request). Returns an empty list if there are none.

```python
for lst in listmonk.lists():
    print(f'{lst.name}: {lst.subscriber_count} subscribers')
```

### `list_by_id(list_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> MailingList`

Get a single list by ID. Returns a `MailingList` (not `Optional`).

```python
my_list = listmonk.list_by_id(7)
print(my_list.name, my_list.uuid)
```

Raises an `Exception` if the server returns a result set that does not contain the requested `list_id` (a workaround for a known Listmonk server quirk, [issue #2117](https://github.com/knadh/listmonk/issues/2117)), plus the usual `OperationNotAllowedError` / `ValidationError` / `httpx2.HTTPStatusError`.

### `create_list(list_name: str, list_type: str = 'public', optin: str = 'single', tags: Optional[list[str]] = None, description: Optional[str] = None, timeout_config: Optional[httpx2.Timeout] = None) -> MailingList`

Create a new mailing list. The name is stripped before submission.

```python
new_list = listmonk.create_list(
    list_name='Newsletter',
    list_type='public',     # 'public' or 'private'
    optin='single',         # 'single' or 'double'
    tags=['newsletter', 'weekly'],
    description='Our weekly newsletter',
)
```

**Parameters:**

| Parameter | Type | Default | Options |
|-----------|------|---------|---------|
| `list_name` | `str` | required | — |
| `list_type` | `str` | `'public'` | `'public'`, `'private'` |
| `optin` | `str` | `'single'` | `'single'`, `'double'` |
| `tags` | `Optional[list[str]]` | `None` | — |
| `description` | `Optional[str]` | `None` | — |

Returns the created `MailingList`. Raises `ValueError` if `list_name` is empty, or if `list_type`/`optin` is not one of its accepted values.

### `update_list(list_id: int, list_name: Optional[str] = None, list_type: Optional[str] = None, status: Optional[str] = None, optin: Optional[str] = None, tags: Optional[list[str]] = None, description: Optional[str] = None, timeout_config: Optional[httpx2.Timeout] = None) -> MailingList`

Update an existing list. Only the parameters you pass (that are not `None`) are sent, so omitted fields are left unchanged.

```python
updated = listmonk.update_list(
    list_id=7,
    list_name='Renamed List',
    status='archived',  # 'active' or 'archived'
)
```

Returns the updated `MailingList`. Raises `ValueError` if `list_id` is falsy, or if `list_type` (`'public'`/`'private'`), `status` (`'active'`/`'archived'`), or `optin` (`'single'`/`'double'`) is not an accepted value.

### `delete_list(list_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Delete a list by ID. A pre-flight existence check is performed via `list_by_id` (which raises if the list does not exist).

```python
deleted = listmonk.delete_list(7)
```

Returns `True` on success. Raises `ValueError` if `list_id` is falsy.

---

## Campaigns

### Retrieving Campaigns

#### `campaigns(timeout_config: Optional[httpx2.Timeout] = None) -> list[Campaign]`

Get all campaigns (single large-page request). Returns an empty list if there are none.

```python
for c in listmonk.campaigns():
    print(f'{c.name}: {c.status} ({c.sent}/{c.to_send} sent)')
```

#### `campaign_by_id(campaign_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Campaign]`

Get a single campaign by ID. Returns `None` if the server returns no campaign data.

```python
campaign = listmonk.campaign_by_id(15)
```

#### `campaign_preview_by_id(campaign_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> CampaignPreview`

Get the rendered HTML preview of a campaign. Returns a `CampaignPreview` (not `Optional`) whose `preview` attribute holds the HTML string.

```python
preview = listmonk.campaign_preview_by_id(15)
print(preview.preview)  # HTML string
```

### Creating Campaigns

#### `create_campaign(name: str, subject: str, list_ids: Optional[set[int]] = None, from_email: Optional[str] = None, campaign_type: Optional[str] = None, content_type: Optional[str] = None, body: Optional[str] = None, alt_body: Optional[str] = None, send_at: Optional[datetime] = None, messenger: Optional[str] = None, template_id: Optional[int] = None, tags: Optional[list[str]] = None, headers: Optional[list[dict[str, Optional[str]]]] = None, media_ids: Optional[list[int]] = None, timeout_config: Optional[httpx2.Timeout] = None) -> Campaign`

Create a new campaign. Only `name` and `subject` are required; everything else falls back to a default (notably `list_ids` defaults to `{1}` and `from_email` falls back to the instance settings when omitted).

```python
from datetime import datetime, timedelta

campaign = listmonk.create_campaign(
    name='Weekly Update',
    subject='This Week at Acme',
    body='<p>Hello {{ .Subscriber.FirstName }}!</p>',
    alt_body='Hello!',                              # Plain text fallback
    list_ids={1, 2},                                # Default: {1}
    template_id=5,                                  # Default: None
    content_type='html',                            # 'richtext', 'html', 'markdown', 'plain'
    send_at=datetime.now() + timedelta(hours=1),    # Schedule for later
    tags=['weekly', 'update'],
    from_email='newsletter@yourdomain.com',
    headers=[{'X-Custom-Header': 'value'}],         # List of single-entry dicts
    media_ids=[42],                                 # Attach uploaded media (see upload_media)
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | required | Campaign name |
| `subject` | `str` | required | Email subject line |
| `list_ids` | `Optional[set[int]]` | `{1}` | Target lists |
| `from_email` | `Optional[str]` | `None` | Sender address (uses server default if omitted) |
| `campaign_type` | `Optional[str]` | `None` | `'regular'` or `'optin'` |
| `content_type` | `Optional[str]` | `None` | `'richtext'`, `'html'`, `'markdown'`, `'plain'` |
| `body` | `Optional[str]` | `None` | Campaign body (in the configured content type) |
| `alt_body` | `Optional[str]` | `None` | Plain text alternative for multipart HTML |
| `send_at` | `Optional[datetime]` | `None` | Scheduled send time |
| `messenger` | `Optional[str]` | `None` | Delivery channel, usually `'email'` |
| `template_id` | `Optional[int]` | `None` | Template to wrap the body |
| `tags` | `Optional[list[str]]` | `None` | Tags for organization |
| `headers` | `Optional[list[dict[str, Optional[str]]]]` | `None` | Custom email headers, each a single-entry dict |
| `media_ids` | `Optional[list[int]]` | `None` | IDs from `upload_media()` to attach |

Returns the created `Campaign`. Raises `ValueError` if `name` (after stripping) or `subject` is empty.

### Updating Campaigns

#### `update_campaign(campaign: Campaign, media_ids: Optional[list[int]] = None, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Campaign]`

Update an existing campaign. Modify fields on the `Campaign` object, then pass it back; after the update, the campaign is re-fetched and returned (or `None` if it can no longer be found).

```python
campaign = listmonk.campaign_by_id(15)
campaign.name = 'Better Name'
campaign.subject = 'Updated Subject'
campaign.body = '<p>New content</p>'
campaign.lists = [3, 4]  # Change target lists (IDs are normalized before sending)

updated = listmonk.update_campaign(campaign)
```

- **Attachments:** Listmonk replaces the whole attachment set on every update. With `media_ids=None` (the default), the campaign's **existing** attachments are re-sent unchanged. Pass `media_ids=[...]` to change them, or `media_ids=[]` to remove them all.
- **Scheduling:** a `send_at` already in the past is silently dropped (set to `None`) to prevent the API rejecting a stale scheduled time.

Raises `ValueError` if `campaign` is `None` or has no `id`.

### Deleting Campaigns

#### `delete_campaign(campaign_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Delete a campaign by ID. The campaign is first looked up; if it does not exist, no delete is issued and `False` is returned.

```python
deleted = listmonk.delete_campaign(15)
```

Returns `True` on success, `False` if no campaign with the given ID was found. Raises `ValueError` if `campaign_id` is falsy.

---

## Media

### `upload_media(file: Path | bytes, filename: Optional[str] = None, timeout_config: Optional[httpx2.Timeout] = None) -> Media`

Upload a file to the Listmonk media library. The returned `Media` object's `id` can be passed to `create_campaign()` / `update_campaign()` via `media_ids` to attach the file to a campaign.

```python
from pathlib import Path

# From a path (filename defaults to the path's file name)
media = listmonk.upload_media(Path('/path/to/report.png'))

# From raw bytes (filename is REQUIRED)
media = listmonk.upload_media(png_bytes, filename='report.png')

campaign = listmonk.create_campaign(
    name='Report', subject='This month', media_ids=[media.id],
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `file` | `Path \| bytes` | required | A path to a file on disk, or the raw file bytes |
| `filename` | `Optional[str]` | `None` | Name to store under; **required** when `file` is bytes, defaults to the path name otherwise |

Returns a `Media` object describing the uploaded file (including its `id`).

**Raises:**

- `ListmonkFileNotFoundError` — if `file` is a `Path` that does not point to an existing file.
- `ValueError` — if `file` is bytes and `filename` is not provided.
- `TypeError` — if `file` is neither a `Path` nor `bytes`.

> The default Listmonk server settings only permit image file extensions in the media library, and there is no delete-media endpoint wrapped by this client.

---

## Templates

Listmonk has two template types: `'campaign'` templates and `'tx'` (transactional) templates. Every template body **must** contain the Go-template placeholder `{{ template "content" . }}` exactly where the rendered content should be injected.

### Retrieving Templates

#### `templates(timeout_config: Optional[httpx2.Timeout] = None) -> list[Template]`

Get all templates (both campaign and transactional). Returns an empty list if none are defined.

```python
for t in listmonk.templates():
    print(f'{t.name} (type: {t.type}, default: {t.is_default})')
```

#### `template_by_id(template_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Template]`

Get a single template by ID. Returns `None` if the server returns no template data.

```python
template = listmonk.template_by_id(2)
```

#### `template_preview_by_id(template_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> TemplatePreview`

Get a rendered preview of a template (using lorem-ipsum sample content). Returns a `TemplatePreview` (not `Optional`).

```python
preview = listmonk.template_preview_by_id(3)
print(preview.preview)
```

### Creating Templates

#### `create_template(name: str, body: str, type: Optional[str] = None, is_default: Optional[bool] = None, subject: Optional[str] = None, timeout_config: Optional[httpx2.Timeout] = None) -> Template`

Create a new template. `name` and `body` are required, and the body must contain `{{ template "content" . }}`.

```python
# Campaign template — must include the content placeholder
campaign_tpl = listmonk.create_template(
    name='My Campaign Template',
    body='<html><body>{{ template "content" . }}</body></html>',
    type='campaign',
)

# Transactional template
tx_tpl = listmonk.create_template(
    name='Password Reset',
    subject='Reset Your Password',
    body='<p>Hi {{ .Subscriber.FirstName }}, your code: {{ template "content" . }}</p>',
    type='tx',
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | `str` | required | Template name (must be non-empty) |
| `body` | `str` | required | Template markup; must contain `{{ template "content" . }}` |
| `type` | `Optional[str]` | `None` | `'campaign'` or `'tx'` (transactional) |
| `is_default` | `Optional[bool]` | `None` | Whether this becomes the default for its type |
| `subject` | `Optional[str]` | `None` | Default subject line (for transactional templates) |

Returns the created `Template`. Raises `ValueError` if `name` is empty, `body` is empty, or `body` does not contain the `{{ template "content" . }}` placeholder.

### Updating Templates

#### `update_template(template: Template, timeout_config: Optional[httpx2.Timeout] = None) -> Optional[Template]`

Update an existing template. The `name`, `subject`, `body`, and `type` are sent; the `is_default` flag is **not** changed by this call (use `set_default_template` for that). After the update, the template is re-fetched and returned.

```python
template = listmonk.template_by_id(2)
template.name = "Bob's Great Template"
template.body = '<html><body>{{ template "content" . }}</body></html>'

updated = listmonk.update_template(template)
```

Raises `ValueError` if `template` is `None` or has no `id`.

### Deleting Templates

#### `delete_template(template_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Delete a template by ID (looked up first; `False` if it does not exist).

```python
deleted = listmonk.delete_template(3)
```

Returns `True` on success. Raises `ValueError` if `template_id` is falsy.

### Setting the Default Template

#### `set_default_template(template_id: int, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Set a template as the default for its type (looked up first; `False` if it does not exist).

```python
listmonk.set_default_template(5)
```

Returns `True` on success. Raises `ValueError` if `template_id` is falsy.

---

## Transactional Email

### `send_transactional_email(subscriber_email: str, template_id: int, from_email: Optional[str] = None, template_data: Optional[dict[str, Any]] = None, messenger_channel: str = 'email', content_type: str = 'markdown', altbody: Optional[str] = None, attachments: Optional[list[Path]] = None, email_headers: Optional[list[dict[str, Optional[str]]]] = None, timeout_config: Optional[httpx2.Timeout] = None) -> bool`

Send a one-off transactional email (e.g. password resets, order confirmations) to a single recipient. The recipient must already be a subscriber on at least one list, and `template_id` must reference a **transactional (`tx`)** template, not a campaign template. The email is lowercased/stripped before sending.

```python
# Basic transactional email
success = listmonk.send_transactional_email(
    subscriber_email='user@example.com',
    template_id=3,                          # Must be a *transactional* template
    from_email='app@yourdomain.com',        # Optional, uses server default
    template_data={                         # Available as {{ .Tx.Data.* }} in the template
        'full_name': 'Jane Doe',
        'reset_code': 'abc123',
    },
    content_type='html',                    # 'html', 'markdown', or 'plain'
)
```

**With attachments** (each path must exist and be a file):

```python
from pathlib import Path

success = listmonk.send_transactional_email(
    subscriber_email='user@example.com',
    template_id=3,
    template_data={'order_id': 1772},
    attachments=[
        Path('/path/to/invoice.pdf'),
        Path('/path/to/receipt.png'),
    ],
    content_type='html',
)
```

**With custom headers** (a list of single-entry dicts):

```python
success = listmonk.send_transactional_email(
    subscriber_email='user@example.com',
    template_id=3,
    email_headers=[
        {'X-Custom-Header': 'value'},
        {'X-Priority': '1'},
    ],
)
```

**Parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `subscriber_email` | `str` | required | Recipient (must be an existing subscriber) |
| `template_id` | `int` | required | Transactional template ID |
| `from_email` | `Optional[str]` | `None` | Sender address (server default if omitted) |
| `template_data` | `Optional[dict[str, Any]]` | `None` | Merge data for the template (`{{ .Tx.Data.key }}`) |
| `messenger_channel` | `str` | `'email'` | Delivery channel |
| `content_type` | `str` | `'markdown'` | `'html'`, `'markdown'`, or `'plain'` |
| `altbody` | `Optional[str]` | `None` | Alternate plain-text body for multipart HTML |
| `attachments` | `Optional[list[Path]]` | `None` | File attachments (each must exist) |
| `email_headers` | `Optional[list[dict[str, Optional[str]]]]` | `None` | Custom email headers, each a single-entry dict |

Returns `True` if the server accepted the send, `False` otherwise. Raises `ValueError` if `subscriber_email` is empty, and `ListmonkFileNotFoundError` if any attachment path does not exist or is not a file.

---

## Models Reference

All models are Pydantic `BaseModel` subclasses, importable from `listmonk.models`.

```python
from listmonk.models import (
    MailingList, SubscriberStatus, SubscriberStatuses, Subscriber, CreateSubscriberModel,
    Campaign, CreateCampaignModel, UpdateCampaignModel, CampaignPreview,
    Template, CreateTemplateModel, TemplatePreview, Media,
)
```

### `SubscriberStatuses` (Enum)

A `strenum.LowercaseStrEnum` of a subscriber's global status.

```python
class SubscriberStatuses(LowercaseStrEnum):
    enabled = 'enabled'          # active; will receive campaigns
    disabled = 'disabled'        # paused; will not receive campaigns
    blocklisted = 'blocklisted'  # blocked; will not receive any mail
```

### `SubscriberStatus`

A breakdown of subscriber counts by subscription status for a mailing list.

| Field | Type | Description |
|-------|------|-------------|
| `unconfirmed_count` | `Optional[int]` | Subscriptions awaiting double opt-in (read from the API's `unconfirmed` field via alias) |

### `MailingList`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | List ID |
| `created_at` | `datetime` | Creation timestamp |
| `updated_at` | `Optional[datetime]` | Last update timestamp |
| `uuid` | `str` | List UUID |
| `name` | `Optional[str]` | List name |
| `type` | `Optional[str]` | `'public'` or `'private'` |
| `optin` | `Optional[str]` | `'single'` or `'double'` |
| `tags` | `list[str]` | Associated tags |
| `description` | `Optional[str]` | List description |
| `subscriber_count` | `Optional[int]` | Number of subscribers |
| `subscriber_statuses` | `Optional[SubscriberStatus]` | Per-status subscriber breakdown |

### `Subscriber`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | Subscriber ID |
| `email` | `str` | Email address |
| `name` | `Optional[str]` | Full name |
| `created_at` | `datetime` | Creation timestamp |
| `updated_at` | `Optional[datetime]` | Last update timestamp |
| `uuid` | `Optional[str]` | Subscriber UUID |
| `lists` | `list[dict[str, Any]]` | Lists the subscriber belongs to (each a membership dict; `model_dump()` emits plain list IDs) |
| `attribs` | `dict[str, Any]` | Custom attributes (queryable) |
| `status` | `Optional[str]` | `'enabled'`, `'disabled'`, or `'blocklisted'` |

### `Campaign`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | Campaign ID |
| `created_at` | `datetime` | Creation timestamp |
| `updated_at` | `Optional[datetime]` | Last update timestamp |
| `views` | `int` | View/open count |
| `clicks` | `int` | Link click count |
| `lists` | `list[dict[str, Any]]` | Target lists (each a dict) |
| `started_at` | `Optional[datetime]` | When sending began |
| `to_send` | `int` | Total recipients |
| `sent` | `int` | Emails sent so far |
| `uuid` | `str` | Campaign UUID |
| `name` | `Optional[str]` | Campaign name |
| `type` | `Optional[str]` | `'regular'` or `'optin'` |
| `subject` | `Optional[str]` | Email subject |
| `from_email` | `Optional[str]` | Sender email |
| `body` | `Optional[str]` | Campaign body content |
| `altbody` | `Optional[str]` | Plain text alternative |
| `send_at` | `Optional[datetime]` | Scheduled send time |
| `status` | `Optional[str]` | `draft`, `scheduled`, `running`, `paused`, `cancelled`, `finished` |
| `content_type` | `Optional[str]` | `richtext`, `html`, `markdown`, `plain` |
| `tags` | `list[str]` | Campaign tags |
| `template_id` | `int` | Template ID used |
| `messenger` | `Optional[str]` | Delivery channel |
| `headers` | `list[dict[str, Optional[str]]]` | Custom headers (each a single-entry dict) |
| `media` | `Optional[list[dict[str, Any]]]` | Attached media files, or `None` |

### `Template`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | Template ID |
| `created_at` | `datetime` | Creation timestamp |
| `updated_at` | `Optional[datetime]` | Last update timestamp |
| `name` | `Optional[str]` | Template name |
| `subject` | `Optional[str]` | Subject (for transactional templates) |
| `body` | `Optional[str]` | Template HTML body |
| `type` | `Optional[str]` | `'campaign'` or `'tx'` |
| `is_default` | `Optional[bool]` | Whether this is the default template |

### `Media`

| Field | Type | Description |
|-------|------|-------------|
| `id` | `int` | Media ID (pass to `create_campaign`/`update_campaign` via `media_ids`) |
| `uuid` | `str` | Media UUID |
| `filename` | `Optional[str]` | Stored file name |
| `content_type` | `Optional[str]` | MIME type, if reported |
| `created_at` | `Optional[datetime]` | Upload timestamp |
| `uri` | `Optional[str]` | Server path the file is served from |
| `thumb_uri` | `Optional[str]` | Server path of the generated thumbnail, if any |

### `CampaignPreview` / `TemplatePreview`

| Field | Type | Description |
|-------|------|-------------|
| `preview` | `Optional[str]` | Rendered HTML preview string |

### Request-body models (internal)

These models represent the raw request bodies the client builds for you; you normally interact with the higher-level functions rather than constructing them directly.

- **`CreateSubscriberModel`** — payload for creating (and, in `update_subscriber`, replacing) a subscriber. Fields: `email: str`, `name: Optional[str]`, `status: str` (required), `lists: list[int]`, `preconfirm_subscriptions: bool` (required), `attribs: dict[str, Any]`.
- **`CreateCampaignModel`** — payload for creating a campaign. Mirrors the `create_campaign` inputs; `template_id: Optional[int]` is a required field with no default; `send_at` is serialized to an ISO-8601 string (or `None`).
- **`UpdateCampaignModel`** — subclass of `CreateCampaignModel` used for updates. Adds a validator that drops a `send_at` already in the past (sets it to `None`) so updates don't fail on a stale scheduled time.
- **`CreateTemplateModel`** — payload for creating/updating a template. Fields: `name`, `subject`, `body`, `type` (all `Optional[str]`), `is_default: Optional[bool] = False`.

---

## Errors

Custom exceptions live in `listmonk.errors`.

```python
from listmonk.errors import ValidationError, OperationNotAllowedError, ListmonkFileNotFoundError
```

| Exception | Parent | When Raised |
|-----------|--------|-------------|
| `ValidationError` | `Exception` | Client-side input validation failed, or the server returned an empty/malformed JSON response |
| `OperationNotAllowedError` | `ValidationError` | An operation was attempted before `set_url_base()` / a successful `login()` |
| `ListmonkFileNotFoundError` | `FileNotFoundError` | A local file referenced for upload (attachment or media) cannot be found |

Additionally, standard-library `ValueError` / `TypeError` are raised for bad arguments (see each function), and `httpx2.HTTPStatusError` may be raised on HTTP 4xx/5xx responses.

---

## Error-Handling Patterns

The client uses two distinct error models — know which functions raise and which return `False`:

**Functions that never raise on failure (return `False` / `None` instead):**

- `login()`, `is_healthy()`, `verify_login()` — return `False` on rejected credentials or an unreachable server.
- `confirm_optin()` — returns `False` on a non-success HTTP status.
- `add_subscribers_to_lists()` — returns `False` on empty inputs or an error status.
- Single-item lookups (`subscriber_by_*`, `campaign_by_id`, `template_by_id`) — return `None` when nothing matches.

**Functions that raise:**

```python
import httpx2
from listmonk.errors import OperationNotAllowedError, ValidationError, ListmonkFileNotFoundError

try:
    sub = listmonk.create_subscriber('user@example.com', 'Jane')
except OperationNotAllowedError:
    ...  # set_url_base()/login() not done yet
except ValueError:
    ...  # empty email
except httpx2.HTTPStatusError as e:
    ...  # e.g. 4xx duplicate, or 403 missing subscribers:sql_query permission
except ValidationError:
    ...  # empty/invalid server response
```

Guard the required setup order once, up front — every data call runs `validate_state()` and raises `OperationNotAllowedError` if the base URL is unset or you are not logged in.

---

## API Endpoints

The library targets these Listmonk API endpoints (defined in `listmonk.urls`):

| Constant | Path | Used By |
|----------|------|---------|
| `health` | `/api/health` | `login()`, `is_healthy()`, `verify_login()` |
| `lists` | `/api/lists` | `lists()`, `create_list()` |
| `lst` | `/api/lists/{list_id}` | `list_by_id()`, `delete_list()`, `update_list()` |
| `subscribers` | `/api/subscribers` | `subscribers()`, `subscriber_by_*()`, `create_subscriber()` |
| `subscriber` | `/api/subscribers/{subscriber_id}` | `update_subscriber()`, `delete_subscriber()` |
| `subscriber_lists` | `/api/subscribers/lists` (PUT) | `add_subscribers_to_lists()` |
| `opt_in` | `/subscription/optin/{subscriber_uuid}` | `confirm_optin()` |
| `send_tx` | `/api/tx` | `send_transactional_email()` |
| `campaigns` | `/api/campaigns` | `campaigns()`, `create_campaign()` |
| `campaign_id` | `/api/campaigns/{campaign_id}` | `campaign_by_id()`, `update_campaign()`, `delete_campaign()` |
| `campaign_id_preview` | `/api/campaigns/{campaign_id}/preview` | `campaign_preview_by_id()` |
| `media` | `/api/media` | `upload_media()` |
| `templates` | `/api/templates` | `templates()`, `create_template()` |
| `template_id` | `/api/templates/{template_id}` | `template_by_id()`, `update_template()`, `delete_template()` |
| `template_id_preview` | `/api/templates/{template_id}/preview` | `template_preview_by_id()` |
| `template_id_default` | `/api/templates/{template_id}/default` | `set_default_template()` |

---

## FAQ / Troubleshooting

### `httpx2.HTTPStatusError: Client error '403 Forbidden'`

```text
httpx2.HTTPStatusError: Client error '403 Forbidden' for url '...?query=subscribers.email=...'
```

The authenticated user lacks the `subscribers:sql_query` permission (needed whenever you pass `query_text` to `subscribers()`). Update the user's role in the Listmonk admin panel to include it. See the [Listmonk roles docs](https://listmonk.app/docs/roles-and-permissions/#user-roles).

### `ssl.SSLCertVerificationError` against a self-hosted instance

`httpx2` validates TLS certificates against the bundled `certifi` CA list. If you self-host behind a custom or corporate CA, point the client at your CA bundle via the standard environment variables before using the library:

```bash
export SSL_CERT_FILE=/path/to/your-ca-bundle.pem   # or SSL_CERT_DIR=/path/to/ca-dir
```

### Required Call Order

You **must** call functions in this order:

1. `listmonk.set_url_base('...')`
2. `listmonk.login('user', 'pass')`
3. Any other API call

Calling API functions before setup raises `OperationNotAllowedError`.

### Timeouts

All network functions accept an optional `timeout_config` (default: 10 seconds). Use `httpx2.Timeout` (from `httpx2`, not `httpx`):

```python
import httpx2

timeout = httpx2.Timeout(timeout=30.0)
subs = listmonk.subscribers(timeout_config=timeout)
```

### Global State

The library uses module-level global variables for authentication:

- Credentials persist for the application lifetime.
- Only one Listmonk instance can be configured at a time.
- Thread safety is not guaranteed for credential changes.

---

## Complete Public API

All public functions exported by `listmonk`:

| Function | Category | Returns |
|----------|----------|---------|
| `set_url_base(url)` | Config | `None` |
| `get_base_url()` | Config | `Optional[str]` |
| `login(user, pw)` | Auth | `bool` |
| `verify_login()` | Auth | `bool` |
| `is_healthy()` | Health | `bool` |
| `lists()` | Lists | `list[MailingList]` |
| `list_by_id(id)` | Lists | `MailingList` |
| `create_list(name, ...)` | Lists | `MailingList` |
| `update_list(id, ...)` | Lists | `MailingList` |
| `delete_list(id)` | Lists | `bool` |
| `subscribers(query, list_id)` | Subscribers | `list[Subscriber]` |
| `subscriber_by_email(email)` | Subscribers | `Optional[Subscriber]` |
| `subscriber_by_id(id)` | Subscribers | `Optional[Subscriber]` |
| `subscriber_by_uuid(uuid)` | Subscribers | `Optional[Subscriber]` |
| `create_subscriber(email, ...)` | Subscribers | `Subscriber` |
| `update_subscriber(sub, add, remove)` | Subscribers | `Optional[Subscriber]` |
| `disable_subscriber(sub)` | Subscribers | `Optional[Subscriber]` |
| `enable_subscriber(sub)` | Subscribers | `Optional[Subscriber]` |
| `block_subscriber(sub)` | Subscribers | `Optional[Subscriber]` |
| `delete_subscriber(email)` | Subscribers | `bool` |
| `add_subscribers_to_lists(ids, lists)` | Subscribers | `bool` |
| `confirm_optin(sub_uuid, list_uuid)` | Subscribers | `bool` |
| `campaigns()` | Campaigns | `list[Campaign]` |
| `campaign_by_id(id)` | Campaigns | `Optional[Campaign]` |
| `campaign_preview_by_id(id)` | Campaigns | `CampaignPreview` |
| `create_campaign(name, subject, ...)` | Campaigns | `Campaign` |
| `update_campaign(campaign, media_ids)` | Campaigns | `Optional[Campaign]` |
| `delete_campaign(id)` | Campaigns | `bool` |
| `upload_media(file, filename)` | Media | `Media` |
| `templates()` | Templates | `list[Template]` |
| `template_by_id(id)` | Templates | `Optional[Template]` |
| `template_preview_by_id(id)` | Templates | `TemplatePreview` |
| `create_template(name, body, ...)` | Templates | `Template` |
| `update_template(template)` | Templates | `Optional[Template]` |
| `delete_template(id)` | Templates | `bool` |
| `set_default_template(id)` | Templates | `bool` |
| `send_transactional_email(email, ...)` | Transactional | `bool` |
</content>
</invoke>
