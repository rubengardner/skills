# Specialist F — Mobile Constraints

## MOBILE IS NOT A SUBSET OF WEB

Mobile clients have constraints that web clients don't: intermittent connectivity, bandwidth cost, battery consumption, app store versioning (you cannot force users to update), and radically different UX expectations around loading states.

---

## MEDIATION

**⚙ BACKEND SAYS**

> Mobile calls the same API as the web app. Why do I need to treat it differently? JSON is JSON. If you want a smaller payload, compress it. The endpoints are the same. I'm not maintaining two different APIs.

**📱 CLIENT SAYS**

> On mobile, every byte costs money (metered data), every radio wakeup costs battery, every dropped connection needs offline fallback. The web app never goes offline. The mobile app does — on the subway, in a tunnel, in a building basement. I need: smaller payloads, offline support, a graceful degradation mode, and — critically — I cannot force the user to update the app. If you rename a field, old app versions break. You need to be much more careful about backward compatibility than you are with the web app.

**⚡ THE REAL TENSION**

The web API is designed for web constraints (fast connection, always online, easy deployment). Mobile has fundamentally different constraints that the API must accommodate without duplicating it.

---

## 1. PAYLOAD SIZE

Mobile should receive the minimum viable payload. Apply these in priority order:

```python
# 1. Always enable compression
# Django: django-compression-middleware or nginx gzip

# 2. Return only necessary fields by default
class OrderSummaryMobileDTO(BaseModel):
    id: str
    status: str            # not the full status object
    total: str             # pre-formatted string, not Decimal
    item_count: int        # not the items list
    # No: customer_email, shipping_address, billing_info, internal_notes

# 3. Use integer/short IDs in list responses (full UUIDs only in detail)
# 36-char UUID vs 8-char short ID: 4.5x difference per record, multiplied by N rows

# 4. Dates as Unix timestamps for list responses (smaller than ISO strings)
# "2024-03-14T12:00:00Z" = 20 chars
# 1710417600 = 10 chars (50% smaller)
```

---

## 2. OFFLINE SUPPORT

Design for offline-first: the app works without a connection, syncs when connectivity returns.

### Backend responsibilities for offline support:

```python
# Every resource must have a version/timestamp the client can use for sync
class OrderSyncDTO(BaseModel):
    id: str
    updated_at: datetime   # client compares this to local cache
    checksum: str | None   # optional: MD5 of content for change detection

# Sync endpoint: client sends its last sync time, backend returns only changed records
@router.get("/orders/sync")
def sync_orders(request, since: str):
    since_dt = datetime.fromisoformat(since)
    changed = Order.objects.filter(
        user=request.user,
        updated_at__gt=since_dt,
    ).select_related("customer").prefetch_related("items")

    deleted = DeletedOrder.objects.filter(
        user=request.user,
        deleted_at__gt=since_dt,
    )

    return {
        "updated": [to_sync_dto(o) for o in changed],
        "deleted": [str(d.order_id) for d in deleted],
        "server_time": datetime.utcnow().isoformat(),  # use as next `since`
    }
```

### Client responsibilities:
- Store all fetched data in local SQLite / IndexedDB
- Queue writes made while offline
- Replay the queue when connectivity returns
- Show "last synced at" indicator to user

---

## 3. BACKWARD COMPATIBILITY (the most critical mobile constraint)

The web app can be updated with a deploy. A mobile app can only be updated when the user chooses. Old app versions will call your API months after you've changed it.

### Rules that are non-negotiable for mobile APIs:

```python
# NEVER remove a field — mark it deprecated, keep returning it
class OrderDTO(BaseModel):
    id: str
    status: str
    status_label: str  # DEPRECATED — kept for app v1.x, remove after v1.x sunset

# NEVER change a field's type
# was: "amount": "99.99" (string)
# now: "amount": 99.99 (number)  ← breaks old clients that did parseFloat()
# Solution: add a new field, keep the old one

# NEVER rename a field
# was: "customer_name": "Jane"
# now: "name": "Jane"  ← breaks all old clients
# Solution: return both for a transition period

# NEVER change an enum value's meaning
# was: status = "pending" means "awaiting payment"
# now: status = "pending" means "awaiting fulfillment" ← semantic change, silent bug

# SAFE: add new optional fields (old clients ignore them)
# SAFE: add new endpoints
# SAFE: add new enum values (old clients must handle unknown values gracefully)
```

### API versioning for breaking changes:

```python
# URL versioning — explicit, unambiguous
@router.get("/v1/orders/{id}")   # old contract, maintained for old app versions
def get_order_v1(request, id): ...

@router.get("/v2/orders/{id}")   # new contract
def get_order_v2(request, id): ...

# Sunset header: tells clients when the old version will be removed
response["Sunset"] = "Sat, 01 Jan 2026 00:00:00 GMT"
response["Deprecation"] = "true"
response["Link"] = '</v2/orders/{id}>; rel="successor-version"'
```

---

## 4. CONNECTIVITY HANDLING

The backend must handle idempotent retries from mobile clients gracefully:

```python
# Mobile retries on network failure — backend must handle duplicate requests
@router.post("/orders")
def create_order(request, payload: CreateOrderSchema):
    # Idempotency key sent by mobile client
    idempotency_key = request.headers.get("Idempotency-Key")

    if idempotency_key:
        # Check if this request was already processed
        cached = IdempotencyCache.get(idempotency_key)
        if cached:
            return cached["status_code"], cached["response"]  # return same response

    order = order_service.create(payload)
    response = to_order_response(order)

    if idempotency_key:
        IdempotencyCache.set(idempotency_key, {
            "status_code": 201,
            "response": response.model_dump(),
        }, ttl=86400)  # keep for 24h

    return 201, response
```

```swift
// iOS client: generate idempotency key per user action
let idempotencyKey = UUID().uuidString  // generated once per button tap
// Retry the request with the same key if it fails
```

---

## 5. OPTIONS SUMMARY

### Option 1: One API, mobile-aware
Same endpoints, but designed with mobile constraints in mind from the start: small payloads, backward-compat rules, idempotency keys, sync endpoint.
**Trade-off:** Small added complexity. No duplication. The right choice for most monoliths.

### Option 2: BFF (Backend for Frontend)
A thin API layer dedicated to the mobile client that transforms the core API into mobile-optimized payloads.
**Trade-off:** Full control over mobile responses. Extra service to maintain. Only warranted if mobile and web needs diverge significantly.

### Option 3: GraphQL
Client specifies exactly which fields it needs. One endpoint, flexible queries, no over/under-fetching.
**Trade-off:** Steep setup cost. Works well for mobile. Introduces complexity in auth, caching, and error handling. Consider only if you have many client types with very different needs.

**The decision is yours.**
