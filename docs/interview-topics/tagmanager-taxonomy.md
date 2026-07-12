# TagManager & Taxonomy

## Core APIs

```java
@Reference
private TagManager tagManager; // com.day.cq.tagging.TagManager

// Or adapt from ResourceResolver:
TagManager tm = resourceResolver.adaptTo(TagManager.class);

// Resolve a tag by ID
Tag tag = tm.resolve("mysite:topic/aem-development");

// Get localised title in a specific locale
String title = tag.getTitle(Locale.ENGLISH);  // "AEM Development"
String localTitle = tag.getLocalizedTitle(Locale.FRENCH); // "Développement AEM"

// Create a tag
Tag newTag = tm.createTag(
    "mysite:events/conference-2025",
    "Conference 2025",
    "Annual AEM conference",
    true  // autoSave
);

// Get all tags on a resource (stored as cq:tags String[])
Tag[] tags = tm.getTagsForSubtree(resource, false);

// Find all tagged pages
RangeIterator<Resource> results = tm.find("/content/mysite", new String[]{"mysite:topic/aem"});
```

## Tag ID Structure

Tags follow a namespace:path structure: `namespace:category/subcategory`

Example: `mysite:topic/performance` means:
- Namespace: `mysite` (stored under `/content/cq:tags/mysite/`)
- Category: `topic`
- Tag: `performance`

## Common Interview Questions

**Q: How are tags stored on a page?**
As a `String[]` property `cq:tags` on the `jcr:content` node. Each value is a tag ID string (e.g. `mysite:topic/aem`).

**Q: How do you get a human-readable tag title from a tag ID in a Sling Model?**
```java
TagManager tm = resourceResolver.adaptTo(TagManager.class);
Tag tag = tm.resolve("mysite:topic/aem");
String title = tag != null ? tag.getTitle(request.getLocale()) : "mysite:topic/aem";
```

**Q: What is the difference between `getTitle()` and `getLocalizedTitle()`?**
`getTitle(Locale)` returns the title in the given locale, falling back to the default title if no localisation is found. `getLocalizedTitle(Locale)` returns only the localised title or null — no fallback. Use `getTitle(Locale)` in production to avoid null titles.
