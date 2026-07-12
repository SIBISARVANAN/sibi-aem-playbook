# OSGi Configuration Registry

This section covers how to manage multiple factory instances of a service (e.g. one reCAPTCHA config per site) and the two lifecycle patterns for maintaining a runtime registry of those instances.

## Pattern Overview

**Reference files in this repo:**
- `v1`: `core/src/main/java/com/sibi/aem/one/core/services/impl/v1/GoogleRecaptchaConfigServiceImpl.java` — Container-managed
- `v2`: `core/src/main/java/com/sibi/aem/one/core/services/impl/v2/GoogleRecaptchaConfigServiceImpl.java` — Application-managed

## Container-Managed Lifecycle (v1 — Bind/Unbind Pattern)

This is the **OSGi Service Tracker Pattern** (DS-based). The OSGi container watches the service registry and calls your `bind()` / `unbind()` methods whenever a matching service appears or disappears. **You only react — the container decides when.**

```java
@Reference(
    service     = GoogleRecaptchaConfigService.class,
    cardinality = ReferenceCardinality.MULTIPLE,
    policy      = ReferencePolicy.DYNAMIC
)
protected void bind(GoogleRecaptchaConfigService service) {
    REGISTRY.put(service.getSiteName(), service);
}

protected void unbind(GoogleRecaptchaConfigService service) {
    REGISTRY.remove(service.getSiteName());
}
```

The container handles:
- Watching the service registry for new/removed instances
- Deciding when `bind()` / `unbind()` are called
- Enforcing ordering, ranking, and thread safety
- Dynamic hot-swap when a config changes at runtime

You are just reacting to container events — that's why it's called container-managed.

## Application-Managed Lifecycle (v2 — Self-Registration Pattern)

Each factory instance registers itself into a static map on activation and removes itself on deactivation. **Your application code owns the lifecycle.**

```java
private static final Map<String, GoogleRecaptchaConfigService> REGISTRY =
    new ConcurrentHashMap<>();

@Activate
@Modified
public void activate(GoogleRecaptchaConfig config) {
    siteName = config.siteName();
    // ...
    REGISTRY.put(siteName, this);   // application registers itself
}

@Deactivate
public void deactivate() {
    REGISTRY.remove(siteName);      // application removes itself
}
```

Your code owns:
- When to register and deregister
- Where instances are stored
- Thread safety of the map (use `ConcurrentHashMap`)
- Handling config updates (`@Modified` must re-register)

The container only creates the object and calls `@Activate`/`@Deactivate` — everything else is your responsibility. That's why it's application-managed.

## TL;DR Comparison

| Concept | Static Registry (v2 — App-managed) | Bind/Unbind (v1 — Container-managed) |
|---|---|---|
| Who tracks instances? | Your code | OSGi runtime |
| Who decides when to add/remove? | You (`@Activate` / `@Deactivate`) | Container (`bind()` / `unbind()`) |
| Failure handling | You write it | Container handles it |
| Thread safety | You manage | OSGi ensures |
| Dynamic hot-swap | Harder — you implement it | Built-in |
| Code complexity | Simpler | Slightly more boilerplate |

> Use **v1 (container-managed)** when hot-swap correctness and thread safety guarantees matter (multi-bundle, production-critical).
> Use **v2 (application-managed)** for simpler cases where all factory instances live in the same bundle.

## OSGi Config File Naming — Run Mode Targeting

OSGi config files in `ui.config` are placed in folders named by run mode. This is how you have different values per environment without code changes.

```
ui.config/src/main/content/jcr_root/apps/mysite/osgiconfig/
├── config/                        ← applies to ALL run modes
│   └── com.example.MyService.cfg.json
├── config.author/                 ← author only
│   └── com.example.MyService.cfg.json
├── config.publish/                ← publish only
│   └── com.example.MyService.cfg.json
├── config.dev/                    ← dev environment only
│   └── com.example.MyService.cfg.json
├── config.stage/                  ← stage only
│   └── com.example.MyService.cfg.json
└── config.prod/                   ← prod only
    └── com.example.MyService.cfg.json
```

**Factory config naming** — for `@Designate(factory=true)`, the file name must include a unique identifier after the PID, separated by a tilde:

```
com.example.GoogleRecaptchaConfigServiceImpl~site1.cfg.json
com.example.GoogleRecaptchaConfigServiceImpl~site2.cfg.json
```

## Common Interview Questions

**Q: How do you have different database URLs for dev, stage, and prod without changing code?**
Place the same `@ObjectClassDefinition` config file in `config.dev/`, `config.stage/`, and `config.prod/` folders under `ui.config`, each with the appropriate value. OSGi reads the most specific matching folder.

**Q: What is the underscore-to-dot naming rule?**
In `@interface Config`, method names use underscores as separators (Java doesn't allow dots in method names). OSGi maps `my_property()` to the key `"my.property"` in the `.cfg.json` file. Example: `scheduler_expression()` → `"scheduler.expression"`.

**Q: What is a factory configuration vs a singleton configuration?**
A singleton config (`@Designate(factory=false)`, the default) allows exactly one instance of the component. A factory config (`@Designate(factory=true)`) allows multiple named instances — one per `.cfg.json` file with a unique tilde suffix.
