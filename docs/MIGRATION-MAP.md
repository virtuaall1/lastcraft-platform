# Westorix Migration Map (LastCraft → Westorix)

Status of this document: **Legacy Audit complete (2026-09-15). Populated from targeted analysis of `legacy/source/`.**

Legacy architecture is not the target architecture.
Legacy behavior is the source of behavioral knowledge.

---

## 1. Legacy Inventory

`legacy/source/` contains **45 repositories**, ~1.7M lines of Java. They are *not* one system — they are
three overlapping generations plus vendored third-party forks.

### 1.1 Classification

| Class | Repositories | Decision |
|---|---|---|
| **Current generation — authoritative** | `core-application-main`, `core-protocol-main`, `commons-master`, `bukkit-connector-master`, `bungee-api-master`, `core-bans-main`, `core-commands-main`, `core-friends-main`, `core-games-main`, `core-mail-main`, `core-party-main`, `core-reports-main`, `core-anticheat-main`, `server-stream-main`, `darta-api-master`, `game-api-master`, `lobby-api-master`, `game-effects-main`, `kitpvp-main`, `arcade-games-main`, `parkour-main`, `vampirez-main`, `anti-cheat-master`, `alert-ext-master` | **Audit source of truth** |
| **Previous generation — superseded snapshots** | `last-core-master` (49.5k LOC monorepo of the `core-*` modules), `last-games-master` (77k LOC monorepo of the game plugins) | `NOT_MIGRATING` as code. Retained **read-only** for behavioral archaeology where a split repo lost history. |
| **Separate product lines** | `plugins-survival-master` (74.8k — survival/skyblock/anarchy/creative/market/economy/auction), `lastcraft-site-backend-master`, `lastcraft-site-frontend-master`, `discord-bot-main`, `vk-bot-main` | Out of scope for phases 1–13. Re-audited before `modules/` and `integrations/` phases. |
| **Vendored third-party forks** | `paper-1-12-master`, `paper-1-16-master`, `paper-1-19-master`, `paper-1.20-ver-1.20.1` (1.17M LOC of Paper forks), `wproxy-master` (BungeeCord fork), `wcommons-master` (`io.github.whilein` utility library), `slimeworldmanager-master`, `builders-utilities-master` (`net.arcaniax`) | `NOT_MIGRATING`. Replaced by Purpur 26.2 (calendar versioning; ADR-0022) / Velocity 4.2.0 / first-party utilities. |
| **Empty** | `core-gift-main`, `core-guilds-main`, `core-velocity-main`, `lastcraft-bugs-main`, `lastcraft-localization-master` | No source. `core-velocity-main` being empty is itself a finding: **the Velocity migration was started and abandoned.** |

### 1.2 Runtime topology (as built)

```text
Minecraft client
      │
      ▼
wproxy (BungeeCord fork)  ──┐
  + bungee-api              │   custom Netty protocol (core-protocol)
  + bungee-connector        ├─────────────────────────► Core (core-application-main)
                            │                             standalone Java app, port N
Purpur/Paper 1.12.2|1.16.5  │                             modules/*.jar hot-loaded
  + darta-api (NMS)         │                             owns: auth, players, registry
  + bukkit-connector      ──┘
  + game-api / lobby-api
  + <game plugin>
      │
      └──────── direct MySQL (own HikariCP pool) ────────► MySQL
                                                            ▲
Core ───────────────────────────────────────────────────────┘
Core ── Jedis ──► Redis (cache only; no pub/sub fan-out)
```

**The central architectural fact:** LastCraft is *not* a plugin suite. It is a **custom distributed
system** with a bespoke first-party Netty protocol and a standalone authoritative Core service. The
modern platform must preserve that topology (it is correct) while replacing the transport, the
persistence access pattern, and the module boundaries.

---

## 2. Priorities & Statuses

| Priority | Meaning |
|---|---|
| P0 | Foundation / blocking |
| P1 | Critical platform |
| P2 | Important gameplay |
| P3 | Secondary |
| P4 | Optional / deprecated |

Statuses: `DISCOVERED` · `AUDITED` · `DESIGNED` · `IMPLEMENTING` · `IMPLEMENTED` · `TESTING` · `VERIFIED` · `DEPRECATED` · `NOT_MIGRATING`

---

## 3. Legacy → Modern Component Map

### 3.1 P0 — Foundation

---

#### `core-application-main` → Core service

| Field | Value |
|---|---|
| **Legacy Component** | `net.lastcraft.core.Core` (543 LOC god object) + `Bootstrap` + `CoreMainThread` + `ConnectionStorage` |
| **Responsibility** | Authoritative network brain: accepts Netty connections from every proxy and backend, owns the online-player registry, the server/proxy registry, auth (PBKDF2 + email), the module container, the task scheduler, the command framework, the Nashorn scripting engine, GeoIP, and the Redis pool. |
| **Dependencies** | `commons-master` (163 imports), `core-protocol-main` (66), `server-stream-main`, `wcommons` (eventbus/agent/geo), Netty 4.1.72, log4j 2.17, Jedis 4.0.1, Sentry, javax.mail, Nashorn |
| **Used By** | Every `core-*-main` module (as a hot-loaded `Module`), `vk-bot-main` (68 imports), `discord-bot-main` (19) |
| **Target Module** | `platform/platform-core` + `platform/platform-api` + `infrastructure/messaging` |
| **Priority** | **P0** |
| **Status** | `AUDITED` |
| **Risks** | Highest-risk component in the system. Single point of failure with **no clustering and no health model**. Rewriting it changes the wire contract with every proxy and backend simultaneously. |
| **Required Behavior** | Authoritative single source of online state; cross-server player lookup by id **and** name; server/proxy registry with liveness; module hot-load/unload; global broadcast; month rollover event; temp-donor-group expiry sweep. |
| **Migration Strategy** | Decompose, do not port. `Core` becomes: `PlayerSessionRegistry`, `ServerRegistry`, `ProxyRegistry`, `AuthenticationService`, `ModuleContainer`, `PlatformScheduler` — each independently constructible and testable. Keep the process model (standalone JVM); replace the singleton with constructor injection. Protocol compatibility handled by ADR-0006 (dual-stack bridge). |

**Defects found (must not be reproduced):**

- `Core.java:85` — `private static Core instance;` assigned as the **first statement of the constructor**
  (`instance = this`) before any field is initialized. Any code reachable during construction (module
  load, `KeysManager.registerBaseKeys()`, `Language.reloadAll()`) can observe a half-built `Core`.
  Unsafe publication across threads.
- `getOfflinePlayer(String)` falls through to `GlobalLoader.containsPlayerID(name)`, which is
  `loadPlayerId(name).join()` — a **blocking JDBC call reachable from Netty event-loop threads and from
  command dispatch**.
- `getOfflinePlayer(String)` linearly scans `offlinePlayerCache.asMap().values()` with an
  `equalsIgnoreCase` filter on every miss. O(cache size) per lookup.
- MaxMind license key `b8Q9ivieZzTQ3d5Q` **hardcoded in source**.
- Redis host/password from env vars, port hardcoded `6379`, no TLS, no pool tuning, no failure handling
  on construction.
- `shutdown()` calls `awaitTermination(Long.MAX_VALUE, NANOSECONDS)` on both event-loop groups, then
  `System.exit(1)` on a **clean** shutdown. Hangs forever on a stuck channel; always reports failure to
  the supervisor.
- `PacketProtocol.BUKKIT.getMapper().unregisterPacket(0x09, BukkitServerInfo.class)` with the comment
  `//todo временный костыль блять!!`. Global mutation of the protocol table at startup to work around a
  version skew.
- Nashorn `ScriptEngine` with `bindings.put("core", this)` and `BukkitScriptExecute` /
  `@ScriptExecutable` packets. **Arbitrary remote code execution by design**, reachable from the wire.
  Also pins the platform to a removed JDK feature (`org.openjdk.nashorn:nashorn-core:15.3`).
- `CoreMainThreadImpl` — unbounded `LinkedBlockingQueue`, `catch (Exception) { e.printStackTrace() }`,
  no backpressure, no task timing, no starvation detection.
- `ConnectionStorage` — `addServer()` stores `servers.put(bukkit.getName(), …)` (original case) but
  `getBukkit()` reads `servers.get(name.toLowerCase())`. **Lookup silently fails for any server whose
  name is not already lowercase.** Proxies use the lowercase path consistently; servers do not.
- `ConnectionStorage.getPlayers()` returns `Collections.unmodifiableMap(playersById)` — a **live view
  that escapes the `ReadWriteLock`**. `Core.alert()` streams over it; concurrent join/quit ⇒
  `ConcurrentModificationException`.
- `addServer`/`addProxy` use `checkState(!contains…)` — a duplicate server name **throws inside a Netty
  handler** instead of rejecting the handshake.

