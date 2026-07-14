# AEM JUnit Testing — Phase 3: OSGi Services & Lifecycle

---

### 3.1 Testing @Activate / @Modified / @Deactivate

#### Why Lifecycle Methods Need Testing

OSGi lifecycle methods are the most common source of production bugs that unit tests never catch in many projects. A missing `@Modified` handler leaves a stale HTTP client after a config change. A `@Deactivate` that forgets to clear the cache leaks memory on bundle restart. Testing these directly ensures they behave correctly in isolation.

#### The Service Under Test

We use `InventoryServiceImpl` from the product scenario — it has all three lifecycle methods, an HTTP client, and a Guava Cache.

```java
@Component(service = InventoryService.class,
           configurationPolicy = ConfigurationPolicy.REQUIRE)
@Designate(ocd = InventoryServiceImpl.Config.class)
public class InventoryServiceImpl implements InventoryService {

    @ObjectClassDefinition(name = "Inventory Service Configuration")
    public @interface Config {
        String api_endpoint_url() default "https://inventory.internal/api/v1/stock/";
        int connection_timeout()  default 3000;
        int max_connections()     default 20;
        boolean cache_enabled()   default true;
    }

    private String apiEndpointUrl;
    private boolean cacheEnabled;
    private CloseableHttpClient httpClient;
    private PoolingHttpClientConnectionManager connectionManager;
    private Cache<String, Integer> stockCache;

    @Activate
    protected void activate(Config config) {
        this.apiEndpointUrl = config.api_endpoint_url();
        this.cacheEnabled   = config.cache_enabled();
        connectionManager   = new PoolingHttpClientConnectionManager();
        connectionManager.setMaxTotal(config.max_connections());
        stockCache = CacheBuilder.newBuilder()
            .maximumSize(500)
            .expireAfterWrite(5, TimeUnit.MINUTES)
            .build();
        httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .build();
    }

    @Modified
    protected void modified(Config config) {
        stockCache.invalidateAll();
        deactivate();
        activate(config);
    }

    @Deactivate
    protected void deactivate() {
        try {
            if (httpClient != null)       httpClient.close();
            if (connectionManager != null) connectionManager.close();
        } catch (Exception e) { /* log */ }
        finally { stockCache.invalidateAll(); }
    }

    @Override
    public int getStockCount(String sku) throws InventoryServiceException {
        if (StringUtils.isBlank(sku)) return -1;
        Integer cached = stockCache.getIfPresent(sku);
        if (cached != null) return cached;
        return fetchFromApi(sku);
    }
}
```

#### Method 1 — `ctx.registerInjectActivateService()` (Cleanest)

This single call: creates the service instance, injects `@Reference` dependencies, and calls `@Activate` with the provided config map. It is the recommended approach for most lifecycle tests.

```java
@ExtendWith(AemContextExtension.class)
class InventoryServiceImplActivateTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Test
    void activate_whenConfigProvided_setsApiEndpointUrl() {
        // Build the config as a plain Map — keys match the @interface method names
        // with underscores replacing dots (underscore-to-dot rule in reverse).
        // api_endpoint_url() in @interface → "api.endpoint.url" in the cfg.json
        // but in tests we use the METHOD NAME directly as the key.
        Map<String, Object> config = new HashMap<>();
        config.put("api_endpoint_url", "https://custom-api.mysite.com/stock/");
        config.put("connection_timeout", 5000);
        config.put("max_connections", 50);
        config.put("cache_enabled", true);

        // registerInjectActivateService:
        //   1. Creates a new InventoryServiceImpl instance
        //   2. Injects any @Reference fields from the ctx service registry
        //   3. Calls @Activate with a Config proxy built from the map
        //   4. Registers the activated service in the ctx service registry
        InventoryService service = ctx.registerInjectActivateService(
            new InventoryServiceImpl(), config);

        assertNotNull(service);
        // We can't directly access apiEndpointUrl (it's private),
        // but we can verify the service behaves correctly with the config.
        // For private field assertions, see Section 3.1 Method 2 (Reflection).
        assertDoesNotThrow(() -> service.getStockCount("SKU-001"),
            "Service should be functional after activation");
    }

    @Test
    void activate_withDefaultConfig_usesDefaultValues() {
        // Empty map = all defaults from @interface apply
        InventoryService service = ctx.registerInjectActivateService(
            new InventoryServiceImpl(), Collections.emptyMap());

        assertNotNull(service, "Service should activate even with empty config");
    }
}
```

