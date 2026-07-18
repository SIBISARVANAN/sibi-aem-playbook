# AEM JUnit Testing — Phase 4: Servlets & Filters

---

### 4.1 SlingSafeMethodsServlet (GET) Tests

#### What We Are Testing

`SlingSafeMethodsServlet` handles read-only GET/HEAD requests. Testing it means:
- Simulating an HTTP GET with parameters
- Reading the response body
- Asserting status codes, content type, and response content

#### The Servlet Under Test

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
        resourceTypes = "sibi-aem-one/services/propertysearch",
        methods       = HttpConstants.METHOD_GET,
        selectors     = "search",
        extensions    = "json"
)
public class PropertySearchServlet extends SlingSafeMethodsServlet {

    private static final long serialVersionUID = 1L;

    @Reference
    private PropertySearchService searchService;

    @Override
    protected void doGet(SlingHttpServletRequest request,
                         SlingHttpServletResponse response) throws IOException {

        String query     = request.getParameter("q");
        String types     = request.getParameter("types");
        int    page      = parseIntOrDefault(request.getParameter("page"), 0);
        int    pageSize  = parseIntOrDefault(request.getParameter("pageSize"), 12);

        if (StringUtils.isBlank(query)) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().write("{\"error\":\"query parameter 'q' is required\"}");
            return;
        }

        PropertySearchRequest req = PropertySearchRequest.builder()
                .fullText(query)
                .propertyTypes(splitParam(types))
                .page(page, pageSize)
                .build();

        PropertySearchResult result = searchService.searchProperties(
                req, request.getResourceResolver());

