# AEM JUnit Testing — Phase 5: Schedulers, Jobs & Listeners

---

### 5.1 Scheduler Testing

#### Why Schedulers Are the Easiest Thing to Test

A Sling Scheduler is just a `Runnable`. The OSGi Scheduler framework calls `run()` on a cron schedule — but in tests, you call `run()` directly. No cron, no timer, no async behaviour. The test is synchronous and deterministic.

#### The Scheduler Under Test

```java
@Component(service = Runnable.class)
@Designate(ocd = SimpleScheduledTask.Config.class)
public class SimpleScheduledTask implements Runnable {

    @ObjectClassDefinition(name = "Simple Scheduled Task")
    public @interface Config {
        @AttributeDefinition(name = "Scheduler Expression")
        String scheduler_expression() default "0/30 * * * * ?";

        @AttributeDefinition(name = "Enabled")
        boolean enabled() default false;

        @AttributeDefinition(name = "Custom Parameter")
        String myParameter() default "";
    }

    private static final Logger LOG = LoggerFactory.getLogger(SimpleScheduledTask.class);

    private boolean enabled;
    private String myParameter;

    @Activate
    protected void activate(Config config) {
        this.enabled     = config.enabled();
        this.myParameter = config.myParameter();
    }

    @Override
    public void run() {
        if (!enabled) {
            LOG.debug("Scheduler is disabled — skipping execution");
            return;
        }
        LOG.debug("Scheduler executing with parameter: {}", myParameter);
        doWork(myParameter);
    }

    protected void doWork(String parameter) {
        // Business logic here — protected so tests can spy on it
        LOG.info("Doing work with: {}", parameter);
    }
}
```

#### Full Scheduler Test Class

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class SimpleScheduledTaskTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Test
    void run_whenEnabled_executesBusinessLogic() {
        Map<String, Object> config = Map.of(
            "enabled",     true,
            "myParameter", "test-value",
            "scheduler_expression", "0/30 * * * * ?"
        );

        SimpleScheduledTask task = ctx.registerInjectActivateService(
            new SimpleScheduledTask(), config);

        // Just call run() directly — no timer, no cron, instant
        // If run() throws any exception, the test fails automatically
        assertDoesNotThrow(task::run);
    }

    @Test
    void run_whenDisabled_doesNotExecuteBusinessLogic() {
        // Use Spy to verify doWork() is NOT called when disabled
        SimpleScheduledTask realTask = new SimpleScheduledTask();
        SimpleScheduledTask spyTask  = spy(realTask);

        Map<String, Object> config = Map.of(
            "enabled",     false,   // disabled
            "myParameter", "value"
        );
        ctx.registerInjectActivateService(spyTask, config);

        spyTask.run();

        // Verify doWork() was never called because enabled=false
        verify(spyTask, never()).doWork(anyString());
    }

    @Test
    void run_whenEnabled_callsDoWorkWithCorrectParameter() {
        SimpleScheduledTask realTask = new SimpleScheduledTask();
        SimpleScheduledTask spyTask  = spy(realTask);

        Map<String, Object> config = Map.of(
            "enabled",     true,
            "myParameter", "production-param"
        );
        ctx.registerInjectActivateService(spyTask, config);

        spyTask.run();

        // Verify doWork() was called exactly once with the configured parameter
        verify(spyTask, times(1)).doWork("production-param");
    }

    @Test
    void activate_storesConfigValues() throws Exception {
        Map<String, Object> config = Map.of(
            "enabled",     true,
            "myParameter", "my-custom-param"
        );

        SimpleScheduledTask task = ctx.registerInjectActivateService(
            new SimpleScheduledTask(), config);

        // Verify private fields via reflection
        Field enabledField = SimpleScheduledTask.class.getDeclaredField("enabled");
        enabledField.setAccessible(true);
        assertTrue((boolean) enabledField.get(task));

        Field paramField = SimpleScheduledTask.class.getDeclaredField("myParameter");
        paramField.setAccessible(true);
        assertEquals("my-custom-param", paramField.get(task));
    }
}
```

#### Testing the Cluster-Safe Scheduler

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class AuthorUpdateClusterSchedulerTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private ResourceResolverFactory resourceResolverFactory;

    @Mock
    private ResourceResolver serviceResolver;

    @BeforeEach
    void setUp() throws LoginException {
        when(resourceResolverFactory.getServiceResourceResolver(any()))
            .thenReturn(serviceResolver);
        when(serviceResolver.isLive()).thenReturn(true);
        // Adapt to Session returns null in mock context — handled by the scheduler
        when(serviceResolver.adaptTo(Session.class)).thenReturn(null);

        ctx.registerService(ResourceResolverFactory.class, resourceResolverFactory);
    }

    @Test
    void run_whenEnabled_opensAndClosesServiceResolver() throws Exception {
        Map<String, Object> config = Map.of(
            "enabled", true,
            "cronExpression", "0 0 2 * * ?"
        );

        AuthorUpdateClusterScheduler scheduler = ctx.registerInjectActivateService(
            new AuthorUpdateClusterScheduler(), config);

        scheduler.run();

        // Verify service resolver was opened
        verify(resourceResolverFactory).getServiceResourceResolver(any());
        // Verify it was closed — critical to prevent session leak
        verify(serviceResolver).close();
    }

    @Test
    void run_whenResolverFactoryThrows_doesNotPropagateException() throws Exception {
        when(resourceResolverFactory.getServiceResourceResolver(any()))
            .thenThrow(new LoginException("No permission"));

        Map<String, Object> config = Map.of("enabled", true, "cronExpression", "0 0 2 * * ?");
        AuthorUpdateClusterScheduler scheduler = ctx.registerInjectActivateService(
            new AuthorUpdateClusterScheduler(), config);

        // Scheduler run() must never throw — it runs on a background thread
        // An uncaught exception kills the scheduler thread permanently
        assertDoesNotThrow(scheduler::run,
            "Scheduler must handle LoginException without propagating");
    }
}
```

