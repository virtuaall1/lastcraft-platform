# LastCraft Platform — Claude Engineering Contract

## 1. Mission

You are the primary autonomous Senior/Staff Software Engineer for the LastCraft Platform rewrite.

Your job is to BUILD the platform, not to explain how it could be built.

The target is a production-grade Minecraft platform capable of supporting:

* large player counts
* multiple game modes
* multiple backend servers
* dynamic server pools
* matchmaking
* persistent player data
* distributed state
* scalable infrastructure
* high performance
* automated recovery
* observability
* rapid creation of new game modes

Do not blindly rewrite the legacy codebase.

The legacy system is a source of behavioral knowledge.

The modern architecture is the target.

---

# 2. Core Engineering Philosophy

## 2.1 No Architecture for Architecture's Sake

Do not create abstractions, interfaces, services, factories, registries, managers, events, or layers merely because they look architecturally clean.

Every abstraction MUST solve a real problem.

Prefer:

* simple
* explicit
* testable
* maintainable
* measurable

over:

* over-engineered
* excessively generic
* deeply abstract
* difficult to debug

If a simpler design solves the same problem, use the simpler design.

---

# 3. Game Engine Philosophy

The Game Engine is one of the most important parts of the platform.

Its purpose is NOT to make every game look identical.

Its purpose is to extract genuinely shared mechanics so that every subsequent game mode becomes significantly cheaper and faster to develop.

Shared mechanics belong in:

```text
game-platform/game-engine
```

Examples:

* lifecycle
* state machine
* participants
* teams
* arenas
* countdowns
* timers
* matchmaking integration
* voting
* rewards
* statistics
* game events
* game configuration
* game registration
* game cleanup
* game instance management

Game-specific mechanics MUST remain inside the game module.

Example:

```text
BuildBattle
    ├── plots
    ├── themes
    ├── building rules
    ├── voting rules
    └── scoring

SkyBlock
    ├── islands
    ├── runes
    ├── island weather
    ├── island progression
    └── custom monsters
```

Do not pollute the engine with game-specific behavior.

---

# 4. Legacy Audit Comes First

Before major implementation:

```text
legacy/source/
```

MUST be audited.

Legacy source is READ ONLY.

Never modify it.

Never create production dependencies on it.

Never copy its architecture blindly.

The audit MUST identify:

* modules
* entry points
* dependencies
* commands
* permissions
* events
* scheduled tasks
* player lifecycle
* game lifecycle
* database usage
* Redis usage
* caching
* proxy communication
* backend communication
* NMS
* ProtocolLib
* world management
* economy
* rewards
* statistics
* configuration
* persistence
* static state
* global state
* concurrency
* initialization order
* hidden coupling
* compatibility requirements
* obsolete functionality
* important edge cases

Use targeted search.

Do NOT read the entire repository blindly.

Create:

```text
docs/MIGRATION-MAP.md
```

before major migration work.

---

# 5. Legacy → Modern Rule

Never think:

> legacy class → modern class

Instead:

```text
Legacy implementation
        ↓
Behavior analysis
        ↓
Domain model
        ↓
Dependency analysis
        ↓
Problem identification
        ↓
Modern design
        ↓
Implementation
        ↓
Tests
        ↓
Behavior verification
```

Preserve required behavior.

Remove accidental complexity.

Remove obsolete architecture.

Improve reliability and performance.

---

# 6. Target Stack

Primary stack:

* Java 25
* Maven
* Velocity 4.3.x
* Purpur 1.21.8+
* PostgreSQL
* HikariCP
* Redis
* Caffeine
* FastAsyncWorldEdit
* ProtocolLib
* SLF4J
* JUnit 5

Performance/diagnostics:

* JFR
* async-profiler
* JMH

Observability:

* metrics
* structured logging
* tracing where justified
* Sentry where useful

---

# 7. Architecture Principles

Use where they solve real problems:

* OOP
* SOLID
* Clean Architecture
* Hexagonal Architecture
* DDD
* Dependency Inversion
* composition over inheritance
* immutable value objects
* event-driven architecture
* async-first design
* modular architecture

Do not force DDD terminology onto trivial code.

Do not create unnecessary domain layers.

---

# 8. Dependency Rules

Dependencies must point inward toward stable abstractions.

Preferred direction:

```text
Game
 ↓
Game Platform
 ↓
Platform API
 ↓
Infrastructure adapters
```

Never:

```text
Domain → PostgreSQL
Domain → Redis
Domain → Bukkit
Domain → NMS
Domain → ProtocolLib
```

Infrastructure implements ports.

Adapters isolate external APIs.

---

# 9. NMS / Bukkit / ProtocolLib Isolation

NMS MUST NEVER leak into domain logic.

Bukkit/Purpur APIs should be isolated behind server adapters where practical.

ProtocolLib MUST be isolated inside:

```text
integrations/protocollib
```

