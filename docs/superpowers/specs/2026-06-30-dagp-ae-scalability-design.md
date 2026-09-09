# DAGP Scalability for Appian AE Repository

**Date:** 2026-06-30  
**Status:** Approved  
**Goal:** Make DAGP run reliably against the ae monorepo (~1000+ Gradle projects) without OOM, producing accurate unused-dependency reports.

---

## Context

The Appian `ae` repository is a massive Gradle monorepo:
- ~1000+ Gradle build files (232 top-level `appian-libraries`, each with recursive sub-projects)
- Dynamic project inclusion via `projectAndSubProjects()` in `settings.gradle`
- Custom dependency DSL: `globalDep('group:artifact')` resolving versions from `globalDependencies.groovy`
- Custom configurations: `unitTestImplementation`, `integrationTestImplementation`, `systemTestImplementation`
- Convention plugins: `appian-app`, `appian-java-conventions`, `tiered-ci-v3`
- Current Gradle daemon: 4GB heap, parallel builds, configuration cache enabled

Running upstream DAGP on ae crashes with OutOfMemoryError before completing analysis.

### Why This Matters

- ae triggers ~260 CI jobs per PR based on the dependency graph
- Unused dependencies cause unnecessary downstream job triggers
- Prior spike found 140 unused deps in expression-evaluator alone, and ~120 unnecessary integration-tier triggers in `/test`
- Cleaning the dependency graph directly reduces CI cycle time and compute cost

---

## Problem Analysis: Why DAGP OOMs on ae

Three sources of unbounded memory growth:

### 1. Unbounded InMemoryCache

DAGP uses a shared Caffeine-backed `BuildService` (`InMemoryCache`) to cache:
- `ExplodingJar` — bytecode analysis results (class lists, constants, exceptions, reflective accesses) per JAR
- `KotlinCapabilities` — inline members, typealiases per Kotlin JAR
- `AnnotationProcessorDependency` — processor metadata per annotation processor JAR

Default cache size is `-1` (unlimited). With 1000+ projects sharing 500-800 unique external JARs, each containing potentially thousands of analyzed classes, this cache grows without bound.

### 2. Simultaneous Project Analysis

DAGP's `RootPlugin` registers inter-project resolvers that wire ALL projects as dependencies of root aggregation tasks. When Gradle executes with `parallel=true` and `workers.max=4`, multiple projects are being analyzed concurrently. Each project's analysis produces in-memory intermediates: source import sets, resolved dependency graphs, exploded class lists, usage computation results.

### 3. Aggregation Holds All Outputs

The `generateBuildHealth` task depends on project health reports from every project via `adviceResolver.artifactFilesProvider()`. Gradle must keep all upstream task outputs reachable until aggregation completes, preventing GC of per-project data.

### Key Architectural Insight

DAGP's analysis is fundamentally per-project. Each project independently:
1. Resolves its dependencies via Gradle's Configuration API
2. Explodes JAR bytecode to find what classes/capabilities each dependency provides
3. Parses source files for imports and usages
4. Computes which dependencies are actually used

Cross-project data sharing is limited to the InMemoryCache (avoiding redundant JAR re-analysis). The aggregation step only needs final JSON reports, not intermediate data.

---

## Solution Architecture

Three layers, each independently valuable and incrementally deployable:

### Layer 1: Bounded InMemoryCache

**Change:** Set explicit `maximumSize` on all Caffeine caches in `InMemoryCache`.

**Configuration (via Gradle property `dependency.analysis.cache.size`):**
- `ExplodingJar` cache: 300 entries (LRU eviction)
- `KotlinCapabilities` cache: 300 entries
- `AnnotationProcessorDependency` cache: 50 entries

**Trade-off:** Evicted JARs get re-analyzed if needed again by a later project. Costs CPU time, saves memory. Acceptable under "completeness over speed" constraint.

**Implementation:**
- Modify `InMemoryCache.newCache()` to always set a positive `maximumSize`
- Expose per-cache size configuration via `InMemoryCache.Params`
- Default: 300 (sufficient for most builds, safe for ae)

### Layer 2: Project-Group Batching

**Change:** Instead of wiring all ~1000 projects as direct dependencies of the root aggregation task, process them in configurable batches.

**Mechanism:**
1. `ProjectBatcher` partitions `allprojects` into ordered groups (default batch size: 100)
2. Each batch gets an intermediate aggregation task that collects per-project outputs to disk
3. Final `generateBuildHealth` reads from batch output files (not individual project outputs)
4. Gradle's task graph naturally limits concurrent work to one batch at a time

**Configuration:** `dependency.analysis.batch.size=100` (Gradle property)

**Implementation:**
- New `ProjectBatcher` utility class in `com.autonomousapps.internal`
- Modify `RootPlugin.configureRootProject()` to use batched wiring
- Each batch intermediate task writes partial results as JSON to `build/dagp-batches/batch-N/`
- Final aggregation reads from all batch directories

**Trade-off:** Introduces a level of indirection in the task graph. Slightly more complex than the current direct wiring, but fundamentally compatible with DAGP's architecture.

### Layer 3: Disk-Backed Intermediate Results

**Change:** Ensure per-project analysis tasks eagerly flush outputs to disk and don't retain large object graphs after task completion.

**Implementation:**
- Verify all per-project output tasks use `RegularFileProperty` (already the case in upstream DAGP)
- Ensure batch intermediate tasks read from file providers lazily
- Add explicit nulling of large collections in task actions after writing to disk (where safe)
- Configure Gradle worker isolation to minimize cross-task memory sharing