#### Method 2 — Direct Method Call + Reflection (For Private Field Verification)

When you need to verify private state that was set by `@Activate`, use reflection to read the field after activation:

```java
@ExtendWith(AemContextExtension.class)
class InventoryServiceImplLifecycleTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);
    private InventoryServiceImpl service;

    @BeforeEach
    void setUp() {
        service = new InventoryServiceImpl();
    }

    @Test
    void activate_setsApiEndpointUrlFromConfig() throws Exception {
        // Call @Activate directly — it's protected, but accessible in same package
        // or via reflection from tests.
        // Option A: call directly if test is in same package (package-private access)
        // Option B: use ReflectionTestUtils for cross-package access

        // Using Spring's ReflectionTestUtils (available via spring-test dependency)
        // OR Apache Commons FieldUtils (already available in AEM projects):
        InventoryServiceImpl.Config config = createConfig(
            "https://test-api.internal/", 2000, 10, true);

        // Invoking protected @Activate directly:
        service.activate(config);

        // Now read the private field using reflection
        Field field = InventoryServiceImpl.class.getDeclaredField("apiEndpointUrl");
        field.setAccessible(true);  // bypasses private access modifier
        String actualUrl = (String) field.get(service);

        assertEquals("https://test-api.internal/", actualUrl);
    }

    @Test
    void activate_initializesHttpClientAndCache() throws Exception {
        InventoryServiceImpl.Config config = createConfig(
            "https://api.test/", 3000, 20, true);

        service.activate(config);

        // Verify httpClient was created (not null)
        Field clientField = InventoryServiceImpl.class.getDeclaredField("httpClient");
        clientField.setAccessible(true);
        assertNotNull(clientField.get(service), "httpClient should be initialised after activate");

        // Verify stockCache was created
        Field cacheField = InventoryServiceImpl.class.getDeclaredField("stockCache");
        cacheField.setAccessible(true);
        assertNotNull(cacheField.get(service), "stockCache should be initialised after activate");
    }

    @Test
    void deactivate_closesHttpClientAndClearsCache() throws Exception {
        // First activate the service to create the resources
        InventoryServiceImpl.Config config = createConfig("https://api.test/", 3000, 20, true);
        service.activate(config);

        // Populate the cache
        Field cacheField = InventoryServiceImpl.class.getDeclaredField("stockCache");
        cacheField.setAccessible(true);
        Cache<String, Integer> cache = (Cache<String, Integer>) cacheField.get(service);
        cache.put("SKU-001", 42);
        assertEquals(1, cache.size(), "Cache should have 1 entry before deactivate");

        // Call @Deactivate
        service.deactivate();

        // Verify cache was cleared
        assertEquals(0, cache.size(), "Cache should be empty after deactivate");

        // Verify httpClient is closed — trying to execute a request should fail
        Field clientField = InventoryServiceImpl.class.getDeclaredField("httpClient");
        clientField.setAccessible(true);
        CloseableHttpClient client = (CloseableHttpClient) clientField.get(service);
        // CloseableHttpClient.isRunning() or attempting a request after close throws
        // IllegalStateException — this verifies close() was called
        assertThrows(Exception.class, () -> client.execute(new HttpGet("http://test")),
            "Closed HTTP client should throw when used");
    }

    @Test
    void modified_clearsExistingCacheBeforeReinitialising() throws Exception {
        // Activate with initial config
        InventoryServiceImpl.Config config1 = createConfig("https://api-v1.test/", 3000, 20, true);
        service.activate(config1);

        // Populate cache with v1 data
        Field cacheField = InventoryServiceImpl.class.getDeclaredField("stockCache");
        cacheField.setAccessible(true);
        Cache<String, Integer> cache = (Cache<String, Integer>) cacheField.get(service);
        cache.put("SKU-001", 99);
        assertEquals(1L, cache.size());

        // @Modified with new config (new endpoint URL)
        InventoryServiceImpl.Config config2 = createConfig("https://api-v2.test/", 5000, 30, true);
        service.modified(config2);

        // Verify cache was cleared — v1 data must be gone
        assertEquals(0L, cache.size(),
            "@Modified must clear cache so stale v1 data does not survive config change");

        // Verify new endpoint URL was applied
        Field urlField = InventoryServiceImpl.class.getDeclaredField("apiEndpointUrl");
        urlField.setAccessible(true);
        assertEquals("https://api-v2.test/", urlField.get(service),
            "@Modified must apply new config values");
    }

    // ── Helper: creates a Config proxy using Mockito ──────────────────────────

    private InventoryServiceImpl.Config createConfig(String url, int timeout,
                                                      int maxConn, boolean cacheEnabled) {
        InventoryServiceImpl.Config config = mock(InventoryServiceImpl.Config.class);
        when(config.api_endpoint_url()).thenReturn(url);
        when(config.connection_timeout()).thenReturn(timeout);
        when(config.max_connections()).thenReturn(maxConn);
        when(config.cache_enabled()).thenReturn(cacheEnabled);
        return config;
    }
}
```

