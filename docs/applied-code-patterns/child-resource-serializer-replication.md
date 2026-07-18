# Child Resource, Custom Serializer, Replication & Notification

> Added after a code-quality review identified these as missing patterns. All four are demonstrated together via one connected real-world scenario: an e-commerce product page with auto-publish and an approval workflow.

## @ChildResource — Multifield to Sling Model List

A dialog multifield stores repeating child nodes (e.g. product size/colour variants) under a container node. `@ChildResource(name = "variants")` injects them as `List<Resource>`; each `Resource` is then adapted to its own Sling Model (`adaptables = Resource.class`) inside `@PostConstruct`.

```java
@ChildResource(name = "variants")
private List<Resource> variantResources;

@PostConstruct
protected void init() {
    variants = variantResources.stream()
            .map(r -> r.adaptTo(ProductVariant.class))
            .filter(Objects::nonNull)   // adaptTo() can return null — never skip this
            .collect(Collectors.toList());
}
```

Child models always use `adaptables = Resource.class` — they have no servlet request of their own.

## Custom Jackson Serializer

The default exporter dumps every getter on a Sling Model into JSON. A custom `JsonSerializer<List<X>>`, applied via `@JsonSerialize(using = ...)` on a single DTO field, lets you control exactly which fields appear, rename them, and conditionally omit entries — none of which is possible with plain `@JsonIgnore`/`@JsonProperty` annotations alone.

## Programmatic Replication Trigger

Calling `Replicator.replicate()` directly from Java — from a `ResourceChangeListener` or a workflow step — instead of relying on a human clicking "Publish".

```java
ReplicationOptions options = new ReplicationOptions();
options.setSynchronous(false);     // never block the event thread
replicator.replicate(session, ReplicationActionType.ACTIVATE, pagePath, options);
```

Always use a service-user session for this in a listener (never admin), and always set `setSynchronous(false)` so the calling thread isn't blocked waiting on the publish instance.

## Workflow Replication Step + Notification Step

A two-step pattern: one `WorkflowProcess` activates the payload via `Replicator` and records the outcome (`SUCCESS`/`FAILED`) into `WorkflowData`'s `MetaDataMap`; a second step reads that outcome and emails the workflow initiator using AEM's native `MessageGatewayService` (NOT `org.apache.sling.commons.mail.MailService`, which is not the standard AEM mail API).

```java
// Step 1 — publish, write outcome to shared workflow metadata
workItem.getWorkflowData().getMetaDataMap().put("publishStatus", "SUCCESS");

// Step 2 — read it back, send notification
MessageGateway<HtmlEmail> gateway = messageGatewayService.getGateway(HtmlEmail.class);
gateway.send(email);
```

A notification failure must never fail the workflow if the actual business action (publishing) already succeeded — log and swallow, don't throw `WorkflowException`.

## Common Interview Questions — These Patterns

**Q: Why does `@ChildResource` need `adaptables = Resource.class` on the child model?**
Because individual child nodes of a multifield don't have an associated `SlingHttpServletRequest` — only the top-level component resource being rendered does. The child is just a JCR resource.

**Q: When do you need a custom Jackson serializer instead of `@JsonIgnore`?**
When the exclusion/inclusion logic depends on a computed value rather than presence/absence of a field — e.g. "only include variants that are in stock" can't be expressed by an annotation; it needs imperative code in a `JsonSerializer`.

**Q: Why must replication calls from a `ResourceChangeListener` be asynchronous?**
The resource-change event thread is shared and processes events serially. A synchronous `replicate()` call blocks that thread until the publish instance responds over the network — backing up the entire observation event queue for the whole node.

**Q: What's the correct AEM API for sending email from a workflow step?**
`com.day.cq.mailer.MessageGatewayService` → `getGateway(HtmlEmail.class)` → `gateway.send(email)`. There is no `org.apache.sling.commons.mail.MailService.sendEmail(Email, String[])` API in standard AEM.

**Q: Should a failed notification email fail the whole workflow?**
No. If the actual business action (publishing) already succeeded, a notification failure is a secondary concern — log it and continue. Throwing `WorkflowException` here would incorrectly mark a successfully-published page as a failed workflow instance.

## Concepts Clarified Alongside This Scenario

| Concept | One-line definition |
|---|---|
| Serialization | Converting an in-memory object into another format (JSON/bytes/XML) for storage or transmission — `Serializable`/`serialVersionUID` is the unrelated binary-object variant used by `HttpServlet`. |
| Connection pooling | Reusing already-open TCP connections instead of paying handshake cost per request; `PoolingHttpClientConnectionManager` caps total and per-route connections. |
| `ConcurrentHashMap` | Thread-safe map allowing concurrent reads/writes without external `synchronized` blocks — required whenever a singleton OSGi service's field is touched by multiple concurrent request threads. |
| Circular reference (OSGi) | Two components each holding a runtime `@Reference` to a service the other provides, forming a dependency loop. A nested `@interface Config` inside a component is metadata read at build/activation time — it is not a service reference and cannot cause this. |
