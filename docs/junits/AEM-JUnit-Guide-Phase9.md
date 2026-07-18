# AEM JUnit Testing Guide — Phase 9: Advanced Mockito

**Scenario used throughout:** the `PropertyListingService`/search/workflow stack built across Phases 6–8 — this phase focuses on *technique*, not a new domain area, so examples pull from property pricing, search predicates, job scheduling, and property-type validation.

---

## 9.0 Concept — Why "Advanced" Mockito

Phases 6–8 covered mocking AEM's own APIs. Phase 9 covers Mockito techniques that apply *regardless* of AEM — partial mocking, capturing arguments for deep verification, mocking static methods (normally impossible), reflection-based test setup, the `@InjectMocks` vs `AemContext` decision, and JUnit 5's parameterized test machinery. These are the tools that separate "tests that compile and pass" from tests that actually catch the bugs your Property Listing services will have in production.

---

## 9.1 Spy

### 9.1.1 Concept
`mock()` creates an object with **no real behavior** — every method returns a default (null/0/false) unless stubbed. `spy()` wraps a **real object instance** — every method runs its real implementation unless you explicitly override it. Use a spy when you want most of an object's genuine logic to execute (so you're testing real integration between its methods) but need to neutralize one expensive, external, or non-deterministic piece (a network call, a slow computation, `System.currentTimeMillis()`-based logic).

### 9.1.2 Technicalities to know
- **When to spy vs when to mock:** mock when you're testing a class's interaction with a *collaborator* and don't care about the collaborator's real logic (typical unit-test isolation). Spy when the class **under test itself** has one problematic method (e.g., `calculateMarketTrendScore()` calls an external pricing API) but you want the rest of its real logic — including calls between its own methods — to genuinely execute.
- **`doReturn()` vs `when()` on spies — this is the single most important spy gotcha.** `when(spy.expensiveMethod()).thenReturn(x)` **actually calls the real `expensiveMethod()` first** (to record what invocation is being stubbed), and only *then* wires up the stub. If `expensiveMethod()` has side effects (an HTTP call, a DB write, throwing an exception) or is slow, `when(...)` triggers all of that immediately, every time, even though you intended to bypass it. `doReturn(x).when(spy).expensiveMethod()` never invokes the real method — it stubs by matching on the call afterward without executing it. **Rule of thumb: always use `doReturn()`/`doThrow()`/`doNothing()` on spies, and reserve `when()` for plain mocks.**
- A spy is a real object — instance fields set in its constructor are real and populated; only stubbed methods are intercepted. Don't be surprised when unstubbed methods on a spy produce real (sometimes unexpected, e.g. network-dependent) results — that's exactly what spying means, and if that's undesirable for a given method, stub it too.

### 9.1.3 spy(new MyServiceImpl()) — real object with selectively overridden methods

```java
@ExtendWith(MockitoExtension.class)
class PropertyPricingServiceSpyTest {

    @Test
    void shouldUseRealDiscountLogicButStubExpensiveMarketDataCall() {
        PropertyPricingServiceImpl realService = new PropertyPricingServiceImpl();
        PropertyPricingServiceImpl spyService = spy(realService);

        // Bypass the real (slow/external) market trend lookup
        doReturn(1.05).when(spyService).fetchMarketTrendMultiplier("Chennai");

        // finalPrice() internally calls fetchMarketTrendMultiplier() AND real discount logic
        long finalPrice = spyService.finalPrice(15000000L, "Chennai", 10); // 10% discount

        // Real discount math (15,000,000 * 0.9) * real market multiplier stub (1.05)
        assertEquals(14175000L, finalPrice);
        verify(spyService).fetchMarketTrendMultiplier("Chennai");
    }
}
```

### 9.1.4 doReturn() vs when() — demonstrating the failure mode

