# AEM JUnit Testing — Phase 2: Sling Models

---

### 2.1 Basic Sling Model Test

#### The Model We Are Testing

Before writing tests, always know your model's structure. We use `AuthorImpl` from your repository as the base example — it covers the most common injection patterns.

```java
// The interface (public contract)
public interface Author {
    String getFirstName();
    String getLastName();
    String getTitle();
    String getGender();
    String getEmail();
}

// The implementation (what we actually test)
@Model(
    adaptables = { Resource.class, SlingHttpServletRequest.class },
    adapters   = Author.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL
)
public class AuthorImpl implements Author {

    @Inject @Default(values = "Sibi")
    private String firstName;

    @ValueMapValue @Default(values = "Sarvanan")
    private String lastName;

    @Inject @Named("author:title") @Default(values = "Mr")
    private String title;

    @ValueMapValue @Default(values = "Male")
    private String gender;

    @ValueMapValue @Default(values = "sibi@example.com")
    private String email;

    @Override public String getFirstName() { return firstName; }
    @Override public String getLastName()  { return lastName; }
    @Override public String getTitle()     { return title; }
    @Override public String getGender()    { return gender; }
    @Override public String getEmail()     { return email; }
}
```

#### Full Test Class — Registering and Adapting

```java
package com.sibi.aem.one.core.models;

import com.sibi..aem.one.core.models.impl.AuthorImpl;
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.apache.sling.testing.mock.sling.ResourceResolverType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(AemContextExtension.class)
class AuthorImplTest {

    // JCR_MOCK is sufficient — AuthorImpl only reads JCR properties,
    // no JCR queries, no Session access.
    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        // Step 1: Tell AemContext which Sling Model classes to register.
        // Without this, adaptTo() returns null because the model factory
        // doesn't know AuthorImpl exists.
        // You can register multiple classes: addModelsForClasses(A.class, B.class)
        // Or register an entire package: addModelsForPackage("com.sibi.aem.one.core.models")
        ctx.addModelsForClasses(AuthorImpl.class);
    }

    // ── Adapting from Resource ──────────────────────────────────────────────

    @Test
    void getFirstName_whenPropertyExists_returnsStoredValue() {
        // Step 2: Create a resource in the mock JCR with the required properties.
        // Map.of() / varargs key-value pairs both work.
        ctx.create().resource("/content/author",
            "firstName",   "Sibi",
            "lastName",    "Sarvanan",
            "gender",      "Male",
            "email",       "sibi@test.com",
            "author:title", "Mr");

        // Step 3: Set the current resource — this is what ctx.request().adaptTo()
        // and ctx.resourceResolver().getResource() will use.
        ctx.currentResource("/content/author");

        // Step 4: Adapt and assert.
        // Adapting from Resource:
        Author modelFromResource = ctx.currentResource().adaptTo(Author.class);
        assertNotNull(modelFromResource, "Model should not be null — check addModelsForClasses()");
        assertEquals("Sibi", modelFromResource.getFirstName());
    }

    // ── Adapting from SlingHttpServletRequest ──────────────────────────────

    @Test
    void getEmail_whenAdaptingFromRequest_returnsStoredValue() {
        ctx.create().resource("/content/author",
            "email", "sibi@test.com");
        ctx.currentResource("/content/author");

        // Adapting from request is needed when the model injects
        // @Self SlingHttpServletRequest or @ScriptVariable fields.
        // ctx.request() automatically points to ctx.currentResource().
        Author modelFromRequest = ctx.request().adaptTo(Author.class);

        assertNotNull(modelFromRequest);
        assertEquals("sibi@test.com", modelFromRequest.getEmail());
    }

    // ── Testing @Default fallback values ───────────────────────────────────

    @Test
    void getFirstName_whenPropertyAbsent_returnsDefaultValue() {
        // Create a resource with NO properties at all.
        // @Default(values = "Sibi") should kick in.
        ctx.create().resource("/content/empty-author");
        ctx.currentResource("/content/empty-author");

        Author model = ctx.request().adaptTo(Author.class);

        assertNotNull(model);
        // @Default(values = "Sibi") is declared on the firstName field
        assertEquals("Sibi", model.getFirstName(),
            "Should return the @Default value when property is absent");
    }

    @Test
    void getTitle_whenNamespacedPropertyAbsent_returnsDefaultValue() {
        // "author:title" uses a JCR namespace — verify the @Named mapping works
        ctx.create().resource("/content/author-no-title",
            "firstName", "TestUser"
            // intentionally NO "author:title" property
        );
        ctx.currentResource("/content/author-no-title");

        Author model = ctx.request().adaptTo(Author.class);
        assertEquals("Mr", model.getTitle(),
            "Should return @Default 'Mr' when author:title is absent");
    }

    // ── Testing with JSON fixture ───────────────────────────────────────────

    @Test
    void allGetters_whenLoadedFromJsonFixture_returnCorrectValues() {
        // Load the fixture file — second arg is where it mounts in the JCR.
        // The JSON root object maps to /content.
        // So a key "author" inside the JSON creates /content/author.
        ctx.load().json("/com/sibi/aem/one/core/models/AuthorImplTest/content.json",
                        "/content");

        ctx.currentResource("/content/author");
        Author model = ctx.request().adaptTo(Author.class);

        assertNotNull(model);
        assertEquals("Sibi",          model.getFirstName());
        assertEquals("Sarvanan",      model.getLastName());
        assertEquals("Mr",            model.getTitle());
        assertEquals("Male",          model.getGender());
        assertEquals("sibi@test.com", model.getEmail());
    }

    // ── Verifying adaptTo() returns null gracefully ─────────────────────────

    @Test
    void adaptTo_whenResourceHasWrongResourceType_returnsNullSafely() {
        // Models registered with resourceType will only adapt from nodes
        // with that sling:resourceType. Without it, adaptation may fail.
        // This test ensures your code handles null adaptTo() correctly.
        ctx.create().resource("/content/wrong-type",
            "sling:resourceType", "some/other/component");
        ctx.currentResource("/content/wrong-type");

        // If your model has resourceType set and the resource doesn't match,
        // adaptTo() returns null. Good code always null-checks the result.
        Author model = ctx.request().adaptTo(Author.class);
        // For AuthorImpl (no resourceType restriction), this adapts fine.
        // In a model WITH resourceType, this would return null.
        // assertNull(model); // uncomment for restricted-resourceType models
    }
}
```