---

### 5.2 Sling Job Consumer Testing

#### Key Concept

A Sling Job Consumer processes jobs from a topic. In tests:
- Create a mock `Job` object
- Stub its `getProperty()` calls to return test values
- Call `consumer.process(job)` directly
- Assert the `JobResult` and verify downstream service calls

#### The Job Consumer Under Test

```java
@Component(service = JobConsumer.class,
    property = { JobConsumer.PROPERTY_TOPICS + "=sibi-aem-one/author/update" })
public class AuthorUpdateJobConsumer implements JobConsumer {

    private static final Logger LOG = LoggerFactory.getLogger(AuthorUpdateJobConsumer.class);

    @Reference
    private ExternalApiService externalApiService;

    @Override
    public JobResult process(Job job) {
        String payloadPath = job.getProperty("payloadPath", String.class);

        if (StringUtils.isBlank(payloadPath)) {
            LOG.error("Job missing required payloadPath property");
            return JobResult.CANCEL; // permanent failure — don't retry
        }

        try {
            LOG.info("Processing author update for: {}", payloadPath);
            String result = externalApiService.fetchProductData(payloadPath);
            if (result == null) {
                LOG.warn("No data returned for path: {}", payloadPath);
                return JobResult.FAILED; // transient — retry
            }
            LOG.info("Successfully processed: {}", payloadPath);
            return JobResult.OK;
        } catch (Exception e) {
            LOG.error("Error processing job for {}: {}", payloadPath, e.getMessage(), e);
            return JobResult.FAILED; // transient — retry
        }
    }
}
```

