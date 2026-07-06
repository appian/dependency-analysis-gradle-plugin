# DAGP Fork — Appian AE Scalability

**Fork of:** [autonomousapps/dependency-analysis-gradle-plugin](https://github.com/autonomousapps/dependency-analysis-gradle-plugin)  
**Fork repo:** [marceltft/dependency-analysis-gradle-plugin](https://github.com/marceltft/dependency-analysis-gradle-plugin)  
**Task:** LCP-61433 (removal PR), LCP-60585 (spike/plugin work)

---

## What This Fork Does

Makes DAGP work on the Appian `ae` monorepo (~1000 Gradle projects) without OOM, and fixes bugs encountered during analysis.

## Changes From Upstream

| Commit | File | What |
|--------|------|------|
| `c71076f8` | `services/InMemoryCache.kt` | Bound Caffeine caches to 300 entries (LRU eviction). Was unbounded → OOM. |
| `dced1fe1` | `internal/ProjectBatcher.kt`, `Flags.kt` | Partition projects into batches. Flag: `-Ddependency.analysis.batch.size=100` |
| `6a5b36e0` | `tasks/BatchAggregateTask.kt` | Intermediate task that collects batch outputs to disk |
| `98242b12` | `subplugin/RootPlugin.kt` | Wires batch tasks with `mustRunAfter` ordering when project count > batch size |
| `b98ad7fa` | `internal/transform/StandardTransform.kt` | Fix `ConcurrentModificationException` — was iterating mutable sets while modifying them |

## How to Build & Publish

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 21) ./gradlew :publishToMavenLocal \
  -x test -x functionalTest -x smokeTest -x signMavenPublication
```

Publishes as version `3.16.1-SNAPSHOT` to `~/.m2/repository`.

## How to Run on AE

See `docs/ae-integration.md` for full instructions. Quick version:

1. Add to ae's `settings.gradle`:
```groovy
id "org.jetbrains.kotlin.jvm" version "2.1.10" apply false
id "com.autonomousapps.build-health" version "3.16.1-SNAPSHOT"
```

2. Add to ae's `build.gradle`:
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

3. Run:
```bash
./gradlew generateBuildHealth --no-configuration-cache --no-build-cache --continue \
  -Dorg.gradle.jvmargs="-Xmx12G -XX:+UseG1GC" \
  -Ddependency.analysis.cache.max=300 \
  -Ddependency.analysis.batch.size=100 \
  -Pdependency.analysis.project.includes='^(?!.*(tempo|lcp-api-server-generated|appian-gwt-components)).*$' \
  -x :appian-libraries:gwt:appian-gwt-components:compressJavascript
```

## Results on AE

- **728 projects analyzed** in ~14 minutes, 12GB heap, no OOM
- **1,663 unused dependencies detected** initially
- **807 removed** (verified by compilation), **856 remaining** (false positives + pattern mismatches)

## Known Issues / Next Steps

See `docs/fixing-false-positives.md` for detailed breakdown of:

1. **False positives** — runtime-only deps, transitive exposure via `api`, annotation-only deps
2. **Pattern mismatches** — `globalDeps()` plural lists, `projectDeps()`, closures, conditionals
3. **Priority order** for fixes

## Repo Structure (key files)

```
src/main/kotlin/com/autonomousapps/
├── Flags.kt                          # Configuration flags (cache size, batch size)
├── internal/
│   ├── ProjectBatcher.kt             # Batch partitioning logic
│   └── transform/StandardTransform.kt # Advice computation (CME fix here)
├── services/
│   └── InMemoryCache.kt              # Bounded Caffeine cache
├── subplugin/
│   └── RootPlugin.kt                 # Root project wiring (batching here)
└── tasks/
    ├── BatchAggregateTask.kt         # Intermediate batch collection
    ├── ComputeUsagesTask.kt          # Core "is dep used?" logic
    ├── ComputeAdviceTask.kt          # Generates remove/add/change advice
    └── GenerateBuildHealthTask.kt    # Final aggregation into report
```

## Remotes

- `origin` → `git@github.com:marceltft/dependency-analysis-gradle-plugin.git` (our fork)
- `upstream` → `https://github.com/autonomousapps/dependency-analysis-gradle-plugin` (original)

## Requirements

- Java 21 (Gradle 9.6 requirement)
- Gradle 9.6 (bundled via wrapper)
