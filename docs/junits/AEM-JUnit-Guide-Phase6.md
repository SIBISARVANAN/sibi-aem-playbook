# AEM JUnit Testing Guide — Phase 6: Core Sling/JCR API Mocking

**Scenario used throughout:** `PropertyListingService` — a backend service for a real-estate property listing component/API on AEM.

---

## 6.0 Setup Recap

All examples use `AemContext` (io.wcm.testing.mock.aem.junit5) with `MockitoExtension`, so most Sling objects (ResourceResolver, PageManager, TagManager) are provided by the mock context out of the box. Where a real-world team prefers pure Mockito (e.g., unit-testing a class that receives a `ResourceResolver` via constructor injection instead of via `AemContext`), a parallel "Pure Mockito" example is given.

```java
@ExtendWith(AemContextExtension.class)
class PropertyListingServiceTest {

    private final AemContext context = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        context.addModelsForClasses(PropertyListingModel.class);
        context.load().json("/testdata/property-listing.json", "/content/properties/villa-101");
    }
}
```

**Concept — `ResourceResolverType` matters.** AemContext supports several backing repository types:
- `RESOURCERESOLVER_MOCK` — pure in-memory map, fastest, but doesn't enforce real JCR node-type/constraint rules.
- `JCR_MOCK` — a lightweight fake JCR implementation (Sling's `jcr-mock`). Good middle ground; supports `Session`/`Node` APIs but not full Oak query indexing.
- `JCR_OAK` — a real embedded Apache Oak repository. Slowest to bootstrap but gives you accurate JCR/QueryBuilder/Sling query behavior — required for Phase 7 (QueryBuilder) tests.

Pick the lightest type that still exercises the behavior you're testing — this single choice is the biggest lever on your test-suite runtime.

---

## 6.1 ResourceResolver & ValueMap Mocking

### 6.1.1 Concept
`ResourceResolver` is the entry point to the entire JCR content tree in Sling — it resolves paths to `Resource` objects, runs searches, and adapts resources to other types (`ValueMap`, `Node`, Sling Models). `ValueMap` is a `Map<String, Object>`-like read view over a resource's JCR properties, with type-safe getters (`get(key, Class)` / `get(key, defaultValue)`) that handle JCR's underlying type coercion (String, Long, Boolean, Calendar, multi-value arrays) for you.

Because nearly every AEM class touches one or both of these, getting the mocking pattern right is the highest-leverage skill in this guide — nearly every other phase builds on it.

### 6.1.2 Technicalities to know
- `ValueMap` is **read-only**. To write properties you go through `ModifiableValueMap` (via `resource.adaptTo(ModifiableValueMap.class)`), which only works inside a `ResourceResolver` that hasn't been closed and typically requires an open JCR session underneath.
- `ValueMap.get(key, Class)` returns `null` if the key is absent or the stored type can't be coerced; `ValueMap.get(key, defaultValue)` never returns `null` — always prefer the default-value overload in production code to avoid NPEs.
- Multi-value JCR properties (String[], Long[], etc.) are returned as **arrays**, not `List` — a very common test-writing mistake is asserting against a `List` when the real API gives you `String[]`.
- `resource.adaptTo(X.class)` can legitimately return `null` (missing adapter factory, wrong resource type). Production code — and your tests — must handle the null-adapt case, not just the happy path.
- In pure Mockito, `ValueMapDecorator` (from `org.apache.sling.api.wrappers`) wraps a plain `Map` into a real `ValueMap` implementation — this is usually better than mocking `ValueMap` method-by-method because you get real type-coercion behavior for free.

### 6.1.3 Using AemContext (preferred)

```java
@Test
void shouldFetchPropertyTitleFromValueMap() {
    // Arrange — AemContext-backed resource already has a ValueMap
    Resource propertyResource = context.create().resource(
        "/content/properties/villa-101",
        "jcr:title", "Villa 101 - Sea View",
        "price", 15000000L,
        "bedrooms", 4,
        "isFeatured", true
    );

    PropertyListingService service = context.registerInjectActivateService(new PropertyListingServiceImpl());

    // Act
    PropertyDetails details = service.getPropertyDetails(propertyResource);

    // Assert
    assertEquals("Villa 101 - Sea View", details.getTitle());
    assertEquals(15000000L, details.getPrice());
    assertTrue(details.isFeatured());
}
```

### 6.1.4 Pure Mockito ResourceResolver + ValueMap

Useful when the class under test takes `ResourceResolverFactory` or a raw `ResourceResolver`, and you don't want the overhead of a full `AemContext`.

```java
@ExtendWith(MockitoExtension.class)
class PropertyPriceCalculatorTest {

    @Mock private ResourceResolver resourceResolver;
    @Mock private Resource propertyResource;
    @Mock private ValueMap valueMap;

    @InjectMocks
    private PropertyPriceCalculator calculator;

    @Test
    void shouldApplyDiscountBasedOnValueMap() {
        // Arrange
        when(resourceResolver.getResource("/content/properties/villa-101"))
            .thenReturn(propertyResource);
        when(propertyResource.adaptTo(ValueMap.class)).thenReturn(valueMap);
        when(valueMap.get("price", Long.class)).thenReturn(15000000L);
        when(valueMap.get("discountPercent", 0)).thenReturn(10);

        // Act
        long finalPrice = calculator.calculateFinalPrice(resourceResolver, "/content/properties/villa-101");

        // Assert
        assertEquals(13500000L, finalPrice);
        verify(resourceResolver).getResource("/content/properties/villa-101");
    }

    @Test
    void shouldReturnZeroWhenResourceMissing() {
        when(resourceResolver.getResource(anyString())).thenReturn(null);

        long finalPrice = calculator.calculateFinalPrice(resourceResolver, "/content/properties/missing");

        assertEquals(0L, finalPrice);
    }
}
```

### 6.1.5 ValueMapDecorator instead of mocking ValueMap method-by-method

```java
@Test
void shouldReadMultiValuePropertyAsArray() {
    Map<String, Object> props = new HashMap<>();
    props.put("amenityTags", new String[]{"amenities:pool", "amenities:gym"});

    ValueMap valueMap = new ValueMapDecorator(props);

    String[] tags = valueMap.get("amenityTags", String[].class);

    assertArrayEquals(new String[]{"amenities:pool", "amenities:gym"}, tags);
}
```

### 6.1.6 Mocking ResourceResolver.getResources() / traversal

```java
@Test
void shouldCountFeaturedPropertiesUnderRoot() {
    Resource root = mock(Resource.class);
    Resource p1 = mock(Resource.class);
    Resource p2 = mock(Resource.class);

    ValueMap vm1 = new ValueMapDecorator(Map.of("isFeatured", true));
    ValueMap vm2 = new ValueMapDecorator(Map.of("isFeatured", false));

    when(p1.getValueMap()).thenReturn(vm1);
    when(p2.getValueMap()).thenReturn(vm2);
    when(root.listChildren()).thenAnswer(inv -> List.of(p1, p2).iterator());

    long featuredCount = StreamSupport.stream(
            Spliterators.spliteratorUnknownSize(root.listChildren(), 0), false)
        .filter(r -> r.getValueMap().get("isFeatured", false))
        .count();

    assertEquals(1, featuredCount);
}
```

**Pitfall — Iterator exhaustion.** `listChildren()` returns an `Iterator`, not an `Iterable`. `when(...).thenReturn(iterator)` hands back the *same* iterator instance on every call — once consumed, it stays exhausted, so a second traversal in production code will silently see zero children. Use `thenAnswer(inv -> list.iterator())` to hand back a fresh iterator per invocation, exactly as the real Sling implementation does.

### 6.1.7 ValueMap with defaults and type coercion

```java
@Test
void shouldFallBackToDefaultWhenPropertyAbsent() {
    ValueMap valueMap = new ValueMapDecorator(new HashMap<>());
    // no "bedrooms" key set

    int bedrooms = valueMap.get("bedrooms", 2); // default = 2

    assertEquals(2, bedrooms);
}
```

---

## 6.2 Session & Node (JCR API) Mocking

### 6.2.1 Concept
`ResourceResolver`/`Resource`/`ValueMap` is the Sling-level abstraction; underneath it sits the raw JCR API — `Session`, `Node`, `Property`. You reach it via `resourceResolver.adaptTo(Session.class)`. Some AEM subsystems (workflows, ACL/permission code, versioning, low-level node-type manipulation) still operate directly on JCR because Sling's Resource API doesn't expose everything JCR can do (e.g., mixin types, node ordering, explicit locking).

JCR is intentionally exception-heavy: nearly every `Session`/`Node` method declares `throws RepositoryException` (or a more specific subclass like `PathNotFoundException`, `ItemExistsException`, `AccessDeniedException`, `ConstraintViolationException`). Your test methods therefore typically need `throws RepositoryException` in their signature, or need to wrap the exception-throwing stub in `assertThrows`.

### 6.2.2 Technicalities to know
- `Node.setProperty(...)` stages a change in-session; nothing is persisted until `session.save()` (or, in newer JCR, `session.refresh()`/autosave in certain resource resolver configurations). A common bug is asserting persistence without verifying `save()` was actually invoked — always `verify(session).save()` when your test cares about durability.
- `PathNotFoundException` extends `ItemNotFoundException`? No — actually `PathNotFoundException extends RepositoryException` directly and is thrown by `getNode`/`getItem`/`getProperty` when the path doesn't resolve; `AccessDeniedException` is thrown for permission failures. Know the hierarchy when writing `assertThrows` — asserting the wrong exception type is a frequent false-positive in generated tests.
- `Node.hasNode(relPath)` / `hasProperty(relPath)` are cheap existence checks that avoid a `PathNotFoundException` try/catch — production code that skips these checks and instead catches exceptions for control flow is a code-smell worth flagging in review, and worth a dedicated negative test.
- Mocking deep chains (`session.getNode(...).getNode(...).getProperty(...)`) gets unwieldy fast — this is the strongest signal to switch to AemContext's in-memory JCR (6.2.4) instead of continuing to stub the chain.

### 6.2.3 Pure Mockito Session/Node example

```java
@ExtendWith(MockitoExtension.class)
class PropertyApprovalWorkflowTest {

    @Mock private Session session;
    @Mock private Node propertyNode;
    @Mock private Node statusNode;
    @Mock private Property statusProperty;

    private PropertyApprovalService approvalService;

    @BeforeEach
    void setUp() {
        approvalService = new PropertyApprovalService();
    }

    @Test
    void shouldApprovePropertyAndSetStatusNode() throws RepositoryException {
        // Arrange
        when(session.getNode("/content/properties/villa-101")).thenReturn(propertyNode);
        when(propertyNode.hasNode("jcr:content")).thenReturn(true);
        when(propertyNode.getNode("jcr:content")).thenReturn(statusNode);
        when(statusNode.hasProperty("approvalStatus")).thenReturn(true);
        when(statusNode.getProperty("approvalStatus")).thenReturn(statusProperty);

        // Act
        approvalService.approveProperty(session, "/content/properties/villa-101");

        // Assert
        verify(statusNode).setProperty("approvalStatus", "APPROVED");
        verify(session).save();
    }

    @Test
    void shouldThrowWhenNodeDoesNotExist() throws RepositoryException {
        when(session.getNode(anyString())).thenThrow(new PathNotFoundException("no such node"));

        assertThrows(PathNotFoundException.class,
            () -> approvalService.approveProperty(session, "/content/properties/missing"));

        verify(session, never()).save();
    }

    @Test
    void shouldNotSaveWhenAccessDenied() throws RepositoryException {
        when(session.getNode("/content/properties/villa-101")).thenReturn(propertyNode);
        when(propertyNode.hasNode("jcr:content")).thenReturn(true);
        when(propertyNode.getNode("jcr:content")).thenReturn(statusNode);
        doThrow(new AccessDeniedException("insufficient privileges"))
            .when(statusNode).setProperty("approvalStatus", "APPROVED");

        assertThrows(AccessDeniedException.class,
            () -> approvalService.approveProperty(session, "/content/properties/villa-101"));

        verify(session, never()).save();
    }
}
```

### 6.2.4 Using AemContext to get a real (mocked) Session

`AemContext` with `ResourceResolverType.JCR_MOCK` or `JCR_OAK` gives you an in-memory JCR repository, so you often don't need to hand-mock `Session`/`Node` at all — you can create real nodes and assert real behavior, which is more robust than stubbing every JCR method and doesn't silently drift from real JCR semantics the way a hand-rolled stub chain can.

```java
@Test
void shouldPersistApprovalStatusInRealMockRepo() throws Exception {
    context.create().resource("/content/properties/villa-101/jcr:content",
        "approvalStatus", "PENDING");

    Session session = context.resourceResolver().adaptTo(Session.class);
    PropertyApprovalService service = new PropertyApprovalService();

    service.approveProperty(session, "/content/properties/villa-101");

    Node contentNode = session.getNode("/content/properties/villa-101/jcr:content");
    assertEquals("APPROVED", contentNode.getProperty("approvalStatus").getString());
}
```

**Guidance:** Prefer 6.2.4 (real in-memory JCR via `AemContext`) for anything beyond 2–3 method calls deep. Reserve pure Mockito Session/Node stubbing (6.2.3) for testing **error paths** (`RepositoryException`, `PathNotFoundException`, `AccessDeniedException`) that are difficult or impossible to trigger organically in an in-memory repo.

---

## 6.3 TagManager & Tag Mocking

### 6.3.1 Concept
AEM's tagging system (`com.day.cq.tagging`) stores taxonomy nodes (`cq:Tag`) under `/content/cq:tags`, namespaced by category (e.g., `amenities:swimming-pool`, `property-type:villa`). `TagManager` (adapted from `ResourceResolver`) is the service that resolves a tag ID string to a `Tag` object (giving you title, description, and the tag's own resource), and can also do the reverse — find all resources tagged with a given tag.