#### Full Job Consumer Test Class

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class AuthorUpdateJobConsumerTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private ExternalApiService externalApiService;

    @Mock
    private Job job;  // Mock the Sling Job object

    private AuthorUpdateJobConsumer consumer;

    @BeforeEach
    void setUp() {
        ctx.registerService(ExternalApiService.class, externalApiService);
        consumer = ctx.registerInjectActivateService(new AuthorUpdateJobConsumer());
    }

    // ── Happy path ─────────────────────────────────────────────────────────

    @Test
    void process_whenValidPayloadAndServiceSucceeds_returnsOK() {
        // Stub the Job to return a valid payloadPath property
        when(job.getProperty("payloadPath", String.class))
            .thenReturn("/content/mysite/en/products/shirt");

        // Stub the service to return data
        when(externalApiService.fetchProductData("/content/mysite/en/products/shirt"))
            .thenReturn("{ \"sku\": \"SHIRT-001\" }");

        JobResult result = consumer.process(job);

        assertEquals(JobResult.OK, result,
            "Should return OK when processing succeeds");

        // Verify the service was called with the correct path
        verify(externalApiService).fetchProductData("/content/mysite/en/products/shirt");
    }

    // ── Missing payload path ────────────────────────────────────────────────

    @Test
    void process_whenPayloadPathMissing_returnsCancel() {
        // Job has no payloadPath property — returns null
        when(job.getProperty("payloadPath", String.class)).thenReturn(null);

        JobResult result = consumer.process(job);

        // CANCEL = permanent failure, no retry — correct for missing required data
        assertEquals(JobResult.CANCEL, result,
            "Missing payloadPath should result in CANCEL (no retry)");

        // Service should NOT be called with a null path
        verifyNoInteractions(externalApiService);
    }

    @Test
    void process_whenPayloadPathBlank_returnsCancel() {
        when(job.getProperty("payloadPath", String.class)).thenReturn("   ");

        JobResult result = consumer.process(job);

        assertEquals(JobResult.CANCEL, result);
        verifyNoInteractions(externalApiService);
    }

    // ── Service returns null ────────────────────────────────────────────────

    @Test
    void process_whenServiceReturnsNull_returnsFailed() {
        when(job.getProperty("payloadPath", String.class))
            .thenReturn("/content/product");
        // Service returns null — transient issue, should retry
        when(externalApiService.fetchProductData(anyString())).thenReturn(null);

        JobResult result = consumer.process(job);

        // FAILED = transient, Sling will retry with backoff
        assertEquals(JobResult.FAILED, result,
            "Null service response should result in FAILED (retry)");
    }

    // ── Service throws exception ────────────────────────────────────────────

    @Test
    void process_whenServiceThrowsException_returnsFailed() {
        when(job.getProperty("payloadPath", String.class))
            .thenReturn("/content/product");
        when(externalApiService.fetchProductData(anyString()))
            .thenThrow(new RuntimeException("Connection refused"));

        JobResult result = consumer.process(job);

        assertEquals(JobResult.FAILED, result,
            "Runtime exception should result in FAILED (retry)");
    }

    // ── JobResult semantics ─────────────────────────────────────────────────

    @Test
    void process_jobResultSemantics_areCorrectlyApplied() {
        // This test documents the semantic meaning of each JobResult value
        // and verifies your consumer uses the right one for each scenario:
        //
        // JobResult.OK     → success, job removed from queue
        // JobResult.FAILED → transient failure, Sling retries with backoff
        // JobResult.CANCEL → permanent failure, job removed, no retry
        //
        // Correct mapping:
        //   Missing required data  → CANCEL (retrying won't fix it)
        //   Service down/timeout   → FAILED (retrying may fix it)
        //   Success                → OK

        // Verify CANCEL for missing data (non-retryable)
        when(job.getProperty("payloadPath", String.class)).thenReturn(null);
        assertEquals(JobResult.CANCEL, consumer.process(job));

        // Verify FAILED for service error (retryable)
        when(job.getProperty("payloadPath", String.class)).thenReturn("/path");
        when(externalApiService.fetchProductData(anyString()))
            .thenThrow(new RuntimeException("Timeout"));
        assertEquals(JobResult.FAILED, consumer.process(job));
    }
}
```

---

### 5.3 Sling Job Producer Testing

#### The Job Producer Under Test

```java
@Component(service = AuthorUpdateJobProducer.class)
public class AuthorUpdateJobProducer {

    private static final String JOB_TOPIC = "sibi-aem-one/author/update";

    @Reference
    private JobManager jobManager;

