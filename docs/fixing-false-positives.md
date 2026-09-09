# Fixing DAGP False Positives for AE

## Context

We're running DAGP (Dependency Analysis Gradle Plugin) against the Appian `ae` monorepo (~1000 Gradle projects). After removing 807 verified unused dependencies, 856 remain flagged as "unused." Many of these are **false positives** — dependencies that ARE actually needed but DAGP doesn't detect their usage.

There are also **pattern mismatches** in our removal script that prevent automated cleanup of legitimately unused deps.

**Repos:**
- DAGP (our fork): `~/repo/dependency-analysis-gradle-plugin`
- ae worktree: `~/repo/wt/ae/LCP-60585-dagp-integration`
- DAGP is published to mavenLocal as version `3.16.1-SNAPSHOT`

**Task:** LCP-60585

---

## Latest Run — July 23, 2026

### Results

| Metric | Value |
|--------|-------|
| Projects analyzed | 732 |
| Unused deps (implementation + api) | 1,479 |
| Unused deps (all configs incl. test) | 5,325 |
| In truth table (still reported) | 747 |
| New findings (not in truth table) | 732 |
| No longer reported (resolved) | 109 |

### Comparison with Truth Table

| Category | Count |
|----------|-------|
| KEPT_NEEDED (false positives) still present | 84 |
| REMOVED (true positives) re-flagged | 96 |
| NOT_ATTEMPTED still present | 567 |
| Dropped from report (resolved) | 109 |

### 732 New Findings Breakdown

| Category | Count | False Positive Risk |
|----------|-------|-------------------|
| Project deps (transitive exposure) | 302 | HIGH |
| Spring framework deps | 69 | HIGH |
| Runtime annotation deps | 27 | HIGH |
| Logging deps (slf4j, log4j) | 26 | HIGH |
| Feature toggle client | 16 | MEDIUM |
| Komodo/Kafka | 8 | MEDIUM |
| Other external modules | 284 | MIXED |

Top projects: `:deployment:assembly` (41), `:serverless-sail-evaluator` (30), `:docs-evaluator` (27), `:test` (20)

### 88 False Positives (KEPT_NEEDED) Root Causes

| Root Cause | Count | % |
|-----------|-------|---|
| Transitive exposure (project deps) | 80 | 91% |
| Runtime-only deps | 3 | 3% |
| Framework classpath deps | 2 | 2% |
| Runtime annotations | 1 | 1% |
| Other | 2 | 2% |

### Correct Run Command

```bash
# Do NOT pass -Dorg.gradle.jvmargs — let gradle.properties (16G) take effect
./gradlew generateBuildHealth --no-configuration-cache --no-build-cache --continue \
  -Ddependency.analysis.cache.max=300 \
  -Ddependency.analysis.batch.size=100 \
  -Pdependency.analysis.project.includes='^(?!.*(tempo|lcp-api-server-generated|appian-gwt-components)).*$' \
  -x :appian-libraries:gwt:appian-gwt-components:compressJavascript
```

**Important:** The `gradle.properties` already sets `-Xmx16G`. Passing `-Dorg.gradle.jvmargs="-Xmx12G"` on the command line OVERRIDES and REDUCES the heap, causing OOM.

---

## Problem 1: False Positives (Plugin-Side Fixes)

DAGP marks a dependency as "unused" when bytecode analysis doesn't find direct usage. But some dependencies ARE used through patterns DAGP doesn't fully detect.

### 1A: Runtime-Only Dependencies

DAGP analyzes compile-time bytecode. Dependencies needed only at runtime get flagged as unused.

**Examples from ae:**
- `com.appian.komodo:kafka-util` — Kafka runtime infrastructure
- `com.appian:safe-tracer-client` — tracing agent loaded at runtime
- `jakarta.resource:jakarta.resource-api` — JCA runtime
- `io.prometheus:simpleclient_log4j` — metrics bridge loaded via config

**What DAGP already does:** It detects `Class.forName("com.foo.Bar")` calls in bytecode and marks those as reflective usage. It also scans `META-INF/services/` for ServiceLoader patterns.

**What's missing:**
- Spring dependency injection: classes referenced only in `@Configuration` classes via `@Bean` return types or `@ComponentScan` packages
- Plugin systems: Appian's OSGI plugin framework (`appian-plugin.xml` references)
- Indirect reflection: `Class.forName(variable)` where the class name comes from a variable/config, not a string literal
- Runtime-only annotations: `@Inject`, `@Resource`, `jakarta.ws.rs` annotations that wire dependencies at deployment time

