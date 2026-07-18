# Cluster-Aware Listeners

## The Problem

In AEM AMS, author runs as a cluster (typically 2 nodes). When node 1 makes a change, a plain `ResourceChangeListener` on node 2 will NOT receive that event — it silently misses it. Cluster-aware listeners fix this.

## The Two Interfaces

`ResourceChangeListener` — listens to changes on the current node only.

`ExternalResourceChangeListener` — marker interface. Implement it alongside `ResourceChangeListener` to also receive changes that originated on other cluster nodes.

```java
// Listens to changes on THIS node only
public class MyListener implements ResourceChangeListener { }

// Listens to changes from THIS node AND all other cluster nodes
public class MyListener implements ResourceChangeListener, ExternalResourceChangeListener { }
```

Both are in the package `org.apache.sling.api.resource.observation`.

## The Key Method: isExternal()

`change.isExternal()` — tells you where the change came from. This is the only way to distinguish local from external events inside `onChange()`.

| Returns | Meaning |
|---|---|
| `false` | Change happened on **this** node (local) |
| `true` | Change happened on **another** cluster node (external) |

## When to Use Each Approach

| Use case | Approach |
|---|---|
| Cache invalidation, search index updates — every node must react | Implement `ExternalResourceChangeListener`, process all changes regardless of origin |
| Background processing, data sync — only one node should do the work | Check `!isExternal()` and skip if external — the node that made the change processes it, others ignore it to avoid duplicate job execution |
| Primary + secondary store updates on different nodes | Check `isExternal()` and branch logic accordingly — primary data store updates on the originating node, secondary data store sync on all other nodes |

## The Duplicate Job Problem

This is the most common mistake with cluster-aware listeners. If you implement `ExternalResourceChangeListener` and fire a Sling Job in `onChange()` without checking `isExternal()`, every node in the cluster fires its own job for the same change.

- 2-node cluster → 2 identical jobs
- 4-node cluster → 4 identical jobs

**Fix:** Check `isExternal()` before firing a job, and only fire on the node where the change originated. Or use a Sling Job (which is cluster-aware by nature and will run on exactly one node) and let that handle deduplication.

```java
@Override
public void onChange(List<ResourceChange> changes) {
    for (ResourceChange change : changes) {
        if (!change.isExternal()) {
            jobManager.addJob("my.topic.internal", props); // only this node fires
        } else {
            jobManager.addJob("my.topic.external", props); // other cluster nodes
        }
    }
}
```

## Author Cluster vs Publish Farm

| | Author Cluster | Publish Farm |
|---|---|---|
| Nodes | Peers sharing a single JCR (MongoMK/TarMK-shared) | Independent — each node has its own JCR |
| How changes propagate | Oak/Jackrabbit replication between peers | Author-to-publish replication via Sling replication |
| `ExternalResourceChangeListener` useful? | **Yes** | **No** — publish nodes do not share a JCR |

Author cluster — nodes are peers. Changes on node 1 replicate to node 2 via Oak/Jackrabbit. `ExternalResourceChangeListener` handles this.

Publish farm — nodes are independent. They do NOT share a JCR. A change on publish node 1 is NOT visible to publish node 2 via `ResourceChangeListener` at all. Replication from author is the only way publish nodes get content. `ExternalResourceChangeListener` does NOT help across publish nodes.

## Gotchas

**REMOVED events may be for a parent.** If `/content/mysite` is deleted, you will not get individual REMOVED events for every child. You get one event for the parent. Your handler must account for this by checking if the removed path is an ancestor of your registered path, not just an exact match.

**`onChange()` must be fast.** It runs on the event thread. Any slow or blocking operation inside it will back up the event queue for the whole instance. Always delegate work to a Sling Job and return immediately.

**External events may arrive slightly delayed.** Do not assume real-time consistency across cluster nodes. Oak replication has a small lag. Design your handler to be tolerant of out-of-order or slightly delayed events.

**Do not open a ResourceResolver inside `onChange()`.** The event thread does not have a session. You must use a service ResourceResolver from `ResourceResolverFactory`, and always close it in a finally block.

**`immediate = true` is mandatory.** Without it the OSGi framework may create and destroy the component for every event, causing missed events during instantiation.

## OSGi Config Properties That Matter

`ResourceChangeListener.PATHS` — one or more JCR paths or glob patterns to watch. Use `glob:` prefix for pattern matching. Use `.` to watch everything (use with caution in production).

`ResourceChangeListener.CHANGES` — which change types to subscribe to. Values are ADDED, CHANGED, REMOVED, PROVIDER_ADDED, PROVIDER_REMOVED. Only subscribe to what you actually need.

## Evolution of Listener APIs — What to Use

| API | Status | Notes |
|---|---|---|
| `JCR EventListener` | Legacy — avoid | Raw JCR API, no cluster awareness, no glob paths |
| `OSGi EventHandler` + `SlingConstants` resource topics | Deprecated for resource changes | No cluster awareness, no glob paths |
| `ResourceChangeListener` alone | Current | Correct, but misses external cluster events |
| `ResourceChangeListener` + `ExternalResourceChangeListener` | Current — preferred | Receives changes from all cluster nodes |