#### Fixture File — `content.json`

Place at: `core/src/test/resources/com/sibi/aem/one/core/models/AuthorImplTest/content.json`

```json
{
  "author": {
    "jcr:primaryType": "nt:unstructured",
    "firstName":    "Sibi",
    "lastName":     "Sarvanan",
    "gender":       "Male",
    "email":        "sibi@test.com",
    "author:title": "Mr"
  }
}
```

---

### 2.2 Testing @PostConstruct

#### Why @PostConstruct Is Automatically Called

When you call `adaptTo()`, AemContext wires all injections AND calls `@PostConstruct` automatically — you never call it manually in a test. This means your test assertions on derived fields (fields set by `@PostConstruct`) work exactly like assertions on directly-injected fields.

#### The Model Under Test

```java
@Model(adaptables = Resource.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class HelloWorldModel {

    @ValueMapValue(injectionStrategy = InjectionStrategy.OPTIONAL)
    private String resourceType;

    @SlingObject
    private Resource currentResource;

    @SlingObject
    private ResourceResolver resourceResolver;

    // Derived field — computed by @PostConstruct, not injected directly
    private String message;

    @PostConstruct
    protected void init() {
        // This runs automatically after all field injections.
        // Uses the injected resourceResolver to get PageManager.
        PageManager pm = resourceResolver.adaptTo(PageManager.class);
        String pagePath = Optional.ofNullable(pm)
            .map(p -> p.getContainingPage(currentResource))
            .map(Page::getPath)
            .orElse("unknown");

        String rt = StringUtils.defaultIfBlank(resourceType, "No resourceType");
        this.message = "Component: " + rt + " on page: " + pagePath;
    }

    public String getMessage() { return message; }
}
```

#### Testing Derived Fields Set by @PostConstruct

