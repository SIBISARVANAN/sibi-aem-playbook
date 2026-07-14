# AEM JUnit Testing Guide — Phase 7: QueryBuilder Mocking

**Scenario used throughout:** `PropertyListingService` — searching, filtering, and faceting a real-estate property catalog (locality, price range, bedroom count, amenity tags).

---

## 7.0 Concept — What QueryBuilder Is and Why It's Different

`QueryBuilder` (`com.day.cq.search`) is AEM's higher-level, predicate-based search API sitting on top of the JCR/Oak query engine. Instead of writing raw XPath/SQL2, you build a `PredicateGroup` (a map of predicate types like `path`, `fulltext`, `tagid`, `property`, `daterange`) and hand it to `QueryBuilder.createQuery(...)`, which compiles it into an actual Oak query under the hood, executes it, and wraps the result in a `SearchResult` with pagination, facets, and `Hit` objects.

This matters for testing because QueryBuilder is a **thin API wrapping a real query engine** — unlike `ResourceResolver`/`PageManager`/`TagManager` (Phase 6), which are mostly structural tree-walkers you can fake with an in-memory content tree, QueryBuilder's actual value (correct predicate matching, index usage, result ranking) can only be verified against a real, indexed repository. This is why Phase 7 introduces `JCR_OAK` as a first-class requirement, not just an option.

---

## 7.1 Real QueryBuilder with JCR_OAK