Property listings typically store tags as a multi-value String property (e.g., `amenityTags`), and the UI/API layer needs `TagManager` to turn those raw IDs into human-readable, locale-aware labels.

### 6.3.2 Technicalities to know
- `TagManager.resolve(tagId)` returns `null` — not an exception — when the tag doesn't exist. This is the single most common oversight in production code: editors/content authors delete a tag from the taxonomy console, and any code that assumes `resolve()` always succeeds throws an NPE downstream.
- `Tag.getTitle()` without a `Locale` argument returns the default (English) title; `Tag.getTitle(Locale)` supports localized tag titles — relevant if the property portal supports multiple languages (e.g., Tamil + English for a Tamil Nadu real-estate site).
- Tag IDs are namespace-qualified (`namespace:path/to/tag`); `TagManager.resolve()` also accepts the tag's raw JCR path (`/content/cq:tags/amenities/swimming-pool`) — tests should cover both forms if production code accepts user/author input that might use either.
- `TagManager.createTag(...)` requires session write access and is normally tested against AemContext's in-memory repo rather than mocked, since taxonomy creation logic (namespace auto-creation, path sanitization) is non-trivial to stub faithfully.

### 6.3.3 AemContext-based TagManager test

```java
@Test
void shouldResolveAmenityTagTitles() {
    context.create().resource("/content/cq:tags/amenities/swimming-pool",
        "jcr:primaryType", "cq:Tag",
        "jcr:title", "Swimming Pool");

    context.create().resource("/content/properties/villa-101",
        "amenityTags", new String[]{"amenities:swimming-pool"});

    TagManager tagManager = context.resourceResolver().adaptTo(TagManager.class);
    PropertyListingService service = context.registerInjectActivateService(new PropertyListingServiceImpl());

    List<String> amenityLabels = service.getAmenityLabels(
        context.resourceResolver().getResource("/content/properties/villa-101"), tagManager);

    assertEquals(List.of("Swimming Pool"), amenityLabels);
}
```

