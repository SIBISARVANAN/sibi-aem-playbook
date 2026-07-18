# AEM JUnit Testing — Complete Guide

## Phase 1 — Foundation

---

### 1.1 Testing Stack and pom.xml Setup

#### What You Need and Why

AEM unit testing uses four libraries working together:

| Library | Purpose |
|---|---|
| `io.wcm.testing.aem-mock-junit5` | The core — provides `AemContext`, mock JCR, mock Sling, mock AEM objects |
| `org.mockito:mockito-core` | Mocking external dependencies (services, JCR objects you don't control) |
| `org.junit.jupiter:junit-jupiter` | JUnit 5 test runner |
| `mockito-inline` | Required ONLY for static mocking (`MockedStatic`) — Phase 9 |

Without `aem-mock`, you would have to manually wire up a Sling resource resolver, a JCR session, OSGi service registries — hundreds of lines of setup before writing a single assertion. `aem-mock` does all that for you in one line: `new AemContext()`.

#### Version Compatibility

This is the most common setup mistake. Use the wrong version combination and you get cryptic classpath errors.

| AEM Version | Recommended aem-mock version | JUnit |
|---|---|---|
| AEM 6.5 (UberJar 6.5.x) | `aem-mock-junit5` 4.1.x or 5.x | JUnit 5 |
| AEM as a Cloud Service (SDK) | `aem-mock-junit5` 5.x | JUnit 5 |
| Legacy AEM 6.4 | `aem-mock-junit5` 3.x | JUnit 5 |

#### core/pom.xml — Add These Dependencies

Find the `<dependencies>` section in `core/pom.xml` and add:

```xml
<!-- ═══════════════════════════════════════════════════════════════
     TEST DEPENDENCIES
     All scoped as "test" — NOT bundled into your OSGi JAR.
     Never use compile or provided scope for test libraries.
     ═══════════════════════════════════════════════════════════════ -->

<!-- AEM Mock — provides AemContext, mock JCR, mock Sling runtime.
     This single dependency transitively pulls in:
       - sling-mock (ResourceResolver, Resource, ValueMap mocks)
       - jcr-mock (JCR Session, Node mocks)
       - osgi-mock (OSGi service registry simulation)
       - AEM-specific mocks (PageManager, TagManager etc.) -->
<dependency>
    <groupId>io.wcm.testing</groupId>
    <artifactId>io.wcm.testing.aem-mock.junit5</artifactId>
    <version>5.3.0</version>
    <scope>test</scope>
</dependency>

<!-- JUnit 5 API — the @Test, @BeforeEach, @ExtendWith annotations -->
<dependency>
    <groupId>org.junit.jupiter</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>

<!-- Mockito Core — @Mock, @InjectMocks, when(), verify(), ArgumentCaptor -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-core</artifactId>
    <version>5.5.0</version>
    <scope>test</scope>
</dependency>

<!-- Mockito JUnit 5 integration — enables @ExtendWith(MockitoExtension.class) -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-junit-jupiter</artifactId>
    <version>5.5.0</version>
    <scope>test</scope>
</dependency>

<!-- Mockito Inline — ONLY needed for static mocking (Phase 9).
     Allows Mockito.mockStatic(SomeClass.class).
     Requires JVM arg: --add-opens in surefire config (see below). -->
<dependency>
    <groupId>org.mockito</groupId>
    <artifactId>mockito-inline</artifactId>
    <version>5.2.0</version>
    <scope>test</scope>
</dependency>

<!-- JUnit Vintage Engine — ONLY needed if your project mixes JUnit 4
     and JUnit 5 tests. Allows @RunWith(MockitoJUnitRunner.class) tests
     to run alongside @ExtendWith(MockitoExtension.class) tests.
     Remove this if your project is purely JUnit 5. -->
<dependency>
    <groupId>org.junit.vintage</groupId>
    <artifactId>junit-vintage-engine</artifactId>
    <version>5.10.0</version>
    <scope>test</scope>
</dependency>
```

#### maven-surefire-plugin — Critical Configuration

JUnit 5 tests will silently not run (zero tests executed, zero failures, build passes — but nothing was tested) unless Surefire is properly configured. This is the #1 silent failure in AEM projects.

Find or add the `maven-surefire-plugin` in your `core/pom.xml` build plugins:

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <!-- Must be 2.22.0+ for JUnit 5 support.
                 Earlier versions only understand JUnit 4. -->
            <version>3.1.2</version>
            <configuration>
                <!-- This arg-line is required for mockito-inline (static mocking)
                     in Java 17+. Without it, static mocking fails with
                     InaccessibleObjectException at runtime.
                     Safe to include even if you don't use static mocking yet. -->
                <argLine>
                    --add-opens java.base/java.lang=ALL-UNNAMED
                    --add-opens java.base/java.util=ALL-UNNAMED
                </argLine>

                <!-- Run tests in parallel for faster builds.
                     classes = run test classes in parallel (safe for independent tests).
                     methods = run methods in the same class in parallel (use carefully). -->
                <parallel>classes</parallel>
                <threadCount>4</threadCount>

                <!-- Include both JUnit 4 and JUnit 5 test class naming patterns -->
                <includes>
                    <include>**/*Test.java</include>
                    <include>**/*Tests.java</include>
                    <include>**/*IT.java</include>
                </includes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

#### How to Verify the Setup Works

Run this from the project root:

```bash
mvn test -pl core -Dtest=HelloWorldModelTest
```

If you see:
```
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
```
Setup is correct.

If you see:
```
Tests run: 0, Failures: 0, Errors: 0, Skipped: 0
```
Surefire is not discovering JUnit 5 tests — check the plugin version and configuration.

---

### 1.2 AemContext — JCR_MOCK vs JCR_OAK vs RESOURCERESOLVER_MOCK

This is the most important decision you make at the start of every test class. The wrong choice either makes your test fail with cryptic errors or makes it unnecessarily slow.

#### What AemContext Actually Does

`AemContext` is a JUnit extension that spins up a lightweight, in-memory simulation of the AEM runtime before each test and tears it down after. It provides:

- A mock `ResourceResolver` backed by a simulated repository
- A mock OSGi service registry (you register services into it)
- A mock Sling Model factory (you register model classes into it)
- Mock `SlingHttpServletRequest` and `SlingHttpServletResponse`
- AEM-specific adapters: `PageManager`, `TagManager`, etc.

It does NOT start a real AEM instance or connect to any external system.

#### RESOURCERESOLVER_MOCK — Fastest, Most Limited

```java
AemContext ctx = new AemContext(ResourceResolverType.RESOURCERESOLVER_MOCK);
```

**What it provides:**
- A pure in-memory `ResourceResolver` with no JCR backend at all
- Resources created via `ctx.create().resource()` are stored in a simple HashMap
- `ValueMap` reading works perfectly

**What it CANNOT do:**
- No JCR `Session` (adapting to Session returns null)
- No JCR queries of any kind
- No node ordering or hierarchy traversal beyond simple parent/child

**Use when:**
- Your model only reads simple properties via `@ValueMapValue`
- No JCR queries, no Session, no `Node` access
- You need the absolute fastest test execution

```java
// RESOURCERESOLVER_MOCK example
@ExtendWith(AemContextExtension.class)
class SimpleModelTest {

    // No explicit type specified — defaults to RESOURCERESOLVER_MOCK
    private final AemContext ctx = new AemContext();

    @Test
    void testTitleProperty() {
        ctx.addModelsForClasses(MyModel.class);
        ctx.create().resource("/content/test",
            "jcr:title", "Hello World",
            "subtitle",  "A subtitle");

        ctx.currentResource("/content/test");
        MyModel model = ctx.request().adaptTo(MyModel.class);

        assertNotNull(model);
        assertEquals("Hello World", model.getTitle());
    }
}
```

#### JCR_MOCK — The Most Common Choice

```java
AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);
```

**What it provides:**
- Everything RESOURCERESOLVER_MOCK provides, PLUS
- A real JCR `Session` (adapting `ResourceResolver` to `Session` works)
- JCR `Node`, `Property`, `NodeType` access
- JCR `ValueFactory`
- Support for JCR node types (`cq:Page`, `nt:unstructured`, `dam:Asset`)

**What it CANNOT do:**
- JCR queries (SQL-2, XPath, QueryBuilder) — the query engine is not present
- Oak-specific APIs (segment store, node state, etc.)

**Use when:**
- Your model or service accesses the JCR `Session` or `Node` directly
- You use `resource.adaptTo(Node.class)`
- You need `PageManager`, `Page`, `TagManager` (these need proper node types)
- The vast majority of Sling Model, OSGi service, servlet, filter tests

```java
// JCR_MOCK example — testing a model that uses PageManager
@ExtendWith(AemContextExtension.class)
class PageAwareModelTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Test
    void testPagePath() {
        ctx.addModelsForClasses(HelloWorldModel.class);

        // ctx.create().page() works because JCR_MOCK supports node types
        ctx.create().page("/content/mysite/en/home",
            "/apps/mysite/templates/page",
            "jcr:title", "Home Page");

        ctx.currentResource("/content/mysite/en/home/jcr:content");
        HelloWorldModel model = ctx.request().adaptTo(HelloWorldModel.class);

        assertNotNull(model);
        assertEquals("/content/mysite/en/home", model.getPagePath());
    }
}
```

#### JCR_OAK — Real Oak Repository, Slowest

```java
AemContext ctx = new AemContext(ResourceResolverType.JCR_OAK);
```

**What it provides:**
- A REAL Oak in-memory repository
- Full JCR query execution (SQL-2, XPath)
- QueryBuilder works properly
- Lucene indexing (async — you may need to wait or use property indexes in tests)
- Full Oak API including node states, segments

**What it costs:**
- Significantly slower startup — Oak initialises a full repository per test class
- More complex setup when testing indexes
- Async indexing means full-text queries may not return results immediately after save

**Use when:**
- Testing code that uses `QueryBuilder` directly
- Testing code that executes JCR SQL-2 or XPath queries
- Testing Oak-specific behaviour

```java
// JCR_OAK example — QueryBuilder test
@ExtendWith(AemContextExtension.class)
class ArticleSearchServiceTest {

    // Must use JCR_OAK — JCR_MOCK has no query engine
    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_OAK);

    @Test
    void testSearchReturnsMatchingPages() {
        // Create test pages in the real Oak repository
        ctx.create().page("/content/mysite/en/villa-maldives");
        ctx.create().resource("/content/mysite/en/villa-maldives/jcr:content",
            "propertyType", "villa",
            "status",       "available");

        // Now run the real QueryBuilder query against Oak
        // ... (full example in Phase 7)
    }
}
```

#### Decision Chart

```
Does your code run JCR queries (QueryBuilder, SQL-2, XPath)?
    YES → JCR_OAK
    NO  ↓
Does your code adapt ResourceResolver/Resource to Session or Node?
    YES → JCR_MOCK
    NO  ↓
Does your code use PageManager, TagManager, or Asset?
    YES → JCR_MOCK
    NO  ↓
Does your code only read properties via ValueMap or @ValueMapValue?
    YES → RESOURCERESOLVER_MOCK (or default AemContext())
```

---

### 1.3 JUnit 4 vs JUnit 5 in AEM

#### Why This Matters

The AEM Maven archetype still generates projects with JUnit 4 by default. Many senior developers have institutional muscle memory for JUnit 4. But JUnit 5 is clearly superior and worth migrating to. You need to know both because you will encounter both in real projects.

#### Annotation Mapping

| JUnit 4 | JUnit 5 | Notes |
|---|---|---|
| `@RunWith(MockitoJUnitRunner.class)` | `@ExtendWith(MockitoExtension.class)` | Class-level runner |
| `@RunWith(AemContextExtension.class)` | `@ExtendWith(AemContextExtension.class)` | Same annotation name, different package |
| `@Before` | `@BeforeEach` | Runs before every test method |
| `@After` | `@AfterEach` | Runs after every test method |
| `@BeforeClass` | `@BeforeAll` | Runs once before all tests in the class — must be static |
| `@AfterClass` | `@AfterAll` | Runs once after all tests — must be static |
| `@Test(expected=Exception.class)` | `assertThrows(Exception.class, () -> ...)` | Exception testing |
| `@Test(timeout=1000)` | `@Timeout(1)` | Timeout testing |
| `@Ignore` | `@Disabled` | Skip a test |
| `@Rule` | No direct equivalent | Use extensions or `@BeforeEach` instead |
| `Assert.assertEquals()` | `Assertions.assertEquals()` — or static import | Import changed |

#### JUnit 4 Test (Old Style — What You'll Find in Legacy Projects)

```java
import org.junit.Before;
import org.junit.Rule;
import org.junit.Test;
import org.junit.runner.RunWith;
import io.wcm.testing.mock.aem.junit.AemContext;           // JUnit 4 package
import io.wcm.testing.mock.aem.junit.AemContextExtension; // JUnit 4
import org.mockito.junit.MockitoJUnitRunner;

import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotNull;

@RunWith(MockitoJUnitRunner.class)  // JUnit 4 runner
public class AuthorImplTest {

    // JUnit 4 AemContext is a @Rule, not a constructor field
    @Rule
    public final AemContext ctx = new AemContext();

    @Before  // not @BeforeEach
    public void setUp() {
        ctx.addModelsForClasses(AuthorImpl.class);
        ctx.load().json("/content/test-author.json", "/content");
        ctx.currentResource("/content/author");
    }

    @Test  // same annotation, different import
    public void testGetFirstName() {
        Author model = ctx.request().adaptTo(Author.class);
        assertNotNull(model);
        assertEquals("Sibi", model.getFirstName());
    }
}
```

#### JUnit 5 Test (Modern — What You Should Write)

```java
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import io.wcm.testing.mock.aem.junit5.AemContext;           // JUnit 5 package
import io.wcm.testing.mock.aem.junit5.AemContextExtension; // JUnit 5
import org.mockito.junit.jupiter.MockitoExtension;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;

// Multiple extensions — both AemContext AND Mockito active at the same time
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class AuthorImplTest {

    // JUnit 5 AemContext is a regular field, not a @Rule
    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach  // not @Before
    void setUp() {
        ctx.addModelsForClasses(AuthorImpl.class);
        ctx.load().json("/content/test-author.json", "/content");
        ctx.currentResource("/content/author");
    }

    @Test
    void testGetFirstName() {
        Author model = ctx.request().adaptTo(Author.class);
        assertNotNull(model);
        assertEquals("Sibi", model.getFirstName());
    }
}
```

#### Key Differences Explained

**1. `@Rule` vs constructor field**

In JUnit 4, `AemContext` must be declared as `@Rule` — this is JUnit 4's way of managing objects that need setup/teardown per test. In JUnit 5, this mechanism was replaced by the Extension system. `AemContext` becomes a plain `final` field — the `AemContextExtension` registered via `@ExtendWith` handles its lifecycle automatically.

**2. Multiple extensions in JUnit 5**

JUnit 5 supports multiple `@ExtendWith` annotations simultaneously. This lets you use `AemContextExtension` (for the AEM runtime) alongside `MockitoExtension` (for `@Mock` field injection) in the same test class — without any conflicts.

**3. `assertThrows` in JUnit 5**

JUnit 4 used `@Test(expected = SomeException.class)` but this had a flaw — if the exception was thrown by setup code rather than the method under test, the test would still pass. JUnit 5 fixes this:

```java
// JUnit 4 — imprecise, could catch exception from wrong place
@Test(expected = WorkflowException.class)
public void testWorkflowFailure() throws Exception {
    // if this line threw WorkflowException, test would wrongly pass
    workflowProcess = new ProductPublishProcess();
    workflowProcess.execute(workItem, session, args);  // intended to throw
}

// JUnit 5 — precise, only the lambda is expected to throw
@Test
void testWorkflowFailure() {
    WorkflowException thrown = assertThrows(
        WorkflowException.class,
        () -> workflowProcess.execute(workItem, session, args)
    );
    assertEquals("Publishing failed", thrown.getMessage());
}
```

---

### 1.4 JSON Fixtures

JSON fixtures are test data files that `AemContext` loads into its mock JCR repository. Instead of programmatically calling `ctx.create().resource()` fifty times, you write one JSON file and load it in one line.

#### Where to Put Fixture Files

```
core/src/test/
├── java/
│   └── com/sibi/aem/one/core/models/
│       └── AuthorImplTest.java
└── resources/
    └── com/sibi/aem/one/core/models/      ← mirror the test class package
        └── AuthorImplTest/                ← folder named after the test class
            ├── content.json               ← main fixture
            └── author-with-variants.json  ← fixture for a specific scenario
```

Mirroring the package structure is a convention — it keeps fixtures next to the test that uses them and avoids name collisions.

#### Loading a Fixture

```java
// The second argument is WHERE in the mock JCR to mount the content.
// The JSON file's root maps to this path.
ctx.load().json("/com/sibi/aem/one/core/models/AuthorImplTest/content.json",
                "/content");

// Now the JCR has nodes at /content/author, /content/author/jcr:content etc.
ctx.currentResource("/content/author/jcr:content");
```

#### JSON Structure — Simple Node

```json
{
  "author": {
    "jcr:primaryType": "nt:unstructured",
    "firstName": "Sibi",
    "lastName":  "Sarvanan",
    "gender":    "Male",
    "email":     "sibi@example.com",
    "author:title": "Mr"
  }
}
```

This creates a node at `/content/author` (because it was mounted at `/content`) with those properties.

#### JSON Structure — cq:Page (Most Common in AEM Tests)

```json
{
  "mysite": {
    "jcr:primaryType": "cq:Page",
    "en": {
      "jcr:primaryType": "cq:Page",
      "properties": {
        "jcr:primaryType": "cq:Page",
        "jcr:content": {
          "jcr:primaryType":    "cq:PageContent",
          "jcr:title":          "Property Listings",
          "sling:resourceType": "sibi-aem-one/components/page/listingpage",
          "propertyType":       "villa",
          "status":             "available",
          "price":              450000.0,
          "currency":           "GBP",
          "featured":           true
        }
      }
    }
  }
}
```

This creates the path `/content/mysite/en/properties` with a `jcr:content` child.

#### JSON Structure — Multifield (Child Nodes)

Multifield dialog values are stored as child nodes. This is the fixture pattern for `@ChildResource`:

```json
{
  "product": {
    "jcr:primaryType": "nt:unstructured",
    "jcr:title":       "Classic T-Shirt",
    "sku":             "SHIRT-001",
    "price":           29.99,
    "variants": {
      "jcr:primaryType": "nt:unstructured",
      "item0": {
        "jcr:primaryType": "nt:unstructured",
        "size":      "M",
        "colour":    "Red",
        "variantSku": "SHIRT-001-M-RED",
        "stock":     12
      },
      "item1": {
        "jcr:primaryType": "nt:unstructured",
        "size":      "L",
        "colour":    "Blue",
        "variantSku": "SHIRT-001-L-BLU",
        "stock":     0
      }
    }
  }
}
```

When the model has `@ChildResource(name = "variants")`, it receives `item0` and `item1` as individual `Resource` objects in a `List<Resource>`.

#### JSON Structure — Multi-value Property (String Array)

```json
{
  "page": {
    "jcr:primaryType": "nt:unstructured",
    "amenityTags": [
      "mysite:amenities/pool",
      "mysite:amenities/gym",
      "mysite:amenities/parking"
    ]
  }
}
```

AemContext maps JSON arrays to `String[]` properties in JCR.

#### Pro Tip — Validate Your Fixture First

If `adaptTo()` returns null or your model returns unexpected values, the fixture is usually the culprit. Print it to verify:

```java
@Test
void debugFixture() {
    ctx.load().json("/fixtures/content.json", "/content");
    Resource resource = ctx.resourceResolver().getResource("/content/product");
    assertNotNull(resource, "Resource should exist — check fixture path and mount point");

    ValueMap vm = resource.getValueMap();
    System.out.println("Properties: " + vm.keySet()); // prints all property names
    System.out.println("Title: " + vm.get("jcr:title", String.class));
}
```

---

### 1.5 Test Naming Conventions

Good test names are CI output that reads like a specification. When a test fails at 2 AM, the name should tell you exactly what broke without opening the file.

#### The Pattern: `methodName_scenario_expectedBehaviour`

```java
// POOR — tells you nothing when it fails
@Test
void test1() { ... }

@Test
void testGetTitle() { ... }

// GOOD — reads like a specification
@Test
void getTitle_whenTitlePropertyExists_returnsTitle() { ... }

@Test
void getTitle_whenTitlePropertyMissing_returnsEmptyString() { ... }

@Test
void getFormattedPrice_whenCurrencyIsGBP_returnsPoundSign() { ... }

@Test
void getFormattedPrice_whenPriceIsNull_returnsPriceUnavailable() { ... }

@Test
void searchProperties_whenQueryIsBlank_returnsEmptyResult() { ... }

@Test
void execute_whenReplicationFails_throwsWorkflowException() { ... }
```

#### Why This Matters in CI Output

Maven Surefire generates an XML report that CI tools (Jenkins, GitHub Actions) display as a test results table. With good names:

```
FAILED: getFormattedPrice_whenCurrencyIsGBP_returnsPoundSign
```

You immediately know: the formatted price method has a bug in the GBP branch. You don't need to open the file.

With poor names:
```
FAILED: test47
```

You know nothing.

#### Complete Example of a Well-Named Test Class

```java
@ExtendWith(AemContextExtension.class)
class PropertyListingImplTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @BeforeEach
    void setUp() {
        ctx.addModelsForClasses(PropertyListingImpl.class);
    }

    // ── getFormattedPrice ──────────────────────────────────────────────────

    @Test
    void getFormattedPrice_whenPriceAndCurrencyPresent_returnsFormattedString() {
        ctx.create().resource("/content/prop",
            "price", 450000.0, "currency", "GBP");
        ctx.currentResource("/content/prop");
        PropertyListing model = ctx.request().adaptTo(PropertyListing.class);
        assertEquals("£450,000.00", model.getFormattedPrice());
    }

    @Test
    void getFormattedPrice_whenPriceIsNull_returnsPriceUnavailableMessage() {
        ctx.create().resource("/content/prop", "currency", "GBP"); // no price
        ctx.currentResource("/content/prop");
        PropertyListing model = ctx.request().adaptTo(PropertyListing.class);
        assertEquals("Price unavailable", model.getFormattedPrice());
    }

    @Test
    void getFormattedPrice_whenCurrencyIsEUR_returnsEuroSign() {
        ctx.create().resource("/content/prop",
            "price", 350000.0, "currency", "EUR");
        ctx.currentResource("/content/prop");
        PropertyListing model = ctx.request().adaptTo(PropertyListing.class);
        assertTrue(model.getFormattedPrice().startsWith("€"));
    }

    // ── getVariants ────────────────────────────────────────────────────────

    @Test
    void getVariants_whenNoChildNodes_returnsEmptyList() {
        ctx.create().resource("/content/prop", "sku", "SHIRT-001");
        ctx.currentResource("/content/prop");
        PropertyListing model = ctx.request().adaptTo(PropertyListing.class);
        assertNotNull(model.getVariants());
        assertTrue(model.getVariants().isEmpty());
    }

    @Test
    void getVariants_whenTwoChildNodesExist_returnsTwoVariants() {
        ctx.load().json("/fixtures/product-with-variants.json", "/content");
        ctx.currentResource("/content/product");
        PropertyListing model = ctx.request().adaptTo(PropertyListing.class);
        assertEquals(2, model.getVariants().size());
    }
}
```

---

### Phase 1 — Summary

| Concept | Key Takeaway |
|---|---|
| pom.xml | `aem-mock-junit5` + `mockito-core` + `junit-jupiter` all at `test` scope. Surefire must be 2.22.0+ |
| RESOURCERESOLVER_MOCK | Fastest. Only for pure property reading. No JCR Session, no queries. |
| JCR_MOCK | Default for 90% of tests. Has Session, Node, Page, TagManager. No queries. |
| JCR_OAK | Real Oak. Slow. Only for QueryBuilder / JCR query tests. |
| JUnit 4 vs 5 | `@Rule` → field, `@Before` → `@BeforeEach`, `@RunWith` → `@ExtendWith`. Add vintage-engine for mixed projects. |
| JSON fixtures | Mirror the test class package. Mount at `/content`. Array → `String[]`. Child nodes → `@ChildResource`. |
| Test naming | `method_scenario_expected`. Saves debugging time in CI. |
