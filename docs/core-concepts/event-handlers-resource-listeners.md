# Sling Event Handlers & Resource Listeners

## When to Use Which

| | OSGi `EventHandler` | `ResourceChangeListener` |
|---|---|---|
| Use for | System events (replication, DAM, workflow, bundle) | JCR content / resource changes |
| Package | `org.osgi.service.event` | `org.apache.sling.api.resource.observation` |
| Status | Current | Current (replaces deprecated EventHandler + SlingConstants resource topics) |
| `immediate = true` required? | Always | Always |
| Heavy work inside? | Never — delegate to Sling Job | Never — delegate to Sling Job |

## Common Event Topics

**OSGi EventHandler topics:**

| Event | Topic Constant |
|---|---|
| Page activated / deactivated | `ReplicationAction.EVENT_TOPIC` |
| Page created / modified / deleted | `PageEvent.EVENT_TOPIC` |
| DAM asset events | `DamEvent.EVENT_TOPIC` |
| Workflow completed | `WorkflowEvent.TOPIC` |
| Bundle started / stopped | `BundleEvent.TOPIC` |

**ResourceChangeListener change types:**

| Constant | Meaning |
|---|---|
| `ADDED` | Node/resource created |
| `CHANGED` | Node/resource modified |
| `REMOVED` | Node/resource deleted |

## The Golden Rule — Keep the Handler Fast

The event thread processes events serially. Any slow work inside `handleEvent()` or `onChange()` will back up the entire event queue for that AEM instance. Since the event handler service might receive a lot of events even concurrently, it is advised to set `immediate = true` on the component — otherwise the handler would be created and destroyed with every event coming in.

**Never do heavy work inside an event handler. Always delegate to a Sling Job:**

```
EventHandler.handleEvent()
└── jobManager.addJob(topic, props)   ← fire and return immediately
        │
        ▼
JobConsumer.process(job)              ← heavy work here, with automatic retry
```

## ResourceChangeListener — Path Registration with Glob Patterns

```java
@Component(service = ResourceChangeListener.class, immediate = true, property = {
    ResourceChangeListener.PATHS   + "=/content/sibi-aem-one",
    ResourceChangeListener.PATHS   + "=glob:/content/sibi-aem-one/**/*.html",
    ResourceChangeListener.PATHS   + "=glob:/content/sibi-aem-one/**/products/**",
    ResourceChangeListener.PATHS   + "=glob:/content/sibi-aem-one/**/jcr:content",
    ResourceChangeListener.CHANGES + "=ADDED",
    ResourceChangeListener.CHANGES + "=CHANGED",
    ResourceChangeListener.CHANGES + "=REMOVED"
})
```

- Use the `glob:` prefix for wildcard path matching.
- Subscribe only to the change types you actually need — over-subscribing fires unnecessary events.