```java
@ExtendWith(AemContextExtension.class)
class HelloWorldModelTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        ctx.addModelsForClasses(HelloWorldModel.class);
    }

    @Test
    void getMessage_whenResourceTypeSet_includesResourceTypeInMessage() {
        // Create a page — PageManager.getContainingPage() needs a real page hierarchy
        ctx.create().page("/content/mysite/en/home");

        // Create the component resource INSIDE the page's jcr:content
        ctx.create().resource("/content/mysite/en/home/jcr:content/mycomponent",
            "sling:resourceType", "mysite/components/helloworld");

        ctx.currentResource("/content/mysite/en/home/jcr:content/mycomponent");

        // adaptTo() triggers ALL of:
        //   1. @SlingObject injection of currentResource + resourceResolver
        //   2. @ValueMapValue injection of resourceType
        //   3. @PostConstruct init() — builds the message string
        HelloWorldModel model = ctx.request().adaptTo(HelloWorldModel.class);

        assertNotNull(model);
        // message is set by init(), not directly injected
        assertNotNull(model.getMessage());
        assertTrue(model.getMessage().contains("mysite/components/helloworld"),
            "Message should contain the resource type");
        assertTrue(model.getMessage().contains("/content/mysite/en/home"),
            "Message should contain the page path");
    }

    @Test
    void getMessage_whenResourceTypeAbsent_usesDefaultText() {
        ctx.create().page("/content/mysite/en/home");
        ctx.create().resource("/content/mysite/en/home/jcr:content/mycomponent");
        ctx.currentResource("/content/mysite/en/home/jcr:content/mycomponent");

        HelloWorldModel model = ctx.request().adaptTo(HelloWorldModel.class);

        assertNotNull(model);
        // When sling:resourceType is absent, @PostConstruct uses "No resourceType"
        assertTrue(model.getMessage().contains("No resourceType"));
    }

    @Test
    void getMessage_whenResourceNotInsidePage_usesUnknownPagePath() {
        // Resource that is NOT inside any cq:Page hierarchy
        ctx.create().resource("/var/mycomponent",
            "sling:resourceType", "mysite/components/helloworld");
        ctx.currentResource("/var/mycomponent");

        HelloWorldModel model = ctx.request().adaptTo(HelloWorldModel.class);

        assertNotNull(model);
        // PageManager.getContainingPage() returns null for /var — should be "unknown"
        assertTrue(model.getMessage().contains("unknown"),
            "Should contain 'unknown' when no containing page exists");
    }

    // ── Testing exception safety in @PostConstruct ─────────────────────────

    @Test
    void getMessage_whenResourceResolverReturnsNullPageManager_doesNotThrow() {
        // If PageManager is not available, @PostConstruct should handle it
        // gracefully via Optional.ofNullable() — no NullPointerException
        ctx.create().resource("/content/test",
            "sling:resourceType", "mysite/components/helloworld");
        ctx.currentResource("/content/test");

        // If this throws any exception, the test fails — that's the assertion
        assertDoesNotThrow(() -> {
            HelloWorldModel model = ctx.request().adaptTo(HelloWorldModel.class);
            assertNotNull(model);
        });
    }
}
```

---

### 2.3 Testing @ChildResource (Multifield)

#### The Model Under Test

```java
@Model(adaptables = Resource.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ProductImpl implements Product {

    @ValueMapValue private String sku;
    @ValueMapValue private Double price;

    // @ChildResource injects the "variants" child node's children as a List<Resource>
    @ChildResource(name = "variants")
    private List<Resource> variantResources;

    private List<ProductVariant> variants;

    @PostConstruct
    protected void init() {
        if (variantResources == null || variantResources.isEmpty()) {
            variants = Collections.emptyList();
            return;
        }
        variants = variantResources.stream()
            .map(r -> r.adaptTo(ProductVariant.class))
            .filter(Objects::nonNull)
            .collect(Collectors.toList());
    }

    public List<ProductVariant> getVariants() { return variants; }
    public String getSku()   { return sku; }
    public Double getPrice() { return price; }
}
```

#### Testing @ChildResource with JSON Fixture

