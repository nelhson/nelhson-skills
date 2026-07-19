# nelhson-skills

Personal Claude Code **skills marketplace** for Kotlin development.

Each technology is a separate plugin (skill pack) that can be enabled independently in any project:

- **android-kotlin** — my own Android development skills (Compose, ViewModel, coroutines, architecture, testing, data layer, Gradle).
- **backend-kotlin** — Kotlin server-side skills (Ktor, Spring Boot in Kotlin, server coroutines).
- **android-official** — curated skills referenced live from Google's official [android/skills](https://github.com/android/skills) repo.

## Quick start

```powershell
# 1. Validate (optional, after edits)
claude plugin validate .

# 2. Register the marketplace (once per machine)
claude plugin marketplace add E:\projects\kotlin-server\nelhson-skills
#    or inside a Claude Code session:
#    /plugin marketplace add E:\projects\kotlin-server\nelhson-skills
```

Then, in a Claude Code session:

```
/plugin install android-kotlin@nelhson-skills
/plugin install backend-kotlin@nelhson-skills
/plugin install android-official@nelhson-skills
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
| `android-viewmodel` | StateFlow UI state, SharedFlow events |
| `coil-compose` | Image loading with Coil in Compose |
| `compose-navigation` | Navigation Compose patterns |
| `compose-performance-audit` | Diagnosing recomposition/jank issues |
| `compose-ui` | State hoisting, performance, theming |
| `gradle-build-performance` | Build speed debugging and optimization |
| `kotlin-concurrency-expert` | Coroutine review and remediation |
| `rxjava-to-coroutines-migration` | RxJava → Coroutines/Flow migration |
| `xml-to-compose-migration` | XML Views → Compose migration |

### backend-kotlin

| Skill | Purpose |
|---|---|
| `ktor-server` | Ktor routing organization, plugins, Koin DI, config, testApplication |
| `spring-boot-kotlin` | Kotlin-idiomatic Spring Boot: DI, compiler plugins, entities vs DTOs, coroutine controllers |
| `kotlin-backend-coroutines` | Server coroutine patterns: dispatchers, fan-out, Flow streaming, timeouts, cancellation |

### android-official (external, referenced)

Curated skill groups pulled live from `github.com/android/skills` (jetpack-compose, navigation, testing, performance, system). Note: Google's skills target multiple agents and a few reference their `android` CLI — treat those parts as reference content.

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

A marketplace entry with a `github` source, like `android-official`:

```json
{
  "name": "android-official",
  "source": { "source": "github", "repo": "android/skills" },
  "strict": false,
  "skills": ["./jetpack-compose", "./navigation", "..."]
}
```

- `strict: false` — required when the external repo has no `plugin.json` (it's a plain skills collection).
- `skills` — list of directories containing `<skill>/SKILL.md` to load.
- To pin a known-good state, add `"sha": "<40-char commit>"` to the source object.
- To add another external repo, append another entry. For a single skill from a large repo, use a sparse source:
  ```json
  { "source": "git-subdir", "url": "android/skills", "path": "navigation/navigation-3" }
  ```
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
- The original personal copies of the android skills lived in `C:\Users\boris\.claude\skills\`; once the plugin versions are confirmed working, delete the migrated ones there to avoid duplicate skill listings (only `owasp-security` intentionally remains personal).