    public void triggerJob(String payloadPath) {
        if (StringUtils.isBlank(payloadPath)) {
            return;
        }
        Map<String, Object> props = new HashMap<>();
        props.put("payloadPath", payloadPath);
        props.put("triggeredAt", System.currentTimeMillis());
        jobManager.addJob(JOB_TOPIC, props);
    }
}
```

#### Job Producer Test — Using ArgumentCaptor

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class AuthorUpdateJobProducerTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private JobManager jobManager;

    private AuthorUpdateJobProducer producer;

    @BeforeEach
    void setUp() {
        ctx.registerService(JobManager.class, jobManager);
        producer = ctx.registerInjectActivateService(new AuthorUpdateJobProducer());
    }

    @Test
    void triggerJob_whenValidPath_addsJobWithCorrectTopic() {
        producer.triggerJob("/content/product/shirt");

        // Verify jobManager.addJob was called with the correct topic
        verify(jobManager).addJob(
            eq("sibi-aem-one/author/update"),
            anyMap()
        );
    }

    @Test
    void triggerJob_whenValidPath_includesPayloadPathInProperties() {
        // Use ArgumentCaptor to inspect the properties map
        ArgumentCaptor<Map<String, Object>> propsCaptor =
            ArgumentCaptor.forClass(Map.class);

        producer.triggerJob("/content/product/shirt");

        verify(jobManager).addJob(anyString(), propsCaptor.capture());

        Map<String, Object> capturedProps = propsCaptor.getValue();
        assertEquals("/content/product/shirt", capturedProps.get("payloadPath"),
            "payloadPath must be set in job properties");
    }

    @Test
    void triggerJob_whenValidPath_includesTimestampInProperties() {
        ArgumentCaptor<Map<String, Object>> propsCaptor =
            ArgumentCaptor.forClass(Map.class);

        long before = System.currentTimeMillis();
        producer.triggerJob("/content/product");
        long after = System.currentTimeMillis();

        verify(jobManager).addJob(anyString(), propsCaptor.capture());

        Map<String, Object> props = propsCaptor.getValue();
        assertTrue(props.containsKey("triggeredAt"),
            "triggeredAt timestamp must be in job properties");

        long timestamp = (long) props.get("triggeredAt");
        assertTrue(timestamp >= before && timestamp <= after,
            "triggeredAt should be close to the current time");
    }

    @Test
    void triggerJob_whenPathIsBlank_doesNotAddJob() {
        producer.triggerJob("   "); // blank

        verifyNoInteractions(jobManager);
    }

    @Test
    void triggerJob_whenPathIsNull_doesNotAddJob() {
        producer.triggerJob(null);

        verifyNoInteractions(jobManager);
    }

    @Test
    void triggerJob_calledMultipleTimes_addsMultipleJobs() {
        producer.triggerJob("/content/page1");
        producer.triggerJob("/content/page2");
        producer.triggerJob("/content/page3");

        // Verify jobManager.addJob was called 3 times total
        verify(jobManager, times(3)).addJob(anyString(), anyMap());
    }
}
```

---

### 5.4 ResourceChangeListener Testing

#### The Listener Under Test

```java
@Component(service = ResourceChangeListener.class, immediate = true, property = {
    ResourceChangeListener.PATHS   + "=/content/sibi-aem-one",
    ResourceChangeListener.CHANGES + "=ADDED",
    ResourceChangeListener.CHANGES + "=CHANGED"
})
public class ContentChangeListener implements ResourceChangeListener {

    @Reference
    private JobManager jobManager;

    private static final String JOB_TOPIC = "sibi-aem-one/contentchangelistener/job";

    @Override
    public void onChange(List<ResourceChange> changes) {
        for (ResourceChange change : changes) {
            // Skip jcr:content sub-property noise
            if (change.getPath().contains("jcr:content/")) {
                continue;
            }
            // Skip REMOVED events — only process additions and changes
            if (change.getType() == ResourceChange.ChangeType.REMOVED) {
                continue;
            }
            // Skip external cluster changes — let the originating node handle it
            if (change.isExternal()) {
                continue;
            }

            Map<String, Object> props = new HashMap<>();
            props.put("path",       change.getPath());
            props.put("changeType", change.getType().toString());
            jobManager.addJob(JOB_TOPIC, props);
        }
    }
}
```

#### How to Construct a ResourceChange for Tests

`ResourceChange` has no public constructor — use `MockResourceChange` or Mockito:

```java
// Option A: mock with Mockito (most flexible)
ResourceChange change = mock(ResourceChange.class);
when(change.getPath()).thenReturn("/content/mysite/en/page");
when(change.getType()).thenReturn(ResourceChange.ChangeType.CHANGED);
when(change.isExternal()).thenReturn(false);

// Option B: use Sling Mock's builder (cleaner)
// Available in sling-mock 3.x+
ResourceChange change = new MockResourceChange.Builder()
    .path("/content/mysite/en/page")
    .changeType(ResourceChange.ChangeType.CHANGED)
    .isExternal(false)
    .build();
```

