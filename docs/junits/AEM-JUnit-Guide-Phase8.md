# AEM JUnit Testing Guide — Phase 8: Workflow Mocking

**Scenario used throughout:** a property-listing approval workflow — a custom `WorkflowProcess` (`PropertyApprovalProcessStep`) that validates a submitted property page, updates metadata, replicates (publishes) it on approval, and enriches listing data from a Content Fragment (e.g., a shared "Locality Info" fragment).

---

## 8.0 Concept — How AEM Workflows Fit Together

AEM's workflow engine (`com.adobe.granite.workflow`) models a business process as a graph of steps. Each step is either a built-in process (Activate Page, Send Email) or a custom Java class implementing `WorkflowProcess`, registered as an OSGi service and referenced by process-step-label in the workflow model.

The core objects you'll mock throughout this phase:
- **`WorkItem`** — represents one execution instance of the workflow moving through a step; carries the payload reference and step-level metadata.
- **`WorkflowSession`** — the workflow engine's session object; used to advance the workflow, complete/terminate it, and access the underlying JCR `Session`.
- **`WorkflowData`** — describes *what* the workflow is operating on (the payload — typically a JCR path or a `Node`) and carries its own separate metadata map.
- **`MetaDataMap`** (`args`) — the process-step's configured arguments (author-configured values from the workflow model, e.g., `PROCESS_ARGS`), distinct from the two metadata maps above.

Because `WorkflowProcess.execute(WorkItem, WorkflowSession, MetaDataMap)` is a plain synchronous method with no Sling/OSGi request-context magic, it is one of the most unit-testable pieces of AEM backend code — a fully mocked setup with no `AemContext` at all is usually sufficient and fastest.

---

## 8.1 WorkflowProcess Testing

### 8.1.1 Concept
`WorkflowProcess` is a single-method OSGi service interface:
```java
void execute(WorkItem workItem, WorkflowSession workflowSession, MetaDataMap args) throws WorkflowException;
```
Testing it is fundamentally a matter of: stub the three inputs to represent a specific scenario (a submitted property page, specific process-step arguments), call `execute(...)` directly, then verify the *side effects* — metadata written, replication triggered, exceptions thrown — since the method itself returns `void`.

### 8.1.2 Technicalities to know
- **The payload path is buried three calls deep.** `workItem.getWorkflowData()` returns `WorkflowData`; `.getPayload()` returns an `Object` (its actual runtime type depends on the payload type — `JCR_PATH` payloads return a `String`, `JCR_UUID` payloads return a UUID string, and some legacy flows use `Node` payloads directly). Production code almost always assumes a `JCR_PATH` payload and does `getPayload().toString()` — your test should stub `getPayload()` to return the path `String` directly (not wrap it further), and it's worth a dedicated test asserting behavior when `getPayloadType()` is unexpectedly not `JCR_PATH`.
- **Process arguments come from a separate object than either metadata map.** `args` (the third parameter to `execute`) is the step's *author-configured* arguments from the workflow model editor — typically a single semicolon- or newline-delimited string retrieved via `args.get("PROCESS_ARGS", String.class)`, which your process then parses itself (there's no structured multi-arg API — this is a common point of confusion for developers new to AEM workflows, since it looks like it should be a map of many keys but is conventionally a single delimited string).
- `MetaDataMap.get(key, Class)` follows the same "type-safe with default-friendly overload" convention as `ValueMap` (Phase 6.1) — prefer stubbing/asserting through the two-arg (with-default) form in production code to avoid null-handling bugs.
- `WorkflowException` is a checked exception — a step is expected to throw it (not swallow the failure) when it cannot complete meaningfully, since the workflow engine uses this to decide whether to retry, halt, or route to an error step depending on the workflow model. A process that catches all exceptions internally and never rethrows `WorkflowException` silently breaks the workflow model's error-handling routes — a good architectural point to raise in code review, and a good behavior to lock in with a test.

### 8.1.3 Mocking WorkItem, WorkflowSession, MetaDataMap, WorkflowData