**Authentication defects** (`Credentials`, `PBKDF2Wrapper`; found while designing ADR-0021 — several are
live security holes, not untidiness):

- **Same IP within 24 hours logs you in with no password.** `shouldResumeSession(actualIp)` compares the
  latest IP-history entry against the current address and, if it matches and is under a day old, admits
  the player. Anyone behind the same NAT — a mobile carrier, a dormitory, an office — enters someone
  else's account. Under Russian CGNAT this is not theoretical. **The single largest security defect in
  the audit.**
- **The pepper is in the source repository**: `private static final String PEPPER = "Sc27C4HT2cDu6gQ";`
  — precisely the one place it must not be, since its only value is being absent from a database dump.
  It also cannot be replaced, because a new value invalidates every stored password. The authors knew:
  `// Сохранить это куда-то, если потеряем, то всем пизда`.
- Pepper is applied by **concatenation** (`password + PEPPER`), blurring the boundary between the two.
- **PBKDF2-HMAC-SHA256 with 50,000 iterations** — against a present-day recommendation of 600,000 for
  that same function, and PBKDF2 is the family GPUs handle best.
- **Hash parameters are constants in code**, so the cost could never be raised: any change makes every
  stored hash unverifiable. The database is frozen at its registration-day strength permanently.
- `comparePasswords` uses `String.equals` — comparison stops at the first differing character, so the
  response time leaks how much of the hash was guessed.
- **No attempt limiting at all**, by account or by address. Guessing through the login is unbounded,
  which matters far more than hash cost.
- The **security audit log is keyed by enum ordinal** (`type.ordinal()` / `ActionType.get(int)`).
  Inserting a constant in the middle rewrites the meaning of every historical row.
- `saveAction` catches `SQLException` and prints the stack: audit entries are silently lost.
- `onDuplicateKeyUpdateExcept("salt", …)` — the salt is never regenerated, so a password change keeps
  the old one.
- `Credentials` is model and DAO at once, executing SQL through the static `GlobalLoader.getDataSource()`.
- `EmailVerifyState` keeps verification codes in Redis with **no attempt counter**, so codes can be
  guessed without limit, and makes Redis the source of truth for them (CLAUDE.md §12 warns against this).

---

#### `core-protocol-main` → messaging

| Field | Value |
|---|---|
| **Legacy Component** | `net.lastcraft.core.io.*` — hand-rolled Netty codec, 77 files |
| **Responsibility** | Wire format between Core ↔ proxy ↔ backend. `DefinedPacket`, `PacketMapper` (id→class), zlib compression with a **native JNI implementation** (`NativeZlib`/`NativeCompressImpl`), `ServerInfo` field system, `AutoRegister` annotation processing. |
| **Dependencies** | Netty codec/handler, `commons-master` (10) |
| **Used By** | `core-application` (66), `bungee-api` (26), `bukkit-connector` (37), `server-stream` (25), `core-mail` (41), `core-games` (27), `core-commands` (20), `game-api` (5), `darta-api` (6) |
| **Target Module** | `infrastructure/messaging/messaging-api` + `messaging-redis`, with a compatibility codec in `platform/platform-core` |
| **Priority** | **P0** |
| **Status** | `AUDITED` |
| **Risks** | Changing it breaks every server at once. Native compression code is a JVM-crash surface. |
| **Required Behavior** | Ordered, compressed, typed request/response and fire-and-forget messages between Core, proxies and backends; server metadata publication (`ServerInfo`). |
| **Migration Strategy** | **Versioned envelope from day one.** New transport = Redis Streams/pub-sub for events + direct request/response where latency demands it. Keep a `legacy-codec` adapter behind `messaging-api` so old backends keep working during rollout (ADR-0006). Drop native zlib; use JDK `Deflater` and measure before reintroducing anything native. |

**Defects found:** packet identity is a bare byte id with **no protocol version negotiation** — hence
the `unregisterPacket(0x09, …)` hack in `Core`. `NativeCode`/`NativeCompressImpl` load a
platform-specific `.so` guarded by `me.catcoder.useNativeCompression=true` set in `Bootstrap`; a
mismatch is a segfault, not an exception. `UpdateLanguage.INSTANCE` is a **shared mutable packet
singleton**.

---

#### `commons-master` (`net.lastcraft.base`) → split across the whole platform

| Field | Value |
|---|---|
| **Legacy Component** | 257 files / 18.6k LOC "shared everything" library |
| **Responsibility** | JDBC access (`GlobalLoader`, `PlayerInfoLoader`, `MessengerFactory`), the player domain model (`GamerBase` + 11 `Section`s), the game-mode enums (`SubType`, `GameType`), ranks (`Group`), currencies (`PurchaseType`), the ASM-compiled command framework, DI (`InjectManager`), localization (`Language`), skins, gradients, nicknames, chat log, boosters, mystery keys, SlimeWorld DAOs. |
| **Dependencies** | `wcommons` (sql/util/unsafe/config/impl-loader), HikariCP, Caffeine, ASM, MySQL + Postgres + ClickHouse drivers, Jackson, Adventure |
| **Used By** | **Everything.** 449 internal + `lobby-api`(440), `darta-api`(374), `core-commands`(255), `game-api`(172), `core-application`(163), `kitpvp`(132), `vk-bot`(127) … |
| **Target Module** | **Deliberately dissolved.** `platform-player`, `platform-permissions`, `platform-economy`, `platform-command`, `platform-config`, `infrastructure/database/*`, `game-platform/game-api` |
| **Priority** | **P0** |
| **Status** | `AUDITED` |
| **Risks** | It is the hub of every coupling in the system. Dissolving it is the single highest-value and highest-effort action of the migration. |
| **Migration Strategy** | **Never recreate `commons`.** Each concern moves to its own module with its own ports. Nothing in the new platform may depend on a module named "commons", "base", "util" or "shared" that spans more than one bounded concern. Enforced by an architecture test. |

---

#### `commons/sql/GlobalLoader` → `infrastructure/database` + repositories

| Field | Value |
|---|---|
| **Legacy Component** | `@UtilityClass GlobalLoader` — 783 LOC, 51 public static methods |
| **Responsibility** | All global player persistence: id↔name resolution, money, groups, boosters, purchases, skins, settings, friends, join data, license state. |
| **Dependencies** | `w.sql.Messenger`, HikariCP, MySQL |
| **Used By** | 53 files in `last-core`, 24 in `commons`, plus every Bukkit plugin transitively through `Section`s |
| **Target Module** | `infrastructure/database/database-api` + `database-postgres` + per-aggregate repositories in `platform-player` / `platform-economy` |
| **Priority** | **P0** |
| **Status** | `AUDITED` |
| **Risks** | Every call site is a static import with no seam. Mechanical replacement is impossible; each of the 51 methods must be reassigned to an owning aggregate. |
| **Required Behavior** | Player id ↔ name resolution with caching and in-flight coalescing; the persisted column set (compatibility with the live MySQL schema during cutover). |
| **Migration Strategy** | Reassign the 51 methods to `PlayerIdentityRepository`, `PlayerProfileRepository`, `EconomyRepository`, `PurchaseRepository`, `SettingsRepository`. All async (`CompletableFuture`) on an owned, bounded executor. PostgreSQL + versioned migrations replace `TableConstructor`. |

**Defects found (each one is a rule for the new code):**

- **Static initializer creates a HikariCP pool and executes DDL** (`new TableConstructor("properties", …)
  .create(…)`) at class-load time. Class-loading a data object initiates network I/O and schema
  mutation. Failure ⇒ `ExceptionInInitializerError` with no recovery path.
- `ID_LOAD` is a plain `HashMap<String, CompletableFuture<Integer>>` used for request coalescing,
  guarded by `synchronized(ID_LOAD)` in two places but **read outside the monitor** in the completion
  path.
- `containsPlayerID(name)` = `loadPlayerId(name).join()`. Blocking. Called from Core command dispatch
  and from `Core.getOfflinePlayer`.
- Dialect is **MySQL**, not PostgreSQL. Driver `mysql-connector-java:5.1.42` (2017, EOL, known CVEs).
- Schema is created imperatively at runtime by whichever process starts first. **No migrations, no
  versioning, no ownership.**
- `MySqlDatabase.inBukkit()` — the persistence layer **sniffs its own runtime environment** to pick
  pool sizes.

---

#### Connection pooling topology — **P0 scaling wall**

Every Bukkit backend and every proxy links `commons` and therefore opens **its own direct MySQL pools**
(`GlobalPool` + `PlayerInfoPool`, ~10 connections each). At 50 backends that is ~1000 connections
against a single MySQL instance, all bypassing Core, all writing the same player rows concurrently with
no coordination.

**Target:** backends and proxies get **no database driver on the classpath at all**, enforced by an
architecture test. All persistence flows through Core over `messaging-api`. PostgreSQL behind one pool
per service, sized and monitored.

---

### 3.2 P1 — Critical Platform

---

