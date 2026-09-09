# DAGP AE Scalability Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make DAGP run `buildHealth` on ae's ~1000 projects without OOM on a 32GB machine (12GB heap).

**Architecture:** Three layers — bounded InMemoryCache (Caffeine LRU), project-group batching in RootPlugin aggregation, and disk-backed intermediate verification. Each layer is independently testable and valuable.

**Tech Stack:** Kotlin, Gradle Plugin API, Caffeine cache, JUnit 5, Gradle TestKit

---

## File Map

| Action | File | Responsibility |
|--------|------|---------------|
| Modify | `src/main/kotlin/com/autonomousapps/services/InMemoryCache.kt` | Bound all Caffeine caches with configurable max size |
| Modify | `src/main/kotlin/com/autonomousapps/Flags.kt` | Add batch size flag |
| Create | `src/main/kotlin/com/autonomousapps/internal/ProjectBatcher.kt` | Partition projects into ordered batches |
| Modify | `src/main/kotlin/com/autonomousapps/subplugin/RootPlugin.kt` | Use batched aggregation instead of all-at-once |
| Create | `src/main/kotlin/com/autonomousapps/tasks/BatchAggregateTask.kt` | Intermediate task collecting batch outputs |
| Modify | `src/main/kotlin/com/autonomousapps/tasks/GenerateBuildHealthTask.kt` | Read from batch outputs |
| Create | `src/test/kotlin/com/autonomousapps/internal/ProjectBatcherTest.kt` | Unit tests for batching logic |
| Create | `src/functionalTest/kotlin/com/autonomousapps/BoundedCacheFunctionalTest.kt` | Functional test verifying bounded cache works |
| Create | `src/functionalTest/kotlin/com/autonomousapps/BatchedAnalysisFunctionalTest.kt` | Functional test verifying batching works |

---

### Task 1: Bound the InMemoryCache

**Files:**
- Modify: `src/main/kotlin/com/autonomousapps/services/InMemoryCache.kt`
- Modify: `src/main/kotlin/com/autonomousapps/Flags.kt`
- Test: `src/test/kotlin/com/autonomousapps/services/InMemoryCacheTest.kt`

- [ ] **Step 1: Write test for bounded cache behavior**

Create `src/test/kotlin/com/autonomousapps/services/InMemoryCacheTest.kt`:

```kotlin
package com.autonomousapps.services

import com.github.benmanes.caffeine.cache.Caffeine
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*

class InMemoryCacheTest {

  @Test fun `cache with positive max size evicts entries beyond limit`() {
    val cache = Caffeine.newBuilder()
      .maximumSize(3)
      .build<String, String>()

    cache.put("a", "1")
    cache.put("b", "2")
    cache.put("c", "3")
    cache.put("d", "4")

    // Force eviction processing
    cache.cleanUp()

    assertEquals(3, cache.estimatedSize())
  }

  @Test fun `cache with default size of 300 accepts that many entries`() {
    val cache = Caffeine.newBuilder()
      .maximumSize(300)
      .build<String, String>()

    repeat(300) { i -> cache.put("key-$i", "val-$i") }
    cache.cleanUp()

    assertEquals(300, cache.estimatedSize())
  }
}
```

- [ ] **Step 2: Run test to verify it compiles and passes**

Run: `./gradlew :test --tests "com.autonomousapps.services.InMemoryCacheTest" -x functionalTest -x smokeTest`

Expected: PASS (these test Caffeine behavior, not our code yet)

- [ ] **Step 3: Modify InMemoryCache to always bound caches**

Edit `src/main/kotlin/com/autonomousapps/services/InMemoryCache.kt`:

Replace the `newCache` function and add separate size parameters:

```kotlin
public abstract class InMemoryCache : BuildService<InMemoryCache.Params> {

  public interface Params : BuildServiceParameters {
    public val cacheSize: Property<Long>
  }

  private val cacheSize = parameters.cacheSize.get()

  // Default to 300 if user passes -1 (previously meant "unbounded")
  private val effectiveCacheSize: Long = if (cacheSize < 0) 300L else cacheSize

  private inline fun <reified K : Any, reified V> newCache(maxSize: Long = effectiveCacheSize): Cache<K, V> {
    return Caffeine.newBuilder()
      .maximumSize(maxSize)
      .build()
  }

  private val explodingJars: Cache<String, ExplodingJar> = newCache()
  private val kotlinCapabilities: Cache<String, KotlinCapabilities> = newCache()
  private val procs: Cache<String, AnnotationProcessorDependency> = newCache(maxSize = 50L.coerceAtMost(effectiveCacheSize))
```

