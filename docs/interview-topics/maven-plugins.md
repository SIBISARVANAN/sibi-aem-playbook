# AEM Maven Plugins Reference

A senior-level reference for the Maven plugins found in a typical AEM multi-module project (`core`, `ui.apps`/`all`, `ui.content`, `ui.config`), covering what each plugin actually does, key configuration, and common misconfiguration symptoms — the kind of detail interviewers probe for beyond "I just add dependencies."

## Concept — Dependencies vs Plugins

- **Dependencies** (`<dependencies>`) are code your project *compiles against and/or bundles* — libraries your classes call.
- **Plugins** (`<build><plugins>`) are tools that run *during the build lifecycle itself* — they compile, test, package, transform, and deploy your project. They don't add code to your classpath at runtime; they control what happens when you type `mvn <phase>`.

Every `mvn` command (`compile`, `test`, `package`, `verify`, `install`, `deploy`) is a **phase** in Maven's build lifecycle, and each phase runs zero or more plugin **goals** bound to it.

## 1. maven-compiler-plugin

Compiles `.java` source into `.class` bytecode. Bound to `compile` (and `test-compile`).

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-compiler-plugin</artifactId>
  <configuration>
    <release>11</release>   <!-- preferred over separate source/target -->
  </configuration>
</plugin>
```

`<release>` (Java 9+) is preferred over `<source>`/`<target>` — it also constrains which JDK *APIs* are available, not just language syntax. AEM as a Cloud Service currently targets Java 11 (Java 17 support introduced more recently) — a mismatch between this config and the JDK actually used by CI/Cloud Manager is a classic "works locally, fails in pipeline" cause.

| Symptom | Likely cause |
|---|---|
| `UnsupportedClassVersionError` at runtime on AEM | Compiled with a newer Java version than the AEM runtime supports |
| Build passes locally, fails in Cloud Manager pipeline | Different JDK version between local machine and pipeline; `<release>` not pinned |

## 2. maven-bundle-plugin (or bnd-maven-plugin)

This is the plugin that makes AEM development "OSGi development." It generates the OSGi `MANIFEST.MF` (`Bundle-SymbolicName`, `Import-Package`, `Export-Package`, `Bundle-Version`), turning a plain JAR into a valid OSGi bundle Felix can load.

```xml
<plugin>
  <groupId>org.apache.felix</groupId>
  <artifactId>maven-bundle-plugin</artifactId>
  <extensions>true</extensions> <!-- required -->
  <configuration>
    <instructions>
      <Bundle-SymbolicName>${project.artifactId}</Bundle-SymbolicName>
      <Export-Package>com.sibi.aem.one.core.api.*</Export-Package>
      <Import-Package>*</Import-Package>
      <Sling-Model-Packages>com.sibi.aem.one.core.models</Sling-Model-Packages>
    </instructions>
  </configuration>
</plugin>
```

Many current AEM archetypes instead use the newer **bnd-maven-plugin** with a separate `bnd.bnd` file — same goal, different tooling.

- **`Import-Package` vs `Export-Package` is the single most important concept here.** `Export-Package` declares which of your own packages other bundles may use. `Import-Package` declares which external packages your bundle needs, and at what version range. Getting this wrong is the #1 cause of `"Unresolved constraint... package uses conflict"` errors on deploy.
- `Import-Package: *` (wildcard, BND auto-detects from bytecode) is convenient but can miss reflection-based dependencies (`Class.forName`, dynamic proxies), which are invisible to static bytecode analysis.
- `Sling-Model-Packages` tells Sling Models' bundle-scanning where to look for `@Model`-annotated classes — omitting a package here means your models simply never register, with no obvious error.
- **Package versioning ranges** — BND auto-generates version ranges for imported packages based on the exporting bundle's version at build time (e.g. `[1.2,2)`). If a dependency is later upgraded outside that range, your bundle fails to resolve — a "worked yesterday, broken today" bug.

| Symptom | Likely cause |
|---|---|
| Bundle shows "Installed" but not "Active" in Felix Console | Unresolved package import |
| Sling Model never gets invoked, no error | `Sling-Model-Packages` doesn't include the model's package |
| `ClassNotFoundException` despite the class being in your JAR | Package wasn't exported, or import/export mismatch |
| Bundle fails to resolve after an unrelated platform upgrade | Auto-generated version range no longer matches the new exporter's version |

## 3. filevault-package-maven-plugin (formerly content-package-maven-plugin)

Builds the content package `.zip` (`ui.apps`, `ui.content`, `all`) from `jcr_root` + `filter.xml` — installs pages/components/configs/design content, as distinct from the bundle plugin installing *code*.

```xml
<plugin>
  <groupId>org.apache.jackrabbit</groupId>
  <artifactId>filevault-package-maven-plugin</artifactId>
  <extensions>true</extensions>
  <configuration>
    <group>com.sibi</group>
    <name>sibi-aem-one.ui.apps</name>
    <packageType>application</packageType>
    <filterSource>src/main/content/META-INF/vault/filter.xml</filterSource>
  </configuration>