```java
@Test
void whenOnSpyCanTriggerRealMethodUnexpectedly() {
    PropertyPricingServiceImpl spyService = spy(new PropertyPricingServiceImpl());

    // ANTI-PATTERN — do not do this on a spy:
    // when(spyService.fetchMarketTrendMultiplier("Chennai")).thenReturn(1.05);
    // The line above calls the REAL fetchMarketTrendMultiplier("Chennai") first,
    // which in production hits a live pricing API — this can throw, time out,
    // or (worse) silently succeed with a real network call inside a "unit" test.

    // CORRECT — doReturn never invokes the real method:
    doReturn(1.05).when(spyService).fetchMarketTrendMultiplier("Chennai");

    assertEquals(1.05, spyService.fetchMarketTrendMultiplier("Chennai"));
}
```

### 9.1.5 Spying on AemContext-registered services

```java
@Test
void shouldSpyOnServiceRegisteredThroughAemContext() {
    PropertyListingServiceImpl realImpl = new PropertyListingServiceImpl();
    PropertyListingServiceImpl spyImpl = spy(realImpl);

    // Register the SPY (not a fresh instance) so AemContext-driven Sling Models / OSGi
    // components that reference PropertyListingService receive the spy
    context.registerService(PropertyListingService.class, spyImpl);

    doReturn(List.of()).when(spyImpl).getSimilarProperties(any(Page.class), anyString());

    PropertyListingService injected = context.getService(PropertyListingService.class);

    assertSame(spyImpl, injected);
    // Any other real method on PropertyListingService still executes genuinely
}
```

