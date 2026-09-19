# Westorix Platform

> Modern, scalable and production-grade Minecraft platform built from the ground up.

LastCraft Platform is a complete modernization of the legacy LastCraft Minecraft network.

The goal is not to blindly rewrite the old codebase.

The goal is to understand the existing behavior, remove obsolete architecture, build a clean and scalable platform foundation, and make future development significantly faster.

---

## 🚀 Vision

LastCraft is being transformed from a collection of legacy Minecraft plugins and modules into a unified platform.

The platform is designed around:

* Java 25
* Purpur 1.21.8+
* Velocity 4.3.x
* PostgreSQL
* HikariCP
* Redis
* Caffeine
* FastAsyncWorldEdit
* ProtocolLib
* Maven
* JUnit 5
* JFR
* async-profiler
* JMH
* Sentry where useful

---

# 🧠 Core Philosophy

## Platform First

Common infrastructure should be implemented once and reused everywhere.

The platform should provide:

```text
Players
Permissions
Economy
Statistics
Cosmetics
Friends
Parties
Persistence
Caching
Messaging
Matchmaking
Server Pools
Game Lifecycle
Rewards
World Management
Observability
Automation
```

Games should focus on their unique gameplay.

---

## No Architecture for Architecture's Sake

The project must not become an abstract framework nobody wants to maintain.

Every:

* interface
* abstraction
* service
* event
* registry
* factory
* manager
* layer

must solve a real problem.

Prefer simple and explicit solutions when they are sufficient.

---

# 🎮 Game Engine

The Game Engine is one of the most important parts of LastCraft.

Its purpose is to extract genuinely shared game mechanics.

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

Shared systems may include:

```text
Countdown
Timer
Teams
Voting
Rewards
Statistics
Arena
Players
Score
Cleanup
Boundary
```

Game-specific mechanics remain inside the game module.

### Goal

After the Game Engine is complete:

> Every new game mode should become significantly cheaper, faster and safer to develop.

A new game should reuse the platform instead of rebuilding the same infrastructure.

---

# 🏗️ Architecture

High-level architecture:

```text
                    PLAYERS
                       │
                       ▼
                  ┌─────────┐
                  │ Velocity│
                  └────┬────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      ┌────────┐   ┌────────┐   ┌────────┐
      │ Purpur │   │ Purpur │   │ Purpur │
      │ Server │   │ Server │   │ Server │
      └────┬───┘   └────┬───┘   └────┬───┘
           │            │            │
           └────────────┼────────────┘
                        ▼
                ┌──────────────┐
                │ Game Platform│
                └──────┬───────┘
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
  BuildBattle       SkyBlock       ColorControl
```

Detailed architecture:

```text
docs/ARCHITECTURE.md
```

---

# 📦 Project Structure