```java
@ExtendWith(AemContextExtension.class)
class ProductImplTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        // Register BOTH the parent model AND the child model
        // If you forget ProductVariant, adaptTo(ProductVariant.class) returns null
        // inside @PostConstruct, and your variants list will be empty
        ctx.addModelsForClasses(ProductImpl.class, ProductVariant.class);
    }

    @Test
    void getVariants_whenTwoChildNodesExist_returnsTwoAdaptedVariants() {
        // Load fixture with nested multifield structure
        ctx.load().json(
            "/com/sibi/aem/one/core/models/ProductImplTest/product-with-variants.json",
            "/content");

        ctx.currentResource("/content/product");
        Product model = ctx.currentResource().adaptTo(Product.class);

        assertNotNull(model);
        assertNotNull(model.getVariants());
        assertEquals(2, model.getVariants().size(), "Should have 2 variants");

        // Verify first variant
        ProductVariant first = model.getVariants().get(0);
        assertEquals("M",              first.getSize());
        assertEquals("Red",            first.getColour());
        assertEquals("SHIRT-001-M-RED", first.getVariantSku());
        assertEquals(12,               first.getStock());
        assertTrue(first.isInStock(),  "Stock=12 means in stock");
        assertEquals("In Stock",       first.getAvailabilityLabel());

        // Verify second variant (out of stock)
        ProductVariant second = model.getVariants().get(1);
        assertEquals("L",    second.getSize());
        assertFalse(second.isInStock(), "Stock=0 means out of stock");
        assertEquals("Out of Stock", second.getAvailabilityLabel());
    }

    @Test
    void getVariants_whenNoChildNodesExist_returnsEmptyListNotNull() {
        // Product with NO variants node at all
        ctx.create().resource("/content/product-no-variants",
            "sku", "SHIRT-002", "price", 49.99);
        ctx.currentResource("/content/product-no-variants");

        Product model = ctx.currentResource().adaptTo(Product.class);

        assertNotNull(model);
        // Critical: must be empty list, NOT null — HTL data-sly-list breaks on null
        assertNotNull(model.getVariants(), "Variants list must never be null");
        assertTrue(model.getVariants().isEmpty());
    }

    @Test
    void getVariants_whenVariantsNodeExistsButEmpty_returnsEmptyList() {
        // The "variants" container node exists but has no children
        ctx.create().resource("/content/product-empty-variants",
            "sku", "SHIRT-003");
        // Create the container node with no children
        ctx.create().resource("/content/product-empty-variants/variants");
        ctx.currentResource("/content/product-empty-variants");

        Product model = ctx.currentResource().adaptTo(Product.class);

        assertNotNull(model);
        assertTrue(model.getVariants().isEmpty());
    }

    @Test
    void getVariants_whenChildAdaptationReturnsNull_filtersOutNullEntries() {
        // This tests the .filter(Objects::nonNull) in the stream.
        // If a child resource fails to adapt (e.g. wrong resource type),
        // it should be silently filtered, not cause a NullPointerException.

        // Create a variant with MISSING required properties — adaptation still
        // works because ProductVariant uses OPTIONAL injection strategy.
        ctx.create().resource("/content/product-bad-variants",
            "sku", "SHIRT-004");
        ctx.create().resource("/content/product-bad-variants/variants/item0",
            // All properties missing — ProductVariant will adapt but fields will be null/default
            "size", "S");
        ctx.currentResource("/content/product-bad-variants");

        Product model = ctx.currentResource().adaptTo(Product.class);

        assertNotNull(model);
        // Should not throw — null filter in stream handles bad children
        assertDoesNotThrow(() -> model.getVariants().forEach(v -> {
            assertNotNull(v); // no nulls in the list
        }));
    }
}
```

#### Fixture File for Multifield Test

`core/src/test/resources/com/sibi/aem/one/core/models/ProductImplTest/product-with-variants.json`

```json
{
  "product": {
    "jcr:primaryType": "nt:unstructured",
    "sku":   "SHIRT-001",
    "price": 29.99,
    "variants": {
      "jcr:primaryType": "nt:unstructured",
      "item0": {
        "jcr:primaryType": "nt:unstructured",
        "size":       "M",
        "colour":     "Red",
        "variantSku": "SHIRT-001-M-RED",
        "stock":      12
      },
      "item1": {
        "jcr:primaryType": "nt:unstructured",
        "size":       "L",
        "colour":     "Blue",
        "variantSku": "SHIRT-001-L-BLU",
        "stock":      0
      }
    }
  }
}
```

---