```java
@ExtendWith(MockitoExtension.class)
class PropertyApprovalProcessStepTest {

    @Mock private WorkItem workItem;
    @Mock private WorkflowSession workflowSession;
    @Mock private MetaDataMap args;
    @Mock private WorkflowData workflowData;
    @Mock private MetaDataMap workflowDataMetaDataMap;

    private PropertyApprovalProcessStep processStep;

    @BeforeEach
    void setUp() {
        processStep = new PropertyApprovalProcessStep();
    }

    @Test
    void shouldMarkPropertyAsApprovedWhenValidationPasses() throws WorkflowException {
        // Arrange — payload path
        when(workItem.getWorkflowData()).thenReturn(workflowData);
        when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
        when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
        when(workflowData.getMetaDataMap()).thenReturn(workflowDataMetaDataMap);

        // Arrange — process args
        when(args.get("PROCESS_ARGS", String.class)).thenReturn("autoApprove=true");

        // Act
        processStep.execute(workItem, workflowSession, args);

        // Assert — side effect on WorkflowData's own metadata map
        verify(workflowDataMetaDataMap).put("approvalStatus", "APPROVED");
    }
}
```

### 8.1.4 Stubbing the payload path — workItem.getWorkflowData().getPayload().toString()

```java
@Test
void shouldReadPayloadPathCorrectlyForJcrPathPayload() throws WorkflowException {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
    when(workflowData.getMetaDataMap()).thenReturn(workflowDataMetaDataMap);
    when(args.get("PROCESS_ARGS", String.class)).thenReturn(null);

    processStep.execute(workItem, workflowSession, args);

    // Verify the path was correctly extracted and used downstream
    verify(workflowDataMetaDataMap).put(eq("processedPath"), eq("/content/properties/chennai/villa-101"));
}

@Test
void shouldThrowWorkflowExceptionWhenPayloadTypeIsUnexpected() {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_UUID"); // not the expected JCR_PATH

    assertThrows(WorkflowException.class,
        () -> processStep.execute(workItem, workflowSession, args));
}
```

### 8.1.5 Stubbing process arguments — args.get("PROCESS_ARGS", String.class)

```java
@Test
void shouldSkipAutoApprovalWhenProcessArgFlagIsFalse() throws WorkflowException {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
    when(workflowData.getMetaDataMap()).thenReturn(workflowDataMetaDataMap);
    when(args.get("PROCESS_ARGS", String.class)).thenReturn("autoApprove=false");

    processStep.execute(workItem, workflowSession, args);

    verify(workflowDataMetaDataMap, never()).put(eq("approvalStatus"), eq("APPROVED"));
    verify(workflowDataMetaDataMap).put("approvalStatus", "PENDING_MANUAL_REVIEW");
}

@Test
void shouldTreatMissingProcessArgsAsDefaultBehavior() throws WorkflowException {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
    when(workflowData.getMetaDataMap()).thenReturn(workflowDataMetaDataMap);
    when(args.get("PROCESS_ARGS", String.class)).thenReturn(null); // author never configured the step

    processStep.execute(workItem, workflowSession, args);

    verify(workflowDataMetaDataMap).put("approvalStatus", "PENDING_MANUAL_REVIEW");
}
```

### 8.1.6 Calling process.execute(workItem, workflowSession, args) directly

This is shown throughout 8.1.3–8.1.5 — worth calling out explicitly because it's the entire point of testing `WorkflowProcess` implementations: no `AemContext`, no OSGi component activation lifecycle, no Sling request — just a plain method call against mocked collaborators. This is why workflow-process unit tests are typically fast and cheap despite workflows being a "heavyweight" AEM feature at runtime.

### 8.1.7 Verifying WorkflowData.getMetaDataMap().put() was called with expected key/value

```java
@Test
void shouldRecordApprovalTimestampInWorkflowMetadata() throws WorkflowException {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
    when(workflowData.getMetaDataMap()).thenReturn(workflowDataMetaDataMap);
    when(args.get("PROCESS_ARGS", String.class)).thenReturn("autoApprove=true");

    processStep.execute(workItem, workflowSession, args);

    // ArgumentCaptor lets us assert on the actual value passed, not just "any Calendar"
    ArgumentCaptor<Object> valueCaptor = ArgumentCaptor.forClass(Object.class);
    verify(workflowDataMetaDataMap).put(eq("approvedAt"), valueCaptor.capture());
    assertTrue(valueCaptor.getValue() instanceof Calendar);
}
```