        response.setContentType("application/json");
        response.setCharacterEncoding("UTF-8");
        response.getWriter().write(new Gson().toJson(result));
    }

    private int parseIntOrDefault(String val, int def) {
        try { return val != null ? Integer.parseInt(val) : def; }
        catch (NumberFormatException e) { return def; }
    }

    private List<String> splitParam(String param) {
        if (StringUtils.isBlank(param)) return Collections.emptyList();
        return Arrays.asList(param.split(","));
    }
}
```

#### Full GET Servlet Test Class

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class PropertySearchServletTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private PropertySearchService searchService;

    private PropertySearchServlet servlet;

    @BeforeEach
    void setUp() {
        // Register mock service dependency
        ctx.registerService(PropertySearchService.class, searchService);

        // Create and activate the servlet
        servlet = ctx.registerInjectActivateService(new PropertySearchServlet());

        // Create a resource for the servlet to operate on
        ctx.create().resource("/content/mysite/en/search",
                "sling:resourceType", "sibi-aem-one/services/propertysearch");
        ctx.currentResource("/content/mysite/en/search");
    }

    // ── Happy path ─────────────────────────────────────────────────────────

    @Test
    void doGet_whenValidQueryProvided_returns200WithJsonBody() throws Exception {
        // Arrange: stub the service to return a known result
        PropertySearchResult mockResult = buildMockResult(3, false);
        when(searchService.searchProperties(any(), any())).thenReturn(mockResult);

        // Set request parameter
        ctx.request().addRequestParameter("q", "villa");

        // Act: call doGet directly — no HTTP server needed
        servlet.doGet(ctx.request(), ctx.response());

        // Assert: status code
        assertEquals(HttpServletResponse.SC_OK, ctx.response().getStatus());

        // Assert: content type
        assertEquals("application/json", ctx.response().getContentType());

        // Assert: response body contains expected data
        String body = ctx.response().getOutputAsString();
        assertNotNull(body);
        assertFalse(body.isEmpty(), "Response body should not be empty");
        assertTrue(body.contains("totalMatches"),
                "JSON response should contain totalMatches field");
    }

    @Test
    void doGet_whenQueryProvided_passesQueryToService() throws Exception {
        // Use ArgumentCaptor to inspect what was passed to the service
        ArgumentCaptor<PropertySearchRequest> requestCaptor =
                ArgumentCaptor.forClass(PropertySearchRequest.class);

        when(searchService.searchProperties(requestCaptor.capture(), any()))
                .thenReturn(buildMockResult(0, false));

        ctx.request().addRequestParameter("q", "beachfront villa");
        ctx.request().addRequestParameter("types", "villa,penthouse");
        ctx.request().addRequestParameter("page", "2");
        ctx.request().addRequestParameter("pageSize", "6");

        servlet.doGet(ctx.request(), ctx.response());

        // Verify the service received the correct request object
        PropertySearchRequest captured = requestCaptor.getValue();
        assertEquals("beachfront villa", captured.getFullTextQuery());
        assertEquals(2, captured.getPageNumber());
        assertEquals(6, captured.getPageSize());
        assertTrue(captured.getPropertyTypes().contains("villa"));
        assertTrue(captured.getPropertyTypes().contains("penthouse"));
    }

    // ── Validation / error paths ────────────────────────────────────────────

    @Test
    void doGet_whenQueryParameterMissing_returns400() throws Exception {
        // No "q" parameter added — should return 400 Bad Request
        servlet.doGet(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_BAD_REQUEST, ctx.response().getStatus());

        String body = ctx.response().getOutputAsString();
        assertTrue(body.contains("error"),
                "400 response should contain an error message");

        // Service should NOT be called when validation fails
        verifyNoInteractions(searchService);
    }

    @Test
    void doGet_whenQueryIsBlankString_returns400() throws Exception {
        ctx.request().addRequestParameter("q", "   "); // blank, not empty

        servlet.doGet(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_BAD_REQUEST, ctx.response().getStatus());
    }

    @Test
    void doGet_whenPageParameterIsInvalid_usesDefaultPage() throws Exception {
        ArgumentCaptor<PropertySearchRequest> captor =
                ArgumentCaptor.forClass(PropertySearchRequest.class);
        when(searchService.searchProperties(captor.capture(), any()))
                .thenReturn(buildMockResult(0, false));

        ctx.request().addRequestParameter("q", "villa");
        ctx.request().addRequestParameter("page", "notANumber"); // invalid

        servlet.doGet(ctx.request(), ctx.response());

        // parseIntOrDefault should fallback to 0
        assertEquals(0, captor.getValue().getPageNumber(),
                "Invalid page parameter should default to 0");
    }

    // ── Service failure handling ────────────────────────────────────────────

    @Test
    void doGet_whenServiceThrowsException_returns500() throws Exception {
        when(searchService.searchProperties(any(), any()))
                .thenThrow(new RuntimeException("Search engine down"));

        ctx.request().addRequestParameter("q", "villa");

        // The servlet should handle the exception gracefully
        // If no try-catch in servlet, the exception propagates —
        // this test documents the expected behaviour
        assertDoesNotThrow(() -> servlet.doGet(ctx.request(), ctx.response()),
                "Servlet should handle service exceptions without propagating");
    }

    // ── Content type and encoding ───────────────────────────────────────────

    @Test
    void doGet_always_setsJsonContentType() throws Exception {
        when(searchService.searchProperties(any(), any()))
                .thenReturn(buildMockResult(1, false));
        ctx.request().addRequestParameter("q", "villa");

        servlet.doGet(ctx.request(), ctx.response());

        assertEquals("application/json", ctx.response().getContentType());
        assertEquals("UTF-8", ctx.response().getCharacterEncoding());
    }

    // ── Helper ─────────────────────────────────────────────────────────────

    private PropertySearchResult buildMockResult(long total, boolean hasMore) {
        return new PropertySearchResult(
                Collections.emptyList(), total, hasMore,
                Collections.emptyMap(), 5L);
    }
}
```

---

### 4.2 SlingAllMethodsServlet (POST) Tests

#### The Servlet Under Test

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
        resourceTypes = "sibi-aem-one/services/openhouse",
        methods       = HttpConstants.METHOD_POST,
        extensions    = "json"
)
public class OpenHouseRsvpServlet extends SlingAllMethodsServlet {

    private static final long serialVersionUID = 1L;

