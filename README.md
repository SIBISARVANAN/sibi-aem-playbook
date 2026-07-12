# AEM Interview Reference Notes

A structured knowledge base built while preparing for Senior AEM Developer interviews after ~8 years of hands-on AEM backend/API development. These are conceptual deep-dives and cheat sheets, not a runnable project — for working code, see [sibi-aem-one](https://github.com/SIBISARVANAN/sibi-aem-one).

## Why this exists

Most AEM work happens behind client NDAs, so day-to-day project experience rarely touches every corner of the platform at once. These notes are the result of deliberately filling in and revising the full breadth of AEM concepts — OSGi internals, Sling request handling, Oak/JCR internals, testing patterns, and build tooling — beyond whatever a single project happens to require.

## Contents

### Core Concepts (`/docs/core-concepts`)
- [Sling Models](docs/core-concepts/sling-models.md)
- [OSGi Lifecycle Patterns](docs/core-concepts/osgi-lifecycle-patterns.md) — container-managed vs application-managed services, service ranking, service name resolution
- [Servlets & Registries](docs/core-concepts/servlets-and-registries.md) — path vs resourceType registration
- [Sling Jobs](docs/core-concepts/sling-jobs.md) — full scheduled-job lifecycle, JCR persistence, retry semantics
- [Event Handlers & Cluster-Aware Listeners](docs/core-concepts/event-handlers-and-listeners.md) — ResourceChangeListener vs ExternalResourceChangeListener, author cluster vs publish farm behavior
- [Sling Filters](docs/core-concepts/filters.md) — scopes, service ranking, response wrapping
- [Request Flow: CDN → Dispatcher → AEM](docs/core-concepts/request-flow-cdn-dispatcher.md)
- [AEM Workflows](docs/core-concepts/workflows.md) — engine architecture, WorkItem vs MetaDataMap, transient workflows, AEMaaCS asset microservices shift

### Testing (`/docs/testing`)
- JUnit Testing Guide, Phases 1–10 — OSGi service mocking, ResourceResolver/ValueMap, QueryBuilder, WorkflowProcess, Replicator, ContentFragment, Mockito Spy/ArgumentCaptor/MockedStatic, parameterized tests
- Master cheat sheet: MockitoExtension vs AemContextExtension decision table

### Build Tooling (`/docs/build-tooling`)
- Maven Plugins Reference — compiler, bundle/bnd, filevault-package, surefire, failsafe, jacoco, sling plugins with full lifecycle walkthroughs

### Auth (`/docs/auth`)
- Sling Authentication — AuthenticationHandler contract, IMS/SAML2, token-based auth for headless endpoints

## How to use this

Each doc pairs a plain-language explanation with the technical depth and edge cases behind it — written to be read once for learning, then skimmed again for interview revision.
