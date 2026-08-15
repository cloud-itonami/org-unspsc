# Operator quickstart

Everything below was walked end to end on 2026-08-15 from a fresh clone, on
macOS 25.3 with **node v26.3.0 / npm 11.16.0**. The observed output is quoted, so
you can tell a broken checkout from a broken machine.

The point of this page is that **you can check this repository's logic without
installing anything.** Node 26 strips TypeScript types natively, and `npx` fetches
the test runner on demand. The parts that *do* need an install are blocked today
(step 4) — that is a known state, not something you misconfigured.

## 0. Get it

```sh
git clone https://github.com/cloud-itonami/org-unspsc.git
cd org-unspsc
```

Inside the west superproject it is already at `orgs/cloud-itonami/org-unspsc`;
note that the git remote there is named `cloud-itonami`, not `origin`.

## 1. Run the pure test suite — no install

```sh
cd kotoba
npx --yes vitest@4 run src/types.test.ts
```

```
 Test Files  1 passed (1)
      Tests  56 passed (56)
```

Those 56 cases are the repository's actual contract: the 2-digit code format, the
slug grammar, and every boundary of the UNSPSC-segment → CPC-section mapping.
If this is green, `segments.csv` can be validated and converted.

The first `npx` invocation on a machine downloads vitest and takes noticeably
longer; once it is cached the run itself is under half a second.

## 2. Check the CPC concordance coverage of the real data

`cpcSectionFor()` maps segment-code *ranges* to CPC sections, so a code that falls
between two ranges silently publishes without a concordance. This asks the actual
table how many do:

```sh
node --input-type=module -e '
import { readFileSync } from "node:fs";
import { cpcSectionFor } from "./src/types.ts";
const rows = readFileSync("../segments.csv", "utf8").trim().split("\n").slice(1);
const missing = rows.map(l => l.split(",")[0]).filter(c => !cpcSectionFor(c));
console.log(`segments=${rows.length} without-cpc=${missing.length} codes=${missing.join(",")}`);
'
```

```
segments=50 without-cpc=3 codes=32,49,54
```

Run from `kotoba/`. No install, no `tsx` — node imports the `.ts` file directly.

**Three of fifty is the current state, not a failure.** It is the open question
described under "Provenance" in the [README](../README.md): either the range table
is incomplete, or codes 32 / 49 / 54 do not belong in an official segment list.
Do not "fix" it by widening the ranges until the codeset provenance is settled —
that would encode a guess as a concordance.

## 3. Read the taxonomy

```sh
head -4 ../segments.csv
```

```
code,slug,name
10,live-animals,Live Animals and Livestock and Agricultural Products
11,mineral-textile,Mineral and Textile and Inedible Plant and Animal Materials
12,chemicals,Chemicals including Bio Chemicals and Gas Materials
```

50 rows, no duplicate codes, no commas inside any name. (`seed.ts` has a branch
for commas inside the name field; no row in the shipped data exercises it — only
the synthetic case in `seed.test.ts` does.)

## 4. What you cannot do yet, and the exact error you will get

Do not spend time on these; they are the repository's known blocked paths.

### `npm install` fails

```sh
npm install --no-audit --no-fund
```

```
npm error git dep preparation failed
npm error npm error code EALLOWSCRIPTS
npm error npm error --allow-scripts is not allowed in project-scoped installs.
```

`@etzhayyim/sdk` is a git dependency that publishes `dist/`, so npm must run its
prepare script to build it. npm 11.16 refuses to do that in a project-scoped
install. `--ignore-scripts` does not help — npm re-enters with `--force` for the
git-dep preparation either way, and trips the same check. It leaves an **empty**
`node_modules/` behind; delete it, because `.gitignore` here covers only
`training/adapters/*.safetensors` and would otherwise let it be committed.

Everything downstream is blocked by this: `npm run typecheck`, `seed`, `query`,
`verify`.

### The second test file cannot load

```sh
npx --yes vitest@4 run          # the whole suite
```

```
 FAIL  src/seed.test.ts
 Error: Cannot find package '@etzhayyim/sdk' imported from .../kotoba/src/seed.ts

 Test Files  1 failed | 1 passed (2)
      Tests  56 passed (56)
```

Its 12 declared cases never run. `seed.test.ts` only exercises pure functions,
but reaches them through `seed.ts`,
which constructs an SDK client at module scope. Run `src/types.test.ts` by name
(step 1) to get a clean exit.

### Nothing in `appview/` is reachable

`unispsc.etzhayyim.com` (the Worker's route) and `lg-open-unispsc.etzhayyim.com`
(its upstream) are both NXDOMAIN. `pds.etzhayyim.com`, the seeder's default write
target, resolves but answers HTTP 530. The Worker also declares
`@etzhayyim/kotodama-host-sdk` as `workspace:*` with no workspace root in this
repository. Deploying it is not a matter of running `wrangler deploy`.

## 5. If you are here to unblock it

In dependency order — each step is a precondition of the next:

1. **Re-point the SDK dependency.** `etzhayyim/com-etzhayyim-sdk` is now
   `kotoba-lang/sdk`; the pinned commit `12314a0c` still exists there. Depending on
   a built artifact instead of a git source removes the `EALLOWSCRIPTS` wall
   entirely.
2. **Move the SDK client out of module scope in `seed.ts`** so its pure exports
   load without it. That alone makes `seed.test.ts` runnable and the whole suite
   green — worth doing regardless of step 1.
3. **Settle the codeset provenance** (README, "Provenance"). Until then the three
   uncovered segments cannot be resolved either way.
4. **Restore or drop the charter symlinks.** Both `CHARTER-RIDER.md` symlinks
   dangle, so the licence terms the `NOTICE` files point at cannot be read.
5. **Decide whether `appview/` still has a target.** Its hosts no longer exist.

Steps 1, 2 and 4 are self-contained. Step 3 needs an authoritative UNSPSC codeset
release. Step 5 is a product decision, not a repair.
