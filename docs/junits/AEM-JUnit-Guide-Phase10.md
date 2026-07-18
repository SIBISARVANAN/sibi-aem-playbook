# AEM JUnit Testing Guide — Phase 10: Master Cheat Sheet & JaCoCo Setup

**Scenario used throughout:** consolidates every mock pattern from Phases 6–9 against the Property Listing codebase into quick-reference form, then covers coverage tooling for the same Maven-based project.

---

## 10.0 Concept — Why a Cheat Sheet Phase

Phases 6–9 built understanding, one topic at a time. Phase 10 is deliberately reference-shaped, not narrative — the goal is something you can scan in an interview prep sprint or keep open in a second monitor while writing real tests, without re-reading the conceptual explanations each time.

---

## 10.1 MockitoExtension vs AemContextExtension — Decision Guide

*(Answering directly, and consolidating what's been implicit since Phase 6.)*

### 10.1.1 Concept
Every phase so far has used one or both of these, but let's make the decision rule explicit and complete here.

- **`MockitoExtension`** activates `@Mock`/`@InjectMocks`/`@Captor`/`@Spy` field processing — pure Java object graph, zero Sling/JCR/OSGi runtime. Fastest possible test startup.
- **`AemContextExtension`** boots a mock (or real, for `JCR_OAK`) Sling/JCR runtime — real `ResourceResolver`, real `PageManager`/`TagManager`/`AssetManager` behavior backed by an actual (fake or real) content tree, real Sling Models injection processing, real OSGi component activation lifecycle (`@Activate`/`@Reference` resolution via `registerInjectActivateService`).

### 10.1.2 Can be done with EITHER (your choice is about test style, not capability)
- Testing a plain OSGi service class's business logic where all collaborators (JCR, other services) can be reasonably stubbed by hand: `MockitoExtension` with hand-built mocks, **or** `AemContextExtension` with `context.create().resource(...)` real content — Phase 6 showed both paths side-by-side for exactly this reason (ResourceResolver/ValueMap, PageManager/Page, TagManager/Tag, AssetManager).
- `WorkflowProcess.execute(...)` testing (Phase 8) — technically *could* be done with `AemContext` (register the process as a service, build real workflow content), but there's no real benefit since `WorkItem`/`WorkflowSession`/`MetaDataMap` aren't Sling/JCR-native objects requiring a resolver — `MockitoExtension` is simply faster and was the recommended default throughout Phase 8.
- Simple ResourceResolver traversal/ValueMap reads — either works; prefer `MockitoExtension` for narrow, few-collaborator logic, and `AemContext` once you're touching 3+ chained real-ish objects (a resource tree with several levels), since a real in-memory tree becomes easier to set up correctly than a deep hand-stubbed chain.

### 10.1.3 Can ONLY be done with AemContextExtension (Mockito alone is structurally insufficient)
- **Sling Models field injection** (`@ValueMapValue`, `@ChildResource`, `@OSGiService`, `@Self`, `@SlingObject`, etc.) — these annotations are inert without the real Sling Models injector framework processing them; `@InjectMocks` does plain type-matching and has no awareness of these annotations at all (explicitly called out in Phase 9.5.2). There is no pure-Mockito way to exercise this.
- **Real QueryBuilder/predicate-matching correctness** (Phase 7.1) — `JCR_MOCK`/`JCR_OAK` via `AemContext` is mandatory because QueryBuilder needs an actual query engine underneath; there is nothing to "mock" here that would prove predicates genuinely match the right content — that's an integration property, not a unit-testable one via stubs.
- **OSGi component activation lifecycle behavior** — testing what happens inside a class's own `@Activate`/`@Modified` method when given a real `ComponentContext`/OSGi config, via `context.registerInjectActivateService(new MyComponent(), configMap)`, exercises the actual activation call; Mockito alone has no concept of OSGi component lifecycle to invoke.
- **Real in-memory JCR node-type/constraint enforcement** (Phase 6.2.4) — asserting that an operation is rejected because it violates a real JCR node type definition (not just "some exception was thrown because I stubbed it to be") requires an actual (fake-but-real) repository; a hand-mocked `Session`/`Node` can only ever return what you told it to return, never something JCR itself would independently reject.
- **Sling servlet/filter request-response round-trips** — testing a `SlingSafeMethodsServlet`/`Filter` against a real (mock) `SlingHttpServletRequest`/`Response` pair, including request parameter resolution, `RequestPathInfo` (selectors/extension parsing), and resource-type based servlet resolution, needs `context.request()`/`context.response()` from `AemContext` — Mockito mocks of `HttpServletRequest` can technically be hand-built, but reproducing Sling's real selector/suffix/extension parsing logic by hand-stubbing is impractical and error-prone; `AemContext` gives you the real parsing for free.

### 10.1.4 Quick decision table

| Scenario | MockitoExtension | AemContextExtension | Notes |
|---|---|---|---|
| Plain OSGi service business logic, few collaborators | ✅ preferred | ✅ works | Phase 6/8 pattern |
| Sling Model with `@OSGiService`/`@ValueMapValue` | ❌ cannot | ✅ required | Phase 9.5 |
| Real QueryBuilder predicate matching | ❌ cannot | ✅ required (`JCR_OAK`) | Phase 7.1 |
| Mocked QueryBuilder chain (transformation logic only) | ✅ preferred | ✅ works but slower | Phase 7.2 |
| WorkflowProcess.execute() logic | ✅ preferred | ✅ works but unnecessary | Phase 8 |
| OSGi `@Activate`/config binding behavior | ❌ cannot | ✅ required | `registerInjectActivateService` |
| Real JCR constraint/node-type enforcement | ❌ cannot | ✅ required (`JCR_MOCK`/`JCR_OAK`) | Phase 6.2 |
| Servlet/Filter selector/suffix parsing | ❌ impractical | ✅ required | `context.request()` |
| Deep ResourceResolver/PageManager tree reads | ✅ works for shallow cases | ✅ preferred for deep/multi-level | Phase 6 |

---

## 10.2 Master Cheat Sheet

### 10.2.1 One-liner setup patterns per component type

```java
// Sling Model
@ExtendWith(AemContextExtension.class)
class MyModelTest {
    private final AemContext context = new AemContext();
    @BeforeEach void setUp() { context.addModelsForClasses(MyModel.class); }
}

// Plain OSGi Service (POJO-style)
@ExtendWith(MockitoExtension.class)
class MyServiceImplTest {
    @Mock private SomeCollaborator collaborator;
    @InjectMocks private MyServiceImpl service;
}

// Sling Servlet
@ExtendWith(AemContextExtension.class)
class MyServletTest {
    private final AemContext context = new AemContext();
    private final MyServlet servlet = new MyServlet();
    @Test void test() throws Exception {
        context.currentResource(context.create().resource("/content/foo"));
        servlet.doGet(context.request(), context.response());
    }
}

// Sling Filter
@ExtendWith(AemContextExtension.class)
class MyFilterTest {
    private final AemContext context = new AemContext();
    @Mock private FilterChain chain;
    @Test void test() throws Exception {
        new MyFilter().doFilter(context.request(), context.response(), chain);
        verify(chain).doFilter(context.request(), context.response());
    }
}

// Sling Job Consumer
@ExtendWith(MockitoExtension.class)
class MyJobConsumerTest {
    @Mock private Job job;
    @InjectMocks private MyJobConsumer consumer;
    @Test void test() {
        when(job.getProperty("path", String.class)).thenReturn("/content/properties/villa-101");
        JobExecutionResult result = consumer.process(job, mock(JobExecutionContext.class));
    }
}

// Event Listener (OSGi EventHandler / JCR EventListener)
@ExtendWith(MockitoExtension.class)
class MyEventListenerTest {
    @Mock private Event event;
    @InjectMocks private MyEventListener listener;
    @Test void test() {
        when(event.getProperty("path")).thenReturn("/content/properties/villa-101");
        listener.handleEvent(event);
    }
}

// WorkflowProcess
@ExtendWith(MockitoExtension.class)
class MyWorkflowProcessTest {
    @Mock private WorkItem workItem;
    @Mock private WorkflowSession workflowSession;
    @Mock private MetaDataMap args;
    private final MyWorkflowProcess process = new MyWorkflowProcess();
    @Test void test() throws WorkflowException { process.execute(workItem, workflowSession, args); }
}
```

### 10.2.2 Complete mock stub chains for every common AEM object

```java
// ResourceResolver -> Resource -> ValueMap
Resource resource = context.create().resource("/content/x", "prop", "value");
ValueMap vm = resource.getValueMap();

// ResourceResolver -> Session -> Node -> Property
Session session = context.resourceResolver().adaptTo(Session.class);
Node node = session.getNode("/content/x/jcr:content");
node.getProperty("prop").getString();

// ResourceResolver -> TagManager -> Tag
TagManager tagManager = context.resourceResolver().adaptTo(TagManager.class);
Tag tag = tagManager.resolve("namespace:tagId");
tag.getTitle();

// ResourceResolver -> PageManager -> Page
PageManager pageManager = context.pageManager();
Page page = pageManager.getPage("/content/x");
page.getTitle(); page.getParent(); page.getProperties();

// ResourceResolver -> AssetManager -> Asset -> Rendition
AssetManager assetManager = context.resourceResolver().adaptTo(AssetManager.class);
Asset asset = assetManager.getAsset("/content/dam/x.jpg");
Rendition rendition = asset.getRendition("cq5dam.thumbnail.140.100.png");

// QueryBuilder -> Query -> SearchResult -> Hit
QueryBuilder qb = mock(QueryBuilder.class);
Query query = mock(Query.class);
SearchResult result = mock(SearchResult.class);
Hit hit = mock(Hit.class);
when(qb.createQuery(any(PredicateGroup.class), any(Session.class))).thenReturn(query);
when(query.getResult()).thenReturn(result);
when(result.getHits()).thenReturn(List.of(hit));
when(hit.getPath()).thenReturn("/content/x");

// SearchResult -> Facets -> Bucket
Facet facet = mock(Facet.class);
Bucket bucket = mock(Bucket.class);
when(result.getFacets()).thenReturn(Map.of("locality", facet));
when(facet.getBuckets()).thenReturn(List.of(bucket));
when(bucket.getValue()).thenReturn("Chennai");
when(bucket.getCount()).thenReturn(12L);

// WorkItem -> WorkflowData -> MetaDataMap (workflow-scoped, NOT step-scoped)
WorkItem workItem = mock(WorkItem.class);
WorkflowData workflowData = mock(WorkflowData.class);
MetaDataMap workflowMeta = mock(MetaDataMap.class);
when(workItem.getWorkflowData()).thenReturn(workflowData);
when(workflowData.getPayload()).thenReturn("/content/x");
when(workflowData.getMetaDataMap()).thenReturn(workflowMeta);

// Replicator
Replicator replicator = mock(Replicator.class);
verify(replicator).replicate(session, ReplicationActionType.ACTIVATE, "/content/x");

// ContentFragment -> ContentElement
ContentFragment fragment = mock(ContentFragment.class);
ContentElement element = mock(ContentElement.class);
when(fragment.getElement("title")).thenReturn(element);
when(element.getContent()).thenReturn("Adyar, Chennai");
```

### 10.2.3 Common assertion patterns

```java
// Basic value assertions
assertEquals(expected, actual);
assertTrue(condition);
assertNotNull(value);
assertArrayEquals(expectedArray, actualArray);   // NOT assertEquals for arrays/multi-value props

// Exception assertions
WorkflowException ex = assertThrows(WorkflowException.class, () -> process.execute(...));
assertTrue(ex.getMessage().contains("expected fragment"));
assertDoesNotThrow(() -> service.doSomethingThatShouldNotFail());

// Interaction verification
verify(mock).methodCalled(arg);
verify(mock, times(3)).methodCalled(any());
verify(mock, never()).methodThatShouldNotRun();
verify(mock, atLeastOnce()).method();
verifyNoMoreInteractions(mock);

// Argument capturing
ArgumentCaptor<Map> captor = ArgumentCaptor.forClass(Map.class);
verify(mock).method(captor.capture());
assertEquals("expected", captor.getValue().get("key"));

// Spy stubbing (NEVER when() on a spy)
doReturn(value).when(spyObject).expensiveMethod();
```

### 10.2.4 Common pitfalls and their fixes

| Pitfall | Symptom | Fix |
|---|---|---|
| `when()` used on a spy | Real (possibly slow/external) method runs unexpectedly during stubbing | Use `doReturn()`/`doThrow()`/`doNothing()` on spies |
| `listChildren()` / iterator stubbed with `thenReturn` | Second traversal in production code sees zero children | `thenAnswer(inv -> list.iterator())` for a fresh iterator each call |
| `assertEquals` on arrays | False failures — arrays compare by reference, not content | `assertArrayEquals` |
| Forgot to stub terminal `getParent()` → `null` in recursive tests | NPE or infinite loop in ancestor/breadcrumb traversal | Explicitly stub the terminal case to return `null` |
| `MockedStatic` not closed | Static mock leaks into unrelated tests, order-dependent failures | Always try-with-resources |
| `@InjectMocks` used on a Sling Model | `@OSGiService`/`@ValueMapValue` fields silently stay `null` | Use `AemContext` + Sling Models injector instead |
| `TagManager.resolve()` / `fragment.getElement()` assumed non-null | NPE when a tag/fragment field was deleted/renamed after content was authored | Always test the null-return path explicitly |
| Confusing `workItem.getMetaDataMap()` with `workItem.getWorkflowData().getMetaDataMap()` | Data that should persist to later workflow steps silently doesn't | Write to the workflow-scoped map (`getWorkflowData().getMetaDataMap()`) for cross-step data |
| Loose `verify(mock).method(any(), any())` on Replicator-style calls | Wrong-action-type bugs (e.g. DEACTIVATE instead of ACTIVATE) go undetected | Verify exact argument values, not blanket `any()`, when correctness of the specific value matters |
| Testing `JCR_MOCK` for QueryBuilder logic | Query always returns empty / throws | Use `JCR_OAK` — `JCR_MOCK` has no query engine |
| Not testing null vs empty-string separately | Production NPE on a case the test suite never exercised | Add a dedicated test for `null`, a separate one for `""`, and a separate one for whitespace-only if relevant |

---

## 10.3 JaCoCo Setup

### 10.3.1 Concept
JaCoCo (Java Code Coverage) instruments your compiled classes during `mvn test`/`mvn verify` and reports which lines/branches actually executed. Coverage numbers are a *signal*, not a goal in themselves — a high percentage with weak assertions (tests that execute code but assert nothing meaningful) is worse than a lower percentage of genuinely meaningful tests, but coverage thresholds are still valuable as a regression-prevention gate, and Adobe Cloud Manager's default quality gates check exactly this via SonarQube.

### 10.3.2 Technicalities to know
- JaCoCo works via **bytecode instrumentation**, either offline (rewriting `.class` files) or, more commonly in Maven builds, via a **Java agent** attached during test execution (`prepare-agent` goal) — this is why the plugin must bind its `prepare-agent` execution *before* the `surefire`/`failsafe` test execution in the Maven lifecycle, or no coverage data is collected at all.
- The `check` goal is what actually **fails the build** below a threshold — merely having the `report` goal only *generates* the HTML/XML report; it does not enforce anything by itself. Teams sometimes add JaCoCo, see a report get generated, and mistakenly assume a coverage gate is now enforced when it isn't — the `check` execution with `<rules>` is the piece that actually blocks a bad build.
- Coverage counters come in several types (`INSTRUCTION`, `LINE`, `BRANCH`, `COMPLEXITY`, `METHOD`, `CLASS`) — `LINE` is the most commonly gated on for readability, but `BRANCH` coverage is arguably more meaningful for catching untested conditional logic (an `if`/`else` where only one branch ever executes can still show 100% line coverage on that line while missing half the actual behavior).
- **Excluding generated/interface classes matters because they inflate or deflate the denominator misleadingly**: Sling Model *interfaces* (no logic to cover), AEM-generated classes, and DTOs with only getters/setters shouldn't count against your meaningful coverage percentage — excluding them via `<excludes>` keeps the reported number honest and focused on classes that actually have logic worth testing.

### 10.3.3 Adding jacoco-maven-plugin to core/pom.xml

```xml
<build>
  <plugins>
    <plugin>
      <groupId>org.jacoco</groupId>
      <artifactId>jacoco-maven-plugin</artifactId>
      <version>0.8.12</version>
      <executions>
        <!-- Attaches the Java agent before tests run -->
        <execution>
          <id>prepare-agent</id>
          <goals>
            <goal>prepare-agent</goal>
          </goals>
        </execution>
        <!-- Generates the HTML/XML report after tests -->
        <execution>
          <id>report</id>
          <phase>test</phase>
          <goals>
            <goal>report</goal>
          </goals>
        </execution>
        <!-- Enforces the minimum coverage threshold, fails the build if not met -->
        <execution>
          <id>jacoco-check</id>
          <phase>verify</phase>
          <goals>
            <goal>check</goal>
          </goals>
          <configuration>
            <rules>
              <rule>
                <element>BUNDLE</element>
                <limits>
                  <limit>
                    <counter>LINE</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.80</minimum>
                  </limit>
                  <limit>
                    <counter>BRANCH</counter>
                    <value>COVEREDRATIO</value>
                    <minimum>0.70</minimum>
                  </limit>
                </limits>
              </rule>
            </rules>
          </configuration>
        </execution>
      </executions>
      <configuration>
        <excludes>
          <exclude>**/models/*Model.class</exclude>          <!-- Sling Model interfaces -->
          <exclude>**/*Impl.class</exclude>                   <!-- adjust per team convention, or exclude generated impls only -->
          <exclude>**/dto/**</exclude>                        <!-- plain data-transfer objects -->
          <exclude>**/*Constants.class</exclude>
        </excludes>
      </configuration>
    </plugin>
  </plugins>
</build>
```

**Note:** the `**/*Impl.class` exclude above is illustrative only — most teams do **not** want to exclude their actual service implementations (that's precisely the logic-bearing code coverage is meant to measure); adjust exclusion patterns to your project's actual generated-vs-hand-written class naming conventions rather than copying this list verbatim.

### 10.3.4 Configuring minimum coverage threshold (build fails below X%)

Shown inline in 10.3.3's `<rule>` block — the key mechanics worth calling out separately:
- `<element>BUNDLE</element>` applies the rule across the whole module; use `CLASS` instead if you want per-class enforcement (stricter — one weak class fails the whole build, not just drags down an average).
- `<value>COVEREDRATIO</value>` with `<minimum>0.80</minimum>` means 80% — JaCoCo's ratio values are fractional (0.0–1.0), not percentages (0–100) — a very common copy-paste bug is writing `<minimum>80</minimum>`, which JaCoCo interprets as 8000%, i.e., an impossible-to-meet threshold that fails every build.
- Multiple `<limit>` entries under one `<rule>` are combined with AND semantics — the build fails if **any** limit isn't met, not just if all are missed.

### 10.3.5 Excluding generated classes and model interfaces from coverage

```xml
<configuration>
  <excludes>
    <exclude>**/generated/**</exclude>
    <exclude>**/*Model.class</exclude>
    <exclude>com/realestate/core/models/*.class</exclude>
  </excludes>
</configuration>
```

Applies to both the `report` and `check` goal executions by default when placed in the plugin's top-level `<configuration>` (as in 10.3.3) — if you need different exclusions per-execution, nest `<configuration>` inside the specific `<execution>` block instead.

### 10.3.6 Running mvn verify to generate the HTML coverage report

```bash
mvn clean verify
```

`clean` ensures no stale instrumented `.class` files or old reports linger from a previous run — a stale JaCoCo `.exec` data file combined with newly compiled classes is a real source of "coverage numbers that don't make sense" confusion, so `clean verify` (not just `verify`) is the safer habit for local runs, especially when debugging why a number looks wrong.

### 10.3.7 Where the report lives

```
core/target/site/jacoco/index.html
```

Open directly in a browser — it's a drill-down report: module → package → class → line-by-line source view with green (covered) / red (uncovered) / yellow (partially covered branch) highlighting. The line-level XML equivalent (`core/target/site/jacoco/jacoco.xml`) is what CI tooling (including SonarQube, 10.3.8) actually consumes — the HTML is for humans, the XML is for machines.

### 10.3.8 Integrating with SonarQube (Cloud Manager quality gate)

- Adobe Cloud Manager's default pipeline runs a SonarQube analysis step that reads the JaCoCo XML report (`jacoco.xml`) — not the raw `.exec` binary file directly — so the `report` goal (which produces the XML alongside the HTML) must run **before** the Sonar analysis step in the pipeline, or Sonar reports 0% coverage regardless of your actual tests.
- Point Sonar at the report via the standard property, typically already wired into Cloud Manager's provided Sonar configuration, but worth knowing if debugging a misconfigured local Sonar run:
  ```properties
  sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
  ```
- **Cloud Manager's quality gate coverage check and your local `jacoco-maven-plugin` `<check>` rule are two separate enforcement points** — your Maven build can pass locally (if you haven't configured `check`, or configured a lower threshold than Cloud Manager's gate) while still failing Cloud Manager's pipeline; conversely, aligning your local `<minimum>` threshold with the actual Cloud Manager gate percentage (commonly communicated by your DevOps/Cloud Manager admin, since it's configurable per program) lets you catch coverage regressions locally before pushing, rather than discovering them only after a pipeline failure.
- New-code coverage vs overall coverage: SonarQube's default quality gate in many Cloud Manager setups checks coverage **on new/changed code** specifically (not just overall bundle coverage) — meaning a large legacy codebase with historically low coverage can still pass the gate as long as *newly written* code in the current change is well-covered; this is a materially different (and more forgiving, for a mature codebase) target than the flat `BUNDLE`-wide JaCoCo Maven rule in 10.3.3, which has no concept of "new" vs "old" code at all.

---

## 10.4 Phase 10 Summary Table

| Area | Key takeaway |
|---|---|
| MockitoExtension vs AemContextExtension | Sling Model injection, real QueryBuilder, OSGi activation, real JCR constraints, and servlet/filter request parsing are AemContext-only; most plain service-logic testing can use either |
| Cheat sheet | Keep as a living reference — update the pitfalls table whenever a new bug class is caught in code review |
| JaCoCo `prepare-agent` | Must run before tests, or no coverage data is collected at all |
| JaCoCo `check` goal | The actual enforcement mechanism — `report` alone doesn't fail builds |
| Threshold values | Fractional (0.0–1.0), not percentages — `<minimum>80</minimum>` is a common, build-breaking mistake |
| Report location | `target/site/jacoco/index.html` (human) and `jacoco.xml` (machine/SonarQube) |
| Cloud Manager quality gate | Often checks new-code coverage via SonarQube, a different (usually more forgiving) target than a flat local JaCoCo bundle rule |

---

**This completes the AEM JUnit Testing Guide, Phases 1–10.** Let me know if you'd like a consolidated single-document version combining all ten phases, a condensed "night-before-the-interview" summary sheet, or a deeper dive into any specific area (e.g., SonarQube new-code coverage configuration, or additional parameterized-test scenarios for the Property Listing validation matrix).
