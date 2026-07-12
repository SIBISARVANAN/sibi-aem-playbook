# Sling Servlets

## serialVersionUID

```java
private static final long serialVersionUID = 1L;
```

`HttpServlet` implements `Serializable`, so every servlet inherits that requirement. The `serialVersionUID` is a version-control ID used during Java deserialization to verify that the class definition on the receiving end matches the one that serialized the object.

If the IDs differ at deserialization time, Java throws `java.io.InvalidClassException`. Always declare it explicitly to avoid compiler warnings and unexpected runtime errors.

## Servlet Registration: Two Approaches

### ResourceType Registration ✅ (Best Practice)

Reference in this repo: `core/src/main/java/com/sibi/aem/one/core/servlets/ResourceTypeRegistrationServlet.java`

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "sibi-aem-one/services/getDetails",
    methods       = { HttpConstants.METHOD_GET, HttpConstants.METHOD_POST },
    selectors     = "data",
    extensions    = "json"
)
public class ResourceTypeRegistrationServlet extends SlingAllMethodsServlet { }
```

- The servlet is bound to a resource type, not a path.
- A JCR node with that resource type must exist for the servlet to be reachable.
- **ACLs of that JCR node apply to the servlet** — giving you repository-level access control for free.
- This is the currently recommended approach.

### Path Registration ⚠️ (Legacy)️️

Reference in this repo: `core/src/main/java/com/sibi/aem/one/core/servlets/PathRegistrationServlet.java`

```java
@Component(service = Servlet.class)
@SlingServletPaths("/bin/sibi-aem-one/services/getDetails")
public class PathRegistrationServlet extends SlingAllMethodsServlet { }
```

- The servlet is bound to a fixed URL path.
- By default all path-registered servlets require authentication. To open one to anonymous access, add the path to the `SlingAuthenticator` OSGi config at `ui.config/.../org.apache.sling.engine.impl.auth.SlingAuthenticator.cfg.json`.
- Avoid in new development — path registration bypasses JCR ACLs and introduces security risks.

## SlingSafeMethodsServlet vs SlingAllMethodsServlet

| Class | Use when |
|---|---|
| `SlingSafeMethodsServlet` | Read-only (GET, HEAD) — idempotent operations |
| `SlingAllMethodsServlet` | Write operations (POST, PUT, DELETE) |

## Sling DataSource — Dynamic Dialog Dropdowns

A common senior-level pattern: populate a Touch UI `select` field dynamically from a servlet instead of hardcoding values in the dialog XML.

```java
@Component(service = Servlet.class)
@SlingServletResourceTypes(
    resourceTypes = "mysite/datasource/categories",
    methods       = HttpConstants.METHOD_GET
)
public class CategoriesDataSource extends SlingSafeMethodsServlet {

    @Override
    protected void doGet(SlingHttpServletRequest request,
                         SlingHttpServletResponse response) {
        List<ValueMap> options = new ArrayList<>();
        // build from JCR query, API call, or static list
        options.add(new ValueMapDecorator(Map.of("text", "Technology", "value", "tech")));
        options.add(new ValueMapDecorator(Map.of("text", "Sports",     "value", "sport")));

        DataSource ds = new SimpleDataSource(options.stream()
            .map(ValueMapResource::new)
            .iterator());
        request.setAttribute(DataSource.class.getName(), ds);
    }
}
```

In the dialog XML, point the `select` field's `datasource` to this resource type:

```xml
<datasource
    jcr:primaryType="nt:unstructured"
    sling:resourceType="mysite/datasource/categories"/>
```

## Common Interview Questions

**Q: How does Sling resolve which servlet handles a request?**
Sling uses a resolution chain: it matches on `sling:resourceType`, then `selectors`, then `extension`, then `HTTP method`. The most specific match wins. This is called **Servlet Resolution** and is documented in the Sling Servlet Resolution spec.

**Q: What is the difference between `doGet()` and `GET()`?**
`SlingSafeMethodsServlet` exposes `doGet(SlingHttpServletRequest, SlingHttpServletResponse)`. `SlingAllMethodsServlet` adds `doPost()`, `doPut()`, `doDelete()`. Never override the raw `service()` method — let the base class dispatch.

**Q: How do you return JSON from a servlet?**
```java
response.setContentType("application/json");
response.setCharacterEncoding("UTF-8");
response.getWriter().write(new Gson().toJson(myObject));
```

**Q: Why should you prefer resource-type registration over path registration?**
Path-registered servlets bypass JCR ACLs. Any authenticated user can reach them. Resource-type-registered servlets inherit the ACLs of the JCR node with that resource type, giving you repository-level access control. Path registration also has known security vulnerabilities in older AEM versions.

**Q: Can a Sling servlet be called server-side (not just via HTTP)?**
Yes — using `RequestDispatcher.include()` or `RequestDispatcher.forward()`. This is how AEM's `sling:include` HTL tag works internally.
