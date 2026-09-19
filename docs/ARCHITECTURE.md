# Westorix Platform — Architecture

## 1. Vision

Westorix is the rebuild of the legacy LastCraft system as a modular distributed Minecraft platform, rather than as a collection of independent plugins.

The architecture must support:

* multiple proxies
* multiple backend servers
* dynamic server pools
* multiple game modes
* matchmaking
* persistent player data
* distributed communication
* scalable infrastructure
* automated recovery
* high-performance gameplay
* rapid game development

The system should behave like a platform, not like a large plugin.

---

# 2. Architectural Model

High-level:

```text
                    ┌─────────────────────┐
                    │      Players        │
                    └──────────┬──────────┘
                               │
                               ▼
                    ┌─────────────────────┐
                    │      Velocity       │
                    │                     │
                    │ Routing             │
                    │ Matchmaking         │
                    │ Queue               │
                    │ Server Pools        │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
              ▼                ▼                ▼
        ┌──────────┐     ┌──────────┐     ┌──────────┐
        │ Purpur   │     │ Purpur   │     │ Purpur   │
        │ Server   │     │ Server   │     │ Server   │
        └────┬─────┘     └────┬─────┘     └────┬─────┘
             │                │                │
             └────────────────┼────────────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Game Platform  │
                     │                 │
                     │ Game Engine     │
                     │ Runtime         │
                     │ Arena           │
                     │ Teams           │
                     │ Voting          │
                     │ Rewards         │
                     └────────┬────────┘
                              │
              ┌───────────────┼────────────────┐
              ▼               ▼                ▼
        ┌───────────┐   ┌───────────┐   ┌───────────┐
        │ BuildBattle│   │ SkyBlock  │   │ ColorControl│
        └───────────┘   └───────────┘   └───────────┘
```

---

# 3. Core Layers

## Platform

Contains stable platform-level concepts:

```text
Player
Permission
Economy
Statistics
Cosmetics
Friends
Party
Configuration
```

Platform code should be independent from concrete infrastructure implementations.

---

## Infrastructure

Provides external persistence and communication:

```text
PostgreSQL
HikariCP
Redis
Caffeine
Messaging
Migrations
```

Infrastructure implements ports defined by higher layers.

---

## Proxy

Responsible for network-level orchestration:

```text
Velocity
Routing
Queues
Matchmaking
Server Pools
Server Registry
Capacity
Recovery
```

---

## Server

Responsible for Minecraft server integration:

```text
Purpur
Bukkit
World
GUI
Items
Scheduler
Packets
```

Minecraft-specific APIs remain isolated here.

---

## Game Platform

Contains reusable game infrastructure:

```text
Game API
Game Engine
Game Runtime
Lifecycle
Arena
Matchmaking
Teams
Voting
Rewards
```

---

## Games

Contains actual game implementations:

```text
BuildBattle
SkyBlock
ColorControl
SurvivalGames
Survival
KitPvP
Parkour
Arcade
```

Games depend on the Game Platform.

The Game Platform must NOT depend on individual games.

---

# 4. Game Engine

The central abstraction:

```text
GameDefinition
       │
       ▼
GameRuntime
       │
       ▼
GameInstance
       │
       ▼
GameState
       │
       ▼
GameSystems
```

## GameDefinition

Static definition of a game.

Contains:

* identifier
* version
* configuration
* supported player count
* rules
* factories
* systems

---

## GameRuntime

Controls runtime execution.

Responsible for:

* instance management
* lifecycle
* ticking
* system registration
* cleanup
* state transitions

---

## GameInstance

Represents one actual running game.

Example:

```text
buildbattle-102
skyblock-17
colorcontrol-04
```

---

## GameState

Represents current game state.

States:

```text
CREATED
INITIALIZING
WAITING
COUNTDOWN
STARTING
RUNNING
ENDING
REWARDS
RESETTING
FINISHED
```

