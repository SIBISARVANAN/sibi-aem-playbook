# Request Flow: Browser to AEM Publish (End-to-End)

Covers the complete path of an anonymous request from browser to AEM Publish and back, including Azure CDN, load balancer, WAF, dispatcher's internal decision chain, vanity/shorturl resolution, and the parallel flows (cache invalidation, author requests, vanity path indexing).

---

## Full Flow Skeleton

```
1. Browser (End User)
        |
        v
2. DNS Resolution
        |
        v
3. Azure CDN / Front Door (Edge Node)
   |-- WAF Layer #1 (if placed here) -> inspects for malicious signatures,
   |     rate limiting, geo-blocking, BEFORE cache lookup
   |-- TLS termination may happen HERE (edge SSL)
   |-- CDN-level cache check (separate cache layer from dispatcher)
   |      HIT -> served from CDN edge, request never goes further
   |      MISS -> continue
        |
        v
4. Load Balancer / Application Gateway
   |-- WAF Layer #2 (alternative/additional placement)
   |-- TLS termination may happen HERE instead (if not done at CDN)
   |-- Re-encryption to origin (TLS bridging) OR forwards plain HTTP +
   |     X-Forwarded-Proto header
   |-- Sticky session decision (session affinity cookie, if configured)
   |-- Routes to available Dispatcher/Apache node
        |
        v
5. Apache + Dispatcher Module

   5a. Virtual Host Matching (conf.d/enabled_vhosts)
       -> decides WHICH SITE this request belongs to, based on
          ServerName/ServerAlias matching the Host header
       -> BRANCH: Author or Publish vhost?

   5a-ii. Apache Basic Auth Check (if configured on this vhost)
       -> typically used on non-prod domains (dev/stage/UAT)
       MISSING/INVALID -> 401 + WWW-Authenticate header -> browser shows
                           native login popup, blank page behind it
       VALID -> continue

   5b. Rewrite Rules (conf.d/rewrites/rewrite.rules)
       -> static pattern-based URL normalization (e.g. strip .html for
          pretty URLs) -- NOT the same mechanism as vanity path resolution

   5c. Farm Matching (conf.dispatcher.d/enabled_farms)
       -> which ruleset (cache, filters, renders) applies

   5c-ii. Farm's own /virtualhosts re-check
       -> the farm file has its own Host-header allowlist, separate from
          the Apache vhost file -- both must be kept in sync

   5d. Client Headers (/clientheaders)
       -> allowlist of which incoming headers pass through to AEM

   5e. Filter Check (/filters)
       -> structural allow/deny on the URL PATH (e.g. blocks /bin/,
          /crx/, /system/) -- evaluated top to bottom, LAST match wins
       -> not authentication; doesn't care who is asking

   5f. Cache Check (/cache, .stat files)
       -> auth/personalization bypass: login-token cookie or
          Authorization header present -> skip cache read AND write,
          forward live
       -> cache key built from URL path + configured query params
          (ignoreUrlParams controls which params are folded in)
       -> .stat file hierarchy checked per directory level to determine
          if the cached copy has been invalidated
       HIT  -> serve from disk, Publish never touched
       MISS -> continue

   5g. Request Method / Cacheability Check
       -> only GET/HEAD are cache-eligible; POST/PUT/etc. always live

   5h. Renders Block (/renders)
       -> picks which Publish instance handles this request
       -> load-balances/retries across the render list
       -> if all publish targets fail -> /renderers block serves a
          static error page instead of a raw connection failure
        |
        v
6. Publish Instance (AEM)

   6a. Auth Check
       -> Oak/JCR-level ACL evaluation -- the real authentication layer,
          distinct from dispatcher's Basic Auth and cache-bypass heuristic

   6b. Sling Resource Resolution -- vanity/shorturl resolution happens HERE
       -> ResourceResolver.resolve(): checks exact path -> /etc/map or
          Sling Mapping -> in-memory vanity path map (built from
          sling:vanityPath properties, indexed at startup + kept live
          via JCR observation)
       -> reverse direction: ResourceResolver.map() converts an internal
          JCR path back to its public vanity/shortened form when AEM
          renders an outbound link

   6c. CSRF Token Check (on state-changing requests)
       -> AEM's CSRF Protection framework validates a token before
          servlet/component logic executes on POST-type requests

   6d. Component/Servlet Rendering
       -> HTL, Sling Models, backend Java logic execute
       -> Cache-Control response headers set here
        |
        v
7. Response travels back UP through Dispatcher
   -> cached under the ORIGINALLY requested URL (the vanity one, not the
      resolved internal path) if /rules marks it cacheable
        |
        v
8. Back through Load Balancer -> CDN
   -> CDN applies its own independent caching rules based on the
      Cache-Control headers AEM set
        |
        v
9. Browser renders the page
```

---