#### `commons/gamer/*` → `platform/platform-player`

| Field | Value |
|---|---|
| **Legacy Component** | `GamerBase`, `IBaseGamer`, `OnlineGamer`, `OfflineGamer`, `Section` + 11 subclasses (`Money`, `Booster`, `Friends`, `Online`, `Networking`, `Properties`, `FakeName`, `Settings`, `Skin`, `Base`) |
| **Responsibility** | The player aggregate: identity, profile, currencies, social, settings, cosmetics, session. |
| **Dependencies** | `GlobalLoader` (static), `Language`, `Group`, `PurchaseType` |
| **Used By** | Core (`CorePlayer extends GamerBase`), Bungee (`BungeeGamer`), Bukkit (`BukkitGamer`) — three parallel subclasses of one base |
| **Target Module** | `platform/platform-player` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | The `Section` pattern is the one genuinely good idea in the legacy player model. Preserve the idea, fix the execution. |
| **Required Behavior** | Composable per-concern player state with independent load; identity by stable integer id; name is mutable and non-unique over time. |
| **Migration Strategy** | Keep composition, split by lifetime per CLAUDE.md §13: `PlayerIdentity` (immutable, id + uuid + name history) / `PlayerProfile` (persistent, loaded async before join completes) / `PlayerSession` (runtime, per-connection) / `TemporaryMatchState` (per-game, discarded). Sections become **explicitly loaded async components with declared dependencies**, not constructor side effects. |

**Defects found:** `CorePlayer`'s constructor calls `initSection(...)` ×7 then `load0()` — **synchronous
JDBC inside a constructor**, on the thread handling the login packet. `isGold()` then triggers a second
blocking load via `GradientManager.IMP`. Three platform-specific subclasses (`CorePlayer`,
`BungeeGamer`, `BukkitGamer`) each re-derive the same state from the same tables and reconcile by
sending each other packets.

---

#### `commons/gamer/constans/Group` → `platform/platform-permissions`

| Field | Value |
|---|---|
| **Legacy Component** | `enum Group` — 18 hardcoded ranks with embedded Russian display names and `§` colour codes |
| **Responsibility** | Authorization and chat prefix rendering. |
| **Dependencies** | `Language` |
| **Used By** | Platform-wide |
| **Target Module** | `platform/platform-permissions` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Changing the authorization model touches every command and GUI. Group ids are persisted. |
| **Required Behavior** | Rank ordering; donor-vs-staff distinction; temporary (expiring) ranks; rank-driven prefix/suffix rendering; reserved-slot privilege. |
| **Migration Strategy** | Replace level comparison with **named permission servers** resolved from a group graph with inheritance. Groups become data (PostgreSQL + cache), not an enum. Presentation (prefix, colour, localized name) moves out of the authorization model entirely. Keep the integer ids as a persisted compatibility key. |

**Defects found:** there is effectively **no permission system**. Authorization is
`gamer.getGroup().getLevel() >= Group.X.getLevel()` (39 sites) and `getGroup() == Group.X` (35 sites) —
**74 hardcoded authorization checks against an enum**. Granting one capability to one person requires a
code change and a redeploy. `PermissionManager.IMP.loadPermissions(lastperms.yml)` exists in `darta-api`
but is a **per-server YAML file**, unsynchronized across the network, loaded only if the file happens to
exist. Display strings (`"§2Хелпер ($)"`) are baked into the authorization enum.

---

#### `commons/gamer/sections/MoneySection` + `GlobalLoader.setMoney` → `platform/platform-economy`

| Field | Value |
|---|---|
| **Legacy Component** | `MoneySection`, `PurchaseType` (MYSTERY_DUST / GOLD / BOX_DUST), `BukkitBalancePacket` |
| **Responsibility** | Player currencies and purchases. |
| **Dependencies** | `GlobalLoader` (static) |
| **Used By** | Every game, every shop GUI, rewards, boosters, keys |
| **Target Module** | `platform/platform-economy` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | **Currently incorrect under concurrency. This is a live duplication/loss bug, not a design preference.** |
| **Required Behavior** | Three currencies; balance never negative; purchases atomic; balance visible on Core, proxy and backend. |
| **Migration Strategy** | See **ADR-0004**. Append-only `economy_transaction` ledger + materialized balance, single writer (Core), `UPDATE … SET balance = balance + ? WHERE balance + ? >= 0` atomic deltas, idempotency key per transaction, full audit trail. Backends never write money — they submit intents. |

**Defects found:**

- `changeMoney()` does read → compute → **absolute `setMoney(newValue)`**. Classic lost update. Two
  concurrent grants (e.g. a game reward on the backend and a purchase on Core) overwrite each other.
- `updateMoneyForCore(type, money, boolean update)` — a boolean-mode method that either **adds** or
  **sets** depending on a flag carried over the wire. **Non-idempotent: a redelivered packet duplicates
  currency.** The protocol has no dedup key.
- The `boolean mysql` parameter lets callers change the in-memory balance **without persisting**,
  producing a balance that is correct on one server and wrong everywhere else until the next reload.
- No transactions, no audit log, no reconciliation, no negative-balance enforcement on the DB side
  (only the in-memory `if (value + delta < 0) return false`).

---

#### `commons/game/SubType` + `GameType` → `game-platform/game-api` registry

| Field | Value |
|---|---|
| **Legacy Component** | `enum SubType` (17 modes), `enum GameType` (17 families), `enum ArcadeType` |
| **Responsibility** | Identify which game mode a server is running; key stats tables, shop items, localization, matchmaking channels, lobby routing. |
| **Dependencies** | `Language`, `GameSkin` |
| **Used By** | 87 files across 12 repositories |
| **Target Module** | `game-platform/game-api` (`GameDefinition` registry) |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Ids are persisted (stats table names, shop item scoping, DB columns). Cutover needs an id-stable registry. |
| **Required Behavior** | Stable mode identity; family→mode hierarchy; solo/doubles/team/ranked team sizing; lobby channel routing; localized names. |
| **Migration Strategy** | See **ADR-0003**. Replace both enums with a `GameDefinition` value object registered by each game module at startup. Identity becomes a namespaced key (`lastcraft:bedwars/solo`) with a stable numeric id retained purely for persistence compatibility. |

**Defects found — this is the single biggest obstacle to the "cheap new game mode" goal:**

- `public static SubType current = MISC;` and `public static GameType current = UNKNOWN;` — **public,
  non-volatile, mutable static fields** holding the identity of the running server. Written once in
  `DartaAPI.onLoad()` and read from 21 sites including GUI rendering and stats resolution. No
  happens-before guarantee between the writing plugin-load thread and reading tick/async threads.
- Adding one new game mode requires: editing `SubType` **in `commons`**, editing `GameType`, bumping the
  `commons` version, and **rebuilding and redeploying every module that links it** — Core, both proxies,
  and all 15 backend plugins. This is precisely the cost the new Game Engine must eliminate.
- `GameType.KIT_PVP(90, …)` and `GameType.VAMPIREZ(90, …)` have **duplicate id 90**. Any lookup keyed on
  `GameType.id` is ambiguous.
- `SubType` mixes identity, team sizing, localization keys, presentation and routing in one enum.
- `DartaAPI.registerType()` resolves server identity by `SubType.valueOf(serverType.toUpperCase())`,
  falls back to `System.getProperty("subType")`, then to scanning `GameType.values()` for a matching
  `lobbyChannel` string. **Server identity is derived from string parsing with three fallbacks and no
  validation.**

---

#### `commons/command/*` + `core/api/command/*` + `BukkitCommandManager` → `platform/platform-command`

| Field | Value |
|---|---|
| **Legacy Component** | Annotation-driven command framework that **compiles executors to bytecode with ASM** at runtime (`CommandExecutorCompiler`, `ArgumentTransformers`) |
| **Responsibility** | Command registration, argument binding, tab completion, conditions (`PlayerOnly`, `ConsoleOnly`, `WithGroup`, `MinArgs`, `NotVanishedOnly`), cooldowns, help. |
| **Dependencies** | ASM 9.2, `InjectManager`, `Group` |
| **Used By** | Core, Bungee, every Bukkit plugin — **three separate manager implementations** of the same annotations |
| **Target Module** | `platform/platform-command` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Command surface is large and user-facing; behaviour (aliases, cooldowns, localized errors) must be preserved exactly. |
| **Required Behavior** | The full annotation vocabulary (`@Command`, `@Subcommand`, `@Main`, `@Arg`, `@Attr`, `@Before`, `@Fallback`, `@WithAlias`, `@WithCooldown`, `@WithDescription`, `@WithUsage`, `@WithHelp`, `@HelpBehavior`) and the condition set; cross-server command sync (`BungeeSyncCommands`). |
| **Migration Strategy** | **Keep the annotation surface — it is genuinely good and heavily used.** Replace runtime ASM generation with `MethodHandles`/`LambdaMetafactory` (same performance, no bytecode generation, no `ByteBuddyAgent`, works under a module system). One implementation with platform adapters, not three. Conditions become permission-node based. |

