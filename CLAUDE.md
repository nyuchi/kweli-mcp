# CLAUDE.md — kweli-mcp

AI assistant briefing for this repository — the mental model and the rules
to follow when changing it.

## What this repo is

Mukoko Kweli's agentic surface: **one Kweli MCP** plus **independent place
generation agents** and their shared skills, packaged as a **pnpm + turbo
monorepo of Cloudflare Workers**. It is the intended eventual extraction
target for the MCP surface that today still lives partly in `nyuchi/kweli`
(`app/mcp/route.ts`) and partly in `nyuchi/kweli`'s `workers/fundi-ingestion/`
— see [`MIGRATION.md`](./MIGRATION.md) for exactly what has and hasn't moved
yet. Mirrors the precedent nhimbe already set: `nyuchi/mukoko-events-mcp` was
extracted from `nyuchi/nhimbe`'s former `worker/` directory the same way.

Every fact Kweli owns (places, organizations, the 4-tier verification
ladder) still lives in the shared Mukoko v3.1 MongoDB cluster, in the same
`places.places` / `entity.entities` collections documented in `nyuchi/kweli`'s
own CLAUDE.md. This repo never becomes a second source of truth for that
data — it reads and writes the same collections, through the same field
shapes. **The verification tier ladder is mirrored in three places and must
never drift**: `nyuchi/kweli`'s `lib/verification-tiers.ts`, `nyuchi/api-gateway`'s
`gateway/lib/verification.py` (`TIER_LADDER`), and this repo's
`packages/mongo/src/verification-tiers.ts`. Update all three in the same PR
if the ladder ever changes.

## The core design: agents are independent of the MCP

The MCP is **one caller among many**, not a gatekeeper the agents depend on.
`bulk-place-agent` and `single-place-agent` each expose their own public
`POST /tasks`, gated by **WorkOS M2M (`client_credentials`)** — any
Nyuchi/Mukoko app can call them directly with its own copy of that agent's
client_id/secret, exactly the way the Kweli MCP itself does. There is no
special "trusted because it's the MCP" bypass anywhere in this repo.

## Layout and why it's split this way

```
apps/
  mcp/                          The Kweli MCP. WorkOS OAuth-gated (Authorization
                                 Code + PKCE), one Cloudflare Agent (KweliMcp, a
                                 Durable Object) holding every Kweli function:
                                 graph reads (search_places, get_place,
                                 get_organization, get_verification,
                                 get_open_stats), ingestion inspection
                                 (task_status, task_records, list_recent_places,
                                 list_geo_areas, overpass_lookup,
                                 resolve_hierarchy, compute_pluscode), and place
                                 generation (seed_region, seed_admin_bulk,
                                 request_place). Generation tools mint their own
                                 WorkOS M2M token and call the two agents below
                                 over a service binding — same as any other
                                 caller, just routed through Cloudflare's network.

  bulk-place-agent/             The bulk place generator — FundiAgent, one
                                 Cloudflare Agent (SQLite-backed Durable Object)
                                 per task, driving Overpass/Wikidata lookups,
                                 classification, and enrichment for
                                 region/country-scale seeding. Owns its own
                                 public POST /tasks (WorkOS M2M, ORG-RESTRICTED
                                 to the Nyuchi org via WORKOS_ALLOWED_ORG_IDS),
                                 the queue consumer, the cron sweeper, and the
                                 `fundi-ingestion-ledger` D1 database. The direct
                                 working port of nyuchi/kweli's
                                 workers/fundi-ingestion agent runtime — this is
                                 the piece that must actually work, not a stub.

  single-place-agent/           The single place generator — STUB. Creates
                                 exactly one named place on request instead of
                                 an area sweep — the gap a cross-repo audit
                                 found (nhimbe's "create an event for your
                                 company" flow needs this, not bulk seeding).
                                 Owns its own public POST /tasks too, WorkOS M2M
                                 gated but with NO org restriction — a
                                 deliberately different M2M application than
                                 bulk-place-agent's. See its agent-do.ts TODO.

  verification-review-agent/    STUB. Assists (never replaces) human review of
                                 entity.representativeClaims. Never writes
                                 bundu.verificationTier itself — Kweli's own
                                 admin console stays the sole writer of tiers,
                                 same rule as api-gateway's read-only entities
                                 namespace and the generation agents (tier 0
                                 only, never higher).

packages/
  mongo/       Shared MongoClient builder + DB/COLLECTION name constants +
               the verification-tier ladder. Consumed by every app that talks
               to the shared Mukoko cluster.
  shared/      The ingestion task/region/bulk-intent domain: types.ts (Zod
               schemas), ledger.ts (D1 client), africa.ts (the boundary guard —
               config-driven, not hardcoded to a continent check baked into the
               engine), uuid.ts (UUIDv7), pluscode.ts (Open Location Code,
               vendored, no API/key), generators.ts (bulk-intent → many
               SeedTasks). Consumed by bulk-place-agent and read by apps/mcp
               (task_status/task_records).
  skills/      Fundi's agentic capability modules: overpass (OSM query),
               classify (OSM feature → schema.org type), description (AI
               description via Workers AI / shamwari gateway), resolve-hierarchy
               (Nominatim reverse-geocode → placesGeo containment),
               tile-region (bbox tiling math), what3words + wikidata
               (enrichment), write-records (the actual Mongo place+entity
               write). Consumed by bulk-place-agent (all of them) and apps/mcp
               (overpass + resolve-hierarchy, for its read tools).
  workos-m2m/  WorkOS client_credentials, both sides: verify.ts (agent side —
               bulk-place-agent and single-place-agent check an inbound
               token) and mint.ts (caller side — apps/mcp, or any other
               Nyuchi/Mukoko app, mints an outbound one). This is the package
               that makes "agents independent of the MCP" real: apps/mcp
               imports the exact same verify.ts contract every other caller
               is held to, just from the mint.ts side.
```