</plugin>
```

- **`filter.xml` defines the "workspace filter"** — the JCR paths this package owns and will overwrite on install. **The single highest-stakes piece of AEM Maven config**: a filter root too broad (`/content` instead of `/content/mysite`) can wipe unrelated content on deploy; a missing filter root means content silently doesn't deploy.
- `<packageType>` matters specifically for AEMaaCS — Cloud Manager validates that `application` packages (code, OSGi configs, `ui.apps`) and `content` packages (`ui.content`) are correctly separated; mixing them can fail the readiness check.
- Filter rules support `mode="merge"`/`mode="replace"` — `replace` (default) wipes and replaces everything under that root; `merge` only adds/updates. Using `replace` on a root that authors also write to is a common way to accidentally delete author-created content on next deploy.
- The `all` package typically embeds `core` and `ui.apps`/`ui.content` via `<embeddeds>` — why deploying `all` installs everything in one shot, and why a broken filter in a sub-package breaks the whole `all` deployment.

| Symptom | Likely cause |
|---|---|
| Author-created content disappears after a code deploy | Overly broad filter root using `replace` mode covering author-editable content |
| New component/template doesn't appear after deploy | Missing filter root for that path |
| Cloud Manager fails at "content package validation" | `application` package contains paths that should be in a `content` package, or vice versa |

## 4. maven-resources-plugin

Copies non-Java resource files (`.content.xml`, properties, OSGi config JSON under `src/main/resources`) into the build output.

Runs in `process-resources`, before `compile`. Resource filtering (`${project.version}` substitution) happens here if `<filtering>true</filtering>` is set. A resource "not showing up" is very often a wrong `<resource>/<directory>` path, not a bug in this plugin.

| Symptom | Likely cause |
|---|---|
| OSGi config present in source but missing from deployed bundle | Not under a configured `<resources>` directory, or excluded by filter pattern |
| `${project.version}` literally appears unsubstituted | `<filtering>` not enabled on that resource directory |

## 5. maven-jar-plugin

Packages compiled classes + resources into a plain `.jar`, which the bundle plugin then post-processes with the OSGi manifest. Bound to `package`.

Usually invisible/default-configured. If you see plain-JAR manifest entries (no `Import-Package`/`Export-Package`) rather than OSGi-flavored ones, it usually means the bundle plugin isn't configured with `<extensions>true</extensions>`.

## 6. maven-surefire-plugin

Runs unit tests (`**/*Test.java` by default) during `test` — the plugin actually executing your JUnit/Mockito/AemContext suite.

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-surefire-plugin</artifactId>
  <configuration>
    <argLine>-Xmx1024m</argLine>
    <includes><include>**/*Test.java</include></includes>
  </configuration>
