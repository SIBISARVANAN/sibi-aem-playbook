# Sling Filters

## What Is a Sling Filter

A Sling Filter is a Java class that intercepts HTTP requests and responses in AEM before they reach a servlet or component, and after the response is generated. It is the AEM equivalent of a standard Java Servlet Filter and follows the same `javax.servlet.Filter` contract.

**Use a filter when you need cross-cutting logic that applies to many requests:**
- Authentication and token validation
- Adding or modifying response headers
- Logging and request monitoring
- Redirects
- Response body modification (e.g. injecting scripts)
- Global error handling

## Old Way vs New Way

The old way uses string-based OSGi component properties inside the `@Component` annotation to configure the filter. It works but is error-prone because property names and values are plain strings with no type safety.

```java
@Component(service = Filter.class, property = {
    EngineConstants.SLING_FILTER_SCOPE + "=" + EngineConstants.FILTER_SCOPE_REQUEST
})
@ServiceRanking(-700)
public class LoggingFilter implements Filter { }
```

The new way uses the `@SlingServletFilter` annotation alongside `@Component`. It is type-safe, cleaner to read, and is the currently recommended approach for all new AEM development.

```java
@Component(property = { "service.ranking:Integer=-800" })
@SlingServletFilter(
    scope         = { SlingServletFilterScope.REQUEST },
    pattern       = "/content/sibi-aem-one/.*",
    resourceTypes = { "sibi-aem-one/components/page" },
    selectors     = { "print", "mobile" },
    extensions    = { "html", "json" },
    methods       = { "GET", "POST", "HEAD" }
)
public class ModernLoggingFilter implements Filter { }
```

## The Five Filter Scopes

Scope is the most important configuration decision. It controls at what point in the request lifecycle your filter fires.

| Scope | When it fires |
|---|---|
| `REQUEST` | Every incoming HTTP request from a client. The most commonly used scope — covers auth checks, logging, header injection, and response modification. |
| `INCLUDE` | When `RequestDispatcher.include()` is called — one component including another, such as a parsys including its child components. |
| `FORWARD` | When `RequestDispatcher.forward()` is called. Less common in AEM but can occur in error handling flows or custom routing logic. |
| `ERROR` | When `HttpServletResponse.sendError()` is called or when an uncaught `Throwable` propagates up from the servlet. Use for global error handling and graceful fallback pages. |
| `COMPONENT` | Legacy scope kept for backwards compatibility. Fires across REQUEST, INCLUDE, and FORWARD. Avoid it in new code. |

## Filter Narrowing Properties

Every property you add to `@SlingServletFilter` narrows the set of requests that trigger the filter. Only requests matching **all** specified conditions will call `doFilter()`.

| Property | What it restricts |
|---|---|
| `pattern` | A regex the request path must match. If not specified, applies to all paths — almost never correct in production. |
| `extensions` | Restricts to specific file extensions (html, json, xml). A filter registered for html will not fire on API calls returning json. |
| `methods` | Restricts to specific HTTP methods (GET, POST, HEAD). Always restrict to only the methods you actually need. |
| `resourceTypes` | Restricts to requests where the resolved resource has a specific `sling:resourceType`. Useful when you want a filter to fire only for a specific component type. |
| `selectors` | Restricts to requests containing specific Sling selectors in the URL. |

## Service Ranking — Filter Execution Order

Service ranking controls the order in which multiple filters execute on the same request. It is set as an integer on the `@Component` annotation. A **more negative** value means **higher priority** — the filter runs earlier in the chain.

| Ranking Range | Use for |
|---|---|
| `-100` to `-500` | Authentication checks, security headers |
| `-500` to `-800` | Logging, monitoring |
| `-800` to `-1000` | Response modification, caching |

If two filters have the same ranking the execution order is not guaranteed. Always assign explicit rankings when order matters.

## The Filter Chain

Every filter receives a `FilterChain` object. Calling `chain.doFilter()` passes the request to the next filter in the chain, or to the servlet if no more filters remain.

```java
@Override
public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {

    // --- PRE-PROCESSING ---
    // Runs before the request reaches the servlet/component.
    // Auth checks, validation, request header modification go here.

    chain.doFilter(request, response); // ← MUST call this or the request is blocked

    // --- POST-PROCESSING ---
    // Runs after the response has been generated.
    // Add response headers, log status code, modify response body here.
}
```

If you do not call `chain.doFilter()` the request is blocked entirely and nothing further executes. The client receives only what your filter writes to the response. This is intentional for auth filters that must return a 401 or 403.

## Response Wrapping

If you need to read or modify the response body, you cannot do it directly because by the time your post-processing code runs the response has already been written. You must wrap the response before calling `chain.doFilter()`.

```java
// 1. Wrap the response to intercept the servlet's output
BufferedHttpResponseWrapper wrappedResponse =
    new BufferedHttpResponseWrapper((HttpServletResponse) response);

// 2. Let the servlet render into the buffer
chain.doFilter(request, wrappedResponse);

// 3. Read the buffered content
String html = wrappedResponse.getBufferedContent();

// 4. Modify and write to the real response
String modified = html.replace("</body>",
    "<script src='/etc/clientlibs/tracking.js'></script></body>");
response.setContentLength(modified.getBytes(response.getCharacterEncoding()).length);
response.getWriter().write(modified);
```

> **Performance warning:** Response wrapping is expensive because the full response body is held in memory. Only use it when you genuinely need to modify the output. Never use it on high-traffic paths without measuring the memory impact first.

## Disabling a Filter at Runtime

You can disable any filter without redeploying code by pushing an OSGi config that sets the `sling.filter.scope` property to an invalid value such as the string `disabled`. The Sling framework ignores filters with an unrecognised scope and the filter stops executing immediately. This is useful in production when a filter causes issues and you need to turn it off quickly without a full deployment.

## Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Forgetting `chain.doFilter()` in a conditional branch | Request is silently blocked for matching conditions | Trace every code path; ensure `chain.doFilter()` always executes when the request should proceed |
| Heavy work inside `doFilter()` | Blocks the request thread; any slow database/HTTP call/content traversal slows every request on that path | Delegate heavy work to a Sling Job |
| No `pattern` or `methods` restriction | Filter fires on every single request in AEM including internal Sling requests, clientlib requests, and system calls | Always scope your filter as tightly as possible |
| Modifying response headers after the response is committed | Headers are silently ignored | Set headers before calling `chain.doFilter()` or immediately after, while the response is still open |
| Not handling encoding correctly in response wrapping | Encoding mismatch on non-UTF-8 responses | Always use `response.getCharacterEncoding()`, never hardcode UTF-8 |

## Filters vs Event Handlers vs Schedulers

| | Sling Filter | Event Handler / Listener | Scheduler |
|---|---|---|---|
| Execution thread | Request thread — synchronous | Background event thread | Background scheduler thread |
| Triggered by | HTTP request | JCR/OSGi event | Cron expression or interval |
| Must be fast? | Yes — directly impacts request latency | Yes — backs up event queue | Less critical — no user waiting |
| Right tool for | Inspecting/modifying HTTP requests & responses | Reacting to content changes or system events | Periodic background tasks |

Filters are synchronous and run on the request thread — they must complete quickly or they degrade request performance directly. Event handlers and schedulers are asynchronous and run on background threads, suitable for heavier work. A filter is the wrong tool for background processing, scheduled work, or reacting to JCR changes.
