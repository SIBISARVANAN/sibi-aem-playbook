# AEM cheat sheet: Filters, Listeners, Event Handlers, Jobs, Schedulers

One-line summary to hold the whole set together:
**Filters guard the request. Listeners watch the repository. Event handlers watch the JVM's own event bus. Jobs are the only one of the four safe to run exactly once across a cluster. Schedulers are the only one that isn't reactive at all — they manufacture their own trigger out of time.**

---

## Quick decision table

| You need to... | Use |
|---|---|
| Inspect/block/modify an HTTP request or response | **Filter** |
| React when content changes in the repository | **Listener** (`ResourceChangeListener`) |
| React to a replication action or a custom cross-service signal posted via `EventAdmin` | **Event handler** (`EventHandler`) |
| Do deferred, retryable, cluster-safe work triggered by something else | **Job** (`JobConsumer` + `JobManager`) |
| Run something on a timer, with no external trigger | **Scheduler** (`Runnable` + `Scheduler`, or `JobManager.schedule().cron()`) |

---

## 1. Filters

**Trigger:** every HTTP request/response passing through Sling's engine.
**Runs on:** the request thread, synchronously — slow filter = slow page.
**Cluster-safe?** N/A — runs per-request, per-node, no cross-node concern.

### Annotation skeleton (modern)
```java
@Component(property = { "service.ranking:Integer=-200" })
@SlingServletFilter(
    scope         = { SlingServletFilterScope.REQUEST },   // REQUEST | INCLUDE | FORWARD | COMPONENT | ERROR
    pattern       = "/content/myapp/secure/.*",
    resourceTypes = { "myapp/components/page" },
    selectors     = { "print" },
    extensions    = { "html", "json" },
    methods       = { "GET", "POST" }
)
public class MyFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        // PRE-processing
        chain.doFilter(request, response);   // MUST call this or request is blocked
        // POST-processing
    }

    @Override public void init(FilterConfig fc) {}
    @Override public void destroy() {}
}
```

### Key gotchas
- `chain.doFilter()` not called → request stops dead. Forgetting it by accident → blank response, no error.
- `service.ranking`: **more negative runs earlier**. Auth/security filters should rank very negative; response-shaping filters closer to zero.
- Scope matters: a `REQUEST`-only filter will NOT fire for `<sling:include>` — use `COMPONENT` or `INCLUDE` if you need that.
- Filters are singletons — never store per-request state in instance fields; only local variables inside `doFilter()`.
- To read/modify a response body: wrap the response BEFORE calling the chain, read the buffer AFTER.

```java
BufferedHttpResponseWrapper wrapped = new BufferedHttpResponseWrapper((HttpServletResponse) response);
chain.doFilter(request, wrapped);
String html = wrapped.getBufferedContent();
String modified = html.replace("</body>", "<script src='...'></script></body>");
response.setContentLength(modified.getBytes(response.getCharacterEncoding()).length);
response.getWriter().write(modified);
```

---

## 2. Listeners (`ResourceChangeListener`)

**Trigger:** a JCR node change (add/change/remove) under a watched path.
**Runs on:** an Oak background observation thread — NOT the request thread, always a beat behind.
**Cluster-safe?** Cluster-*aware* — via `isExternal()` — not automatically cluster-safe (you decide what to do with external vs internal changes).

### Annotation skeleton
```java
@Component(
    service = ResourceChangeListener.class,
    immediate = true,
    property = {
        ResourceChangeListener.PATHS   + "=/content/myapp",
        ResourceChangeListener.PATHS   + "=glob:/content/myapp/**/jcr:content",
        ResourceChangeListener.CHANGES + "=ADDED",
        ResourceChangeListener.CHANGES + "=CHANGED",
        ResourceChangeListener.CHANGES + "=REMOVED"
    }
)
public class MyListener implements ResourceChangeListener, ExternalResourceChangeListener {
    // implementing ExternalResourceChangeListener is REQUIRED to even receive external changes

    @Reference
    private JobManager jobManager;

    @Override
    public void onChange(List<ResourceChange> changes) {
        for (ResourceChange change : changes) {
            if (change.isExternal()) {
                // change originated on a DIFFERENT cluster node
            } else {
                // change originated on THIS node
            }
            // never do heavy/blocking work here — hand off to a job
            Map<String, Object> props = new HashMap<>();
            props.put("path", change.getPath());
            jobManager.addJob("myapp.contentchange", props);
        }
    }
}
```

### Key gotchas
- `PATHS` property is mandatory — no path, no registration. Repeated `PATHS` keys = OR match. `glob:` enables `**` wildcards.
- `onChange()` always receives a **batch** (`List<ResourceChange>`) — one save can produce many entries.
- Without implementing `ExternalResourceChangeListener`, Sling filters out external changes before your code ever sees them.
- `isExternal()` tells you if the commit came from THIS JVM or another cluster node reading the same shared repository — no direct node-to-node communication involved.
- Always filter out noisy sub-property paths (`jcr:content` property writes) you don't care about, or you'll flood downstream jobs.
- Never hold a JCR session open across calls — open a service-user resolver, use it, close it, every time.

