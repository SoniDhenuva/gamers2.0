# Backend Cookie/CORS Fix — Verification & Instructions

> **Why this matters:** The frontend calendar on `pages.opencodingsociety.com` makes
> cross-origin requests to `spring.opencodingsociety.com`. For the session cookie to
> be sent cross-origin, it **must** have `SameSite=None; Secure`. Without this, modern
> browsers silently drop the cookie on cross-origin requests, making the user appear
> unauthenticated even after logging in.

---

## 🔍 The Problem (Confirmed via cURL)

```bash
# When hitting a protected endpoint without a cookie, the backend returns:
curl -sS -D - "https://spring.opencodingsociety.com/api/calendar/events" \
  -H "Origin: https://pages.opencodingsociety.com" \
  -H "X-Origin: client" 2>&1 | grep -E "Set-Cookie|HTTP/"

# Response:
# HTTP/1.1 302
# Set-Cookie: sess_java_spring=...; Path=/; HttpOnly
#                                    ^^^^^^^^^^^^^^^^
#                                    MISSING: SameSite=None; Secure
```

The cookie is set as `Path=/; HttpOnly` — it's **missing `SameSite=None; Secure`**.

### What happens:
1. User logs in on `pages.opencodingsociety.com` → POST to `spring.opencodingsociety.com/authenticate`
2. Backend sets `Set-Cookie: sess_java_spring=...; Path=/; HttpOnly`
3. Browser stores the cookie for `spring.opencodingsociety.com`
4. User navigates to calendar page (on `pages.opencodingsociety.com`)
5. Calendar JS fetches `spring.opencodingsociety.com/api/calendar/events` with `credentials: 'include'`
6. **Browser refuses to send the cookie** because `SameSite` defaults to `Lax` → cross-origin requests don't include it
7. Backend sees no cookie → responds with 302 → `/login`
8. The `/login` page is also missing `Access-Control-Allow-Credentials: true` header, so the redirect causes a CORS error
9. User sees "Session expired" even though they just logged in

---

## ✅ The Fix

### Step 1: Set `SameSite=None; Secure` on the session cookie

In your Spring Boot `application.properties` or `application.yml`:

```properties
# application.properties
server.servlet.session.cookie.same-site=none
server.servlet.session.cookie.secure=true
server.servlet.session.cookie.http-only=true
```

Or in YAML:
```yaml
# application.yml
server:
  servlet:
    session:
      cookie:
        same-site: none
        secure: true
        http-only: true
```

### Step 2 (Recommended): Return 401 instead of 302 for API requests

This prevents the redirect-to-login problem entirely for API calls:

```java
// In your SecurityConfig or SecurityFilterChain configuration:

@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        // ... existing config ...
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint((request, response, authException) -> {
                // For API requests, return 401 instead of redirecting to /login
                String uri = request.getRequestURI();
                String origin = request.getHeader("Origin");
                if (uri.startsWith("/api/") || origin != null) {
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.setContentType("application/json");
                    response.getWriter().write("{\"error\":\"Authentication required\"}");
                } else {
                    // For browser page requests, redirect to login as usual
                    response.sendRedirect("/login");
                }
            })
        );
    // ... rest of config ...
    return http.build();
}
```

### Step 3 (If Step 2 is not done): Add `Access-Control-Allow-Credentials` to `/login`

If you keep the 302 redirect behavior, the `/login` page must also include CORS headers:

```bash
# Currently, /login is missing Access-Control-Allow-Credentials:
curl -sS -D - "https://spring.opencodingsociety.com/login" \
  -H "Origin: https://pages.opencodingsociety.com" 2>&1 | grep "Access-Control"

# Returns:
# Access-Control-Allow-Origin: https://pages.opencodingsociety.com
# (missing Access-Control-Allow-Credentials!)
```

Make sure your CORS configuration covers the `/login` endpoint too.

---

## 🧪 How to Verify the Fix is Deployed

Run these commands from your terminal to check:

### Test 1: Check cookie attributes
```bash
curl -sS -D - "https://spring.opencodingsociety.com/api/calendar/events" \
  -H "Origin: https://pages.opencodingsociety.com" \
  -H "X-Origin: client" \
  -H "Content-Type: application/json" 2>&1 | grep -i "set-cookie"
```

**Expected (after fix):**
```
Set-Cookie: sess_java_spring=...; Path=/; HttpOnly; SameSite=None; Secure
```

**Broken (current):**
```
Set-Cookie: sess_java_spring=...; Path=/; HttpOnly
```

### Test 2: Check if API returns 401 instead of 302 (if Step 2 was applied)
```bash
curl -sS -o /dev/null -w "%{http_code}" \
  "https://spring.opencodingsociety.com/api/calendar/events" \
  -H "Origin: https://pages.opencodingsociety.com" \
  -H "X-Origin: client"
```

**Expected (after fix):** `401`
**Broken (current):** `302`

### Test 3: Full login + cookie round-trip test
```bash
# Step A: Login and save cookies
curl -sS -c /tmp/test_cookies.txt -D - \
  "https://spring.opencodingsociety.com/authenticate" \
  -X POST \
  -H "Origin: https://pages.opencodingsociety.com" \
  -H "Content-Type: application/json" \
  -H "X-Origin: client" \
  -d '{"uid":"YOUR_USERNAME","password":"YOUR_PASSWORD"}' 2>&1 | grep -E "HTTP/|Set-Cookie"

# Step B: Use the saved cookie to fetch calendar events
curl -sS -b /tmp/test_cookies.txt -o /dev/null -w "%{http_code}" \
  "https://spring.opencodingsociety.com/api/calendar/events" \
  -H "Origin: https://pages.opencodingsociety.com" \
  -H "X-Origin: client" \
  -H "Content-Type: application/json"
```

**Expected:** Step A returns `200` with `Set-Cookie`, Step B returns `200`
**Broken:** Step B returns `302` (cookie not being accepted)

### Test 4: Check /login CORS headers (if Step 3 was applied)
```bash
curl -sS -D - "https://spring.opencodingsociety.com/login" \
  -H "Origin: https://pages.opencodingsociety.com" 2>&1 | grep "Access-Control"
```

**Expected (after fix):**
```
Access-Control-Allow-Origin: https://pages.opencodingsociety.com
Access-Control-Allow-Credentials: true
```

---

## 📋 Priority / What's Needed

| Priority | Fix | Impact |
|----------|-----|--------|
| **🔴 Critical** | `SameSite=None; Secure` on session cookie | Without this, cross-origin auth is completely broken |
| **🟡 Recommended** | Return 401 for `/api/*` instead of 302 | Prevents confusing redirect-based CORS failures |
| **🟢 Nice-to-have** | `Access-Control-Allow-Credentials` on `/login` | Only needed if keeping 302 redirects for API routes |

**The #1 fix (SameSite=None; Secure) is just 3 lines in `application.properties`.** Everything else is optional.

---

## 🖥️ Frontend Status

The frontend has been updated to handle both the current (broken) and fixed states:
- ✅ Uses `credentials: 'include'` on all fetch requests
- ✅ Catches CORS `TypeError` (caused by 302→/login redirect) and shows "Session expired" banner
- ✅ Checks 401/403 status codes for when backend returns proper error responses
- ✅ Checks `response.redirected` + URL for cases where the browser follows redirects
- ✅ Auth banner has a direct link to the login page
- ✅ `dateClick` guard prevents event creation when not authenticated

**Once the backend cookie fix is deployed, everything will work without any frontend changes.**