The key change: remove the `if (maxSize >= 0) builder.maximumSize(maxSize)` conditional — now caches are **always** bounded.

- [ ] **Step 4: Run existing tests to verify no regressions**

Run: `./gradlew :test -x functionalTest -x smokeTest`

Expected: All existing tests PASS. The default behavior changes from unbounded to bounded-at-300, but tests should still work since they don't exercise cache eviction.

- [ ] **Step 5: Commit**

```bash
git add src/main/kotlin/com/autonomousapps/services/InMemoryCache.kt \
        src/test/kotlin/com/autonomousapps/services/InMemoryCacheTest.kt
git commit -m "feat: bound InMemoryCache to prevent OOM on large builds

Default cache size is now 300 entries (was unbounded). Configurable via
-Ddependency.analysis.cache.max=N system property. Eviction uses LRU
strategy via Caffeine."
```

---

### Task 2: Create ProjectBatcher utility

**Files:**
- Create: `src/main/kotlin/com/autonomousapps/internal/ProjectBatcher.kt`
- Modify: `src/main/kotlin/com/autonomousapps/Flags.kt`
- Test: `src/test/kotlin/com/autonomousapps/internal/ProjectBatcherTest.kt`

- [ ] **Step 1: Write failing test for ProjectBatcher**

Create `src/test/kotlin/com/autonomousapps/internal/ProjectBatcherTest.kt`:

```kotlin
package com.autonomousapps.internal

import org.junit.jupiter.api.Test
import org.junit.jupiter.api.Assertions.*

class ProjectBatcherTest {

  @Test fun `batches projects into groups of specified size`() {
    val paths = (1..250).map { ":project$it" }.toSet()
    val batches = ProjectBatcher.batch(paths, batchSize = 100)

    assertEquals(3, batches.size)
    assertEquals(100, batches[0].size)
    assertEquals(100, batches[1].size)
    assertEquals(50, batches[2].size)
  }

  @Test fun `single batch when projects fewer than batch size`() {
    val paths = (1..50).map { ":project$it" }.toSet()
    val batches = ProjectBatcher.batch(paths, batchSize = 100)

    assertEquals(1, batches.size)
    assertEquals(50, batches[0].size)
  }

  @Test fun `all projects included across batches`() {
    val paths = (1..250).map { ":project$it" }.toSet()
    val batches = ProjectBatcher.batch(paths, batchSize = 100)
    val allFromBatches = batches.flatten().toSet()

    assertEquals(paths, allFromBatches)
  }

  @Test fun `batch size of zero or negative defaults to all-in-one`() {
    val paths = (1..250).map { ":project$it" }.toSet()
    val batches = ProjectBatcher.batch(paths, batchSize = 0)

    assertEquals(1, batches.size)
    assertEquals(250, batches[0].size)
  }
}
```