---

## 3. Event handlers (`EventHandler`)

**Trigger:** something posts to the OSGi EventAdmin bus (e.g. `Replicator.replicate()` internally posts a replication event).
**Runs on:** synchronously with whoever posted the event — potentially the caller's own thread.
**Cluster-safe?** No cross-cluster visibility at all — OSGi events never leave the JVM that posted them.

### Annotation skeleton
```java
@Component(
    service = EventHandler.class,
    immediate = true,
    property = { EventConstants.EVENT_TOPIC + "=" + ReplicationAction.EVENT_TOPIC }
)
public class MyEventHandler implements EventHandler {

    @Reference
    private JobManager jobManager;

    @Override
    public void handleEvent(Event event) {
        String actionType = (String) event.getProperty(ReplicationAction.PN_ACTION_TYPE);
        String path        = (String) event.getProperty(SlingConstants.PROPERTY_PATH);
        // keep this FAST — no blocking I/O here
        if (ReplicationActionType.ACTIVATE.name().equals(actionType)) {
            Map<String, Object> props = new HashMap<>();
            props.put("path", path);
            jobManager.addJob("myapp.replicated", props);
        }
    }
}
```

### Key gotchas
- `EVENT_TOPIC` is an OSGi bus topic string, not a JCR path. Built-ins: `ReplicationAction.EVENT_TOPIC`, `SlingConstants.TOPIC_RESOURCE_*`, or your own custom topic posted via `EventAdmin.postEvent()`.
- Local-JVM only — this is THE distinguishing feature vs listeners. No `isExternal()` concept exists because there's nothing to be external to.
- Never do blocking work inside `handleEvent()` — it can measurably slow down whatever triggered the event. Enqueue a job instead.
- Use this specifically for lifecycle-style signals that aren't naturally a JCR write (replication actions, workflow completion, custom cross-service events).

---

## 4. Jobs (`JobConsumer` + `JobManager`)

**Trigger:** explicit `jobManager.addJob()` call (usually from a listener, event handler, or scheduler), or a cron schedule registered via the job framework.
**Runs on:** a job-processing thread pool, claimed by exactly ONE node in the cluster.
**Cluster-safe?** Yes, by design — the job resource lives in the shared repository (`/var/eventing/jobs`) and claiming it is a single atomic write.

### Producer (fire-once)
```java
@Component(service = MyJobProducer.class, immediate = true)
public class MyJobProducer {
    private static final String TOPIC = "myapp.dowork";

    @Reference
    private JobManager jobManager;

    public void triggerJob(String payload) {
        Map<String, Object> props = new HashMap<>();
        props.put("payloadPath", payload);
        jobManager.addJob(TOPIC, props);
    }
}
```

### Consumer
```java
@Component(service = JobConsumer.class, immediate = true,
    property = { JobConsumer.PROPERTY_TOPICS + "=myapp.dowork" })
public class MyJobConsumer implements JobConsumer {

    @Override
    public JobResult process(Job job) {
        try {
            String path = (String) job.getProperty("payloadPath");
            doWork(path);
            return JobResult.OK;
        } catch (IllegalArgumentException e) {
            // permanent failure — retrying won't help
            return JobResult.CANCEL;
        } catch (Exception e) {
            // transient failure — safe to retry
            return JobResult.FAILED;
        }
    }
}
```

### Producer — cron-scheduled, cluster-safe (preferred over Runnable+Scheduler for cluster-wide cron)
```java
@Component(service = MyScheduledJobProducer.class, immediate = true)
@Designate(ocd = MyJobConfig.class)
public class MyScheduledJobProducer {
    private static final String TOPIC = MyScheduledJobConsumer.JOB_TOPIC;

    @Reference
    private JobManager jobManager;
    private MyJobConfig config;

    @Activate
    protected void activate(MyJobConfig config) {
        this.config = config;
        scheduleJob();
    }

    @Modified
    protected void modified(MyJobConfig config) {
        this.config = config;
        unscheduleJob();
        scheduleJob();
    }

    @Deactivate
    protected void deactivate() { unscheduleJob(); }

    private void scheduleJob() {
        if (!config.isEnabled()) return;
        if (!jobManager.getScheduledJobs(TOPIC, 1, null).isEmpty()) return;   // duplicate guard
        jobManager.createJob(TOPIC).schedule().cron(config.cronExpression()).add();
    }

    private void unscheduleJob() {
        jobManager.getScheduledJobs(TOPIC, 1, null).forEach(ScheduledJobInfo::unschedule);
    }
}
```

### Key gotchas
- `JobResult` is a contract:
  - `OK` — done, remove from queue.
  - `FAILED` — transient, will be retried per queue retry config.
  - `CANCEL` — permanent, never retried.
  Misclassifying these is the #1 job-design mistake — `FAILED` on a permanent error clogs the queue forever; `CANCEL` on a transient error silently drops work.
