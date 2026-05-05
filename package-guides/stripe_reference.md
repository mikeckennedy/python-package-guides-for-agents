# Stripe Python — Comprehensive API Reference

> Official Python bindings for the Stripe API: payments, billing, subscriptions, Connect, Issuing, Terminal, Climate, Tax, Identity, Treasury, and more. The SDK exposes a typed, code-generated `StripeClient` that mirrors the entire Stripe REST API, plus webhook signature verification, OAuth helpers for Connect, async equivalents of every request method, and pluggable HTTP clients (`requests`, `httpx`, `aiohttp`, `pycurl`, `urllib`, GAE `urlfetch`).

> Version: 15.1.0 | License: MIT | Python: 3.9+ | Stripe API version: 2026-04-22.dahlia
> Built from the source code of the `stripe-python` 15.1.0 repository.

---

## Table of contents

- [Installation](#installation)
- [Quick start](#quick-start)
- [Two ways to use the SDK](#two-ways-to-use-the-sdk)
  - [StripeClient (recommended)](#stripeclient-recommended)
  - [Legacy global API](#legacy-global-api)
- [Configuring the client](#configuring-the-client)
  - [StripeClient constructor](#stripeclient-constructor)
  - [Global configuration variables](#global-configuration-variables)
  - [Base URLs (`base_addresses`)](#base-urls-base_addresses)
  - [Timeouts and automatic retries](#timeouts-and-automatic-retries)
  - [Proxies](#proxies)
  - [App info (plugin authors)](#app-info-plugin-authors)
- [Per-request options](#per-request-options)
  - [The `RequestOptions` TypedDict](#the-requestoptions-typeddict)
  - [Idempotency keys](#idempotency-keys)
  - [Per-request API version / account](#per-request-api-version--account)
- [Service architecture](#service-architecture)
  - [`v1` services (full list)](#v1-services-full-list)
  - [`v2` services](#v2-services)
  - [Method shape](#method-shape)
  - [Sub-services](#sub-services)
- [Working with resources](#working-with-resources)
  - [`StripeObject` — base class](#stripeobject--base-class)
  - [`ListObject` and pagination](#listobject-and-pagination)
  - [`SearchResultObject`](#searchresultobject)
  - [Accessing the underlying HTTP response](#accessing-the-underlying-http-response)
- [Async usage](#async-usage)
- [Webhooks](#webhooks)
  - [V1 snapshot events (`Webhook.construct_event`)](#v1-snapshot-events-webhookconstruct_event)
  - [V2 thin event notifications (`parse_event_notification`)](#v2-thin-event-notifications-parse_event_notification)
  - [Manual signature verification](#manual-signature-verification)
- [Errors](#errors)
  - [Exception hierarchy](#exception-hierarchy)
  - [Inspecting an error](#inspecting-an-error)
- [File uploads](#file-uploads)
- [Raw / undocumented requests](#raw--undocumented-requests)
- [Stripe Connect (OAuth)](#stripe-connect-oauth)
- [HTTP clients](#http-clients)
- [Logging](#logging)
- [Telemetry](#telemetry)
- [Common resources at a glance](#common-resources-at-a-glance)
  - [Customer](#customer)
  - [PaymentIntent](#paymentintent)
  - [Charge](#charge)
  - [Refund](#refund)
  - [Subscription](#subscription)
  - [Invoice](#invoice)
  - [Product and Price](#product-and-price)
  - [Checkout Session](#checkout-session)
  - [PaymentMethod](#paymentmethod)
  - [SetupIntent](#setupintent)
  - [PayoutsTransfersBalance](#payouts-transfers-and-balance)
- [Common patterns](#common-patterns)
  - [Listing all results](#listing-all-results)
  - [Search](#search)
  - [Idempotent retries](#idempotent-retries)
  - [Connected accounts (`Stripe-Account` header)](#connected-accounts-stripe-account-header)
  - [Metering events (high-throughput)](#metering-events-high-throughput)
  - [Test clocks and test helpers](#test-clocks-and-test-helpers)

---

## Installation

```sh
pip install --upgrade stripe
```

For async use, install with the `async` extra so an async-capable HTTP client (`httpx`) is pulled in:

```sh
pip install --upgrade "stripe[async]"
```

`stripe` requires Python 3.9+. The package name on PyPI is `stripe`; the import is also `stripe`. Type annotations rely on PEP 692 `Unpack[TypedDict]`, which works best with Pyright; MyPy support degrades gracefully.

Optional runtime dependencies the SDK can pick up automatically (in priority order, see [HTTP clients](#http-clients)):

- `urlfetch` (Google App Engine) — preferred when present
- `requests` ≥ 2.20 — recommended default for sync
- `pycurl` — fallback
- `urllib` — last-resort fallback (warns)

For async: `httpx` (default if installed) or `aiohttp`.

---

## Quick start

```python
from stripe import StripeClient

client = StripeClient("sk_test_...")

# create a customer
customer = client.v1.customers.create(params={"email": "ada@example.com"})
print(customer.id, customer.email)

# retrieve it
same = client.v1.customers.retrieve(customer.id)

# list customers (returns a ListObject)
page = client.v1.customers.list(params={"limit": 3})
for c in page.auto_paging_iter():
    print(c.id, c.email)

# update
client.v1.customers.update(customer.id, params={"name": "Ada Lovelace"})

# delete
client.v1.customers.delete(customer.id)
```

Every service method has an `_async` twin: `client.v1.customers.create_async(...)`.

---

## Two ways to use the SDK

### StripeClient (recommended)

Introduced in v8. All resources are reached through `client.v1.*` and `client.v2.*` services. New API endpoints are only added to the StripeClient namespace.

```python
from stripe import StripeClient

client = StripeClient("sk_test_...")
intent = client.v1.payment_intents.create(params={
    "amount": 2000,
    "currency": "usd",
    "automatic_payment_methods": {"enabled": True},
})
```

Top-level shortcut services like `client.customers` still exist but emit `DeprecationWarning` and forward to `client.v1.customers`. Prefer `client.v1.<service>`.

**Source:** `stripe/_stripe_client.py:136` (`class StripeClient`).

### Legacy global API

Pre-v8 style: configure `stripe.api_key` and call resource classes directly.

```python
import stripe

stripe.api_key = "sk_test_..."
customer = stripe.Customer.create(email="ada@example.com")
charge = stripe.Charge.retrieve("ch_...")

for c in stripe.Customer.list(limit=3).auto_paging_iter():
    print(c.id)

stripe.Customer.modify(customer.id, name="Ada Lovelace")  # not .save()
stripe.Customer.delete(customer.id)
```

The global API is being phased out — it will be marked deprecated. New endpoints land only in `StripeClient`. The legacy `obj.save()` method on resources is already deprecated in favor of `Resource.modify(id, ...)` (see `stripe/_updateable_api_resource.py:17`).

The global module uses lazy imports through `__getattr__` (`stripe/__init__.py:872`), so `stripe.Customer`, `stripe.PaymentIntent`, `stripe.error.CardError`, `stripe.checkout.Session`, etc., all resolve transparently.

---

## Configuring the client

### StripeClient constructor

```python
StripeClient(
    api_key: str,
    *,
    stripe_account: Optional[str] = None,
    stripe_context: Optional[Union[str, "StripeContext"]] = None,
    stripe_version: Optional[str] = None,
    base_addresses: Optional[BaseAddresses] = None,
    client_id: Optional[str] = None,
    verify_ssl_certs: bool = True,
    proxy: Optional[str] = None,
    max_network_retries: Optional[int] = None,
    http_client: Optional["HTTPClient"] = None,
)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `api_key` | `str` | required | Secret key (`sk_test_...` / `sk_live_...`). Passing `None` raises `AuthenticationError`. |
| `stripe_account` | `str \| None` | `None` | Sets the `Stripe-Account` header on every request — used to act on behalf of a connected account. |
| `stripe_context` | `str \| StripeContext \| None` | `None` | Sets `Stripe-Context` header (used by some preview/Connect features). |
| `stripe_version` | `str \| None` | the SDK's pinned version | Override the API version sent in `Stripe-Version`. Defaults to `_ApiVersion.CURRENT` (currently `2026-04-22.dahlia`). |
| `base_addresses` | `BaseAddresses` (TypedDict) | Stripe production URLs | Override per-base URLs. Useful for `stripe-mock`. See below. |
| `client_id` | `str \| None` | `None` | OAuth `client_id` (used by `client.oauth.*`). |
| `verify_ssl_certs` | `bool` | `True` | Disable TLS verification (don't, except in dev). Cannot combine with a custom `http_client`. |
| `proxy` | `str \| None` | `None` | HTTP/HTTPS proxy URL. Cannot combine with a custom `http_client`. |
| `max_network_retries` | `int \| None` | `None` (falls back to `stripe.max_network_retries = 2`) | Automatic retries on transient errors and HTTP 409. |
| `http_client` | `HTTPClient \| None` | auto-selected | Plug in a specific transport (see [HTTP clients](#http-clients)). |

**Source:** `stripe/_stripe_client.py:137`.

### Global configuration variables

Configuration on the `stripe` module (used by the legacy global API and by webhook helpers):

```python
import stripe

stripe.api_key            # Optional[str]
stripe.client_id          # Optional[str]
stripe.api_version        # str — defaults to the SDK's pinned version
stripe.api_base           # "https://api.stripe.com"
stripe.connect_api_base   # "https://connect.stripe.com"
stripe.upload_api_base    # "https://files.stripe.com"
stripe.meter_events_api_base  # "https://meter-events.stripe.com"
stripe.verify_ssl_certs   # True
stripe.proxy              # Optional[str]
stripe.default_http_client  # Optional[HTTPClient]
stripe.app_info           # Optional[AppInfo]
stripe.enable_telemetry   # True
stripe.max_network_retries  # 2
stripe.ca_bundle_path     # path to bundled Mozilla CA certs
stripe.log                # None | "info" | "debug"
```

Module-level constants:

```python
stripe.DEFAULT_API_BASE             # "https://api.stripe.com"
stripe.DEFAULT_CONNECT_API_BASE     # "https://connect.stripe.com"
stripe.DEFAULT_UPLOAD_API_BASE      # "https://files.stripe.com"
stripe.DEFAULT_METER_EVENTS_API_BASE  # "https://meter-events.stripe.com"
```

**Source:** `stripe/__init__.py:14-44`.

### Base URLs (`base_addresses`)

A `BaseAddresses` TypedDict has four keys, each optional:

```python
{
    "api": "https://api.stripe.com",
    "connect": "https://connect.stripe.com",
    "files": "https://files.stripe.com",
    "meter_events": "https://meter-events.stripe.com",
}
```

Overriding is mostly useful for [stripe-mock](https://github.com/stripe/stripe-mock):

```python
client = StripeClient(
    "sk_test_xxx",
    base_addresses={"api": "http://localhost:12111"},
)
```

### Timeouts and automatic retries

`max_network_retries` triggers retries with exponential backoff on:

- connection errors / timeouts
- HTTP `409 Conflict`
- HTTP `429`, `5xx` (subject to internal rules)

Retries respect the server's `Retry-After` header (capped by `HTTPClient.MAX_RETRY_AFTER = 60`). Backoff is bounded by `HTTPClient.MAX_DELAY = 5` seconds and starts at `HTTPClient.INITIAL_DELAY = 0.5`. Each retry adds a `Stripe-Should-Retry`/idempotency telemetry header so the SDK can guarantee safety; if you don't supply an `idempotency_key`, the SDK auto-generates one for retries.

```python
client = StripeClient("sk_test_...", max_network_retries=3)
```

The default HTTP request timeout for `RequestsClient` is **80 seconds** (connect+read combined). For `HTTPXClient` and `AIOHTTPClient` the default is also `80`. Override by constructing the HTTP client yourself:

```python
client = StripeClient(
    "sk_test_...",
    http_client=stripe.RequestsClient(timeout=30),
)
```

`timeout` accepts an `(int, int)` tuple for `RequestsClient` (connect, read), an `httpx.Timeout` for `HTTPXClient`, or an `aiohttp.ClientTimeout` for `AIOHTTPClient`.

### Proxies

```python
client = StripeClient("sk_test_...", proxy="https://user:pass@proxy.example.com:1234")
```

This is forwarded to whichever transport is auto-selected. If you supply your own `http_client`, configure the proxy on it instead (the `proxy` and `verify_ssl_certs` constructor args raise `ValueError` if combined with a custom `http_client`).

### App info (plugin authors)

If you build a library or plugin on top of `stripe`, identify it so Stripe sees the right `User-Agent`:

```python
import stripe
stripe.set_app_info("MyAwesomePlugin", version="1.2.34", url="https://myawesome.example")
```

`set_app_info(name, partner_id=None, url=None, version=None)` populates the `stripe.app_info` `AppInfo` TypedDict. **Source:** `stripe/__init__.py:94`.

---

## Per-request options

Every service method accepts an `options` keyword (a `RequestOptions` `TypedDict`) that overrides client-level configuration for a single call.

```python
client.v1.customers.list(
    options={
        "api_key": "sk_test_alternate_...",
        "stripe_account": "acct_1234",
        "stripe_version": "2024-04-10",
        "max_network_retries": 0,
        "idempotency_key": "abc123",
        "headers": {"X-Custom": "value"},
    }
)
```

### The `RequestOptions` TypedDict

```python
class RequestOptions(TypedDict):
    api_key: NotRequired[str | None]
    stripe_version: NotRequired[str | None]
    stripe_account: NotRequired[str | None]
    stripe_context: NotRequired[str | StripeContext | None]
    max_network_retries: NotRequired[int | None]
    idempotency_key: NotRequired[str | None]
    content_type: NotRequired[str | None]
    headers: NotRequired[Mapping[str, str] | None]
```

`api_key`, `stripe_version`, `stripe_account`, `stripe_context` *persist* on the resulting `StripeObject` so subsequent operations against that object (e.g., `obj.refresh()`, paging) use the same auth. The other keys are single-shot.

**Source:** `stripe/_request_options.py:9`.

### Idempotency keys

Stripe guarantees that any `POST` with an `Idempotency-Key` returns the same response if retried. The SDK auto-generates one when retrying a request internally. Supply your own to make retries safe across process restarts:

```python
intent = client.v1.payment_intents.create(
    params={"amount": 2000, "currency": "usd", "customer": "cus_..."},
    options={"idempotency_key": "order-9821-payment-intent"},
)
```

Reusing a key with **different** parameters returns `IdempotencyError`.

### Per-request API version / account

```python
# Pin one request to an older API version
client.v1.subscriptions.retrieve(
    "sub_...",
    options={"stripe_version": "2024-04-10"},
)

# Act as a connected account (Stripe Connect)
client.v1.charges.list(options={"stripe_account": "acct_1ABCxyz..."})
```

---

## Service architecture

Each Stripe resource has a generated **service** class (e.g. `CustomerService`) and a corresponding **resource** class (e.g. `Customer`). The StripeClient surface is `client.v1.<service>` / `client.v2.<namespace>.<service>`. The legacy surface is `stripe.<Resource>`.

### `v1` services (full list)

Available as `client.v1.<name>`:

```
accounts                        account_links                   account_sessions
apple_pay_domains               application_fees                apps
balance                         balance_settings                balance_transactions
billing                         billing_portal                  charges
checkout                        climate                         confirmation_tokens
country_specs                   coupons                         credit_notes
customers                       customer_sessions               disputes
entitlements                    ephemeral_keys                  events
exchange_rates                  files                           file_links
financial_connections           forwarding                      identity
invoices                        invoice_items                   invoice_payments
invoice_rendering_templates     issuing                         mandates
payment_attempt_records         payment_intents                 payment_links
payment_methods                 payment_method_configurations   payment_method_domains
payment_records                 payouts                         plans
prices                          products                        promotion_codes
quotes                          radar                           refunds
reporting                       reviews                         setup_attempts
setup_intents                   shipping_rates                  sigma
sources                         subscriptions                   subscription_items
subscription_schedules          tax                             tax_codes
tax_ids                         tax_rates                       terminal
test_helpers                    tokens                          topups
transfers                       treasury                        webhook_endpoints
```

**Source:** `stripe/_v1_services.py:233-304`.

Some entries (`apps`, `billing`, `billing_portal`, `checkout`, `climate`, `entitlements`, `financial_connections`, `forwarding`, `identity`, `issuing`, `radar`, `reporting`, `sigma`, `tax`, `terminal`, `test_helpers`, `treasury`) are *namespace* services that contain further sub-services (e.g. `client.v1.checkout.sessions`, `client.v1.billing.meters`, `client.v1.terminal.readers`).

### `v2` services

```python
client.v2.billing.meter_events           # high-throughput meter events
client.v2.billing.meter_event_session    # session for meter event streams
client.v2.billing.meter_event_stream     # streaming meter events endpoint
client.v2.billing.meter_event_adjustments

client.v2.core.accounts
client.v2.core.account_links
client.v2.core.account_tokens
client.v2.core.events                    # V2 thin events
client.v2.core.event_destinations
```

**Source:** `stripe/v2/_billing_service.py:19-37`, `stripe/v2/_core_service.py:16-33`.

### Method shape

Every generated service method follows one of these signatures:

```python
# Create
service.create(params: TypedDict | None = None, options: RequestOptions | None = None) -> Resource

# Retrieve
service.retrieve(id: str, params: ... = None, options: ... = None) -> Resource

# Update
service.update(id: str, params: ... = None, options: ... = None) -> Resource

# Delete
service.delete(id: str, params: ... = None, options: ... = None) -> Resource

# List
service.list(params: ... = None, options: ... = None) -> ListObject[Resource]

# Search
service.search(params: SearchParams, options: ... = None) -> SearchResultObject[Resource]
```

Each has an async twin: `create_async`, `retrieve_async`, `update_async`, `delete_async`, `list_async`, `search_async`.

`params` is a `TypedDict` per operation (e.g., `CustomerCreateParams`, `PaymentIntentCreateParams`) under `stripe.params`. You don't have to import them — they're for type-checking only — just pass a regular dict. Optional fields use `NotRequired[...]`; some accept the literal `""` to clear a previously-set value (e.g., `metadata=""`).

**Source for the customer service:** `stripe/_customer_service.py`.

### Sub-services

Several services expose nested services:

```python
client.v1.customers.balance_transactions   # CustomerBalanceTransactionService
client.v1.customers.cash_balance           # CustomerCashBalanceService
client.v1.customers.cash_balance_transactions
client.v1.customers.funding_instructions
client.v1.customers.payment_methods        # CustomerPaymentMethodService
client.v1.customers.payment_sources
client.v1.customers.tax_ids
```

Sub-services are loaded lazily through `__getattr__` (see `stripe/_customer_service.py:85-99`), so importing `StripeClient` doesn't pay the cost of every service file.

---

## Working with resources

### `StripeObject` — base class

Every resource returned by the SDK is a `StripeObject` (or a subclass). It behaves like a dict and an attribute-access object:

```python
customer = client.v1.customers.retrieve("cus_...")
customer.email          # attribute access
customer["email"]       # dict access
customer.metadata.get("plan")
"deleted" in customer
```

Useful methods (`stripe/_stripe_object.py`):

| Method | Description |
|---|---|
| `to_dict(recursive=True, for_json=False)` | Plain dict copy of the underlying data. `for_json=True` makes non-JSON-serializable values (e.g. `Decimal`) JSON-friendly. |
| `update(d)` | Merge a mapping into the object. |
| `__str__()` | Pretty-printed JSON of the object. |
| `last_response` | `StripeResponse` with `.code`, `.headers`, `.body`, `.idempotency_key`. |
| `serialize(previous=None)` | Diff against a prior state for partial updates (used internally by the deprecated `.save()`). |
| `request(method, url, params=None)` | Low-level request that inherits the object's auth context. |
| `refresh()` / `refresh_async()` | Re-fetch the object from the API. |

### `ListObject` and pagination

A list endpoint returns a `ListObject[T]` (a `StripeObject` subclass) with these attributes:

```python
class ListObject(StripeObject, Generic[T]):
    OBJECT_NAME = "list"
    data: List[T]
    has_more: bool
    url: str
```

Iteration over a `ListObject` only iterates the current page. To paginate the whole result set, use `auto_paging_iter()`:

```python
page = client.v1.customers.list(params={"limit": 100})

# manual: just this page
for c in page:
    print(c.id)

# automatic: every page, transparently
for c in page.auto_paging_iter():
    print(c.id)

# async equivalent
async for c in (await client.v1.customers.list_async()).auto_paging_iter():
    print(c.id)
```

`auto_paging_iter()` returns an `AnyIterator[T]` that implements both `Iterable` and `AsyncIterable`, so the same call works in both contexts.

Manual page navigation:

```python
page = client.v1.customers.list(params={"limit": 50})
next_page = page.next_page()           # newer items, walking forward
prev_page = page.previous_page()       # older items
# also: next_page_async() / previous_page_async()
```

If `ending_before` is set without `starting_after`, `auto_paging_iter()` walks **backwards** automatically.

A `ListObject` also supports `.create(**params)` and `.retrieve(id, **params)` against its `url` — useful when you reach a list through a parent resource (e.g. `customer.subscriptions.create(...)`).

**Source:** `stripe/_list_object.py`.

### `SearchResultObject`

Search endpoints return `SearchResultObject[T]`:

```python
class SearchResultObject(StripeObject, Generic[T]):
    OBJECT_NAME = "search_result"
    data: List[T]
    has_more: bool
    next_page: str
```

Use `auto_paging_iter()` exactly as with `ListObject`. The cursor uses an opaque `next_page` token instead of `starting_after`. Search is eventually consistent — Stripe warns it can lag by up to a minute under normal load, more during incidents (`stripe/_customer_service.py:362-366`).

**Source:** `stripe/_search_result_object.py`.

### Accessing the underlying HTTP response

```python
customer = client.v1.customers.retrieve("cus_...")

print(customer.last_response.code)         # int, HTTP status
print(customer.last_response.headers)      # Mapping[str, str]
print(customer.last_response.idempotency_key)
```

`last_response` is a `StripeResponse` with `.code`, `.headers`, `.body`, `.data`. There are also `StripeStreamResponse` / `StripeStreamResponseAsync` for streamed file downloads.

---

## Async usage

Every service method has an `_async` companion that returns a coroutine:

```python
import asyncio
from stripe import StripeClient

async def main():
    client = StripeClient("sk_test_...")

    customer = await client.v1.customers.create_async(
        params={"email": "ada@example.com"},
    )

    # auto_paging_iter() works in both directions
    async for c in (
        await client.v1.customers.list_async(params={"limit": 50})
    ).auto_paging_iter():
        print(c.id)

asyncio.run(main())
```

The default async transport is `httpx` (preferred) or `aiohttp` (fallback) — installed via `pip install stripe[async]`. If you mix sync and async calls on a single `HTTPXClient`, set `allow_sync_methods=True`:

```python
my_client = stripe.HTTPXClient(allow_sync_methods=True)
client = StripeClient("sk_test_...", http_client=my_client)
```

Otherwise calling a sync method on an `HTTPXClient` initialized with `allow_sync_methods=False` raises `RuntimeError` (`stripe/_http_client.py:1297-1301`).

`AIOHTTPClient` is async-only — sync methods raise.

The legacy global API also supports async:

```python
import stripe

stripe.api_key = "sk_test_..."
customer = await stripe.Customer.retrieve_async("cus_...")
```

There is **no** `save_async`. The legacy `obj.save()` is deprecated for both sync and async — use `Resource.modify(id, ...)` / `Resource.modify_async(id, ...)` instead.

---

## Webhooks

Two webhook flavors:

- **V1 (Snapshot) events** — the long-standing model, with the entire `Event` payload included.
- **V2 thin event notifications** — newer; the webhook body contains only an event id and minimal context, and you fetch the full event/related object on demand.

Both share the same signature scheme (`v1`, HMAC-SHA256), and both use `Webhook.DEFAULT_TOLERANCE = 300` seconds for replay protection.

### V1 snapshot events (`Webhook.construct_event`)

```python
from stripe import Webhook, SignatureVerificationError

@app.route("/webhooks", methods=["POST"])
def webhooks():
    payload = request.data
    sig_header = request.headers.get("Stripe-Signature", "")
    secret = os.environ["STRIPE_WEBHOOK_SECRET"]

    try:
        event = Webhook.construct_event(payload, sig_header, secret)
    except ValueError:
        return "bad payload", 400
    except SignatureVerificationError:
        return "bad signature", 400

    if event.type == "payment_intent.succeeded":
        intent = event.data.object   # a PaymentIntent
        ...
    return "", 200
```

`Webhook.construct_event` signature:

```python
Webhook.construct_event(
    payload: bytes | str,
    sig_header: str,
    secret: str,
    tolerance: int = Webhook.DEFAULT_TOLERANCE,  # 300 seconds
    api_key: Optional[str] = None,
) -> Event
```

`payload` may be bytes or str (bytes are decoded as UTF-8). If the payload turns out to be a V2 thin notification (`object == "v2.core.event"`), `construct_event` raises `ValueError` telling you to use `parse_event_notification` instead.

**Source:** `stripe/_webhook.py:15`.

### V2 thin event notifications (`parse_event_notification`)

This lives on `StripeClient` — there is no module-level helper.

```python
from stripe import StripeClient
from stripe.events import UnknownEventNotification

client = StripeClient(os.environ["STRIPE_API_KEY"])

@app.post("/webhook")
def webhook():
    body = request.data
    sig = request.headers.get("Stripe-Signature", "")
    secret = os.environ["WEBHOOK_SECRET"]

    notif = client.parse_event_notification(body, sig, secret)

    if notif.type == "v1.billing.meter.error_report_triggered":
        meter = notif.fetch_related_object()      # full Meter
        event = notif.fetch_event()               # full Event
    elif isinstance(notif, UnknownEventNotification):
        # Forward-compatibility for events the SDK doesn't yet know about
        ...

    return {"ok": True}
```

The return type is a `Union` of all known event-notification classes (`stripe.events.ALL_EVENT_NOTIFICATIONS`), which a type-checker can narrow on `notif.type`. Unknown event types return `UnknownEventNotification` — match on `notif.type` manually for those.

Each notification typically exposes:

- `notif.type` — string discriminator
- `notif.id`, `notif.created`, `notif.livemode`, `notif.context`
- `notif.related_object` — `{id, type, url}` summary (may be `None` for some notification types)
- `notif.fetch_related_object()` / `_async()` — fetch the full object
- `notif.fetch_event()` / `_async()` — fetch the full V2 `Event`

**Source:** `stripe/_stripe_client.py:216`.

### Manual signature verification

If you want to verify the signature without parsing the body into an `Event`, call `WebhookSignature.verify_header` directly:

```python
from stripe import WebhookSignature

WebhookSignature.verify_header(payload_str, sig_header, secret, tolerance=300)
```

Raises `SignatureVerificationError` on a missing `t=`/`v1=`, missing matching signatures, or a stale timestamp. **Source:** `stripe/_webhook.py:43`.

---

## Errors

Every API request that fails raises a subclass of `stripe.StripeError`. Import error classes from `stripe` directly or from `stripe.error` (legacy alias):

```python
from stripe import (
    StripeError,
    APIError,
    APIConnectionError,
    AuthenticationError,
    CardError,
    IdempotencyError,
    InvalidRequestError,
    PermissionError,
    RateLimitError,
    SignatureVerificationError,
    TemporarySessionExpiredError,
)
```

### Exception hierarchy

```
StripeError
├── APIError                       # generic 5xx / unknown errors
├── APIConnectionError             # network problem, may have should_retry=True
├── AuthenticationError            # bad / missing API key
├── PermissionError                # API key lacks the required permission
├── RateLimitError                 # 429
├── IdempotencyError               # idempotency-key reuse with different body
├── SignatureVerificationError     # webhook signature failed
├── TemporarySessionExpiredError   # generated; transient session error
└── StripeErrorWithParamCode       # base for errors that include a "param"
    ├── InvalidRequestError        # 400, with .param indicating the bad field
    └── CardError                  # the card was declined; .param + .code
```

**Source:** `stripe/_error.py`.

### Inspecting an error

```python
from stripe import CardError, StripeError

try:
    intent = client.v1.payment_intents.confirm("pi_...")
except CardError as e:
    print(e.user_message)     # raw message from Stripe
    print(e.code)             # e.g. "card_declined"
    print(e.param)            # which param caused it
    print(e.error.decline_code if e.error else None)
    print(e.http_status)      # int
    print(e.request_id)       # "req_xxxx" — share with Stripe support
    print(e.json_body)        # full JSON body
    print(e.headers)          # response headers
except StripeError as e:
    # Catch-all for any other Stripe error
    print(repr(e))
```

`StripeError.__str__` is `"Request <req_id>: <message>"` if the request id is known, otherwise just the message. Use `e.user_message` to get just the bare message.

`APIConnectionError.should_retry` is `True` if the SDK considers the failure retryable (used by the auto-retry machinery).

---

## File uploads

File uploads target a **separate** base URL (`files.stripe.com`) and use `multipart/form-data`. The service handles that for you:

```python
with open("dispute_evidence.pdf", "rb") as fp:
    file = client.v1.files.create(params={
        "file": fp,
        "purpose": "dispute_evidence",
    })

print(file.id, file.size, file.purpose)
```

`FileCreateParams` (`stripe/params/_file_create_params.py:9`):

| Param | Type | Required | Description |
|---|---|---|---|
| `file` | file-like object (or `(filename, fileobj, content_type)` tuple) | yes | The file to upload. Must follow RFC 2388 multipart conventions. |
| `purpose` | one of `account_requirement`, `additional_verification`, `business_icon`, `business_logo`, `customer_signature`, `dispute_evidence`, `identity_document`, `issuing_regulatory_reporting`, `pci_document`, `platform_terms_of_service`, `tax_document_user_upload`, `terminal_android_apk`, `terminal_reader_splashscreen`, `terminal_wifi_certificate`, `terminal_wifi_private_key` | yes | Why the file is being uploaded. |
| `file_link_data` | TypedDict (`{create: bool, expires_at?: int, metadata?: dict}`) | no | Auto-create a `FileLink` for the upload (only valid for some purposes). |
| `expand` | `List[str]` | no | Fields to expand. |

The service auto-sets `options["content_type"] = "multipart/form-data"` and routes to the `files` base address (`stripe/_file_service.py:66-78`). To create a sharable link to an existing file:

```python
link = client.v1.file_links.create(params={"file": file.id, "expires_at": 1900000000})
print(link.url)
```

---

## Raw / undocumented requests

For private betas or to bypass the SDK's typed wrappers entirely:

```python
client = StripeClient("sk_test_...")

response = client.raw_request(
    "post",
    "/v1/beta_endpoint",
    param=123,
    stripe_version="2022-11-15; feature_beta=v3",
)

# Optional: turn the StripeResponse into a typed StripeObject
deserialized = client.deserialize(response, api_mode="V1")
```

`raw_request(method_, url_, **params)` accepts **all** `RequestOptions` keys as kwargs (`api_key`, `stripe_version`, `stripe_account`, `idempotency_key`, `headers`, etc.) plus arbitrary body params. There's also `client.raw_request_async(method_, url_, **params)`.

`deserialize(resp, params=None, *, api_mode)` accepts `api_mode="V1"` or `api_mode="V2"`.

**Source:** `stripe/_stripe_client.py:266-324`.

---

## Stripe Connect (OAuth)

Connect's OAuth flow has helpers on both `StripeClient` and the global `OAuth` class.

```python
from stripe import StripeClient

client = StripeClient(
    api_key=os.environ["STRIPE_SECRET_KEY"],
    client_id=os.environ["STRIPE_CLIENT_ID"],
)

# Step 1: redirect the user to the Connect OAuth page
url = client.oauth.authorize_url(scope="read_write")
return redirect(url)

# Step 2: in your callback, exchange the code for a token
resp = client.oauth.token(params={
    "grant_type": "authorization_code",
    "code": request.args["code"],
})
connected_account_id = resp["stripe_user_id"]
access_token = resp["access_token"]

# Step 3 (optional): later, disconnect
client.oauth.deauthorize(params={"stripe_user_id": connected_account_id})
```

The legacy module-level helpers (`stripe.OAuth.authorize_url`, `stripe.OAuth.token`, `stripe.OAuth.deauthorize`) work the same way and read `stripe.client_id` from module configuration.

`authorize_url(express=False, **OAuthAuthorizeUrlParams) -> str` builds (but does **not** call) the URL. `express=True` switches to the `/express/oauth/authorize` path. If you don't pass `client_id`, `stripe.client_id` is used; if neither is set, `AuthenticationError` is raised. `response_type` defaults to `"code"`. **Source:** `stripe/_oauth.py:314`.

`token(api_key=None, **OAuthTokenParams) -> OAuthToken` — the response object exposes `access_token`, `refresh_token`, `scope`, `livemode`, `token_type`, `stripe_user_id`, `stripe_publishable_key`. **Source:** `stripe/_oauth.py:330`.

`deauthorize(api_key=None, **OAuthDeauthorizeParams) -> OAuthDeauthorization` — returns `{stripe_user_id: ...}` confirming success. **Source:** `stripe/_oauth.py:347`.

OAuth-specific errors come from `stripe.oauth_error` — catch `OAuthError`:

```python
from stripe.oauth_error import OAuthError

try:
    client.oauth.token(params={"grant_type": "authorization_code", "code": code})
except OAuthError as e:
    ...
```

To act on a connected account in subsequent requests, set `stripe_account`:

```python
client.v1.charges.list(options={"stripe_account": "acct_..."})
```

---

## HTTP clients

The SDK ships with seven transports. All inherit from `stripe.HTTPClient`:

| Class | Sync | Async | Notes |
|---|:-:|:-:|---|
| `RequestsClient` | ✓ | via fallback | Default sync transport. Uses thread-local `requests.Session`. |
| `UrlFetchClient` | ✓ | – | Google App Engine. Auto-selected when `google.appengine.api.urlfetch` is importable. |
| `PycurlClient` | ✓ | – | Fallback when `pycurl` is installed but `requests` is not. |
| `UrllibClient` | ✓ | – | Last-resort sync fallback. |
| `HTTPXClient` | optional | ✓ | Default async transport. Set `allow_sync_methods=True` to share with sync calls. |
| `AIOHTTPClient` | – | ✓ | Async-only fallback when `aiohttp` is installed. |
| `NoImportFoundAsyncClient` | – | ✓ (raises) | Placeholder when no async lib is installed. |

Key constructor signatures:

```python
RequestsClient(
    timeout: Union[float, Tuple[float, float]] = 80,
    session: Optional[requests.Session] = None,
    verify_ssl_certs: bool = True,
    proxy: Optional[Union[str, dict]] = None,
    async_fallback_client: Optional[HTTPClient] = None,
)

HTTPXClient(
    timeout: Optional[Union[float, httpx.Timeout]] = 80,
    allow_sync_methods: bool = False,
    verify_ssl_certs: bool = True,
    proxy: Optional[Union[str, dict]] = None,
)

AIOHTTPClient(
    timeout: Optional[Union[float, aiohttp.ClientTimeout]] = 80,
    verify_ssl_certs: bool = True,
    proxy: Optional[Union[str, dict]] = None,
)

UrllibClient(verify_ssl_certs=True, proxy=None)
PycurlClient(verify_ssl_certs=True, proxy=None)
UrlFetchClient(verify_ssl_certs=True, proxy=None, deadline=55)
```

Auto-selection precedence (`new_default_http_client`, `stripe/_http_client.py:67`):

1. `urlfetch` (Google App Engine)
2. `requests`
3. `pycurl`
4. `urllib` (with a warning)

For async (`new_http_client_async_fallback`, `stripe/_http_client.py:102`):

1. `httpx` + `anyio`
2. `aiohttp`
3. `NoImportFoundAsyncClient` (raises if used)

Plug a custom client into `StripeClient`:

```python
client = StripeClient("sk_test_...", http_client=stripe.HTTPXClient(allow_sync_methods=True))
```

…or set the legacy global:

```python
stripe.default_http_client = stripe.AIOHTTPClient()
```

You can subclass `HTTPClient` for a fully custom transport (e.g. instrumentation). Implement `request`, `request_stream`, `request_async`, `request_stream_async`. The retry/backoff logic lives on `HTTPClient` itself (`request_with_retries`, `request_stream_with_retries`, etc.).

Class-level constants on `HTTPClient`:

```python
HTTPClient.MAX_DELAY        # 5     (max retry backoff seconds)
HTTPClient.INITIAL_DELAY    # 0.5
HTTPClient.MAX_RETRY_AFTER  # 60    (cap on Retry-After header)
```

---

## Logging

Three ways to enable logging, all equivalent:

```sh
# 1. environment variable
export STRIPE_LOG=info   # or "debug"
```

```python
# 2. set the global
import stripe
stripe.log = "debug"
```

```python
# 3. standard logging module
import logging
logging.basicConfig()
logging.getLogger("stripe").setLevel(logging.DEBUG)
```

`info` is appropriate for production; `debug` includes full request/response details.

---

## Telemetry

By default the SDK sends timing information about the previous request as an `X-Stripe-Client-Telemetry` header on the next request. Disable it if you want:

```python
import stripe
stripe.enable_telemetry = False
```

Telemetry contains the previous request's id, latency, and usage tags — no payload data.

---

## Common resources at a glance

Every method below is also available with an `_async` suffix on the service. Where parameters are listed, they are the most common ones; for the full list see `stripe/params/_<resource>_<op>_params.py`.

### Customer

```python
client.v1.customers.create(params={
    "email": "ada@example.com",
    "name": "Ada Lovelace",
    "metadata": {"plan": "pro"},
})

client.v1.customers.retrieve("cus_...", params={"expand": ["subscriptions"]})

client.v1.customers.update("cus_...", params={"email": "new@example.com"})

client.v1.customers.delete("cus_...")

client.v1.customers.list(params={"email": "ada@example.com", "limit": 10})

client.v1.customers.search(params={"query": "email:'ada@example.com'"})

# nested services
client.v1.customers.payment_methods.list("cus_...", params={"type": "card"})
client.v1.customers.tax_ids.create("cus_...", params={"type": "us_ein", "value": "..."})
client.v1.customers.balance_transactions.create("cus_...", params={"amount": -500, "currency": "usd"})
```

Customer is `Createable | Deletable | Listable | Searchable | Updateable`. The legacy global API exposes `stripe.Customer.create(...)`, `stripe.Customer.retrieve(id)`, `stripe.Customer.modify(id, ...)`, `stripe.Customer.delete(id)`, `stripe.Customer.list(...)`, `stripe.Customer.search(...)`, plus class methods for the same nested operations (`stripe.Customer.list_payment_methods(...)`, `stripe.Customer.create_funding_instructions(...)`, etc.). **Source:** `stripe/_customer.py`.

### PaymentIntent

The flow: **create → confirm → capture (if manual)** or **create → confirm** (automatic capture).

```python
intent = client.v1.payment_intents.create(params={
    "amount": 2000,
    "currency": "usd",
    "automatic_payment_methods": {"enabled": True},
    "customer": "cus_...",
    "metadata": {"order_id": "ORD-9821"},
})

# After collecting payment details on the front end…
client.v1.payment_intents.confirm(intent.id, params={
    "payment_method": "pm_card_visa",
    "return_url": "https://example.com/return",
})

# Manual capture flow
intent = client.v1.payment_intents.create(params={
    "amount": 2000, "currency": "usd",
    "capture_method": "manual",
    "payment_method": "pm_card_visa",
    "confirm": True,
})
client.v1.payment_intents.capture(intent.id)
client.v1.payment_intents.cancel(intent.id, params={"cancellation_reason": "requested_by_customer"})

# Other operations
client.v1.payment_intents.list(params={"customer": "cus_...", "limit": 10})
client.v1.payment_intents.search(params={"query": "status:'succeeded' AND amount>1000"})
client.v1.payment_intents.update(intent.id, params={"description": "Refurbished order"})
client.v1.payment_intents.apply_customer_balance(intent.id)
client.v1.payment_intents.increment_authorization(intent.id, params={"amount": 2500})
client.v1.payment_intents.verify_microdeposits(intent.id, params={"amounts": [32, 45]})
```

`PaymentIntent.status` walks through `requires_payment_method`, `requires_confirmation`, `requires_action`, `processing`, `requires_capture`, `succeeded`, `canceled`. The `client_secret` is what you hand to Stripe.js / Elements / mobile SDKs — never expose your API key to the front end.

**Source:** `stripe/_payment_intent_service.py`.

### Charge

Charges are still supported but the **PaymentIntents API is the recommended way** to take new payments. Charges are most useful for reading historical data and creating refunds.

```python
client.v1.charges.retrieve("ch_...")
client.v1.charges.list(params={"customer": "cus_...", "limit": 5})
client.v1.charges.update("ch_...", params={"description": "Updated"})
client.v1.charges.capture("ch_...", params={"amount": 2000})  # only for manual capture charges
client.v1.charges.search(params={"query": "amount>1000"})
```

### Refund

```python
refund = client.v1.refunds.create(params={
    "charge": "ch_...",
    "amount": 500,                       # partial refund (in cents)
    "reason": "requested_by_customer",
    "metadata": {"reason": "duplicate"},
})

client.v1.refunds.retrieve(refund.id)
client.v1.refunds.update(refund.id, params={"metadata": {"reviewed": "true"}})
client.v1.refunds.list(params={"charge": "ch_..."})
client.v1.refunds.cancel(refund.id)   # only when status="requires_action"
```

`reason` is one of `duplicate`, `fraudulent`, `requested_by_customer`, `expired_uncaptured_charge`. You can also refund a `payment_intent` instead of a `charge`.

### Subscription

```python
sub = client.v1.subscriptions.create(params={
    "customer": "cus_...",
    "items": [{"price": "price_..."}],
    "payment_behavior": "default_incomplete",
    "expand": ["latest_invoice.payment_intent"],
})

client.v1.subscriptions.retrieve(sub.id)
client.v1.subscriptions.update(sub.id, params={
    "items": [{"id": sub["items"]["data"][0]["id"], "price": "price_other"}],
    "proration_behavior": "create_prorations",
})
client.v1.subscriptions.cancel(sub.id, params={"prorate": True, "invoice_now": False})
client.v1.subscriptions.resume(sub.id)        # if paused
client.v1.subscriptions.list(params={"customer": "cus_...", "status": "active"})
client.v1.subscriptions.search(params={"query": "status:'active' AND created>1700000000"})

# Subscription items
client.v1.subscription_items.create(params={"subscription": sub.id, "price": "price_addon"})
client.v1.subscription_items.update("si_...", params={"quantity": 5})
client.v1.subscription_items.delete("si_...")

# Subscription schedules (phased changes)
schedule = client.v1.subscription_schedules.create(params={
    "from_subscription": sub.id,
})
```

### Invoice

```python
invoice = client.v1.invoices.create(params={
    "customer": "cus_...",
    "auto_advance": True,
    "collection_method": "charge_automatically",
})

client.v1.invoice_items.create(params={
    "customer": "cus_...",
    "invoice": invoice.id,
    "amount": 1500,
    "currency": "usd",
    "description": "Setup fee",
})

client.v1.invoices.finalize_invoice(invoice.id)
client.v1.invoices.pay(invoice.id, params={"payment_method": "pm_..."})
client.v1.invoices.send_invoice(invoice.id)              # for collection_method="send_invoice"
client.v1.invoices.void_invoice(invoice.id)
client.v1.invoices.mark_uncollectible(invoice.id)
client.v1.invoices.upcoming(params={"customer": "cus_..."})  # preview the next invoice

# Line items
client.v1.invoices.list_line_items(invoice.id)

# Search
client.v1.invoices.search(params={"query": "status:'paid'"})
```

### Product and Price

```python
product = client.v1.products.create(params={
    "name": "Pro Plan",
    "description": "Everything you need",
})

# Recurring price (subscription)
price = client.v1.prices.create(params={
    "product": product.id,
    "currency": "usd",
    "unit_amount": 2000,
    "recurring": {"interval": "month"},
})

# One-off price
client.v1.prices.create(params={
    "product": product.id,
    "currency": "usd",
    "unit_amount": 4900,
})

client.v1.prices.list(params={"product": product.id, "active": True})
client.v1.products.search(params={"query": "active:'true' AND name:'Pro Plan'"})
```

### Checkout Session

The hosted Stripe Checkout flow lives under `client.v1.checkout`:

```python
session = client.v1.checkout.sessions.create(params={
    "mode": "payment",     # or "subscription", "setup"
    "line_items": [{"price": "price_...", "quantity": 1}],
    "success_url": "https://example.com/success?sid={CHECKOUT_SESSION_ID}",
    "cancel_url": "https://example.com/cancel",
    "customer_email": "ada@example.com",
})

print(session.url)        # redirect the user to this URL

# Later
client.v1.checkout.sessions.retrieve(session.id, params={"expand": ["line_items"]})
client.v1.checkout.sessions.list_line_items(session.id)
client.v1.checkout.sessions.expire(session.id)
```

### PaymentMethod

```python
client.v1.payment_methods.create(params={
    "type": "card",
    "card": {"token": "tok_visa"},   # or use Stripe.js / Elements on the front end
})

client.v1.payment_methods.attach("pm_...", params={"customer": "cus_..."})
client.v1.payment_methods.detach("pm_...")
client.v1.payment_methods.update("pm_...", params={"billing_details": {"email": "..."}})
client.v1.payment_methods.list(params={"customer": "cus_...", "type": "card"})
```

### SetupIntent

For collecting payment details to charge later (off-session), use a SetupIntent:

```python
setup = client.v1.setup_intents.create(params={
    "customer": "cus_...",
    "automatic_payment_methods": {"enabled": True},
    "usage": "off_session",
})

client.v1.setup_intents.confirm(setup.id, params={"payment_method": "pm_..."})
client.v1.setup_intents.cancel(setup.id, params={"cancellation_reason": "abandoned"})
client.v1.setup_intents.verify_microdeposits(setup.id, params={"amounts": [32, 45]})
```

### Payouts, Transfers, and Balance

```python
# Your platform balance
client.v1.balance.retrieve()
client.v1.balance_transactions.list(params={"limit": 10})

# Payouts to your bank account
client.v1.payouts.create(params={
    "amount": 5000,
    "currency": "usd",
    "method": "instant",
})
client.v1.payouts.list(params={"status": "paid"})
client.v1.payouts.cancel("po_...")
client.v1.payouts.reverse("po_...")

# Transfers to connected accounts (Stripe Connect)
client.v1.transfers.create(params={
    "amount": 1000,
    "currency": "usd",
    "destination": "acct_...",
    "transfer_group": "ORDER_94",
})
```

---

## Common patterns

### Listing all results

```python
# Sync — same call works in async too
for customer in client.v1.customers.list(params={"limit": 100}).auto_paging_iter():
    print(customer.id)

# Async
async for customer in (
    await client.v1.customers.list_async(params={"limit": 100})
).auto_paging_iter():
    print(customer.id)
```

`limit` defaults to 10 and maxes out at 100 server-side. Bigger pages = fewer round-trips when you intend to walk the whole result set.

### Search

Search uses Stripe's [Search Query Language](https://docs.stripe.com/search#search-query-language):

```python
# Customers with a specific email
result = client.v1.customers.search(params={"query": "email:'ada@example.com'"})

# Subscriptions created after a Unix timestamp, status active
result = client.v1.subscriptions.search(params={
    "query": "status:'active' AND created>1700000000",
    "limit": 50,
})

for sub in result.auto_paging_iter():
    ...
```

Search is **eventually consistent** (typically <1 minute lag). Don't use it for read-after-write.

### Idempotent retries

```python
# Wrap a critical create with a stable idempotency key,
# so a retry won't double-charge.
intent = client.v1.payment_intents.create(
    params={"amount": 2000, "currency": "usd", "customer": cus_id},
    options={"idempotency_key": f"order:{order_id}:intent"},
)
```

For automatic *transport-level* retries (which the SDK adds an internally generated key to), set `max_network_retries`. The two are complementary — your key keeps client-side retries safe, the SDK's keeps its own retries safe.

### Connected accounts (`Stripe-Account` header)

```python
# At construction (sticky for the whole client)
acct_client = StripeClient("sk_test_...", stripe_account="acct_1ABCxyz")

# Per request (overrides the client default)
client.v1.charges.list(options={"stripe_account": "acct_1ABCxyz"})

# Build a customer on a connected account
client.v1.customers.create(
    params={"email": "buyer@example.com"},
    options={"stripe_account": "acct_1ABCxyz"},
)
```

This is how you act *as* a connected account on Stripe Connect. The connected account id is returned by the OAuth flow as `stripe_user_id`.

### Metering events (high-throughput)

For usage-based billing meters, regular `client.v1.billing.meters.*` calls work fine for low volume; for high-throughput ingestion, the V2 meter event stream uses a session token and a separate base URL:

```python
session = client.v2.billing.meter_event_session.create()

# Use the session-scoped token as the API key for the stream
stream_client = StripeClient(session["authentication_token"])
stream_client.v2.billing.meter_event_stream.create(params={
    "events": [{
        "event_name": "alpaca_ai_tokens",
        "payload": {"stripe_customer_id": "cus_...", "value": "25"},
    }],
})
```

Sessions expire — re-create when `session.expires_at` passes. **Source:** `examples/meter_event_stream.py`.

### Test clocks and test helpers

Every test-only operation lives under `client.v1.test_helpers.*`:

```python
clock = client.v1.test_helpers.test_clocks.create(
    params={"frozen_time": 1730000000, "name": "billing_test"},
)

# Create a customer attached to the clock
cus = client.v1.customers.create(params={
    "email": "buyer@example.com",
    "test_clock": clock.id,
})

# Advance time on the clock to fast-forward through subscription cycles
client.v1.test_helpers.test_clocks.advance(
    clock.id,
    params={"frozen_time": 1730000000 + 60 * 60 * 24 * 31},
)

# Other helpers
client.v1.test_helpers.refunds.expire("re_...")              # force-expire pending refunds
client.v1.test_helpers.customers.fund_cash_balance("cus_...", params={"amount": 5000, "currency": "usd"})
client.v1.test_helpers.issuing.cards.deliver_card("ic_...")  # simulate card delivery
client.v1.test_helpers.terminal.readers.present_payment_method("tmr_...")
```

These only work with test-mode keys (`sk_test_...`).