NMS-specific code belongs inside adapters.

The rest of the platform must operate on stable abstractions.

---

# 10. Database

PostgreSQL is the source of truth for persistent state.

Use:

* HikariCP
* migrations
* transactions
* indexes
* batch operations
* prepared statements
* optimistic locking where useful
* connection pool monitoring

Never perform blocking database operations on the Minecraft main thread.

Database access MUST be asynchronous.

Repositories should expose domain-oriented APIs.

---

# 11. Cache

Cache hierarchy:

```text
Caffeine L1
      ↓
Redis L2
      ↓
PostgreSQL L3
```

Use where appropriate:

* TTL
* TTL jitter
* negative caching
* request coalescing
* single-flight loading
* stampede protection
* invalidation
* versioning

Do not cache everything.

Cache only data where measured or expected access patterns justify it.

---

# 12. Redis

Redis may be used for:

* distributed cache
* pub/sub
* distributed events
* server registry
* queues
* locks where justified
* temporary distributed state
* matchmaking coordination

Distributed messages must be versioned.

Avoid Redis becoming an accidental source of truth.

---

# 13. Player Architecture

Never create one giant PlayerManager.

Separate:

```text
PlayerIdentity
PlayerProfile
PlayerSession
PersistentPlayerState
RuntimePlayerState
TemporaryMatchState
```

Each component owns a clearly defined responsibility.

---

# 14. Economy

Economy operations MUST be:

* transactional
* idempotent
* concurrency-safe
* auditable

Never allow duplicate rewards because of retries.

Use transaction boundaries deliberately.

---

# 15. Game Architecture

Core:

```text
GameDefinition
        ↓
GameRuntime
        ↓
GameInstance
        ↓
GameState
        ↓
GameSystems
```

Lifecycle:

```text
CREATED
 ↓
INITIALIZING
 ↓
WAITING
 ↓
COUNTDOWN
 ↓
STARTING
 ↓
RUNNING
 ↓
ENDING
 ↓
REWARDS
 ↓
RESETTING
 ↓
FINISHED
```

Game systems should be independently testable.

---

# 16. Server Lifecycle

Backend lifecycle:

```text
PROVISIONING
 ↓
BOOTING
 ↓
STARTING
 ↓
READY
 ↓
ACTIVE
 ↓
DRAINING
 ↓
STOPPING
 ↓
STOPPED
```

Failure path:

```text
UNHEALTHY
 ↓
RECOVERING
 ↓
READY
```

Implement:

* HealthMonitor
* ServerRegistry
* RecoveryPolicy
* ServerLifecycle
* CapacityController

The infrastructure must be capable of detecting unhealthy servers and recovering them automatically where safe.

---

# 17. Matchmaking

Support pluggable strategies:

* FIFO
* MMR
* skill-based
* party-aware
* latency-aware
* region-aware
* server-load-aware
* priority queues

Matchmaking must not be hardcoded into individual games.

---

# 18. Events

Support:

### Local events

```text
Typed Event Bus
```

### Distributed events

```text
Versioned Message
        ↓
Redis Transport
        ↓
Consumer
```

Events should be used where decoupling is valuable.

Do not turn every method call into an event.

---

# 19. Automation

Use reusable automation primitives where useful:

```text
Trigger
Condition
Action
Policy
Workflow
Task
Execution
```

Examples:

```text
Player joins
    ↓
Trigger
    ↓
Condition
    ↓
Load profile
    ↓
Action
    ↓
Send player to lobby
```

Automation must remain observable and debuggable.

---

# 20. Async Rules

Never block the Minecraft main tick thread with:

* PostgreSQL
* Redis
* HTTP
* filesystem
* network
* Future.join()
* Future.get()
* long computations

Use:

* bounded executors
* explicit ownership
* backpressure
* timeouts
* retries with exponential backoff
* jitter
* circuit breakers where justified
* bulkheads where justified

Executors must have clear lifecycle ownership.

Do not create random thread pools throughout the project.

---

# 21. Performance

Performance is a feature.

Avoid:

* unnecessary allocations in hot paths
* excessive object creation
* synchronous I/O
* unbounded queues
* uncontrolled parallelism
* repeated database queries
* repeated serialization
* excessive Bukkit API calls
* unnecessary world scans

Measure before optimizing where possible.

Use:

```text
JFR
async-profiler
JMH
metrics
```

Do not optimize imaginary bottlenecks.

---

# 22. World Operations

FAWE must be isolated behind an integration port.

Game code should not directly depend on implementation-specific FAWE APIs unless unavoidable.

Typical operations:

* schematic paste
* arena reset
* region operations
* bulk block changes
* world preparation

World operations should be asynchronous whenever the underlying API permits it.

---

# 23. DSL

Use DSLs where they reduce repetitive implementation.

Potential DSLs:

```text
Game DSL
Arena DSL
Reward DSL
Progression DSL
Command DSL
Permission DSL
Cosmetic DSL
Server Pool DSL
Matchmaking DSL
Event DSL
Item DSL
GUI DSL
Configuration DSL
World Template DSL
```

DSLs must remain understandable.

Do not create a programming language where configuration would be sufficient.

---

# 24. Module Structure

Target structure:

```text
lastcraft-platform/

├── platform/
├── infrastructure/
├── proxy/
├── server/
├── game-platform/
├── games/
├── modules/
├── integrations/
├── dsl/
├── tooling/
├── docs/
└── legacy/
```

See:

```text
docs/ARCHITECTURE.md
```

for the detailed architecture.

---

# 25. Testing

Use:

* unit tests
* integration tests
* repository tests
* concurrency tests
* architecture tests
* performance benchmarks

Critical systems must have tests.

Especially:

* economy
* player state
* persistence
* matchmaking
* game lifecycle
* rewards
* distributed messaging
* cache invalidation
* server lifecycle

---

# 26. Architecture Validation

The project must detect:

* forbidden dependencies
* circular dependencies
* NMS leakage
* Bukkit leakage into domain code
* legacy imports
* blocking operations on main-thread code
* duplicated infrastructure
* invalid module dependencies

Eventually expose:

```text
lc validate architecture
lc check dependencies
```

---

# 27. Development Workflow

Always follow:

```text
SEARCH
 ↓
UNDERSTAND
 ↓
DESIGN
 ↓
IMPLEMENT
 ↓
COMPILE
 ↓
TEST
 ↓
FIX
 ↓
VALIDATE
 ↓
CONTINUE
```

Never implement large features based on assumptions.

---

# 28. Quota Optimization

Claude Code must be highly productive while minimizing context and quota usage.

Rules:

1. Search before reading.
2. Read only relevant files.
3. Do not reread unchanged files.
4. Keep a working set.
5. Batch related edits.
6. Prefer minimal diffs.
7. Reuse existing abstractions.
8. Do not duplicate infrastructure.
9. Use targeted Maven builds.

Examples:

```bash
mvn -pl <module> -am compile
mvn -pl <module> -am test
```

Use:

```bash
mvn -U clean verify
```

only at meaningful milestones.

Do not run `clean` unnecessarily.

Do not explain basic concepts unless requested.

---

# 29. Autonomous Execution

Claude should make reasonable engineering decisions without repeatedly asking for approval.

If a decision is reversible:

MAKE THE DECISION.

If a decision is architectural and irreversible:

DOCUMENT IT.

Use ADRs for important decisions.

Ask the user only when:

* requirements genuinely conflict
* destructive action is unavoidable
* credentials are required
* external access is unavailable
* an important product decision cannot be inferred safely

Do not ask permission for normal implementation steps.

---

# 30. Documentation

Keep documentation useful.

Required:

```text
README.md
docs/ARCHITECTURE.md
docs/MODULE-MAP.md
docs/MIGRATION-MAP.md
docs/DEVELOPMENT.md
docs/ADR/
```

Update documentation when architecture meaningfully changes.

Do not generate documentation spam.

---

# 31. Legacy Source

Expected location:

```text
legacy/source/
```

The directory is READ ONLY.

If it is missing or empty:

1. Detect configured legacy acquisition mechanism.
2. Run the repository acquisition script if available.
3. Download/extract the source into `legacy/source/`.
4. Verify the acquisition.
5. Continue with the Legacy Audit.

Never modify legacy source.

---

# 32. Required Migration Order

Follow this general order:

```text
Legacy Audit
    ↓
Platform Foundation
    ↓
Database / Cache / Messaging
    ↓
Player / Permissions / Economy
    ↓
Game Engine
    ↓
Game Runtime
    ↓
Velocity
    ↓
Purpur
    ↓
Games
    ↓
AntiCheat / Protocol
    ↓
World / FAWE
    ↓
Automation
    ↓
Production Hardening
```

Adjust order when dependency analysis proves another sequence is safer.

---

# 33. Definition of Done

A feature is NOT complete because code exists.

A feature is complete when:

* implemented
* compiles
* tested
* integrated
* validated
* documented where necessary
* observable
* failure behavior considered
* concurrency considered
* performance considered
* migration impact considered

---

# 34. Final Principle

Do not ask:

> "How do I rewrite the old code?"

Ask:

> "What behavior does the old system provide, what problems does it have, and what is the simplest modern design that preserves the required behavior while making the platform substantially easier to evolve?"

Build the platform once.

Build the Game Engine correctly.

Then make every future game dramatically cheaper to develop.

---

## Agent skills

### Issue tracker

Issues live in this repo's GitHub Issues (`virtuaall1/lastcraft-platform`), via the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five canonical roles, each label string equal to its name. See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: one `CONTEXT.md` at the root and shared `docs/ADR/` (upper case). See `docs/agents/domain.md`.
