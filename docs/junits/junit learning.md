## Phase 1 — Foundation

**Testing stack and pom.xml setup**
- Exact dependencies to add: `aem-mock-junit5`, `mockito-core`, `junit-jupiter`, `sling-mock`
- Version compatibility matrix — which aem-mock version works with which AEM version
- Difference between `aem-mock-junit4` vs `aem-mock-junit5` and why it matters
- How the `maven-surefire-plugin` must be configured for JUnit 5 to actually run

**AemContext — JCR_MOCK vs JCR_OAK vs RESOURCERESOLVER_MOCK**
- What each one provides internally and what it cannot do
- When `JCR_MOCK` is enough (most Sling Model tests)
- When you must use `JCR_OAK` (QueryBuilder tests, JCR query execution)
- When `RESOURCERESOLVER_MOCK` is sufficient (pure property reading, no JCR operations)
- Performance implications of each — `JCR_OAK` spins up a real Oak instance, noticeably slower

**JUnit 4 vs JUnit 5 in AEM**
- The annotation mapping: `@RunWith` → `@ExtendWith`, `@Before` → `@BeforeEach`, `@Test` (both), `@Rule` → no equivalent
- Why AEM Maven archetype projects still default to JUnit 4 and how to migrate
- Mixing JUnit 4 and JUnit 5 in the same project — the vintage engine dependency

**JSON Fixtures**
- How `ctx.load().json("/path/to/fixture.json", "/jcr/path")` maps to JCR nodes
- Exact JSON structure that maps to `cq:Page` with `jcr:content`
- JSON structure for multifield child nodes
- Organising fixture files under `src/test/resources`

**Test naming conventions**
- `methodName_scenario_expectedOutcome` pattern
- Why this matters in CI output (Surefire reports show method names)

---

## Phase 2 — Sling Models

**Basic model test**
- Full `AemContext` setup with `addModelsForClasses()`
- Loading fixture JSON and setting `currentResource`
- Adapting from `Resource` vs adapting from `SlingHttpServletRequest`
- Asserting field values from `@ValueMapValue` injections
- Testing `@Default` fallback values when properties are absent

**@PostConstruct testing**
- How `@PostConstruct` is automatically called during `adaptTo()` — you never call it manually
- Testing derived fields that `@PostConstruct` computes from injected values
- Testing null-safety: what happens when a required resource is missing

**@ChildResource (multifield) testing**
- Setting up nested JSON fixture for multifield child nodes
- Verifying `List<ChildModel>` is populated correctly
- Testing empty multifield (no children) — list should be empty, not null

**@OSGiService injection in a Sling Model**
- `ctx.registerService(MyService.class, mockImpl)` before `adaptTo()`
- Why registration order matters — register services BEFORE adapting the model
- Testing when the OSGi service is unavailable (not registered) with OPTIONAL strategy

**Jackson exporter output**
- `ctx.request().adaptTo(MyModel.class)` on a model with `@Exporter`
- How to actually invoke the exporter and read the JSON output
- Verifying `@JsonIgnore` fields are absent from output
- Verifying `@JsonProperty` renamed fields appear correctly
- Verifying custom `JsonSerializer` output

---

## Phase 3 — OSGi Services and Lifecycle

**Testing @Activate / @Modified / @Deactivate**
- Calling `ctx.registerInjectActivateService(new MyServiceImpl(), configMap)` to inject OSGi config AND activate in one step
- Calling `activate()` / `modified()` directly via reflection when the method is `protected`
- Verifying state changes after `@Modified` (cache cleared, client reinitialised)
- Verifying cleanup in `@Deactivate` (HTTP client closed, cache emptied)

**OSGi config injection**
- `Map<String, Object>` passed as the config map
- Matching config map keys to `@interface Config` method names (underscore-to-dot rule in tests too)
- Testing with partial config — what defaults apply for unprovided keys

**Mocking dependent services**
- `@Mock` + `@InjectMocks` for pure unit tests with no AemContext
- `ctx.registerService()` for integration-style tests that use AemContext
- Testing service ranking — registering two implementations and verifying which one wins

---

## Phase 4 — Servlets and Filters

**SlingSafeMethodsServlet (GET) tests**
- Setting up `MockSlingHttpServletRequest` and `MockSlingHttpServletResponse`
- Setting request parameters: `request.addRequestParameter("q", "villa")`
- Reading the response body from `MockSlingHttpServletResponse.getOutputAsString()`
- Asserting content type and character encoding
- Asserting HTTP status code

**SlingAllMethodsServlet (POST) tests**
- Setting the HTTP method: `request.setMethod("POST")`
- Setting request body / form parameters
- Testing the CSRF token header presence (mock-level verification)
- Testing 403 response when validation fails

**Filter chain testing**
- Mocking `FilterChain` with Mockito: `mock(FilterChain.class)`
- Verifying `chain.doFilter()` was called (normal flow)
- Verifying `chain.doFilter()` was NOT called (blocked request — auth filter rejecting)
- Testing pre-processing logic (code before `chain.doFilter()`)
- Testing post-processing logic (code after `chain.doFilter()`)
- Verifying response headers added by a security headers filter