### 2.4 Testing @OSGiService Injection in a Sling Model

#### The Model Under Test

```java
@Model(adaptables = SlingHttpServletRequest.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class ProductImpl implements Product {

    @ValueMapValue private String sku;
    @ValueMapValue private Double price;

    // OSGi service injected directly into the Sling Model
    @OSGiService
    private InventoryService inventoryService;

    private int stockCount = -1;
    private String formattedPrice;

    @PostConstruct
    protected void init() {
        buildFormattedPrice();
        fetchStockCount();
    }

    private void fetchStockCount() {
        if (StringUtils.isBlank(sku) || inventoryService == null) return;
        try {
            stockCount = inventoryService.getStockCount(sku);
        } catch (Exception e) {
            stockCount = -1;
        }
    }

    private void buildFormattedPrice() {
        formattedPrice = (price != null) ? "£" + price : "Price unavailable";
    }

    public int getStockCount()    { return stockCount; }
    public String getFormattedPrice() { return formattedPrice; }
}
```

#### Testing with @OSGiService — The Key Pattern

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class ProductImplOsgiServiceTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    // Mockito creates a mock implementation of InventoryService
    @Mock
    private InventoryService inventoryService;

    @BeforeEach
    void setUp() {
        // CRITICAL ORDER:
        // 1. Register the mock service FIRST
        // 2. Then register/adapt the model
        // If you reverse this order, the model's @PostConstruct runs before
        // the service is registered, and stockCount will always be -1.
        ctx.registerService(InventoryService.class, inventoryService);
        ctx.addModelsForClasses(ProductImpl.class);
    }

    @Test
    void getStockCount_whenServiceReturnsValue_returnsCorrectCount()
            throws InventoryService.InventoryServiceException {
        // Arrange — stub the service
        when(inventoryService.getStockCount("SHIRT-001")).thenReturn(42);

        ctx.create().resource("/content/product", "sku", "SHIRT-001", "price", 29.99);
        ctx.currentResource("/content/product");

        // Act — adaptTo triggers @PostConstruct which calls inventoryService
        Product model = ctx.request().adaptTo(Product.class);

        // Assert
        assertNotNull(model);
        assertEquals(42, model.getStockCount());
        // Verify the service was called exactly once with the correct SKU
        verify(inventoryService, times(1)).getStockCount("SHIRT-001");
    }

    @Test
    void getStockCount_whenServiceThrowsException_returnsMinus1()
            throws InventoryService.InventoryServiceException {
        // Simulate a service failure (e.g. Magento API is down)
        when(inventoryService.getStockCount(anyString()))
            .thenThrow(new InventoryService.InventoryServiceException("API down", null));

        ctx.create().resource("/content/product", "sku", "SHIRT-001");
        ctx.currentResource("/content/product");

        Product model = ctx.request().adaptTo(Product.class);

        assertNotNull(model);
        // When service throws, model should return -1 (the safe fallback)
        assertEquals(-1, model.getStockCount(),
            "Should return -1 when inventory service throws");
    }

    @Test
    void getStockCount_whenServiceNotRegistered_returnsMinus1() {
        // Create a FRESH AemContext without registering InventoryService
        // This simulates a missing/unavailable OSGi service
        AemContext freshCtx = new AemContext(ResourceResolverType.JCR_MOCK);
        freshCtx.addModelsForClasses(ProductImpl.class);
        // Note: freshCtx.registerService() is NOT called here

        freshCtx.create().resource("/content/product", "sku", "SHIRT-001");
        freshCtx.currentResource("/content/product");

        Product model = freshCtx.request().adaptTo(Product.class);

        assertNotNull(model);
        // @OSGiService with OPTIONAL strategy — model adapts even when service missing
        // The service field is null, @PostConstruct null-checks it, returns -1
        assertEquals(-1, model.getStockCount(),
            "Should return -1 when service not available");
    }

    @Test
    void getStockCount_whenSkuIsBlank_doesNotCallService()
            throws InventoryService.InventoryServiceException {
        // Resource with no SKU property
        ctx.create().resource("/content/product-no-sku", "price", 29.99);
        ctx.currentResource("/content/product-no-sku");

        ctx.request().adaptTo(Product.class);

        // Verify the service was NEVER called — blank SKU should be a no-op
        verify(inventoryService, never()).getStockCount(anyString());
    }

    @Test
    void getFormattedPrice_isIndependentOfInventoryService() {
        // getFormattedPrice comes from a local calculation, not from inventoryService.
        // This test verifies the two concerns are properly separated.
        ctx.create().resource("/content/product",
            "sku", "SHIRT-001", "price", 450000.0);
        ctx.currentResource("/content/product");

        Product model = ctx.request().adaptTo(Product.class);

        assertNotNull(model);
        assertEquals("£450000.0", model.getFormattedPrice());
        // InventoryService is not involved in price formatting — no verify needed
    }
}
```

#### Registering Multiple Services

```java
@BeforeEach
void setUp() {
    // Register multiple services — order matters if they depend on each other
    ctx.registerService(InventoryService.class, inventoryServiceMock);
    ctx.registerService(TagManager.class, tagManagerMock);
    ctx.registerService(ExternalApiService.class, externalApiMock);

    ctx.addModelsForClasses(ProductImpl.class);
}
```

#### Registering a Service with OSGi Properties

Some services use `@Reference(target = "...")` filters. You can register with properties to match those filters:

```java
// Register with a service property so LDAP filter @Reference(target="(type=primary)") matches
ctx.registerService(MyService.class, myMock,
    "type", "primary",
    "service.ranking", 100);
