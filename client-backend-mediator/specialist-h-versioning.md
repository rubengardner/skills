# Specialist H — API Versioning & Migration

## THE CORE CONFLICT

The backend wants to evolve freely. The client cannot be instantly updated everywhere — especially mobile. Breaking the contract breaks the app.

---

## MEDIATION

**⚙ BACKEND SAYS**

> I renamed `customer_name` to `full_name` because that's what the domain model calls it. It's a two-second change. The frontend can update their code in the same PR. We're a monolith — we deploy together. Stop acting like we're maintaining a public API.

**📱 CLIENT SAYS**

> If this is a web app, sure. But if there's a mobile app, that field name change just broke every installed version of the app that hasn't updated yet — which could be 30% of users. And even for web: "deploy together" works until it doesn't. A deploy that's 50% old backend, 50% new backend (rolling deploy, blue-green, canary) will have the old client talking to the new backend or vice versa. Field renames are breaking changes. Treat them as such.

**⚡ THE REAL TENSION**

The backend thinks "we deploy together" eliminates versioning concerns. It does not — rolling deploys, mobile clients, and cached responses all break this assumption.

---

## WHAT IS A BREAKING CHANGE

```
BREAKING (never do without a migration plan):
  - Remove a field from a response
  - Rename a field
  - Change a field's type (string → number, object → array)
  - Change the meaning of an existing enum value
  - Change a 200 to a 4xx for a previously valid request
  - Change required vs optional on a request field (making optional → required)
  - Remove an endpoint

NON-BREAKING (safe to do at any time):
  - Add a new optional field to a response
  - Add a new endpoint
  - Add a new enum value (client must handle unknown values)
  - Make a previously required request field optional
  - Add a new optional request parameter
```

---

## STRATEGY 1: EXPAND-CONTRACT (preferred for monoliths)

Never rename — add the new field alongside the old one. Remove the old field only after all clients have migrated.

```python
# Phase 1: Add new field, keep old one (deploy)
class CustomerDTO(BaseModel):
    customer_name: str   # DEPRECATED — will be removed in 3 months
    full_name: str       # new canonical field

# Phase 2: Client migrates to full_name (deploy)
# Phase 3: Remove customer_name (deploy — only after mobile sunset window)
class CustomerDTO(BaseModel):
    full_name: str
```

**Add a deprecation comment and a removal date.** Without a deadline, deprecated fields live forever.

```python
class CustomerDTO(BaseModel):
    customer_name: str = Field(
        ...,
        description="DEPRECATED: use full_name. Will be removed 2026-06-01.",
    )
    full_name: str
```

---

## STRATEGY 2: URL VERSIONING (for true breaking changes)

When expand-contract isn't enough (structural changes, new auth model, fundamentally different response), add a version prefix:

```
/v1/orders/{id}  → old contract, maintained
/v2/orders/{id}  → new contract
```

```python
# Django Ninja — versioned router
v1_router = Router(prefix="/v1")
v2_router = Router(prefix="/v2")

@v1_router.get("/orders/{order_id}")
def get_order_v1(request, order_id: str):
    order = order_repo.get(order_id)
    return to_v1_response(order)  # old shape

@v2_router.get("/orders/{order_id}")
def get_order_v2(request, order_id: str):
    order = order_repo.get(order_id)
    return to_v2_response(order)  # new shape
```

**Set a sunset date** for v1. Use headers:

```python
def add_deprecation_headers(response, sunset_date: str, successor_url: str):
    response["Deprecation"] = "true"
    response["Sunset"] = sunset_date          # "Sat, 01 Jun 2026 00:00:00 GMT"
    response["Link"] = f'<{successor_url}>; rel="successor-version"'
```

---

## STRATEGY 3: VERSION HEADER (alternative to URL versioning)

```
GET /orders/123
Accept: application/vnd.myapp.v2+json
```

```python
@router.get("/orders/{order_id}")
def get_order(request, order_id: str):
    version = request.headers.get("Accept", "").split("v")[-1].split("+")[0] or "1"

    order = order_repo.get(order_id)
    if version == "2":
        return to_v2_response(order)
    return to_v1_response(order)
```

**Trade-off:** Cleaner URLs. Harder to test in browser. Harder for clients to reason about. URL versioning is preferred for discoverability.

---

## MOBILE SUNSET POLICY

For mobile apps specifically, you cannot force an update. Define a policy before you need it:

```
1. When releasing a breaking change:
   - New version goes to /v2
   - /v1 is maintained with "Sunset" header
   - Sunset date = current date + 6 months minimum

2. Force-upgrade gate (optional):
   - Backend returns a minimum required app version in a response header:
     X-Min-App-Version: 2.4.0
   - Mobile app checks this header and shows a "please update" screen if below minimum
   - Use sparingly — it's a hostile UX
```

```python
# Backend: inject minimum app version header
MINIMUM_APP_VERSIONS = {
    "ios": "2.4.0",
    "android": "2.4.0",
}

class AppVersionMiddleware:
    def __call__(self, request):
        response = self.get_response(request)
        platform = request.headers.get("X-App-Platform")
        if platform in MINIMUM_APP_VERSIONS:
            response["X-Min-App-Version"] = MINIMUM_APP_VERSIONS[platform]
        return response
```

---

## ROLLING DEPLOY SAFETY

During a rolling deploy (new backend, old backend running simultaneously), both old and new backend must be able to handle requests. Contract rules:

```
Old backend must handle new client requests:
  - New optional request fields → ignored by old backend ✓
  - New required request fields → old backend returns 422 ✗ (breaking — don't do this)

New backend must handle old client requests:
  - Old client sends old field names → new backend must still accept them
  - Old client doesn't send new optional fields → new backend must have defaults
```

```python
# Safe during rolling deploy: new field is optional with default
class CreateOrderSchema(BaseModel):
    product_id: str
    quantity: int
    priority: str = "normal"  # new field — old clients don't send it, defaults to "normal"
```

---

## OPTIONS SUMMARY

| Scenario | Strategy |
|----------|----------|
| Rename a field | Expand-contract: add new, deprecate old, remove after sunset |
| Small semantic change | Expand-contract |
| Large structural change | URL versioning /v1 → /v2 |
| Mobile app can't be forced to update | Sunset policy + minimum version gate |
| Rolling deploy safety | All new request fields must be optional with defaults |

**The decision is yours.**
