# AEM Interview Reference Notes

A structured knowledge base built while preparing for Senior AEM Developer interviews after ~8 years of hands-on AEM backend/API development. These are conceptual deep-dives and cheat sheets, not a runnable project — for working code, see [sibi-aem-one](https://github.com/SIBISARVANAN/sibi-aem-one).

## Why this exists

Most AEM work happens behind client NDAs, so day-to-day project experience rarely touches every corner of the platform at once. These notes are the result of deliberately filling in and revising the full breadth of AEM concepts — OSGi internals, Sling request handling, Oak/JCR internals, testing patterns, and build tooling — beyond whatever a single project happens to require.

## Contents

### Core Concepts (`docs/core-concepts/`)
Fundamentals shared across nearly every AEM backend task.

- [01 — Sling Models](docs/core-concepts/sling-models.md)
- [02 — OSGi Service Registry](docs/core-concepts/osgi-service-registry.md)
- [03 — OSGi Configuration Registry](docs/core-concepts/osgi-configuration-registry.md)
- [04 — Sling Servlets](docs/core-concepts/sling-servlets.md)
- [05 — Sling Jobs](docs/core-concepts/sling-jobs.md)
- [06 — Event Handlers & Resource Listeners](docs/core-concepts/event-handlers-resource-listeners.md)
- [07 — Cluster-Aware Listeners](docs/core-concepts/cluster-aware-listeners.md)
- [08 — Sling Filters](docs/core-concepts/sling-filters.md)
- [09 — Request Flow: CDN → Dispatcher → AEM](docs/core-concepts/request-flow-cdn-dispatcher-aem.md)
- [10 — AEM Workflows](docs/core-concepts/aem-workflows.md)

### Applied Code Patterns (`docs/applied-code-patterns/`)
Patterns tied to real scenarios and security/ops concerns, not just API surface.

- [Child Resource, Custom Serializer, Replication & Notification](docs/applied-code-patterns/child-resource-serializer-replication.md)
- [Granite Widget, Content Fragment & Adobe Launch](docs/applied-code-patterns/granite-widget-cf-adobe-launch.md)
- [CSRF Token Handling](docs/applied-code-patterns/csrf-token-handling.md)
- [XSS Protection — XSSAPI](docs/applied-code-patterns/xss-protection-xssapi.md)
- [Thread Dumps — Reading & Analysis](docs/applied-code-patterns/thread-dumps.md)
- [Heap Dumps — Eclipse MAT](docs/applied-code-patterns/heap-dumps-eclipse-mat.md)
- [CIF GraphQL Caching](docs/applied-code-patterns/cif-graphql-caching.md)

### Interview Topics (`docs/interview-topics/`)
Broader platform knowledge for senior-level interview breadth.

- [JCR & Oak — Fundamentals](docs/interview-topics/jcr-oak-fundamentals.md)
- [QueryBuilder & Search](docs/interview-topics/querybuilder-and-search.md) *(includes the full Lucene/index-internals deep dive)*
- [Dispatcher](docs/interview-topics/dispatcher.md)
- [Security — Service Users](docs/interview-topics/security-service-users.md)
- [CAConfig](docs/interview-topics/caconfig.md)
- [AEMaaCS vs 6.5](docs/interview-topics/aemaacs-vs-65.md)
- [Unit Testing — Stack Overview](docs/interview-topics/unit-testing-overview.md)
- [HTL & Component Development](docs/interview-topics/htl-and-components.md)
- [TagManager & Taxonomy](docs/interview-topics/tagmanager-taxonomy.md)
- [Replication API](docs/interview-topics/replication-api.md)
- [Senior Interview Q&A](docs/interview-topics/senior-interview-qa.md) — architecture, performance, debugging, data migration, common gotchas
- [REST APIs & GraphQL in AEM](docs/interview-topics/rest-and-graphql.md)
- [Oak & JCR Internals — NodeStore, MVCC, TarMK, MongoMK](docs/interview-topics/oak-jcr-internals-nodestore.md)
- [Sling Request Processing Pipeline](docs/interview-topics/sling-request-processing-pipeline.md)
- [Maven Plugins Reference](docs/interview-topics/maven-plugins.md) — compiler, bundle/bnd, filevault-package, resources, jar, surefire, failsafe, sling-maven-plugin, jacoco, full lifecycle walkthrough

### Testing (`docs/junits/`)
- JUnit Testing Guide, Phases 1–10 — OSGi service mocking, ResourceResolver/ValueMap, QueryBuilder, WorkflowProcess, Replicator, ContentFragment, Mockito Spy/ArgumentCaptor/MockedStatic, parameterized tests, JaCoCo setup

## How to use this

Each doc pairs a plain-language explanation with the technical depth and edge cases behind it — written to be read once for learning, then skimmed again for interview revision.

---

*This repository is shared publicly for portfolio and interview purposes. Please do not copy or redistribute without permission.*