### 6.3.4 Pure Mockito TagManager/Tag

```java
@ExtendWith(MockitoExtension.class)
class AmenityLabelResolverTest {

    @Mock private TagManager tagManager;
    @Mock private Tag poolTag;
    @Mock private Tag gymTag;

    @InjectMocks
    private AmenityLabelResolver resolver;

    @Test
    void shouldResolveMultipleTagsToTitles() {
        when(tagManager.resolve("amenities:swimming-pool")).thenReturn(poolTag);
        when(tagManager.resolve("amenities:gym")).thenReturn(gymTag);
        when(poolTag.getTitle()).thenReturn("Swimming Pool");
        when(gymTag.getTitle()).thenReturn("Gym");

        List<String> labels = resolver.resolveLabels(
            tagManager, new String[]{"amenities:swimming-pool", "amenities:gym"});

        assertEquals(List.of("Swimming Pool", "Gym"), labels);
    }

    @Test
    void shouldSkipUnresolvableTagsGracefully() {
        when(tagManager.resolve("amenities:invalid-tag")).thenReturn(null);

        List<String> labels = resolver.resolveLabels(tagManager, new String[]{"amenities:invalid-tag"});

        assertTrue(labels.isEmpty());
    }

    @Test
    void shouldResolveLocalizedTagTitleWhenLocaleProvided() {
        when(tagManager.resolve("amenities:swimming-pool")).thenReturn(poolTag);
        when(poolTag.getTitle(Locale.of("ta"))).thenReturn("நீச்சல் குளம்");

        String label = resolver.resolveLabel(tagManager, "amenities:swimming-pool", Locale.of("ta"));

        assertEquals("நீச்சல் குளம்", label);
    }
}
```

