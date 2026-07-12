# Sling Jobs

## Why Sling Jobs Instead of Plain Schedulers?

| Feature | Plain Sling Scheduler | Scheduled Sling Job |
|---|---|---|
| Survives server restart? | No | **Yes** — persisted in JCR |
| Cluster-aware execution? | No — runs on every node | **Yes** — runs on exactly one node |
| Retry on failure? | No | **Yes** — automatic with backoff |
| Persistent audit trail? | No | Yes — `/var/eventing/jobs/` |

## End-to-End Flow of a Scheduled Sling Job

### Step 1 — OSGi Config is Loaded

AEM reads your `.cfg.json` on startup or bundle deploy:

```json
{
  "enabled": true,
  "timezone1.cron": "0 0 5 * * ?",
  "timezone1.id": "America/New_York"
}
```

Each JSON key maps to an `@interface Config` method using the **underscore-to-dot rule**: `timezone1_cron()` → `"timezone1.cron"`.

### Step 2 — Config is Injected into the Component

`@Designate(ocd = Config.class)` links the component to the config schema. OSGi calls `@Activate` and passes in the populated config object.

```
.cfg.json  →  @interface Config  →  @Activate(Config config)
```

If the config is later changed in the Felix console, `@Modified` fires — old jobs are unscheduled and new ones registered with the updated values.

### Step 3 — JobManager Registers the Scheduled Job

```java
jobManager
    .createJob(TOPIC)
    .properties(props)       // e.g. timezoneId, schedulerName
    .schedule()
    .cron("0 0 5 * * ?")
    .add();
```

> **Important:** Before calling `.add()`, always call `getScheduledJobs()` to check for duplicates. Without this guard, the same job gets re-registered on every bundle restart.

### Step 4 — Job is Persisted in JCR

Unlike a plain `Sling Scheduler`, a scheduled Sling Job is written to the JCR at `/var/eventing/scheduled-jobs/`. This means it **survives a server restart** — the schedule is not lost when AEM goes down.

### Step 5 — Cron Fires, Job Instance is Created

When the cron expression fires, Sling Eventing creates a job instance at `/var/eventing/jobs/`. The properties attached when scheduling (e.g. `timezoneId`, `schedulerName`) are carried into the job instance and available to the consumer.

### Step 6 — JobConsumer Processes the Job

```java
@Component(
    service  = JobConsumer.class,
    property = { JobConsumer.PROPERTY_TOPICS + "=com/example/myapp/topic" }
)
public class MyConsumer implements JobConsumer {
    public JobResult process(Job job) {
        // heavy work here
        return JobResult.OK;
    }
}
```

In a clustered AEM/AMS environment, Sling ensures the job runs on **exactly one node** — no manual cluster coordination needed.

### Step 7 — JobResult Determines What Happens Next

| Return | Meaning | Job removed? | Retried? |
|---|---|---|---|
| `OK` | Success | Yes | No |
| `FAILED` | Error — try again | No | Yes (with automatic backoff) |
| `CANCEL` | Intentional abort | Yes | No |

Retry count and backoff delay are configurable at `/system/console/configMgr` in the OSGi job queue config.

## Complete Flow Diagram

```
.cfg.json
   ↓ OSGi reads and maps keys
@interface Config
   ↓ @Designate + @Activate
JobRegistrar (Producer)
   ↓ @Reference + createJob().schedule().cron().add()
JobManager
   ↓ persists to JCR
/var/eventing/scheduled-jobs   ← survives server restart
   ↓ cron fires
/var/eventing/jobs             ← job instance created with properties
   ↓ topic matched
JobConsumer.process(job)
   ↓ returns
OK → done | FAILED → retry | CANCEL → abort
```

## Multi-Timezone Job Pattern

To run a job at midnight in three different timezones, register three separate scheduled jobs from a single producer — one per timezone. Each job carries its timezone ID as a property, and the consumer uses `ZoneId.of(timezoneId)` to log and process in the correct local time.

**Deduplication key:** Store a `schedulerName` as a job property and use it to detect duplicates in `getScheduledJobs()` before calling `.add()`.

```java
// Duplicate guard
Collection<ScheduledJobInfo> existing = jobManager.getScheduledJobs(TOPIC, -1, null);
for (ScheduledJobInfo info : existing) {
    if (schedulerName.equals(info.getJobProperties().get("schedulerName"))) {
        return; // already registered
    }
}
```

## Common Interview Questions

**Q: What is the difference between a Sling Scheduler and a Sling Job?**
A Sling Scheduler is a simple `Runnable` fired by a cron expression. It runs on every cluster node, has no persistence, and has no retry. A Sling Job is persisted in JCR (`/var/eventing/`), runs on exactly one cluster node, and retries automatically on `FAILED`. Use schedulers for lightweight per-node tasks (e.g. cache warm-up). Use Sling Jobs for all business-critical work.

**Q: What happens if the AEM instance goes down mid-job?**
For plain schedulers — the job is lost. For Sling Jobs — the job instance in `/var/eventing/jobs/` survives. When AEM restarts, the Sling Eventing framework picks it up and re-delivers it to a consumer.

**Q: How do you pass data from a producer to a consumer?**
Via job properties. The producer adds key-value pairs to the `Map<String, Object>` when calling `jobManager.addJob(topic, props)`. The consumer reads them via `job.getProperty("myKey", String.class)`.

**Q: How do you prevent a job from running on every node in a cluster?**
Sling Jobs are cluster-aware by design — the job queue distributes to one node automatically. You do not need to add any cluster coordination. The issue only arises with plain schedulers, which must be guarded with JCR locks or `isLeader()` checks.

**Q: What is `JobResult.CANCEL` vs `JobResult.FAILED`?**
`FAILED` signals a transient failure — retry will be attempted. `CANCEL` signals a permanent, intentional abort — no retry, job is removed. Use `CANCEL` when the error is unrecoverable (e.g. invalid payload path, configuration error).