</plugin>
```

- `mvn package`/`mvn install` run tests by default — `-DskipTests` skips execution but still compiles test classes; `-Dmaven.test.skip=true` skips both compiling and running (faster, riskier).
- **The JaCoCo argLine pitfall:** Surefire and JaCoCo interact through the `argLine` property — JaCoCo's `prepare-agent` goal injects a `-javaagent` flag into that same `argLine`. If your `pom.xml` manually overrides `<argLine>` elsewhere without including `@{argLine}` (the JaCoCo-populated placeholder), you silently lose coverage instrumentation — no error, just a 0% report.
- Test JVM forking (`forkCount`) affects speed and isolation — a shared/reused fork across many test classes can leak static state between test classes if `MockedStatic` isn't properly closed.

| Symptom | Likely cause |
|---|---|
| JaCoCo report shows 0% coverage despite tests passing | Custom `<argLine>` override doesn't include `@{argLine}`, dropping the agent |
| Tests silently don't run at all | Test class naming doesn't match `<includes>` pattern (e.g. `*Tests.java` instead of `*Test.java`) |
| `OutOfMemoryError` during `mvn test` on a large suite | Default heap too small for the test JVM; needs `<argLine>-Xmx...` increase |

## 7. maven-failsafe-plugin

Runs *integration* tests (`**/*IT.java`), bound to `integration-test`/`verify` — distinct from surefire's unit tests, which run earlier in `test`. Integration tests often need a running environment (deployed AEM instance, external service) that shouldn't block a fast local `mvn test`.

Failsafe runs `integration-test` (execute) and `verify` (check results) as **two separate goal bindings**, so post-integration-test cleanup can run even if a test failed — why failsafe exists as a distinct plugin rather than surefire also handling `*IT.java`.

Most AEM projects don't have integration tests configured by default from the archetype — if your project has none, this plugin may not even be present in `pom.xml`; that's normal, not a gap.

## 8. sling-maven-plugin

Installs/deploys the built package or bundle directly to a running AEM instance — the plugin actually invoked by `-PautoInstallPackage` / `-PautoInstallBundle`.

```xml
<profile>
  <id>autoInstallPackage</id>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.sling</groupId>
        <artifactId>sling-maven-plugin</artifactId>
        <configuration>
          <slingUrl>http://localhost:4502/system/console</slingUrl>
        </configuration>
        <executions>
          <execution><goals><goal>install</goal></goals></execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</profile>
```

- `-P` activates a Maven **profile** — `autoInstallPackage`/`autoInstallBundle` are conventionally defined profiles that add this plugin's execution only when explicitly requested. This is why a plain `mvn clean install` never touches a running AEM instance.
- `autoInstallBundle` deploys just the OSGi bundle (fast, backend-only iteration); `autoInstallPackage` deploys the full content package (slower, includes content/config changes).
- **Irrelevant to Cloud Manager deployments** — Cloud Manager uses its own orchestration reading the built artifacts, not this plugin against a live URL. Strictly a local/on-prem dev convenience.

| Symptom | Likely cause |
|---|---|
| `mvn clean install -PautoInstallPackage` succeeds but nothing changes on the instance | Wrong `slingUrl` port/host, or instance not running |
| Deploy "succeeds" but bundle doesn't update | Used `autoInstallPackage` when only bundle code changed — `autoInstallBundle` would be faster |

## 9. jacoco-maven-plugin

*(Full detail in `/docs/testing/junit-phase-10.md` — summarized here.)*

- `prepare-agent` goal must run before `surefire`/`failsafe` execution to attach the coverage-collecting Java agent.
- `report` goal generates HTML (`target/site/jacoco/index.html`) and XML (`jacoco.xml`, consumed by SonarQube/Cloud Manager quality gates).
- `check` goal is the actual build-failing enforcement — `report` alone does not block a bad build.
- Threshold values are **fractional (0.0–1.0), not percentages** — `<minimum>80</minimum>` is a common, silently-impossible-to-meet misconfiguration (should be `0.80`).

## Full Lifecycle Walkthrough — What `mvn clean install -PautoInstallPackage` Actually Does

1. **`clean`** — deletes `target/`: stale compiled classes, old JaCoCo `.exec` data, old artifacts.
2. **`validate`/`initialize`** — Maven resolves effective config, including active profiles.
3. **`process-resources`** — `maven-resources-plugin` copies non-Java resources into `target/classes`.
4. **`compile`** — `maven-compiler-plugin` compiles `.java` sources.
5. **`process-test-resources`/`test-compile`** — same, for test sources.
6. **`test`** — `maven-surefire-plugin` runs unit tests; JaCoCo's `prepare-agent` (if run earlier) collects coverage here.
7. **`package`** — `maven-jar-plugin` builds a plain JAR, then `maven-bundle-plugin`/`bnd-maven-plugin` post-processes it into an OSGi bundle (`core`); `filevault-package-maven-plugin` builds the content package `.zip` (`ui.apps`/`ui.content`/`all`).
8. **`integration-test`/`verify`** — `maven-failsafe-plugin` runs `*IT.java` if present; JaCoCo's `check` goal enforces coverage thresholds here, potentially failing the build.
9. **`install`** — the built artifact (bundle JAR or package ZIP) copies into local `~/.m2`.
10. **(profile-only) `sling:install`** — because `-PautoInstallPackage` was passed, `sling-maven-plugin` pushes the freshly built artifact to the running AEM instance.

## Quick Reference Table

| Plugin | Phase it binds to | What breaks without it | Local-dev only? |
|---|---|---|---|
| maven-compiler-plugin | compile | Nothing compiles | No |
| maven-bundle-plugin / bnd-maven-plugin | package | No OSGi manifest — bundle won't be recognized | No |
| filevault-package-maven-plugin | package | No deployable content package | No |
| maven-resources-plugin | process-resources | Non-Java files missing from build | No |
| maven-jar-plugin | package | No JAR to turn into a bundle | No |
| maven-surefire-plugin | test | Unit tests never run | No |
| maven-failsafe-plugin | integration-test/verify | Integration tests never run | No |
| sling-maven-plugin | install (profile-bound) | No direct deploy to a running local instance | Yes |
| jacoco-maven-plugin | test/verify | No coverage report, no coverage gate enforcement | No |

> **Interview framing tip:** if asked "walk me through what happens when you build an AEM project," the Full Lifecycle Walkthrough above is essentially the answer — naming the phases in order and which plugin does what at each step is what separates "I run the Maven command" from genuinely understanding the build.
