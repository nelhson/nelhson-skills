# Gradle Build Performance — Bottleneck Reference

Cause/fix tables for diagnosing specific bottleneck symptoms. Read the section
matching the symptom identified in the build scan or profile report.

## Slow Configuration Phase

**Symptoms**: Build scan shows long "Configuring build" time

**Causes & Fixes**:
| Cause | Fix |
|-------|-----|
| Eager task creation | Use `tasks.register()` instead of `tasks.create()` |
| buildSrc with many dependencies | Migrate to Convention Plugins with `includeBuild` |
| File I/O in build scripts | Use `providers.fileContents()` |
| Network calls in plugins | Cache results or use offline mode |

## Slow Compilation

**Symptoms**: `:app:compileDebugKotlin` takes too long

**Causes & Fixes**:
| Cause | Fix |
|-------|-----|
| Non-incremental changes | Avoid `build.gradle.kts` changes that invalidate cache |
| Large modules | Break into smaller feature modules |
| Excessive kapt usage | Migrate to KSP |
| Kotlin compiler memory | Increase `kotlin.daemon.jvmargs` |

## Cache Misses

**Symptoms**: Tasks always rerun despite no changes

**Causes & Fixes**:
| Cause | Fix |
|-------|-----|
| Unstable task inputs | Use `@PathSensitive`, `@NormalizeLineEndings` |
| Absolute paths in outputs | Use relative paths |
| Missing `@CacheableTask` | Add annotation to custom tasks |
| Different JDK versions | Standardize JDK across environments |
