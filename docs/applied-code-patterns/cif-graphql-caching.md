# CIF GraphQL Caching

## Architecture

CIF caches GraphQL responses **in-process inside the AEM JVM** (via a Guava Cache), not in Dispatcher — because GraphQL POST requests have no stable URL to cache against at the HTTP layer.

```
Browser → AEM Publish
              ↓
    GraphqlClientImpl (Guava Cache — in JVM heap)
              ↓ cache miss only
    Adobe Commerce / Magento GraphQL API
```

## Cache key

The cache key is a **SHA-256 hash of the query string plus its serialised variables** — not just the query name. Same query + same variables = cache hit regardless of which user triggered it. Two calls to the same named query with different variables (e.g. different `sku`) are different cache entries; this is why a cache-hit-rate problem is often actually a "variables aren't being normalized/sorted consistently" bug rather than a real cache miss.

## OSGi config (factory — one instance per commerce endpoint)

```json
{
  "cacheEnabled": true,
  "cachingTime": 300,
  "cacheSize": 100,
  "httpMethod": "GET"
}
```

This is a **factory** config — one instance per site. A common pattern: author gets a short TTL (editors need to see fresh data), publish gets a longer TTL (performance).

## Viewing cache stats

```
/system/console/status-com.adobe.cq.commerce.graphql.client  ← hit rate, miss rate, evictions
/system/console/jmx → "GraphqlClient"                         ← live stats + invalidateAll operation
```

## Clearing the cache

| Method | How |
|---|---|
| TTL expiry | Automatic — wait `cachingTime` seconds |
| JMX | `/system/console/jmx` → GraphqlClient → `invalidateAll` |
| OSGi config save | Triggers `@Modified` → cache rebuilt |
| Bundle restart | Stop/Start the bundle in `/system/console/bundles` |

## Common Interview Q&A

**Q: Why doesn't CIF use Dispatcher to cache GraphQL responses?**
Dispatcher caches HTTP responses keyed by URL. GraphQL queries are typically POST requests — POST has no stable URL, and Dispatcher doesn't cache POST by design. CIF's JVM-level cache intercepts at the Java client layer before the HTTP response leaves AEM, making it framework-agnostic. Persisted queries (GET) CAN be Dispatcher-cached and are the recommended production approach.

**Q: What happens to the CIF GraphQL cache on AEM restart?**
Lost entirely — it's in JVM heap memory only, not persisted. First requests after a restart are all cache misses and call the commerce backend directly. Design your `cachingTime` so this cold-start period is acceptable.

**Q: How do you handle stale product prices after a commerce-side price update?**
Three options: (1) a short `cachingTime` (60–120s) so stale data self-heals quickly, (2) a commerce webhook on price change triggering the AEM JMX `invalidateAll` operation, (3) a Commerce Event Bus triggering a custom CIF cache-invalidation listener. Most production teams combine (1) and (2).

**Q: How is CIF's GraphQL cache different from a JVM cache you'd write yourself (e.g. an inventory service cache)?**
Functionally the same pattern — both are Guava Caches with `maximumSize` + `expireAfterWrite`. The difference is CIF's is built-in and configured via OSGi factory config per commerce endpoint, while your own cache is custom-written per service. The same "no `ConcurrentHashMap` without eviction" rule applies to both.
