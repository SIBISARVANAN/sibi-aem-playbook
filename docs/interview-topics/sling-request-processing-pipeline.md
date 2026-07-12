# Sling Request Processing Pipeline

> This is the internal request lifecycle **inside** AEM/Sling — distinct from `core-concepts/09-request-flow-cdn-dispatcher-aem.md`, which covers what happens *before* the request even reaches this pipeline (Browser → CDN → Dispatcher → AEM).

## The Pipeline

```
HTTP request arrives at the Sling Engine (on Jetty/Felix HTTP Whiteboard)
│
▼
1. Authentication — SlingAuthenticator picks a handler
   (Form/Basic/Token/SSO) → resolves a Session/ResourceResolver
   (or falls back to anonymous)
   │
   ▼
2. Resource Resolution — ResourceResolver.resolve(path)
   Strips selectors/extension/suffix, walks UP the path until
   it finds an actual existing resource; remainder becomes
   the request "suffix". Vanity URLs / sling:redirect / /etc/map
   mappings are also applied at this stage.
   │
   ▼
3. Servlet/Script Resolution
   Resource's sling:resourceType (+ resourceSuperType chain)
   combined with selectors + extension + HTTP method →
   picks the MOST SPECIFIC matching script/servlet under
   /apps or /libs.
   │
   ▼
4. REQUEST-scope Filter Chain (pre-processing)
   Filters run in service.ranking order, each calling
   chain.doFilter() to proceed
   │
   ▼
5. Component Rendering (HTL/JSP/Servlet executes)
   Nested sling:include / sling:resource calls trigger
   steps 2-4 AGAIN recursively for each child component —
   page rendering is really a TREE of nested Sling requests
   │
   ▼
6. Filter post-processing (response unwinds back through
   the same filters, in reverse, after chain.doFilter() returns)
   │
   ▼
   Response sent to client
```

## Two Design Facts Worth Knowing Well

**Resources, not JCR nodes, are the real abstraction.** Sling doesn't hard-wire itself to JCR. It talks to a `ResourceProvider` SPI — JCR is just one provider (`JcrResourceProvider`). Other providers can mount virtual resources from anywhere (an external API, a config map, OSGi bundle resources) into the same resource tree at a chosen mount point. This is why the same Sling Model/HTL code can render content regardless of whether it actually came from JCR — the rest of the stack only ever talks to "Resources," never to raw JCR.

**The `/apps` over `/libs` overlay mechanism (Resource Merger)** is how customisations of Adobe's out-of-the-box components work without ever touching `/libs` directly: Sling's script/servlet resolution checks `/apps` first, falls back to `/libs` if nothing's there, and the Resource Merger can even merge node properties from both locations (not just whole resources) for partial overlay scenarios — this underlies the well-known "never modify `/libs`, always override in `/apps`" rule.

## Why This Matters for Debugging

Because step 5 recursively re-triggers steps 2–4 for every nested `sling:include`/`sling:resource`, a slow page is really a *tree* of nested requests — a single slow child component (say, one that opens an unclosed `ResourceResolver` or calls a slow external API) will drag down the total render time of every ancestor component wrapping it, even though those ancestors' own code did nothing wrong. This is why component-level render-time profiling (via Developer Mode) is more useful than page-level timing alone when hunting a slow page.