---

## 6.4 PageManager & Page Mocking

### 6.4.1 Concept
`PageManager` (adapted from `ResourceResolver`, `com.day.cq.wcm.api`) is the WCM-level API for working with pages rather than raw resources — it understands `cq:Page`/`cq:PageContent` structure, page properties, templates, and hierarchy (`getParent()`, `listChildren()`). It's used heavily for navigation, breadcrumbs, related-content, and template/design lookups.

A `Page` wraps a `jcr:content` child resource; most page-property reads (`getTitle()`, `getProperties()`) actually delegate to that child resource's `ValueMap` under the hood — useful to know when a test's expectations don't match because a property was set on the page resource itself instead of on `jcr:content`.

### 6.4.2 Technicalities to know
- `Page.getTitle()` falls back through `jcr:title` → `nodeName` if no title is set — don't assume a `null`/empty title means "no page," it may just mean "untitled page using the node name as its display title" depending on your version/config.
- `PageManager.getContainingPage(Resource)` walks *up* the tree to find the nearest ancestor `cq:Page` — different from `getPage(path)` which requires an exact page path. Confusing these two is a common bug when working with component resources nested inside a page.
- Recursive parent-walks (breadcrumbs, ancestor lookups) require an explicit stop condition. In real AEM, `Page.getParent()` returns `null` once you pass `/content` (or the JCR root) — but a **stubbed** mock does *not* know this automatically; if you forget to stub the terminal `getParent()` call to return `null`, the loop under test will spin until it hits a `NullPointerException` from an un-stubbed call, or — worse — hang if the loop condition is written to tolerate nulls incorrectly.
- `page.getProperties()` returns a `ValueMap` (same object type/semantics as 6.1) — everything you know about ValueMap defaults/type coercion applies directly here.

