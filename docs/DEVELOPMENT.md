# Westorix Platform — Development

## Prerequisites

- **JDK 25** (`java -version` must report 25 or later — the enforcer fails the build otherwise)
- Maven 3.9+, or just use the committed wrapper
- **On Windows: a UTF-8 console.** Run `chcp 65001` once per session, or set the terminal to UTF-8.
  Comments, exception messages and test display names are Russian; a legacy OEM code page renders
  them as `?????`. The build itself is already forced to UTF-8 (`.mvn/jvm.config` for Maven's own
  JVM, surefire `argLine` for the forked test JVM) — only the terminal's own code page is left.

## Build

Unit tests only — fast, no external services:

```bash
./mvnw test
```

Everything, including integration tests:

```bash
./mvnw verify
```

**`verify` needs Docker running.** Integration tests (`*IT.java`, run by failsafe) start real
PostgreSQL and Redis via Testcontainers — not H2, not a fake. Transactions, rollback, read-only
connections and Flyway checksum validation are exactly where engines differ, so an adapter verified
against a substitute is an unverified adapter (ADR-0008).

A targeted build while working on one module (CLAUDE.md §28 — prefer these over full builds):

```bash
./mvnw -pl game-platform/game-engine -am test
```

Run only the architecture rules:

```bash
./mvnw -pl tooling/architecture-tests -am test
```

`clean` is not needed for ordinary work. Use `./mvnw -U clean verify` at milestones.

## Reactor

```text
westorix-platform                     me.westorix:westorix-platform
├── platform/
│   └── platform-api                  identity, messaging envelope, cache port; JDK only
├── infrastructure/
│   ├── cache                         Caffeine L1, TieredCache
│   ├── database                      HikariCP + PostgreSQL + Flyway migrations
│   └── messaging                     Redis pub/sub transport (Lettuce)
├── game-platform/
│   ├── game-api                      GameDefinition, GameState, GameInstance, GameRuntime, GameSystem
│   └── game-engine                   GameLifecycle, GameSystems, EngineGameRuntime, definition registry
└── tooling/
    ├── testkit                       ArchUnit rule library
    └── architecture-tests            runs the rules over the whole platform
```

Modules are created when they have content, not to mirror the target tree (ADR-0007).
`docs/MODULE-MAP.md` lists every planned module and its status.

## What the build enforces

`mvn verify` fails on any of these — none of them is a review comment:

| Check | Mechanism |
|---|---|
| Compiler warnings | `-Xlint:all` + `failOnWarning` |
| Unpinned plugin versions | enforcer `requirePluginVersions` |
| Legacy artifacts on the classpath (`net.lastcraft:*`) | enforcer `bannedDependencies` |
| EOL / superseded artifacts (MySQL 5.x, log4j 1.x, JUnit 4, BungeeCord, Nashorn) | enforcer `bannedDependencies` |
| Legacy imports | ArchUnit `NO_LEGACY_DEPENDENCIES` |
| NMS, Bukkit, Velocity, ProtocolLib, FAWE in domain code | ArchUnit, one rule each |
| JDBC or Redis in domain code | ArchUnit |
| `platform-api` depending on anything but the JDK | ArchUnit |
| Game platform depending on a concrete game | ArchUnit |
| Module cycles | ArchUnit slices |
| Infrastructure depending on domain implementations | ArchUnit |
| `public static` mutable fields | ArchUnit |

The rules themselves are tested: `ArchitectureRulesFailOnViolationTest` points them at fixtures that
deliberately break each one and asserts they fail.

## Adding a module

1. Create the module under the right aggregator and add it to that aggregator's `<modules>`.
2. Declare its coordinates in the root `dependencyManagement` (never a version in a child POM).
3. Add it to `tooling/architecture-tests/pom.xml` — it is then governed automatically, with no rule
   change.
4. Update `docs/MODULE-MAP.md`.

## Conventions

- **Comments and Javadoc are Russian. Identifiers, package names and API names are English.**
- Value objects are Java records with validation in the compact constructor: a value that exists is
  valid.
- Lombok is available but `@Data` and `@Cleanup` are build errors (ADR-0007). Records cover value
  objects; Lombok is for the mutable services of later phases.
- Every abstraction must answer CLAUDE.md §2.1. A module that would hold one class and have one
  consumer is not a module.

## IDE

IntelliJ occasionally reports `Unresolved plugin` for a plugin that Maven resolves fine from the CLI —
its own cache is stale. **Reload All Maven Projects** (the refresh button in the Maven tool window)
fixes it. Point the IDE at the committed wrapper so it builds with the same Maven the CLI uses.