#### Full Listener Test Class

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class ContentChangeListenerTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private JobManager jobManager;

    private ContentChangeListener listener;

    @BeforeEach
    void setUp() {
        ctx.registerService(JobManager.class, jobManager);
        listener = ctx.registerInjectActivateService(new ContentChangeListener());
    }

    // ── Helper to create mock ResourceChange ───────────────────────────────

    private ResourceChange mockChange(String path,
                                       ResourceChange.ChangeType type,
                                       boolean external) {
        ResourceChange change = mock(ResourceChange.class);
        when(change.getPath()).thenReturn(path);
        when(change.getType()).thenReturn(type);
        when(change.isExternal()).thenReturn(external);
        return change;
    }

    // ── Happy path ─────────────────────────────────────────────────────────

    @Test
    void onChange_whenLocalChangedEvent_enqueuesJob() {
        ResourceChange change = mockChange(
            "/content/mysite/en/page",
            ResourceChange.ChangeType.CHANGED,
            false  // local change
        );

        listener.onChange(List.of(change));

        verify(jobManager, times(1)).addJob(
            eq("sibi-aem-one/contentchangelistener/job"),
            anyMap()
        );
    }

    @Test
    void onChange_whenLocalChangedEvent_includesPathInJobProperties() {
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);

        ResourceChange change = mockChange(
            "/content/mysite/en/page",
            ResourceChange.ChangeType.CHANGED,
            false
        );

        listener.onChange(List.of(change));

        verify(jobManager).addJob(anyString(), captor.capture());
        assertEquals("/content/mysite/en/page", captor.getValue().get("path"));
        assertEquals("CHANGED", captor.getValue().get("changeType"));
    }

    // ── Filtering: jcr:content sub-property noise ──────────────────────────

    @Test
    void onChange_whenPathContainsJcrContentSubPath_skipsEvent() {
        // This is sub-property noise — jcr:content/ in the MIDDLE of path
        ResourceChange change = mockChange(
            "/content/mysite/en/page/jcr:content/text",  // sub-property
            ResourceChange.ChangeType.CHANGED,
            false
        );

        listener.onChange(List.of(change));

        verifyNoInteractions(jobManager);
    }

    // ── Filtering: REMOVED events ──────────────────────────────────────────

    @Test
    void onChange_whenRemovedEvent_skipsEvent() {
        ResourceChange change = mockChange(
            "/content/mysite/en/page",
            ResourceChange.ChangeType.REMOVED,  // REMOVED should be ignored
            false
        );

        listener.onChange(List.of(change));

        verifyNoInteractions(jobManager);
    }

    // ── Filtering: external cluster changes ────────────────────────────────

    @Test
    void onChange_whenExternalChange_skipsEvent() {
        ResourceChange change = mockChange(
            "/content/mysite/en/page",
            ResourceChange.ChangeType.CHANGED,
            true  // external — from another cluster node
        );

        listener.onChange(List.of(change));

        verifyNoInteractions(jobManager);
    }

    // ── Multiple changes in one batch ──────────────────────────────────────

    @Test
    void onChange_whenMixedBatch_processesOnlyEligibleChanges() {
        List<ResourceChange> changes = List.of(
            mockChange("/content/mysite/en/page1",              // eligible
                ResourceChange.ChangeType.CHANGED, false),
            mockChange("/content/mysite/en/page2/jcr:content/prop", // sub-property — skip
                ResourceChange.ChangeType.CHANGED, false),
            mockChange("/content/mysite/en/page3",              // eligible
                ResourceChange.ChangeType.ADDED, false),
            mockChange("/content/mysite/en/page4",              // REMOVED — skip
                ResourceChange.ChangeType.REMOVED, false),
            mockChange("/content/mysite/en/page5",              // external — skip
                ResourceChange.ChangeType.CHANGED, true)
        );

        listener.onChange(changes);

        // Only 2 out of 5 changes are eligible
        verify(jobManager, times(2)).addJob(anyString(), anyMap());
    }

    // ── Empty change list ──────────────────────────────────────────────────

    @Test
    void onChange_whenEmptyList_doesNothing() {
        listener.onChange(Collections.emptyList());
        verifyNoInteractions(jobManager);
    }
}
```

---

### 5.5 OSGi EventHandler Testing

#### The EventHandler Under Test

```java
@Component(service = EventHandler.class, immediate = true,
    property = { EventConstants.EVENT_TOPIC + "=" + ReplicationAction.EVENT_TOPIC })
public class ReplicationEventHandler implements EventHandler {

    @Reference
    private JobManager jobManager;

    private static final String JOB_TOPIC = "sibi-aem-one/replication/job";