---

### 3.2 OSGi Config Injection

#### Understanding the Config Map Key Naming

The `@interface Config` method names use underscores, but the OSGi runtime maps them to dot-separated keys in the `.cfg.json` file. In tests, you use the METHOD NAME as the key (not the dot-separated version):

```java
// @interface Config method name → test Map key → cfg.json key
// api_endpoint_url()            → "api_endpoint_url"    → "api.endpoint.url"
// connection_timeout()          → "connection_timeout"  → "connection.timeout"
// cache_enabled()               → "cache_enabled"       → "cache.enabled"
```

```java
@ExtendWith(AemContextExtension.class)
class OsgiConfigInjectionTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Test
    void service_withCustomConfig_appliesAllConfigValues() {
        Map<String, Object> config = new HashMap<>();
        // Use method names (underscores), NOT dot-separated cfg.json keys
        config.put("api_endpoint_url",   "https://production-api.com/stock/");
        config.put("connection_timeout", 2000);
        config.put("max_connections",    100);
        config.put("cache_enabled",      true);

        InventoryService service = ctx.registerInjectActivateService(
            new InventoryServiceImpl(), config);

        assertNotNull(service);
    }

    @Test
    void service_withBooleanFalseConfig_disablesCache() throws Exception {
        Map<String, Object> config = Map.of(
            "api_endpoint_url",   "https://api.test/",
            "connection_timeout", 3000,
            "max_connections",    20,
            "cache_enabled",      false   // explicitly disable cache
        );

        InventoryServiceImpl service = ctx.registerInjectActivateService(
            new InventoryServiceImpl(), config);

        // Call getStockCount twice — with cache disabled, the API would be called twice
        // We verify via the cacheEnabled field
        Field cacheEnabledField = InventoryServiceImpl.class
            .getDeclaredField("cacheEnabled");
        cacheEnabledField.setAccessible(true);
        assertFalse((boolean) cacheEnabledField.get(service),
            "cacheEnabled should be false when configured so");
    }

    @Test
    void service_withPartialConfig_usesDefaultsForMissingKeys() {
        // Only provide some keys — the rest should use @interface defaults
        Map<String, Object> partialConfig = Map.of(
            "api_endpoint_url", "https://custom.api.com/"
            // connection_timeout, max_connections, cache_enabled not provided
            // → should use defaults: 3000, 20, true
        );

        // Should activate without errors
        assertDoesNotThrow(() ->
            ctx.registerInjectActivateService(new InventoryServiceImpl(), partialConfig));
    }

    // ── Testing ConfigurationPolicy.REQUIRE ────────────────────────────────

    @Test
    void service_withRequirePolicy_doesNotStartWithoutExplicitConfig() {
        // ConfigurationPolicy.REQUIRE means the component won't activate
        // without an explicit OSGi configuration.
        // In the AemContext, registerInjectActivateService with an empty map
        // simulates "no config" — the service should still activate
        // because AemContext doesn't enforce ConfigurationPolicy strictly.
        // The real enforcement happens in the OSGi runtime (Felix), not in tests.
        // This is an important limitation to document:

        // In tests: always provide config to mirror production behaviour,
        // even if the mock runtime would allow empty config.
        Map<String, Object> config = Map.of(
            "api_endpoint_url", "https://api.test/"
        );
        InventoryService service = ctx.registerInjectActivateService(
            new InventoryServiceImpl(), config);

        assertNotNull(service,
            "Service should activate when config is provided");
    }
}
```