### 8.1.8 Testing WorkflowException is thrown when expected

```java
@Test
void shouldThrowWorkflowExceptionWhenPayloadPathIsBlank() {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn(""); // malformed payload

    WorkflowException ex = assertThrows(WorkflowException.class,
        () -> processStep.execute(workItem, workflowSession, args));

    assertTrue(ex.getMessage().contains("payload"));
}
```

---

## 8.2 WorkflowSession and WorkItem Mocking

### 8.2.1 Concept
Beyond the single-step `execute()` call, some processes need the broader `WorkflowSession` — to look up the parent `Workflow`, to explicitly `complete()`/`terminate()` a work item under certain conditions, or (most commonly) to reach the underlying JCR `Session` for direct node manipulation the Sling APIs don't conveniently expose.

### 8.2.2 Technicalities to know
- **`workItem.getMetaDataMap()` vs `workItem.getWorkflowData().getMetaDataMap()` — these are two genuinely different metadata stores, and mixing them up is the single most common bug in workflow-process code:**
  - `workItem.getMetaDataMap()` is **step-scoped** metadata — data relevant only to this specific work item at this specific step (e.g., "did this step already run once due to a retry"). It does **not** persist forward to subsequent steps in the workflow.
  - `workItem.getWorkflowData().getMetaDataMap()` is **workflow-scoped** metadata — attached to the `WorkflowData` object that travels with the payload through the *entire* workflow instance, visible to every subsequent step. Approval status, audit timestamps, and anything a later step needs to read should go here, not on `workItem.getMetaDataMap()`.
  - A test suite is a good place to lock in which one your process is *supposed* to use — `verify(workItem, never()).getMetaDataMap()` alongside asserting the correct write on `workflowData.getMetaDataMap()` catches a regression where someone "fixes a null pointer" by swapping to the wrong map.
- `workflowSession.getSession()` returns the JCR `Session` the workflow engine is operating under — typically a system/service session with elevated privileges, not the requesting user's session. Code that assumes it's the same session as `resourceResolver.adaptTo(Session.class)` from the originating request is a subtle bug source (different privilege scope, different `save()` timing).
- `workflowSession.getWorkflow(workItem)` returns the parent `Workflow` object — useful for reading top-level workflow metadata (initiator, start time) as opposed to the step-scoped `WorkItem`.

### 8.2.3 Full mock setup for a multi-step workflow test

```java
@ExtendWith(MockitoExtension.class)
class PropertyPublishProcessStepTest {

    @Mock private WorkItem workItem;
    @Mock private WorkflowSession workflowSession;
    @Mock private MetaDataMap args;
    @Mock private WorkflowData workflowData;
    @Mock private MetaDataMap workflowScopedMetaData;
    @Mock private MetaDataMap stepScopedMetaData;
    @Mock private Session jcrSession;

    private PropertyPublishProcessStep processStep;

    @BeforeEach
    void setUp() {
        processStep = new PropertyPublishProcessStep();
    }

    @Test
    void shouldWriteAuditDataToWorkflowScopedMetadataNotStepScoped() throws WorkflowException {
        when(workItem.getWorkflowData()).thenReturn(workflowData);
        when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
        when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
        when(workflowData.getMetaDataMap()).thenReturn(workflowScopedMetaData);
        when(workflowSession.getSession()).thenReturn(jcrSession);

        processStep.execute(workItem, workflowSession, args);

        // Correct target: workflow-scoped map, visible to later steps
        verify(workflowScopedMetaData).put(eq("publishedBy"), any());
        // Explicitly assert the step-scoped map was untouched for this piece of data
        verify(workItem, never()).getMetaDataMap();
    }
}
```

### 8.2.4 workItem.getMetaDataMap() vs workItem.getWorkflowData().getMetaDataMap()

```java
@Test
void shouldUseStepScopedMapOnlyForRetryTracking() throws WorkflowException {
    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
    when(workflowData.getMetaDataMap()).thenReturn(workflowScopedMetaData);
    when(workItem.getMetaDataMap()).thenReturn(stepScopedMetaData);
    when(stepScopedMetaData.get("retryCount", 0)).thenReturn(1);
    when(workflowSession.getSession()).thenReturn(jcrSession);

    processStep.execute(workItem, workflowSession, args);

    // Retry counter is step-local — must not leak into workflow-scoped metadata
    verify(stepScopedMetaData).put("retryCount", 2);
    verify(workflowScopedMetaData, never()).put(eq("retryCount"), any());
}
```