```

---

### 2.5 Testing Jackson Exporter Output

#### What the Exporter Does

A model annotated with `@Exporter(name="jackson")` can be accessed at `<resource>.model.json`. The Jackson library serialises the model's public getters into JSON. Testing the exporter verifies: (a) the JSON shape is correct, (b) `@JsonIgnore` fields are excluded, (c) `@JsonProperty` renamed fields appear correctly.

#### The Model Under Test

```java
@Model(adaptables = { Resource.class, SlingHttpServletRequest.class },
       adapters   = Author.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL,
       resourceType = "sibi-aem-one/components/content/author")
@Exporter(name = "jackson", selector = "model", extensions = "json",
    options = {
        @ExporterOption(name = "SerializationFeature.WRAP_ROOT_VALUE", value = "true")
    })
@JsonRootName("AuthorDetails")
public class AuthorImpl implements Author {

    @ValueMapValue
    private String firstName;

    @ValueMapValue
    private String lastName;

    @ValueMapValue
    @JsonIgnore  // excluded from JSON output
    private String internalNotes;

    @ValueMapValue
    @JsonProperty("authorEmail")  // renamed in JSON output
    private String email;

    // Getters...
    public String getFirstName()     { return firstName; }
    public String getLastName()      { return lastName; }
    public String getInternalNotes() { return internalNotes; }
    public String getEmail()         { return email; }
}
```

#### Testing the Exporter

Testing the Jackson exporter requires invoking Sling's model exporter machinery. The cleanest approach is to use `ctx.request()` with the `.model.json` extension and read the response.

```java
@ExtendWith(AemContextExtension.class)
class AuthorImplExporterTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        ctx.addModelsForClasses(AuthorImpl.class);
    }

    @Test
    void modelJson_whenExported_containsFirstNameAndLastName() throws Exception {
        ctx.create().resource("/content/author",
            "sling:resourceType", "sibi-aem-one/components/content/author",
            "firstName",    "Sibi",
            "lastName",     "Sarvanan",
            "email",        "sibi@test.com",
            "internalNotes","DO NOT EXPOSE");

        ctx.currentResource("/content/author");

        // Adapt to the model and then export manually using ObjectMapper
        AuthorImpl model = (AuthorImpl) ctx.request().adaptTo(Author.class);
        assertNotNull(model);

        // Use Jackson's ObjectMapper directly — the same one AEM's exporter uses
        ObjectMapper mapper = new ObjectMapper();
        mapper.enable(SerializationFeature.WRAP_ROOT_VALUE);

        String json = mapper.writeValueAsString(model);

        // ── Assert included fields ──────────────────────────────────────
        assertTrue(json.contains("\"firstName\":\"Sibi\""),
            "firstName should be in JSON output");
        assertTrue(json.contains("\"lastName\":\"Sarvanan\""),
            "lastName should be in JSON output");

        // ── Assert @JsonProperty renaming ───────────────────────────────
        assertTrue(json.contains("\"authorEmail\":\"sibi@test.com\""),
            "email should be renamed to authorEmail in JSON output");
        assertFalse(json.contains("\"email\""),
            "original 'email' key should NOT appear — it was renamed");

        // ── Assert @JsonIgnore exclusion ────────────────────────────────
        assertFalse(json.contains("internalNotes"),
            "@JsonIgnore field must NOT appear in JSON output");
        assertFalse(json.contains("DO NOT EXPOSE"),
            "Value of @JsonIgnore field must not leak into JSON");

        // ── Assert @JsonRootName wrapping ───────────────────────────────
        assertTrue(json.contains("\"AuthorDetails\""),
            "JSON should be wrapped under the root name 'AuthorDetails'");
    }

    @Test
    void modelJson_whenFieldIsNull_omitsFieldFromOutput() throws Exception {
        // Create resource with only firstName — other fields will be null
        ctx.create().resource("/content/author-partial",
            "sling:resourceType", "sibi-aem-one/components/content/author",
            "firstName", "Sibi"
            // lastName, email are absent
        );
        ctx.currentResource("/content/author-partial");

        AuthorImpl model = (AuthorImpl) ctx.request().adaptTo(Author.class);

        ObjectMapper mapper = new ObjectMapper();
        // Configure to exclude null values — same as production AEM behaviour
        mapper.setSerializationInclusion(JsonInclude.Include.NON_NULL);
        mapper.enable(SerializationFeature.WRAP_ROOT_VALUE);

        String json = mapper.writeValueAsString(model);

        assertTrue(json.contains("\"firstName\":\"Sibi\""));
        // lastName is null and NON_NULL configured — should not appear
        assertFalse(json.contains("lastName"));
    }

    @Test
    void customSerializer_whenVariantListPresent_outputsOnlyExposedFields()
            throws Exception {
        // For models using custom @JsonSerialize(using=...) serializers,
        // test that the serializer produces the correct shape.
        // Example: ProductSummary with ProductVariantListSerializer

        ctx.addModelsForClasses(ProductImpl.class, ProductVariant.class);
        ctx.load().json("/fixtures/product-with-variants.json", "/content");
        ctx.currentResource("/content/product");

        ProductImpl model = (ProductImpl) ctx.currentResource().adaptTo(Product.class);
        assertNotNull(model);

        ProductSummary summary = model.getSummary();
        ObjectMapper mapper = new ObjectMapper();
        String json = mapper.writeValueAsString(summary);

        // Custom serializer maps variantSku → "sku" and excludes availabilityLabel
        assertTrue(json.contains("\"sku\""),
            "variantSku should be renamed to 'sku' by custom serializer");
        assertFalse(json.contains("availabilityLabel"),
            "Internal UI field should not appear in JSON output");
        assertFalse(json.contains("variantSku"),
            "Original field name should not appear — serializer renamed it");

        // Verify the "available" boolean is present
        assertTrue(json.contains("\"available\""),
            "isInStock() should be serialized as 'available'");
    }
}
```

---

### Phase 2 — Summary

| Pattern | Key Points |
|---|---|
| Basic model test | `addModelsForClasses()` before `adaptTo()`. Always null-check the result. |
| Adapting from Resource vs Request | Resource for background/no-request contexts. Request when model needs `@ScriptVariable`, `@Self SlingHttpServletRequest`. |
| @Default values | Create a resource with NO properties, verify the default is returned. |
| @PostConstruct | Called automatically by `adaptTo()`. Test derived fields the same as injected fields. Test the null/absent dependency path. |
| @ChildResource | Register child model class too (`addModelsForClasses(Parent.class, Child.class)`). Fixture must have child node structure. Test empty list scenario. |
| @OSGiService in model | `ctx.registerService()` BEFORE `ctx.addModelsForClasses()`. Test service-unavailable path (no registration = null injection with OPTIONAL). Use `verify()` to confirm service was called correctly. |
| Jackson exporter | Use `ObjectMapper` directly in tests. Test `@JsonIgnore` exclusion, `@JsonProperty` renaming, `@JsonRootName` wrapping, and custom serializer output. |