### 6.4.3 AemContext PageManager test

```java
@Test
void shouldFindSimilarPropertiesUnderSameLocalityPage() {
    context.create().page("/content/properties/chennai", "Chennai Properties");
    context.create().page("/content/properties/chennai/villa-101", "Villa 101");
    context.create().page("/content/properties/chennai/villa-102", "Villa 102");

    PageManager pageManager = context.pageManager();
    Page localityPage = pageManager.getPage("/content/properties/chennai");

    PropertyListingService service = context.registerInjectActivateService(new PropertyListingServiceImpl());
    List<Page> siblings = service.getSimilarProperties(localityPage, "/content/properties/chennai/villa-101");

    assertEquals(1, siblings.size());
    assertEquals("Villa 102", siblings.get(0).getTitle());
}
```

### 6.4.4 Pure Mockito PageManager/Page

```java
@ExtendWith(MockitoExtension.class)
class BreadcrumbBuilderTest {

    @Mock private PageManager pageManager;
    @Mock private Page currentPage;
    @Mock private Page parentPage;
    @Mock private Page rootPage;

    @InjectMocks
    private BreadcrumbBuilder breadcrumbBuilder;

    @Test
    void shouldBuildBreadcrumbTrailUpToPropertiesRoot() {
        when(currentPage.getTitle()).thenReturn("Villa 101");
        when(currentPage.getParent()).thenReturn(parentPage);
        when(parentPage.getTitle()).thenReturn("Chennai");
        when(parentPage.getPath()).thenReturn("/content/properties/chennai");
        when(parentPage.getParent()).thenReturn(rootPage);
        when(rootPage.getPath()).thenReturn("/content/properties");
        when(rootPage.getTitle()).thenReturn("Properties");
        when(rootPage.getParent()).thenReturn(null); // explicit stop condition — do not omit

        List<String> trail = breadcrumbBuilder.build(currentPage);

        assertEquals(List.of("Properties", "Chennai", "Villa 101"), trail);
    }

    @Test
    void shouldReturnOnlyCurrentPageWhenNoParent() {
        when(currentPage.getTitle()).thenReturn("Villa 101");
        when(currentPage.getParent()).thenReturn(null);

        List<String> trail = breadcrumbBuilder.build(currentPage);

        assertEquals(List.of("Villa 101"), trail);
    }
}
```

