# Configuration Cache Slowdown with `fromResolutionResult()`

This repository reproduces a Gradle bug where configuration cache (CC) store times become extremely slow when running `generatePomFileFor*Publication` or `generateMetadataFileFor*Publication` tasks with `versionMapping { fromResolutionResult() }` enabled.

**Gradle Issue**: tbd

## The Problem

Running POM or GMM generation tasks is very slow to store in configuration cache when `fromResolutionResult()` is enabled - even with the tasks disabled. This is an issue if you have `publishToMavenLocal` as part of your build.

I think the root cause is `VersionMappingComponentDependencyResolver.maybeResolveVersion()` - I believe it performs a **full dependency graph traversal for every single dependency/constraint** via `eachElement()`, with no caching between calls.

```
Configuration Cache Serialization
  └── Cached$Deferred.writeReplace()
        └── GenerateMavenPom / GenerateModuleMetadata
              └── MavenComponentParser.getDependenciesForVariant()
                    └── VersionMappingComponentDependencyResolver.maybeResolveVersion()
                          └── DefaultResolvedComponentResult.eachElement()  // FULL GRAPH TRAVERSAL
```

## Reproducing

```bash
# Clone this repo (150 subprojects with 4000 dependency constraints each)
git clone <this-repo>
cd gmm-cc-reproducer

# Test POM generation - observe slow CC store time
rm -rf .gradle/configuration-cache && ./gradlew generatePomFileForMavenPublication --configuration-cache --dry-run --scan

# Or test GMM generation
rm -rf .gradle/configuration-cache && ./gradlew generateMetadataFileForMavenPublication --configuration-cache --dry-run --scan
```

**To see the difference:** Comment out `fromResolutionResult()` in `build.gradle` and run again.

## Store Time Comparison

| Task | `fromResolutionResult()` | Store Time |
|------|--------------------------|------------|
| `generatePomFileForMavenPublication` | No | ~1s        |
| `generatePomFileForMavenPublication` | Yes | **~30s**   |
| `generateMetadataFileForMavenPublication` | No | ~1s        |
| `generateMetadataFileForMavenPublication` | Yes | **~30s**   |

> Note: With fromResolutionResult() and configuration cache the total time is around 40s without configuration cache the time is only 1s

## Why Disabling Tasks Doesn't Help

```groovy
tasks.withType(GenerateModuleMetadata).configureEach { enabled = false }
tasks.withType(GenerateMavenPom).configureEach { enabled = false }
```

**This doesn't fix the slowdown** because disabled tasks are still in the task graph and still get serialized by CC. The expensive computation happens during serialization regardless of whether the task is enabled.