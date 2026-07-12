# JCR & Oak — Repository Fundamentals

## JCR Node Types You Must Know

| Node Type | Use |
|---|---|
| `cq:Page` | Every AEM page. Must have a `jcr:content` child node. |
| `cq:PageContent` | The `jcr:content` node of a page. Holds all page properties. |
| `dam:Asset` | Every asset in the DAM. Must have a `jcr:content` child. |
| `dam:AssetContent` | The `jcr:content` of an asset. |
| `nt:unstructured` | Untyped, free-form node. Used for component nodes and dialog data. |
| `nt:file` | Binary file node — requires a `jcr:content` child with `jcr:data`. |
| `sling:Folder` | Orderable folder node with resource resolution support. |
| `cq:Template` | Page template stored under `/conf` or `/libs`. |
| `rep:User` | JCR user node stored under `/home/users`. |
| `rep:Group` | JCR group node stored under `/home/groups`. |

## JCR Property Types

| Type | Java equivalent | Notes |
|---|---|---|
| `String` | `String` | Most common |
| `Long` | `Long` | Integer values |
| `Double` | `Double` | Decimal values |
| `Boolean` | `Boolean` | true/false |
| `Date` | `Calendar` | Always stored as ISO-8601 |
| `Binary` | `Binary` | For file data |
| `Path` | `String` | JCR path reference |
| `Reference` | `String` (UUID) | Hard reference — prevents deletion of the referenced node |
| `WeakReference` | `String` (UUID) | Soft reference — deletion of the referenced node is allowed |
| `Name` | `String` | JCR qualified name |

## ValueMap — Reading Properties Safely

```java
// Never do: resource.adaptTo(Node.class).getProperty("title").getString()
// Always use ValueMap — null-safe, type-converting:

ValueMap vm = resource.getValueMap();
String title       = vm.get("jcr:title", "Default Title");   // with default
Boolean isHidden   = vm.get("hideInNav", false);
Calendar modified  = vm.get("jcr:lastModified", Calendar.class);  // returns null if absent
String[] tags      = vm.get("cq:tags", String[].class);           // multi-value
```

`ValueMap` never throws for a missing property when a default (or nullable type) is used — the raw `Node`/`Property` JCR API throws `PathNotFoundException` instead, which is why `ValueMap` is always preferred in application code.

## Common Interview Questions

**Q: What is the difference between a `Reference` and a `WeakReference` property type?**
A `Reference` (hard reference) prevents the referenced node from being deleted — the JCR will throw a `ReferentialIntegrityException`. A `WeakReference` allows deletion of the referenced node; the property simply becomes a dangling reference.

**Q: What is the difference between `session.save()` and `resourceResolver.commit()`?**
They both persist changes to JCR, but `session.save()` is the raw JCR API and `resourceResolver.commit()` is the Sling API. In OSGi components, always use `resourceResolver.commit()` — it ensures the Sling lifecycle is respected and works correctly with the Sling resource provider abstraction.

**Q: What is `jcr:lastModifiedBy` and when is it set?**
It's a standard JCR mixin property (`mix:lastModified`) that records which user last modified the node. AEM sets it automatically when content is saved via the author UI. In code, if you write to JCR as a service user, `jcr:lastModifiedBy` reflects the service user name, not the human author.

**Q: What is the difference between `nt:unstructured` and `nt:base`?**
`nt:base` is the root node type — every node type extends from it. `nt:unstructured` extends `nt:base` and adds the ability to have any properties and any child nodes without a schema constraint. AEM component dialog data is stored as `nt:unstructured`.

*(For the deeper Oak architecture — NodeStore, MVCC, TarMK, MongoMK — see `oak-jcr-internals-nodestore.md`.)*
