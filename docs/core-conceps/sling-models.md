# Sling Models

## Best Practice: Interface + Implementation

Always define a public interface and keep the implementation class package-private. Components and tests depend on the interface, not the implementation.

```java
@Model(
    adaptables = { Resource.class, SlingHttpServletRequest.class },
    adapters    = Author.class,
    defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL,
    resourceType = AuthorImpl.RESOURCE_TYPE
)
@Exporter(
    name       = "jackson",
    selector   = "model",
    extensions = "json",
    options    = {
        @ExporterOption(name = "SerializationFeature.WRAP_ROOT_VALUE",        value = "true"),
        @ExporterOption(name = "MapperFeature.SORT_PROPERTIES_ALPHABETICALLY", value = "true")
    }
)
@JsonRootName("AuthorDetails")
public class AuthorImpl implements Author {}
```

## Adaptables vs Adapters

| Term | Meaning | Example |
|---|---|---|
| **Adaptables** | Where the model comes from — the **input** types it can be created from | `Resource.class`, `SlingHttpServletRequest.class` |
| **Adapters** | What the model can be seen as — the **output** type it exposes | `Author.class` (the interface) |

- Use `SlingHttpServletRequest.class` when you need access to request attributes, headers, or session data.
- Use `Resource.class` when adapting from a JCR node directly (e.g. in a background context with no active request).

## Injection Strategies

`defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL` means a missing property will not cause the model to fail — the field simply remains `null`. Use `REQUIRED` only when the property is truly mandatory for the model to function.

## @PostConstruct

```java
@PostConstruct
protected void init() {
    // Runs once, after ALL field injections have completed.
    // Safe to use injected fields here.
}
```

Use `@PostConstruct` for derived fields, null-checks, and any initialization logic that depends on injected values.

## All Sling Model Injector Annotations — Quick Reference

| Annotation | What it injects | Typical use |
|---|---|---|
| `@Inject` | Any value — tries all injectors in order | General-purpose; less explicit |
| `@ValueMapValue` | A property from the resource's `ValueMap` | JCR node properties |
| `@Named("jcr:title")` | Same as above but with a custom property name | Properties with colons or different names |
| `@Default(values="x")` | Fallback value if the property is absent | Paired with any value injector |
| `@ChildResource` | A child node as a `Resource` or `List<Resource>` | Multifield dialog nodes |
| `@RequestAttribute` | A value set on the `SlingHttpServletRequest` | Cross-component data passing in JSP/HTL |
| `@ResourcePath` | Another `Resource` resolved by a stored path string | Linked resource fields in dialog |
| `@OSGiService` / `@OsgiService` | An OSGi service | Injecting services into models |
| `@Self` | The adaptable itself (request or resource) | Accessing the raw request/resource |
| `@ScriptVariable` | A variable from the HTL/JSP script bindings | `currentPage`, `wcmMode`, `resourceResolver` |
| `@SlingObject` | Core Sling objects | `ResourceResolver`, `ResourceResolverFactory`, `SlingHttpServletResponse` |

## @ChildResource — Multifield Pattern

```java
// Dialog multifield creates child nodes: jcr:content/items/item0, item1, ...
@ChildResource
private List<Resource> items;

// In @PostConstruct, adapt each child resource:
items.stream()
     .map(r -> r.adaptTo(LinkItem.class))
     .filter(Objects::nonNull)
     .collect(Collectors.toList());
```

Child models always use `adaptables = Resource.class` — individual child nodes of a multifield don't have their own `SlingHttpServletRequest`, only the top-level component resource being rendered does.

## Sling Model Delegation Pattern

Used to wrap or extend a Core Component model without copying its source:

```java
@Model(adaptables = SlingHttpServletRequest.class,
       adapters   = Teaser.class,
       defaultInjectionStrategy = DefaultInjectionStrategy.OPTIONAL)
public class CustomTeaserImpl implements Teaser {

    @Self
    @Via(type = ResourceSuperType.class)   // delegates up to the Core Component
    private Teaser delegate;

    @Override
    public String getTitle() {
        return delegate.getTitle();        // pass-through
    }
}
```

> **Interview trap:** Interviewers often ask "how do you extend a Core Component without copying its Java?" — the delegation pattern via `@Via(type = ResourceSuperType.class)` is the answer.

## JSON Export

**Q: How do you expose a Sling Model as a JSON endpoint?**
Add `@Exporter(name="jackson", extensions="json", selector="model")` to the model class. The URL becomes `<resource>.model.json`.

**Q: How do you exclude a field from JSON output?**
Annotate the getter or field with `@JsonIgnore`.

**Q: How do you rename a JSON key?**
Annotate the getter with `@JsonProperty("customName")`.

**Q: What is `WRAP_ROOT_VALUE`?**
It wraps the entire JSON output under the `@JsonRootName` value as a root key. Without it: `{"title":"x"}`. With it: `{"AuthorDetails":{"title":"x"}}`.

## Common Interview Questions

**Q: What is the difference between `@Inject` and `@ValueMapValue`?**
`@Inject` is a meta-annotation that tries all registered injectors in priority order until one succeeds. `@ValueMapValue` is explicit — it only reads from the resource's `ValueMap`. Using `@ValueMapValue` is preferred because it's predictable and faster (no injector chain traversal).

**Q: Can a Sling Model adapt from both `Resource` and `SlingHttpServletRequest`?**
Yes — list both in `adaptables`. But be careful: if you inject `@SlingObject SlingHttpServletResponse`, it is only available when adapting from a request, not from a resource. Mixing both adaptables requires `DefaultInjectionStrategy.OPTIONAL` so resource-only adaptation doesn't fail on request-only injections.

**Q: When does `@PostConstruct` fail silently?**
If `defaultInjectionStrategy = REQUIRED` and any required field is missing, the model adaptation fails before `@PostConstruct` is even called. The `adaptTo()` call returns `null`. Always check for null when using `adaptTo()`.

**Q: How do you unit-test a Sling Model?**
Use `AemContext` from the `io.wcm.testing.aem-mock` library. Register your model, load mock content, adapt, and assert:

```java
AemContext ctx = new AemContext();
ctx.addModelsForClasses(AuthorImpl.class);
ctx.load().json("/content.json", "/content");
ctx.currentResource("/content/mynode");
Author model = ctx.request().adaptTo(Author.class);
assertNotNull(model);
assertEquals("Sibi", model.getFirstName());
```

*(For the full JUnit testing workflow, see `/docs/testing/`.)*