### 8.2.5 Stubbing workflowSession.getSession() to return a mock JCR Session

```java
@Test
void shouldPersistPublishFlagDirectlyViaJcrSession() throws Exception {
    Node propertyNode = mock(Node.class);

    when(workItem.getWorkflowData()).thenReturn(workflowData);
    when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
    when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
    when(workflowData.getMetaDataMap()).thenReturn(workflowScopedMetaData);
    when(workflowSession.getSession()).thenReturn(jcrSession);
    when(jcrSession.getNode("/content/properties/chennai/villa-101/jcr:content")).thenReturn(propertyNode);

    processStep.execute(workItem, workflowSession, args);

    verify(propertyNode).setProperty("publishedFlag", true);
    verify(jcrSession).save();
}
```

---

## 8.3 Replicator Mocking (in Workflow Steps)

### 8.3.1 Concept
`Replicator` (`com.day.cq.replication`) is AEM's publish/activate API — it's how a workflow step (or any backend code) triggers content replication to publish instances programmatically, as opposed to an author manually clicking "Publish" in the UI. `replicate(Session, ReplicationActionType, String path)` is the core method; `ReplicationActionType.ACTIVATE` publishes, `DEACTIVATE` unpublishes, `DELETE` removes from publish.

### 8.3.2 Technicalities to know
- `Replicator` is normally injected as an OSGi `@Reference` into the `WorkflowProcess` component — in a pure-Mockito test with `@InjectMocks`, make sure the field name/type matches so Mockito's field-injection can wire the mock in (or use a constructor if your class supports it, which is more explicit and less fragile than field injection).
- `replicate(...)` throws a checked `ReplicationException` — this is the *expected* failure mode when a publish instance is unreachable, an agent is misconfigured, or the content violates a replication constraint. Workflow steps that call `replicate()` should have an explicit catch block that transitions the workflow/content into a `FAILED` state rather than letting the exception propagate uncaught and leave the workflow instance stuck.
- Verifying `replicator.replicate(session, ACTIVATE, path)` should assert the **exact** session and path used — a common bug is replicating the wrong path (e.g., the `jcr:content` child instead of the page path, or a stale path captured before an earlier rename step ran).

### 8.3.3 mock(Replicator.class) and injecting into the workflow process

```java
@ExtendWith(MockitoExtension.class)
class PropertyReplicationProcessStepTest {

    @Mock private WorkItem workItem;
    @Mock private WorkflowSession workflowSession;
    @Mock private MetaDataMap args;
    @Mock private WorkflowData workflowData;
    @Mock private MetaDataMap workflowScopedMetaData;
    @Mock private Session jcrSession;
    @Mock private Replicator replicator;

    @InjectMocks
    private PropertyReplicationProcessStep processStep; // has a @Reference Replicator field

    @BeforeEach
    void setUp() {
        when(workItem.getWorkflowData()).thenReturn(workflowData);
        when(workflowData.getPayloadType()).thenReturn("JCR_PATH");
        when(workflowData.getPayload()).thenReturn("/content/properties/chennai/villa-101");
        when(workflowData.getMetaDataMap()).thenReturn(workflowScopedMetaData);
        when(workflowSession.getSession()).thenReturn(jcrSession);
    }
```

### 8.3.4 Verifying replicator.replicate(session, ACTIVATE, path) was called

```java
    @Test
    void shouldActivatePropertyPageOnApproval() throws Exception {
        processStep.execute(workItem, workflowSession, args);

        verify(replicator).replicate(jcrSession, ReplicationActionType.ACTIVATE,
            "/content/properties/chennai/villa-101");
    }
```

### 8.3.5 Testing the exception path — replicator throws ReplicationException

