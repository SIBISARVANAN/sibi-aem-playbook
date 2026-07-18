# Unit Testing in AEM — Stack Overview

> This is a general overview of the AEM testing stack and common patterns. For the full step-by-step JUnit build-up (OSGi mocking, ResourceResolver/ValueMap, QueryBuilder, WorkflowProcess, Replicator, ContentFragment, Mockito Spy/ArgumentCaptor/MockedStatic, parameterized tests, and the MockitoExtension vs AemContextExtension decision table), see `/docs/testing/`.

## Testing Stack

| Library | Purpose |
|---|---|
| `io.wcm.testing.aem-mock-junit5` | Core AEM mock framework — provides `AemContext`, mock JCR, mock Sling |
| `org.mockito:mockito-core` | Mocking OSGi services and external dependencies |
| `org.junit.jupiter:junit-jupiter` | JUnit 5 test runner |
| `org.apache.sling:org.apache.sling.testing.sling-mock` | Underlying Sling mock |

## Testing a Sling Model

```java
@ExtendWith(AemContextExtension.class)
class AuthorImplTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        ctx.addModelsForClasses(AuthorImpl.class);
        // Load test content from a JSON fixture
        ctx.load().json("/content/test-author.json", "/content/author");
        ctx.currentResource("/content/author");
    }

    @Test
    void testGetFirstName() {
        Author model = ctx.request().adaptTo(Author.class);
        assertNotNull(model, "Model should not be null");
        assertEquals("Sibi", model.getFirstName());
    }

    @Test
    void testDefaultValues() {
        // Resource with no properties set — test defaults
        ctx.create().resource("/content/empty", new HashMap<>());
        ctx.currentResource("/content/empty");
        Author model = ctx.request().adaptTo(Author.class);
        assertEquals("Sibi", model.getFirstName()); // default value
    }
}
```

### Test JSON Fixture (`/content/test-author.json`)

```json
{
  "jcr:primaryType": "nt:unstructured",
  "firstName": "Sibi",
  "lastName": "Sarvanan",
  "gender": "Male",
  "email": "sibi@example.com",
  "author:title": "Mr"
}
```

## Testing an OSGi Service with Mockito

```java
@ExtendWith(MockitoExtension.class)
class ExternalApiServiceImplTest {

    @Mock
    private CloseableHttpClient httpClient;

    @InjectMocks
    private ExternalApiServiceImpl service;

    @Test
    void testFetchProductData_success() throws Exception {
        // Arrange
        CloseableHttpResponse mockResponse = mock(CloseableHttpResponse.class);
        StatusLine statusLine = mock(StatusLine.class);
        HttpEntity entity = new StringEntity("{\"sku\":\"ABC123\"}");

        when(statusLine.getStatusCode()).thenReturn(200);
        when(mockResponse.getStatusLine()).thenReturn(statusLine);
        when(mockResponse.getEntity()).thenReturn(entity);
        when(httpClient.execute(any(HttpGet.class))).thenReturn(mockResponse);

        // Act
        String result = service.fetchProductData("ABC123");

        // Assert
        assertNotNull(result);
        assertTrue(result.contains("ABC123"));
    }
}
```

## Testing a Scheduler

```java
@Test
void testSchedulerRun() {
    // Schedulers are simple Runnables — just call run() directly
    SimpleScheduledTask task = new SimpleScheduledTask();
    // inject mocks via reflection or constructor
    assertDoesNotThrow(task::run);
}
```

## Common Interview Questions

**Q: What is `AemContext` and what does it provide?**
`AemContext` is the central test fixture from wcm.io's aem-mock library. It provides a mock JCR repository, mock `ResourceResolver`, mock `SlingHttpServletRequest`/`Response`, a model factory, and helper methods to create resources and load JSON fixtures — all in-memory with no actual AEM instance needed.

**Q: What is the difference between `ResourceResolverType.JCR_MOCK` and `ResourceResolverType.JCR_OAK`?**
`JCR_MOCK` is a lightweight in-memory mock — fast but doesn't support JCR queries. `JCR_OAK` spins up a real in-memory Oak repository — slower but supports full JCR query execution. Use `JCR_MOCK` for model and service tests; use `JCR_OAK` when you need to test QueryBuilder logic.

**Q: How do you test a component that uses `@OSGiService`?**
Register a mock implementation with `ctx.registerService(MyService.class, mockImpl)` before calling `ctx.request().adaptTo(MyModel.class)`. The `AemContext` will inject the registered mock.
