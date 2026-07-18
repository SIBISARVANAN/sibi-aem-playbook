# OSGi Service Registry

## How Multiple Implementations of a Service Are Resolved

When multiple classes implement the same OSGi service interface, the framework must pick one. By default, the implementation with the **lowest ServiceID** (the one registered first) is used. This is non-deterministic across restarts.

There are two reliable ways to control which implementation is used.

## Option A — Service Ranking

The implementation with the **highest ranking integer** wins.

```java
@Component(service = MyService.class)
@ServiceRanking(1001)
public class MyServiceImplA implements MyService { }

@Component(service = MyService.class)
@ServiceRanking(1002)
public class MyServiceImplB implements MyService { }
```

`ImplB` (ranking 1002) will always be injected wherever `MyService` is referenced, because it has the higher ranking.

**Injecting in a Sling Model:**
```java
@OsgiService
private MyService service;
```

**Injecting in another OSGi component:**
```java
@Reference
private MyService service;
```

## Option B — Named Services with Targeted Injection

Give each implementation a unique name and use an LDAP filter to select a specific one at the injection point.

```java
@Component(service = MyService.class, name = "impla")
public class MyServiceImplA implements MyService { }

@Component(service = MyService.class, name = "implb")
public class MyServiceImplB implements MyService { }
```

**Inject `ImplA` by name in a Sling Model:**
```java
@OsgiService(filter = "(component.name=impla)")
private MyService service;
```

**Inject `ImplB` by name in an OSGi component:**
```java
@Reference(target = "(component.name=implb)")
private MyService service;
```

> **When to use which:**
> Use **Service Ranking** when one implementation should always win globally.
> Use **Named Services** when different consumers legitimately need different implementations.

## OSGi Component Lifecycle — @Activate, @Modified, @Deactivate

```java
@Activate
protected void activate(MyConfig config) {
    // Called when the component is first created or its config is applied.
    // Safe to read config and initialise resources here.
}

@Modified
protected void modified(MyConfig config) {
    // Called when the OSGi config is updated at runtime (e.g. via Felix console).
    // If not declared, @Activate is called again on modification.
    // Declare @Modified separately when you need different logic (e.g. teardown + reinit).
}

@Deactivate
protected void deactivate() {
    // Called when the bundle stops or the component is deregistered.
    // Release all resources: close HTTP clients, ResourceResolvers, thread pools.
}
```

## OSGi Reference Cardinality & Policy

| Cardinality | Meaning |
|---|---|
| `MANDATORY` (default) | Exactly one — component won't start without it |
| `OPTIONAL` | Zero or one — component starts even if the service is absent |
| `MULTIPLE` | Zero or more — all matching services |
| `AT_LEAST_ONE` | One or more — at least one must be present |

| Policy | Meaning |
|---|---|
| `STATIC` (default) | Reference is bound at activation and fixed until restart |
| `DYNAMIC` | Reference can be updated at runtime — requires `synchronized` bind/unbind |

## ConfigurationPolicy

| Value | Meaning |
|---|---|
| `OPTIONAL` (default) | Component starts even with no explicit OSGi config |
| `REQUIRE` | Component only starts if an explicit config exists in `configMgr` |
| `IGNORE` | Component ignores `@Designate` — never reads config |

> **Interview trap:** Setting `ConfigurationPolicy.REQUIRE` is the right way to ensure a component doesn't start with default values in production. Interviewers often ask why a scheduler isn't running — a common root cause is a missing config file when `REQUIRE` is set.

## Common Interview Questions

**Q: What is the difference between `@Component` and `@Service`?**
`@Service` is a legacy Felix annotation (pre-DS 1.3). `@Component` from `org.osgi.service.component.annotations` is the current standard. Always use the OSGi DS annotations, never the Felix ones.

**Q: What happens if two services have the same `@ServiceRanking`?**
The one with the lower ServiceID (registered first) wins. This is non-deterministic across restarts. Always use explicit, unique rankings when order matters.

**Q: How do you verify your bundle is active?**
Check `/system/console/bundles`. A bundle in `Installed` or `Resolved` state (not `Active`) usually means an unresolved import — a missing package dependency in the bundle's manifest.

**Q: What is a bundle fragment? When would you use one?**
A fragment bundle attaches to a host bundle and contributes its classpath. Used to inject configuration or resources into a host bundle without modifying it. Rare in modern AEM development but appears in legacy setups.