```java
    @Test
    void shouldMarkMetadataFailedWhenReplicationThrows() throws Exception {
        doThrow(new ReplicationException("agent unreachable"))
            .when(replicator).replicate(any(Session.class), eq(ReplicationActionType.ACTIVATE), anyString());

        // The process is expected to catch ReplicationException internally and record failure,
        // not propagate it as an uncaught RuntimeException (that would strand the workflow instance)
        assertDoesNotThrow(() -> processStep.execute(workItem, workflowSession, args));

        verify(workflowScopedMetaData).put("replicationStatus", "FAILED");
    }

    @Test
    void shouldWrapReplicationFailureAsWorkflowExceptionWhenConfiguredToHaltOnFailure() throws Exception {
        when(args.get("PROCESS_ARGS", String.class)).thenReturn("haltOnFailure=true");
        doThrow(new ReplicationException("agent unreachable"))
            .when(replicator).replicate(any(Session.class), eq(ReplicationActionType.ACTIVATE), anyString());

        assertThrows(WorkflowException.class,
            () -> processStep.execute(workItem, workflowSession, args));
    }
```

### 8.3.6 Verifying WorkflowData metadata is set to "FAILED" when replication throws

```java
    @Test
    void shouldNotAdvanceApprovalStatusWhenReplicationFails() throws Exception {
        doThrow(new ReplicationException("agent unreachable"))
            .when(replicator).replicate(any(Session.class), eq(ReplicationActionType.ACTIVATE), anyString());

        processStep.execute(workItem, workflowSession, args);

        verify(workflowScopedMetaData).put("replicationStatus", "FAILED");
        verify(workflowScopedMetaData, never()).put(eq("approvalStatus"), eq("PUBLISHED"));
    }
}
```

**Edge case worth testing:** a `Replicator` call that succeeds but replicates the **wrong** action type (e.g., `DEACTIVATE` instead of `ACTIVATE` due to a workflow-model misconfiguration bug) will not throw — `verify(replicator).replicate(eq(jcrSession), eq(ReplicationActionType.ACTIVATE), anyString())` (an exact-match verify, not `any()`) is what actually catches this class of bug; a loose `verify(replicator).replicate(any(), any(), any())` would pass even with the wrong action type.

---

## 8.4 ContentFragment Mocking

### 8.4.1 Concept
Content Fragments (`com.adobe.cq.dam.cfm`) are structured, reusable content stored in the DAM — e.g., a shared "Locality Info" fragment (school ratings, connectivity notes, average price trends) referenced by multiple property listing pages so the data is maintained once and reused everywhere. A `ContentFragment` exposes named `ContentElement`s (its structured fields), each of which can be read as a string, a typed value, or a multi-value array depending on the underlying data type defined in the fragment model.

### 8.4.2 Technicalities to know
- `fragment.getElement(name)` returns `null` — not an exception — for an element name that doesn't exist on the fragment (e.g., the fragment model changed and dropped a field, or a caller has a typo). This mirrors the `TagManager.resolve()` null-return pattern from Phase 6.3 and deserves the same defensive-null-check discipline in production code and a matching test.
- `ContentElement.getContent()` returns the element's value as a `String` regardless of the underlying field type — appropriate for single-line/multi-line text fields. For structured/typed fields (number, boolean, date, tag references, multi-value), use `getValue(Class)` instead, which performs the actual type coercion; calling `getContent()` on a numeric field gives you its string representation, not a validated number.
- `getValue(Integer.class)` / `getValue(String[].class)` follow the DAM API's typed-adapter convention — an incompatible type request (e.g., `getValue(Integer.class)` on a genuinely text field) can return `null` or throw depending on implementation/version; a defensive test asserting your service's fallback behavior for a type mismatch is worthwhile, not just the happy path.
- Multi-value fragment fields (e.g., `tags`) come back as arrays (`String[]`), the same array-not-List gotcha flagged for multi-value JCR properties in Phase 6.1 — consistent with the rest of the AEM API surface once you notice the pattern.

### 8.4.3 mock(ContentFragment.class) and mock(ContentElement.class)

```java
@ExtendWith(MockitoExtension.class)
class LocalityInfoEnrichmentServiceTest {

    @Mock private ContentFragment localityFragment;
    @Mock private ContentElement titleElement;
    @Mock private ContentElement scoreElement;
    @Mock private ContentElement tagsElement;

    @InjectMocks
    private LocalityInfoEnrichmentService enrichmentService;
```

