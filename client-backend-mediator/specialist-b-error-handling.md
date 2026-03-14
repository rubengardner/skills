# Specialist B — Error Handling & Validation

## THE CORE CONFLICT

The backend classifies errors by cause (domain exception, validation failure, auth error).
The client classifies errors by what to show the user (field error, toast, redirect, retry button).
These taxonomies do not map 1:1.

---

## MEDIATION

**⚙ BACKEND SAYS**

> I raise `InsufficientInventoryError`, `OrderAlreadyClosedError`, `InvalidCouponError`. These are precise domain exceptions. They carry structured data. I return them as 422s with a `type` and `detail`. The client should read the `type` field and decide what to display. That's clean. I'm not in the business of writing UI copy.

**📱 CLIENT SAYS**

> I get a 422 with `{"type": "InsufficientInventoryError", "detail": "qty > stock"}`. I need to show the user: "Sorry, only 3 of this item are available." I have to parse `detail`, regex-extract the number, and write copy. This is fragile. Either give me machine-readable data (`available_qty: 3`) so I can format the message myself, or give me a human-readable `message` I can display directly. Preferably both.

**⚡ THE REAL TENSION**

Backend wants to expose error semantics. Client needs error display data. Neither alone is sufficient.

---

## CANONICAL ERROR RESPONSE SHAPE

Agree on this contract and never deviate:

```json
{
  "code": "INSUFFICIENT_INVENTORY",
  "message": "Only 3 units of this item are available.",
  "field": null,
  "meta": {
    "available_qty": 3,
    "requested_qty": 10,
    "product_id": "prod_abc"
  }
}
```

| Field | Backend owns | Client uses |
|-------|-------------|-------------|
| `code` | Domain error identifier (SCREAMING_SNAKE_CASE) | Switch statement for routing / i18n lookup |
| `message` | Default English message | Display directly in simple apps; override with i18n in complex apps |
| `field` | Which input field caused the error (`null` if not field-specific) | Highlight the form field |
| `meta` | Machine-readable context (counts, IDs, limits) | Format custom messages, populate UI state |

---

## VALIDATION ERRORS (multiple fields)

```json
{
  "code": "VALIDATION_ERROR",
  "message": "The request contains invalid fields.",
  "field": null,
  "errors": [
    {
      "code": "REQUIRED",
      "message": "Email is required.",
      "field": "email"
    },
    {
      "code": "TOO_SHORT",
      "message": "Password must be at least 8 characters.",
      "field": "password",
      "meta": { "min_length": 8, "actual_length": 5 }
    }
  ]
}
```

**Implementation (Django Ninja):**

```python
from ninja import Router
from ninja.errors import ValidationError
from pydantic import ValidationError as PydanticValidationError

@router.post("/orders")
def create_order(request, payload: CreateOrderSchema):
    try:
        order = order_service.create(payload)
        return 201, OrderDetailResponse.from_domain(order)
    except InsufficientInventoryError as e:
        return 422, ErrorResponse(
            code="INSUFFICIENT_INVENTORY",
            message=f"Only {e.available_qty} units available.",
            meta={"available_qty": e.available_qty, "requested_qty": e.requested_qty},
        )
    except OrderAlreadyClosedError:
        return 409, ErrorResponse(
            code="ORDER_ALREADY_CLOSED",
            message="This order has already been completed and cannot be modified.",
        )
```

---

## HTTP STATUS CODE CONTRACT

| Situation | Status | Code |
|-----------|--------|------|
| Input doesn't pass schema validation | 422 | `VALIDATION_ERROR` |
| Input is valid but violates business rules | 422 | Specific domain code |
| Resource not found | 404 | `NOT_FOUND` |
| Already exists / conflict | 409 | `CONFLICT` |
| Not authenticated | 401 | `UNAUTHENTICATED` |
| Authenticated but no permission | 403 | `FORBIDDEN` |
| Unexpected server error | 500 | `INTERNAL_ERROR` |

**Rules:**
- Never return 200 with `{"success": false}`. Use the correct status code.
- Never return 400 for domain/business rule violations. 400 means the request is malformed. Use 422 for semantic errors.
- 500 responses must not include stack traces. Log them server-side, return a correlation ID.

```python
# 500 handler — safe for clients
@app.exception_handler(Exception)
def unhandled_exception(request, exc):
    correlation_id = generate_correlation_id()
    logger.error("unhandled_exception", correlation_id=correlation_id, error=str(exc))
    return JsonResponse({
        "code": "INTERNAL_ERROR",
        "message": "An unexpected error occurred. Please try again.",
        "correlation_id": correlation_id,  # client can report this
    }, status=500)
```

---

## OPTIONS FOR THE TEAM

### Option 1: Backend owns messages, client displays directly
Backend writes the `message` field. Client renders it verbatim.
**Trade-off:** Fast for simple apps. Breaks for i18n. Backend writes UI copy, which feels wrong.

### Option 2: Client owns messages, backend provides codes + meta
Backend returns `code` and `meta`. Client has an error message map.
```typescript
const ERROR_MESSAGES: Record<string, (meta: any) => string> = {
  INSUFFICIENT_INVENTORY: (m) => `Only ${m.available_qty} units available.`,
  ORDER_ALREADY_CLOSED: () => "This order can no longer be modified.",
};
```
**Trade-off:** Client has full control over copy and i18n. Backend and frontend must stay in sync on `code` values. Add new backend errors → update frontend map.

### Option 3: Both (recommended)
Backend provides a default `message` AND machine-readable `code` + `meta`. Client uses `message` for simplicity, overrides with i18n when needed.
**Trade-off:** Slightly more backend work. Maximum flexibility.

**The decision is yours.**