---

#### `bukkit-connector-master` + `bungee-api-master` → `server/purpur-adapter` + `proxy/velocity-core`

| Field | Value |
|---|---|
| **Legacy Component** | `AbstractConnector`, `BukkitConnector`, `BungeeConnector`, `PacketHandler`, `SocketUtils` |
| **Responsibility** | Client side of the Core protocol: connect, reconnect, handshake, dispatch inbound packets as platform events. |
| **Dependencies** | `core-protocol`, `server-stream` |
| **Used By** | Every backend and proxy |
| **Target Module** | `server/purpur-adapter`, `proxy/velocity-core` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Reconnect semantics determine whether a server "disappears" from the network during a network blip. |
| **Required Behavior** | Automatic reconnect with backoff; buffering or explicit rejection while disconnected; state-change notification to plugins (`CoreStateChangeEvent`). |
| **Migration Strategy** | `proxy/velocity-core` is a **new implementation** (Velocity 4.2.0, not BungeeCord) — `core-velocity-main` is empty, so nothing to port. Backend side becomes a bounded, observable client with an explicit circuit breaker and backpressure. |

---

#### `wproxy-master` → `proxy/velocity-*`

| Field | Value |
|---|---|
| **Legacy Component** | Fork of BungeeCord (`net.md_5.bungee`), 22.3k LOC |
| **Responsibility** | The production proxy. |
| **Target Module** | `proxy/velocity-core`, `velocity-routing`, `velocity-server-pool`, `velocity-matchmaking`, `velocity-queue` |
| **Priority** | **P1** |
| **Status** | `NOT_MIGRATING` (code) / `AUDITED` (behavior) |
| **Risks** | A maintained fork of a dead proxy. Every upstream security fix must be hand-applied. Pins the platform to the 1.12/1.16 protocol era. |
| **Required Behavior** | Extract from the fork *what was patched and why* (compression, Russian usernames, chat rewriting, server switching). Those patches are the requirements list for the Velocity implementation. |
| **Migration Strategy** | Replace with stock Velocity 4.2.0 + first-party plugins. Audit the fork's diff against upstream BungeeCord as a separate task before `proxy/` work begins. |

---

### 3.3 P1/P2 — Game Platform

---

#### `game-api-master` → `game-platform/game-engine` + `game-runtime`

| Field | Value |
|---|---|
| **Legacy Component** | `net.lastcraft.gameapi` — 175 files / 15.7k LOC. Centre: `Game` (1154 LOC abstract god class), `GameSession` (457), `GameUser` (652), `TeamManager`, `GameUserRegistry`, `GameManagerRegistry`. |
| **Responsibility** | The shared minigame framework: lifecycle, teams, spectators, boards, kits/perks/shop items, loot tables, stats, rewards, GUIs, arena world generation, compass tracking, ~25 gameplay listeners. |
| **Dependencies** | `darta-public-api` (252), `commons` (172), `darta-api` (29), `darta-nms` (13), **`core-games` (3)**, **`core-party` (2)**, `box-api` (5), `core-protocol` (5), FAWE, WorldEdit 6.1.4 |
| **Used By** | `arcade-games` (237), `vampirez` (33), `kitpvp` (20), `parkour` (8), `game-effects` (3) |
| **Target Module** | `game-platform/game-engine` (shared mechanics), `game-runtime`, `game-lifecycle`, `game-arena`, `game-teams`, `game-rewards` |
| **Priority** | **P1** (the engine is the multiplier for every later game) |
| **Status** | `IMPLEMENTING` — lifecycle, systems, multi-instance runtime, definition registry, the lobby→start path (participants, spectators, countdown), team assignment with win conditions, arena allocation, reward settlement and voting are implemented and tested (ADR-0003, ADR-0007, ADR-0014 … ADR-0018). Boards are presentation and wait on the server adapter; terrain restoration waits on `server/world`. |
| **Risks** | Rewriting the engine while games still depend on the old one. Mitigated by migrating games one at a time onto the new engine rather than big-bang. |
| **Required Behavior** | Waiting-lobby countdown with player-count thresholds; team assignment incl. party-aware (`PartyTeamMaker`) and fixed (`FixedTeamMaker`); spectator mode with camera, teleport menu and glow; scoreboards per phase; per-mode shop items with purchase predicates (coins/group/level); loot tables from YAML **and** script; session stats → rewards (coins, keys, boosters); end-of-game top display; arena reset. |
| **Migration Strategy** | This is the **core deliverable of the rewrite**. Rebuild around `GameDefinition → GameRuntime → GameInstance → GameState → GameSystems` (ARCHITECTURE.md §4). Genuinely shared mechanics move into `game-engine`; everything mode-specific stays in the game module. Success criterion: **a new game mode is a single new module with no edits anywhere else.** |

**Defects found — these define what the new engine must *not* do:**

- **One `Game` per JVM.** `private static Game instance` (already annotated `@Deprecated //избавляемся от
  этого вообще` by the previous authors) plus `Preconditions.checkState(LastCraft.isGame())`. A backend
  can host exactly one match. Dynamic server pools therefore mean **one process per match** — the
  dominant infrastructure cost driver. The new engine must support **N concurrent `GameInstance`s per
  server**.
- `GameState` has **5 states** (`WAITING, STARTING, GAME, END, RESTART`) and `setState()` performs **no
  transition validation** — any state to any state, no guards, no entry/exit hooks. Target lifecycle is
  10 states with a validated transition table.
- `Game(Plugin)` + `loadSettings()` is a **constructor that generates worlds, registers ~10 listeners,
  schedules repeating tasks, builds scoreboards, registers commands, creates teams and mutates a global
  manager registry.** Untestable without a running Bukkit server.
- `gameSessionMap` is keyed by **lowercased player name**, not id. A nickname change mid-session
  detaches the player from their own match state.
- `GameManagerRegistry` is a static service locator; `StatsManager` is fetched via
  `(StatsManager) GameManagerRegistry.getManager(StatsManager.class)` — **unchecked cast on a global
  map** at field-initialization time.
- `listenerMap` is `Map<Class<? extends Listener>, Listener>` — **one listener instance per class,
  globally**, so per-instance game listeners are impossible by construction.
- Settings are untyped: `(int) getSetting(GameSettings.MAX_TEAMS)`, `getSetting(X, default)` returning
  `Object`. Every read is an unchecked cast.
- `ServerInfoKeeper` publishes server state on a **5-second async timer**. All matchmaking decisions are
  made against data that is **up to 5 seconds stale**.
- `WorldTime.freezeTime(...)` mutates **global static world-time state** keyed by world-name string.
- `game-api` imports `net.lastcraft.games.*` (`core-games`, a **Core-side** module) and
  `net.lastcraft.lobby.game.top.TopStandData` (`lobby-api`). **Layering inversion**: the game framework
  depends on the Core service module and on the lobby plugin.

**Reward defects** (found while designing ADR-0017; each is a live correctness problem, not just bad
structure):

- **A player who disconnects before the end is paid nothing.** `Game` settles by looping over
  `GameUserRegistry.getUsers()` and calling `gameSession.saveSessionRewards(...)` — i.e. over whoever is
  *still on the server*. Everything the leaver earned is silently discarded.
- **The amount shown is not the amount paid.** `DefaultReward.sendMessage` displays
  `coins × multiplier` computed **at the moment of earning**, while `saveSessionRewards` sums the
  **base** amounts and multiplies the total at the end. Any change of multiplier during the match
  (a booster expiring or activating) makes the two diverge, and even without one the rounding differs:
  `Σ⌊cᵢ·m⌋ ≠ ⌊Σcᵢ·m⌋`.
- **No idempotency whatsoever.** `saveSessionRewards` calls `gamer.changeMoney(...)` directly. Because
  `setState` validates nothing, re-entering the end state pays everyone a second time.
- **Rewards are written straight to the player object**, in the game thread, with no transaction and no
  ledger — so "where did these coins come from" has no answer (the problem ADR-0004 exists to fix).
- `getTotalRewards(Class)` matches with `getClass() == clazz`, so a mode that **subclasses**
  `DefaultReward` earns nothing, silently; the result is also an unchecked cast.
- `addSessionReward` resolves the player with `GameUserRegistry.getUser(cachedUser.getName())` — the
  nickname-key defect again.
- The `privateState` branch skips settlement entirely with no record that it did.

**Voting defects** (found while designing ADR-0018; voting existed only inside BuildBattle, never as a
shared mechanic):

- `VoteManager.findMostValuableTheme` scores candidates as `votes * 100f / getVotedPlayers().size()` —
  **the same denominator for every candidate**, so the comparator is vote count with extra arithmetic,
  and `getVotedPlayers()` rebuilds a `HashSet` over every theme's voters **inside the comparator**, on
  every comparison.