### 8.4.4 Stubbing fragment.getElement("title").getContent()

```java
    @Test
    void shouldReadLocalityTitleFromContentFragment() {
        when(localityFragment.getElement("title")).thenReturn(titleElement);
        when(titleElement.getContent()).thenReturn("Adyar, Chennai");

        String title = enrichmentService.getLocalityTitle(localityFragment);

        assertEquals("Adyar, Chennai", title);
    }
```

### 8.4.5 Stubbing fragment.getElement("score").getValue(Integer.class) for numeric fields

```java
    @Test
    void shouldReadConnectivityScoreAsTypedInteger() {
        when(localityFragment.getElement("score")).thenReturn(scoreElement);
        when(scoreElement.getValue(Integer.class)).thenReturn(87);

        int connectivityScore = enrichmentService.getConnectivityScore(localityFragment);

        assertEquals(87, connectivityScore);
    }

    @Test
    void shouldDefaultToZeroWhenScoreElementReturnsNullValue() {
        when(localityFragment.getElement("score")).thenReturn(scoreElement);
        when(scoreElement.getValue(Integer.class)).thenReturn(null);

        int connectivityScore = enrichmentService.getConnectivityScore(localityFragment);

        assertEquals(0, connectivityScore);
    }
```

### 8.4.6 Stubbing fragment.getElement("tags").getValue(String[].class) for multi-value

```java
    @Test
    void shouldReadHighlightTagsAsStringArray() {
        when(localityFragment.getElement("tags")).thenReturn(tagsElement);
        when(tagsElement.getValue(String[].class))
            .thenReturn(new String[]{"metro-connectivity", "top-rated-schools"});

        String[] highlightTags = enrichmentService.getHighlightTags(localityFragment);

        assertArrayEquals(new String[]{"metro-connectivity", "top-rated-schools"}, highlightTags);
    }
```

### 8.4.7 Testing null element — fragment.getElement("nonExistent") returns null

```java
    @Test
    void shouldReturnEmptyStringWhenTitleElementDoesNotExistOnFragment() {
        when(localityFragment.getElement("title")).thenReturn(null);

        String title = enrichmentService.getLocalityTitle(localityFragment);

        assertEquals("", title); // service defensively falls back rather than throwing NPE
    }

    @Test
    void shouldReturnEmptyArrayWhenTagsElementMissing() {
        when(localityFragment.getElement("tags")).thenReturn(null);

        String[] highlightTags = enrichmentService.getHighlightTags(localityFragment);

        assertArrayEquals(new String[0], highlightTags);
    }
}
```

**Edge case worth testing:** a fragment whose model changed to rename `score` → `connectivityScore` mid-project. Any code (and test) still stubbing/reading the old element name should fail loudly in a test long before it fails silently in production as "connectivity score always shows 0."

---

## 8.5 Phase 8 Summary Table

| API | What it represents | Key gotcha |
|---|---|---|
| `WorkItem` | One execution instance of a workflow at a given step | `getMetaDataMap()` is step-scoped, doesn't persist to later steps |
| `WorkflowData` | The payload + workflow-wide metadata that travels with it | `getMetaDataMap()` here is workflow-scoped — the one later steps read |
| `MetaDataMap args` | Author-configured process-step arguments | Conventionally a single delimited string via `PROCESS_ARGS`, not a rich map |
| `WorkflowSession` | Engine-level session for advancing/inspecting the workflow | `getSession()` returns a system/service JCR session, not the request session |
| `Replicator` | Publish/unpublish trigger | Verify exact `ReplicationActionType` and path, not `any()` — wrong-action bugs don't throw |
| `ContentFragment` / `ContentElement` | Structured reusable DAM content | `getElement(name)` returns `null` for missing fields; `getContent()` vs `getValue(Class)` — string vs typed read |

---

**Next: Phase 9 — Advanced Mockito** (`spy()` for partial mocking, `ArgumentCaptor` deep-dive — including the `PredicateGroup` capture flagged in Phase 7 — `MockedStatic` for static utility methods like `ResourceUtil`/`PageUtil`, reflection-based testing of private fields/methods, and JUnit 5 parameterized tests for property-validation matrices).

Ready for Phase 9 whenever you are.
