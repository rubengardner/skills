# Specialist G — State & Caching

## THE CORE CONFLICT

The backend caches for server-side performance. The client caches for perceived performance and offline access. These caches can go out of sync — and when they do, the user sees stale data.

---

## MEDIATION

**⚙ BACKEND SAYS**

> I cache `GET /products` in Redis for 5 minutes. That's 300 seconds of reduced database load. If you update a product, the cache is stale for up to 5 minutes. That's acceptable. I'm not going to add cache invalidation logic for every write operation — cache invalidation is famously hard, and the performance benefit of caching outweighs 5 minutes of staleness for a product listing.

**📱 CLIENT SAYS**

> The user updates a product name, the API returns 200, the UI re-fetches the product list, and shows the old name for 5 minutes because the backend cache hasn't expired. The user thinks their update didn't work. They submit again. Now you have a duplicate edit. The UX is broken. I need either: immediate cache invalidation when I write, or the ability to do an optimistic update locally so the UI reflects the change even if the server is slow to agree.

**⚡ THE REAL TENSION**

Backend caching optimizes server performance but introduces visible staleness. Optimistic updates solve the UX problem but create a divergence risk if the server rejects the write.

---

## PATTERN 1: CACHE INVALIDATION ON WRITE

Invalidate the cache immediately when the resource is updated:

```python
import django_redis

cache = django_redis.get_redis_connection("default")

@router.put("/products/{product_id}")
def update_product(request, product_id: str, payload: UpdateProductSchema):
    product = product_service.update(product_id, payload)

    # Invalidate affected caches immediately after write
    cache.delete(f"products:list")           # the list
    cache.delete(f"products:{product_id}")   # the detail

    return to_product_response(product)

@router.get("/products")
def list_products(request):
    cache_key = "products:list"
    cached = cache.get(cache_key)
    if cached:
        return cached

    products = product_repo.list_active()
    response = [to_product_summary(p) for p in products]
    cache.setex(cache_key, 300, response)  # 5-min TTL, auto-expires as fallback
    return response
```

**Trade-off:** Cache is always consistent after writes. Requires knowing which cache keys to invalidate. For complex graph-like data with many relationships, invalidation logic can grow complex.

---

## PATTERN 2: OPTIMISTIC UPDATES (client-side)

The client updates its local state immediately on user action, before the server confirms. If the server returns an error, it rolls back.

```typescript
// React Query optimistic update pattern
const queryClient = useQueryClient();

const updateProduct = useMutation({
  mutationFn: (data: UpdateProductInput) =>
    api.put(`/products/${data.id}`, data),

  onMutate: async (newData) => {
    // Cancel any in-flight refetch
    await queryClient.cancelQueries({ queryKey: ["products"] });

    // Snapshot the previous value
    const previousProducts = queryClient.getQueryData(["products"]);

    // Optimistically update the cache
    queryClient.setQueryData(["products"], (old: Product[]) =>
      old.map((p) => (p.id === newData.id ? { ...p, ...newData } : p))
    );

    return { previousProducts };  // returned as context
  },

  onError: (error, newData, context) => {
    // Roll back on failure
    queryClient.setQueryData(["products"], context?.previousProducts);
    showErrorToast("Update failed. Your changes have been reverted.");
  },

  onSettled: () => {
    // Refetch to sync with server truth after success or failure
    queryClient.invalidateQueries({ queryKey: ["products"] });
  },
});
```

**Trade-off:** Instant perceived UX — the user sees their change immediately. If the server rejects the write (validation error, conflict), the rollback can feel jarring. Use only for low-risk writes (name changes, descriptions). Do not use for writes with side effects (payments, inventory reservation).

---

## PATTERN 3: ETAG / CONDITIONAL REQUESTS

Backend sends an `ETag` with each response. Client sends `If-None-Match` on refetch. Backend returns 304 Not Modified if nothing changed — saves bandwidth and processing.

```python
import hashlib
import json

@router.get("/products/{product_id}")
def get_product(request, product_id: str):
    product = product_repo.get(product_id)
    response_data = to_product_response(product)

    # Generate ETag from content
    content = json.dumps(response_data.model_dump(), sort_keys=True)
    etag = hashlib.md5(content.encode()).hexdigest()

    if request.headers.get("If-None-Match") == etag:
        return HttpResponse(status=304)  # Not Modified

    response = JsonResponse(response_data.model_dump())
    response["ETag"] = etag
    response["Cache-Control"] = "private, max-age=60"
    return response
```

**Trade-off:** Reduces bandwidth and server load on refetch. Client must handle 304 responses. Best for high-read, low-write resources.

---

## STALENESS RULES

Agree on a staleness budget for each resource type:

| Resource | Acceptable staleness | Strategy |
|----------|---------------------|----------|
| User's own profile | 0s | Invalidate on write |
| Product listing (public) | 5 min | TTL cache |
| Order status | 0s | No cache, or SSE push |
| Search results | 30s–5 min | TTL cache |
| Static content (categories) | 1 hour | Long TTL + CDN |
| Notifications | 0s | SSE push |

---

## CONFLICT RESOLUTION (concurrent edits)

When two users (or tabs) edit the same resource simultaneously:

```python
# Optimistic locking via version field
class UpdateProductSchema(BaseModel):
    name: str
    version: int  # client sends the version it last saw

@router.put("/products/{product_id}")
def update_product(request, product_id: str, payload: UpdateProductSchema):
    updated = Product.objects.filter(
        id=product_id,
        version=payload.version  # only update if version matches
    ).update(
        name=payload.name,
        version=F("version") + 1,
    )

    if updated == 0:
        return 409, ErrorResponse(
            code="CONFLICT",
            message="This record was modified by someone else. Please refresh and try again.",
            meta={"current_version": product_repo.get_version(product_id)},
        )

    return to_product_response(product_repo.get(product_id))
```

**The decision is yours** on which cache strategy fits the staleness budget for each resource.
