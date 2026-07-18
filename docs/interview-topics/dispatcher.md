# Dispatcher

## What the Dispatcher Does

The Dispatcher is an Apache httpd module that acts as AEM's caching layer and load balancer. It sits between the CDN (or browser) and AEM publish instances.

Its two jobs:
1. **Cache** — serve static HTML from disk without hitting AEM for every request
2. **Load balance** — distribute requests across multiple AEM publish nodes

## What Dispatcher Caches

By default, Dispatcher caches GET and HEAD requests that return HTTP 200. It does **not** cache:
- Requests with query strings (unless explicitly configured)
- Requests for paths in the `/bin/` or `/system/` namespace
- Responses with `Set-Cookie` headers (unless configured to ignore them)
- POST requests

## Dispatcher Cache Invalidation

When an author activates (publishes) a page:

1. AEM's replication agent sends a `DELETE` (flush) request to Dispatcher
2. Dispatcher deletes the cached `.html` file for that page
3. It also deletes all `.stat` files up the directory tree
4. Next request for the page is a cache miss → AEM renders it → Dispatcher caches again

**Statfile level:** the `.stat` file mechanism means publishing one page can invalidate cached files in parent directories. Configure `statfileslevel` in `dispatcher.any` to control how far up the invalidation propagates.

## Key Dispatcher Configuration

```apache
# dispatcher.any — example farm config (key settings)
/cache {
    /docroot "/var/www/html"          # where HTML files are cached on disk

    /rules {
        /0001 { /type "allow" /glob "*.html" }   # cache HTML
        /0002 { /type "deny"  /glob "/bin/*" }   # never cache /bin
        /0003 { /type "deny"  /glob "/system/*" }
    }

    /statfileslevel "3"   # invalidate up to 3 directory levels

    /invalidate {
        /0001 { /type "allow" /glob "*" }  # allow all invalidation requests
    }
}
```

## Security Headers at Dispatcher Level

```apache
# httpd.conf or vhost config — via mod_headers
<IfModule mod_headers.c>
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-Content-Type-Options "nosniff"
    Header always set Content-Security-Policy "default-src 'self'"
    Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
</IfModule>
```

Setting these at Dispatcher (not in a Sling Filter) ensures they appear on **all** responses, including cached ones.

## Common Interview Questions

**Q: What is the difference between Dispatcher flush and cache invalidation?**
They are the same thing. AEM's replication agent sends a flush request (an HTTP GET with `CQ-Action: Delete` headers) to the Dispatcher. Dispatcher deletes the file from its cache. "Flush agent" and "invalidation" both refer to this process.

**Q: Why would a page not be getting cached by Dispatcher?**
Common causes: the page URL has a query string; the response has a `Set-Cookie` header (session cookies prevent caching); the path is on the Dispatcher deny list; the response status is not 200; the `Cache-Control: no-cache` header is set.

**Q: How do you force Dispatcher to cache a page with query parameters?**
Use `/ignoreUrlParams` in `dispatcher.any` to list parameters that should be ignored for cache key computation. Parameters in this list are stripped from the cache key, so `page.html?utm_source=x` and `page.html` are treated as the same cache entry.

**Q: What is the Dispatcher TTL and how does it work?**
By default, Dispatcher does not use TTL — it caches indefinitely until a flush request arrives. You can enable TTL-based expiry with `/enableTTL "1"` in `dispatcher.any`, which then respects the `Cache-Control: max-age` header from AEM.

**Q: How does Dispatcher handle author vs publish?**
Dispatcher is only deployed in front of **publish**. The author environment does not use Dispatcher — authors must see live, un-cached content. Author instances are accessed directly (or via a reverse proxy with no caching).