- Consumers must be **idempotent** — at-least-once delivery means `process()` can run more than once for the same logical unit of work.
- Always guard scheduled-job registration with a duplicate check before `@Modified` re-adds it — otherwise every config tweak stacks a new schedule on top of the old one.
- `.schedule().cron()` is cluster-safe for free — this is the modern replacement for hand-rolled JCR-lock schedulers.
- `JobExecutor` (newer API) exists for jobs that need to signal async completion later — `JobConsumer` is enough for anything that finishes synchronously inside `process()`.

---

## 5. Schedulers (`Runnable` + Sling Commons `Scheduler`)

**Trigger:** a cron expression — no external event at all. This is the only mechanism of the five that's proactive, not reactive.
**Runs on:** a local Quartz-backed thread pool, per JVM.
**Cluster-safe?** No, by default — every node in the cluster fires independently. Must be hand-coded for cluster safety if needed (JCR lock pattern), or replaced entirely with `JobManager.schedule().cron()`.

### Plain scheduler (per-node execution — use when you WANT node-local behavior)
```java
@Component(service = Runnable.class, immediate = true, configurationPolicy = ConfigurationPolicy.REQUIRE)
@Designate(ocd = MySchedulerConfig.class)
public class MyScheduler implements Runnable {
    private String schedulerName;

    @Reference
    private Scheduler scheduler;

    @Activate
    protected void activate(MySchedulerConfig config) {
        schedulerName = "myapp.scheduler";
        if (config.enabled()) {
            ScheduleOptions options = scheduler.EXPR(config.cronExpression());
            options.name(schedulerName);
            options.canRunConcurrently(false);   // prevents overlap WITHIN this node only
            scheduler.schedule(this, options);
        }
    }

    @Deactivate
    protected void deactivate() {
        scheduler.unschedule(schedulerName);
    }

    @Override
    public void run() {
        // keep fast, or hand off to a Job
    }
}
```

### Cluster-safe scheduler (manual JCR lock pattern)
```java
@Override
public void run() {
    if (!jvmLock.tryLock()) return;                 // guard against overlap on THIS node
    try (ResourceResolver resolver = getResourceResolver()) {
        if (!acquireClusterLock(resolver)) return;    // guard against overlap ACROSS nodes
        try {
            doRealWork();
        } finally {
            releaseClusterLock(resolver);
        }
    } catch (Exception e) {
        // log
    } finally {
        jvmLock.unlock();
    }
}

private boolean acquireClusterLock(ResourceResolver resolver) throws RepositoryException {
    Session session = resolver.adaptTo(Session.class);
    LockManager lockManager = session.getWorkspace().getLockManager();
    if (lockManager.isLocked(LOCK_PATH)) return false;
    lockManager.lock(LOCK_PATH, false, true, 300, null);  // isDeep=false, isSessionScoped=true
    return true;
}
```

### Key gotchas
- `canRunConcurrently(false)` only prevents overlap on ONE JVM's thread pool — it says nothing about a second cluster node.
- Two different "scheduler" mechanisms exist and answer different needs:
  | | `Runnable` + `Scheduler` | `JobManager.schedule().cron()` |
  |---|---|---|
  | Cron lives | Local to JVM (Quartz) | Shared repository |
  | Cluster-safe | No | Yes |
  | Use when | Deliberately want per-node behavior, or willing to hand-roll locking | Default choice for anything that should run once, cluster-wide |
- `configurationPolicy = ConfigurationPolicy.REQUIRE` prevents activation until OSGi config exists — a safety net against running with silent defaults in production.
- `@Modified` must unschedule-then-reschedule, same as job producers, or config changes stack duplicate schedules.
- Session-scoped JCR locks (`isSessionScoped=true`) self-release when the resolver closes — even if the node crashes mid-run — but there's still a TOCTOU race between `isLocked()` check and `lock()` call; catch `LockException` specifically to treat "someone else has it" as expected, not an error.
- Never hardcode the cron expression — always drive it from `@Designate` config so it's changeable via `/system/console/configMgr` without a redeploy.

---

## The whole picture in one paragraph

A browser request hits a **filter** first — it can block, log, or reshape the request/response before anything else runs. If the request (or any background process) writes to the repository, a **listener** notices asynchronously, cluster-aware via `isExternal()`. If AEM's own internal machinery — like replication — posts an OSGi event, an **event handler** reacts, but only on the JVM where it happened. Both listeners and event handlers typically hand off real work to a **job**, the only mechanism guaranteed to run exactly once across the whole cluster, with built-in retry semantics via `JobResult`. And sitting outside this entire reactive chain, a **scheduler** is what starts things on a timer with no upstream trigger at all — either per-node (`Runnable` + `Scheduler`) or cluster-wide for free (`JobManager.schedule().cron()`).