    @Override
    protected void doPost(SlingHttpServletRequest request,
                          SlingHttpServletResponse response) throws IOException {

        // CSRF is already validated by AEM's CSRFFilter before we reach here
        String propertyId = request.getParameter("propertyId");
        String email      = request.getParameter("email");

        if (StringUtils.isAnyBlank(propertyId, email)) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().write("{\"error\":\"propertyId and email are required\"}");
            return;
        }

        if (!isValidEmail(email)) {
            response.setStatus(HttpServletResponse.SC_BAD_REQUEST);
            response.getWriter().write("{\"error\":\"invalid email format\"}");
            return;
        }

        // Process the RSVP
        response.setStatus(HttpServletResponse.SC_OK);
        response.setContentType("application/json");
        response.getWriter().write("{\"status\":\"ok\",\"propertyId\":\"" + propertyId + "\"}");
    }

    private boolean isValidEmail(String email) {
        return email != null && email.contains("@") && email.contains(".");
    }
}
```

#### POST Servlet Test Class

```java
@ExtendWith(AemContextExtension.class)
class OpenHouseRsvpServletTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);
    private OpenHouseRsvpServlet servlet;

    @BeforeEach
    void setUp() {
        servlet = ctx.registerInjectActivateService(new OpenHouseRsvpServlet());
        ctx.create().resource("/content/property",
                "sling:resourceType", "sibi-aem-one/services/openhouse");
        ctx.currentResource("/content/property");
        // Set the HTTP method to POST
        ctx.request().setMethod("POST");
    }

    @Test
    void doPost_whenValidParams_returns200WithOkStatus() throws Exception {
        ctx.request().addRequestParameter("propertyId", "PROP-001");
        ctx.request().addRequestParameter("email", "visitor@example.com");

        servlet.doPost(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_OK, ctx.response().getStatus());
        String body = ctx.response().getOutputAsString();
        assertTrue(body.contains("\"status\":\"ok\""));
        assertTrue(body.contains("PROP-001"));
    }

    @Test
    void doPost_whenPropertyIdMissing_returns400() throws Exception {
        // Only email, no propertyId
        ctx.request().addRequestParameter("email", "visitor@example.com");

        servlet.doPost(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_BAD_REQUEST, ctx.response().getStatus());
        assertTrue(ctx.response().getOutputAsString().contains("error"));
    }

    @Test
    void doPost_whenEmailMissing_returns400() throws Exception {
        ctx.request().addRequestParameter("propertyId", "PROP-001");
        // No email

        servlet.doPost(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_BAD_REQUEST, ctx.response().getStatus());
    }

    @Test
    void doPost_whenEmailFormatInvalid_returns400() throws Exception {
        ctx.request().addRequestParameter("propertyId", "PROP-001");
        ctx.request().addRequestParameter("email", "not-an-email"); // no @ or .

        servlet.doPost(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_BAD_REQUEST, ctx.response().getStatus());
        assertTrue(ctx.response().getOutputAsString().contains("invalid email format"));
    }

    @Test
    void doPost_whenBothParamsMissing_returns400() throws Exception {
        // No params at all

        servlet.doPost(ctx.request(), ctx.response());

        assertEquals(HttpServletResponse.SC_BAD_REQUEST, ctx.response().getStatus());
    }
}
```

---

### 4.3 Filter Chain Testing

#### Understanding Filter Testing

A Sling Filter has three parts to test:
1. **Pre-processing** — code before `chain.doFilter()` is called
2. **The chain call itself** — did it get called or was the request blocked?
3. **Post-processing** — code after `chain.doFilter()` returns

The key Mockito tool here is `mock(FilterChain.class)` — you verify whether `doFilter()` was called.

#### The Filters Under Test

```java
// Filter 1: Auth Token Filter — blocks requests with missing/invalid token
@Component(service = Filter.class)
@SlingServletFilter(scope = SlingServletFilterScope.REQUEST)
public class AuthTokenFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
                         FilterChain chain) throws IOException, ServletException {
        SlingHttpServletRequest  request  = (SlingHttpServletRequest) req;
        SlingHttpServletResponse response = (SlingHttpServletResponse) res;

        String token = request.getHeader("X-Auth-Token");
        if (StringUtils.isBlank(token) || !isValid(token)) {
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.getWriter().write("{\"error\":\"Unauthorized\"}");
            return; // chain.doFilter() NOT called — request is blocked
        }
        chain.doFilter(req, res); // passes through
    }

    private boolean isValid(String token) {
        return "valid-token-123".equals(token);
    }

    @Override public void init(FilterConfig filterConfig) {}
    @Override public void destroy() {}
}