## Parallel Flow #1 -- Author Publish -> Cache Invalidation

```
Author activates/publishes content
        |
        v
Replication Agent (Author) fires
        |
        v
Flush Agent -> sends invalidation request to Dispatcher
        |
        v
Dispatcher deletes/updates relevant .stat files
        |
        v
Next matching request -> cache is now considered stale/missing
        -> step 5f becomes a MISS -> forwarded live to Publish again
```

## Parallel Flow #2 -- Author Request

```
Author logs into AEM Author instance
        -> typically hits a SEPARATE vhost/farm (or bypasses dispatcher
           entirely depending on architecture)
        -> bypasses CDN edge cache and dispatcher disk-cache -- always live
        -> full authentication required at AEM level (CRX/OAK security)
```

## Parallel Flow #3 -- Vanity Path Index Build

```
AEM startup (or content change under any vanity-path-carrying node)
        |
        v
Sling scans repository for sling:vanityPath properties
        |
        v
Builds/updates in-memory vanity path map
        -> newly-added vanity paths can take a moment to become live
        -> heavy vanity path usage has real startup-time and memory cost
```

---

## Step-by-Step: Layman + Technical

### 1. Browser -> DNS
**Layman:** You type the URL; your computer asks "who is this domain, and what's their address?"
**Technical:** DNS resolves the domain to an IP -- typically the CDN's anycast IP, not the origin server's IP directly. This is why the real server IP is usually hidden.

### 2. Azure CDN / Front Door -- Edge Node
**Layman:** The nearest gatekeeper to you geographically. Blocks bad traffic, serves cached content if possible, and only bothers the real servers if it has to.

**Technical (in order):**
- **WAF Layer #1 (if placed here):** Inspects raw request against rule sets (OWASP core rule set typically) for SQLi, XSS, bad bot signatures, rate-limit thresholds, geo-blocking. Runs before cache lookup deliberately.
- **TLS termination (if here):** CDN decrypts HTTPS at the edge. Downstream traffic (CDN -> LB -> Apache) may travel plain HTTP internally or get re-encrypted (TLS bridging). CDN typically injects `X-Forwarded-Proto: https` so downstream layers know the original request was HTTPS.
- **CDN cache check:** Separate cache from dispatcher's disk cache -- different TTLs, different invalidation (usually a purge API call, not `.stat` files). HIT = served, dies here. MISS = forwarded on.

### 3. Load Balancer / Application Gateway
**Layman:** Distributes traffic across multiple dispatcher/Apache servers, and does a second round of security checks.

**Technical:**
- **WAF Layer #2:** Azure App Gateway's WAF_v2 SKU -- same class of inspection, redundant defense-in-depth vs. added latency trade-off.
- **TLS termination (if not at CDN):** App Gateway decrypts here instead.
- **Sticky session decision:** If session affinity is configured (e.g. Azure `ARRAffinity` cookie), subsequent requests from this browser route to the same backend node.
- **Routing:** Picks a healthy Dispatcher/Apache node via health checks.

### 4a. Virtual Host Matching
**Layman:** "Which website is this request even for?"
**Technical:** Apache matches the `Host` header against `ServerName`/`ServerAlias` in `conf.d/enabled_vhosts`. First match wins -- misconfigured ordering is a real source of "wrong site served" bugs. Also the branch point for Author vs Publish vhost.

### 4a-ii. Apache Basic Auth (if configured)
**Layman:** A locked door with a shared password, used on non-prod environments so random people/crawlers can't stumble onto dev/UAT.
**Technical:** `AuthType Basic` + `AuthUserFile` on the vhost. Missing/invalid `Authorization: Basic <base64>` header -> `401` + `WWW-Authenticate: Basic realm="..."` -> browser renders its native credential popup. Happens before any other dispatcher logic -- fail this and farm/filter/cache rules never get evaluated.

### 4b. Rewrite Rules
**Layman:** Cleaning up the URL into an expected shape before anything else looks at it.
**Technical:** Standard Apache `mod_rewrite` in `conf.d/rewrites/rewrite.rules`. Pattern-based, static -- e.g. stripping `.html`. Not the same mechanism as vanity paths -- pure regex, no repository lookup.

### 4c. Farm Matching
**Layman:** Now that we know the site, which rulebook (caching, filters, security) applies?
**Technical:** Matched via `conf.dispatcher.d/enabled_farms`, typically based on vhost + path patterns configured in the farm's own `/virtualhosts` block.

### 4c-ii. Farm's `/virtualhosts` Re-check
**Layman:** A second ID check, this time by the department itself.
**Technical:** The farm file has its own `/virtualhosts` Host-header allowlist, in a different file from the Apache vhost. Adding a domain alias to the Apache vhost without updating the farm's list causes a silent mismatch -- classic production gotcha.