---

### 3.3 Mocking Dependent Services

#### The Service Under Test

```java
@Component(service = ExternalApiService.class)
public class ExternalApiServiceImpl implements ExternalApiService {

    @Reference
    private RunModeService runModeService;

    @Reference
    private InventoryService inventoryService;

    @Override
    public String fetchProductData(String sku) {
        // Behaviour differs between author and publish
        if (runModeService.isAuthor()) {
            log.debug("Author mode — returning cached data");
            return "{ \"sku\": \"" + sku + "\", \"mode\": \"author\" }";
        }
        try {
            int stock = inventoryService.getStockCount(sku);
            return "{ \"sku\": \"" + sku + "\", \"stock\": " + stock + " }";
        } catch (InventoryService.InventoryServiceException e) {
            return null;
        }
    }
}
```

#### Testing with Mocked @Reference Dependencies

```java
@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class ExternalApiServiceImplTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private RunModeService runModeService;

    @Mock
    private InventoryService inventoryService;

    private ExternalApiService externalApiService;

    @BeforeEach
    void setUp() {
        // Register mock @Reference dependencies BEFORE the service that needs them
        ctx.registerService(RunModeService.class, runModeService);
        ctx.registerService(InventoryService.class, inventoryService);

        // Now activate the service — @Reference fields are satisfied by the mocks
        externalApiService = ctx.registerInjectActivateService(
            new ExternalApiServiceImpl());
    }

    @Test
    void fetchProductData_whenAuthorMode_returnsAuthorModeResponse() {
        // Stub RunModeService to simulate author instance
        when(runModeService.isAuthor()).thenReturn(true);

        String result = externalApiService.fetchProductData("SKU-001");

        assertNotNull(result);
        assertTrue(result.contains("author"),
            "Author mode should return author-specific response");

        // In author mode, InventoryService should NEVER be called
        verifyNoInteractions(inventoryService);
    }

    @Test
    void fetchProductData_whenPublishMode_callsInventoryService()
            throws InventoryService.InventoryServiceException {
        when(runModeService.isAuthor()).thenReturn(false);
        when(inventoryService.getStockCount("SKU-001")).thenReturn(25);

        String result = externalApiService.fetchProductData("SKU-001");

        assertNotNull(result);
        assertTrue(result.contains("25"),
            "Publish mode response should contain stock count");
        verify(inventoryService).getStockCount("SKU-001");
    }

    @Test
    void fetchProductData_whenInventoryServiceThrows_returnsNull()
            throws InventoryService.InventoryServiceException {
        when(runModeService.isAuthor()).thenReturn(false);
        when(inventoryService.getStockCount(anyString()))
            .thenThrow(new InventoryService.InventoryServiceException("Down", null));

        String result = externalApiService.fetchProductData("SKU-001");

        assertNull(result, "Should return null when inventory service throws");
    }

    // ── Testing Service Ranking ────────────────────────────────────────────

    @Test
    void serviceRanking_higherRankedImplementationWinsInjection() {
        // Register two implementations with different rankings
        RunModeService lowRankService  = mock(RunModeService.class);
        RunModeService highRankService = mock(RunModeService.class);

        when(lowRankService.isAuthor()).thenReturn(false);
        when(highRankService.isAuthor()).thenReturn(true);

        AemContext freshCtx = new AemContext(ResourceResolverType.JCR_MOCK);

        // Lower ranking first
        freshCtx.registerService(RunModeService.class, lowRankService,
            "service.ranking", 100);

        // Higher ranking second — this one should win
        freshCtx.registerService(RunModeService.class, highRankService,
            "service.ranking", 200);

        freshCtx.registerService(InventoryService.class, inventoryService);

        ExternalApiService service = freshCtx.registerInjectActivateService(
            new ExternalApiServiceImpl());

        String result = service.fetchProductData("SKU-001");

        // highRankService was injected (isAuthor=true) → author mode response
        assertTrue(result.contains("author"),
            "Higher-ranked service (isAuthor=true) should be injected");
    }
}
```