// Filter 2: Security Headers Filter — adds headers AFTER chain
@Component(service = Filter.class)
@SlingServletFilter(scope = SlingServletFilterScope.REQUEST)
public class SecurityHeadersFilter implements Filter {

    @Override
    public void doFilter(ServletRequest req, ServletResponse res,
                         FilterChain chain) throws IOException, ServletException {
        chain.doFilter(req, res); // call chain first

        // Post-processing: add security headers to the response
        HttpServletResponse response = (HttpServletResponse) res;
        response.setHeader("X-Frame-Options", "SAMEORIGIN");
        response.setHeader("X-Content-Type-Options", "nosniff");
        response.setHeader("X-XSS-Protection", "1; mode=block");
    }

    @Override public void init(FilterConfig filterConfig) {}
    @Override public void destroy() {}
}
```

#### Auth Token Filter Tests

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class AuthTokenFilterTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    // Mock the filter chain — we verify whether doFilter() was called
    @Mock
    private FilterChain filterChain;

    private AuthTokenFilter filter;

    @BeforeEach
    void setUp() {
        filter = new AuthTokenFilter();
        ctx.create().resource("/content/secure/page");
        ctx.currentResource("/content/secure/page");
    }

    // ── Chain is called (request passes through) ───────────────────────────

    @Test
    void doFilter_whenValidTokenProvided_callsFilterChain() throws Exception {
        // Set the auth token header on the mock request
        ctx.request().addHeader("X-Auth-Token", "valid-token-123");

        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        // Verify chain.doFilter() WAS called — request passed through
        verify(filterChain, times(1))
            .doFilter(ctx.request(), ctx.response());

        // Status should be default 200 — not modified by the filter
        assertNotEquals(HttpServletResponse.SC_UNAUTHORIZED, ctx.response().getStatus());
    }

    // ── Chain is NOT called (request is blocked) ───────────────────────────

    @Test
    void doFilter_whenTokenMissing_blocksRequestWith401() throws Exception {
        // No X-Auth-Token header added

        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        // Verify chain.doFilter() was NEVER called
        verify(filterChain, never()).doFilter(any(), any());

        // Verify 401 was set on the response
        assertEquals(HttpServletResponse.SC_UNAUTHORIZED, ctx.response().getStatus());
    }

    @Test
    void doFilter_whenTokenIsInvalid_blocksRequestWith401() throws Exception {
        ctx.request().addHeader("X-Auth-Token", "wrong-token");

        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        verify(filterChain, never()).doFilter(any(), any());
        assertEquals(HttpServletResponse.SC_UNAUTHORIZED, ctx.response().getStatus());
    }

    @Test
    void doFilter_whenTokenIsBlank_blocksRequestWith401() throws Exception {
        ctx.request().addHeader("X-Auth-Token", "   "); // blank string

        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        verify(filterChain, never()).doFilter(any(), any());
        assertEquals(HttpServletResponse.SC_UNAUTHORIZED, ctx.response().getStatus());
    }

    @Test
    void doFilter_whenTokenMissing_writesUnauthorizedBody() throws Exception {
        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        String body = ctx.response().getOutputAsString();
        assertTrue(body.contains("Unauthorized"),
                "401 response body should contain Unauthorized message");
    }
}
```