## Worker bindings — the literal "bound together" wiring

Two real Cloudflare `[[services]]` bindings exist on `apps/mcp` (a service
binding's fetch never leaves Cloudflare's network — this is the actual
mechanism, not shared resources or a naming convention):

1. `mcp` → `bulk-place-agent` (`BULK_PLACE_AGENT`): `seed_region` and
   `seed_admin_bulk` mint a WorkOS M2M token with `BULK_M2M_CLIENT_ID` /
   `BULK_M2M_CLIENT_SECRET` (org-scoped to `WORKOS_ORGANIZATION_ID`) and call
   `POST /tasks` there — the exact same call any other Nyuchi/Mukoko app
   would make.
2. `mcp` → `single-place-agent` (`SINGLE_PLACE_AGENT`): `request_place` mints
   a token with `SINGLE_M2M_CLIENT_ID` / `SINGLE_M2M_CLIENT_SECRET` (no org
   restriction) and calls `POST /tasks` there.

Beyond those, `apps/mcp` and `bulk-place-agent` share the
`fundi-ingestion-ledger` D1 database **by id** (not a service binding, but
the same multiple-workers-one-resource pattern `nyuchi/nyuchi-docs`'s two AI
Search workers already use): `bulk-place-agent` writes it, `apps/mcp` reads
it for `task_status`/`task_records`.

`verification-review-agent` does not yet bind to anything beyond its own
Durable Object — it's a stub; wire it up for real once its TODO is
implemented.

## Rules

1. **`bundu.verificationTier` is never written above 0 by anything in this
   repo.** Bulk and single-place generation always create tier-0
   (unverified) records — Kweli's own claim → verify flow is the only path
   to a higher tier. `verification-review-agent` drafts and flags; it never
   approves.
2. **Every agent's `POST /tasks` is independently callable — never add a
   bypass for "internal" callers.** apps/mcp authenticates to
   bulk-place-agent/single-place-agent exactly like any other Nyuchi/Mukoko
   app would (mint a real WorkOS M2M token, send it as a normal bearer
   header). If you're tempted to skip that because "it's just the MCP
   calling its own agent," that's the exact shortcut this design rules out.
3. **Each agent has its OWN dedicated WorkOS M2M application** — never reuse
   bulk-place-agent's client_id for single-place-agent or vice versa, and
   never reuse either for apps/mcp's own interactive OAuth login app
   (`WORKOS_CLIENT_ID`). Three distinct WorkOS applications, three distinct
   credential pairs, each app's copy of a given pair lives in that app's own
   secrets.
4. **The Africa boundary guard (`packages/shared/africa.ts`) is
   config-driven** (`FUNDI_BOUNDARY_BBOX`), not hardcoded into the ingestion
   engine — lifting to global scope later is a var change, not a rewrite.
5. **Every collection this repo touches already exists in the shared Mukoko
   v3.1 cluster.** Never invent a parallel collection for a concept that
   already has a home (`places.places`, `entity.entities`,
   `entity.representativeClaims`, `places.placesGeo`) — this is the same
   "single source of truth" rule `nyuchi/kweli`'s CLAUDE.md states, and it
   applies here with equal force since this repo writes to the same cluster.
6. **`bson` is pinned to `7.2.0`** everywhere (dependency + pnpm override):
   newer `bson` generates random bytes in module global scope, which Workers
   upload validation rejects (error 10021).
7. **Ambient `Env` types are hand-maintained** (`src/env.d.ts` per app) since
   `wrangler types` needs a live Cloudflare account this environment doesn't
   have. Regenerate with `wrangler types` (per app) once first deployed, and
   keep the hand-written version in sync with `wrangler.jsonc` until then.
8. **Deploy order matters on first deploy**: `bulk-place-agent` and
   `single-place-agent` before `apps/mcp`, since the latter declares
   `[[services]]` bindings pointing at the former two by worker name.

## Commands

```bash
pnpm install
pnpm build            # turbo run build (each app: tsc --noEmit)
pnpm test             # turbo run test (packages/shared, packages/skills)
pnpm type-check
pnpm --filter <app> dev       # wrangler dev for one app
pnpm --filter <app> deploy    # wrangler deploy for one app
```

See each app's own `.dev.vars.example` for required secrets. Both agents'
WorkOS M2M applications are registered for real (`bulk-place-agent` = "Kweli
Fundi", `single-place-agent` = "Kweli", both org Nyuchi Africa) —
what's still outstanding is generating each app's client secret in the
WorkOS dashboard (deliberately not exposed via the admin API) and setting it
as `BULK_M2M_CLIENT_SECRET` / `SINGLE_M2M_CLIENT_SECRET` on `apps/mcp` before
first deploy.
