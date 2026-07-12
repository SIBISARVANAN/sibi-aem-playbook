# Sling Context-Aware Configuration (CAConfig)

## What Is CAConfig?

CAConfig allows you to store and retrieve configuration values that vary per site, per language branch, or per content tree — without writing code that hard-codes environment checks. The configuration is stored in JCR under `/conf` and is resolved by walking up the content tree from the current resource.

## Resolution Order

Given a resource at `/content/mysite/en/about`:
1. Looks for config at `/conf/mysite` (mapped via `sling:configRef` on the page tree root)
2. Falls back to `/conf/global`
3. Falls back to defaults in the `@Configuration` annotation

## Defining a Configuration

```java
@Configuration(label = "Site Header Configuration")
public @interface HeaderConfig {
    String logoPath() default "";
    boolean enableSiteSearch() default true;
}

// Collection config — for repeating items (nav links, etc.)
@Configuration(label = "Header Nav Items", collection = true)
public @interface HeaderNavItemsConfig {
    String pageName() default "";
    String pagePath() default "";
}
```

## Reading CAConfig in a Sling Model

```java
@Model(adaptables = SlingHttpServletRequest.class)
public class HeaderModel {

    @Self
    private SlingHttpServletRequest request;

    private String logoPath;
    private List<HeaderNavItemsConfig> navItems;

    @PostConstruct
    protected void init() {
        ConfigurationBuilder cb = request.adaptTo(ConfigurationBuilder.class);
        if (cb != null) {
            HeaderConfig config = cb.as(HeaderConfig.class);
            logoPath = config.logoPath();

            navItems = cb.asCollection(HeaderNavItemsConfig.class)
                         .stream()
                         .collect(Collectors.toList());
        }
    }
}
```

## Storing a CAConfig Value in JCR

CAConfig values are stored as JCR nodes under the mapped `/conf` path:

```
/conf/mysite/sling:configs/com.mysite.core.configs.HeaderConfig
    logoPath = "/content/dam/mysite/logo.svg"
    enableSiteSearch = true
```

Set via the `ConfigurationManager` API or directly in CRXDE.

## Common Interview Questions

**Q: How is CAConfig different from OSGi config?**
OSGi config is per-environment (dev/stage/prod) and is set by operators. CAConfig is per-site/content-tree and can be set by developers or even authors. OSGi config is the right tool for environment-specific values (API keys, URLs). CAConfig is the right tool for site-specific design decisions (logo path, navigation structure, feature flags per locale).

**Q: What is `sling:configRef`?**
A property set on a content root node (e.g. the language root `/content/mysite/en`) that points to a `/conf` path. It tells CAConfig's resolver which `/conf` bucket to use when resolving configuration for that content tree.

**Q: What happens if no CAConfig is found?**
`ConfigurationBuilder.as(HeaderConfig.class)` never returns null — it returns a proxy object whose methods return the default values defined in the `@Configuration @interface`. This makes CAConfig null-safe by design.