- **With no votes the denominator is zero.** Every score is `NaN`, `Double.compare(NaN, NaN) == 0`, and
  `max` returns the **first** element — the expected random pick never happened.
- **Ties always go to the first maximum in stream order**, so the first theme in configuration wins
  disproportionately often.
- If the winning theme's word list is empty, `GameUtil.resetGame(game)` **ends the match** — a
  configuration gap terminates play.

---

#### `core-games-main` → `game-platform/game-matchmaking` + `velocity-matchmaking` + `game-rewards`

| Field | Value |
|---|---|
| **Legacy Component** | `RedirectManager`, `QueueCommand`, `CoreMapSelector` / `SmartMapSelector`, `RewardManager` + `RewardMode`s, `DefaultStatsDao`, `GameMapManager` |
| **Responsibility** | Server selection / "matchmaking", map selection and voting, end-of-game rewards, monthly top resets. |
| **Dependencies** | `core-app` (44), `commons` (68), `core-protocol` (27), `core-party` (4), `core-mail` (6), `core-bans` (1) |
| **Used By** | `game-api` (inverted dependency), `lobby-api`, `vk-bot` |
| **Target Module** | `game-platform/game-matchmaking`, `proxy/velocity-matchmaking`, `game-platform/game-rewards` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Player-visible routing behaviour; donor slot reservation is a paid feature and must be preserved exactly. |
| **Required Behavior** | Route a player to a `WAITING` server of the requested type, optionally filtered by map; give `EMERALD`+ donors a way in when servers are full; enforce exactly-4 parties for ranked; tell the player when no server is available. **Correction (ADR-0019):** an earlier reading of this row said legacy routed a player *and their whole party atomically*. It does not — `redirectToBestServer` counts the party size against capacity and then redirects **only the requesting player**, holding nothing for the rest. Atomic party placement is a new requirement, not preserved behaviour. |
| **Migration Strategy** | Replace with a real matchmaking service (CLAUDE.md §17): pluggable strategies, **slot reservation with a TTL**, party-aware batch placement, a queue with position feedback, and a server registry fed by **heartbeats**, not by a 5-second info timer. See **ADR-0005**. |

**Defects found:**

- **The party is counted but not moved.** Capacity is checked against `player + party.size()`, then
  `player.redirect(availableServer)` moves **one** player. No seats are held for the others, so while
  they follow, anyone can take those seats — the very outcome the count was meant to prevent.
- **The donor privilege is overbooking, not a reserved slot.** With no room anywhere, an `EMERALD`+
  player is sent to `fullServer` — *the last full server the iteration happened to touch* — i.e. past
  capacity, onto an arbitrary server. ADR-0019 preserves the privilege as queue precedence plus
  optional configured headroom, and refuses to exceed capacity.
- `redirectToBestServer` is a **linear scan over every connected server on every redirect**, filtered by
  `bukkit.getName().startsWith(serverType)` — **server role is inferred from a name prefix string**.
- **No slot reservation.** `requestedSlots > slots` is a check-then-act against a counter that is up to
  5 seconds stale. Two concurrent redirects can both be admitted into the same last slot.
- Candidate comparison is `requestedSlots > availableServer.getPlayerCount()` — it compares the
  *candidate's* occupancy **including the incoming party** against the *incumbent's* occupancy
  **excluding it**. The comparison is asymmetric, so "best fit" selection is wrong whenever party size
  differs from 1.
- The ranked branch calls `sendMessageLocale` and `break`s **inside the scan loop**, so the outcome
  depends on connected-server iteration order.
- No queue. If no server is available the player is simply told "none available".
- Matchmaking policy is hardcoded in one method — CLAUDE.md §17 requires pluggable strategies.
- **`SmartMapSelector`'s map-rotation limit has never worked.** Its documented rule — no map may run on
  more than 30% of a mode's servers — is computed as `onlineServers / totalServers`, both `int`. Integer
  division makes the rate 0 or 1, so `rate >= MAP_REPEAT_RATE` (0.3f) excludes a map only once **every**
  server in the mode is already running it. Found during the arena audit (ADR-0016). Map selection is a
  network-wide concern and moves to matchmaking, not to the engine.

---

#### Statistics (`StatsManager`, `DefaultStatsDao`, `StatsMode`, `StatsField`) → `platform/platform-statistics`

| Field | Value |
|---|---|
| **Legacy Component** | Bukkit-side `StatsManager` (direct MySQL) + Core-side `DefaultStatsDao` |
| **Responsibility** | Per-mode player statistics, all-time and monthly, leaderboards, holographic top stands. |
| **Dependencies** | `GlobalLoader`/`MessengerFactory`, `SubType` |
| **Used By** | Every game, lobby holograms, stats GUIs, `/stats`, monthly reward jobs |
| **Target Module** | `platform/platform-statistics` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Player-visible historical data. Schema is discovered, not declared — the real schema exists only in production. |
| **Required Behavior** | Per-mode stat fields; all-time vs monthly (`type` 0/1); top-N leaderboards; monthly reset with reward distribution; win-rate and time formatters. |
| **Migration Strategy** | Declared schema with migrations. Stats become **write-through events from backends to Core**, never direct backend→DB writes. Leaderboards materialized and cached (Caffeine L1 / Redis L2), not computed by `ORDER BY … LIMIT` on every open. |

**Defects found:**

- **One MySQL table per game mode**, named after the enum constant: ``DELETE FROM `GameStats`.`%s` ``
  formatted with `subType.name()`. A new mode needs a new table **and** a new enum constant.
- Schema is discovered at runtime via `SHOW TABLES` and ``SHOW FULL COLUMNS FROM `%s` `` on every backend
  startup. There is no declared schema anywhere in the repository.
- **Field metadata lives in the MySQL column comment.** `StatsManager.loadMode` reads each column's
  `Comment` and parses it as `"<localization key>;<formatter name>"`, split on `;`. A column whose
  comment is empty is skipped entirely — the stat becomes invisible to the platform. So the schema,
  the localization key and the presentation formatter are all stored in a MySQL column comment, and
  none of it is in version control. Consequences: the set of statistics cannot be known without
  connecting to the production database; a migration that drops a comment silently deletes a stat
  from the game; renaming a formatter breaks rendering with no compile-time signal.
- ``SELECT `userID`, `%s` FROM `%s` WHERE `type` = ? ORDER BY `%s` DESC LIMIT ?`` — **three
  string-interpolated identifiers** in one query, and an unindexed sort for every leaderboard read.
- Backends write statistics **directly to MySQL**, concurrently, with no coordination and no idempotency.

---

#### `darta-api-master` → `server/purpur-adapter` + `integrations/protocollib` + `server/packets`

| Field | Value |
|---|---|
| **Legacy Component** | 564 files / 44.6k LOC. `public-api` (`net.lastcraft.api`), `darta-api` (`net.lastcraft.dartaapi`), `nms` (`net.lastcraft.packetlib`) with `v1_12_R1` / `v1_16_R1` / `v1_20_R1` source sets. |
| **Responsibility** | The Bukkit-side foundation: NMS abstraction, ProtocolLib integration (71 files), GUIs, items, holograms, NPCs, fake nicknames, combat tweaks, crash protection, permissions, world time, command manager, chat log. |
| **Dependencies** | ProtocolLib 4.7.0, ByteBuddy, ASM, `commons` (374), `core-protocol` (6) |
| **Used By** | `lobby-api` (759+106), `kitpvp` (268+71), `game-api` (252+29), `arcade-games` (165+17), `game-effects` (85+14), `core-mail` (100), `core-commands` (28) |
| **Target Module** | `server/purpur-adapter`, `server/packets`, `server/gui`, `server/items`, `integrations/protocollib` |
| **Priority** | **P1** |
| **Status** | `AUDITED` |
| **Risks** | Largest NMS surface in the project (244 `v1_x_R` references). Purpur 26.2 invalidates essentially all of it. |
| **Required Behavior** | The *capabilities* — holograms, NPCs, fake names, custom GUIs, packet-level item/entity manipulation, action bar, titles, tab list — not the implementations. |
| **Migration Strategy** | A single target version (Purpur 26.2) eliminates the multi-version source sets entirely. Capabilities re-expressed against modern Paper/Purpur API first; NMS only where the API genuinely cannot express it, isolated in `server/purpur-adapter`. ProtocolLib confined to `integrations/protocollib` behind a port, per CLAUDE.md §9. |

**Defects found:**

- **Runtime bytecode redefinition of third-party classes.** `onEnable()` installs `ByteBuddyAgent`, then
  `ByteBuddy().redefine(ViaVersion's ChatRewriter).method(named("legacyTextToJsonString"))
  .intercept(MethodDelegation.to(patch))` — wrapped in `catch (Exception ignored) {}`. Silent failure,
  no fallback, breaks on any ViaVersion update.