---

## 6.5 AssetManager Mocking

### 6.5.1 Concept
`AssetManager` (`com.day.cq.dam.api`, adapted from `ResourceResolver`) is DAM's equivalent of `PageManager` — it manages `dam:Asset` nodes rather than `cq:Page` nodes. An `Asset` (typically an image, PDF brochure, or video for a property listing) has one or more `Rendition`s — the original binary plus generated variants (thumbnails, web-optimized sizes) produced asynchronously by DAM update workflows.

### 6.5.2 Technicalities to know
- Renditions are **not** created synchronously on upload — `createAsset(...)` persists the original binary immediately, but thumbnail/web renditions are generated by an async DAM update workflow. Code that assumes `getRendition("cq5dam.thumbnail.140.100.png")` is available immediately after upload is a real production bug; always test the "rendition not yet generated" fallback (6.5.4, second test).
- `Asset.getRendition(name)` returns `null` if that specific rendition doesn't exist; `Asset.getOriginal()` always returns the original rendition and is a safe fallback.
- `Asset.getMetadataValue(String)` reads DAM XMP/technical metadata (`dam:size`, `dc:format`, `tiff:ImageWidth`, etc.) — values come back as `String` regardless of the underlying JCR property type, so numeric metadata (like file size) typically needs manual parsing (`Long.parseLong(...)`) in production code — a good place to assert both valid-numeric and malformed-metadata test cases.
- `Rendition` extends `Resource`, so once you have a `Rendition` you can `adaptTo(...)` it just like any other resource (e.g., to get an `InputStream` via `rendition.getStream()` — note: `Rendition.getStream()`, not `adaptTo(InputStream.class)`, is the documented API).
- `AemContext` offers a convenience `context.create().asset(path, width, height, mimeType)` helper (via `context.load()`/`AemContextBuilder`) that's often faster than manually building `dam:Asset`/`dam:AssetContent` resource trees node-by-node.

### 6.5.3 AemContext AssetManager test

```java
@Test
void shouldUploadPropertyBrochureAsAsset() throws Exception {
    context.registerService(MimeTypeService.class, mock(MimeTypeService.class));
    AssetManager assetManager = context.resourceResolver().adaptTo(AssetManager.class);

    InputStream brochureStream = new ByteArrayInputStream("dummy-pdf-content".getBytes());
    Asset asset = assetManager.createAsset(
        "/content/dam/properties/villa-101/brochure.pdf",
        brochureStream, "application/pdf", true);

    assertNotNull(asset);
    assertEquals("/content/dam/properties/villa-101/brochure.pdf", asset.getPath());
}
```

### 6.5.4 Pure Mockito AssetManager/Asset/Rendition

