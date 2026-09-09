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

Contains per-project advice:
- Which dependencies are declared but unused
- Which configuration each unused dep is declared on
- The project path where the unused dep lives

## Troubleshooting

- **OOM**: Reduce batch size or increase heap (`-Xmx16G`)
- **Slow**: Increase cache size to reduce re-analysis (`-Ddependency.analysis.cache.max=500`)
- **Config cache hang**: Always use `--no-configuration-cache`
- **Plugin not found**: Verify `mavenLocal()` is in `settings.gradle` pluginManagement repositories

## How It Works

DAGP analyzes each project independently using Gradle's Configuration API to discover resolved dependencies, then explodes JAR bytecode to determine which classes/capabilities each dependency provides, and finally matches that against source code imports and usages.

To handle ae's scale (~1000 projects):
- The InMemoryCache (shared bytecode analysis results) is bounded with LRU eviction
- Projects are processed in batches with sequential ordering constraints
- Both settings are configurable via system properties