**Relevant DAGP source files:**
- `src/main/kotlin/com/autonomousapps/tasks/ComputeUsagesTask.kt` — where "used" vs "unused" is decided
- `src/main/kotlin/com/autonomousapps/internal/asm.kt` — bytecode analysis (detects `Class.forName`)
- `src/main/kotlin/com/autonomousapps/tasks/FindServiceLoadersTask.kt` — ServiceLoader detection
- `src/main/kotlin/com/autonomousapps/model/internal/intermediates/producer/ReflectingDependency.kt` — reflection model

### 1B: Transitive Exposure

A dep declared as `api` in module A exposes its classes to A's consumers. DAGP may say it's "unused" in A (A doesn't use it directly), but removing it breaks downstream modules.

**Examples from ae:**
- `appian-redisson`: `api globalDep('io.micrometer:micrometer-registry-prometheus')` — `:appian-libraries:ae` uses it transitively
- `ae-common`: `api project(':appian-libraries:jvm-classloaders-common')` — ae uses it transitively
- `teneo-shadow`: `implementation 'org.eclipse.emf.teneo:*'` — ae uses teneo classes through this shadow jar
- `record-security-java` — `lcp-api-plugin` accesses its classes through `records-java`

**Root cause:** DAGP checks if the DECLARING module uses the dep. It doesn't check if DOWNSTREAM consumers need it exposed via `api`. When we remove an "unused" `api` dep, downstream modules lose transitive access.

**What DAGP should do:** Before recommending removal of an `api`-declared dependency, verify no downstream consumer references classes from that dependency. This requires cross-project analysis.

**Relevant DAGP source files:**
- `src/main/kotlin/com/autonomousapps/tasks/ComputeAdviceTask.kt` — where advice is generated
- `src/main/kotlin/com/autonomousapps/internal/transform/StandardTransform.kt` — advice transformation
- `src/main/kotlin/com/autonomousapps/tasks/GenerateBuildHealthTask.kt` — aggregation (has access to all projects)

### 1C: Annotation-Only Dependencies

Dependencies whose classes appear only as annotations (e.g., `@Inject`, `@Resource`, `@WebServlet`) that are retained at runtime for framework scanning.

**Examples:**
- `javax.ws.rs:javax.ws.rs-api` — JAX-RS annotations (`@Path`, `@GET`) scanned at runtime
- `jakarta.inject:jakarta.inject-api` — `@Inject` used by DI frameworks
- `jakarta.servlet:jakarta.servlet-api` — servlet annotations

**What DAGP already does:** It has a `compileOnly` candidate detector. But these deps are often declared as `implementation` because they're needed at both compile AND runtime.

---

## Problem 2: Pattern Mismatches (Script-Side Fixes)

Our removal script (`commit-per-module.py`) uses regex to find and delete dependency lines from `.gradle` files. Some ae patterns don't match.

### 2A: `globalDeps()` Plural (Multi-Argument List)

146 ae modules use this pattern:

```groovy
dependencies {
  implementation(
    globalDeps(
      'com.appian:eng-feature-toggles-client',
      'com.google.code.gson:gson',
      'org.apache.commons:commons-lang3',
    ),
    projectDeps(
      ':appian-libraries:ae',
      ':appian-libraries:ae-common',
    )
  )
}
```

**Current handling:** Our script can remove individual lines from within a `globalDeps()` list (matching `'group:artifact'` on its own line). This works for most cases.

**What breaks:** When the item to remove is the LAST item and has a trailing comma issue, or when removing it leaves an empty `globalDeps()` call.

### 2B: `projectDeps()` Plural

137 ae modules use this for project dependencies:

```groovy
projectDeps(
  ':appian-libraries:ae',
  ':appian-libraries:common-configuration',
)
```

**Current handling:** NOT handled. Our script only matches `project(':path')` single-line patterns, not entries inside `projectDeps()` lists.

### 2C: Closure-Style Dependencies

```groovy
implementation globalDep("org.apache.rampart:rampart") {
  artifact {
    name = 'rampart'
    type = 'mar'
  }
}
```

**Current handling:** Partially handled — we detect the opening `{` and count braces to remove the whole block. But it only works when the identifier exactly matches the DAGP report's identifier.

### 2D: Conditional Dependencies

```groovy
if (isCI()) {
  errorprone globalDep('com.google.errorprone:error_prone_core')
}
```

**Current handling:** NOT handled. These deps are inside conditionals and shouldn't be removed blindly.

### 2E: Configuration Mismatch

DAGP reports the dep on configuration `implementation` but the actual declaration uses a different config:
- Declared as `systemTestCompileOnly` but DAGP reports as `implementation`
- Declared as `sharedTestImplementation` but DAGP reports as `implementation`

This happens because we told DAGP to `ignoreSourceSet('test', 'unitTest', etc.)` but it still sometimes reports deps from those source sets.

---

## Problem 3: What to Fix Where

### In DAGP (plugin code at `~/dev/dependency-analysis-gradle-plugin`):

1. **Cross-project transitive check** — Before advising removal of an `api`-declared dep, check if any downstream project in the build uses classes from that dep. This is the highest-value fix for reliability.

2. **Spring/DI awareness** — Scan for `@Bean` return types, `@ComponentScan` packages, and Spring XML configs to mark deps as "used" even without direct bytecode references.

3. **Appian plugin manifest scanning** — Parse `appian-plugin.xml` files and mark referenced classes' dependencies as used.

4. **Better runtime annotation detection** — Deps that provide annotations with `RUNTIME` retention (`@Path`, `@Inject`, `@Resource`) should be considered "used" if those annotations appear in the project's source.

### In the removal script (at `~/repo/wt/ae/LCP-60585-dagp-integration/commit-per-module.py`):

1. **Add `projectDeps()` plural pattern** — Match and remove items from `projectDeps(...)` lists (same as we already do for `globalDeps()`).

2. **Handle empty list cleanup** — After removing the last item from a `globalDeps()` or `projectDeps()` list, remove the entire empty `configuration(globalDeps())` block.

3. **Skip conditional deps** — If a dep line is inside an `if` block, skip it.

4. **Configuration name flexibility** — When matching, try alternative config names if the exact one doesn't match (e.g., if DAGP says `implementation`, also check `api` and `compileOnly`).

---

## Files to Understand

### DAGP plugin (analysis engine):
| File | Purpose |
|------|---------|
| `src/main/kotlin/com/autonomousapps/tasks/ComputeUsagesTask.kt` | Core "is this dep used?" logic |
| `src/main/kotlin/com/autonomousapps/tasks/ComputeAdviceTask.kt` | Generates add/remove/change advice |
| `src/main/kotlin/com/autonomousapps/internal/transform/StandardTransform.kt` | Transforms raw usage into actionable advice |
| `src/main/kotlin/com/autonomousapps/internal/asm.kt` | Bytecode analysis (class scanning) |
| `src/main/kotlin/com/autonomousapps/tasks/FindServiceLoadersTask.kt` | ServiceLoader detection |
| `src/main/kotlin/com/autonomousapps/tasks/GenerateBuildHealthTask.kt` | Aggregates all projects into final report |
| `src/main/kotlin/com/autonomousapps/subplugin/RootPlugin.kt` | Root plugin wiring (has access to allprojects) |

### Removal script:
| File | Purpose |
|------|---------|
| `~/repo/wt/ae/LCP-60585-dagp-integration/commit-per-module.py` | Per-module dep removal with compile validation |

---

## How to Validate Changes

1. **Publish DAGP to mavenLocal:**
   ```bash
   cd ~/dev/dependency-analysis-gradle-plugin
   JAVA_HOME=$(/usr/libexec/java_home -v 21) ./gradlew :publishToMavenLocal -x test -x functionalTest -x smokeTest -x signMavenPublication
   ```

2. **Run on ae:**
   ```bash
   cd ~/repo/wt/ae/LCP-60585-dagp-integration
   ./gradlew generateBuildHealth --no-configuration-cache --no-build-cache --continue \
     -Dorg.gradle.jvmargs="-Xmx12G -XX:+UseG1GC" \
     -Ddependency.analysis.cache.max=300 \
     -Ddependency.analysis.batch.size=100 \
     -Pdependency.analysis.project.includes='^(?!.*(tempo|lcp-api-server-generated|appian-gwt-components)).*$' \
     -x :appian-libraries:gwt:appian-gwt-components:compressJavascript
   ```

3. **Check results:**
   ```bash
   python3 -c "
   import json
   with open('build/reports/dependency-analysis/build-health-report.json') as f:
       data = json.load(f)
   print(f'Unused deps: {data[\"unusedCount\"]}')
   "
   ```

4. **Test removal reliability:** Run `commit-per-module.py` on a module and verify `compileJava` passes for the module AND its downstream consumers.

---

## Success Criteria

- `unusedCount` drops below 856 (fewer false positives)
- Zero false positives when removing based on the report (if DAGP says it's unused, removing it shouldn't break compilation of any module in the build)
- Script handles `globalDeps()`, `projectDeps()`, and closure patterns
- Running the full removal + compile-check cycle produces no failures

---

## Priority Order

1. **Script: `projectDeps()` pattern** — Quick win, unblocks ~137 more modules
2. **DAGP: Transitive exposure check** — Highest-value fix for reliability. If an `api` dep's classes are used by downstream consumers, don't recommend removal.
3. **Script: Empty list cleanup** — Prevents leaving broken empty `globalDeps()` blocks
4. **DAGP: Runtime annotation retention** — Catches `@Inject`, `@Path`, etc.
5. **DAGP: Spring DI awareness** — Catches `@Bean` return types
6. **DAGP: Appian plugin manifest** — ae-specific, catches OSGI plugin references
