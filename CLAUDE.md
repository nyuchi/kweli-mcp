# CLAUDE.md — kweli-mcp

AI assistant briefing for this repository — the mental model and the rules
to follow when changing it.

## What this repo is

Mukoko Kweli's agentic surface: MCP servers, ingestion agents, and their
shared skills, packaged as a **pnpm + turbo monorepo of Cloudflare Workers**.
It is the intended eventual extraction target for the MCP surface that today
still lives partly in `nyuchi/kweli` (`app/mcp/route.ts`) and partly in
`nyuchi/kweli`'s `workers/fundi-ingestion/` — see [`MIGRATION.md`](./MIGRATION.md)
for exactly what has and hasn't moved yet. Mirrors the precedent nhimbe
already set: `nyuchi/mukoko-events-mcp` was extracted from `nyuchi/nhimbe`'s
former `worker/` directory the same way.

Every fact Kweli owns (places, organizations, the 4-tier verification
ladder) still lives in the shared Mukoko v3.1 MongoDB cluster, in the same
`places.places` / `entity.entities` collections documented in `nyuchi/kweli`'s
own CLAUDE.md. This repo never becomes a second source of truth for that
data — it reads and writes the same collections, through the same field
shapes. **The verification tier ladder is mirrored in three places and must
never drift**: `nyuchi/kweli`'s `lib/verification-tiers.ts`, `nyuchi/api-gateway`'s
`gateway/routers/kweli.py` (`TIER_LADDER`), and this repo's
`packages/mongo/src/verification-tiers.ts`. Update all three in the same PR
if the ladder ever changes.

## Layout and why it's split this way

```
apps/
  mcp-places/                 Public, unauthenticated, read-only graph MCP
                               (search_places, get_place, get_organization,
                               get_verification, get_open_stats) + the one
                               write tool, request_place. Stateless JSON-RPC
                               over POST /mcp — no Durable Object needed,
                               ported near-verbatim from nyuchi/kweli's
                               lib/mcp/server.ts + lib/mcp/tools.ts.

  mcp-ingestion/               WorkOS OAuth-gated MCP (FundiMcp, a Cloudflare
                               Agent / Durable Object) + the M2M-gated
                               POST /tasks submit surface. Different auth
                               model than mcp-places — that's the reason it's
                               a separate worker, not a merged "kitchen sink"
                               MCP. Producer on the shared
                               `fundi-ingestion-tasks` queue; reader of the
                               shared `fundi-ingestion-ledger` D1 database.

  bulk-ingestion-agent/        FundiAgent — the actual executor, one Cloudflare
                               Agent (SQLite-backed Durable Object) per task.
                               Sole consumer of the queue, sole writer of the
                               D1 ledger and of places.places/entity.entities.
                               The direct working port of nyuchi/kweli's
                               workers/fundi-ingestion agent runtime — this is
                               the piece that must actually work, not a stub.

  single-place-agent/          STUB. Creates exactly one named place on
                               request instead of an area sweep — the gap a
                               cross-repo audit found (nhimbe's "create an
                               event for your company" flow needs this, not
                               bulk seeding). See its agent-do.ts TODO.

  verification-review-agent/   STUB. Assists (never replaces) human review of
                               entity.representativeClaims. Never writes
                               bundu.verificationTier itself — Kweli's own
                               admin console stays the sole writer of tiers,
                               same rule as api-gateway's read-only `kweli`
                               namespace and the ingestion agents (tier 0
                               only, never higher).

packages/
  mongo/     Shared MongoClient builder + DB/COLLECTION name constants +
             the verification-tier ladder. Consumed by every app that talks
             to the shared Mukoko cluster.
  shared/    The ingestion task/region/bulk-intent domain: types.ts (Zod
             schemas), ledger.ts (D1 client), africa.ts (the boundary guard —
             config-driven, not hardcoded to a continent check baked into the
             engine), uuid.ts (UUIDv7), pluscode.ts (Open Location Code,
             vendored, no API/key), generators.ts (bulk-intent → many
             SeedTasks). Consumed by mcp-ingestion (producer) and
             bulk-ingestion-agent (consumer).
  skills/    Fundi's agentic capability modules: overpass (OSM query),
             classify (OSM feature → schema.org type), description (AI
             description via Workers AI / shamwari gateway), resolve-hierarchy
             (Nominatim reverse-geocode → placesGeo containment),
             tile-region (bbox tiling math), what3words + wikidata
             (enrichment), write-records (the actual Mongo place+entity
             write). Consumed by bulk-ingestion-agent (all of them) and
             mcp-ingestion (overpass + resolve-hierarchy, for the
             overpass_lookup / resolve_hierarchy read tools).
```

