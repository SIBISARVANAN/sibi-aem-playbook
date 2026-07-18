# Replication API

## Programmatic Page Activation

```java
@Reference
private Replicator replicator;

// Activate (publish) a page
public void publishPage(String pagePath, Session session) {
    try {
        replicator.replicate(session, ReplicationActionType.ACTIVATE, pagePath);
        log.info("Published: {}", pagePath);
    } catch (ReplicationException e) {
        log.error("Failed to publish {}: {}", pagePath, e.getMessage(), e);
    }
}

// Deactivate (unpublish) a page
public void unpublishPage(String pagePath, Session session) throws ReplicationException {
    replicator.replicate(session, ReplicationActionType.DEACTIVATE, pagePath);
}

// Delete from publish (when a page is deleted on author)
public void deleteFromPublish(String pagePath, Session session) throws ReplicationException {
    replicator.replicate(session, ReplicationActionType.DELETE, pagePath);
}
```

## Replication Action Types

| Type | What it does |
|---|---|
| `ACTIVATE` | Publish the content to publish instances |
| `DEACTIVATE` | Unpublish — removes the content from publish |
| `DELETE` | Deletes the page from publish instances (used when the author page is deleted) |
| `TEST` | Sends a test ping to the replication agent |
| `REVERSE` | Pulls content from publish back to author (Reverse Replication) |

## ReplicationOptions — Batch Replication

```java
ReplicationOptions options = new ReplicationOptions();
options.setSynchronous(false);    // async — don't block the calling thread
options.setSuppressVersions(true); // don't create versions during replication
options.setFilter(agent -> agent.getId().equals("publish")); // target specific agent

replicator.replicate(session, ReplicationActionType.ACTIVATE, pagePath, options);
```

## Common Interview Questions

**Q: What is the difference between synchronous and asynchronous replication?**
Synchronous replication blocks the calling thread until the publish instance confirms receipt. Asynchronous adds the replication action to a queue and returns immediately — the actual transfer happens in the background. For bulk activation (e.g. activating 1000 pages from a workflow), always use async to avoid thread exhaustion.

**Q: What is Reverse Replication?**
The `REVERSE` action pulls content from publish back to author. Used for user-generated content (form submissions, ratings) where content is created on publish and needs to be stored on author. Rarely used in modern AEM projects — most UGC is stored in external databases.

**Q: What permission does a service user need to trigger replication?**
The service user must have `crx:replicate` permission on the content being replicated, plus `jcr:read`. Without `crx:replicate`, the `replicator.replicate()` call throws a `ReplicationException` with a permission error.

**Q: What is a replication agent and where is it configured?**
A replication agent is an OSGi-managed transport queue stored at `/etc/replication/agents.author/`. The default agent (`publish`) points to the publish instance URL. Flush agents (for Dispatcher cache invalidation) are stored at `/etc/replication/agents.author/flush`.
