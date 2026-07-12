# Request Flow — Browser → CDN → Dispatcher → AEM

## Full Request Lifecycle

```
Browser
   │
   │ HTTP request
   ▼
CDN
   ├── cache hit?  ──► return cached response to browser
   │                   (AEM, Dispatcher, and filters never see this request)
   │ cache miss
   ▼
Dispatcher
   ├── cache hit?  ──► return cached HTML to CDN
   │                   (AEM and filters never see this request)
   │ cache miss or invalidation
   ▼
AEM Publish
   │
   ▼
Sling Authentication (login / session resolution)
   │
   ▼
REQUEST Filter — pre-processing          ◄── your filter fires here (before chain.doFilter)
   │
   ▼
Sling Resource Resolution (path → resource → component)
   │
   ▼
[INCLUDE scope filters fire per component inclusion]
   │
   ▼
Servlet / Component renders HTML
   │
   ▼
REQUEST Filter — post-processing         ◄── your filter fires here (after chain.doFilter)
   │
   ▼
Response leaves AEM
   │
   ▼
Dispatcher caches HTML on filesystem → returns to CDN
   │
   ▼
CDN caches response → returns to Browser
   │
   ▼
Browser renders page
```

## Where Exactly Inside AEM Does the Filter Fire

Once the request enters AEM publish, the processing order is:

1. Sling Authentication layer handles login and session resolution.
2. Your REQUEST scope Sling Filters fire in service ranking order — pre-processing phase, before `chain.doFilter()`.
3. Sling resolves the resource and selects the appropriate servlet or component script.
4. If the component includes other components via `RequestDispatcher.include`, your INCLUDE scope filters fire for each inclusion.
5. The component renders the HTML and writes it to the response.
6. Your REQUEST scope filter post-processing runs — after `chain.doFilter()`. This is where you add response headers or modify the response body.
7. The response leaves AEM and goes back to Dispatcher.

## Practical Implications

**Your filter only fires when the request actually reaches AEM.** If CDN or Dispatcher serves a cached response, your filter is never called for that request. This means:

- You **cannot** use a Sling Filter to intercept every single user request.
- Logic that must run on every page view regardless of caching (analytics, personalisation) belongs in the browser via JavaScript, not in a Sling Filter.
- Logic that only needs to run when AEM actually renders a page (security headers, token validation, response modification) is correct in a Sling Filter.

## Security Headers and Caching — A Critical Note

Security headers added by a Sling Filter (e.g. `X-Frame-Options`, `Content-Security-Policy`) will only be present on responses that AEM renders directly. **Cached responses from Dispatcher or CDN will not carry those headers.**

**Recommended approach:** Configure security headers at the Dispatcher level using the Apache `mod_headers` directive. This ensures headers are present on all responses, including cached ones.

## Dispatcher Cache Invalidation and Filters

When an author publishes a page:

1. AEM sends a cache invalidation request to Dispatcher.
2. Dispatcher marks the cached file as stale/invalid.
3. The next request for that page misses the Dispatcher cache.
4. The request reaches AEM — **your filter fires**.
5. AEM renders the page fresh.
6. Dispatcher caches the new HTML.

Your filter therefore fires on the **first request after every publish event**, and then not again until the cache is invalidated next time.