### 4d. Client Headers
**Layman:** Deciding which of your request's information is even allowed to reach AEM.
**Technical:** `/clientheaders` is an explicit allowlist. Anything not listed is stripped -- both for security and because AEM may need a specific custom header for personalization logic.

### 4e. Filter Check
**Layman:** A bouncer checking if this URL path is allowed to exist publicly, regardless of who's asking.
**Technical:** `/filters` block, ordered allow/deny rules, **last match wins** (common gotcha -- people assume first match wins). Blocks structural paths like `/bin/`, `/crx/`, `/system/console`, `/libs`. Not authentication.

### 4f. Cache Check
**Layman:** "Do I already have a saved copy of this exact page? If yes, hand that over -- no need to bother the real server."
**Technical, sub-checks:**
- **Auth/personalization bypass:** login-token cookie or `Authorization` header present -> skip cache entirely (read and write), forward live. Prevents leaking one user's personalized response to another.
- **Cache key construction:** URL path + configurably-included query params (`ignoreUrlParams`). Misconfiguring this fragments the cache (tracking params treated as distinct pages) or wrongly serves the same page for genuinely different query-driven content.
- **`.stat` file check:** hierarchy of `.stat` files (one per directory level) determines if the cached file has been invalidated since written. This -- not TTL expiry -- is the primary invalidation mechanism in AEM.

HIT -> serve from disk. MISS -> continue.

### 4g. Request Method / Cacheability Check
**Layman:** Only "just looking" requests (GET) can be served from or saved to cache; anything that changes something always goes live.
**Technical:** Only `GET`/`HEAD` are cache-eligible by default. `POST`/`OPTIONS`/etc. always forwarded live.

### 4h. Renders Block
**Layman:** Deciding which actual publish server handles this, and what to do if none respond.
**Technical:** `/renders` lists publish host:port targets; dispatcher load-balances/retries across them. If all fail, the separate `/renderers` (error-rendering) config serves a static error page instead of a raw connection failure.

### 5a. Auth Check (Publish)
**Layman:** Now that we've reached the real server, does this specific content require login?
**Technical:** Oak/JCR-level ACL evaluation against the resolved path -- the real authentication/authorization layer, distinct from dispatcher's Basic Auth (network gate) and cache-bypass heuristic.

### 5b. Sling Resource Resolution -- Vanity/Shorturl Resolution
**Layman:** Translating the friendly URL you typed into the actual internal filing location of the content.
**Technical:** `ResourceResolver.resolve()`, in order: exact path match -> `/etc/map` or Sling Mapping (`/conf/.../sling:mapping`) -> in-memory vanity path map (built from `sling:vanityPath` properties, indexed at startup, kept live via JCR observation). First match wins.

Reverse direction: `ResourceResolver.map()` converts an internal JCR path back to its public vanity/shortened form when AEM renders an outbound link -- same config, opposite direction.

### 5c. CSRF Token Check
**Layman:** For anything that changes data, AEM double-checks you weren't tricked into submitting it.
**Technical:** AEM's CSRF Protection framework validates a token on state-changing requests before servlet/component logic executes -- sits between resource resolution and rendering.

### 5d. Component/Servlet Rendering
**Layman:** Actually building the page.
**Technical:** Sling Models, HTL scripts, servlet logic execute. `Cache-Control` response headers set here, which matter downstream when CDN decides its own caching.

### 6. Response Travels Back Up
Dispatcher checks its own `/rules` config -- cacheable? -> writes to disk under the **originally requested URL** (the vanity one, not the resolved internal path). This means a vanity URL and the real path are cached as two separate entries if both are ever requested directly.

### 7. Back Through LB -> CDN
CDN applies its own independent caching rules based on `Cache-Control` headers AEM set.

### 8. Browser Renders
Done.

---

## Interview-Ready One-Liners

- **Vhost vs Farm:** "Vhost decides which door you walked through based on domain; farm decides what happens once you're inside -- caching, security filters, and which publish instances actually serve the request."
- **Filter evaluation order:** Last match wins, not first -- a common misconfiguration source.
- **Cache invalidation:** Primarily driven by `.stat` file updates from the flush agent, not TTL expiry.
- **Vanity vs rewrite rules:** Two completely different mechanisms -- rewrite rules are static regex patterns evaluated by dispatcher before farm logic; vanity path resolution is a dynamic repository lookup that happens inside AEM's Sling resource resolution, after the request has already reached Publish.
- **Farm's `/virtualhosts` vs Apache vhost:** Two separate places doing what looks like the same Host-header check -- forgetting to sync both when adding a domain alias is a classic production bug.
- **Dispatcher Basic Auth vs AEM auth vs WAF:** Three unrelated mechanisms that sit at three different layers -- Apache-level static credential gate (usually non-prod only), Oak/JCR-level ACL evaluation (the real app-level auth), and CDN/LB-level malicious-pattern inspection (WAF), respectively.
