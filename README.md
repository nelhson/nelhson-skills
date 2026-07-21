# nelhson-skills

Personal Claude Code **skills marketplace** for Kotlin development.

Each technology is a separate plugin (skill pack) that can be enabled independently in any project:

- **android-kotlin** — my own Android development skills (Compose, ViewModel, coroutines, architecture, testing, data layer, Gradle).
- **backend-kotlin** — Kotlin server-side skills (Ktor, Spring Boot in Kotlin, server coroutines).
- **kotlin-common** — technology-agnostic Kotlin language skills (modern 2.x language features, modernization hints).
- **android-official** — curated skills referenced live from Google's official [android/skills](https://github.com/android/skills) repo.
- **kotlin-official** — tooling/migration/JPA skills referenced live from JetBrains' official [Kotlin/kotlin-agent-skills](https://github.com/Kotlin/kotlin-agent-skills) repo.
- **kmp-community** — curated KMP/Compose Multiplatform skills referenced from [mmiani/kotlin-kmp-claude-agent-skills](https://github.com/mmiani/kotlin-kmp-claude-agent-skills) (community, Apache-2.0), pinned to a reviewed commit.

## Quick start

```powershell
# 1. Validate (optional, after edits)
claude plugin validate .

# 2. Register the marketplace (once per machine)
claude plugin marketplace add E:\projects\kotlin-server\nelhson-skills
#    or inside a Claude Code session:
#    /plugin marketplace add E:\projects\kotlin-server\nelhson-skills

# On any other machine, register straight from GitHub (HTTPS):
claude plugin marketplace add https://github.com/nelhson/nelhson-skills.git
```

Then, in a Claude Code session:

```
/plugin install android-kotlin@nelhson-skills
/plugin install backend-kotlin@nelhson-skills
/plugin install kotlin-common@nelhson-skills
/plugin install android-official@nelhson-skills
/plugin install kotlin-official@nelhson-skills
/plugin install kmp-community@nelhson-skills
```

Skills are invoked as `/<plugin>:<skill>`, e.g. `/android-kotlin:compose-ui` or `/backend-kotlin:ktor-server`, and are also auto-invoked by Claude when their description matches the task.

## Structure

```
nelhson-skills/
├── .claude-plugin/marketplace.json     # marketplace catalog — one entry per plugin
├── plugins/
│   ├── android-kotlin/
│   │   ├── .claude-plugin/plugin.json  # plugin manifest
│   │   └── skills/<skill>/SKILL.md     # one directory per skill
│   └── backend-kotlin/
│       ├── .claude-plugin/plugin.json
│       └── skills/<skill>/SKILL.md
└── README.md
```

- `marketplace.json` — catalog. Local plugin sources are relative paths from the repo root (`"source": "./plugins/android-kotlin"`).
- `plugin.json` — plugin manifest; only `name` is required.
- `SKILL.md` — skill content with YAML frontmatter (`name`, `description`). The `description` drives auto-invocation, so phrase it as "…Use when…".
- Skills may have supporting files (`reference.md`, `scripts/`) beside `SKILL.md`, but must never reference paths outside their plugin directory — plugins are copied wholesale into the plugin cache on install.

## Plugins & skills

### android-kotlin

| Skill | Purpose |
|---|---|
| `android-accessibility` | Auditing/fixing accessibility, especially in Compose |
| `android-architecture` | Clean Architecture + Hilt project structure |
| `android-coroutines` | Production coroutine rules on Android |
| `android-data-layer` | Repository pattern, Room, Retrofit, offline-first |
| `android-emulator-skill` | Build/test/emulator automation scripts |
| `android-gradle-logic` | Convention plugins and version catalogs |
| `android-retrofit` | Type-safe networking with Retrofit |
| `android-testing` | Unit/integration/Hilt/screenshot testing strategy |
| `android-viewmodel` | StateFlow UI state, one-off events modeled as state |
| `coil-compose` | Image loading with Coil in Compose |
| `compose-navigation` | Navigation Compose patterns |
| `compose-performance-audit` | Diagnosing recomposition/jank issues |
| `compose-ui` | State hoisting, performance, theming |
| `gradle-build-performance` | Build speed debugging and optimization |
| `kotlin-concurrency-expert` | Coroutine review and remediation (canonical rules live in `android-coroutines`) |
| `rxjava-to-coroutines-migration` | RxJava → Coroutines/Flow migration |

> `xml-to-compose-migration` was retired in favor of the more rigorous external `android-official` migration skill (`migrate-xml-views-to-jetpack-compose`).

### backend-kotlin

| Skill | Purpose |
|---|---|
| `ktor-server` | Ktor routing organization, plugins, Koin DI, config, testApplication |
| `spring-boot-kotlin` | Kotlin-idiomatic Spring Boot: DI, compiler plugins, entities vs DTOs, coroutine controllers |
| `kotlin-backend-coroutines` | Server coroutine patterns: dispatchers, fan-out, Flow streaming, timeouts, cancellation |

### kotlin-common

| Skill | Purpose |
|---|---|
| `kotlin-modern-features` | Advisory catalog of modern Kotlin 2.x language features (context parameters, guard conditions, explicit backing fields, …) with stability/version info — suggests them while writing or reviewing Kotlin code |

### android-official (external, referenced)

Curated skill groups pulled live from `github.com/android/skills` (jetpack-compose, navigation, testing, performance, profilers/Perfetto, build/AGP-9 upgrade, security, system). Notes:

- Google's skills target multiple agents and a few reference their `android` CLI or write `AGENTS.md` — treat those parts as reference content.
- `jetpack-compose/theming/styles` is **deliberately not referenced**: it requires experimental Compose APIs (`ExperimentalFoundationStyleApi`, alpha BOM), which conflicts with the do-not-use-experimental convention.
- The AGP-9 skill here is Android-only; the KMP AGP-9 migration lives in `kotlin-official` — their descriptions self-disambiguate.
- The repo is updated by Google automation and referenced **by path**, so an upstream re-org can silently drop skills — periodically re-check the referenced paths after `/plugin marketplace update` (see Maintenance notes).

### kotlin-official (external, referenced)

Official JetBrains skills pulled live from `github.com/Kotlin/kotlin-agent-skills` (Apache-2.0, JetBrains Incubator). Tooling/migration focused:

| Skill | Purpose |
|---|---|
| `kotlin-tooling-agp9-migration` | Migrate KMP/Android projects to Android Gradle Plugin 9.0+ |
| `kotlin-tooling-cocoapods-spm-migration` | CocoaPods → Swift Package Manager migration for KMP iOS targets |
| `kotlin-tooling-native-build-performance` | Kotlin/Native build speed diagnosis |
| `kotlin-tooling-immutable-collections-0-5-x-migration` | kotlinx.collections.immutable 0.5.x migration |
| `kotlin-tooling-java-to-kotlin` | Java → Kotlin conversion (12 framework guides: Spring, Hibernate, Dagger/Hilt, …) |
| `kotlin-backend-jpa-entity-mapping` | JPA/Hibernate entity design in Kotlin (equals/hashCode, N+1, lazy-init) — deeper than the short rule in `spring-boot-kotlin`, so both coexist |

### kmp-community (external, referenced, sha-pinned)

Curated subset (8 of 15) from `github.com/mmiani/kotlin-kmp-claude-agent-skills` (community-maintained, Apache-2.0, grounded in official Android/Kotlin/Compose guidance). Neither Google (`android/skills`) nor JetBrains publish CMP UI skills — this fills that gap. Kept: the genuinely KMP-unique skills — `kmp-bridges` (expect/actual), `data-layer`, `state-management`, `navigation-compose-multiplatform`, `app-links-and-deep-links`, `gradle-governance`, `testing-kmp`, `modularization`.

Deliberately excluded (7): `kmp-code-review` (977 lines) and `feature-implementation` (993 lines) — huge generic rubrics that collide with the built-in `/code-review` and normal implementation flow; `ui-adaptive-resources` — superseded by `android-official`'s better `adaptive` skill; `ui-compose-multiplatform`, `architecture-review`, `bugfix`, `refactor-safety` — mostly platform-generic content already covered locally.

**Pinned to sha `b73b7c1f8bad9c3068a619aa69d383d40809c248`** (reviewed 2026-07-21): single-maintainer repo with known editing defects and non-skill payload (`agents/`, `commands/`, `hooks/`, `settings.local.json`) at the root — review upstream diffs before bumping the pin.

## Adding a new technology

1. Create `plugins/<tech>/.claude-plugin/plugin.json` (at minimum `{"name": "<tech>"}`).
2. Add skills under `plugins/<tech>/skills/<skill-name>/SKILL.md` (kebab-case names).
3. Append an entry to `plugins[]` in `.claude-plugin/marketplace.json`:
   ```json
   { "name": "<tech>", "source": "./plugins/<tech>", "description": "..." }
   ```
4. Validate and commit:
   ```powershell
   claude plugin validate .
   git add . ; git commit -m "Add <tech> plugin"
   ```
5. Refresh consumers: `/plugin marketplace update nelhson-skills`, then `/plugin install <tech>@nelhson-skills`.

## External skill sources

Two mechanisms, used for different purposes. Rule of thumb: **reference when you don't modify, vendor when you customize — never both for the same skill** (it produces duplicate skill listings).

### a) Referenced (stays fresh upstream)

A marketplace entry with a git `url` source, like `android-official`:

```json
{
  "name": "android-official",
  "source": { "source": "url", "url": "https://github.com/android/skills.git" },
  "strict": false,
  "skills": ["./jetpack-compose", "./navigation", "..."]
}
```

- `strict: false` — required when the external repo has no `plugin.json` (it's a plain skills collection).
- `skills` — list of directories containing `<skill>/SKILL.md` to load (one level of nesting).
- Use the `url` source type with an HTTPS URL. The `{"source": "github", "repo": "owner/repo"}` shorthand clones over **SSH** by default, which fails on machines without GitHub SSH keys (set `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` if you prefer the shorthand).
- To pin a known-good state, add `"sha": "<40-char commit>"` to the source object.
- To add another external repo, append another entry. For a single skill from a large repo, use a sparse source:
  ```json
  { "source": "git-subdir", "url": "https://github.com/android/skills.git", "path": "navigation/navigation-3" }
  ```
- Google's repo has more categories than currently referenced (camera, security, wear, xr, play, devtools, ...) — add their paths to the `skills` array when needed.
- Refresh with `/plugin marketplace update nelhson-skills` (new upstream commits count as new plugin versions because no `version` is pinned).

### b) Vendored (copied for customization)

Copy the external skill directory into the **owning technology plugin** at `plugins/<tech>/skills/<skill-name>/` (it must live inside a plugin to be installed), then record provenance in `UPSTREAM.md` inside the skill directory:

```markdown
# Vendored from
- Repo: https://github.com/android/skills
- Path: jetpack-compose/theming/styles
- Commit: <sha at copy time>
- Copied: 2026-07-19
- Local changes: <summary>
```

To diff against upstream later: clone the source repo and run
`git diff <recorded-sha>..HEAD -- <path>`.

## Versioning

No `version` fields are set anywhere — **every git commit is a new version** (Claude Code falls back to the commit SHA). `/plugin update <name>` picks up the latest commit; no manual version bumps needed. To switch to pinned releases later, add `"version"` to `plugin.json` (not to both plugin.json and the marketplace entry — plugin.json silently wins).

## Maintenance notes

- After editing skills: commit, then `/plugin marketplace update nelhson-skills` and `/plugin update <plugin>` (or restart Claude Code). Installed plugins are cached copies — edits are **not** live.
- Validate any change with `claude plugin validate .` before committing.
- Periodically (e.g. monthly) run `/plugin marketplace update nelhson-skills` and confirm the `android-official` referenced paths still resolve — Google's automation re-organizes that repo and path-based references fail silently. `kmp-community` is sha-pinned; review its upstream diff before bumping.
- The original personal copies of the android skills lived in `C:\Users\boris\.claude\skills\`; once the plugin versions are confirmed working, delete the migrated ones there to avoid duplicate skill listings (only `owasp-security` intentionally remains personal).