---

# 5. Game Systems

Shared mechanics should be implemented as independent systems.

Examples:

```text
CountdownSystem
TimerSystem
TeamSystem
VotingSystem
RewardSystem
StatisticsSystem
ArenaSystem
PlayerSystem
BoundarySystem
ScoreSystem
CleanupSystem
```

A game composes systems instead of inheriting from a giant base class.

Example:

```text
BuildBattle
 ├── GameLifecycleSystem
 ├── CountdownSystem
 ├── PlotSystem
 ├── ThemeSystem
 ├── VotingSystem
 ├── ScoreSystem
 ├── RewardSystem
 └── CleanupSystem
```

---

# 6. Server Lifecycle

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

Failure:

```text
UNHEALTHY
      ↓
RECOVERING
      ↓
READY
```

Components:

```text
ServerRegistry
HealthMonitor
ServerLifecycle
RecoveryPolicy
CapacityController
ServerPool
```

---

# 7. Matchmaking

Matchmaking sits above individual games.

```text
Player
  ↓
Queue
  ↓
MatchmakingStrategy
  ↓
Match
  ↓
ServerSelection
  ↓
Server
```

Strategies:

```text
FIFO
MMR
Skill
Party
Latency
Region
Load
Priority
```

The system should allow multiple strategies to be composed.

---

# 8. Player Model

Avoid a monolithic player object.

```text
PlayerIdentity
        │
        ├── PlayerProfile
        │
        ├── PlayerSession
        │
        ├── PersistentPlayerState
        │
        ├── RuntimePlayerState
        │
        └── TemporaryMatchState
```

This prevents unrelated concerns from becoming coupled.

---

# 9. Persistence Architecture

```text
Domain
  ↓
Repository Port
  ↓
Repository Implementation
  ↓
HikariCP
  ↓
PostgreSQL
```

Persistent data flow:

```text
Caffeine
   ↓ miss
Redis
   ↓ miss
PostgreSQL
```

PostgreSQL remains the source of truth.

---

# 10. Cache Architecture

```text
             ┌──────────────┐
             │   Request    │
             └──────┬───────┘
                    ▼
             ┌──────────────┐
             │ Caffeine L1  │
             └──────┬───────┘
                    │ miss
                    ▼
             ┌──────────────┐
             │   Redis L2   │
             └──────┬───────┘
                    │ miss
                    ▼
             ┌──────────────┐
             │ PostgreSQL   │
             └──────────────┘
```

Protection mechanisms:

```text
Single Flight
Request Coalescing
TTL Jitter
Negative Cache
Stampede Protection
Invalidation
```

Only use mechanisms where they solve actual load problems.

---

# 11. Messaging

Local:

```text
Typed EventBus
```

Distributed:

```text
Domain Event
     ↓
Message Envelope
     ↓
Serialization
     ↓
Redis Transport
     ↓
Consumer
```

Messages should contain:

```text
messageType
version
eventId
timestamp
source
payload
```

Consumers must be resilient to duplicate delivery where applicable.

---

# 12. Automation Architecture

Automation primitives:

```text
Trigger
Condition
Action
Policy
Workflow
Task
Execution
```

Example:

```text
Server becomes UNHEALTHY
          ↓
Trigger
          ↓
RecoveryPolicy
          ↓
Condition
          ↓
Restart Server
          ↓
Health Check
          ↓
READY
```

Automation must have:

* logging
* metrics
* timeouts
* failure handling
* execution IDs
* traceability

---

# 13. World Architecture

World operations are isolated:

```text
Game
 ↓
World Port
 ↓
FAWE Adapter
 ↓
FAWE
```

Supported operations may include:

* create world
* clone template
* paste schematic
* reset arena
* region manipulation
* bulk changes
* cleanup

Games should not contain FAWE implementation details.

---

# 14. Packet Architecture