#### Security Headers Filter Tests

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class SecurityHeadersFilterTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private FilterChain filterChain;

    private SecurityHeadersFilter filter;

    @BeforeEach
    void setUp() {
        filter = new SecurityHeadersFilter();
        ctx.create().resource("/content/page");
        ctx.currentResource("/content/page");
    }

    @Test
    void doFilter_always_callsFilterChainFirst() throws Exception {
        // The security headers filter must always call chain.doFilter()
        // It's a post-processing filter — it never blocks requests
        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        verify(filterChain, times(1))
            .doFilter(ctx.request(), ctx.response());
    }

    @Test
    void doFilter_always_addsXFrameOptionsHeader() throws Exception {
        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        // Post-processing: verify headers were set on the response
        assertEquals("SAMEORIGIN",
                ctx.response().getHeader("X-Frame-Options"),
                "X-Frame-Options header must be set");
    }

    @Test
    void doFilter_always_addsContentTypeOptionsHeader() throws Exception {
        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        assertEquals("nosniff",
                ctx.response().getHeader("X-Content-Type-Options"));
    }

    @Test
    void doFilter_always_addsXssProtectionHeader() throws Exception {
        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        assertEquals("1; mode=block",
                ctx.response().getHeader("X-XSS-Protection"));
    }

    @Test
    void doFilter_headersAddedAfterChainNotBefore() throws Exception {
        // Verify ordering: chain runs first, THEN headers are added.
        // Use doAnswer to check response headers inside the chain call.
        doAnswer(invocation -> {
            // At the moment chain.doFilter() is called,
            // the security headers should NOT yet be present
            assertNull(ctx.response().getHeader("X-Frame-Options"),
                    "Headers should not be set before chain.doFilter() is called");
            return null;
        }).when(filterChain).doFilter(any(), any());

        filter.doFilter(ctx.request(), ctx.response(), filterChain);

        // After doFilter() returns, headers should now be present
        assertNotNull(ctx.response().getHeader("X-Frame-Options"),
                "Headers should be present after chain.doFilter() completes");
    }
}
```

---

### 4.4 Request and Response Mocking Details

#### Setting Up Request Headers

```java
// Single header
ctx.request().addHeader("X-Auth-Token", "valid-token-123");
ctx.request().addHeader("Content-Type", "application/json");
ctx.request().addHeader("Accept-Language", "en-GB");

// Verify in your servlet/filter code:
// String token = request.getHeader("X-Auth-Token");
```

#### Setting Request Parameters

```java
// Single parameter
ctx.request().addRequestParameter("q", "beachfront villa");

// Multiple parameters
ctx.request().addRequestParameter("types", "villa,apartment");
ctx.request().addRequestParameter("page", "2");
ctx.request().addRequestParameter("pageSize", "12");

// Multi-value parameter (same name, multiple values)
ctx.request().addRequestParameter("tag", "mysite:amenities/pool");
ctx.request().addRequestParameter("tag", "mysite:amenities/gym");
// In servlet: request.getParameterValues("tag") returns String[]{"pool", "gym"}
```

#### Setting HTTP Method

```java
// Default is GET — override for POST/PUT/DELETE tests
ctx.request().setMethod("POST");
ctx.request().setMethod("DELETE");
ctx.request().setMethod("PUT");
```

#### Setting Request Path and Selectors

```java
// Set the resource the request targets
ctx.currentResource("/content/mysite/en/search");

// Simulate a URL like: /content/mysite/en/search.search.json
ctx.request().setSelectors(new String[]{"search"});
ctx.request().setExtension("json");
```

#### Reading the Response

```java
// Get the response body as a String
String body = ctx.response().getOutputAsString();

// Get the HTTP status code
int status = ctx.response().getStatus();

// Get a specific response header
String contentType = ctx.response().getContentType();
String customHeader = ctx.response().getHeader("X-Frame-Options");

