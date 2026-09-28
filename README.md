# workspace-filtered-install-v11

Probe pattern: `workspace-filtered-install-v11`
Target PM version: pnpm 11.28.2
Categories: `tree_structure`, `install_command`
Schema version: 1.2

## Feature exercised

This probe exercises two pnpm 11.28.2 behaviors in a single
workspace:

1. **Workspace project discovery with common-ancestor computation.**
   pnpm 11.28.2 fixes the common-ancestor algorithm so that
   workspace packages at or near the filesystem root are
   discovered correctly. The probe uses three packages under
   `packages/*` — a layout that forces the common-ancestor
   computation across siblings — to exercise this code path.

2. **Filtered-install (`--filter`) and dependency verification.**
   pnpm 11.28.2 changes how `--filter` interacts with the
   dependency verification logic. When a user runs
   `pnpm install --filter @probe/api`, pnpm should still
   validate the full lockfile but only install packages
   required by the selected package(s). The UA shells out
   to pnpm in a similar filtered fashion during its pre-step;
   this probe ensures the resolver reads and interprets the
   resulting tree correctly.

## Workspace layout

```
workspace-filtered-install-v11/
├── package.json              (private root, no deps)
├── pnpm-workspace.yaml       (packages: ['packages/*'])
├── pnpm-lock.yaml            (v9.0 single-document, pnpm 11)
├── .whitesource              (Bucket A: pnpm 11.28.2, node 20.11.1)
├── README.md
├── expected-tree.json
└── packages/
    ├── shared/               (@probe/shared)
    │   └── package.json      — depends on zod@^3.22.4
    ├── api/                  (@probe/api)
    │   └── package.json      — workspace:* @probe/shared, fastify@^4.28.1
    └── cli/                  (@probe/cli)
        └── package.json      — workspace:* @probe/shared, commander@^12.1.0, zod@^3.22.4
```

## Dependency graph shape

```
@probe/api   ──► @probe/shared ──► zod@3.22.4 ◄──┐
                                                   │
@probe/cli   ──► @probe/shared                     │
             ──► zod@3.22.4 ────────────────────────┘
             ──► commander@12.1.0

@probe/api   ──► fastify@4.28.1 ──► [fastify transitive deps]
```

The `zod` package appears as a direct dependency of
`packages/shared` AND as a direct dependency of `packages/cli`.
pnpm resolves it to a single `zod@3.22.4` snapshot (no
duplication). Mend must detect exactly one `zod@3.22.4` entry
in the tree with correct parent chains from both `@probe/shared`
and `@probe/cli`.

## Expected dependency tree summary

Workspace importers:

- Root (`.`) — no dependencies
- `packages/shared` — direct: `zod@3.22.4`
- `packages/api` — direct: `@probe/shared` (local),
  `fastify@4.28.1` (registry); transitive: fastify deps,
  and `zod@3.22.4` via `@probe/shared`
- `packages/cli` — direct: `@probe/shared` (local),
  `commander@12.1.0` (registry), `zod@3.22.4` (registry)

Key assertions:

- `zod@3.22.4` — `source: "registry"`, appears once in
  the flat `packages` map. Both `@probe/shared` and
  `@probe/cli` list it in their `dependencies[]`.
- `@probe/shared` — `source: "local"` in importers of
  `packages/api` and `packages/cli`.
- `fastify@4.28.1` — `source: "registry"` with its
  transitive closure.
- `commander@12.1.0` — `source: "registry"`, no
  transitive deps.

## Mend failure modes targeted

- **Workspace packages not discovered** — if the
  common-ancestor fix is not recognized, `packages/api`
  and `packages/cli` may not be found as importers.
- **Filtered-install tree truncation** — if Mend's
  pre-step shelling out to `pnpm --filter` causes the
  resolver to see only a subset of the lockfile's importers,
  the workspace packages not matching the filter are
  silently dropped.
- **Diamond duplication** — `zod@3.22.4` incorrectly
  reported twice (once per package that directly or
  transitively depends on it).
- **Cross-workspace dep as registry** — `@probe/shared`
  reported with `source: "registry"` instead of `source:
  "local"`.

## Mend config

**Bucket A** — `js-pnpm` has no dynamic version detection
from the manifest. This probe ships `.whitesource` with:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "pnpm": "11.28.2",
      "node": "20.11.1"
    }
  }
}
```

Exact version pins: `pnpm 11.28.2`, `node 20.11.1`. Ranges
are never used — they allow `install-tool` to drift and
silently change the transitive set. `configMode` is `AUTO`
because no `whitesource.config` is present in the probe root.

## Lockfile notes

- Format: v9.0 (pnpm 9+ single-document; pnpm 11 still
  uses `lockfileVersion: '9.0'` for projects without
  `configDependencies`).
- The multi-document split (env lockfile + project lockfile)
  does NOT apply here — no `configDependencies` are
  declared, so pnpm 11.28.2 emits the standard single
  document.
- All workspace cross-references resolve to `link:` in the
  `importers` section (e.g. `@probe/shared` in `packages/api`
  resolves as `link:../shared`).
- Transitive fastify deps are enumerated in `snapshots` but
  intentionally trimmed to a representative subset in this
  hand-authored probe. A live `pnpm install` would produce
  additional transitive entries; the expected tree encodes
  only what is explicitly present in this lockfile.

## Source

pnpm 11.28.2 release — categories: `tree_structure`,
`install_command`

Key changes:
- Common ancestor computation fix for workspace packages
  at/near filesystem root
- Filtered install (`--filter`) + dependency verification
  logic changes