```text
Game / Feature
      ↓
Packet Port
      ↓
Protocol Adapter
      ↓
ProtocolLib
```

NMS-specific implementation must remain behind adapters.

---

# 15. DSL Architecture

DSLs are optional tools for reducing repetitive development.

Possible DSLs:

```text
Game
Arena
Reward
Progression
Command
Permission
Cosmetic
Server Pool
Matchmaking
Event
Item
GUI
Configuration
World Template
```

Example conceptual game DSL:

```text
game("buildbattle") {
    players(2..16)

    lifecycle {
        waiting()
        countdown(30.seconds)
        running()
        ending()
        reset()
    }

    systems {
        plots()
        themes()
        voting()
        scoring()
        rewards()
    }
}
```

The DSL should compile into ordinary domain objects.

---

# 16. Module Dependency Model

Preferred:

```text
                    platform-api
                         ▲
                         │
              ┌──────────┴──────────┐
              │                     │
       game-platform            infrastructure
              ▲
              │
            games
              ▲
              │
       server adapters
```

Concrete dependencies must never leak downward into stable domain layers.

---

# 17. Threading Model

Minecraft main thread:

```text
Gameplay
Bukkit state
Player interaction
World interaction where required
```

Async workers:

```text
Database
Redis
HTTP
Serialization
Heavy computation
Analytics
Non-blocking world preparation
```

Never block the main thread.

Centralize executors.

Every executor must define:

* purpose
* ownership
* queue size
* rejection policy
* shutdown behavior
* metrics

---

# 18. Reliability

Use where justified:

```text
Timeout
Retry
Backoff
Jitter
Circuit Breaker
Bulkhead
Rate Limit
Backpressure
```

Never blindly retry:

* non-idempotent economy operations
* transactions without idempotency
* destructive operations

---

# 19. Observability

Every major subsystem should expose:

```text
Metrics
Logs
Health
Errors
Latency
Throughput
Failures
Queue depth
Pool utilization
```

Critical operations should be traceable.

Examples:

```text
player.load
matchmaking.create
server.allocate
game.start
reward.grant
world.reset
```

---

# 20. Security

Treat all external input as untrusted.

Validate:

* commands
* packets
* configuration
* database values
* Redis messages
* player-provided data

Never trust client-side game state.

Permissions must be centralized.

---

# 21. Testing Architecture

Testing layers:

```text
Unit
 ↓
Component
 ↓
Integration
 ↓
Architecture
 ↓
Performance
 ↓
Production verification
```

Architecture tests should verify:

* dependency rules
* no legacy imports
* no NMS leakage
* no forbidden Bukkit dependencies
* no cycles
* module boundaries

---

# 22. Migration Architecture

Legacy is not imported into modern modules.

Instead:

```text
legacy/source
      ↓
Audit
      ↓
MIGRATION-MAP.md
      ↓
Behavior extraction
      ↓
Modern domain
      ↓
Adapters / migration tools
      ↓
Modern platform
```

Legacy remains available for behavioral comparison until migration is verified.

---

# 23. Target Project Structure

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

# 24. Architectural North Star

The final platform should make this possible:

```text
Create New Game
       ↓
Define GameDefinition
       ↓
Compose Existing Systems
       ↓
Implement Only Unique Mechanics
       ↓
Add Configuration / DSL
       ↓
Tests
       ↓
Deploy
```

A new game should NOT require rebuilding:

* player handling
* lifecycle
* teams
* rewards
* matchmaking
* statistics
* persistence
* server allocation
* arena management
* configuration
* observability
* cleanup

Those capabilities belong to the platform.

---

# 25. Final Rule

The architecture exists to make development cheaper, safer and faster.

Not to make diagrams impressive.

The most successful architecture is the one where:

```text
more games
    ↓
less duplicated code
    ↓
less development time
    ↓
more reliability
    ↓
better performance
    ↓
faster iteration
```

Every architectural decision must move the platform in that direction.