// Check if a header was set
boolean hasHeader = ctx.response().containsHeader("X-Custom-Header");
```

#### Full Request/Response Setup — Complete Example

```java
@Test
void doGet_withAllRequestOptions_returnsCorrectResponse() throws Exception {
    // ── Request setup ──────────────────────────────────────────────────────
    ctx.request().setMethod("GET");
    ctx.request().addHeader("Accept", "application/json");
    ctx.request().addHeader("X-Forwarded-For", "192.168.1.1");
    ctx.request().addRequestParameter("q", "villa");
    ctx.request().addRequestParameter("page", "0");
    ctx.currentResource("/content/mysite/en/search");

    // ── Execute ────────────────────────────────────────────────────────────
    servlet.doGet(ctx.request(), ctx.response());

    // ── Response assertions ────────────────────────────────────────────────
    assertEquals(200, ctx.response().getStatus());
    assertEquals("application/json", ctx.response().getContentType());
    assertEquals("UTF-8", ctx.response().getCharacterEncoding());

    String body = ctx.response().getOutputAsString();
    assertFalse(body.isBlank(), "Response body should not be empty");

    // Parse the JSON and assert structure
    // Using Gson for simplicity (same lib used in the servlet)
    JsonObject json = new Gson().fromJson(body, JsonObject.class);
    assertTrue(json.has("totalMatches"), "Response should contain totalMatches");
    assertTrue(json.has("hits"),         "Response should contain hits array");
}
```

---

### 4.5 Testing the XSSAPI Filter (Combining Filter + Service)

This example tests `PropertySearchServlet` which uses `XSSAPI` to escape output — showing how to test a servlet that has an OSGi service dependency alongside request/response testing:

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class XssApiServletTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private XSSAPI xssAPI;

    private PropertySearchServlet servlet;

    @BeforeEach
    void setUp() {
        // Stub XSSAPI methods to return safe versions of inputs
        when(xssAPI.encodeForHTML(anyString()))
            .thenAnswer(inv -> "SAFE:" + inv.getArgument(0));
        when(xssAPI.encodeForHTMLAttr(anyString()))
            .thenAnswer(inv -> "SAFEATTR:" + inv.getArgument(0));

        ctx.registerService(XSSAPI.class, xssAPI);
        servlet = ctx.registerInjectActivateService(new PropertySearchServlet());

        ctx.create().resource("/content/search",
                "sling:resourceType", "sibi-aem-one/services/propertysearch");
        ctx.currentResource("/content/search");
    }

    @Test
    void doGet_whenXssAttackInQueryParam_encodesOutput() throws Exception {
        String maliciousInput = "<script>alert('xss')</script>";
        ctx.request().addRequestParameter("q", maliciousInput);

        servlet.doGet(ctx.request(), ctx.response());

        // Verify XSSAPI was called to encode the output
        verify(xssAPI, atLeastOnce()).encodeForHTML(maliciousInput);

        // The raw script tag should NOT appear in the output
        String body = ctx.response().getOutputAsString();
        assertFalse(body.contains("<script>"),
                "Raw script tag must not appear in encoded output");
    }
}
```

---

### Phase 4 — Summary

| Topic | Key Takeaways |
|---|---|
| GET servlet | `ctx.request().addRequestParameter()` to set params. Read response with `ctx.response().getOutputAsString()`. Call `servlet.doGet()` directly. |
| POST servlet | `ctx.request().setMethod("POST")` before calling `servlet.doPost()`. Test each validation branch separately. |
| Filter chain | `mock(FilterChain.class)`. `verify(chain, times(1)).doFilter()` for pass-through. `verify(chain, never()).doFilter()` for blocked. |
| Pre-processing | Code before `chain.doFilter()` — test headers/params that affect the decision. |
| Post-processing | Code after `chain.doFilter()` — use `doAnswer()` to verify ordering if important. |
| Response assertions | `ctx.response().getStatus()`, `getContentType()`, `getHeader()`, `getOutputAsString()`. |
| Service in servlet | `ctx.registerService()` before `registerInjectActivateService()`. Mock the service response. |
| ArgumentCaptor in servlet tests | Capture the object passed to the service to verify mapping from HTTP params to DTO. |