```java
@ExtendWith(MockitoExtension.class)
class PropertyGalleryServiceTest {

    @Mock private AssetManager assetManager;
    @Mock private Asset heroImageAsset;
    @Mock private Rendition thumbnailRendition;

    @InjectMocks
    private PropertyGalleryService galleryService;

    @Test
    void shouldReturnThumbnailRenditionPathForHeroImage() {
        when(assetManager.getAsset("/content/dam/properties/villa-101/hero.jpg"))
            .thenReturn(heroImageAsset);
        when(heroImageAsset.getRendition("cq5dam.thumbnail.140.100.png"))
            .thenReturn(thumbnailRendition);
        when(thumbnailRendition.getPath())
            .thenReturn("/content/dam/properties/villa-101/hero.jpg/jcr:content/renditions/cq5dam.thumbnail.140.100.png");

        String thumbPath = galleryService.getThumbnailPath("/content/dam/properties/villa-101/hero.jpg");

        assertTrue(thumbPath.endsWith("cq5dam.thumbnail.140.100.png"));
    }

    @Test
    void shouldFallBackToOriginalWhenThumbnailRenditionMissing() {
        when(assetManager.getAsset(anyString())).thenReturn(heroImageAsset);
        when(heroImageAsset.getRendition(anyString())).thenReturn(null);
        when(heroImageAsset.getOriginal()).thenReturn(thumbnailRendition);
        when(thumbnailRendition.getPath()).thenReturn("/content/dam/properties/villa-101/hero.jpg/jcr:content/renditions/original");

        String thumbPath = galleryService.getThumbnailPath("/content/dam/properties/villa-101/hero.jpg");

        assertTrue(thumbPath.endsWith("original"));
    }

    @Test
    void shouldReadMetadataFromAsset() {
        when(assetManager.getAsset(anyString())).thenReturn(heroImageAsset);
        when(heroImageAsset.getMetadataValue("dam:size")).thenReturn("204800");

        long sizeInBytes = galleryService.getAssetSize("/content/dam/properties/villa-101/hero.jpg");

        assertEquals(204800L, sizeInBytes);
    }

    @Test
    void shouldHandleMalformedSizeMetadataGracefully() {
        when(assetManager.getAsset(anyString())).thenReturn(heroImageAsset);
        when(heroImageAsset.getMetadataValue("dam:size")).thenReturn("not-a-number");

        long sizeInBytes = galleryService.getAssetSize("/content/dam/properties/villa-101/hero.jpg");

        assertEquals(0L, sizeInBytes); // service defensively returns 0 rather than throwing
    }
}
```

---

## 6.6 Phase 6 Summary Table

| API | Preferred mocking approach | When to use pure Mockito instead | Key gotcha |
|---|---|---|---|
| `ResourceResolver` / `ValueMap` | AemContext (`context.create().resource(...)`) | Constructor-injected `ResourceResolver`, or null/missing-resource branches | `listChildren()` iterator is single-use — stub with `thenAnswer` |
| `Session` / `Node` | AemContext JCR_MOCK/JCR_OAK (real in-memory repo) | `RepositoryException`/access-denied error paths | Nothing persists without `session.save()` — assert it explicitly |
| `TagManager` / `Tag` | AemContext with real `cq:Tag` resources | Unresolvable/null tag handling, localization | `resolve()` returns `null`, not an exception, for missing tags |
| `PageManager` / `Page` | AemContext (`context.create().page(...)`) | Recursive traversal logic (breadcrumbs, ancestor walks) | Must stub terminal `getParent()` → `null`, or the loop under test misbehaves |
| `AssetManager` / `Asset` / `Rendition` | AemContext `adaptTo(AssetManager.class)` | Rendition-fallback and metadata-edge-case testing | Renditions generate asynchronously — never assume they exist right after upload |

---

**Next: Phase 7 — QueryBuilder mocking** (predicate construction, `SearchResult`, pagination, and full-text/tag-combination queries against the Property Listing search API — will use `JCR_OAK` context since QueryBuilder needs real index behavior).

Ready for Phase 7 whenever you are.
