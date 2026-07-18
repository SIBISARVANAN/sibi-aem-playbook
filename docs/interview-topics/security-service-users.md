# AEM Security — Service Users & Permissions

## Why Service Users?

In code that runs in the background (schedulers, jobs, event handlers, workflow steps), you need a JCR session to read or write content. You should never use admin credentials for this. Service users are system JCR users with minimal, least-privilege permissions.

## Creating a Service User — Three Steps

**Step 1: Create the system user in the JCR via repoinit script**

```
# ui.config/src/main/content/jcr_root/apps/mysite/osgiconfig/config/
# org.apache.sling.jcr.repoinit.RepositoryInitializer-mysite.cfg.json
{
    "scripts": [
        "create service user mysite-scheduler-service with path system/mysite",
        "create service user mysite-workflow-service with path system/mysite",
        "set ACL for mysite-scheduler-service",
        "    allow jcr:read on /content/mysite",
        "end"
    ]
}
```

**Step 2: Map the subservice name to the system user**

```json
// org.apache.sling.serviceusermapping.impl.ServiceUserMapperImpl.amended-mysite.cfg.json
{
    "user.mapping": [
        "com.mysite.core:scheduler=mysite-scheduler-service",
        "com.mysite.core:workflow-service=mysite-workflow-service"
    ]
}
```

**Step 3: Use in code**

```java
@Reference
private ResourceResolverFactory resolverFactory;

private ResourceResolver getServiceResolver(String subService) throws LoginException {
    Map<String, Object> params = new HashMap<>();
    params.put(ResourceResolverFactory.SUBSERVICE, subService);
    return resolverFactory.getServiceResourceResolver(params);
}

// Usage — always close in finally
ResourceResolver resolver = null;
try {
    resolver = getServiceResolver("scheduler");
    // do JCR work
    resolver.commit();
} catch (Exception e) {
    log.error("Error", e);
} finally {
    if (resolver != null && resolver.isLive()) {
        resolver.close();
    }
}
```

## ACL Permissions Quick Reference

| Permission | Meaning |
|---|---|
| `jcr:read` | Read node properties and children |
| `jcr:write` | Add/modify/remove nodes and properties (includes `jcr:addChildNodes`, `jcr:modifyProperties`, `jcr:removeNode`) |
| `jcr:addChildNodes` | Create child nodes |
| `jcr:modifyProperties` | Set/change properties |
| `jcr:removeNode` | Delete this node |
| `jcr:removeChildNodes` | Delete child nodes |
| `rep:write` | Shortcut for write + add + remove |
| `crx:replicate` | Required to trigger replication |

## Common Interview Questions

**Q: Why should you never use `ResourceResolverFactory.SUBSERVICE` with an admin session in production?**
Admin sessions have full JCR access. If exploited (e.g. through a path traversal vulnerability in your code), an attacker can read or write any node in the repository, including user credentials under `/home`. Service users with least-privilege ACLs limit the blast radius.

**Q: What is `loginAdministrative()` and why is it blocked?**
`ResourceResolverFactory.loginAdministrative()` returns a session with full admin rights. Adobe blocked it by default in AEM 6.2+. Any bundle using it must be whitelisted in the `LoginAdminWhitelist` OSGi config — which is itself a security red flag that interviewers will probe.

**Q: What is the difference between a service user and a system user?**
In AEM, "service user" typically refers to a JCR user created under `/home/users/system` with `createServiceUser` in repoinit. Both terms are often used interchangeably. The distinction is that service users are created by repoinit and are mapped via `ServiceUserMapper`; system users can also be created manually in CRXDE.

**Q: What is the repoinit language?**
Repository Initialisation (repoinit) is a domain-specific language processed by the `SlingRepositoryInitializer` on AEM startup. It creates users, groups, paths, and ACLs in a declarative, idempotent way. It is the standard approach for all user/permission setup in both AEM 6.5 and AEMaaCS.
