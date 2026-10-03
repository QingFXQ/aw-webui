# AGENTS.md

## Project scope

This repository is a personal ActivityWatch WebUI fork focused on a simpler, Chinese-first software usage experience.

Primary goals:

1. Make the default experience understandable in seconds.
2. Present software usage as a dense, attractive personal software library: clear app identity, lifetime time, recent-period time, search, sorting, and low cognitive load.
3. Show today / week / month / lifetime active time without turning the home screen into an analytics/admin dashboard.
4. Support per-device and combined multi-device views when upstream ActivityWatch data is available.
5. Reuse upstream ActivityWatch collection, AFK filtering, querying, categories, and sync behavior.
6. Keep the fork small enough to rebase onto upstream without turning it into a separate product stack.

The visual direction may borrow interaction patterns from consumer game/software libraries such as high-density dark lists, but must not be locked to a literal Steam clone or any one product's visual identity.

Do not rewrite watchers, the server, storage, sync, or ActivityWatch query semantics unless a feature cannot be implemented safely in WebUI alone.

This repository is intentionally independent from LifeQuest. Do not add LifeQuest bridges, import/export flows, XP hooks, task settlement, or LifeQuest-specific data contracts unless the user explicitly starts a later integration phase.

## Model and reasoning allocation

When model routing is available, assign work by risk and breadth.

- Low complexity / low reasoning: GPT-5.6 Luna or equivalent. Use for copy changes, simple i18n strings, CSS spacing, icon swaps, and mechanical cleanup.
- Normal feature work / medium reasoning: GPT-5.6 Sol Medium. Use for isolated Vue components, dashboard/library widgets, local state, and ordinary query wiring.
- Cross-module or data-sensitive work / high reasoning: GPT-5.6 Sol High or GPT-6 Astra Work. Use for multi-device aggregation, ActivityWatch query semantics, performance work, sync-facing UI, and changes spanning stores + views + query helpers.
- Highest-risk work: GPT-6 Pro or the strongest available reviewer. Reserve for changes that can corrupt data, alter sync semantics, modify persistence, or create difficult upstream merge conflicts.

Do not spend high-reasoning models on formatting or mechanical edits.

## Engineering policy

Prefer the smallest change that delivers the user-visible behavior.

Avoid defensive architecture that has no current failure mode. Do not add duplicate validation layers, shadow state machines, whole-database convergence checks, redundant checksums, or generalized abstractions for hypothetical future requirements.

Hashing is allowed only when it protects real integrity boundaries such as file identity, sync payload integrity, or cache invalidation. Do not hash large state merely to prove that two already-derived views are equivalent.

Keep upstream-compatible APIs and stores intact where possible. For personal UI behavior, prefer a local adapter/helper or isolated view over changing core ActivityWatch semantics.

Do not silently change what ActivityWatch counts as active time. Any custom metric must document its counting rule in code comments when it differs from the upstream Activity view.

## Testing tiers

Use tiered testing. Do not run the full repository matrix after every small edit.

### Fast gate — every code change

Run the narrowest applicable checks:

- Type/script parse or TypeScript check for touched files.
- Relevant unit tests.
- Lint for touched frontend code.
- Production frontend build when the environment can run it.

### Smoke gate — every user-visible home/library change

Verify at minimum:

1. Home/library loads with one desktop device.
2. Empty/no-data state renders without crashing.
3. Today/week/month totals render.
4. Software list renders and can be searched/sorted.
5. Lifetime calculation can start without blocking the main library.
6. Timeline and Settings routes remain reachable.
7. If multiple devices exist, switching between a single device and all devices works.
8. Repeated refreshes do not leave stale results on screen.
9. Dark library layout remains readable at desktop and narrow widths.

### Full gate — before merge/release or after shared-query changes

Run upstream build/unit/e2e workflows or their local equivalents. Full e2e is required when touching shared query logic, routing infrastructure, stores used by multiple pages, or upstream-facing behavior.

A failing unrelated upstream test must be diagnosed before being fixed. Do not widen the task automatically just because an unrelated test is red.

## Review rules

Before declaring a task complete:

- Review the diff for accidental upstream rewrites.
- Check loading, empty, error, and cancellation states.
- Check desktop and narrow/mobile layouts when UI changed.
- Check that long-range queries are bounded and do not issue one expensive request per day when an aggregate query can do the job.
- Check that multi-device totals deduplicate overlapping time using existing ActivityWatch semantics.
- Record known limitations explicitly instead of hiding them behind extra fallback layers.

## Current product direction

Default navigation should remain simple:

- 总览 / 软件库
- 活动明细
- 时间线
- 设置

Advanced/debug/raw-data tools may remain available, but should be grouped under an advanced menu rather than competing with the main workflow.

The home screen should feel like a polished consumer software library rather than an analytics/admin console: dark, compact, glanceable, list-first, and centered on real usage time.