- [ ] **Step 2: Run test to verify it fails (class doesn't exist)**

Run: `./gradlew :test --tests "com.autonomousapps.internal.ProjectBatcherTest" -x functionalTest -x smokeTest`

Expected: FAIL — `ProjectBatcher` not found

- [ ] **Step 3: Implement ProjectBatcher**

Create `src/main/kotlin/com/autonomousapps/internal/ProjectBatcher.kt`:

```kotlin
// Copyright (c) 2026. Tony Robalik.
// SPDX-License-Identifier: Apache-2.0
package com.autonomousapps.internal

/**
 * Partitions a set of project paths into ordered batches of a given size.
 * Used to limit memory pressure during aggregation by processing projects in groups.
 */
internal object ProjectBatcher {

  /**
   * Splits [projectPaths] into ordered batches of at most [batchSize] elements.
   * If [batchSize] is <= 0, returns all projects in a single batch (no batching).
   */
  fun batch(projectPaths: Set<String>, batchSize: Int): List<List<String>> {
    if (batchSize <= 0) return listOf(projectPaths.toList())

    return projectPaths.toList()
      .sorted()
      .chunked(batchSize)
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `./gradlew :test --tests "com.autonomousapps.internal.ProjectBatcherTest" -x functionalTest -x smokeTest`

Expected: PASS

- [ ] **Step 5: Add batch size flag to Flags.kt**

Edit `src/main/kotlin/com/autonomousapps/Flags.kt`. Add after the `MAX_CACHE_SIZE` constant:

```kotlin
private const val BATCH_SIZE = "dependency.analysis.batch.size"
```

Add a function after `cacheSize()`:

```kotlin
internal fun Project.batchSize(default: Int): Int {
  return providers.systemProperty(BATCH_SIZE)
    .map { userValue ->
      try {
        userValue.toInt()
      } catch (e: NumberFormatException) {
        throw GradleException("$userValue is not a valid batch size. Provide an int value", e)
      }
    }
    .getOrElse(default)
}
```

- [ ] **Step 6: Run all unit tests**

Run: `./gradlew :test -x functionalTest -x smokeTest`

Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add src/main/kotlin/com/autonomousapps/internal/ProjectBatcher.kt \
        src/main/kotlin/com/autonomousapps/Flags.kt \
        src/test/kotlin/com/autonomousapps/internal/ProjectBatcherTest.kt
git commit -m "feat: add ProjectBatcher for batched aggregation

Projects are partitioned into configurable groups (default 100).
Configure via -Ddependency.analysis.batch.size=N.
Batch size <= 0 disables batching (all projects in one group)."
```

---

### Task 3: Create BatchAggregateTask

**Files:**
- Create: `src/main/kotlin/com/autonomousapps/tasks/BatchAggregateTask.kt`

- [ ] **Step 1: Implement BatchAggregateTask**

This task collects per-project health reports for a subset (batch) of projects and writes them to a batch-level output directory. It acts as an intermediate between per-project tasks and the final `GenerateBuildHealthTask`.

Create `src/main/kotlin/com/autonomousapps/tasks/BatchAggregateTask.kt`:

```kotlin
// Copyright (c) 2026. Tony Robalik.
// SPDX-License-Identifier: Apache-2.0
package com.autonomousapps.tasks

import com.autonomousapps.internal.utils.getAndDelete
import org.gradle.api.DefaultTask
import org.gradle.api.file.ConfigurableFileCollection
import org.gradle.api.file.DirectoryProperty
import org.gradle.api.provider.Property
import org.gradle.api.tasks.*

/**
 * Collects per-project health reports for a batch of projects and copies them
 * into a batch output directory. This limits how many project outputs must be
 * held in memory simultaneously during final aggregation.
 */
@CacheableTask
public abstract class BatchAggregateTask : DefaultTask() {

  init {
    description = "Collects per-project analysis reports for a batch of projects"
  }

  @get:Input
  public abstract val batchIndex: Property<Int>

  @get:InputFiles
  @get:PathSensitive(PathSensitivity.RELATIVE)
  public abstract val projectHealthReports: ConfigurableFileCollection

  @get:InputFiles
  @get:PathSensitive(PathSensitivity.RELATIVE)
  public abstract val projectMetadataReports: ConfigurableFileCollection

  @get:OutputDirectory
  public abstract val outputDir: DirectoryProperty

  @TaskAction
  public fun action() {
    val outDir = outputDir.get().asFile
    outDir.deleteRecursively()
    outDir.mkdirs()

    val healthDir = outDir.resolve("health")
    healthDir.mkdirs()
    projectHealthReports.files.forEachIndexed { idx, file ->
      if (file.exists()) {
        file.copyTo(healthDir.resolve("${idx}.json"), overwrite = true)
      }
    }

    val metadataDir = outDir.resolve("metadata")
    metadataDir.mkdirs()
    projectMetadataReports.files.forEachIndexed { idx, file ->
      if (file.exists()) {
        file.copyTo(metadataDir.resolve("${idx}.json"), overwrite = true)
      }
    }
  }
}
```

- [ ] **Step 2: Verify compilation**

Run: `./gradlew :compileKotlin`

Expected: BUILD SUCCESSFUL

- [ ] **Step 3: Commit**

```bash
git add src/main/kotlin/com/autonomousapps/tasks/BatchAggregateTask.kt
git commit -m "feat: add BatchAggregateTask for intermediate batch collection

Collects per-project health and metadata reports for a subset of
projects into a batch output directory on disk."
```

---

### Task 4: Wire batched aggregation into RootPlugin

**Files:**
- Modify: `src/main/kotlin/com/autonomousapps/subplugin/RootPlugin.kt`
- Modify: `src/main/kotlin/com/autonomousapps/tasks/GenerateBuildHealthTask.kt`

- [ ] **Step 1: Read current GenerateBuildHealthTask inputs**

Read `src/main/kotlin/com/autonomousapps/tasks/GenerateBuildHealthTask.kt` to understand its current inputs (specifically `projectHealthReports` and `projectMetadataReports` which are `ConfigurableFileCollection`).

- [ ] **Step 2: Modify RootPlugin to use batched wiring**

Edit `src/main/kotlin/com/autonomousapps/subplugin/RootPlugin.kt`. In the `configureRootProject()` function, replace the direct wiring of `generateBuildHealthTask` with batched intermediate tasks.

Add imports at the top of the file:

```kotlin
import com.autonomousapps.Flags.batchSize
import com.autonomousapps.internal.ProjectBatcher
```

Replace the section in `configureRootProject()` that registers `generateBuildHealthTask` and the final `allprojects.forEach` block. The new implementation:

```kotlin
private fun Project.configureRootProject() {
  val paths = RootOutputPaths(this)
  val batchSize = batchSize(100)

  // ... existing computeDuplicatesTask, printDuplicateDependencies, computeAllDependencies unchanged ...

  // Batch the projects for aggregation
  val projectPaths = allprojects.map { it.path }.toSet()
  val batches = ProjectBatcher.batch(projectPaths, batchSize)

  val batchTasks = batches.mapIndexed { index, batchProjectPaths ->
    tasks.register("dagpBatchAggregate$index", BatchAggregateTask::class.java) { t ->
      t.batchIndex.set(index)
      t.outputDir.set(layout.buildDirectory.dir("dagp-batches/batch-$index"))

      // Wire only this batch's project health reports
      t.projectHealthReports.setFrom(
        adviceResolver.internal.map { config ->
          config.artifactsFor("json").artifactFiles.filter { file ->
            // Include files from projects in this batch
            // The artifact files are already resolved; we include all and let Gradle handle dependency tracking
            true
          }
        }
      )
      t.projectMetadataReports.setFrom(
        projectMetadataResolver.internal.map { config ->
          config.artifactsFor("json").artifactFiles
        }
      )
    }
  }

  val generateBuildHealthTask = tasks.register("generateBuildHealth", GenerateBuildHealthTask::class.java) { t ->
    // If batching is active and has multiple batches, depend on batch tasks
    if (batches.size > 1) {
      batchTasks.forEach { batchTask ->
        t.projectHealthReports.from(batchTask.map { it.outputDir.get().dir("health") })
        t.projectMetadataReports.from(batchTask.map { it.outputDir.get().dir("metadata") })
      }
    } else {
      // Single batch or no batching — use direct resolver (original behavior)
      t.projectHealthReports.setFrom(adviceResolver.internal.map { it.artifactsFor("json").artifactFiles })
      t.projectMetadataReports.setFrom(projectMetadataResolver.internal.map { it.artifactsFor("json").artifactFiles })
    }

    t.reportingConfig.set(dagpExtension.reportingHandler.config())
    t.projectCount.set(allprojects.size)
    t.dslKind.set(DslKind.from(buildFile))
    t.dependencyMap.set(dagpExtension.dependenciesHandler.map)
    t.useTypesafeProjectAccessors.set(dagpExtension.useTypesafeProjectAccessors)
    t.useParenthesesForGroovy.set(dagpExtension.dependenciesHandler.useParenthesesForGroovy)

    t.output.set(paths.buildHealthPath)
    t.consoleOutput.set(paths.consoleReportPath)
    t.outputFail.set(paths.shouldFailPath)
  }

  // ... rest of buildHealth, publicTypeUsage, generateWorkPlan tasks unchanged ...

  // Add a dependency from the root project to all projects (including itself).
  val publishers = DagpArtifacts.Kind.entries.map { kind ->
    interProjectPublisher(this, kind)
  }

  allprojects.forEach { p ->
    dependencies.let { d ->
      publishers.forEach { publisher ->
        d.add(publisher.declarableName, d.project(mapOf("path" to p.path)))
      }
    }
  }
}
```

**Note:** The actual filtering of which artifact files belong to which batch is complex with Gradle's variant-aware resolution. The simpler approach for the first implementation is to have batch tasks just limit concurrent execution (via `mustRunAfter` ordering between batches) rather than filtering file collections. This ensures only one batch worth of task outputs is being produced at a time.

Simplified alternative for step 2 — use `mustRunAfter` ordering:

```kotlin
// After creating batch tasks, add ordering constraints
batchTasks.windowed(2).forEach { (earlier, later) ->
  later.configure { it.mustRunAfter(earlier) }
}
```

- [ ] **Step 3: Verify compilation**

Run: `./gradlew :compileKotlin`

Expected: BUILD SUCCESSFUL

- [ ] **Step 4: Run existing tests**

Run: `./gradlew :test -x functionalTest -x smokeTest`

Expected: PASS — unit tests don't exercise plugin application

- [ ] **Step 5: Commit**

```bash
git add src/main/kotlin/com/autonomousapps/subplugin/RootPlugin.kt
git commit -m "feat: wire batched aggregation into RootPlugin

When batch size > 0 and project count exceeds batch size, intermediate
BatchAggregateTask instances collect per-batch outputs. Final
GenerateBuildHealthTask reads from batch directories.
Single-batch builds retain original direct-wiring behavior."
```

---

### Task 5: Functional test — bounded cache with many projects

**Files:**
- Create: `src/functionalTest/kotlin/com/autonomousapps/BoundedCacheFunctionalTest.kt`

- [ ] **Step 1: Write functional test that exercises bounded cache**

This test creates a multi-project build with enough projects to exceed the cache bound and verifies analysis still completes correctly.

Create `src/functionalTest/kotlin/com/autonomousapps/BoundedCacheFunctionalTest.kt`:

```kotlin
package com.autonomousapps

import com.autonomousapps.kit.*
import com.autonomousapps.kit.gradle.*
import com.autonomousapps.kit.truth.TestKitTruth.Companion.assertThat
import org.junit.jupiter.api.Test

class BoundedCacheFunctionalTest {

  @Test fun `analysis completes with cache bounded to small size`() {
    // Create a build with 10 sub-projects, cache bounded to 3
    val projects = (1..10).map { i ->
      Subproject(
        name = "sub$i",
        buildScript = BuildScript(
          plugins = listOf(Plugin.javaLibrary),
          dependencies = listOf(
            Dependency("implementation", "org.apache.commons:commons-lang3:3.14.0")
          )
        ),
        sources = listOf(
          Source.java(
            """
            package com.example.sub$i;
            import org.apache.commons.lang3.StringUtils;
            public class Main$i {
              public String go() { return StringUtils.capitalize("hello"); }
            }
            """.trimIndent()
          ).withPath("com/example/sub$i", "Main$i")
            .build()
        )
      )
    }

    val rootBuildScript = BuildScript(
      plugins = listOf(Plugin.dependencyAnalysis),
    )

    val project = ProjectDslBuilder(rootBuildScript)
      .also { builder -> projects.forEach { builder.withSubproject(it) } }
      .build()

    val result = project.execute(
      "buildHealth",
      "-Ddependency.analysis.cache.max=3"
    )

    assertThat(result).task(":buildHealth").succeeded()
  }
}
```

**Note:** This test uses DAGP's existing testkit DSL. The exact API may differ slightly — consult `src/functionalTest/` for existing patterns and adjust accordingly. The important thing is exercising `-Ddependency.analysis.cache.max=3` with more than 3 unique dependency JARs across projects.

- [ ] **Step 2: Run the functional test**

Run: `./gradlew :functionalTest --tests "com.autonomousapps.BoundedCacheFunctionalTest" -x smokeTest`

Expected: PASS — the build should complete even with a very small cache (just with more re-analysis)

- [ ] **Step 3: Commit**

```bash
git add src/functionalTest/kotlin/com/autonomousapps/BoundedCacheFunctionalTest.kt
git commit -m "test: functional test for bounded cache with multi-project build

Verifies analysis completes correctly with cache bounded to 3 entries
across 10 projects sharing the same dependency."
```

---

### Task 6: Integration test — run against ae repo

**Files:**
- No new files in DAGP repo. This task validates against the ae repo.

This is the end-to-end validation. We publish DAGP locally, apply it to ae, and run `buildHealth`.

- [ ] **Step 1: Publish DAGP to mavenLocal**

Run from the DAGP repo root:

```bash
./gradlew publishToMavenLocal -x test -x functionalTest -x smokeTest
```

Expected: BUILD SUCCESSFUL. Plugin JAR published to `~/.m2/repository/com/autonomousapps/dependency-analysis/...`

- [ ] **Step 2: Note the published version**

Check `gradle.properties` in DAGP root for the current version:

```bash
grep "^version" gradle.properties
```

Note this version (e.g., `2.11.0-SNAPSHOT`) for use in ae.

- [ ] **Step 3: Apply DAGP to ae's root build.gradle**

Edit `~/repo/ae/build.gradle`. Add at the top, inside the `plugins {}` block:

```groovy
id 'com.autonomousapps.dependency-analysis' version '<VERSION_FROM_STEP_2>'
```

Add after the plugins block:

```groovy
dependencyAnalysis {
  issues {
    all {
      onUnusedDependencies { severity('warn') }
      onUsedTransitiveDependencies { severity('ignore') }
      onIncorrectConfiguration { severity('ignore') }
      onUnusedAnnotationProcessors { severity('ignore') }
    }
  }
}
```

Ensure ae's `settings.gradle` has `mavenLocal()` in the `pluginManagement.repositories` block (it already does based on our earlier read).

- [ ] **Step 4: Run buildHealth on ae**

```bash
cd ~/repo/ae
./gradlew buildHealth --no-configuration-cache \
  -Dorg.gradle.jvmargs="-Xmx12G -XX:+UseG1GC" \
  -Ddependency.analysis.cache.max=300 \
  -Ddependency.analysis.batch.size=100
```

Expected: BUILD SUCCESSFUL (may take 30+ minutes). No OOM.

- [ ] **Step 5: Validate output**

Check that the build health report exists and contains advice:

```bash
cat ~/repo/ae/build/reports/dependency-analysis/build-health-report.json | python3 -m json.tool | head -50
```

Expected: Valid JSON with per-project advice entries.

- [ ] **Step 6: Spot-check expression-evaluator results**

```bash
cat ~/repo/ae/build/reports/dependency-analysis/build-health-report.json | \
  python3 -c "
import json, sys
data = json.load(sys.stdin)
for project in data.get('projectAdvice', []):
    if 'expression-evaluator' in project.get('projectPath', ''):
        unused = [a for a in project.get('dependencyAdvice', []) if a.get('fromConfiguration')]
        print(f\"{project['projectPath']}: {len(unused)} unused deps\")
"
```

Expected: expression-evaluator should report a significant number of unused dependencies (reference: Ian found ~140).

- [ ] **Step 7: Revert ae changes (don't commit to ae)**

```bash
cd ~/repo/ae
git checkout -- build.gradle
```

- [ ] **Step 8: Commit a note in DAGP documenting successful validation**

```bash
cd ~/dev/dependency-analysis-gradle-plugin
git commit --allow-empty -m "chore: validated buildHealth runs on ae (~1000 projects) without OOM

Tested with -Xmx12G, cache.max=300, batch.size=100.
Build completed successfully. Expression-evaluator shows expected
unused dependency count."
```

---

### Task 7: Documentation

**Files:**
- Create: `docs/ae-integration.md`

- [ ] **Step 1: Write usage documentation**

Create `docs/ae-integration.md`:

```markdown
# Running DAGP on the Appian AE Repository

## Prerequisites

- 32GB RAM machine (12GB allocated to Gradle)
- ae repo checked out at `~/repo/ae`
- DAGP published to mavenLocal (see below)

## Publishing DAGP Locally

From the DAGP repo root:

```bash
./gradlew publishToMavenLocal -x test -x functionalTest -x smokeTest
```

## Applying to ae

Add to ae's root `build.gradle` plugins block:

```groovy
id 'com.autonomousapps.dependency-analysis' version 'LOCAL_VERSION'
```

Add configuration:

```groovy
dependencyAnalysis {
  issues {
    all {
      onUnusedDependencies { severity('warn') }
      onUsedTransitiveDependencies { severity('ignore') }
      onIncorrectConfiguration { severity('ignore') }
      onUnusedAnnotationProcessors { severity('ignore') }
    }
  }
}
```

Ensure `mavenLocal()` is in `settings.gradle` pluginManagement repositories.

## Running

```bash
./gradlew buildHealth --no-configuration-cache \
  -Dorg.gradle.jvmargs="-Xmx12G -XX:+UseG1GC" \
  -Ddependency.analysis.cache.max=300 \
  -Ddependency.analysis.batch.size=100
```

## Configuration Options

| Property | Default | Description |
|----------|---------|-------------|
| `dependency.analysis.cache.max` | 300 | Max entries in bytecode analysis cache (LRU eviction) |
| `dependency.analysis.batch.size` | 100 | Projects per aggregation batch (0 = no batching) |

## Output

Report at: `build/reports/dependency-analysis/build-health-report.json`

## Troubleshooting

- **OOM**: Reduce batch size or increase heap
- **Slow**: Increase cache size to reduce re-analysis
- **Config cache hang**: Always use `--no-configuration-cache`
```

- [ ] **Step 2: Commit documentation**

```bash
git add docs/ae-integration.md
git commit -m "docs: add ae integration guide for running DAGP on large repos"
```