```text
lastcraft-platform/

├── platform/
│   ├── platform-api/
│   ├── platform-core/
│   ├── platform-config/
│   ├── platform-command/
│   ├── platform-player/
│   ├── platform-permissions/
│   ├── platform-economy/
│   ├── platform-statistics/
│   ├── platform-cosmetics/
│   ├── platform-friends/
│   └── platform-party/
│
├── infrastructure/
│   ├── database/
│   │   ├── database-api/
│   │   ├── database-hikari/
│   │   ├── database-postgres/
│   │   └── database-migration/
│   │
│   ├── cache/
│   │   ├── cache-api/
│   │   ├── cache-caffeine/
│   │   └── cache-redis/
│   │
│   └── messaging/
│       ├── messaging-api/
│       └── messaging-redis/
│
├── proxy/
│   ├── velocity-api/
│   ├── velocity-core/
│   ├── velocity-routing/
│   ├── velocity-server-pool/
│   ├── velocity-matchmaking/
│   └── velocity-queue/
│
├── server/
│   ├── purpur-core/
│   ├── purpur-adapter/
│   ├── world/
│   ├── gui/
│   ├── items/
│   ├── scheduler/
│   └── packets/
│
├── game-platform/
│   ├── game-api/
│   ├── game-engine/
│   ├── game-runtime/
│   ├── game-lifecycle/
│   ├── game-arena/
│   ├── game-matchmaking/
│   ├── game-teams/
│   ├── game-voting/
│   └── game-rewards/
│
├── games/
│   ├── buildbattle/
│   ├── skyblock/
│   ├── colorcontrol/
│   ├── survivalgames/
│   ├── survival/
│   ├── kitpvp/
│   ├── parkour/
│   └── arcade/
│
├── modules/
│   ├── anticheat/
│   ├── regions/
│   ├── protection/
│   ├── market/
│   ├── creative/
│   └── drops/
│
├── integrations/
│   ├── fawe/
│   ├── protocollib/
│   ├── discord/
│   └── sentry/
│
├── dsl/
│   ├── platform-dsl/
│   ├── game-dsl/
│   ├── server-dsl/
│   └── config-dsl/
│
├── tooling/
│   ├── testkit/
│   ├── benchmarks/
│   └── dev-tools/
│
├── docs/
│   ├── ARCHITECTURE.md
│   ├── MODULE-MAP.md
│   ├── MIGRATION-MAP.md
│   ├── DEVELOPMENT.md
│   └── ADR/
│
└── legacy/
    └── source/
```

---

# 🗄️ Data Architecture

PostgreSQL is the source of truth.

```text
Application
     │
     ▼
Repository
     │
     ▼
HikariCP
     │
     ▼
PostgreSQL
```

Cache hierarchy:

```text
Caffeine L1
     ↓
Redis L2
     ↓
PostgreSQL L3
```

The system may use:

* request coalescing
* single-flight loading
* TTL jitter
* negative caching
* cache invalidation
* optimistic locking
* transactions

Only where these mechanisms solve real problems.

---

# 🌐 Distributed Architecture

Redis can provide:

* distributed cache
* messaging
* pub/sub
* server registry
* matchmaking coordination
* temporary state
* distributed coordination where justified

Distributed messages are versioned.

PostgreSQL remains the persistent source of truth.

---

# 🖥️ Server Infrastructure

Server lifecycle:

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

Failure recovery:

```text
UNHEALTHY
     ↓
RECOVERING
     ↓
READY
```

Core components:

```text
ServerRegistry
HealthMonitor
ServerLifecycle
RecoveryPolicy
CapacityController
ServerPool
```

The platform should be capable of automatically detecting unhealthy servers and recovering them where safe.

---

# 🔎 Legacy Migration

Legacy source:

```text
legacy/source/
```

is READ ONLY.

It exists for:

* behavior discovery
* compatibility analysis
* dependency analysis
* database discovery
* command discovery
* gameplay analysis
* identifying hidden assumptions

It is NOT the target architecture.

Migration flow:

```text
Legacy
   ↓
Audit
   ↓
Behavior Analysis
   ↓
Migration Map
   ↓
Modern Design
   ↓
Implementation
   ↓
Tests
   ↓
Verification
```

The main migration document:

```text
docs/MIGRATION-MAP.md
```

---

# 🧭 Migration Order

General migration strategy:

```text
1. Legacy Audit
2. Platform Foundation
3. Database / Cache / Messaging
4. Player / Permissions / Economy
5. Game Engine
6. Game Runtime
7. Velocity
8. Purpur
9. Games
10. AntiCheat / Protocol
11. World / FAWE
12. Automation
13. Production Hardening
```

The actual order may change if dependency analysis indicates a safer sequence.

---

# 🎯 Games

Planned game ecosystem:

### BuildBattle

Planned systems:

* plots
* themes
* voting
* scoring
* levels
* barriers
* upgrades
* cosmetics
* rewards
* statistics
* leaderboards
* FAWE arena reset
* ProtocolLib integration

