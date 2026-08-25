# AEM Java Annotations — Complete Reference

A categorized reference of every annotation commonly used across AEM Java development: Sling Models, OSGi Components/Services, Filters, Listeners, Event Handlers, Jobs, Schedulers, Workflow Process Steps, OSGi Configurations, Context-Aware Configurations, MBeans, and Servlets.

Non-Java annotations (Granite UI dialog XML) and JUnit/testing annotations are intentionally excluded — covered separately.

---

## Table of Contents

1. [Sling Models](#1-sling-models)
   - 1.1 [Model Definition, Adaptation & Export](#11-model-definition-adaptation--export)
   - 1.2 [Injection Source Annotations](#12-injection-source-annotations)
   - 1.3 [Injection Modifier Annotations](#13-injection-modifier-annotations)
   - 1.4 [Lifecycle Annotations](#14-lifecycle-annotations)
2. [OSGi Components](#2-osgi-components)
3. [OSGi Services](#3-osgi-services)
4. [OSGi Configurations](#4-osgi-configurations)
5. [Context-Aware Configurations](#5-context-aware-configurations)
6. [Sling Filters](#6-sling-filters)
7. [Listeners & Event Handlers](#7-listeners--event-handlers)
8. [Jobs](#8-jobs)
9. [Schedulers](#9-schedulers)
10. [Workflow Process Steps](#10-workflow-process-steps)
11. [Servlets](#11-servlets)
12. [MBeans](#12-mbeans)
13. [QueryBuilder / JCR](#13-querybuilder--jcr)
14. [General Java Annotations Used Throughout](#14-general-java-annotations-used-throughout)
15. [Quick Reference Table](#15-quick-reference-table)

---

## 1. Sling Models

### 1.1 Model Definition, Adaptation & Export

#### `@Model`
**Package**: `org.apache.sling.models.annotations.Model`
**Use case**: Class-level. Declares a POJO as a Sling Model, adaptable from `Resource` / `SlingHttpServletRequest`.

| Attribute | Purpose |
|---|---|
| `adaptables` | Classes this model can be adapted from (`Resource.class`, `SlingHttpServletRequest.class`) |
| `adapters` | Interface(s) exposed on `adaptTo()` — used when implementing an interface |
| `resourceType` | Binds model to `sling:resourceType` — required for JSON Exporter resource-type matching |
| `defaultInjectionStrategy` | `REQUIRED` or `OPTIONAL` default for all fields |
| `cache` | Caches model instance on the adaptable for request lifecycle |

```java
@Model(adaptables = {Resource.class, SlingHttpServletRequest.class},
       adapters = PropertyListingModel.class,
       resourceType = "myproject/components/propertylisting",
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class PropertyListingModelImpl implements PropertyListingModel { }
```

#### `@Exporter`
**Package**: `org.apache.sling.models.annotations.Exporter`
**Use case**: Class-level, paired with `@Model`. Registers the model with an export framework (typically Jackson JSON exporter) so it can be rendered via the `.model.json` selector.

| Attribute | Purpose |
|---|---|
| `name` | Exporter name, typically `"jackson"` |
| `extensions` | Extension the exporter responds to, typically `"json"` |
| `options` | Array of `@ExporterOption` for Jackson serialization config |

```java
@Model(adaptables = Resource.class, adapters = PropertyListingModel.class,
       resourceType = "myproject/components/propertylisting")
@Exporter(name = "jackson", extensions = "json",
          options = @ExporterOption(name = "SerializationFeature.WRITE_DATES_AS_TIMESTAMPS", value = "false"))
public class PropertyListingModelImpl implements PropertyListingModel { }
```

#### `@ExporterOption`
**Package**: `org.apache.sling.models.annotations.Exporter.ExporterOption` (nested)
**Use case**: Passes individual Jackson `ObjectMapper` feature flags into `@Exporter`. Repeatable via array.

```java
@Exporter(name = "jackson", extensions = "json", options = {
    @ExporterOption(name = "SerializationFeature.INDENT_OUTPUT", value = "true")
})
```

---

### 1.2 Injection Source Annotations

These tell the Sling Models injector framework **where** to pull a field/method's value from.

#### `@Inject`
**Package**: `org.apache.sling.models.annotations.injectorspecific.Inject`
**Use case**: Generic injection — lets the framework figure out the right injector by trying each registered injector in order (name-based, then type-based). Least explicit; prefer specific injector annotations below in real code for clarity and performance.

```java
@Inject
private String title;
```

#### `@ValueMapValue`
**Use case**: Injects a property directly from the resource's `ValueMap` (JCR property). Most common injection annotation in AEM.

```java
@ValueMapValue
private String jcrTitle;

@ValueMapValue(name = "cq:tags")
private String[] tags;
```

#### `@ChildResource`
**Use case**: Injects a child resource, or adapts it to another Sling Model — used for nested/multifield structures.

```java
@ChildResource
private Resource imageResource;

@ChildResource
private List<ListItemModel> listItems;
```

#### `@ResourcePath`
**Use case**: Injects a `Resource` (or adapts it) from an absolute or relative JCR path, typically stored as a property value (e.g. a path picker field).

```java
@ResourcePath(path = "/content/dam/myproject/banner.jpg")
private Resource bannerAsset;
```

#### `@RequestAttribute`
**Use case**: Injects a value from `SlingHttpServletRequest.getAttribute()`. Only valid when the model's `adaptables` includes `SlingHttpServletRequest`.

```java
@RequestAttribute
private String flushedFlag;
```

#### `@ScriptVariable`
**Use case**: Injects global HTL/JSP script bindings — `resource`, `currentPage`, `resourcePage`, `pageManager`, `properties`, etc. Requires request-based adaptation.

```java
@ScriptVariable
private Page currentPage;
```

#### `@SlingObject`
**Use case**: Injects Sling-provided objects: `ResourceResolver`, `Resource`, `SlingHttpServletRequest`, `SlingHttpServletResponse`.

```java
@SlingObject
private ResourceResolver resourceResolver;
```

#### `@OSGiService`
**Use case**: Injects an OSGi service directly into the model — bridges Sling Models with the OSGi service registry (e.g. injecting a custom `QueryService` or `TagManager`-style helper).

```java
@OSGiService
private QueryBuilderService queryBuilderService;
```

#### `@Self`
**Use case**: Injects the adaptable itself (e.g. the `Resource` or `SlingHttpServletRequest` being adapted), or another model adapted from the same adaptable — useful for model composition/decoration.

```java
@Self
private Resource currentResource;

@Self
private SomeOtherModel decoratedModel;
```

#### `@Via`
**Use case**: Modifies **where** an injection is sourced from before applying the injector — e.g. resolve via the resource's parent, or via a different resource type context. Often combined with `ResourceSuperType` / `ChildResource` via classes like `ForcedResourceType`, `BeanProperty`.

```java
@ValueMapValue
@Via("resourceSuperType")
private String inheritedProperty;
```

---

### 1.3 Injection Modifier Annotations

These modify the *behavior* of an injection, not the source.

#### `@Default`
**Use case**: Supplies a fallback value if injection returns null.

```java
@ValueMapValue
@Default(values = "No title provided")
private String title;

@ValueMapValue
@Default(intValues = 10)
private int pageSize;
```

#### `@Optional`
**Use case**: Marks a field as not required — injection failure won't cause model instantiation to fail. Needed when `defaultInjectionStrategy` is `REQUIRED`.

```java
@ValueMapValue
@Optional
private String subtitle;
```

#### `@Required`
**Use case**: Explicitly marks a field as mandatory — model instantiation fails (returns null on `adaptTo`) if not resolvable. Used when default strategy is `OPTIONAL` but this one field is critical.

```java
@ValueMapValue
@Required
private String sku;
```

#### `@Named`
**Use case**: Overrides the property/field name used for lookup, when the Java field name doesn't match the JCR property name.

```java
@ValueMapValue
@Named("jcr:title")
private String title;
```

#### `@Source`
**Use case**: Explicitly names which injector to use by ID (e.g. `"valuemap"`, `"script-bindings"`, `"osgi-services"`) when multiple injectors could otherwise apply — resolves ambiguity.

```java
@Inject
@Source("valuemap")
private String description;
```

---

### 1.4 Lifecycle Annotations

#### `@PostConstruct`
**Package**: `javax.annotation.PostConstruct` (or `jakarta.annotation.PostConstruct` on newer Sling Models versions)
**Use case**: Method-level. Runs after all injections are complete — used for derived/computed fields, validation, or additional setup logic that depends on injected values.

```java
@ValueMapValue
private String rawPrice;

private double formattedPrice;

@PostConstruct
protected void init() {
    this.formattedPrice = Double.parseDouble(rawPrice) * 1.18; // apply tax
}
```

---

## 2. OSGi Components

#### `@Component`
**Package**: `org.osgi.service.component.annotations.Component`
**Use case**: Class-level. Declares a class as an OSGi Declarative Services (DS) component — the foundation for services, filters, listeners, servlets, schedulers, workflow steps, etc. Generates the DS XML at build time.

| Attribute | Purpose |
|---|---|
| `service` | Interface(s) this component registers as (e.g. `Filter.class`, `Servlet.class`) |
| `immediate` | If `true`, activates on bundle start rather than lazily on first service lookup |
| `property` | Array of `"key=value"` OSGi properties (registration properties, e.g. `sling.filter.scope`) |
| `configurationPolicy` | `REQUIRE`, `OPTIONAL`, or `IGNORE` — whether an OSGi config is mandatory |
| `name` | PID used for OSGi configuration factory/lookup |

```java
@Component(service = Filter.class,
           property = {
               "sling.filter.scope=REQUEST",
               "service.ranking:Integer=100"
           },
           immediate = true)
public class CustomAuthFilter implements Filter { }
```

#### `@Activate`
**Package**: `org.osgi.service.component.annotations.Activate`
**Use case**: Method-level (or constructor-level). Called when the component is activated — used for initialization logic, reading OSGi config values.

```java
@Activate
protected void activate(MyConfig config) {
    this.threshold = config.threshold();
}
```

#### `@Deactivate`
**Use case**: Method-level. Called when component is deactivated (bundle stop, config change with `REQUIRE` policy) — used for cleanup (closing connections, unregistering listeners registered manually in code).

```java
@Deactivate
protected void deactivate() {
    if (this.listenerRegistration != null) {
        this.listenerRegistration.unregister();
    }
}
```

#### `@Modified`
**Use case**: Method-level. Called when the component's OSGi configuration changes at runtime, without a full deactivate/activate cycle — lets you react to config updates live.

```java
@Modified
protected void modified(MyConfig config) {
    this.threshold = config.threshold();
}
```

---

## 3. OSGi Services

#### `@Reference`
**Package**: `org.osgi.service.component.annotations.Reference`
**Use case**: Field-level (or method-level, or constructor-parameter-level). Declares a dependency on another OSGi service — DS injects the service instance.

| Attribute | Purpose |
|---|---|
| `cardinality` | `MANDATORY` (default), `OPTIONAL`, `MULTIPLE`, `AT_LEAST_ONE` |
| `policy` | `STATIC` (default, requires component restart on service change) or `DYNAMIC` (rebinds live) |
| `policyOption` | `GREEDY` or `RELUCTANT` — controls whether to switch to a higher-ranked service dynamically |
| `target` | LDAP filter expression to select a specific service implementation, e.g. `"(type=custom)"` |
| `service` | Explicit service interface, if it can't be inferred from field type |

```java
@Reference
private ResourceResolverFactory resourceResolverFactory;

@Reference(target = "(component.name=com.myproject.CustomQueryService)")
private QueryService queryService;

@Reference(cardinality = ReferenceCardinality.MULTIPLE, policy = ReferencePolicy.DYNAMIC)
private volatile List<CustomValidator> validators;
```

---

## 4. OSGi Configurations

#### `@ObjectClassDefinition` (OCD)
**Package**: `org.osgi.service.metatype.annotations.ObjectClassDefinition`
**Use case**: Interface-level. Defines the metadata (label, description) for an OSGi configuration shown in Felix Console / AEM Config Manager UI.

```java
@ObjectClassDefinition(name = "My Project - Search Config", description = "Configuration for search service")
public @interface SearchConfig {
    @AttributeDefinition(name = "Result Limit", description = "Max search results returned")
    int resultLimit() default 20;
}
```

#### `@AttributeDefinition`
**Use case**: Method-level, inside an OCD interface. Defines a single configurable property — label, description, default value, type, options.

| Attribute | Purpose |
|---|---|
| `name` | Display label in Config Manager |
| `description` | Help text |
| `type` | `AttributeType` enum (STRING, INTEGER, BOOLEAN, etc.) |
| `options` | Array of `@Option` for dropdown-style configs |
| `required` | Whether the field must be filled |

```java
@AttributeDefinition(name = "Enable Cache", description = "Toggles response caching", type = AttributeType.BOOLEAN)
boolean enableCache() default true;
```

#### `@Designate`
**Package**: `org.osgi.service.component.annotations.Designate`
**Use case**: Class-level, on the `@Component`. Binds the component to a specific OCD interface, so its `@Activate`/`@Modified` methods can accept the typed config object.

| Attribute | Purpose |
|---|---|
| `ocd` | The `@ObjectClassDefinition`-annotated interface class |
| `factory` | If `true`, allows multiple instances of this config (factory PID) — common for multi-site/multi-tenant configs |

```java
@Component(service = SearchService.class)
@Designate(ocd = SearchConfig.class)
public class SearchServiceImpl implements SearchService {
    @Activate
    protected void activate(SearchConfig config) {
        this.limit = config.resultLimit();
    }
}
```

---

## 5. Context-Aware Configurations

#### `@Configuration`
**Package**: `org.apache.sling.caconfig.annotation.Configuration`
**Use case**: Interface-level. Defines a Context-Aware Configuration schema — editable per content path/site via the CA Config UI (`/mnt/overlay` / `/conf` structure), used for site-specific or tenant-specific settings (e.g. per-brand API keys, per-locale flags).

```java
@Configuration(label = "Site Config", description = "Per-site runtime configuration")
public @interface SiteConfig {
    @Property(label = "Google Analytics ID")
    String gaId() default "";
}
```

#### `@Property` (CA Config variant)
**Package**: `org.apache.sling.caconfig.annotation.Property`
**Use case**: Method-level, inside a `@Configuration` interface. Configures label/description/ordering for a single config property (distinct from the deprecated Felix SCR `@Property`).

```java
@Property(label = "Max Items", description = "Number of items shown in carousel", order = 2)
int maxItems() default 5;
```

**Consuming a CA config in code:**
```java
@Self
private ConfigurationBuilder configurationBuilder;
...
SiteConfig config = configurationBuilder.as(SiteConfig.class);
String gaId = config.gaId();
```

---

## 6. Sling Filters

Modern Sling filters are just OSGi components registered as `javax.servlet.Filter` (or `jakarta.servlet.Filter`) — no filter-specific annotation exists beyond `@Component` + property keys.

#### `@Component(service = Filter.class, property = {...})`
**Use case**: Registers a servlet filter into the Sling request-processing chain.

| Property key | Purpose |
|---|---|
| `sling.filter.scope` | `REQUEST`, `INCLUDE`, `FORWARD`, `ERROR`, `COMPONENT` |
| `sling.filter.pattern` | Regex to restrict which request paths the filter applies to |
| `sling.filter.resourceTypes` | Restrict filter to specific resource types (COMPONENT scope) |
| `sling.filter.methods` | Restrict to specific HTTP methods |
| `service.ranking` | Controls execution order — lower runs first |

```java
@Component(service = Filter.class,
           property = {
               "sling.filter.scope=REQUEST",
               "sling.filter.pattern=/content/myproject/.*",
               "service.ranking:Integer=-500"
           })
public class SecurityHeaderFilter implements Filter { }
```

> Older/deprecated style you may still see in legacy code: `@SlingFilter`, `@SlingFilterScope`, `@Property` (from `org.apache.felix.scr.annotations` / `org.apache.sling.commons.osgi`) — no longer recommended, replaced by the `@Component` pattern above.

---

## 7. Listeners & Event Handlers

No dedicated annotation family — these are OSGi components registered against specific service interfaces via `@Component` property keys.

#### OSGi `EventHandler`
```java
@Component(service = EventHandler.class,
           property = {
               EventConstants.EVENT_TOPIC + "=org/apache/sling/api/resource/Resource/*"
           })
public class ResourceEventHandler implements EventHandler {
    @Override
    public void handleEvent(Event event) { }
}
```

#### `ResourceChangeListener` (JCR/Resource observation, replaces old `EventListener` JCR API in modern Sling)
```java
@Component(service = ResourceChangeListener.class,
           property = {
               ResourceChangeListener.PATHS + "=/content/myproject",
               ResourceChangeListener.CHANGES + "=ADDED",
               ResourceChangeListener.CHANGES + "=CHANGED"
           })
public class ContentChangeListener implements ResourceChangeListener {
    @Override
    public void onChange(List<ResourceChange> changes) { }
}
```

#### Raw JCR `EventListener` (older pattern, registered manually in `@Activate`, not property-driven)
```java
@Component(service = SomeCustomListenerService.class)
public class JcrObservationListener implements EventListener {
    @Reference
    private ResourceResolverFactory resolverFactory;

    private Session session;

    @Activate
    protected void activate() throws Exception {
        session = resolverFactory.getServiceResourceResolver(null).adaptTo(Session.class);
        session.getWorkspace().getObservationManager()
               .addEventListener(this, Event.NODE_ADDED, "/content/myproject", true, null, null, false);
    }

    @Override
    public void onEvent(EventIterator events) { }
}
```

---

## 8. Jobs

Sling Jobs use the same `@Component` + service-property pattern; there's no `@Job` annotation.

#### `JobConsumer`
```java
@Component(service = JobConsumer.class,
           property = { JobConsumer.PROPERTY_TOPICS + "=myproject/jobs/importAssets" })
public class AssetImportJobConsumer implements JobConsumer {
    @Override
    public JobResult process(Job job) {
        return JobResult.OK;
    }
}
```

#### `JobExecutor` (newer, preferred over `JobConsumer`)
```java
@Component(service = JobExecutor.class,
           property = { JobExecutor.PROPERTY_TOPICS + "=myproject/jobs/importAssets" })
public class AssetImportJobExecutor implements JobExecutor {
    @Override
    public JobExecutionResult process(Job job, JobExecutionContext context) {
        return context.result().succeeded();
    }
}
```

---

## 9. Schedulers

No `@Scheduled` annotation (unlike Spring). Two patterns:

#### Pattern A — `Runnable` component with `scheduler.expression` property
```java
@Component(service = Runnable.class,
           property = {
               "scheduler.expression=0 0 2 * * ?",
               "scheduler.concurrent:Boolean=false",
               "scheduler.threadPool=myproject-scheduler-pool"
           })
public class NightlyCleanupJob implements Runnable {
    @Override
    public void run() { }
}
```

#### Pattern B — Programmatic scheduling via `Scheduler` API in `@Activate`
```java
@Reference
private Scheduler scheduler;

@Activate
protected void activate() {
    ScheduleOptions options = scheduler.EXPR("0 0 2 * * ?");
    scheduler.schedule(this::runCleanup, options);
}
```

---

## 10. Workflow Process Steps

#### `WorkflowProcess` component
```java
@Component(service = WorkflowProcess.class,
           property = { "process.label=Custom Approval Step" })
public class ApprovalWorkflowStep implements WorkflowProcess {
    @Override
    public void execute(WorkItem workItem, WorkflowSession workflowSession, MetaDataMap metaDataMap) { }
}
```
`process.label` is the property that surfaces the step in the workflow model editor's process step dropdown.

---

## 11. Servlets

#### `@Component(service = Servlet.class, property = {...})` — classic path/resourceType binding
```java
@Component(service = Servlet.class,
           property = {
               "sling.servlet.resourceTypes=myproject/components/search",
               "sling.servlet.methods=GET",
               "sling.servlet.extensions=json"
           })
public class SearchServlet extends SlingSafeMethodsServlet { }
```

#### `@SlingServletPaths`
**Package**: `org.apache.sling.servlets.annotations.SlingServletPaths`
**Use case**: Cleaner, type-safe alternative to raw `property` strings — binds a servlet to explicit paths.

```java
@Component(service = Servlet.class)
@SlingServletPaths("/bin/myproject/search")
public class PathBoundSearchServlet extends SlingAllMethodsServlet { }
```

#### `@SlingServletResourceTypes`
**Package**: `org.apache.sling.servlets.annotations.SlingServletResourceTypes`
**Use case**: Type-safe alternative for resource-type + selector + extension + method binding.

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "myproject/components/search",
    methods = "GET",
    extensions = "json",
    selectors = "results")
public class SearchResultsServlet extends SlingSafeMethodsServlet { }
```

#### `@SlingServletFilter`
**Package**: `org.apache.sling.servlets.annotations.SlingServletFilter`
**Use case**: Type-safe annotation form of filter registration (alternative to raw `sling.filter.*` properties), pairs with `@Component(service = Filter.class)`.

---

## 12. MBeans

Annotation usage here is sparse — most AEM/Sling MBeans are exposed via plain interface-naming convention (`FooMBean`) rather than annotations. Where annotations are used:

#### `@Component(service = DynamicMBean.class, property = {"jmx.objectname=..."})`
**Use case**: Registers a class as a JMX MBean discoverable in the OSGi/Felix Web Console → JMX or standard JMX tooling (e.g. for exposing cache stats, queue depth, custom health metrics).

```java
@Component(service = DynamicMBean.class,
           property = { "jmx.objectname=com.myproject:type=ImportQueueStats" })
public class ImportQueueStatsMBean extends AnnotatedStandardMBean implements ImportQueueStatsMBeanInterface {
    public ImportQueueStatsMBean() throws NotCompliantMBeanException {
        super(ImportQueueStatsMBeanInterface.class);
    }
}
```

#### `javax.management.MXBean`
**Use case**: Marks an interface as an MXBean (a JMX MBean variant with automatic open-type data mapping — useful when exposing complex types like `List`/`Map` without manual `CompositeData` handling).

```java
@MXBean
public interface ImportQueueStatsMXBean {
    int getPendingCount();
}
```

> **Interview note**: MBeans in AEM are more often used for **custom health checks / metrics exposure** than business logic — expect this to come up if you mention monitoring or Felix Inventory Console work.

---

## 13. QueryBuilder / JCR

QueryBuilder itself is **not annotation-driven** — predicates are built programmatically (`PredicateGroup`, `Map<String,String>` params) or via query string syntax. No dedicated annotations exist here. Worth stating this explicitly in an interview if asked, so it doesn't look like a gap in your answer.

JCR-related annotations that do appear in AEM code:
- None directly on JCR API calls — `Session`, `Node`, `Property` are interacted with programmatically, not annotation-configured.

---

## 14. General Java Annotations Used Throughout

These aren't AEM-specific but appear constantly in AEM codebases:

| Annotation | Package | Use case |
|---|---|---|
| `@Override` | `java.lang` | Marks interface/superclass method implementation — compiler-checked |
| `@Deprecated` | `java.lang` | Flags legacy APIs (common in AEM given frequent API evolution, e.g. old `Property`-based OSGi annotations) |
| `@SuppressWarnings` | `java.lang` | Suppresses compiler warnings (e.g. unchecked casts when adapting resources) |
| `@FunctionalInterface` | `java.lang` | Marks single-abstract-method interfaces, e.g. custom callback interfaces used with `ResourceResolverFactory` service-user patterns |
| `@Nonnull` / `@Nullable` | `javax.annotation` / `org.jetbrains.annotations` | Nullability contracts — common on Sling API method signatures |
| `@NotNull` | Bean Validation / JetBrains | Similar nullability annotation, framework-dependent |

---

## 15. Quick Reference Table

| Category | Core annotations |
|---|---|
| Sling Models — definition | `@Model`, `@Exporter`, `@ExporterOption` |
| Sling Models — injection source | `@Inject`, `@ValueMapValue`, `@ChildResource`, `@ResourcePath`, `@RequestAttribute`, `@ScriptVariable`, `@SlingObject`, `@OSGiService`, `@Self`, `@Via` |
| Sling Models — modifiers | `@Default`, `@Optional`, `@Required`, `@Named`, `@Source` |
| Sling Models — lifecycle | `@PostConstruct` |
| OSGi component | `@Component`, `@Activate`, `@Deactivate`, `@Modified` |
| OSGi service | `@Reference` |
| OSGi config | `@ObjectClassDefinition`, `@AttributeDefinition`, `@Designate` |
| CA Config | `@Configuration`, `@Property` |
| Filters | `@Component(service=Filter.class, property={sling.filter.*})` |
| Listeners/Events | `@Component(service=EventHandler.class\|ResourceChangeListener.class)` |
| Jobs | `@Component(service=JobConsumer.class\|JobExecutor.class)` |
| Schedulers | `@Component(service=Runnable.class, property={scheduler.expression})` |
| Workflow steps | `@Component(service=WorkflowProcess.class)` |
| Servlets | `@Component(service=Servlet.class)`, `@SlingServletPaths`, `@SlingServletResourceTypes`, `@SlingServletFilter` |
| MBeans | `@Component(service=DynamicMBean.class)`, `@MXBean` |
| QueryBuilder/JCR | No dedicated annotations — programmatic API |
| General Java | `@Override`, `@Deprecated`, `@SuppressWarnings`, `@FunctionalInterface`, `@Nonnull`/`@Nullable` |

---

*Reference compiled for interview prep and for inclusion in `sibi-aem-one` → `docs/reference/`.*