**Edge case worth testing:** spying on a class with `final` methods — Mockito can spy on final classes/methods only when using the `mockito-inline` (or Mockito 5's default inline mock maker); confirm your project's Mockito setup before relying on this, since older `mockito-core`-only setups silently produce a spy that can't override final methods and will always run the real final method regardless of `doReturn()`.

---

## 9.2 ArgumentCaptor

### 9.2.1 Concept
`verify(mock).someMethod(x)` confirms a call happened with an argument matching `x` — but sometimes you don't know the exact expected value ahead of time (a generated timestamp, a computed predicate map), or you want to assert on multiple *properties* of a complex argument rather than exact equality. `ArgumentCaptor` intercepts the actual argument(s) passed during the call so you can inspect them after the fact with normal assertions.

### 9.2.2 Technicalities to know
- Declare the captor with the argument's type: `ArgumentCaptor<Map> captor = ArgumentCaptor.forClass(Map.class)` — due to Java generics erasure, the captor is technically `ArgumentCaptor<Map>` not `ArgumentCaptor<Map<String,String>>`; use `@Captor` field annotation with `MockitoExtension` for slightly cleaner generic handling if your team prefers it.
- Use `captor.capture()` **in place of** the argument in the `verify(...)` call — it must be mixed with matchers correctly: if any argument in the call uses a Mockito matcher (`eq()`, `any()`, `captor.capture()`), **all** arguments in that call must use matchers, not a mix of raw values and matchers, or Mockito throws `InvalidUseOfMatchersException`.
- `captor.getValue()` returns the **last** captured value — if the method was invoked multiple times, this silently only gives you the final call's argument, which is a common source of tests that pass despite missing a check on an earlier, different call.
- `captor.getAllValues()` returns a `List` of every captured invocation's argument, in call order — this is the correct choice whenever the mocked method may be called more than once in the scenario under test (e.g., a batch job that calls `queue.addJob()` once per property).

### 9.2.3 Basic setup — ArgumentCaptor<Map>

```java
@ExtendWith(MockitoExtension.class)
class PropertyIndexingJobServiceTest {

    @Mock private JobManager jobManager;

    @InjectMocks
    private PropertyIndexingJobService indexingService;

    @Captor
    private ArgumentCaptor<Map<String, Object>> jobPropertiesCaptor;

    @Test
    void shouldQueueIndexingJobWithCorrectPropertyPath() {
        indexingService.queueIndexingJob(jobManager, "/content/properties/chennai/villa-101");

        verify(jobManager).addJob(eq("realestate/indexing"), jobPropertiesCaptor.capture());

        Map<String, Object> capturedProps = jobPropertiesCaptor.getValue();
        assertEquals("/content/properties/chennai/villa-101", capturedProps.get("path"));
        assertNotNull(capturedProps.get("queuedAt"));
    }
}
```

### 9.2.4 captor.getAllValues() when the method was called multiple times

```java
@Test
void shouldQueueOneJobPerPropertyInBatch() {
    List<String> paths = List.of(
        "/content/properties/chennai/villa-101",
        "/content/properties/chennai/villa-102",
        "/content/properties/bengaluru/apt-201");

    indexingService.queueBatchIndexingJobs(jobManager, paths);

    verify(jobManager, times(3)).addJob(eq("realestate/indexing"), jobPropertiesCaptor.capture());

    List<Map<String, Object>> allCapturedProps = jobPropertiesCaptor.getAllValues();
    assertEquals(3, allCapturedProps.size());
    assertEquals("/content/properties/bengaluru/apt-201", allCapturedProps.get(2).get("path"));
}
```

### 9.2.5 Practical use — capturing the exact PredicateGroup passed to QueryBuilder

This directly resolves the technique flagged in Phase 7.2.2 — asserting *what predicates the service actually built*, rather than mocking `PredicateGroup` itself.

```java
@Test
void shouldBuildCorrectPredicateGroupForLocalitySearch() {
    when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
    when(query.getResult()).thenReturn(searchResult);
    when(searchResult.getHits()).thenReturn(Collections.emptyList());

    ArgumentCaptor<PredicateGroup> predicateCaptor = ArgumentCaptor.forClass(PredicateGroup.class);

    searchService.searchByLocality(queryBuilder, resourceResolver, session, "Chennai");

    verify(queryBuilder).createQuery(predicateCaptor.capture(), eq(session));

    PredicateGroup captured = predicateCaptor.getValue();
    assertEquals("Chennai", captured.get(0).get("property.value"));
    assertEquals("/content/properties", captured.get(0).get("path"));
}
```

---

## 9.3 Static Mocking (MockedStatic)

### 9.3.1 Concept
Static methods can't normally be mocked — Mockito's default proxy-based mocking only works on instance methods reachable through an interface/class reference. `Mockito.mockStatic(Class)` (available since Mockito 3.4+ with the inline mock maker) intercepts calls to a class's static methods **for the duration of a scoped block**, which is essential for testing code that calls utility classes (`ResourceUtil`, `PageUtil`) or time-dependent statics (`System.currentTimeMillis()`, `ZonedDateTime.now()`) that you cannot otherwise control in a test.

### 9.3.2 Technicalities to know
- **Requires `mockito-inline` (or Mockito 5+'s default inline mock maker)** — the classic `mockito-core` mock maker (subclass-based proxies) cannot intercept static or final method calls at all. If `mockStatic()` throws `MockitoException: an inline mockmaker is required`, your project's Mockito dependency needs to point at `mockito-inline`, or (Mockito 5+) confirm `org.mockito.plugins.MockMaker` isn't pinned to the old subclass maker via a `mockito-extensions/org.mockito.plugins.MockMaker` resource file.
- **`try-with-resources` is mandatory, not just good practice.** `MockedStatic` registers a global interception for that class **on the current thread** — if you don't close it (via try-with-resources or an explicit `.close()` in `@AfterEach`), the static mock leaks into subsequent tests, causing confusing, order-dependent failures where an unrelated test suddenly gets stubbed static behavior it never asked for. This is one of the most common sources of "test passes alone, fails in the full suite" bug reports.
- Only the specific overload/arguments you stub via `mocked.when(() -> MyUtil.someMethod(args))` are intercepted — calls to the same static class with **different** arguments (or entirely different methods on the class) fall through to their **real** implementation by default, unless you also stub them or call `mocked.when(...).thenCallRealMethod()` explicitly for a catch-all.
- Common AEM use case: `PageUtil`/`ResourceUtil`/custom `DateUtil` classes with static helper methods, and freezing "now" (`ZonedDateTime.now()`, `System.currentTimeMillis()`) so date-dependent business logic (e.g., "listing expires 30 days after posting") is deterministic in tests rather than depending on the wall-clock date the test happens to run on.

### 9.3.3 try (MockedStatic<MyUtil> mocked = Mockito.mockStatic(MyUtil.class))

```java
class PropertyListingExpiryServiceTest {

    @Test
    void shouldMarkListingExpiredWhenThirtyDaysHavePassed() {
        ZonedDateTime fixedNow = ZonedDateTime.of(2026, 7, 8, 10, 0, 0, 0, ZoneId.of("Asia/Kolkata"));

        try (MockedStatic<ZonedDateTime> mockedTime = Mockito.mockStatic(ZonedDateTime.class)) {
            mockedTime.when(ZonedDateTime::now).thenReturn(fixedNow);

            PropertyListingExpiryService expiryService = new PropertyListingExpiryService();
            ZonedDateTime postedDate = ZonedDateTime.of(2026, 6, 1, 10, 0, 0, 0, ZoneId.of("Asia/Kolkata"));

            boolean isExpired = expiryService.isExpired(postedDate);

            assertTrue(isExpired); // 37 days elapsed against fixed "now"
        }
        // MockedStatic automatically closed here — real ZonedDateTime.now() behavior restored
    }
}
```

### 9.3.4 Stubbing mocked.when(() -> MyUtil.someMethod(args)).thenReturn(value)

```java
@Test
void shouldUseUtilClassToFormatPropertyPathForDisplay() {
    try (MockedStatic<ResourceUtil> mockedUtil = Mockito.mockStatic(ResourceUtil.class)) {
        Resource propertyResource = mock(Resource.class);
        mockedUtil.when(() -> ResourceUtil.getName(propertyResource)).thenReturn("villa-101");

        String displayPath = new PropertyPathFormatter().formatForBreadcrumb(propertyResource);

        assertEquals("Villa 101", displayPath); // formatter title-cases the mocked static result
        mockedUtil.verify(() -> ResourceUtil.getName(propertyResource));
    }
}
```

### 9.3.5 Closing scope — why try-with-resources is mandatory (failure demo)

```java
// ANTI-PATTERN — do NOT do this:
@Test
void leaksStaticMockIntoOtherTests() {
    MockedStatic<ZonedDateTime> mockedTime = Mockito.mockStatic(ZonedDateTime.class);
    mockedTime.when(ZonedDateTime::now).thenReturn(ZonedDateTime.of(2026, 1, 1, 0, 0, 0, 0, ZoneId.of("UTC")));
    // ... assertions ...
    // Forgot mockedTime.close() — every subsequent test in the JVM that calls
    // ZonedDateTime.now() on this thread now gets Jan 1 2026, not the real time,
    // until some other test happens to open (and close) its own MockedStatic<ZonedDateTime>.
}

// CORRECT — always try-with-resources, or explicit close() in a finally/@AfterEach:
@Test
void closesStaticMockDeterministically() {
    try (MockedStatic<ZonedDateTime> mockedTime = Mockito.mockStatic(ZonedDateTime.class)) {
        mockedTime.when(ZonedDateTime::now).thenReturn(ZonedDateTime.of(2026, 1, 1, 0, 0, 0, 0, ZoneId.of("UTC")));
        // ... assertions ...
    } // guaranteed close, even if an assertion throws
}
```

---

## 9.4 Reflection (ReflectionTestUtils / Field Injection)

### 9.4.1 Concept
Some OSGi components expose configuration only through `@Reference`/`@Activate`-injected private fields with no public setter (by design — OSGi wires them, not application code). In a pure-Mockito unit test with no OSGi container running, you need another way to populate those fields. Reflection-based field injection lets you set a private field directly, bypassing the normal constructor/setter API.

### 9.4.2 Technicalities to know
- `ReflectionTestUtils.setField(target, "fieldName", value)` (Spring Test — commonly pulled in transitively even in non-Spring AEM projects, or added directly) sets a private/protected field by name via reflection, without requiring a setter. It's a common companion to `@InjectMocks` when a field's type doesn't match cleanly for Mockito's automatic injection (e.g., primitive config fields from `@ObjectClassDefinition`-based OSGi configs), or when directly testing an OSGi component's `@Activate` method.
- `FieldUtils.writeField(target, "fieldName", value, true)` (Apache Commons Lang3) is a very similar alternative — the trailing `true` means "force accessibility" (bypass Java's normal private-field access check), directly analogous to calling `field.setAccessible(true)` yourself. Functionally near-identical to `ReflectionTestUtils`; the choice is usually just "whichever library your project already depends on."
- **When to use reflection vs refactoring for testability:** reflection field injection is appropriate for OSGi-managed fields that genuinely have no other injection path in a unit test (config values from `@ObjectClassDefinition`, `@Reference`-injected services on a class you don't want to fully activate through `AemContext`). It is **not** a good substitute for constructor injection on your *own* application classes — if you find yourself reaching for reflection just to set a field that could have been a constructor parameter, that's a signal to refactor the class for constructor injection instead, which is more explicit, more IDE-navigable, and doesn't silently break if the field is renamed.
- **Accessing private methods** via `method.setAccessible(true)` + `method.invoke(target, args)` should be treated as a stronger design smell than private-field injection — testing a private method directly usually means the test is bypassing the class's actual public contract. Prefer testing the private method's behavior indirectly through the public method that calls it; reach for direct private-method reflection only for genuinely complex internal algorithms where the public-method test would be too indirect to meaningfully pinpoint failures (e.g., a complex pure calculation helper), and consider extracting it to a separate, directly-testable class instead.

### 9.4.3 ReflectionTestUtils.setField — injecting private OSGi config fields

```java
class PropertyPricingServiceActivateTest {

    @Test
    void shouldApplyConfiguredDiscountCapFromOsgiConfig() {
        PropertyPricingServiceImpl service = new PropertyPricingServiceImpl();

        // maxDiscountPercent is a private field populated by @Activate in real OSGi runtime;
        // here we set it directly since no container is running
        ReflectionTestUtils.setField(service, "maxDiscountPercent", 15);

        long finalPrice = service.applyDiscount(15000000L, 25); // requested 25%, capped at configured 15%

        assertEquals(12750000L, finalPrice);
    }
}
```

### 9.4.4 Alternative: FieldUtils.writeField from Apache Commons

```java
@Test
void shouldInjectFieldUsingApacheCommonsFieldUtils() throws IllegalAccessException {
    PropertyPricingServiceImpl service = new PropertyPricingServiceImpl();

    FieldUtils.writeField(service, "maxDiscountPercent", 20, true);

    long finalPrice = service.applyDiscount(15000000L, 20);

    assertEquals(12000000L, finalPrice);
}
```

### 9.4.5 Accessing private methods — and when this is a design smell

```java
@Test
void shouldTestPrivatePriceRoundingMethodDirectlyAsLastResort() throws Exception {
    PropertyPricingServiceImpl service = new PropertyPricingServiceImpl();

    Method roundToNearestThousand = PropertyPricingServiceImpl.class
        .getDeclaredMethod("roundToNearestThousand", long.class);
    roundToNearestThousand.setAccessible(true);

    long result = (long) roundToNearestThousand.invoke(service, 14999499L);

    assertEquals(14999000L, result);

    // Design-smell flag: if this method's logic is complex enough to need direct testing,
    // consider extracting it into a standalone PriceRoundingUtil class with a public
    // method instead — directly testable without reflection, and independently reusable.
}
```

---

## 9.5 @InjectMocks vs ctx.registerService()

### 9.5.1 Concept
Both approaches assemble a class under test with its dependencies wired in — but they operate at different levels. `@InjectMocks` is pure Mockito: it uses reflection to stuff `@Mock`-annotated fields into matching fields/constructor parameters of the target object, with no Sling/OSGi machinery involved at all. `ctx.registerService(...)` operates through a real (mocked) OSGi/Sling runtime provided by `AemContext`, which is what Sling Models rely on for `@OSGiService`/`@Self`/`@ValueMapValue` injection to actually resolve.

### 9.5.2 Technicalities to know
- **`@InjectMocks` — pure Mockito, no AemContext, fast, no Sling runtime.** Works well for plain OSGi service classes (`@Component`-annotated classes with `@Reference` fields) tested as plain Java objects — Mockito's injection doesn't care about the `@Reference`/`@Component` annotations at all, it just does constructor-or-field matching by type. This is the fastest test style available and should be your default whenever the class under test doesn't specifically need Sling request/resource context.
- **`ctx.registerService()` — uses AemContext, supports `@OSGiService` injection in Sling Models.** Required when testing a Sling Model (`@Model(adaptables = {Resource.class, SlingHttpServletRequest.class})`) that declares `@OSGiService private SomeService someService;` — Sling Models' injection annotations are processed by the real (test-scoped) Sling Models injector framework running inside `AemContext`, which `@InjectMocks` has no awareness of. You register the mock/spy as an OSGi service in the mock runtime (`context.registerService(SomeService.class, mockService)`), then `context.registerInjectActivateService(...)` or `context.request().adaptTo(MyModel.class)` triggers real injection resolution.
- **Rule of thumb, matching your team's existing convention:** `@InjectMocks` for OSGi services tested as plain POJOs (workflow processes, standalone service implementations called directly); `ctx.registerService()` + `AemContext` for anything that is, or depends on, a Sling Model, since Sling Model field injection genuinely cannot be exercised by `@InjectMocks` alone — the annotations are inert without the Sling Models injector processing them.
- A frequent mistake: trying to `@InjectMocks` a Sling Model class directly. It will compile and "work" for simple cases where the model has a no-arg constructor and public setters, but any `@ValueMapValue`/`@ChildResource`/`@OSGiService`-annotated field is left `null`, because `@InjectMocks` has no idea those annotations carry injection meaning — only the Sling Models framework (inside `AemContext`) processes them.

### 9.5.3 @InjectMocks example (recap — OSGi service as POJO)

```java
@ExtendWith(MockitoExtension.class)
class PropertyApprovalServicePlainPojoTest {

    @Mock private Replicator replicator;

    @InjectMocks
    private PropertyApprovalServiceImpl approvalService; // plain @Component class, no Sling Model

    @Test
    void shouldWorkWithPureMockitoNoAemContext() throws Exception {
        approvalService.approve("/content/properties/chennai/villa-101");

        verify(replicator).replicate(any(), eq(ReplicationActionType.ACTIVATE), anyString());
    }
}
```

### 9.5.4 ctx.registerService() example (required for Sling Models)

```java
@ExtendWith(AemContextExtension.class)
class PropertyDetailModelTest {

    private final AemContext context = new AemContext();

    @Test
    void shouldInjectOsgiServiceIntoSlingModel() {
        context.addModelsForClasses(PropertyDetailModel.class);

        PropertyPricingService pricingServiceMock = mock(PropertyPricingService.class);
        when(pricingServiceMock.applyDiscount(anyLong(), anyInt())).thenReturn(13500000L);

        // Registering through AemContext makes the mock resolvable by the Sling Models
        // injector for the @OSGiService-annotated field on PropertyDetailModel
        context.registerService(PropertyPricingService.class, pricingServiceMock);

        context.create().resource("/content/properties/villa-101",
            "jcr:title", "Villa 101", "price", 15000000L);
        context.currentResource("/content/properties/villa-101");

        PropertyDetailModel model = context.request().adaptTo(PropertyDetailModel.class);

        assertEquals(13500000L, model.getDiscountedPrice());
    }
}
```

---

## 9.6 Parameterized Tests

### 9.6.1 Concept
JUnit 5's `@ParameterizedTest` runs the same test method body against multiple sets of inputs, avoiding copy-pasted near-identical test methods. This is especially valuable for validation/formatting logic with many input variants — exactly the shape of "does every property type format its price correctly" or "does every combination of status + price behave right."

### 9.6.2 Technicalities to know
- `@ValueSource` supports a single primitive/String array of inputs per test run — simplest form, use when the test only needs one varying parameter.
- `@CsvSource` supports multiple parameters per row as comma-separated values in a string array — each string is one full row/invocation; useful when 2–4 simple parameters vary together. Watch for accidental extra whitespace in CSV values, which is *not* trimmed by default unless you set `@CsvSource(..., ignoreLeadingAndTrailingWhitespace = true)` (default is actually trimmed in modern JUnit 5, but verify your version).
- `@MethodSource("factoryMethodName")` supports arbitrary complex objects as parameters (DTOs, `Arguments.of(...)` tuples) — required whenever a parameter isn't a simple primitive/String, e.g., passing a whole `SearchRequest` object per test case. The factory method must be `static` and return a `Stream<Arguments>` (or `Stream<X>` for a single-parameter case).
- Parameterized tests still benefit from a clear, per-case display name — `@ParameterizedTest(name = "{0} -> {1}")` makes failures immediately legible (which specific input row failed) rather than a generic "test #3 failed."

### 9.6.3 @ValueSource — property type formatting

```java
@ParameterizedTest(name = "propertyType={0} should have a valid display label")
@ValueSource(strings = {"villa", "apartment", "house", "plot", "penthouse"})
void shouldFormatDisplayLabelForEveryPropertyType(String propertyType) {
    String label = PropertyTypeFormatter.getDisplayLabel(propertyType);

    assertNotNull(label);
    assertFalse(label.isBlank());
}
```

### 9.6.4 @CsvSource — multiple parameters per row

```java
@ParameterizedTest(name = "{0}, status={1} -> formattedPrice={2}")
@CsvSource({
    "villa, available, 20,00,000",
    "apartment, sold, 50,00,000",
    "plot, available, 12,00,000",
    "penthouse, under-negotiation, 2,50,00,000"
})
void shouldFormatPriceCorrectlyForPropertyTypeAndStatus(
        String propertyType, String status, String expectedFormattedPrice) {

    String formatted = PropertyPriceFormatter.formatIndianCurrency(propertyType, status);

    assertEquals(expectedFormattedPrice, formatted);
}
```

### 9.6.5 @MethodSource — complex object parameters from a static factory method

```java
@ParameterizedTest(name = "{index} => {0}")
@MethodSource("provideSearchRequests")
void shouldBuildCorrectPredicateGroupForEachSearchRequest(SearchRequest request, int expectedPredicateCount) {
    PredicateGroup predicateGroup = SearchPredicateBuilder.build(request);

    assertEquals(expectedPredicateCount, predicateGroup.size());
}

private static Stream<Arguments> provideSearchRequests() {
    return Stream.of(
        Arguments.of(new SearchRequest("Chennai", null, null), 1),
        Arguments.of(new SearchRequest("Chennai", 3, null), 2),
        Arguments.of(new SearchRequest("Chennai", 3, "villa"), 3),
        Arguments.of(new SearchRequest(null, null, null), 0)
    );
}
```

---

## 9.7 Testing Exception and Edge Case Paths

### 9.7.1 Concept
The tests most likely to catch real production incidents are rarely the happy-path ones — they're the ones covering what happens when a collaborator fails, an expected parameter is missing, or a caller passes an edge-case-but-legal input (empty string, blank, zero, negative). This closing section consolidates the exception/edge-case patterns used throughout Phases 6–9 into a single reference.

### 9.7.2 Technicalities to know
- `assertThrows(ExceptionType.class, () -> ...)` (JUnit 5) returns the thrown exception instance, letting you also assert on its message/cause — don't just assert the type; assert the message content when the message carries meaningful diagnostic information callers depend on (e.g., logging pipelines that parse exception messages).
- `when(mock.method()).thenThrow(new RuntimeException())` simulates a collaborator failure — pair every "happy path calls collaborator successfully" test with at least one "collaborator throws" test, since production failure modes (network partition, downstream service outage) are exactly this shape.
- **Null vs empty string is not the same edge case — test both separately.** `request.getParameter("q")` returning `null` (parameter absent entirely) and returning `""` (parameter present but empty, e.g., a submitted-but-cleared search box) are both legal, common, and require potentially different handling — code that only guards `!= null` will NPE-free on missing params but may still misbehave (or produce a nonsensical "return everything" query) on an empty string, and vice versa for code that only checks `.isEmpty()` without a null-guard first (a direct NPE).

### 9.7.3 assertThrows — JUnit 5 style

```java
@Test
void shouldThrowWorkflowExceptionWithDiagnosticMessage() {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("");

    WorkflowException ex = assertThrows(WorkflowException.class,
        () -> processStep.execute(workItem, workflowSession, args));

    assertTrue(ex.getMessage().contains("payload"));
}
```

### 9.7.4 Simulating collaborator failure

```java
@Test
void shouldPropagateFailureWhenPricingServiceThrows() {
    when(pricingService.calculateFinalPrice(anyLong(), anyString()))
        .thenThrow(new RuntimeException("pricing service unavailable"));

    assertThrows(RuntimeException.class,
        () -> listingService.getPropertyDetails(propertyResource, pricingService));
}
```

### 9.7.5 Testing null input vs empty string separately

```java
@Test
void shouldReturnAllPropertiesWhenSearchQueryParamIsNull() {
    when(request.getParameter("q")).thenReturn(null);

    List<String> results = searchService.searchFromRequest(request);

    assertFalse(results.isEmpty()); // treats missing param as "no filter"
}

@Test
void shouldReturnEmptyResultsWhenSearchQueryParamIsEmptyString() {
    when(request.getParameter("q")).thenReturn("");

    List<String> results = searchService.searchFromRequest(request);

    // Deliberately different expectation than the null case — an explicitly empty
    // submitted search is treated as "user searched for nothing," not "no filter"
    assertTrue(results.isEmpty());
}

@Test
void shouldTrimWhitespaceOnlyQueryToEmptyBehavior() {
    when(request.getParameter("q")).thenReturn("   ");

    List<String> results = searchService.searchFromRequest(request);

    assertTrue(results.isEmpty());
}
```

---

## 9.8 Phase 9 Summary Table

| Technique | Use when | Key gotcha |
|---|---|---|
| `spy()` | You want mostly-real behavior with one method stubbed | Use `doReturn()`, never `when()`, on a spy — `when()` runs the real method first |
| `ArgumentCaptor` | You need to inspect the actual value passed to a mock, not just match it | `getValue()` returns only the last call; use `getAllValues()` for multi-call scenarios |
| `MockedStatic` | Testing code that calls static utility methods or time-based statics | Requires `mockito-inline`; always try-with-resources or the mock leaks across tests |
| `ReflectionTestUtils` / `FieldUtils` | Setting OSGi-managed private fields with no setter, outside a container | Reach for constructor injection on your own classes instead where possible |
| `@InjectMocks` | Plain OSGi service classes tested as POJOs | Does not process Sling Model injection annotations — silently leaves fields null |
| `ctx.registerService()` | Sling Models with `@OSGiService`/`@ValueMapValue` etc. | Required — `@InjectMocks` cannot substitute for the Sling Models injector |
| `@ValueSource` / `@CsvSource` / `@MethodSource` | Running one test body across many inputs | Use `@MethodSource` once parameters stop being simple primitives/Strings |
| `assertThrows` / `thenThrow` | Locking in failure-mode behavior | Test null AND empty-string separately — they are different edge cases with potentially different expected behavior |

---

**Next: Phase 10 — Cheat Sheet + JaCoCo** (a consolidated one-page reference of every mocking pattern from Phases 6–9, plus JaCoCo setup for coverage reporting, reading coverage reports meaningfully, common coverage-gaming anti-patterns to avoid, and coverage-threshold enforcement in a Maven build).

Ready for Phase 10 whenever you are.
