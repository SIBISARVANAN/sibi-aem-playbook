# Common Interview Questions — Senior Level

Questions that appear frequently at the senior/lead AEM developer level, spanning multiple topics.

## Architecture & Design

**Q: How would you design a multi-site, multi-language AEM project?**

Key decisions:
- **Live Copy (MSM)** for sites that share content but need market-specific variations. The blueprint is the master; live copies inherit and can override.
- **Language Copy** for translation workflows. Create language roots under each country site (e.g. `/content/mysite/us/en`, `/content/mysite/fr/fr`).
- **CAConfig** per site root for site-specific configuration (logo, nav, features).
- **Shared component library** under a single app node (`/apps/mysite-components`) referenced by all sites.
- Single `ui.apps` package — avoid duplicate components per site.

**Q: How would you handle a situation where a page takes 10 seconds to render?**

Systematic approach:
1. Check AEM's **Request Performance Log** (`/system/console/requests`) for slow requests.
2. Enable **Developer Mode** in AEM to see per-component render times in the page source.
3. Profile with **YourKit** or heap dumps from `/system/console/jmx`.
4. Common culprits: QueryBuilder queries without `p.guessTotal` doing full traversal; N+1 queries in a list component; `ResourceResolver` not being reused; HTTP calls to external APIs on the render thread.
5. Fix: add `p.guessTotal`, cache expensive computations in `@PostConstruct`, move API calls to Sling Jobs, use `SlingScheduler` to pre-warm caches.

**Q: What is MSM (Multi-Site Manager) and how does it work?**

MSM lets you create Live Copies of a blueprint page tree. Rollout Configs define which properties and children are synced from blueprint to live copy. Live copy pages can have local overrides (breakpoints) that are preserved during rollout. Common rollout triggers: manual rollout, publish, page creation.

## Performance

**Q: How do you find slow queries in AEM?**

1. `/system/console/jmx` → `QueryStat` → `Slow Queries` — lists queries sorted by execution time.
2. Enable Oak query logging: set `org.apache.jackrabbit.oak.query` to DEBUG.
3. Use `/bin/querybuilder.json` with `p.debugTo` to get the SQL2 translation and then use `EXPLAIN SELECT ...` via the Oak JMX bean.

**Q: What causes `javax.jcr.query.InvalidQueryException: Traversal query`?**
A query touches more than 100,000 nodes without using an index. The fix is always to add an indexed predicate — usually `type=cq:Page` + a property index on the filtered property.

**Q: How do you cache data in AEM at the Java level?**

Options in order of preference:
- **Guava Cache / Caffeine** — in-memory cache in the OSGi service with TTL. Fast, no persistence.
- **JCR node** — store computed data as a JCR property, retrieve in `@PostConstruct`. Persists across restarts. Good for expensive data that rarely changes.
- **Servlet response caching** — add `Cache-Control` headers and let Dispatcher cache the JSON output.
- **Sling Scheduler pre-warming** — a scheduler runs periodically, pre-computes data, stores it in a field or JCR node.

## Debugging

**Q: A component works in author but not in publish. How do you debug?**

1. Check that the content has been **activated** (published) to the publish instance.
2. Check that the **OSGi bundle** is `Active` on publish — `/system/console/bundles` on the publish host.
3. Check that the **OSGi config** exists on publish — configs in `config.publish/` are publish-only; `config/` applies to all run modes.
4. Check **error.log** on publish for exceptions.
5. Check **Dispatcher allow/deny rules** — the request may be blocked before reaching AEM publish.
6. Check **user permissions** — publish uses anonymous or a specific user; author uses the logged-in CMS user.

**Q: How do you debug an OSGi component that isn't activating?**

1. `/system/console/bundles` — check if the bundle is `Active`. If `Resolved`, it has unmet package imports.
2. `/system/console/components` — find your component. If `Unsatisfied`, a `@Reference` is not satisfied (the required service is missing or not active).
3. Check `error.log` for `Cannot satisfy reference` messages.
4. Verify the required service is itself active.

**Q: What is CRXDE Lite and when should you NOT use it in production?**
CRXDE Lite is a browser-based JCR repository browser and editor at `/crx/de`. Never use it to make persistent changes in production — changes made in CRXDE are not version-controlled, not repeatable, and will be overwritten by the next code deployment. Use it only for debugging and reading node properties. All changes must go through a deployable package or repoinit script.

## Data Migration & Content Operations

**Q: How do you bulk-update 50,000 JCR nodes efficiently?**

```java
// Use QueryBuilder to find nodes, then batch-process with periodic saves
int batchSize = 500;
int count = 0;

SearchResult result = query.getResult();
for (Hit hit : result.getHits()) {
    Resource resource = hit.getResource();
    ModifiableValueMap mvm = resource.adaptTo(ModifiableValueMap.class);
    mvm.put("myProperty", "newValue");
    count++;

    if (count % batchSize == 0) {
        resourceResolver.commit(); // commit every 500 nodes to avoid OutOfMemory
        log.info("Committed {} nodes", count);
    }
}
resourceResolver.commit(); // final commit for remaining nodes
```

Key points:
- Always commit in batches — never hold all changes in the JCR session at once.
- Use a service user — never admin.
- Run as a Sling Job, not a scheduler, so it can be monitored and retried.
- Add `Thread.sleep(10)` between batches in production to avoid saturating the JCR write queue.

**Q: How do you create a page programmatically?**

```java
PageManager pm = resourceResolver.adaptTo(PageManager.class);
Page newPage = pm.create(
    "/content/mysite/en",       // parent path
    "my-new-page",              // page name (URL segment)
    "/conf/mysite/settings/wcm/templates/article",  // template path
    "My New Page Title",        // title
    true                        // auto-rename if name collision
);
resourceResolver.commit();
```

## Common Gotchas (High Interview Value)

**1. ResourceResolver leak**
The most common production issue. Every `ResourceResolver` opened must be closed in a `finally` block. Not closing it leaks a JCR session — eventually the session pool exhausts and AEM stops responding.

**2. Sling Model returning null**
`adaptTo()` returns `null` if adaptation fails. If `DefaultInjectionStrategy.REQUIRED` is used and any required field is missing, adaptation fails silently. Always null-check `adaptTo()` results and use `OPTIONAL` unless you truly require a field.

**3. QueryBuilder session leak**
`query.getResult()` internally holds a JCR session. If you forget to close it (via `result.getHits()` iteration completing or calling `result.close()`), you leak a session. Use try-with-resources or call `((CloseableQuery)query).close()`.

**4. Thread safety in OSGi services**
OSGi services are singletons. Any instance field in an OSGi service is shared across all threads. Never store request-specific data in instance fields. Use local variables or `ThreadLocal` for per-request state.

**5. Accessing JCR in a filter on every request**
A Sling Filter that opens a `ResourceResolver` on every request will create and close a JCR session for every HTTP hit. Under load, this exhausts the session pool. Cache the data in a service-level field, pre-warmed by a scheduler.

**6. Missing `@Modified` handler**
If a component reads config in `@Activate` but has no `@Modified`, changing the config in the Felix console triggers `@Activate` again — which may not clean up previous state (e.g. re-registers a scheduler without unregistering the old one). Always implement `@Modified` or ensure `@Activate` is idempotent.

**7. `context='html'` in HTL with user content**
Using `@ context='html'` on a value that came from user input is an XSS vulnerability. Only use `html` context for values you control (e.g. from a trusted RTE field stored in JCR via the AEM author UI, which itself sanitizes input).