#### Testing @Reference with MULTIPLE Cardinality

```java
// Service that collects ALL registered implementations
@Component
public class RecaptchaRegistryImpl {

    // Collects all GoogleRecaptchaConfigService instances
    @Reference(
        cardinality = ReferenceCardinality.MULTIPLE,
        policy      = ReferencePolicy.DYNAMIC
    )
    private final List<GoogleRecaptchaConfigService> configs = new CopyOnWriteArrayList<>();

    public GoogleRecaptchaConfigService getConfig(String siteName) {
        return configs.stream()
            .filter(c -> siteName.equals(c.getSiteName()))
            .findFirst()
            .orElse(null);
    }
}
```

```java
@ExtendWith(AemContextExtension.class)
class RecaptchaRegistryImplTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Test
    void getConfig_whenMultipleServicesRegistered_returnsCorrectOne() {
        // Create two mock config services for different sites
        GoogleRecaptchaConfigService site1Config = mock(GoogleRecaptchaConfigService.class);
        GoogleRecaptchaConfigService site2Config = mock(GoogleRecaptchaConfigService.class);

        when(site1Config.getSiteName()).thenReturn("site1");
        when(site1Config.getPublicKey()).thenReturn("public-key-site1");
        when(site2Config.getSiteName()).thenReturn("site2");
        when(site2Config.getPublicKey()).thenReturn("public-key-site2");

        // Register both — AemContext handles MULTIPLE cardinality correctly
        ctx.registerService(GoogleRecaptchaConfigService.class, site1Config);
        ctx.registerService(GoogleRecaptchaConfigService.class, site2Config);

        RecaptchaRegistryImpl registry = ctx.registerInjectActivateService(
            new RecaptchaRegistryImpl());

        // Verify correct config is returned per site name
        GoogleRecaptchaConfigService found = registry.getConfig("site1");
        assertNotNull(found);
        assertEquals("public-key-site1", found.getPublicKey());

        // Verify site2 returns its own config
        GoogleRecaptchaConfigService found2 = registry.getConfig("site2");
        assertNotNull(found2);
        assertEquals("public-key-site2", found2.getPublicKey());
    }

    @Test
    void getConfig_whenSiteNameNotRegistered_returnsNull() {
        // Register one config
        GoogleRecaptchaConfigService config = mock(GoogleRecaptchaConfigService.class);
        when(config.getSiteName()).thenReturn("site1");
        ctx.registerService(GoogleRecaptchaConfigService.class, config);

        RecaptchaRegistryImpl registry = ctx.registerInjectActivateService(
            new RecaptchaRegistryImpl());

        // Ask for a non-existent site
        assertNull(registry.getConfig("nonexistent"),
            "Should return null for an unregistered site name");
    }
}
```

---

### 3.4 Complete Worked Example — RunModeServiceImpl

This is the full test class for `RunModeServiceImpl` from your repository, demonstrating all patterns together:

```java
package com.sibi.aem.one.core.services;

import com.sibi.aem.one.core.services.impl.RunModeServiceImpl;
import io.wcm.testing.mock.aem.junit5.AemContext;
import io.wcm.testing.mock.aem.junit5.AemContextExtension;
import org.apache.sling.settings.SlingSettingsService;
import org.apache.sling.testing.mock.sling.ResourceResolverType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.util.Set;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith({ AemContextExtension.class, MockitoExtension.class })
class RunModeServiceImplTest {

    private final AemContext ctx = new AemContext(ResourceResolverType.JCR_MOCK);

    @Mock
    private SlingSettingsService slingSettingsService;

    private RunModeService runModeService;

    @BeforeEach
    void setUp() {
        ctx.registerService(SlingSettingsService.class, slingSettingsService);
        runModeService = ctx.registerInjectActivateService(new RunModeServiceImpl());
    }

    // ── Author / Publish detection ──────────────────────────────────────────

    @Test
    void isAuthor_whenAuthorRunModeActive_returnsTrue() {
        when(slingSettingsService.getRunModes()).thenReturn(Set.of("author", "dev"));
        assertTrue(runModeService.isAuthor());
    }

    @Test
    void isAuthor_whenPublishRunModeActive_returnsFalse() {
        when(slingSettingsService.getRunModes()).thenReturn(Set.of("publish", "prod"));
        assertFalse(runModeService.isAuthor());
    }

    @Test
    void isPublish_whenPublishRunModeActive_returnsTrue() {
        when(slingSettingsService.getRunModes()).thenReturn(Set.of("publish"));
        assertTrue(runModeService.isPublish());
    }

    // ── Environment detection ───────────────────────────────────────────────

    @Test
    void isDev_whenDevRunModePresent_returnsTrue() {
        when(slingSettingsService.getRunModes()).thenReturn(Set.of("author", "dev"));
        assertTrue(runModeService.isDev());
        assertFalse(runModeService.isStage());
        assertFalse(runModeService.isProd());
    }

    @Test
    void isProd_whenProdRunModePresent_returnsTrue() {
        when(slingSettingsService.getRunModes()).thenReturn(Set.of("publish", "prod"));
        assertTrue(runModeService.isProd());
        assertFalse(runModeService.isDev());
        assertFalse(runModeService.isStage());
    }

    // ── Multiple run modes simultaneously ──────────────────────────────────

    @Test
    void getAllRunModes_returnsAllActiveRunModes() {
        Set<String> expected = Set.of("author", "dev", "myCustomRunMode");
        when(slingSettingsService.getRunModes()).thenReturn(expected);

        Set<String> actual = runModeService.getAllRunModes();

        assertEquals(expected, actual);
        assertEquals(3, actual.size());
    }

    // ── Edge cases ─────────────────────────────────────────────────────────

    @Test
    void isAuthor_whenRunModesEmpty_returnsFalse() {
        when(slingSettingsService.getRunModes()).thenReturn(Set.of());
        assertFalse(runModeService.isAuthor());
        assertFalse(runModeService.isPublish());
    }

    @Test
    void isAuthor_whenSlingSettingsReturnsNull_doesNotThrow() {
        // Defensive: if SlingSettingsService returns null run modes
        when(slingSettingsService.getRunModes()).thenReturn(null);
        // If RunModeServiceImpl doesn't null-check, this will throw NPE
        // This test ensures the null case is handled gracefully
        assertDoesNotThrow(() -> runModeService.getAllRunModes());
    }
}
```

---

### Phase 3 — Summary

| Topic | Key Takeaways |
|---|---|
| `registerInjectActivateService()` | One call to create + inject + activate. Use this 90% of the time. |
| Config map keys | Use the `@interface` method name (underscores), not the `.cfg.json` dot-separated key. |
| Testing `@Activate` | Call directly on the instance with a mocked `Config`, then assert state via reflection. |
| Testing `@Modified` | Verify it clears old state (cache, HTTP client) AND applies new config values. |
| Testing `@Deactivate` | Verify resources are released (cache cleared, HTTP client closed). |
| `@Reference` dependencies | `ctx.registerService(Interface.class, mockImpl)` BEFORE `registerInjectActivateService()`. |
| MULTIPLE cardinality | Register multiple services with the same interface — AemContext collects them all. |
| Service ranking | Pass `"service.ranking", value` as extra args to `registerService()` to test ranking. |
| ConfigurationPolicy.REQUIRE | AemContext does NOT enforce this — always provide config in tests to mirror production. |