## Worker bindings — the literal "bound together" wiring

Two real Cloudflare `[[services]]` bindings exist (a service binding's fetch
never leaves Cloudflare's network — this is the actual mechanism, not shared
resources or a naming convention):

1. `mcp-ingestion` → `bulk-ingestion-agent` (`BULK_AGENT`): `POST /internal/force-run/proxy`
   on mcp-ingestion forwards to `POST /internal/force-run` on
   bulk-ingestion-agent, nudging a specific stuck task's DO to retry
   immediately instead of waiting for the hourly cron sweep.
2. `mcp-places` → `single-place-agent` (`SINGLE_PLACE_AGENT`): the
   `request_place` tool forwards to `POST /tasks` there.

Beyond those, `mcp-ingestion` and `bulk-ingestion-agent` also share two
**named resources by id** (not a service binding, but the same
multiple-workers-one-resource pattern `nyuchi/nyuchi-docs`'s two AI Search
workers already use): the `fundi-ingestion-tasks` queue (mcp-ingestion
produces, bulk-ingestion-agent consumes) and the `fundi-ingestion-ledger` D1
database (bulk-ingestion-agent writes, mcp-ingestion reads for its
`task_status`/`task_records` tools).

`single-place-agent` and `verification-review-agent` do not yet bind to
anything beyond their own Durable Object — they're stubs; wire them up for
real once their TODOs are implemented.

## Rules

1. **`bundu.verificationTier` is never written above 0 by anything in this
   repo.** Bulk and single-place ingestion always create tier-0 (unverified)
   records — Kweli's own claim → verify flow is the only path to a higher
   tier. `verification-review-agent` drafts and flags; it never approves.
2. **The Africa boundary guard (`packages/shared/africa.ts`) is
   config-driven** (`FUNDI_BOUNDARY_BBOX`), not hardcoded into the ingestion
   engine — lifting to global scope later is a var change, not a rewrite.
3. **Every collection this repo touches already exists in the shared Mukoko
   v3.1 cluster.** Never invent a parallel collection for a concept that
   already has a home (`places.places`, `entity.entities`,
   `entity.representativeClaims`, `places.placesGeo`) — this is the same
   "single source of truth" rule `nyuchi/kweli`'s CLAUDE.md states, and it
   applies here with equal force since this repo writes to the same cluster.
4. **`bson` is pinned to `7.2.0`** everywhere (dependency + pnpm override):
   newer `bson` generates random bytes in module global scope, which Workers
   upload validation rejects (error 10021).
5. **Ambient `Env` types are hand-maintained** (`src/env.d.ts` per app) since
   `wrangler types` needs a live Cloudflare account this environment doesn't
   have. Regenerate with `wrangler types` (per app) once first deployed, and
   keep the hand-written version in sync with `wrangler.jsonc` until then.
6. **Deploy order matters on first deploy**: `bulk-ingestion-agent` and
   `single-place-agent` before `mcp-ingestion` / `mcp-places`, since the
   latter two declare `[[services]]` bindings pointing at the former two by
   worker name.

## Commands

```bash
pnpm install
pnpm build            # turbo run build (each app: tsc --noEmit)
pnpm test             # turbo run test (packages/shared, packages/skills)
pnpm type-check
pnpm --filter <app> dev       # wrangler dev for one app
pnpm --filter <app> deploy    # wrangler deploy for one app
```

See each app's own `.dev.vars.example` for required secrets.
