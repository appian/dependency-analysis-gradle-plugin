# DAGP Scalability for AE — Summary of Changes & Results

**Date:** 2026-07-01  
**Task:** LCP-60585  
**Repo:** `dependency-analysis-gradle-plugin` (fork of [autonomousapps/dependency-analysis-gradle-plugin](https://github.com/autonomousapps/dependency-analysis-gradle-plugin))

---

## Goal

Make the Dependency Analysis Gradle Plugin (DAGP) run reliably on the Appian ae monorepo (~1000 Gradle projects) to detect unused dependencies. Unused dependencies cause unnecessary CI job triggers — removing them reduces CI cycle time, compute cost, and build complexity.

---

## What We Changed in DAGP

### 1. Bounded InMemoryCache (`InMemoryCache.kt`)

**Problem:** DAGP's shared bytecode analysis cache (Caffeine) was unbounded by default (`-1` = unlimited). With 1000+ projects sharing 500–800 unique JARs, the cache grew until the JVM ran out of memory.

**Fix:** Cache is now always bounded with LRU eviction. Default size is 300 entries. If a JAR is evicted and needed later, it gets re-analyzed (costs CPU, saves memory).

**Configuration:** `-Ddependency.analysis.cache.max=300`

### 2. ProjectBatcher + Batch Size Flag (`ProjectBatcher.kt`, `Flags.kt`)

**Problem:** All ~1000 project analyses run concurrently, causing memory pressure spikes.

**Fix:** Added a `ProjectBatcher` utility that partitions projects into ordered groups. A new system property controls batch size.

**Configuration:** `-Ddependency.analysis.batch.size=100`

### 3. BatchAggregateTask + RootPlugin Wiring (`BatchAggregateTask.kt`, `RootPlugin.kt`)

**Problem:** The root `generateBuildHealth` task depends on all projects simultaneously — Gradle's task graph holds all upstream outputs in memory.

**Fix:** When the project count exceeds batch size, intermediate `BatchAggregateTask` instances are registered with `mustRunAfter` ordering constraints. This limits how many project analyses are in-flight at once.

### 4. ConcurrentModificationException Fix (`StandardTransform.kt`)

**Problem:** `StandardTransform.simplify()` iterates over mutable sets (`remove`, `change`) while modifying them inside the loop body (`remove -= theRemove`, `change -= theChange`). This causes `ConcurrentModificationException` on large builds with complex dependency graphs.

**Fix:** Changed `remove.forEach` → `remove.toList().forEach` and `change.forEach` → `change.toList().forEach`. Iterates over a snapshot, mutations happen on the original set.

### Files Changed (source only, excludes docs)

| File | Change |
|------|--------|
| `src/main/kotlin/com/autonomousapps/Flags.kt` | Added `BATCH_SIZE` constant and `batchSize()` function |
| `src/main/kotlin/com/autonomousapps/internal/ProjectBatcher.kt` | New — partitions projects into batches |
| `src/main/kotlin/com/autonomousapps/internal/transform/StandardTransform.kt` | Fixed ConcurrentModificationException |
| `src/main/kotlin/com/autonomousapps/services/InMemoryCache.kt` | Bounded all caches with LRU eviction |
| `src/main/kotlin/com/autonomousapps/subplugin/RootPlugin.kt` | Added batched aggregation wiring |
| `src/main/kotlin/com/autonomousapps/tasks/BatchAggregateTask.kt` | New — intermediate batch collection task |
| `src/test/kotlin/com/autonomousapps/internal/ProjectBatcherTest.kt` | New — unit tests for batching |
| `src/test/kotlin/com/autonomousapps/services/InMemoryCacheTest.kt` | New — unit tests for bounded cache |

---

## What Needs to Change in AE

### `settings.gradle` — add two plugins to the settings `plugins {}` block:

```groovy
plugins {
  // ... existing plugins ...
  id "org.jetbrains.kotlin.jvm" version "2.1.10" apply false
  id "com.autonomousapps.build-health" version "3.16.1-SNAPSHOT"
}
```

The Kotlin plugin must be declared here (not just in subprojects) so DAGP can detect it on the same classloader.

### `build.gradle` — add configuration after the `plugins {}` block:

```groovy
dependencyAnalysis {
  issues {
    all {
      onUnusedDependencies { severity('warn') }
      onUsedTransitiveDependencies { severity('ignore') }
      onIncorrectConfiguration { severity('ignore') }
      onUnusedAnnotationProcessors { severity('ignore') }
      ignoreSourceSet('test')
      ignoreSourceSet('unitTest')
      ignoreSourceSet('integrationTest')
      ignoreSourceSet('systemTest')
      ignoreSourceSet('sharedTest')
      ignoreSourceSet('iosTests')
    }
  }
}
```

### `settings.gradle` — ensure `mavenLocal()` is in pluginManagement repositories

Already present in ae — no change needed.

### Run Command

```bash
./gradlew generateBuildHealth --no-configuration-cache --no-build-cache --continue \
  -Dorg.gradle.jvmargs="-Xmx12G -XX:+UseG1GC" \
  -Ddependency.analysis.cache.max=300 \
  -Ddependency.analysis.batch.size=100 \
  -Pdependency.analysis.project.includes='^(?!.*(tempo|lcp-api-server-generated|appian-gwt-components)).*$' \
  -x :appian-libraries:gwt:appian-gwt-components:compressJavascript
```

### Projects Excluded (and why)

| Project | Reason |
|---------|--------|
| `:appian-libraries:tempo` | Uses generated source dirs without declaring task dependency on the generator |
| `:appian-libraries:lcp-api:lcp-api-server-generated-*` | Same — `openApiGenerate` output not wired to DAGP's source exploder |
| `:appian-libraries:gwt:appian-gwt-components` | `compressJavascript` task fails independently of DAGP |

These are ae-side issues (missing `dependsOn` declarations for generated sources), not DAGP bugs. They can be fixed independently by adding proper task dependencies in those modules' build files.

---

## Results

| Metric | Value |
|--------|-------|
| Projects analyzed | 728 |
| Projects with unused dependencies | 372 |
| Total unused dependencies found | 1,663 |
| Runtime | ~14 minutes |
| Peak memory | < 12GB (no OOM) |
| Disk usage | ~23GB intermediates (with test source sets excluded) |

### Top 20 Projects by Unused Dependency Count

| Unused | Project |
|--------|---------|
| 56 | `:test` |
| 48 | `:appian-services:serverless-sail-evaluator` |
| 43 | `:deployment:assembly` |
| 33 | `:appian-libraries:ae` |
| 31 | `:appian-services:docs-evaluator` |
| 30 | `:appian-libraries:process-insights-generation:...-test-utils` |
| 23 | `:appian-libraries:codeless-data-modeling:...-functions` |
| 21 | `:appian-libraries:pdf-to-interface:...-core-java` |
| 19 | `:appian-libraries:generation-pipeline:...-test-utils` |
| 18 | `:appian-libraries:codeless-data-modeling:...-java` |
| 16 | `:appian-libraries:appian-ix-tools` |
| 16 | `:appian-libraries:copilot:core:core-java` |
| 16 | `:appian-libraries:pdf-to-interface:...-sail-functions` |
| 16 | `:appian-libraries:process-hq-record:...-java` |
| 16 | `:appian-services:portal-service` |
| 15 | `:appian-libraries:process-hq:...-governance-java` |
| 15 | `:appian-libraries:process-insights-generation:...-java` |
| 15 | `:appian-libraries:records-chat:...-field-java` |
| 15 | `:test:test-util` |
| 14 | `:appian-libraries:copilot:core:copilot-sail-functions` |

---

## Recommendations

### Immediate (high leverage, low risk)

1. **Remove unused deps from top offenders first.** Start with `:test` (56), `:appian-services:serverless-sail-evaluator` (48), `:deployment:assembly` (43). These are likely causing the most unnecessary CI triggers.

2. **Validate before removing.** Run the affected module's unit tests after removing each dependency to confirm it's truly unused and not accessed via reflection or runtime-only patterns.

3. **Fix the 4 excluded projects.** Add `dependsOn(tasks.named("openApiGenerate"))` to the relevant source compilation tasks in `tempo` and `lcp-api-server-generated` modules.

### Short-term (CI integration)

4. **Publish DAGP to Appian's Artifactory** so the whole team can use it without `mavenLocal()`.

5. **Add a CI job** (initially manual/nightly) that runs `generateBuildHealth` and publishes the report as an artifact. This creates visibility into dependency hygiene over time.

6. **Scope CI enforcement to changed files only.** When a Gradle file is modified in a PR, run DAGP on that specific module and fail if new unused deps are introduced.

### Medium-term (deeper impact)

7. **Enable test source set analysis** once disk space is available (or on CI runners with large scratch volumes). Test deps that are unused are also causing unnecessary triggers in integration-tier jobs.

8. **Investigate the autofix/rewrite feature.** DAGP can auto-remove unused deps, but it needs grammar changes to handle `globalDep()` syntax. This is a separate effort.

9. **Measure CI trigger reduction.** After removing unused deps from the top modules, measure how many fewer integration-tier jobs are triggered per average PR. This quantifies the ROI.

---

## How It Works (for reference)

DAGP does NOT parse build scripts for analysis. It uses Gradle's runtime Configuration API to read the already-resolved dependency graph. This means:

- `globalDep('group:artifact')` — works automatically (Gradle resolves it before DAGP sees it)
- `project(':appian-libraries:...')` — works automatically
- Custom configurations (`unitTestImplementation`, etc.) — auto-discovered from `SourceSetContainer`

The ANTLR grammar (build script parsing) is only used for the autofix/rewrite step, which is NOT part of this MVP.

For each project, DAGP:
1. Resolves dependencies via Gradle's Configuration API
2. Explodes JAR bytecode to find what classes each dependency provides
3. Parses source files for imports and usages
4. Computes which dependencies are actually used vs. declared but unused
5. Aggregates results into a build-wide report