    @Override
    public void handleEvent(Event event) {
        ReplicationAction action = ReplicationAction.fromEvent(event);
        if (action == null) {
            return;
        }
        // Only react to ACTIVATE (publish) events, not deactivate or delete
        if (action.getType() != ReplicationActionType.ACTIVATE) {
            return;
        }
        String path = action.getPath();
        if (StringUtils.isBlank(path)) {
            return;
        }
        Map<String, Object> props = new HashMap<>();
        props.put("path",       path);
        props.put("actionType", action.getType().getName());
        jobManager.addJob(JOB_TOPIC, props);
    }
}
```

#### EventHandler Test Class

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class ReplicationEventHandlerTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private JobManager jobManager;

    private ReplicationEventHandler handler;

    @BeforeEach
    void setUp() {
        ctx.registerService(JobManager.class, jobManager);
        handler = ctx.registerInjectActivateService(new ReplicationEventHandler());
    }

    // ── Helper to build a replication Event ───────────────────────────────

    private Event buildReplicationEvent(ReplicationActionType type, String path) {
        // ReplicationAction.fromEvent() reads specific event properties
        Map<String, Object> props = new HashMap<>();
        props.put(ReplicationAction.PROPERTY_ACTION_TYPE,   type.getName());
        props.put(ReplicationAction.PROPERTY_ACTION_PATHS,  new String[]{ path });
        props.put(ReplicationAction.PROPERTY_ACTION_USER,   "admin");
        return new Event(ReplicationAction.EVENT_TOPIC, props);
    }

    // ── Happy path ─────────────────────────────────────────────────────────

    @Test
    void handleEvent_whenActivateEvent_enqueuesJob() {
        Event event = buildReplicationEvent(
            ReplicationActionType.ACTIVATE,
            "/content/mysite/en/page"
        );

        handler.handleEvent(event);

        verify(jobManager, times(1))
            .addJob(eq("sibi-aem-one/replication/job"), anyMap());
    }

    @Test
    void handleEvent_whenActivateEvent_includesPathInJobProperties() {
        ArgumentCaptor<Map<String, Object>> captor = ArgumentCaptor.forClass(Map.class);

        handler.handleEvent(buildReplicationEvent(
            ReplicationActionType.ACTIVATE, "/content/mysite/en/page"));

        verify(jobManager).addJob(anyString(), captor.capture());

        assertEquals("/content/mysite/en/page", captor.getValue().get("path"));
        assertEquals("Activate", captor.getValue().get("actionType"));
    }

    // ── Filtering: non-ACTIVATE event types ───────────────────────────────

    @Test
    void handleEvent_whenDeactivateEvent_doesNotEnqueueJob() {
        Event event = buildReplicationEvent(
            ReplicationActionType.DEACTIVATE,
            "/content/mysite/en/page"
        );

        handler.handleEvent(event);

        verifyNoInteractions(jobManager);
    }

    @Test
    void handleEvent_whenDeleteEvent_doesNotEnqueueJob() {
        Event event = buildReplicationEvent(
            ReplicationActionType.DELETE,
            "/content/mysite/en/page"
        );

        handler.handleEvent(event);

        verifyNoInteractions(jobManager);
    }

    // ── Null/edge case handling ────────────────────────────────────────────

    @Test
    void handleEvent_whenEventHasNoValidAction_doesNotEnqueueJob() {
        // An event with no replication action properties
        Event emptyEvent = new Event(ReplicationAction.EVENT_TOPIC,
            new HashMap<String, Object>());

        // Should not throw — ReplicationAction.fromEvent() returns null
        assertDoesNotThrow(() -> handler.handleEvent(emptyEvent));
        verifyNoInteractions(jobManager);
    }
}
```

---

### Phase 5 — Summary

| Topic | Key Takeaways |
|---|---|
| Scheduler | Call `run()` directly — no timer/cron needed. Use `spy()` + `verify()` to confirm `doWork()` is/isn't called. Test `enabled=false` path explicitly. |
| Job Consumer | `mock(Job.class)` + `when(job.getProperty(...))`. Call `consumer.process(job)` directly. Assert `JobResult`. Use `CANCEL` for non-retryable, `FAILED` for retryable failures. |
| Job Producer | `mock(JobManager.class)`. Use `ArgumentCaptor<Map>` to inspect what properties were passed to `addJob()`. Verify topic string is correct. |
| ResourceChangeListener | `mock(ResourceChange.class)` + `when(change.getPath/getType/isExternal())`. Call `listener.onChange(List.of(change))`. Test each filter branch separately. |
| EventHandler | Build a real OSGi `Event` with the correct topic and property map. Call `handler.handleEvent(event)` directly. Verify job is/isn't enqueued based on action type. |
| Golden rule for all async components | Call the entry point method directly — `run()`, `process()`, `onChange()`, `handleEvent()`. Never wait for async execution in unit tests. |