**Request/Response mocking details**
- `ctx.request().addHeader("X-Auth-Token", "value")` for header injection
- `ctx.request().setResource(resource)` for resource-type-based filters
- `MockSlingHttpServletResponse` — how to read buffered output

---

## Phase 5 — Schedulers, Jobs and Listeners

**Scheduler testing**
- Why schedulers are the easiest thing to test: just call `scheduler.run()`
- Setting up the scheduler with a config map via `ctx.registerInjectActivateService()`
- Verifying side effects of `run()` — did it call the expected service method?
- Testing the `enabled=false` path — `run()` should be a no-op

**Sling Job Consumer testing**
- Creating a mock `Job` object: `mock(Job.class)`
- Stubbing `job.getProperty("myKey", String.class)` return values
- Calling `consumer.process(job)` directly
- Asserting `JobResult.OK` vs `JobResult.FAILED` return value
- Verifying the downstream service was called with the correct arguments

**Sling Job Producer testing**
- Mocking `JobManager`: `mock(JobManager.class)`
- Using `ArgumentCaptor<Map>` to capture the properties map passed to `addJob()`
- Asserting the captured map contains the expected keys and values

**ResourceChangeListener testing**
- Constructing a `ResourceChange` object with a specific path and change type
- Calling `listener.onChange(List.of(change))` directly
- Verifying `JobManager.addJob()` was called (or not called) based on the change type
- Testing `isExternal()` branching — mock two changes, one external and one local

**EventHandler testing**
- Constructing a mock OSGi `Event` with topic and properties
- Calling `handler.handleEvent(event)` directly
- Verifying downstream behaviour (job enqueued, cache cleared, etc.)

---

## Phase 6 — JCR / Repository Mocking

**ResourceResolver and ValueMap mocking**
- `ctx.resourceResolver()` — the AemContext-provided resolver
- `ctx.create().resource("/path", Map.of("key","value"))` — creating resources programmatically
- `resource.getValueMap()` returning expected values in tests
- Mocking `ResourceResolver.getResource()` returning null — testing the null-safe path

**Session and Node mocking**
- When you need raw JCR `Session` in a test
- `resolver.adaptTo(Session.class)` in JCR_MOCK context
- Mocking `Node.getProperty()` and `Property.getString()` chains
- Avoiding deep mock chains — why flat ValueMap access is easier to test

**TagManager and Tag mocking**
- `ctx.registerAdapter(ResourceResolver.class, TagManager.class, mockTagManager)`
- Stubbing `tagManager.resolve("mysite:topic/aem")` to return a mock `Tag`
- Stubbing `tag.getTitle(Locale.ENGLISH)` to return "AEM Development"
- Testing the null fallback when a tag ID doesn't resolve

**PageManager and Page mocking**
- `ctx.registerAdapter(ResourceResolver.class, PageManager.class, mockPageManager)`
- Stubbing `pageManager.getContainingPage(resource)` to return a mock `Page`
- Stubbing `page.getPath()`, `page.getTitle()`, `page.getProperties()`
- `ctx.create().page("/content/mysite/en/home")` — creating real mock pages in AemContext

**AssetManager mocking**
- `ctx.registerAdapter(ResourceResolver.class, AssetManager.class, mockAssetManager)`
- Stubbing `assetManager.getAsset("/content/dam/image.jpg")` returning a mock `Asset`
- Stubbing `asset.getMetadataValue("dc:title")` returning a value
- Testing when asset path doesn't exist — `getAsset()` returns null

---

## Phase 7 — QueryBuilder

**Real QueryBuilder with JCR_OAK**
- Why `JCR_OAK` is mandatory for QueryBuilder tests (JCR_MOCK has no query engine)
- Setting up `AemContext(ResourceResolverType.JCR_OAK)`
- Registering the QueryBuilder service: `ctx.registerService(QueryBuilder.class, ...)`
- Loading test content and running real queries against it
- Asserting correct paths are returned

**Mocked QueryBuilder and SearchResult**
- When mocking QueryBuilder is better than JCR_OAK (faster, no index setup)
- `mock(QueryBuilder.class)`, `mock(Query.class)`, `mock(SearchResult.class)`
- Stubbing the chain: `queryBuilder.createQuery()` → `query.getResult()` → `result.getHits()`
- Creating mock `Hit` objects with stubbed `getPath()` and `getResource()`
- Testing pagination: stubbing `result.getTotalMatches()` and `result.hasMore()`

**Facet result mocking**
- Mocking `result.getFacets()` returning a `Map<String, Facet>`
- Mocking `Facet.getBuckets()` returning a list of `Bucket` mocks
- Stubbing `bucket.getValue()` and `bucket.getCount()`
- Verifying the service correctly transforms facet data into the result DTO

---

## Phase 8 — Workflows