- `Class.forName("net.lastcraft.patch.ServerRussianUsernamesPatch").getDeclaredMethod("install")
  .invoke(null)` — **reflective patching of the server's username validation**, gated on
  `NmsManager.RELEASE_1_20`.
- `onLoad()` performs the full startup sequence (server-type detection, NMS init, localization reload,
  static registry population) with **implicit ordering and no dependency declaration**. `onEnable()` then
  registers ~20 listeners, 2 ProtocolLib listeners, ~10 commands, and a `runTaskLater(…, 20L * 3)` that
  mutates world game rules — a **3-second startup race** with anything that reads game rules earlier.
- **NMS leakage confirmed outside the abstraction layer**: `net.minecraft.*` / `craftbukkit` imports in
  `game-api` (7 files), `arcade-games` (6), `kitpvp` (3), `plugins-survival` (7), `last-games` (10).
  ProtocolLib used directly in `lobby-api` (11 files), `plugins-survival` (26), `arcade-games` (3).
  **The abstraction exists but is routinely bypassed.**
- `onDisable()` contains a commented-out world-save loop with the note *"почему-то хуево сохраняется с
  этим кодом("* — world persistence on shutdown is knowingly broken.

---

#### `lobby-api-master` → `games/lobby` + `platform-cosmetics`

| Field | Value |
|---|---|
| **Legacy Component** | `box-api`, `gadgets`, `hub`, `limbo`, `lobby-api`, `placeholder`, `promo`, `rewards` — 347 files / 26.9k LOC |
| **Responsibility** | Hub/lobby experience: game selector GUIs, mystery boxes, cosmetic gadgets, daily rewards, promo codes, top-player holograms, limbo fallback server. |
| **Dependencies** | `darta-public-api` (759), `commons` (440), `darta-api` (106), `box-api` (93), `core-games` (1) |
| **Used By** | `game-api` (`TopStandData`), `kitpvp`, `game-effects` |
| **Target Module** | `games/lobby`, `platform/platform-cosmetics` |
| **Priority** | **P2** |
| **Status** | `AUDITED` |
| **Risks** | `limbo` is the failure-mode destination — it must exist before any server-lifecycle work can safely drain servers. |
| **Required Behavior** | Game menu with live per-mode online counts; mystery box open/reward; gadget cosmetics; daily/promo rewards; top holograms; limbo as a fallback target. |
| **Migration Strategy** | Cosmetics and box/reward mechanics → `platform-cosmetics` + `game-rewards` (shared). Lobby presentation → `games/lobby`. `limbo` → a `velocity-server-pool` capability (fallback target), not a Minecraft server plugin. `Limbo.java` currently uses `Thread.sleep`; replaced by scheduled tasks. |

---

#### Games: `kitpvp-main`, `arcade-games-main`, `parkour-main`, `vampirez-main`, `game-effects-main`

| Field | Value |
|---|---|
| **Responsibility** | Concrete game modes. `arcade-games` hosts multiple sub-games (BlockParty etc.) behind `ArcadeType`. |
| **Dependencies** | `game-api`, `darta-api`, `commons`, `game-effects` |
| **Target Module** | `games/kitpvp`, `games/arcade`, `games/parkour`, `games/vampirez` |
| **Priority** | **P2** |
| **Status** | `DISCOVERED` |
| **Risks** | Migrating a game before the engine is proven wastes the work. |
| **Required Behavior** | Per-game rules, extracted during each game's own migration. |
| **Migration Strategy** | **`kitpvp` is the first game to migrate** — it is self-contained (20 `game-api` imports, 3 NMS files) and exercises lifecycle, teams, kits, stats and rewards. It is the acceptance test for the new engine. `arcade-games` migrates last (237 `game-api` imports, heaviest FAWE and NMS coupling). |

---

### 3.4 P2/P3 — Platform Modules

| Legacy Component | Responsibility | Dependencies | Used By | Target Module | Priority | Status |
|---|---|---|---|---|---|---|
| `core-party-main` | Party create/invite/join/leave, party-aware routing and team assignment | `core-app`(27), `commons`(31) | `core-games`, `game-api`, `lobby-api` | `platform/platform-party` | P1 | `AUDITED` |
| `core-friends-main` (`net.lastcraft.joint`) | Friends list, requests, online notifications | `core-app`(15), `commons`(35) | `core-commands`, `discord-bot`, `vk-bot` | `platform/platform-friends` | P2 | `AUDITED` |
| `core-bans-main` | Bans/mutes/warns across Core + Bungee + Bukkit in one artifact | `core-app`(52), `commons`(56), `darta-api`, `bungee-api` | `core-reports`, `vk-bot` | `modules/moderation` | P2 | `AUDITED` |
| `core-mail-main` | In-game mail with attachments; **the reward delivery channel** (`MailRewardSender`) | `darta-public-api`(100), `core-protocol`(41), `core-app`(13) | `core-games` (rewards) | `platform/platform-mail` | P2 | `AUDITED` |
| `core-commands-main` | Staff/utility commands, staff request routing | `commons`(255), `core-app`(133), `darta-public-api`(28) | Core operators | consumers of `platform/platform-command` | P2 | `AUDITED` |
| `core-reports-main` | Player reports → staff queue | `core-app`(14), `core-bans`(1) | Staff tooling, `vk-bot` | `modules/moderation` | P3 | `AUDITED` |
| `core-anticheat-main` + `anti-cheat-master` | Click/aim heuristics, anti-VPN | `commons`, `core-app`, ProtocolLib | Backends | `modules/anticheat` | P3 | `DISCOVERED` |
| `server-stream-main` | Log/console streaming Core ↔ servers | `core-protocol`(25) | Core, connectors | `platform/platform-core` (observability) | P3 | `AUDITED` |
| `game-effects-main` | Cosmetic effects, kill effects, trails | `darta-public-api`(85), `commons`(47) | `kitpvp`, games | `platform/platform-cosmetics` | P3 | `DISCOVERED` |
| `alert-ext-master` | Alert extension (188 LOC) | — | — | `modules/` or drop | P4 | `DISCOVERED` |
| `discord-bot-main`, `vk-bot-main` | Social bridges; **`vk-bot` has 68 direct `core-app` imports** | `core-app`, `commons` | Ops | `integrations/discord` (+ VK decision) | P4 | `DISCOVERED` |
| `plugins-survival-master` | Survival / SkyBlock / anarchy / creative / market / economy / auction / protection-stones — a **separate product line**, 721 files | `darta-api`, ProtocolLib, `commons` | Survival servers | `games/survival`, `games/skyblock`, `modules/*` | P3 | `DISCOVERED` |
| `lastcraft-site-backend/frontend` | Web store, payments, Telegram | Spring, npm | Web | Out of scope | P4 | `DISCOVERED` |
| `wcommons-master` | Third-party utility lib (`io.github.whilein`): eventbus, sql, config, geo, unsafe, asm-patcher, impl-loader | — | `commons`, `core-app` | Replace with first-party / standard libs | P1 | `NOT_MIGRATING` |
| `slimeworldmanager-master` | Third-party fast world format | — | `commons` slimeworld DAOs, map system | `server/world` (evaluate modern SWM/ASWM) | P2 | `DISCOVERED` |
| `builders-utilities-master` | Third-party build tools (`net.arcaniax`) | — | Build servers | `modules/` (vendor as-is) | P4 | `NOT_MIGRATING` |
| `paper-1-12/1-16/1-19/1.20` | Vendored Paper forks (1.17M LOC) | — | Backends | Purpur 26.2 (calendar versioning; ADR-0022) | P0 | `NOT_MIGRATING` |
| `wproxy-master` | BungeeCord fork | — | Production proxy | Velocity 4.2.0 | P1 | `NOT_MIGRATING` |
| `last-core-master`, `last-games-master` | Superseded monorepo snapshots | — | — | — | P4 | `NOT_MIGRATING` |

---

## 4. Cross-Cutting Findings

### 4.1 Circular and inverted dependencies

Measured by counting `import` statements across repository boundaries.

| Edge | Count | Problem |
|---|---|---|
| `game-api` → `core-games` | 3 | **Inversion.** Bukkit game framework imports Core-service module classes (`CoreMapCountEvent`, `BukkitMapCountPacket`). `core-games` → `core-app` (44), so a backend plugin transitively needs the Core application on its classpath. |
| `game-api` → `lobby-api` | ≥1 | **Inversion.** `StatsManager` imports `net.lastcraft.lobby.game.top.TopStandData`; `lobby-api` also depends on `game-api` concepts. Framework ↔ consumer cycle. |
| `game-api` → `core-party` | 2 | Game framework reaches directly into the party module instead of an abstraction. **Resolved** (ADR-0015): the engine takes togetherness as data (`PlayerGroup`) and never names a party. |
| `lobby-api` → `core-games` | 1 | Same inversion, lobby side. |
| `core-bans` → `darta-api` + `bungee-api` | 4+1 | A Core-service module depends on **both** the Bukkit and the Bungee platform APIs — one artifact spanning three runtimes. |
| `core-mail` → `darta-public-api` | 100 | Core-service module heavily coupled to the Bukkit API. |
| `darta-api` → `core-protocol` + `connector` | 6+7 | The NMS abstraction layer knows the Core wire protocol. |
| `core-games` → `core-party`, `core-mail`, `core-bans` | 4/6/1 | Game routing coupled to social and moderation modules. |
| `core-friends` → `core-commands`; `core-party` → `core-commands` | 2/1 | Feature modules depend on the command *module* rather than a command *API*. |
| **everything** → `commons` | 449 internal, 2000+ external | The hub. Dissolving `commons` is the precondition for every other decoupling. |

**Rule for the new platform (architecture-test enforced):** dependencies point inward only —
`Game → Game Platform → Platform API → Infrastructure adapters`. No module may import a sibling's
implementation package. No shared "commons" module.

### 4.2 Static / global mutable state

| Location | Nature |
|---|---|
| `SubType.current`, `GameType.current` | **`public static` non-volatile mutable** server identity, read from 21 sites across threads |
| `Core.instance` | Assigned in the constructor before initialization completes |
| `Game.instance` | Already `@Deprecated` by the previous authors; limits the server to one match |
| `GlobalLoader` | `@UtilityClass`, 51 static methods; static init opens a pool and runs DDL |
| `PlayerInfoLoader` | Second static pool |
| `GameManagerRegistry` | Static service locator with unchecked casts |
| `GameItemRegistry`, `ItemCategoryRegistry`, `StatsModeRegistry`, `StatsFieldFormatterRegistry`, `GuiRegistry`, `GameUserRegistry` | Static registries in `game-api` |
| `BukkitCommandManager.IMP`, `InjectManager.IMP`, `PermissionManager.IMP`, `ChatLogManager.IMP`, `GradientManager.IMP`, `Concurrency.IMP` | `IMP` singleton convention throughout |
| `KeysManager`, `BoosterManager` | Static registries populated at plugin load |
| `Language` | Static localization store; `Language.reloadAll()`, `Language.setEnvironment(...)`. **Migrated** to `platform-i18n` + `infrastructure/localization`: the catalogue is an immutable value behind one `volatile` reference, replaced whole or not at all (ADR-0028) |
| `WorldTime` | Static world-time freeze registry keyed by world-name string |
| `UpdateLanguage.INSTANCE` | Shared **mutable** packet instance |
| `plugins-survival` | 20 further `IMP`/`INSTANCE` singletons |

Total: **~50 identified global mutable singletons.**

### 4.3 Blocking on latency-sensitive threads

| Site | Impact |
|---|---|
| `GlobalLoader.containsPlayerID()` → `.join()` | Blocking JDBC reachable from the Netty event loop and command dispatch |
| `CorePlayer` constructor → `load0()` | Synchronous multi-table JDBC while handling the login packet |
| `MoneySection.loadData()` → `GlobalLoader.getPlayerMoney()` | Synchronous JDBC per section, per login |
| `StatsManager.loadTables()` | `SHOW TABLES` + `SHOW FULL COLUMNS` synchronously during plugin enable |
| `AbstractConcurrency._parallel()` | `latch.await()` with **no timeout**; blocks the caller; permanent hang if any worker task is dropped |
| `Core.shutdown()` | `awaitTermination(Long.MAX_VALUE)` |
| `Thread.sleep` in `Limbo.java`, `RestartServer.java`, `ActionBarAPIImpl.java`, `GuiUpdater.java`, `ScheduledTask.java` | Occupying pooled or main threads |
| Every backend holding its own MySQL pool | All DB latency lands on Bukkit threads |

### 4.4 Version and compatibility constraints

- Backends target **Minecraft 1.12.2** (`net.lastcraft:paper-1.12`) with partial 1.16.5 and 1.20.1
  support. Target is **Purpur 26.2** — the jump now spans the switch to calendar versioning. Essentially **no NMS code survives**.
- Java 17 (`commons`, `last-core`, `lobby-api`) and Java 21 (`game-api`, `kitpvp`, `arcade-games`).
  Target is Java 25.
- `worldedit-bukkit:6.1.4-abelix` (a **custom fork**) and `FAWE:21.03.26` — both years out of date.
- ProtocolLib 4.7.0 — predates 1.19+.
- `mysql-connector-java:5.1.42` — EOL, known CVEs.
- Nashorn via `org.openjdk.nashorn:nashorn-core:15.3` — scripting engine removed from the JDK.
- Gradle builds pull from a private authenticated GitLab (`git.abelix.club`) and Nexus
  (`repo.lc.team`) requiring `LASTCRAFT_ACCESS_TOKEN` / `CI_JOB_TOKEN`. **Legacy cannot be built
  locally.** Behavioral verification must be by reading and by parity tests against the new
  implementation, not by running the old system. *(Not a blocker — the audit is source-based.)*

### 4.5 Missing capabilities (present in the target, absent in legacy)

| Capability | Legacy state |
|---|---|
| Server health monitoring | **None.** No heartbeat, no health check, no liveness probe anywhere in the codebase. Liveness = "the TCP channel is open." |
| Server lifecycle | **None.** No `PROVISIONING`/`DRAINING`/`RECOVERING`. Servers appear on connect and vanish on disconnect. |
| Automated recovery | **None.** `RestartServer.java` uses `Thread.sleep` + process exit. |
| Capacity control | **None.** No autoscaling, no pool sizing, no admission control. |
| Queue | **None.** A redirect either succeeds immediately or tells the player "no server available". |
| Metrics | Prometheus libraries are declared in `libs.gradle` but **no instrumentation exists**. |
| Distributed messaging | Redis is used as a **cache only** — no pub/sub, no streams, no distributed events. All fan-out goes through the single Core process. |
| Schema migrations | **None.** DDL executed imperatively from static initializers. |
| Idempotency / dedup | **None** in the protocol. Redelivery duplicates effects (money, rewards, mail). |
| Tests | Effectively none. Only `commons/src/test/.../ConcurrencyTests.java` found in scope. |
| Tracing | None. |

### 4.6 Security findings

| Finding | Location |
|---|---|
| Remote code execution by design — Nashorn engine bound to `core`, driven by `BukkitScriptExecute` / `@ScriptExecutable` packets over the wire | `Core.newScriptEngine()`, `core-protocol` |
| Hardcoded MaxMind license key `b8Q9ivieZzTQ3d5Q` | `Core.java` |
| SQL identifier interpolation in leaderboard and stats queries (`ORDER BY %s`, `FROM %s`) | `StatsManager`, `DefaultStatsDao` |
| Runtime bytecode redefinition of third-party classes, failures silently swallowed | `DartaAPI.onEnable()` |
| Reflective patching of server username validation | `ServerRussianUsernamesPatch` |
| Per-server YAML permission files, unsynchronized across the network | `lastperms.yml` |
| `print(System.getenv("LASTCRAFT_ACCESS_TOKEN"))` in a Gradle publish block | `core-protocol-main/build.gradle` |
| Redis without TLS, password from an env var, port hardcoded | `Core.java` |
| Auth: PBKDF2 (acceptable) but credentials and IP history handled inside the god object, with `throws SQLException` on the public API | `core-application/security/*` |

---

## 5. Behavioral Compatibility Contracts

### 5.1 Economy

```text
Legacy behavior:
  Read balance into memory at login. Mutations compute newValue = old + delta in the JVM
  and persist an absolute value. A `mysql` flag may skip persistence entirely. Cross-server
  sync is a packet carrying an add-or-set boolean.

Modern behavior:
  Append-only transaction ledger + materialized balance. Atomic conditional delta at the
  database. Every mutation carries an idempotency key. Core is the only writer.

Must remain compatible:
  Three currencies (MYSTERY_DUST=0, GOLD=1, BOX_DUST=2) and their persisted ids.
  Balance never negative. Existing balances migrate exactly.

Intentionally changed:
  Lost updates eliminated. Redelivery no longer duplicates currency. Full audit trail.
  Backends can no longer write money directly.

Deprecated:
  `boolean mysql` opt-out. `updateMoneyForCore(..., boolean update)`.

Unknown:
  Whether any production data was already corrupted by the lost-update race.
  -> Reconciliation report required before cutover.
```

### 5.2 Game mode identity

```text
Legacy behavior:
  Two global mutable enums. Server identity parsed from a string with three fallbacks.
  Stats tables named after enum constants. New mode = edit commons + redeploy everything.

Modern behavior:
  `GameDefinition` registered by its own module. Namespaced key + stable numeric id.
  Server identity supplied explicitly at startup and validated against the registry.

Must remain compatible:
  Numeric ids of all existing modes (persisted in stats tables and shop item scoping).
  Solo/doubles/team/ranked team sizes. Lobby channel routing names.

Intentionally changed:
  Adding a game mode touches exactly one new module. Duplicate id GameType 90
  (KIT_PVP/VAMPIREZ) resolved — VAMPIREZ gets a new id, with a migration.

Deprecated:
  `SubType.current`, `GameType.current`, `SubType.valueOf` server-type parsing.

Unknown:
  Whether any external system (site backend, bots) reads the duplicated id 90.
  -> Grep of `lastcraft-site-backend` and the bots required before the id change.
```

### 5.3 Matchmaking / redirect

```text
Legacy behavior:
  Linear scan of servers whose name starts with the requested prefix, filtered to
  gameState == WAITING, using capacity data up to 5 seconds stale. No reservation,
  no queue. EMERALD+ donors fall back to the last full server encountered.

Modern behavior:
  Server registry fed by heartbeats. Matchmaking strategies (FIFO / party-aware /
  latency-aware / load-aware). Slot reservation with TTL. Real queue with position.

Must remain compatible:
  Party members always land on the same server, together, atomically.
  EMERALD+ (level >= 3) slot reservation when all servers are full.
  Ranked requires a party of exactly 4.
  Localized messages: RANKED_REQUIREMENTS, AVAILABLE_SERVER_NOT_FOUND, SERVER_QUEUE.

Intentionally changed:
  Reservation removes the double-admission race. Best-fit comparison corrected.
  Server role comes from registry metadata, not a name prefix.

Deprecated:
  `Bukkit.getName().startsWith(serverType)`. `Core.isLimbo()` name-prefix check.

Unknown:
  The exact intended packing policy (the legacy comparison is provably inconsistent).
  -> Product decision needed: pack-tightest vs spread-evenly. Defaulting to pack-tightest
     (matches the apparent intent) and documenting it.
```

### 5.4 Game lifecycle

```text
Legacy behavior:
  5 states, unvalidated transitions, one Game per JVM, state change broadcasts server info.

Modern behavior:
  10 states (ARCHITECTURE.md §4 / CLAUDE.md §15) with a validated transition table,
  entry/exit hooks, and N concurrent GameInstances per server.

Must remain compatible:
  Observable phase semantics for players: waiting lobby -> countdown -> play -> end ->
  reset. Server info must continue to publish a state the proxy understands.

Intentionally changed:
  Multiple concurrent instances per server. Illegal transitions rejected, not silently applied.

Deprecated:
  `Game.instance`. `GameManagerRegistry` static lookup. Listener-per-class map.

Unknown:
  Whether any game relies on an irregular transition (e.g. GAME -> WAITING).
  -> Enumerate all setState call sites per game during that game's migration.
```

---

## 6. Recommended Migration Order

Derived from the measured dependency graph, not from the CLAUDE.md template order. The template order is
confirmed correct with two adjustments, marked ⚠.

```text
 0. Build foundation        Maven multi-module, Java 25, architecture tests (ArchUnit), CI.
                            Architecture tests land FIRST so violations can never accumulate.

 1. platform-api            Ports only. No implementations. Defines the inward direction.

 2. infrastructure/database PostgreSQL + HikariCP + migrations + async repository base.
    infrastructure/cache    Caffeine L1 -> Redis L2 -> Postgres L3, single-flight, jitter.
    infrastructure/messaging Versioned envelope, Redis transport, idempotency keys.
                            [WARN] Promoted ahead of the player model: the legacy player model
                              is inseparable from GlobalLoader, so persistence and messaging
                              must exist before a player can be modelled at all.

 3. platform-player         PlayerIdentity / PlayerProfile / PlayerSession /
                            TemporaryMatchState. Async load before join completes.

 4. platform-permissions    Permission servers + group graph. Unblocks commands and every
                            gated feature. Replaces 74 hardcoded enum comparisons.

 5. platform-command        Annotation surface preserved, MethodHandles instead of ASM.
                            Depends on permissions for conditions.

 6. platform-economy        Ledger + atomic deltas + idempotency. ADR-0004.
                            [WARN] Promoted ahead of statistics: it is the only subsystem with
                              a known live correctness defect.

 7. platform-statistics     Declared schema, write-through via Core, cached leaderboards.

 8. platform-core           Core service decomposition: registries, auth, module container,
                            scheduler. Legacy protocol bridge for rollout. ADR-0006.

 9. game-platform/game-api  GameDefinition registry. ADR-0003. Kills SubType/GameType.
    game-platform/game-engine  Lifecycle, instances, teams, arenas, countdown, rewards,
                            spectators, boards. THE deliverable.
    game-platform/game-runtime

10. proxy/velocity-*        Velocity 4.2.0 (not 4.3.x — ADR-0020). Routing, server pool, heartbeat-fed registry,
                            matchmaking, queue. ADR-0005.

11. server/purpur-*         Purpur 26.2 adapter, GUI, items, scheduler, packets.
                            integrations/protocollib, integrations/fawe behind ports.

12. games/kitpvp            FIRST game migration — the engine's acceptance test.
                            Gate: if kitpvp is not dramatically cheaper than the legacy
                            equivalent, fix the engine before migrating any other game.

13. games/parkour, vampirez, arcade   Remaining minigames, easiest -> hardest.

14. modules/anticheat, moderation     AntiCheat, bans, reports.

15. server/world            FAWE integration, arena reset, SlimeWorld evaluation.

16. automation              Trigger/Condition/Action/Policy on top of a working platform.

17. production hardening    Health, recovery, capacity control, metrics, tracing, runbooks.
```

**Ordering constraints that are hard requirements:**

- Architecture tests before any module code. Otherwise the legacy coupling patterns reappear.
- `messaging` before `platform-core`, because decomposing Core requires a transport that can carry a
  versioned envelope.
- `permissions` before `command`, because command conditions are permission checks.
- `game-api` (definitions) before `game-engine`, because the engine is parameterized by definitions.
- `game-engine` before any game, and `kitpvp` before the rest, as the engine's proof.
- `limbo`/fallback capability before any draining or recovery work.

---

## 7. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Economy already corrupted in production by the lost-update race | **High** | Reconciliation report from the ledger migration before cutover. |
| Protocol change breaks all servers simultaneously | **High** | Versioned envelope + dual-stack bridge (ADR-0006). Never a big-bang protocol swap. |
| `commons` dissolution stalls and a new "commons" appears | **High** | Architecture test forbidding any module that spans more than one bounded concern. |
| 9-version Minecraft jump invalidates all NMS | **High** | Accepted: rewrite against modern API, NMS only where unavoidable, isolated in the adapter. |
| Engine rebuilt without actually reducing per-game cost | **High** | `kitpvp` is a hard gate with an explicit cost comparison. |
| Legacy cannot be built (private repos, tokens) | Medium | Audit is source-based; parity verified by tests against documented behavior, not by running legacy. |
| Persisted ids (`SubType`, `GameType`, `Group`, `PurchaseType`) must survive | Medium | Registry retains stable numeric ids; migrations for the duplicate-90 fix. |
| Stats schema exists only in production | Medium | Extract the real schema from a production dump before `platform-statistics`. |
| Single Core with no clustering remains a SPOF | Medium | Not solved in phase 8; explicitly deferred to phase 17 with a documented HA design. |
| `plugins-survival` (74.8k LOC, separate product line) underestimated | Medium | Out of scope until phase 13+; separate audit before it starts. |

---

## 8. Blockers

**None blocking the current phase.** Two items require input before the phases that depend on them:

1. **Production stats schema** — *narrowed at phase 7 (ADR-0012)*. The platform no longer needs it:
   each game declares its own schema in code, and storage requires no per-mode DDL. A production dump
   of `GameStats` is still needed to **migrate existing data** for live modes, since their field names
   exist only as MySQL column comments. That is now a per-game migration concern, not a design blocker.
2. **Server-packing policy** (needed at phase 10). The legacy comparison is provably inconsistent, so
   intent cannot be recovered from the code. Proceeding with **pack-tightest** as the documented
   default; changing it later is a one-line strategy swap.

---

## 9. Audit Coverage

Audited by targeted search, not full reads: entry points (`plugin.yml` / `bungee.yml` / `Bootstrap`),
build files and version catalogs of all in-scope repositories, the cross-repository import graph
(18 package roots × 26 repositories), and full reads of the decisive files: `Core`, `Bootstrap`,
`CoreMainThread`, `ConnectionStorage`, `CorePlayer`, `GlobalLoader`, `MoneySection`,
`AbstractConcurrency`, `Game`, `GameState`, `SubType`, `GameType`, `Group`, `PurchaseType`,
`RedirectManager`, `DefaultStatsDao`, `StatsManager`, `DartaAPI`.

Not yet audited in depth (deferred to their own phases, marked `DISCOVERED` above):
`plugins-survival-master`, `lastcraft-site-*`, `vk-bot`, `discord-bot`, `core-anticheat`,
`game-effects`, and the `wproxy` diff against upstream BungeeCord.
