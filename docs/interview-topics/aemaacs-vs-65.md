# AEM as a Cloud Service — Key Differences from 6.5

## Architecture Changes

| Aspect | AEM 6.5 | AEMaaCS |
|---|---|---|
| Deployment | On-premise / AMS (managed) | Cloud-native on Adobe I/O (Kubernetes) |
| Repository | TarMK or MongoMK | Oak Segment TAR on Azure Blob / AWS S3 |
| Scaling | Manual / semi-auto | Auto-scaling — new pods spin up in minutes |
| Upgrades | Major version upgrades every 2–3 years | Continuous delivery — updated weekly |
| Dispatcher | Apache httpd on a managed VM | CDN-integrated (Adobe CDN / Fastly) |
| Asset processing | DAM Update Asset workflow | Asset Microservices (cloud functions) |

## Mutable vs Immutable Content

A fundamental AEMaaCS constraint: the `/apps`, `/libs`, `/conf` tree is **immutable** at runtime. You cannot write to it via code. Only `/content`, `/conf` (authored), and `/var` are mutable.

| Path | Mutable? | Notes |
|---|---|---|
| `/apps` | **No** | Deployed via Cloud Manager pipeline only |
| `/libs` | **No** | Adobe-managed; never modify directly |
| `/content` | Yes | Authored content |
| `/conf` | Yes (authored part) | CAConfig, editable templates |
| `/var` | Yes | Jobs, workflows, indexes (runtime data) |
| `/home` | Yes | Users and groups |

## Things That Are Banned in AEMaaCS

| Pattern | Why banned | Alternative |
|---|---|---|
| `ResourceResolverFactory.loginAdministrative()` | Security | Service users via `getServiceResourceResolver()` |
| Writing to `/apps` or `/libs` at runtime | Immutable repo | Deploy via Cloud Manager |
| Custom Lucene index configurations that block reindexing | Performance | Use property indexes where possible |
| Mutable state in the OSGi bundle classloader | Pod restarts lose it | Use JCR or external storage |
| Direct binary manipulation in workflows | Replaced by Asset Microservices | Use metadata triggers |

## Cloud Manager Pipeline

Deployments follow: Code Build → Unit Tests → Code Quality (SonarQube) → Functional Tests → Staging Deploy → Production Deploy. There is no manual FTP or package installation in production.

## AEMaaCS — Key APIs and Patterns

**Asset Compute SDK:** for custom rendition generation (replaces DAM workflow binary processing).

**Adobe I/O Events:** for event-driven integrations with external systems. Fire events from AEM, consume in external services — no long-running workflow instances waiting for API responses.

**Content Transfer Tool (CTT):** for migrating content from AEM 6.5 to AEMaaCS.

**Repository Modernization Tool:** converts mutable packages to the immutable structure required by AEMaaCS.

## Common Interview Questions

**Q: What is the biggest architectural difference between AEM 6.5 and AEMaaCS?**
Immutable repository structure. In 6.5 you could write anything anywhere at runtime. In AEMaaCS, `/apps` and `/libs` are read-only at runtime — all code must be deployed via the Cloud Manager pipeline. This fundamentally changes how you think about hotfixes, content migrations, and runtime configuration.

**Q: How do you debug an issue in AEMaaCS when you can't SSH into the server?**
Use the **Developer Console** in Adobe Cloud Manager — it provides real-time log tailing, OSGi bundle status, Sling resource resolution tools, and JVM heap dumps. You can also use the `aio` CLI for log streaming.

**Q: Can you use `session.save()` in AEMaaCS?**
Yes, it is not banned, but `resourceResolver.commit()` is preferred as it works with the Sling abstraction layer. Both work in AEMaaCS.

**Q: What is the RDE (Rapid Development Environment) in AEMaaCS?**
RDE is a fast-feedback cloud environment where you can deploy individual bundles, content packages, or Dispatcher configs without a full Cloud Manager pipeline run. It is used for iterative development and debugging.

*(For the deeper storage-layer difference — why AEMaaCS dropped MongoMK — see `oak-jcr-internals-nodestore.md`.)*