**WorkflowProcess testing**
- Mocking `WorkItem`, `WorkflowSession`, `MetaDataMap` (args), `WorkflowData`
- Stubbing `workItem.getWorkflowData().getPayload().toString()` — the payload path
- Stubbing `args.get("PROCESS_ARGS", String.class)` — process arguments from model
- Calling `process.execute(workItem, workflowSession, args)` directly
- Verifying `WorkflowData.getMetaDataMap().put()` was called with expected key/value
- Testing `WorkflowException` is thrown when expected

**WorkflowSession and WorkItem mocking**
- Full mock setup for a multi-step workflow test
- `workItem.getMetaDataMap()` vs `workItem.getWorkflowData().getMetaDataMap()` — which is which
- Stubbing `workflowSession.getSession()` to return a mock JCR `Session`

**Replicator mocking (in workflow steps)**
- `mock(Replicator.class)` and injecting into the workflow process
- Verifying `replicator.replicate(session, ACTIVATE, path)` was called
- Testing the exception path: `when(replicator.replicate(...)).thenThrow(ReplicationException.class)`
- Verifying `WorkflowData` metadata is set to "FAILED" when replication throws

**ContentFragment mocking**
- `mock(ContentFragment.class)` and `mock(ContentElement.class)`
- Stubbing `fragment.getElement("title").getContent()` returning a value
- Stubbing `fragment.getElement("score").getValue(Integer.class)` for numeric fields
- Stubbing `fragment.getElement("tags").getValue(String[].class)` for multi-value
- Testing null element — `fragment.getElement("nonExistent")` returns null

---

## Phase 9 — Advanced Mockito

**Spy**
- `spy(new MyServiceImpl())` — real object with selectively overridden methods
- When to spy vs when to mock — spy when you want MOST real behaviour but stub one expensive method
- `doReturn()` vs `when()` on spies — why `when(spy.method())` can call the real method and cause issues
- Spying on AemContext-registered services

**ArgumentCaptor**
- `ArgumentCaptor<Map> captor = ArgumentCaptor.forClass(Map.class)`
- `verify(mock).addJob(eq(TOPIC), captor.capture())`
- `captor.getValue()` to get what was passed
- `captor.getAllValues()` when the method was called multiple times
- Practical use: verifying the exact predicate map passed to QueryBuilder, or the exact properties map passed to JobManager

**Static Mocking (MockedStatic)**
- JUnit 5 only: `try (MockedStatic<MyUtil> mocked = Mockito.mockStatic(MyUtil.class))`
- Stubbing `mocked.when(() -> MyUtil.someMethod(args)).thenReturn(value)`
- Common AEM use case: mocking `ZonedDateTime.now()`, `System.currentTimeMillis()`, or a utility class
- Why static mocking requires `mockito-inline` dependency (not in default mockito-core)
- Closing scope — why try-with-resources is mandatory

**Reflection (ReflectionTestUtils / Field injection)**
- `ReflectionTestUtils.setField(target, "fieldName", value)` — injecting private fields when no setter exists
- Alternative: `FieldUtils.writeField(target, "fieldName", value, true)` from Apache Commons
- When to use reflection vs refactoring for testability
- Accessing private methods: `method.setAccessible(true)` and when this is a design smell

**@InjectMocks vs ctx.registerService()**
- `@InjectMocks` — pure Mockito, no AemContext, fast, no Sling runtime
- `ctx.registerService()` — uses AemContext, supports `@OSGiService` injection in Sling Models
- Rule of thumb: `@InjectMocks` for OSGi services; `ctx.registerService()` for Sling Models

**Parameterised tests**
- `@ParameterizedTest` + `@ValueSource(strings = {"villa","apartment","house"})` — run same test with different inputs
- `@CsvSource({"villa,available,200000", "apartment,sold,500000"})` — multiple parameters per row
- `@MethodSource("provideSearchRequests")` — complex object parameters from a static factory method
- Real AEM use case: testing all property types return the correct formatted price

**Testing exception and edge case paths**
- `assertThrows(WorkflowException.class, () -> process.execute(...))` — JUnit 5 style
- `when(mock.method()).thenThrow(new RuntimeException())` — simulating service failure
- Testing null input: what does your model do when `request.getParameter("q")` returns null?
- Testing empty string vs null — common source of NullPointerException in production

---

## Phase 10 — Cheat Sheet and JaCoCo Setup

**Master cheat sheet**
- One-liner setup patterns for every component type (model, service, servlet, filter, job, listener, workflow)
- Complete mock stub chains for every common AEM object
- Common assertion patterns
- Common pitfalls and their fixes in a table

**JaCoCo setup**
- Adding `jacoco-maven-plugin` to `core/pom.xml`
- Configuring minimum coverage threshold (build fails below X%)
- Excluding generated classes and model interfaces from coverage
- Running `mvn verify` to generate the HTML coverage report
- Where the report lives: `target/site/jacoco/index.html`
- Integrating with SonarQube (as used in Cloud Manager quality gate)

---

