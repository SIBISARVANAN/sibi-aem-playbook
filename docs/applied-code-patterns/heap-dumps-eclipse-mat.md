# Heap Dumps — Analysis with Eclipse MAT

## What it is

A full snapshot of every object on the JVM heap at one instant — used to diagnose memory leaks and `OutOfMemoryError`.

```bash
jmap -dump:live,format=b,file=heap.hprof <pid>
```

Open the resulting `.hprof` file in **Eclipse Memory Analyzer Tool (MAT)**.

## Eclipse MAT — four views to know

| View | Use for |
|---|---|
| **Histogram** | Class-level counts — which class has the most instances / retains the most memory |
| **Dominator Tree** | Which single objects are keeping the most memory alive — the "biggest offenders" list |
| **Leak Suspects Report** | MAT's automated analysis — a good first stop, flags likely leak candidates automatically |
| **Path to GC Roots** | For a specific suspect object, shows the exact reference chain keeping it alive — this is how you find *why* it isn't being garbage collected |

## Three most common AEM heap problems

1. **Unclosed `ResourceResolver` instances** — every service-user resolver opened via `ResourceResolverFactory.getServiceResourceResolver()` must be closed in a `finally` block. Leaked resolvers hold JCR session state and accumulate over time until OOM.
2. **Static collections used as caches with no eviction** — a `static Map` used as an ad-hoc cache (common in the application-managed OSGi registry pattern) that's never pruned grows unbounded under load.
3. **Listener/callback references not deregistered** — an object registers itself as a listener somewhere (event bus, cache invalidation hook) but is never removed, so the listener holds a reference that prevents garbage collection even after the "real" owner is done with it.

**Workflow:** start with Leak Suspects Report → confirm with Dominator Tree → use Path to GC Roots on the top suspect to find the actual line of code holding the reference.

## OQL — Targeted Hunting

MAT supports an SQL-like Object Query Language for hunting a specific suspected class directly, rather than browsing the histogram:

```sql
SELECT * FROM java.util.HashMap m WHERE m.size > 10000
```

## Common Interview Q&A

**Q: What's the difference between Shallow Heap and Retained Heap in MAT?**
Shallow Heap = memory used by the object itself (its own fields). Retained Heap = memory that would be freed if this object AND everything it exclusively references were collected. Retained Heap is what matters for finding leaks — an object with 48 bytes shallow but 287MB retained is holding a huge graph of other objects alive.

**Q: What is "Path to GC Roots" in MAT and why do you use it?**
It traces the reference chain from a suspicious object all the way back to a GC Root (a thread, a static field, a JNI reference) — the one thing keeping it alive. This tells you exactly which piece of your code is holding the object and preventing garbage collection.

**Q: Why use `jmap -dump:live` instead of plain `jmap -dump`?**
`live` forces a Full GC first, then dumps only surviving (reachable) objects. This eliminates unreachable objects that haven't been collected yet, making the dump smaller and the analysis focused on genuine leaks rather than GC noise.
