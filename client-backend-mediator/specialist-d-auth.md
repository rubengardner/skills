# Specialist D — Auth & Session

## THE CORE CONFLICT

The backend wants tokens to expire so compromised credentials stop working.
The client wants users to never see a login screen unless they explicitly log out.

---

## MEDIATION

**⚙ BACKEND SAYS**

> Access tokens expire for a reason — security. If a token is compromised, short expiry limits the blast radius. I issue a 15-minute access token and a 7-day refresh token. If the client's access token expires mid-session, it calls my refresh endpoint. Simple. Secure. Standard.

**📱 CLIENT SAYS**

> The user is filling out a long form. They pause for 20 minutes. They submit. I get a 401. Now I need to: detect the 401, call the refresh endpoint, retry the original request, and if the refresh is also expired, redirect to login. If I don't do this correctly, the user's form data is lost, and they see an unexplained "logged out" message. This is a UX disaster. Give me a silent refresh before expiry so the user never sees it.

**⚡ THE REAL TENSION**

Short token expiry is correct for security. Transparent silent refresh is required for UX. Both are achievable simultaneously — it just requires an interceptor pattern.

---

## CANONICAL AUTH PATTERN

### Token storage (web)

```
Access token:  Memory only (JS variable or React state) — never localStorage
Refresh token: HttpOnly cookie (not accessible to JS — XSS safe)
```

**Never store access tokens in `localStorage`.** XSS can steal them. Store the refresh token in an HttpOnly cookie; the backend sets it via `Set-Cookie`. The access token lives only in memory and is re-fetched on page load via the refresh endpoint.

```python
# Django view: login response
def login(request, credentials: LoginSchema):
    user = authenticate(credentials)
    access_token = generate_access_token(user, expires_in=timedelta(minutes=15))
    refresh_token = generate_refresh_token(user, expires_in=timedelta(days=7))

    response = JsonResponse({"access_token": access_token, "user": serialize_user(user)})
    response.set_cookie(
        "refresh_token",
        refresh_token,
        httponly=True,       # JS cannot read this
        secure=True,         # HTTPS only
        samesite="Strict",   # CSRF protection
        max_age=7 * 24 * 3600,
    )
    return response
```

---

### Silent refresh (client interceptor)

```typescript
// axios interceptor — transparent token refresh
let isRefreshing = false;
let failedQueue: Array<{resolve: Function, reject: Function}> = [];

axios.interceptors.response.use(
  (response) => response,
  async (error) => {
    const original = error.config;

    if (error.response?.status !== 401 || original._retry) {
      return Promise.reject(error);
    }

    if (isRefreshing) {
      // Queue requests that arrived while we're already refreshing
      return new Promise((resolve, reject) => {
        failedQueue.push({ resolve, reject });
      }).then((token) => {
        original.headers["Authorization"] = `Bearer ${token}`;
        return axios(original);
      });
    }

    original._retry = true;
    isRefreshing = true;

    try {
      const { data } = await axios.post("/auth/refresh");  // sends HttpOnly cookie automatically
      const newToken = data.access_token;
      setAccessToken(newToken);  // store in memory
      failedQueue.forEach(({ resolve }) => resolve(newToken));
      failedQueue = [];
      original.headers["Authorization"] = `Bearer ${newToken}`;
      return axios(original);
    } catch (refreshError) {
      failedQueue.forEach(({ reject }) => reject(refreshError));
      failedQueue = [];
      redirectToLogin();  // refresh also expired — genuine session end
      return Promise.reject(refreshError);
    } finally {
      isRefreshing = false;
    }
  }
);
```

---

### Refresh endpoint (backend)

```python
@router.post("/auth/refresh", auth=None)
def refresh_token(request):
    refresh_token = request.COOKIES.get("refresh_token")
    if not refresh_token:
        return 401, {"code": "UNAUTHENTICATED", "message": "No refresh token."}

    try:
        user = validate_refresh_token(refresh_token)
    except TokenExpiredError:
        return 401, {"code": "SESSION_EXPIRED", "message": "Session expired. Please log in again."}
    except InvalidTokenError:
        return 401, {"code": "INVALID_TOKEN", "message": "Invalid token."}

    new_access_token = generate_access_token(user)
    return {"access_token": new_access_token}
```

---

## OPTIONS SUMMARY

### Option 1: Short-lived access + silent refresh (recommended)
15-min access token in memory, 7-day refresh in HttpOnly cookie, interceptor handles silent refresh.
**Trade-off:** Most secure. UX is seamless. Requires an axios/fetch interceptor. Access token lost on page refresh → one extra refresh call on load.

### Option 2: Long-lived access token (simpler, less secure)
Access token expiry of 24h–7 days in `localStorage`. No refresh needed.
**Trade-off:** Simpler client code. Token compromise window is large. `localStorage` is XSS-vulnerable. Do not use for apps with sensitive data.

### Option 3: Session cookie (server-side sessions)
Backend issues a session cookie (HttpOnly). No JWTs. Django's built-in session auth.
**Trade-off:** Works great for monoliths. Doesn't work across domains or for native mobile apps. Simplest for pure web-only Django apps.

**The decision is yours.**

---

## AUTH RULES

- **401 = not authenticated.** 403 = authenticated but no permission. Never swap these.
- **The refresh endpoint must not require an Authorization header.** It reads the cookie.
- **Logout must invalidate the refresh token server-side.** Deleting the cookie alone is not enough — the token could have been copied.
- **On 401 during form submission, preserve form state before redirecting to login.** Save to `sessionStorage` and restore post-login.
