# LastCraft — Current Task Context

## Mission

Rebuild the old LastCraft platform into a modern production-grade Minecraft platform.

The goal is not a direct rewrite.

Understand the old system, preserve important behavior, remove obsolete architecture, and build a substantially better foundation.

## Legacy Source

Legacy source:

legacy/source/

READ ONLY.

Never modify it.
Never create production dependencies on it.

## FIRST TASK — LEGACY AUDIT

Before major implementation, perform a comprehensive audit of legacy/source/.

Inspect:

- modules
- entry points
- dependencies
- shared infrastructure
- player lifecycle
- game lifecycle
- database usage
- Redis/cache
- proxy communication
- server communication
- NMS
- ProtocolLib
- commands
- permissions
- events
- scheduled tasks
- world management
- economy
- rewards
- statistics
- configuration
- persistence
- hidden coupling
- static/global state
- concurrency
- initialization order
- obsolete functionality
- compatibility requirements
- important edge cases

Use targeted search.

Do not blindly read the entire repository.

## MIGRATION MAP

Create and continuously maintain:

docs/MIGRATION-MAP.md

The migration map must contain:

- legacy → modern mapping
- dependencies
- responsibilities
- consumers
- migration priorities
- migration status
- behavioral compatibility
- intentional changes
- risks
- blockers
- recommended migration order

Priorities:

P0 — foundational/blocking
P1 — critical platform
P2 — important gameplay
P3 — secondary
P4 — optional/deprecated

Statuses:

DISCOVERED
AUDITED
DESIGNED
IMPLEMENTING
IMPLEMENTED
TESTING
VERIFIED
DEPRECATED
NOT_MIGRATING

## MIGRATION RULE

Never blindly convert:

legacy class → modern class

Instead:

legacy implementation
→ behavior analysis
→ domain model
→ dependency analysis
→ modern design
→ implementation
→ tests
→ behavior verification

Preserve required behavior.

Do not preserve bad architecture.

Do not reproduce legacy bugs unless compatibility explicitly requires them.

## GAME ENGINE

The Game Engine must make every subsequent game significantly cheaper to develop.

Extract genuinely shared functionality:

- lifecycle
- players
- teams
- arenas
- countdown
- matchmaking
- voting
- rewards
- statistics
- scoreboards
- cosmetics
- world reset
- events
- configuration
- observability

Do not create a giant generic framework.

Do not force game-specific mechanics into the Game Engine.

If a new game still requires rebuilding common infrastructure, improve the Game Engine.

## ENGINEERING RULE

Do not build architecture for the sake of architecture.

Every abstraction must solve a real problem.

Before creating an interface, manager, factory, registry, strategy, service, framework or DSL, determine:

1. What real problem does it solve?
2. Does that problem actually exist?
3. Is the abstraction reusable?
4. Does it reduce complexity/coupling/duplication?
5. Is there a simpler solution?

If there is no strong answer:

DO NOT CREATE THE ABSTRACTION.

Prefer:

- simple solutions
- explicit code
- composition
- focused components
- real reuse
- proven patterns

Avoid:

- architecture astronautics
- speculative abstractions
- meaningless interfaces
- unnecessary managers
- unnecessary factories
- excessive layers
- giant generic frameworks

## EXECUTION

Work autonomously.

Use:

Search
→ Understand
→ Design
→ Implement
→ Compile
→ Test
→ Fix
→ Validate
→ Continue

Do not stop at skeletons.

Do not create fake implementations.

Do not wait for approval for normal engineering decisions.

## DEVELOPMENT PRIORITY

After the audit, use approximately:

1. Platform Foundation
2. Database
3. Cache
4. Messaging
5. Player
6. Permissions
7. Economy
8. Statistics
9. Game Engine
10. Game Runtime
11. Velocity
12. Purpur
13. Games
14. AntiCheat / Protocol
15. World / FAWE
16. Automation
17. Production Hardening

Adjust according to actual dependencies discovered during the audit.

## NO GIT REQUIREMENT

Git is not required.

Do not block implementation because Git is unavailable.

## QUOTA OPTIMIZATION

Minimize unnecessary context usage.

- search before reading
- targeted reads
- do not reread unchanged files
- batch related changes
- reuse existing abstractions
- avoid duplicate infrastructure
- use targeted Maven commands
- avoid unnecessary full builds
- avoid unnecessary explanations

## QUALITY BAR

The result must be:

- production-grade
- performant
- reliable
- observable
- testable
- scalable
- maintainable
- easy to extend

But always prefer:

Simplicity > unnecessary architecture

Real reuse > speculative reuse

Working software > documentation theater

Measured optimization > theoretical optimization

Behavioral correctness > legacy implementation similarity

## FINAL PRINCIPLE

Do not ask:

"How do I rewrite the old code?"

Ask:

"What behavior does the old system provide, what problems does it have, and what is the simplest modern design that preserves the required behavior while making the platform substantially easier to evolve?"