### 7.1.1 Concept
`JCR_OAK` boots an embedded, real Apache Oak repository inside your test JVM — the same query engine (minus clustering/cold-start tuning) that runs in production AEM. Unlike `JCR_MOCK` (Sling's lightweight fake, used in Phase 6), Oak actually parses predicates, applies indexes (or falls back to traversal), and returns genuinely correct result sets — including edge cases like predicate combination logic (`AND` vs `OR` groups), range queries, and full-text tokenization that a hand-mocked chain simply cannot reproduce faithfully.

### 7.1.2 Technicalities to know
- **Why `JCR_MOCK` can't be used for QueryBuilder tests:** `JCR_MOCK` implements enough of the `javax.jcr` surface to satisfy `Session`/`Node` CRUD operations, but it has **no query engine** underneath — `QueryBuilder.createQuery(...).getResult()` against a `JCR_MOCK`-backed `ResourceResolver` either throws `UnsupportedOperationException` or silently returns zero hits regardless of matching content. This is the single most common cause of "my query test always returns empty" bug reports from teams new to AEM mocks.
- Oak's default in-memory setup (via AemContext) does **not** automatically create custom indexes matching your production `oak:index` definitions. Without an index, Oak falls back to a full traversal — functionally correct but the query works differently than production. If your test asserts something index-dependent (e.g., ordering by `@price`, or a `tagid` predicate that relies on the `damAssetLucene`/`cqPageLucene`-style index), you must explicitly register a matching test index — otherwise the query may still "work" in the test but silently rely on traversal-order behavior that differs from your indexed production query.
- `AemContext` bootstraps Oak fresh per test method by default — this gives strong test isolation (no cross-test content leakage) but also means Oak's startup cost (index setup, node type registration) is paid per test. Group related query assertions into fewer, larger test methods against a shared `@BeforeEach`-loaded dataset where practical, rather than one property-per-test, to keep suite runtime reasonable.
- Real Oak enforces JCR node type / property constraints that `JCR_MOCK` and hand-rolled Mockito stubs never check — this is actually a feature: a query test that only passes against `JCR_OAK` (not `JCR_MOCK`) is validating something a pure-mock test structurally cannot.

### 7.1.3 Setting up AemContext(ResourceResolverType.JCR_OAK)

```java
@ExtendWith(AemContextExtension.class)
class PropertySearchServiceOakTest {

    private final AemContext context = new AemContext(ResourceResolverType.JCR_OAK);

    @BeforeEach
    void setUp() {
        // Register the real QueryBuilder implementation backed by this Oak repository
        context.registerInjectActivateService(new QueryBuilderImpl()); // wm:io.wcm mock-aem provides a working QueryBuilder against Oak automatically in most setups

        context.create().resource("/content/properties/chennai/villa-101",
            "jcr:primaryType", "nt:unstructured",
            "sling:resourceType", "realestate/components/property",
            "jcr:title", "Villa 101 - Sea View",
            "price", 15000000L,
            "bedrooms", 4,
            "locality", "Chennai");

        context.create().resource("/content/properties/chennai/villa-102",
            "jcr:primaryType", "nt:unstructured",
            "sling:resourceType", "realestate/components/property",
            "jcr:title", "Villa 102 - Garden View",
            "price", 9500000L,
            "bedrooms", 3,
            "locality", "Chennai");

        context.create().resource("/content/properties/bengaluru/apt-201",
            "jcr:primaryType", "nt:unstructured",
            "sling:resourceType", "realestate/components/property",
            "jcr:title", "Apartment 201",
            "price", 7200000L,
            "bedrooms", 2,
            "locality", "Bengaluru");
    }
}
```

**Note:** `io.wcm.testing.mock.aem` auto-registers a working `QueryBuilder` service against the Oak repository when you use `JCR_OAK` — you typically don't need to hand-write a `QueryBuilderImpl`. The explicit `registerInjectActivateService` line above is shown for clarity of what's happening under the hood; in practice check whether your AemContext version already exposes `context.getService(QueryBuilder.class)` without extra setup.

### 7.1.4 Registering the QueryBuilder service and running a real query

```java
@Test
void shouldFindAllChennaiPropertiesUnderThreeBedrooms() {
    // Arrange
    Session session = context.resourceResolver().adaptTo(Session.class);
    Map<String, String> predicateMap = new HashMap<>();
    predicateMap.put("path", "/content/properties/chennai");
    predicateMap.put("property", "locality");
    predicateMap.put("property.value", "Chennai");
    predicateMap.put("property2", "bedrooms");
    predicateMap.put("property2.operation", "less");
    predicateMap.put("property2.value", "4");

    QueryBuilder queryBuilder = context.getService(QueryBuilder.class);
    Query query = queryBuilder.createQuery(PredicateGroup.create(predicateMap), session);

    // Act
    SearchResult result = query.getResult();
    List<String> matchedPaths = result.getHits().stream()
        .map(this::hitPathOrThrow)
        .collect(Collectors.toList());

    // Assert
    assertEquals(1, matchedPaths.size());
    assertTrue(matchedPaths.contains("/content/properties/chennai/villa-102"));
}

private String hitPathOrThrow(Hit hit) {
    try {
        return hit.getPath();
    } catch (RepositoryException e) {
        throw new RuntimeException(e);
    }
}
```

### 7.1.5 Asserting correct paths are returned — full-text example

```java
@Test
void shouldFindPropertiesByFullTextTitleSearch() {
    Session session = context.resourceResolver().adaptTo(Session.class);
    Map<String, String> predicateMap = new HashMap<>();
    predicateMap.put("path", "/content/properties");
    predicateMap.put("fulltext", "Sea View");

    QueryBuilder queryBuilder = context.getService(QueryBuilder.class);
    Query query = queryBuilder.createQuery(PredicateGroup.create(predicateMap), session);

    SearchResult result = query.getResult();

    assertEquals(1, result.getHits().size());
}
```

**Edge case worth testing:** an empty predicate map (or a `path` predicate pointing to a non-existent root) should return zero hits, not throw — verify your service's error handling explicitly rather than assuming QueryBuilder degrades gracefully in every case; some malformed predicate combinations *do* throw `RepositoryException` at `getResult()` time.

```java
@Test
void shouldReturnEmptyResultForNonExistentPathRoot() {
    Session session = context.resourceResolver().adaptTo(Session.class);
    Map<String, String> predicateMap = new HashMap<>();
    predicateMap.put("path", "/content/properties/nonexistent-city");

    QueryBuilder queryBuilder = context.getService(QueryBuilder.class);
    Query query = queryBuilder.createQuery(PredicateGroup.create(predicateMap), session);

    assertEquals(0, query.getResult().getHits().size());
}
```

---

## 7.2 Mocked QueryBuilder and SearchResult

### 7.2.1 Concept
Not every test needs a real query engine. If the class under test's logic is really "take a `SearchResult`, transform it into a DTO / apply pagination metadata / handle empty results" — the *querying* itself is not what you're testing, the **transformation logic** is. For these cases, a fully mocked `QueryBuilder → Query → SearchResult → Hit` chain is faster to write, faster to run (no Oak bootstrap), and keeps the test focused on the actual unit of behavior.

### 7.2.2 Technicalities to know
- **When to prefer mocking over `JCR_OAK`:** use mocks when testing predicate-*construction* logic (did the service build the right `PredicateGroup` for a given search form input?) or result-*transformation* logic (did the service correctly map `Hit`s to DTOs, compute pagination flags, etc.). Use real `JCR_OAK` (7.1) when testing whether predicates actually match the *right content* — that's an integration concern, not a pure-unit one.
- The chain is deep: `queryBuilder.createQuery(...)` → `Query` → `query.getResult()` → `SearchResult` → `result.getHits()` → `List<Hit>` → `hit.getPath()` / `hit.getResource()`. Each mock in this chain needs to be stubbed independently; missing one link returns `null` and produces a confusing `NullPointerException` several layers away from the actual missing stub — a frequent source of wasted debugging time. Build the chain top-down and run the test after adding each stub if you're unsure where it breaks.
- `Hit.getPath()` and `Hit.getResource()` both declare `throws RepositoryException` — same as JCR `Node`/`Property` methods in Phase 6 — so tests need `throws RepositoryException` on the method or must wrap calls.
- `Query` also exposes `setStart(long)` / `setHitsPerPage(long)` for pagination *input* — these are setters on the mock `Query`, so tests that verify pagination request behavior should `verify(query).setStart(20)` etc., not just stub the result side.
- Prefer `PredicateGroup.create(map)` (real object, not mocked) when testing "did the service build the correct predicate map" — capture the map passed into `queryBuilder.createQuery(...)` with an `ArgumentCaptor<PredicateGroup>` (covered in depth in Phase 9) rather than trying to mock `PredicateGroup` itself, since it's a simple value object and mocking it obscures what you're actually asserting.

### 7.2.3 Stubbing the full chain

```java
@ExtendWith(MockitoExtension.class)
class PropertySearchServiceMockedQueryTest {

    @Mock private QueryBuilder queryBuilder;
    @Mock private Query query;
    @Mock private SearchResult searchResult;
    @Mock private Hit villaHit;
    @Mock private Hit apartmentHit;
    @Mock private Resource villaResource;
    @Mock private Resource apartmentResource;
    @Mock private ResourceResolver resourceResolver;
    @Mock private Session session;

    @InjectMocks
    private PropertySearchService searchService;

    @Test
    void shouldReturnMatchingPropertyPaths() throws RepositoryException {
        // Arrange — build the chain top-down
        when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
        when(query.getResult()).thenReturn(searchResult);
        when(searchResult.getHits()).thenReturn(List.of(villaHit, apartmentHit));

        when(villaHit.getPath()).thenReturn("/content/properties/chennai/villa-101");
        when(villaHit.getResource()).thenReturn(villaResource);
        when(apartmentHit.getPath()).thenReturn("/content/properties/bengaluru/apt-201");
        when(apartmentHit.getResource()).thenReturn(apartmentResource);

        // Act
        List<String> paths = searchService.searchByLocality(queryBuilder, resourceResolver, session, "Chennai");

        // Assert
        assertEquals(2, paths.size());
        assertTrue(paths.contains("/content/properties/chennai/villa-101"));
        verify(queryBuilder).createQuery(any(PredicateGroup.class), eq(session));
    }

    @Test
    void shouldReturnEmptyListWhenNoHits() {
        when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
        when(query.getResult()).thenReturn(searchResult);
        when(searchResult.getHits()).thenReturn(Collections.emptyList());

        List<String> paths = searchService.searchByLocality(queryBuilder, resourceResolver, session, "Chennai");

        assertTrue(paths.isEmpty());
    }
}
```

### 7.2.4 Creating mock Hit objects with stubbed getPath() / getResource()

```java
@Test
void shouldMapHitsToPropertySummaryDtos() throws RepositoryException {
    ValueMap villaValueMap = new ValueMapDecorator(Map.of(
        "jcr:title", "Villa 101 - Sea View", "price", 15000000L));

    when(villaHit.getPath()).thenReturn("/content/properties/chennai/villa-101");
    when(villaHit.getResource()).thenReturn(villaResource);
    when(villaResource.getValueMap()).thenReturn(villaValueMap);

    when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
    when(query.getResult()).thenReturn(searchResult);
    when(searchResult.getHits()).thenReturn(List.of(villaHit));

    List<PropertySummary> summaries = searchService.searchAsSummaries(
        queryBuilder, resourceResolver, session, "Chennai");

    assertEquals(1, summaries.size());
    assertEquals("Villa 101 - Sea View", summaries.get(0).getTitle());
    assertEquals(15000000L, summaries.get(0).getPrice());
}
```

### 7.2.5 Testing pagination — getTotalMatches() and hasMore()

```java
@Test
void shouldExposeCorrectPaginationMetadata() {
    when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
    when(query.getResult()).thenReturn(searchResult);
    when(searchResult.getHits()).thenReturn(List.of(villaHit, apartmentHit)); // page of 2
    when(searchResult.getTotalMatches()).thenReturn(47L);
    when(searchResult.hasMore()).thenReturn(true);

    PropertySearchPage page = searchService.searchPage(
        queryBuilder, resourceResolver, session, "Chennai", 0, 2);

    assertEquals(47L, page.getTotalMatches());
    assertTrue(page.hasMoreResults());
    assertEquals(2, page.getResults().size());
}

@Test
void shouldSetPaginationParametersOnQuery() {
    when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
    when(query.getResult()).thenReturn(searchResult);
    when(searchResult.getHits()).thenReturn(Collections.emptyList());

    searchService.searchPage(queryBuilder, resourceResolver, session, "Chennai", 20, 10);

    // Verify the service requested the correct page window on the Query itself
    verify(query).setStart(20);
    verify(query).setHitsPerPage(10);
}
```

**Edge case worth testing:** `getTotalMatches()` can be more expensive to compute than `getHits()` in real Oak (it may trigger a full count rather than a limited fetch) — if your service exposes a "don't compute total count" fast path (common for infinite-scroll UIs), make sure there's a test asserting `getTotalMatches()` is *never* called in that mode: `verify(searchResult, never()).getTotalMatches();`.

---

## 7.3 Facet Result Mocking

### 7.3.1 Concept
Facets let a search UI show "filter by" counts — e.g., "Chennai (12), Bengaluru (8)" for locality, or "2 BHK (5), 3 BHK (9), 4 BHK (3)" for bedroom count — without running a separate query per filter option. QueryBuilder computes these via the `facets` predicate; `SearchResult.getFacets()` returns a `Map<String, Facet>` keyed by property name, and each `Facet` holds a list of `Bucket`s (one bucket per distinct value, with a count).

### 7.3.2 Technicalities to know
- Facets require an index that supports faceting (typically a Lucene property index with `facets: true` configured) — in real `JCR_OAK` tests, an unindexed facet predicate silently returns an empty facet map rather than throwing, which can mask a missing-index configuration bug. This is precisely why facet *logic* (the DTO transformation) is usually tested with mocks (7.3.3+) while facet *correctness against a real index* is a separate, narrower `JCR_OAK` integration test.
- `Facet.getBuckets()` returns buckets in an implementation-defined order (typically count-descending) — don't assert exact ordering unless your service explicitly re-sorts; assert set membership / counts instead, or explicitly sort before asserting if your DTO contract guarantees an order.
- `Bucket.getValue()` returns the raw stored property value as a `String` — for a `tagid`-based facet (e.g., amenity tags) this is the tag ID (`amenities:swimming-pool`), not the human-readable label; if your service is expected to show a label, it needs a `TagManager.resolve()` step (Phase 6, 6.3) chained after the facet read — a good candidate for a combined facet + tag-resolution test.
- A property with no matching documents produces no bucket at all (not a bucket with count zero) — if your UI needs to show zero-count options for discoverability, that's application logic layered on top of the facet result, and deserves its own test asserting the zero-fill behavior.

### 7.3.3 Mocking getFacets() returning Map<String, Facet>

```java
@ExtendWith(MockitoExtension.class)
class PropertyFacetServiceTest {

    @Mock private QueryBuilder queryBuilder;
    @Mock private Query query;
    @Mock private SearchResult searchResult;
    @Mock private Facet localityFacet;
    @Mock private Facet bedroomsFacet;
    @Mock private Bucket chennaiBucket;
    @Mock private Bucket bengaluruBucket;
    @Mock private Bucket threeBedroomBucket;
    @Mock private Session session;

    @InjectMocks
    private PropertyFacetService facetService;

    @Test
    void shouldTransformLocalityFacetsIntoFilterOptions() {
        // Arrange
        when(chennaiBucket.getValue()).thenReturn("Chennai");
        when(chennaiBucket.getCount()).thenReturn(12L);
        when(bengaluruBucket.getValue()).thenReturn("Bengaluru");
        when(bengaluruBucket.getCount()).thenReturn(8L);

        when(localityFacet.getBuckets()).thenReturn(List.of(chennaiBucket, bengaluruBucket));

        when(searchResult.getFacets()).thenReturn(Map.of("locality", localityFacet));
        when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
        when(query.getResult()).thenReturn(searchResult);

        // Act
        List<FacetOption> options = facetService.getLocalityFacetOptions(queryBuilder, session);

        // Assert
        assertEquals(2, options.size());
        assertTrue(options.stream().anyMatch(o -> o.getValue().equals("Chennai") && o.getCount() == 12L));
        assertTrue(options.stream().anyMatch(o -> o.getValue().equals("Bengaluru") && o.getCount() == 8L));
    }

    @Test
    void shouldReturnEmptyListWhenFacetKeyMissing() {
        when(searchResult.getFacets()).thenReturn(Collections.emptyMap());
        when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
        when(query.getResult()).thenReturn(searchResult);

        List<FacetOption> options = facetService.getLocalityFacetOptions(queryBuilder, session);

        assertTrue(options.isEmpty());
    }
}
```

### 7.3.4 Mocking multiple facets and buckets

```java
@Test
void shouldTransformMultipleFacetDimensionsIndependently() {
    when(threeBedroomBucket.getValue()).thenReturn("3");
    when(threeBedroomBucket.getCount()).thenReturn(9L);
    when(bedroomsFacet.getBuckets()).thenReturn(List.of(threeBedroomBucket));

    when(chennaiBucket.getValue()).thenReturn("Chennai");
    when(chennaiBucket.getCount()).thenReturn(12L);
    when(localityFacet.getBuckets()).thenReturn(List.of(chennaiBucket));

    when(searchResult.getFacets()).thenReturn(Map.of(
        "locality", localityFacet,
        "bedrooms", bedroomsFacet));
    when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
    when(query.getResult()).thenReturn(searchResult);

    Map<String, List<FacetOption>> allFacets = facetService.getAllFacets(queryBuilder, session);

    assertEquals(2, allFacets.size());
    assertEquals(1, allFacets.get("bedrooms").size());
    assertEquals("3", allFacets.get("bedrooms").get(0).getValue());
}
```

### 7.3.5 Verifying facet-to-DTO transformation with tag resolution (amenity facet)

```java
@Test
void shouldResolveAmenityTagIdsToLabelsInFacetOptions() {
    Bucket poolBucket = mock(Bucket.class);
    when(poolBucket.getValue()).thenReturn("amenities:swimming-pool");
    when(poolBucket.getCount()).thenReturn(5L);

    Facet amenityFacet = mock(Facet.class);
    when(amenityFacet.getBuckets()).thenReturn(List.of(poolBucket));

    when(searchResult.getFacets()).thenReturn(Map.of("amenityTags", amenityFacet));
    when(queryBuilder.createQuery(any(PredicateGroup.class), eq(session))).thenReturn(query);
    when(query.getResult()).thenReturn(searchResult);

    TagManager tagManager = mock(TagManager.class);
    Tag poolTag = mock(Tag.class);
    when(tagManager.resolve("amenities:swimming-pool")).thenReturn(poolTag);
    when(poolTag.getTitle()).thenReturn("Swimming Pool");

    List<FacetOption> options = facetService.getAmenityFacetOptions(queryBuilder, session, tagManager);

    assertEquals("Swimming Pool", options.get(0).getLabel());
    assertEquals(5L, options.get(0).getCount());
}
```

**Edge case worth testing:** what happens when `tagManager.resolve(bucket.getValue())` returns `null` (a facet bucket referencing a since-deleted tag)? The service should fall back to the raw tag ID as the label (or skip the option) rather than throwing — this directly mirrors the null-tag pitfall flagged in Phase 6.3.

---

## 7.4 Phase 7 Summary Table

| Concern | Approach | Use when | Key gotcha |
|---|---|---|---|
| Predicate matching correctness | Real `QueryBuilder` on `JCR_OAK` | Verifying a search actually returns the right content | `JCR_MOCK` has no query engine — always returns empty/throws |
| Predicate construction logic | Mocked chain + `ArgumentCaptor<PredicateGroup>` | Verifying the service builds the right predicates from form input | Mocking `PredicateGroup` itself obscures the assertion — capture the real object instead |
| Result → DTO transformation | Mocked `QueryBuilder`/`Query`/`SearchResult`/`Hit` chain | Fast, focused unit tests of mapping logic | Missing one link in the chain → NPE far from the real cause; build top-down |
| Pagination | Mocked `SearchResult` (`getTotalMatches`, `hasMore`) + `verify(query).setStart/setHitsPerPage` | Testing page-window request and response metadata together | `getTotalMatches()` can be expensive in real Oak — verify it's skipped in fast-path modes |
| Facets | Mocked `Facet`/`Bucket` maps | Testing facet-to-filter-option DTO transformation | Bucket order isn't guaranteed; bucket value is the raw tag ID, not a label |

---

**Next: Phase 8 — Workflow mocking** (`WorkflowSession`, `Workflow`, `WorkItem`, `WorkflowData`, launching and advancing workflows for the property-approval process, plus testing custom `WorkflowProcess` implementations).

Ready for Phase 8 whenever you are — or let me know if you'd like any Phase 7 section expanded further (e.g., a dedicated `ArgumentCaptor<PredicateGroup>` walkthrough now instead of waiting for Phase 9).