### SkyBlock

Planned systems:

* island progression
* island cosmetics
* leaderboard
* runes
* rune abilities
* custom monsters
* dynamic weather
* automation

Dynamic weather:

```text
Acid Rain
Frost
Solar Flare
Sky Storm
Eclipse
Meteor Shower
Mana Rain
```

### Other Games

```text
ColorControl
SurvivalGames
Survival
KitPvP
Parkour
Arcade
```

---

# ⚙️ Automation

The platform supports reusable automation concepts:

```text
Trigger
Condition
Action
Policy
Workflow
Task
Execution
```

Automation is intended for:

* server recovery
* matchmaking
* player lifecycle
* maintenance
* cleanup
* rewards
* scheduled operations
* infrastructure management

Automation must remain observable and debuggable.

---

# 🛡️ Reliability

The platform should use appropriate:

```text
Timeouts
Retries
Exponential Backoff
Jitter
Circuit Breakers
Bulkheads
Rate Limits
Backpressure
```

Do not blindly retry non-idempotent operations.

Economy and rewards must be concurrency-safe and idempotent.

---

# ⚡ Performance

Performance is a first-class requirement.

Avoid:

* blocking main thread
* blocking database calls
* blocking Redis calls
* unbounded queues
* uncontrolled thread creation
* excessive allocations
* repeated database queries
* unnecessary serialization
* unnecessary world scans

Tools:

```text
JFR
async-profiler
JMH
```

Performance decisions should be based on measurements whenever practical.

---

# 🧪 Testing

Testing strategy:

```text
Unit Tests
     ↓
Component Tests
     ↓
Integration Tests
     ↓
Architecture Tests
     ↓
Performance Tests
```

Critical systems require tests.

Especially:

* persistence
* economy
* rewards
* player state
* matchmaking
* game lifecycle
* distributed messaging
* cache invalidation
* server lifecycle

---

# 🤖 Claude Code

Claude Code is treated as the primary engineering agent for this project.

Read:

```text
CLAUDE.md
```

before making architectural or implementation decisions.

Claude should:

```text
Search
  ↓
Understand
  ↓
Design
  ↓
Implement
  ↓
Compile
  ↓
Test
  ↓
Fix
  ↓
Validate
  ↓
Continue
```

Do not repeatedly ask for approval for normal engineering decisions.

Do not blindly implement based on assumptions.

Do not rewrite legacy code line-by-line.

---

# 📚 Documentation

Important documents:

| Document                | Purpose                              |
| ----------------------- | ------------------------------------ |
| `CLAUDE.md`             | Claude Code engineering contract     |
| `docs/ARCHITECTURE.md`  | Target architecture                  |
| `docs/MIGRATION-MAP.md` | Legacy → modern migration            |
| `docs/MODULE-MAP.md`    | Module responsibilities/dependencies |
| `docs/DEVELOPMENT.md`   | Development workflow                 |
| `docs/ADR/`             | Important architectural decisions    |

---

# 🏁 Definition of Done

A feature is complete only when it is:

* implemented
* compiled
* tested
* integrated
* validated
* observable where necessary
* concurrency-safe where necessary
* performant enough for its workload
* documented where necessary
* compatible with architectural boundaries

Code existing in the repository does not automatically mean the feature is finished.

---

# 🔥 Final Goal

LastCraft should evolve into a platform where:

```text
Build Platform
      ↓
Build Game Engine
      ↓
Build Shared Systems
      ↓
Build New Game
      ↓
Reuse Everything Possible
      ↓
Implement Only Unique Gameplay
```

The result should be:

```text
Less duplicated code
        ↓
Less development time
        ↓
Faster new game development
        ↓
Better reliability
        ↓
Better performance
        ↓
Easier maintenance
        ↓
Scalable LastCraft Platform
```

**Build the platform once.
Build the Game Engine correctly.
Make everything after it dramatically easier.**
