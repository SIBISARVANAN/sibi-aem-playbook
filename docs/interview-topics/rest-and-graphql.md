# REST APIs & GraphQL in AEM

## REST — Multiple Layers, Not One API

| Layer | How it works | Code required? |
|---|---|---|
| Sling Default GET (`.json`, `.1.json`, `.infinity.json`, `.tidy.json`) | Any JCR resource is automatically renderable as JSON via extension/selector | None |
| Sling Model Exporter (`.model.json`) | `@Exporter(name="jackson", extensions="json")` on a Sling Model | Annotation only |
| Custom Sling Servlet | `@SlingServletResourceTypes` + `extensions="json"` | Full servlet |
| Sling POST Servlet | Writes to JCR via form params + `:operation` (create/copy/move/delete/checkin/checkout) | None — built in |
| QueryBuilder JSON | `/bin/querybuilder.json?...` | None — usually wrapped in a custom servlet for security |

```bash
# Sling POST Servlet — write to JCR with zero custom code
curl -u admin:admin -F"jcr:primaryType=nt:unstructured" -F"title=Hello" \
     http://localhost:4502/content/mysite/en/newnode

curl -u admin:admin -F":operation=delete" http://localhost:4502/content/mysite/en/oldnode
```

```java
// Sling Model Exporter — the most common "build a REST API" pattern
@Model(adaptables = Resource.class, adapters = Product.class, resourceType = "...")
@Exporter(name = "jackson", extensions = "json", selector = "model")
public class ProductImpl implements Product { ... }
// GET /content/.../products/shirt.model.json
```

**Key intricacies:**
- Selector + extension combo drives Sling's servlet resolution chain — `.model.json` and `.json` hit completely different code paths.
- `.infinity.json` can leak internal/ACL nodes — restrict via `DefaultGetServlet` config or Dispatcher rules in production.
- No built-in API versioning — must build your own convention (e.g. `/api/v1/...` resourceTypes).
- JSON endpoints are NOT cached by Dispatcher by default (often explicitly denied) — must opt in and consider cache-poisoning risk for personalised data.
- Cross-domain frontends need `org.apache.sling.cors.impl.CrossOriginFilter` configured, or browsers block the calls.

## GraphQL — Headless Content Fragment Delivery ONLY

**Critical distinction:** AEM's GraphQL API is scoped specifically to **Content Fragments**, not arbitrary pages/components — unlike REST, which can expose any resource.

**Workflow:** Content Fragment Model (schema, under `/conf/.../cfm/models`) → actual Content Fragments authored in DAM → AEM **auto-generates** the GraphQL schema (one type per model) → query via an endpoint configured in *Tools → General → GraphQL*.

```graphql
# Ad-hoc POST query — fine in dev, DISABLED by default in production/AEMaaCS (DoS risk)
{ articleModelList { items { title author { name } } } }
```

```
# Persisted Query (GET) — production-recommended, Dispatcher/CDN-cacheable
GET /content/_cq_graphql/mysite/endpoint.json/mysite/getArticleByPath;articlePath=/content/dam/mysite/articles/my-article
```

**Why persisted queries matter:** ad-hoc POST queries let a client construct arbitrarily expensive/deep queries at runtime; persisted queries are invoked via deterministic GET URLs, which Dispatcher/CDN can actually cache — the single biggest practical reason teams adopt them.

| Fact | Detail |
|---|---|
| Read-only | No mutations — AEM author UI remains the only way to create/edit content |
| Nested references resolved in one call | A model field referencing another model is resolved server-side — the core advantage over REST's N+1 calls |
| Schema is derived, not independently versioned | Renaming a CF Model field renames the GraphQL field — no separate schema-versioning layer |
| Endpoint is scoped per `/conf` configuration | Not global to the instance |

**Q: When would you choose REST over GraphQL in AEM, or vice versa?**
REST for arbitrary resources, writes, or quick exposure of existing Sling Models. GraphQL specifically when delivering Content Fragments headlessly to SPAs/mobile apps where avoiding over-fetching and resolving nested references in one round-trip matters, and where persisted-query cacheability is valuable.

*(For CIF's separate GraphQL client — used for commerce/Magento data, not Content Fragments — see `/docs/applied-code-patterns/cif-graphql-caching.md`.)*
