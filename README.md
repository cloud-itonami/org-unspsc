# org-unspsc — the UNSPSC codeset, as source material

**名乗り.** `cloud-itonami/org-unspsc` is an **origin-plane** repository: the name is
`unspsc.org` reversed (`org-` + `unspsc`), so the subject is *someone else's
specification* — the United Nations Standard Products and Services Code, a
four-level classification of the things an organisation buys.

This repository holds **the taxonomy as source material**: the segment table, the
record shape it is published under, and the publisher/reader code that puts it on
a substrate. It does not run a business, and it is not the fleet of per-segment
actors.

## Boundary with the neighbours

Four repositories in this workspace carry "UNSPSC" in the name. They are not
duplicates; they sit on different planes.

| Repository | Owns | Grain |
|---|---|---|
| **`cloud-itonami/org-unspsc`** (this) | the codeset as data + its publication surface | segment (2-digit) |
| `cloud-itonami/unspsc` | the executable commodity-organism fleet | commodity (8-digit) |
| `kotoba-lang/unspsc` | segment → *technology capability* registry for operators | segment (2-digit) |
| `cloud-itonami/cloud-itonami-unspsc-NN` | one deployable actor per segment | one segment each |

The boundary is asserted from the other side too — `cloud-itonami/unspsc`'s README
opens with *"The source catalog boundary is `cloud-itonami/org-unspsc`; this
repository owns the executable commodity-organism fleet."*

`kotoba-lang/unspsc` overlaps on grain (both work at the segment level) but not on
subject: it maps a segment to the *capabilities a business needs*, whereas this
repository carries the segment's *identity* (code, slug, canonical name, CPC
concordance). If you want to know what segment 43 **is**, read here; if you want
to know what running a segment-43 business **requires**, read there.

Concordance to the UN Central Product Classification is the neighbour
`cloud-itonami/cpc`.

## What is actually in here

```
segments.csv                  50 rows — code, slug, name.  The only taxonomy data.
kotoba/src/types.ts           SegmentDef record shape + isValidCode / isValidSlug / cpcSectionFor
kotoba/src/seed.ts            segments.csv → AT Protocol records via @etzhayyim/sdk
kotoba/src/query.ts           read side (key-prefix MST traversal)
kotoba/src/verify.ts          Merkle-proof verification example
kotoba/src/{types,seed}.test.ts   vitest — 56 passing cases + 12 that cannot load (below)
appview/worker/               Cloudflare Worker that proxies com.etzhayyim.apps.unispsc.* XRPC
training/                     13 JSONL examples + a LoRA config for a component generator
260326-unspsc-*.md            two design notes from 2026-03-26 (DID scheme, okaimono EC)
AGENTS.md                     the full four-level model this repo is *aimed* at
migration.edn                 provenance of the extraction from etzhayyim/root
```

**Segments only.** `AGENTS.md` describes families, classes and ~70,000
commodities, and `training/` was built for generating per-commodity components.
None of that data is in this repository. `segments.csv` is the whole corpus.

## What runs today, and what does not

Measured on 2026-08-15 with node v26.3.0 / npm 11.16.0. See
[`docs/operator-quickstart.md`](docs/operator-quickstart.md) for the commands.

| | Status |
|---|---|
| `kotoba/src/types.test.ts` — 56 pure cases | **passes**, needs no install |
| CPC-coverage check over `segments.csv` | **runs** on node's native TypeScript stripping |
| `npm install` in `kotoba/` | **fails** — `EALLOWSCRIPTS`, npm refuses to run the git dependency's prepare script in a project-scoped install |
| `kotoba/src/seed.test.ts` — 12 declared cases | **cannot load** — imports `seed.ts`, which imports `@etzhayyim/sdk` at module top level |
| `npm run typecheck`, `seed`, `query`, `verify` | blocked by the same install failure |
| `appview/worker` | `@etzhayyim/kotodama-host-sdk` is declared `workspace:*`; there is no workspace root in this repository, so it cannot resolve |

Both test files say *"No SDK / network"* in their docstrings, and both are true in
spirit — but `seed.test.ts` reaches its pure functions through a module that
constructs an SDK client at import time, so it inherits the dependency anyway.

### Known post-extraction breakage

This repository was carved out of `etzhayyim/root` (`migration.edn` records the
source tree). Three references did not survive the move, and none of them are
fixed here — they are recorded so the next reader does not re-diagnose them:

1. **Dangling symlinks.** `kotoba/CHARTER-RIDER.md` → `../../../../CHARTER-RIDER.md`
   and `appview/worker/CHARTER-RIDER.md` → `../../../CHARTER-RIDER.md` resolve to
   nothing. Both `NOTICE` files say the licence terms are "see CHARTER-RIDER.md",
   so **the stated licence rider is currently unreadable from this repository.**
2. **Dead hosts.** `unispsc.etzhayyim.com` (the Worker's custom domain) and
   `lg-open-unispsc.etzhayyim.com` (its upstream) are both NXDOMAIN.
   `pds.etzhayyim.com`, the seeder's default write target, resolves but answers
   HTTP 530. Nothing described in `appview/` is reachable.
3. **Renamed dependency.** `kotoba/package.json` pins
   `git+https://github.com/etzhayyim/com-etzhayyim-sdk.git#12314a0c`. That
   repository is now `kotoba-lang/sdk`; the pinned commit still exists there
   (2026-07-17) and GitHub's redirect keeps the URL working, so this is stale
   rather than broken.

### Identity drift

The repository lives at `cloud-itonami/org-unspsc`. Its contents still name the
pre-move identity, in three different spellings:

- `README.edn` — `com-etzhayyim-app-open-unspsc`
- `AGENTS.md` — `etzhayyim-project-open-unispsc`, and DIDs under
  `did:web:unispsc.etzhayyim.com`
- lexicon NSIDs — `com.etzhayyim.apps.unispsc.*` (AGENTS.md) versus
  `com.etzhayyim.apps.openUnispsc.*` (the code)

`cloud-itonami/unspsc` states the policy for this class of drift: *"Historical
commodity DIDs and `com.etzhayyim.*` namespaces remain compatibility
identities."* They are kept, not renamed. Note the two spellings **unspsc** and
**unispsc** are both live and are not interchangeable — the code uses
`openUnispsc`.

## Provenance of `segments.csv` — unverified

`segments.csv` carries 50 rows with no duplicate codes. There is **no record in
this repository of where those 50 rows came from**, and `types.ts` dates them to a
"UNSPSC v25 baseline" (`2023-08-15`) without a citation.

The sibling `kotoba-lang/unspsc` is explicit that its own segment list is "drawn
from general knowledge of the UNSPSC segment taxonomy structure, **not copied from
a verified official UNSPSC codeset file**". Nothing here shows this table has a
better pedigree. Treat the codes and titles as unverified against
[unspsc.org](https://unspsc.org/) until someone diffs them against an official
codeset release and records the result.

One consequence is already visible: three of the 50 segments — **32, 49 and 54** —
fall outside every range in `cpcSectionFor()`, so they publish with no CPC
concordance. Whether that is a gap in the mapping or a sign that those codes do
not belong in the table is exactly the question the missing provenance would
answer.

## Decisions

Recorded in [`docs/adr/`](docs/adr/) as EDN transaction data:

- `2608150100` — this repository is the source-catalog plane; the fleet is not here
- `2608150200` — the pure-helper suite is the only self-contained check, and the
  SDK-dependent paths are documented as blocked rather than left to fail silently

## Licence

Apache-2.0 with the etzhayyim Charter Compliance Rider v3.1 — per the `NOTICE`
files, whose text is currently unreachable (see breakage 1 above).