**This layer is mostly verification that DAGP's existing patterns work correctly at scale, with minor hardening.**

---

## Appian Syntax Support

### Analysis Phase (MVP — already works)

DAGP reads dependencies from Gradle's resolved Configuration API (`ConfigurationContainer.asSequence()`), not by parsing build script text. This means:

- `globalDep('group:artifact')` — Gradle resolves this via `globalDependencies.groovy` before DAGP sees it. **Already works.**
- `project(':appian-libraries:...')` — standard Gradle project dependencies. **Already works.**
- `projectAndSubProjects(...)` — resolved at settings evaluation time. **Already works.**

### What needs explicit support for analysis:

**Custom configuration names** — DAGP's `ConfigurationNames` derives its set of analyzable configurations from the project's registered source sets (via `SourceSetContainer`). Since ae's convention plugins (`appian-app`, `appian-java-conventions`) register custom source sets (`unitTest`, `integrationTest`, `systemTest`), the corresponding configurations are auto-discovered:
- `unitTestImplementation` / `unitTestCompileOnly` / `unitTestRuntimeOnly`
- `integrationTestImplementation` / `integrationTestCompileOnly` / `integrationTestRuntimeOnly`
- `systemTestImplementation` / `systemTestCompileOnly` / `systemTestRuntimeOnly`

**No DAGP code changes needed** for configuration discovery. If a source set exists in Gradle's `SourceSetContainer`, DAGP will find it and analyze its configurations.

**One potential issue:** If ae uses configurations not backed by a source set (e.g., `iosSource` in expression-evaluator is an artifact configuration, not a dependency bucket), DAGP should correctly ignore these. Needs verification.

### Rewrite Phase (Phase 2 — not MVP)

The ANTLR `GradleScript` grammar needs to recognize `globalDep(...)` as a valid dependency declaration for the `fixDependencies` task to work. This includes:
- Grammar production for `globalDep('group:artifact')` as an `externalDependency`
- `AdvicePrinter` formatting fix advice using `globalDep()` syntax
- Handling of ae's `/// OWNERSHIP` comments that shouldn't be disturbed

**Explicitly deferred.** MVP is report-only.

---

## Configuration & Integration with ae

### Applying DAGP to ae

Add to ae's root `build.gradle`:

```groovy
plugins {
  id 'com.autonomousapps.dependency-analysis' version 'local-SNAPSHOT'
}

dependencyAnalysis {
  issues {
    all {
      onUnusedDependencies { severity('warn') }
      onUsedTransitiveDependencies { severity('ignore') }
      onIncorrectConfiguration { severity('ignore') }
      onUnusedAnnotationProcessors { severity('ignore') }
    }
  }
  // DAGP auto-discovers source sets from Gradle's SourceSetContainer.
  // ae's convention plugins register unitTest, integrationTest, systemTest as source sets,
  // so DAGP will find and analyze them automatically. No manual registration needed.
}
```

### Gradle Properties for ae

```properties
# In ae/gradle.properties or command-line
dependency.analysis.cache.size=300
dependency.analysis.batch.size=100
```

### Invocation

```bash
./gradlew buildHealth --no-configuration-cache -Dorg.gradle.jvmargs="-Xmx12G -XX:+UseG1GC"
```

Notes:
- `--no-configuration-cache`: avoids known hang issue with config cache + analysis plugins
- `-Xmx12G`: provides comfortable headroom on 32GB machine with bounded caches + batching
- G1GC: better for large heaps with mixed short/long-lived objects

### Expected Output

`build/reports/dependency-analysis/build-health-report.json` containing per-project advice:
- Which dependencies are declared but unused
- Which configuration each unused dep is declared on
- The project path where the unused dep lives

---

## Scope & Boundaries

### In scope (MVP):
- Bounded InMemoryCache with configurable limits
- Project-group batching for aggregation
- Disk-backed intermediates verification
- Custom configuration name registration for ae
- Report-only unused-dependency detection
- Run on developer machine (32GB RAM, 12GB heap)

### Out of scope (Phase 2+):
- `fixDependencies` / autofix (requires grammar changes for `globalDep()`)
- CI enforcement (GitLab job integration)
- Config cache compatibility
- `trex-server` special handling
- Used-transitive / incorrect-configuration checks
- Publishing plugin to Artifactory
- Upstream contribution of changes

---

## Success Criteria

1. `./gradlew buildHealth` completes on ae without OOM (12GB heap)
2. Produces a valid build-health report covering all ~1000 projects
3. Report correctly identifies unused dependencies (validated against expression-evaluator's known 140 unused deps)
4. Peak memory usage stays below 10GB during the run
5. No modifications required to ae's existing build files beyond adding the plugin application

---

## Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| Batching breaks cross-project dependency resolution | DAGP's analysis is per-project; cross-project data only flows through final reports |
| Cache eviction causes excessive re-analysis, making runs extremely slow | Start with generous cache size (300); monitor hit rates; adjust based on actual unique JAR count |
| ae's convention plugins interfere with DAGP's project detection | DAGP only analyzes projects with `java-library` or similar plugins applied; ae uses these |
| Custom configurations aren't detected | Verify with expression-evaluator first (known ground truth) |
| Gradle 8.12 (ae) vs DAGP built against Gradle 9.6 compatibility | DAGP supports Gradle 8.x; verify minimum version in tests |
