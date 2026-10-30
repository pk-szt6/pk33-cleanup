# PK Rework Assessment #3 — 30 Oct, 01:34 hrs

**Host:** pk33 · SSD CT1000T500 1 TB (915 G usable) at `/` · HDD ST1000DM003 1 TB at `/mnt/hdd1t` · 32 cores
**Purpose:** establish the start point for (a) recording relocation + destructive changes, (b) freeing CPU and storage, (c) identifying finalised vs immediate workings across zsm / zcontent / zncert / content-engine / artefacts, (d) cleaning dirty git trees.
**Scope:** assessment and discussion only. **No changes made.**

**Preceding documents:** `pk-load-audit-1-30oct0134hrs.md` (CPU/load), `pk-prune-assessment-2-30oct0134hrs.md` (storage prune/shift).

---

## 0. Executive summary

Five findings, in order of how much they change the plan:

1. **The cron-recording mechanism already exists and is already the right one.** `scripts/pcl_selfcron.sh` in efus3 has a `disable` verb that removes an entry from the live crontab **and preserves the registry row** with `state=DISABLED`. You do not need to build a recording system. You need to *use* the one that is there, and stop bypassing it.
2. **The real CPU hog is not what audit #1 said — it is the knowledge-graph rebuild, and it is thermally dangerous.** Right now `graphify` is pinning a core at 100% and the CPU package reads **89–100 °C** (critical = 100 °C). It is graphifying the **Flutter SDK**. This was paused on 2026-10-14 for exactly this reason and the pause was later **reverted without the thermal reasoning being re-examined**.
3. **zsm1 is FINALISED and GREEN. The `RESUME-HERE.md` doc is two months stale and understates it.** Indianization reads 0 faults (doc says 130 RED), depth 1961/1961, boundary 4/4. Git tree is clean. Do not "resume" zsm1 — it was deliberately closed down and committed clean.
4. **The dirty-git-tree problem is ~70% cron churn, not work.** ~250 of ~900 dirty files are machine-generated log output that the crons rewrite every cycle. Pausing the crons dissolves most of it. The genuine uncommitted work is small and identifiable.
5. **The knowledge-graph scope list contains literal trash and a duplicate-heavy 11× `claude-memory` entry.** 60 scopes → 49 unique, of which 15 are junk. The KG is spending a full core to produce **45 REDs out of 48** on every pass.

**Recommended start point: Phase 0 (record) → Phase 1 (pause the KG family) → Phase 2 (prune KG scopes).** All three are safe, reversible, and nothing there destroys data. Phase 1 alone stops the thermal problem.

---

## 1. The recording mechanism already exists

### 1.1 What `pcl_selfcron.sh` provides

`/home/pk33/zexp1/efus3/scripts/pcl_selfcron.sh` — a machine-level cron manager with a **registry at `_logs/agent-crons/registry.tsv`**.

Registry columns (13):

    machine  name  schedule  command  purpose  installed_by  installed_at  state
    aka  charter  logic  owner  trigger_kind

Verbs:

    install <name> <schedule> <command> <purpose>
    disable <name>      # remove from crontab, KEEP the registry row as DISABLED
    enable  <name>      # state=ACTIVE, reinstall from registry data
    list                # registered vs actual crontab -l
    verify              # COMPUTE drift (registered but missing = dead)
    heal                # reinstall every dead-but-registered entry
    discover            # backfill registry from real crontab -l
    remove <name>

**`cmd_disable` (line 296) is exactly the archive-of-record you asked for.** It:

1. verifies the entry exists for this machine,
2. calls `crontab_write "tag:${TAG_PREFIX}${name}" _ct_drop_tag ...` — dropping every live line carrying `# pcl-selfcron:<name>`,
3. rewrites the registry row's `state` field to `DISABLED` via a careful Python TSV pass (with an explicit comment warning *never* to use `csv.writer`, because `QUOTE_MINIMAL` would corrupt every row containing a double-quoted cron command),
4. **leaves the row in place** — with its schedule, command, purpose, installed_by, installed_at intact.

So a disabled entry keeps its provenance. `enable` can bring it back from registry data alone, without re-deriving the command.

### 1.2 The write-guard is real, and it is paranoid for good reason

`crontab_write` (line 48) serialises every mutation under `flock`, re-reads the crontab **under the lock**, and **refuses** any write that drops more effective lines than the caller declared:

    REFUSED crontab write on pk33: effective lines 99 -> 2 (allowed drop 0).
    A wipe looks exactly like this; nothing written.
    SELFCRON_ALLOW_SHRINK=1 overrides for a deliberate mass removal.

The rationale is documented in the file header, and it is a post-mortem:

> Two unguarded read-modify-writes ran 3 s apart (a raw `crontab /tmp/file` from a T-21 subagent and this script's `install zk-embed`); the crontab went from 70 effective lines to 2, taking the warden/heal cron with it, so nothing self-healed for 41 min.

Every write also drops a timestamped backup into `_logs/agent-crons/crontab-backups/`, keeping the **40 most recent**.

**Implication for the mass pause:** `disable` uses `max_drop=tag:<name>`, so it is bounded per-entry and safe. But a bulk pause of ~90 entries is a **mass removal** and will trip the shrink guard unless each is disabled individually (tag-scoped, so each is fine) — the guard is per-call, not per-session. Disabling one at a time via `disable` is the intended path and will not trip it.

### 1.3 Current registry state

Per-machine row counts:

| machine | rows |
|---|---|
| pk33 | **103** |
| pk | 60 |
| pk2 | 3 |
| pk32 | 2 |
| pk31 | 1 |

State totals across all machines:

| state | count |
|---|---|
| ACTIVE | 137 |
| DISABLED | 20 |
| DEAD-UNTRIAGED | 8 |
| PAUSED (pk32 being shut down, fleet law; re-pausing on expiry) | 2 |
| SUPERSEDED (zrec:seats-alive) | 1 |

**Live crontab:** 99 effective lines. `pcl_selfcron.sh verify` reports:

    103 registered, 3 dead
      DEAD  zk-graph               (registered 2026-09-16T00:03:21Z, missing from crontab -l NOW)
      DEAD  zk-index-zdash222      (registered 2026-09-16T07:34:12Z, missing from crontab -l NOW)
      DEAD  zdash222-wifi-5g-guard (registered 2026-09-19T02:33:12Z, missing from crontab -l NOW)

All 8 `DEAD-UNTRIAGED` rows belong to machine **`pk`** — a hostname you no longer have. The `pk` block is registry pollution from a prior machine identity.

### 1.4 There is already a pause-marker convention

Your crontab contains 5 lines commented with a reason marker:

    #PAUSED-BY-ADAL-GRAPHIFY:  41 */6 * * * ... bin/zk_graph.sh ...              # pcl-selfcron:zk-graph
    #PAUSED-BY-ADAL-GRAPHIFY:  */10 * * * * ... zskill zk index --if-stale ...   # pcl-selfcron:zk-index
    #PAUSED-BY-ADAL-GRAPHIFY:  23 * * * * ... zskill zk embed --batch 128 ...     # pcl-selfcron:zk-embed
    #PAUSED-BY-ADAL-GRAPHIFY:  4,14,24,34,44,54 * * * * ... zk index --scope ... # pcl-selfcron:zk-index-zdash222
    #PAUSED-BY-ADAL-NONROOT-WIFI: */5 * * * * ... bin/zwifi_5g_guard.sh ...       # pcl-selfcron:zdash222-wifi-5g-guard

**"Pause with a named reason" is established practice on this box.** The reason is recorded in the line itself, and `pcl_selfcron.sh disable` records it a second time in the registry `state` field. Use the script, and the registry becomes the index.

### 1.5 A trap to know before you bulk-disable

`_ct_drop_tag` uses `grep -vF -- "$1"` — a **fixed-string** match, so the drop tag `# pcl-selfcron:zk-index` is a **substring** of `# pcl-selfcron:zk-index-zdash222`.

Verified against the live crontab:

    drop tag = "# pcl-selfcron:zk-index"
      allowed_drop would be: 3
      [comment(not effective)] # pcl-selfcron:zk-index
      [comment(not effective)] # pcl-selfcron:zk-index-zdash222   <- substring collision
      [EFFECTIVE]              # pcl-selfcron:zk-index

Today this is harmless (the colliding line is already a comment, and `zk-index-zdash222` is independently DEAD). But **if you `disable zk-index-zdash222` while `zk-index` is live**, dropping `zk-index` would silently take `zk-index-zdash222` with it. The `allowed_drop` count is computed with the same `grep -cF`, so the shrink guard would *permit* it. Order the disables longest-tag-first, or verify after each.

### 1.6 What the registry still needs

- **60 stale `pk` rows** — wrong hostname. Re-home to `pk33` if the jobs still exist there, or mark SUPERSEDED. Do not leave them ACTIVE-and-absent; they will make `verify` noisy on every other machine.
- **8 `DEAD-UNTRIAGED`** rows, all on `pk`.
- The registry has **no `reason` column for a disable**. The `purpose` and `charter` fields are the closest thing. If you want the *why* of each pause preserved, either append the reason to `purpose` (which is the field `disable` leaves untouched) or keep the ledger file as the index.

---

## 2. The CPU hog: the KG rebuild, and a thermal problem

### 2.1 Live evidence (captured during this assessment)

    /opt/pipx/venvs/graphifyy/bin/python /usr/local/bin/graphify update /home/pk33/fvm/versions/3.24.5

Parent chain:

    bash bin/zk_graph.sh --missing
    bash bin/zk_graph.sh --missing
    /bin/sh -c cd /home/pk33/zdash22x/zdash222 && .../zskill zk index --if-stale ...; bin/zk_graph.sh --missing ...
    /usr/sbin/CRON -f -P

And later in the same session, a second one:

    graphify update /home/pk33/zexp1/efus3-demoguest-merge
      PID 1633847  RES 3.3g (10.9% of RAM)  %CPU 100.0  TIME+ 3:07.35
      PID 1672610  RES 87 MB              %CPU  90.9  TIME+ 0:21.20

### 2.2 The thermal finding — this is the urgent part

    Package id 0: +100.0°C  (high = +80.0°C, crit = +100.0°C)   <- sampled during a graphify run
    Package id 0:  +89.0°C  (high = +80.0°C, crit = +100.0°C)   <- sampled again

The package is running **9–20 °C above its designated high threshold** and has touched the critical limit. This is a hardware-protection concern, not merely an efficiency one.

### 2.3 The pause already happened once, and was reverted

`/home/pk33/GRAPHIFY-PAUSED-2026-10-14.md` — a record from **2026-10-14** (16 days ago), written by hand:

> Paused by request to stop all graphify work (**it was pinning a core at ~100% and driving the CPU to 97-99C**).
>
> The 4 lines are: `bin/zk_graph.sh` (every 6h at :41) → graphify update; `zskill zk index --if-stale` + `zk_graph.sh --missing` (every 10 min) **← the CPU hog**; `zskill zk embed --batch 128` (hourly at :23); `zskill zk index --scope zdash222` (every 10 min at 4,14,24,34,44,54)
>
> Backup of the original crontab: `/home/pk33/crontab.bak-before-graphify-pause.20261014-122945`

Two backup files exist:

    18193  Oct 14 12:29  /home/pk33/crontab.bak-before-graphify-pause.20261014-122945
    18193  Oct 14 12:30  /home/pk33/crontab.bak-before-graphify-pause.20261014-123038

**Today the `zk-index` line is live again.** The comment markers remain on 4 lines, but an *uncommented* `zk-index` line is active (crontab line 104) — so the pause was partially reverted, and the copy that is actually running is the one that was commented out.

This is the single most important thing in this document: **the same thermal event was diagnosed and mitigated 16 days ago, and the mitigation was undone.** The rework you're planning will re-run this workload heavily. Fix it as a *policy* (see §2.7), not a one-off.

### 2.4 The KG scope list contains trash and an SDK

`_logs/zk/scopes.tsv` is generated by `bin/zk scopes` from `docs/registers/clones.tsv`. It holds **60 rows / 49 unique scope names**. The junk:

    3.24.5            /home/pk33/fvm/versions/3.24.5              <- the Flutter SDK, 2.2 GB
    zt331-not-worthy  /home/pk33/trash/zt331-not-worthy          <- trash
    x0-pk-only        /home/pk33/trash/x001-trash-copy/x0-pk-only  <- trash
    claude-memory     /home/pk33/.claude/projects/-home-pk33--secrets3/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-sysnotifier/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-tzerp/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-zdash22x-zcontent1/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-zdash22x-zdash222/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-zexp1-efus3/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-zexp1-twdpt1/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33-zexp1-zvec1/memory
    claude-memory     /home/pk33/.claude/projects/-home-pk33/memory
    claude-memory     /home/pk33/.claude/projects/-tmp-claude-1000--home-pk33--secrets3-.../memory
    claude-memory     /home/pk33/.claude/projects/-tmp-claude-1000--home-pk33-zdash22x-zdash222-.../memory
    claude-memory     /home/pk33/.claude/projects/-tmp/memory

**15 of 60 scope rows (25%) are junk**, including:

- **a Flutter SDK** — 2.2 GB of vendor Dart source being AST-parsed to build a code graph. It produced **230 MB** of `graphify-out` (134 MB of it just AST cache) and is the scope I caught burning 100% CPU.
- **two directories under `~/trash/`** — literally discarded content.
- **`claude-memory` repeated 11×** under the same scope name. The scope *name* is the KG's identity key; 11 rows with the same name and different paths is a namespace collision, and it means the KG is indexing the same conceptual scope many times over.

### 2.5 The KG never goes green — 45 REDs out of 48, every run

`_logs/zk/graph-cron.log`, last 20 runs:

    ZK GRAPH: ok=3 red=45 secs=1324
    ZK GRAPH: ok=3 red=45 secs=689
    ZK GRAPH: ok=3 red=45 secs=1514
    ZK GRAPH: ok=3 red=45 secs=667
    ZK GRAPH: ok=3 red=45 secs=656
    ZK GRAPH: ok=3 red=45 secs=661
    ZK GRAPH: ok=3 red=45 secs=660

**Average pass: 738 seconds (12.3 minutes) — against a 10-minute cron.** The job overlaps itself perpetually. And it is RED on 45 of 48 scopes *every single time*, meaning the work never succeeds: `graphify.log` shows a stream of `RED <scope> 0s` / `RED <scope> 223s` lines.

This is the definition of a job that burns a core indefinitely and delivers nothing. **This is the CPU free you asked for, and it is self-inflicted.**

### 2.6 The graphify task has bitten before

Two commits in efus3 history are directly about this:

**`fd5fa4a83`** — 2026-09-16, by pk-szt6:

> zfts: prune graphify-out/ from the walk (**17,701 derived graph files had entered the zk index**)
> Realizes: A-15

The KG index was ingesting its own derived output — a feedback loop. The fix added `graphify-out` to the walk exclusion list in `scripts/zfts.py:43` (still in place).

**`bc13b9fc5`** — the selfcron runaway:

> fix(selfcron): STOP a live runaway — **heal was pumping the crontab 1294 lines and climbing**
>
> THE MOTOR, measured not inferred: 17 registry rows carried `sched="#PAUSED20260711 47 0 * *"` — a schedule cron cannot parse. `tagged_line` writes it verbatim, so each "healed" entry landed as a COMMENT: never runs, stays DEAD forever, gets appended again next tick. `install` checks for an existing entry; `heal` did not. `warden.sh` calls heal every 5 minutes.
>
> MEASURED: one heal run appended 13 lines. 288 runs/day = ~3,700 lines/day. History: 120 lines (20 Jul) → 137 (25 Jul) → **1,294** today.
>
> A repair verb that cannot recognise its own previous repair is not a repair verb, it is a pump.
>
> CRONTAB COLLAPSED: 1,294 → 148 lines. 1,144 EXACT duplicate comment lines dropped.

So: the cron system has already had a *cascading* failure here, and the graphify family is the one that caused it. This is not theoretical risk.

### 2.7 Recommendation

**Disable the whole zk/graphify family** (7 registry rows):

| machine | name | schedule | what it does |
|---|---|---|---|
| pk33 | zk-graph | 41 */6 * * * | graphify update |
| pk33 | zk-index | */10 * * * * | the CPU hog |
| pk33 | zk-index-zdash222 | 4,14,24,34,44,54 * * * * | zk index --scope |
| pk33 | zk-embed | 23 * * * * | zskill zk embed --batch 128 |
| pk33 | zk-embed-tunnel | */5 * * * * | zk_tunnel.sh |
| pk33 | zk-report | 3 * * * * | zk_report.sh |
| pk | zkos-embed-backfill | 17 */2 * * * | stale `pk` row |

Note `zk-embed-tunnel` is a *tunnel* (connectivity probe for the embedding service) — arguably keep it. `zk-report` is a *report* over the index — if the index is stale, the report is stale. Decide per-entry.

**The index is regenerable.** `zk embed` ran in **125.1 seconds** and produced `docs=35919 chunks=737128`. That is a cheap rebuild compared to a 12-minute graphify loop that never goes green.

**Also consider:** add a thermal guard, since the failure mode (pause → revert → thermal limit) has now happened once. A cron that checks `sensors` and pauses graphify above ~85 °C would close the loop.

---

## 3. Finalised vs live: zsm / zcontent / zncert / content-engine / artefacts

This is the section that most changes the plan, because **the assumption that everything is mid-flight is wrong for zsm1**.

### 3.1 zsm1 — FINALISED, GREEN, CLEAN ✅

Location: `/home/pk33/zexp1/efus3/zsm1` (563 MB) — the SSOT per its own README.

**Git:** clean (0 dirty files). HEAD:

    311036bb close-down sweep 2026-09-18 (trunk2): generated engine/export state committed
               so the submodule is clean for the rest

That commit message is a deliberate close-down. **zsm1 was intentionally closed and committed clean.** It is not in-flight work.

**Its own README states the SSOT position:**

> **SSOT notice (2026-09-16, seat zdash22v8-zsm, pk order)**
> This repository (github `zexp1/zsm1`, checked out in-tree at `efus3/zsm1`) is the SINGLE SOURCE OF TRUTH for zsm study content. It is a fast-forward superset of `tzerp/zsm1-tzerp` (now a read-only archive) plus the NCERT originals under `ncert1-originals/`. Git keeps TEXT only; big derived artefacts (sqlite/ndjson/zip/mp4) live on R2 or in `_archive` with a manifest.

**Re-ran the checkers rather than trusting the docs.** `RESUME-HERE.md` is titled *"emergency stop, 2026-08-10"* and is **two months stale**. It says:

> *Re-run the checkers before trusting any number here.*

So I did:

| Arc | RESUME-HERE (Aug 10) | **Measured now (Oct 30)** | Checker |
|---|---|---|---|
| Depth conveyor | DONE 1961/1961 | ✅ **1961/1961 (100.0%)** — 0 KCs under target on all 3 metrics | `tools/kcgraph/depth_status.py` |
| Indianization | **RED** — 130 faults / 54 strands | ✅ **GREEN — 0 faults across 30,164 artefacts, 149 strands** (987 excluded as legitimately world-scoped) | `tools/kcgraph/indianization_status.py` |
| Hindi debt | **RED** — 5,589 items + 589 wex | 🟡 **144/145 strands GREEN (99.3%)** cards · T3 items 144/144 (100%) · T4 wex 144/144 (100%) · T2 ui 140/140 · **G6-100 still RED** | `tools/kcgraph/g6_hindi_status.py` |
| zsm\|zncert boundary | GREEN | ✅ **GREEN** — 4 laws PASS (L1 no lens vocab in truth, L2 one-way refs, L3 lens can't write truth, L4 namespace) | `tools/kcgraph/boundary_gate.py` |
| Test suite | GREEN 945 passed | not re-run (out of scope for a disk audit) | `pnpm test` |
| ZNCERT Stage 3 | open — 143-topic roadmap | still open — see §3.4 | `docs/m216-corpus-ingestion/zncert-sync/STAGE3-PLAN.md` |
| Media/sims/Blender | FROZEN | FROZEN per pk | — |

**Read the RESUME-HERE lesson section — it is directly relevant to this rework:**

> Four separate bugs today all presented as success: drivers cached by strand name and "completed" without authoring; an invalid model identifier failed 111 calls silently; colliding ids discarded independently verified content; a rejected artefact permanently blocked its own replacement; and a duplicate detector stripped Devanagari so every Hindi artefact after the first looked like a duplicate.
>
> **the general guard is cheaper than any tool: when a round applies less than expected, verify the mechanism before believing the number.**

**Verdict: zsm1 is DONE. Protect it, don't rework it. Its 563 MB is mostly `docs/m216-corpus-ingestion` (141 MB) — source material, not bloat.**

### 3.2 zcontent1 — LIVE, mid-flight, and this is where the rework lives ✅

Location: `/home/pk33/zdash22x/zcontent1` (2.3 GB). Branch `feat/content-consolidation`.

HEAD: `ae50b01 docs: acknowledge secrets-access law (zsecret_get.sh only, 08:02Z token exposure)`

**This is the live work.** The README records the pk ruling verbatim:

> **pk ruling (2026-09-19 16:5xZ, verbatim — ORIENT THE WHOLE WORKING AROUND THIS):** "for completeness sake — do the KLO too for zsm and zncert universe, may be same number of classes, for all grades. This way we shall have much enforcing understanding for the whole hlp build-up … orient the whole working around this." **From now: two parallel KLO universes on the same grade ladder gK–g12 — `zsm-KLO` and `zncert-KLO` — every HLP build-up hangs off a KLO in one universe, cross-linked where they coincide.**

**Design is done, build is not:**

    16,326 bytes  docs/DESIGN-TWO-UNIVERSES.md     (Sep 19 17:08)
    17,175 bytes  docs/KLO-UNIVERSE-MATRIX.tsv      (Sep 20 08:23)

The matrix still carries **gap reasons**:

    g1  Foundational  (no subject)      zncert  0 klos   gap_reason: no-pattern-match (135 class-1/2 chapters in GAPS)
    g1  Foundational  arts              zsm    10 klos   gap_reason: some/all KLOs unplaced in year_plan
    g1  Foundational  computational..   zsm     8 klos   gap_reason: some/all KLOs unplaced in year_plan

Supporting artefacts present: `KLO-LEDGER.tsv`, `KLO-CARD-MAP.tsv`, `KLO-LINK.tsv`, `CURRICULUM-SPLIT.tsv`, `DISAGREEMENTS.tsv` + `DISAGREEMENTS-ALLOWLIST.tsv`, `ITEM-DUPLICATES.tsv`, `KC-ASSIGNMENT-COVERAGE.tsv`, `FOUR-BOXES.md`, `METADATA-CARDS-PROVENANCE.tsv`, `METADATA-ITEMS-PROVENANCE.tsv`, `OPEN-QUESTIONS.tsv`, `RECEIPTS-INDEX.tsv`.

**Four-box orientation** (`docs/FOUR-BOXES.md`): `zsm-engine` × `zsm-curriculum` × `zncert-engine` × `zncert-curriculum`.

**Verdict: LIVE — this is the rework target. Highest protection priority. 2.3 GB, mostly `zsm1/export` (566 MB), `zsm1/docs` (257 MB), `.git/modules` (798 MB).**

### 3.3 zncert-w1 — LIVE, published, Hindi-blocked

Location: `/home/pk33/zncert-w1` (17 GB). Branch `main`, git clean.

HEAD: `6c1f457 Handover note for zdash223` (preceded by `49efcfc Record the two views and version stamping`, `f6192aa v2-w1-p1-s1: gate built and verified; corpus untracked`).

Working id `v2-w1-p1-s1`, edition `v2-2026-10-01` (rationalised).

**Current state (from its own README):**

| | |
|---|---|
| Corpus local | 2,569 PDFs / 45,622 pages / 14 GB, on pk31 and pk33 |
| English | 470 chapters built, 8,299 pages — **published first** |
| Hindi | 505 chapters, 7,547 pages — replicas only, text **not** trustworthy |
| Chapters labelled end-to-end | 3 in progress (Class 12 Physics, Accountancy, Economics) |
| Portal | zncert-w1.web.app and zncert2.web.app — both blank, history purged |
| Metadata | Supabase Postgres, 2,508 assets / 243 books, session-gated |

**Live product surface (from `docs/HANDOVER.md`):**

- Gate: Supabase Edge Function `gate`, HTTP-only cookie, bcrypt via `zn_login()` — address by SLUG not name; 401 without cookie, 200 with; **243 books, 133 published assets**
- Passkey `PkyAwAks@21822192`, hash only in Postgres `access_keys`
- **7 Cloudflare Pages sites, all gated**: zncert2-metadata, zncert2-w1, zncert, zncert1, zncert3, zncert4, zncert5
- Deploy: `CLOUDFLARE_API_TOKEN=<scoped pages token> wrangler pages deploy <dir> --project-name <p> --branch main`

**Its five ground rules, which matter for how you handle it:**

1. Exact NCERT content, or nothing.
2. **A wrong text layer is worse than no text layer.**
3. Never claim verification you did not perform.
4. Secrets never enter the repo.
5. Hindi is not the same problem as English — English first, deliberately.

**Structural note:** `.gitignore` puts `library/`, `out/`, `ocr-out/`, `content-staged/`, `logs/`, `work/` out of git — "bulk data: lives on the workers, never in git". So:

- `library/` (14 GB) = **SOURCE** — 2,569 downloaded PDFs, the input to everything
- `out/` (1.3 GB, 976 dirs) = **DERIVED** — rendered labelled HTML, rebuildable via `bin/zw1-build`
- `.venv/` (1.4 GB) = toolchain

**Verdict: LIVE and PUBLISHED. The 14 GB library is the one asset in this whole assessment that is genuinely irreplaceable-in-place.**

### 3.4 ZNCERT Stage 3 — OPEN, and it is the bridge the KG rework depends on

`zsm1/docs/m216-corpus-ingestion/zncert-sync/STAGE3-PLAN.md` (Sep 16):

> *Opened 2026-08-06, after Stage 2 made the library bilingual. Boundary: zncert is the LENS (NCERT chapter → zsm KC). It reads the library read-only and never edits a strand.*

**The real denominator (computed, not inherited):**

| population | rows |
|---|---|
| all rows in the mapping corpus | 8,147 |
| parse noise (no-letters / LaTeX fragments / too-short titles — 143 of them `computer_science` g11, the known `kecs105.md` fragmented-file bug) | **152 (1.9%)** |
| **genuine chapter rows — the denominator for "complete"** | **7,995** |

Script split: latin 3,145 · devanagari 2,347 · arabic/urdu 2,541 · other 114.
**Verdicts today: CLEAN 2,144 · AMBIGUOUS 5,786 · NO-ZSM-STRAND-YET 217.**

Two matching routes are live: Hindi-native (`tools/zncert/match_hindi_chapters.py`) and a Latin route. Stage 2 changed the input — **2,302 of 2,347 Devanagari rows already carry a strand candidate**; they need title-level matching, not strand discovery.

**Verdict: OPEN. This is the zsm↔zncert join, and it is the substance of the KG rework.**

### 3.5 The summary table

| Component | Status | Evidence | Protect? |
|---|---|---|---|
| **zsm1** (`efus3/zsm1`, 563 M) | **FINALISED / GREEN** | depth 1961/1961, indianization 0 faults, boundary 4/4, git clean, close-down commit | **Protect absolutely** — SSOT |
| **zcontent1** (2.3 G) | **LIVE — the rework** | pk ruling 2026-09-19, KLO matrix with open gaps | **Protect — primary target** |
| **zncert-w1** (17 G) | **LIVE / published** | 7 gated CF sites, 243 books, English shipped | **Protect — product** |
| **zncert Stage 3** | **OPEN** | 7,995 rows, 5,786 AMBIGUOUS | **This is the work** |
| **zk KG** (6.5 G) | **LIVE but pathological** | 45/48 RED every run, 738 s/pass, Flutter SDK scope, 89–100 °C | **Rework target — this is the "core knowledge graph rework"** |
| **tzerp/zsm1-tzerp** (8.5 G) | **READ-ONLY ARCHIVE** | README: "consolidated into the single source of truth github zexp1/zsm1" (2026-09-16) | Archive — worktrees prunable |
| **efus3-demoguest-merge** (11 G) | **WIP merge** | branch `zdash22v21-demoguest-merge`, untracked `skills/_meta/bin/_logs/` | **Decide: close or shift** |

---

## 4. Dirty git trees — mostly churn, not work

### 4.1 The census

    595  /home/pk33/zdash22x/zcontent1
    198  /home/pk33/zexp1/efus3
     65  /home/pk33/zdash22x/zdash222
     19  /home/pk33/zexp1/zprotocols
      6  /home/pk33/ztemp33
      5  /home/pk33/.codex/memories
      1  /home/pk33/zexp1/twdpt1
      1  /home/pk33/zexp1/eduz-claude-home
      1  /home/pk33/zdash22x/zdash223
      1  /home/pk33/zcdx1/z02-gating
      1  /home/pk33/Desktop/notespk

Total ≈ **893 dirty entries**.

### 4.2 Decomposition

**zcontent1 (595) — 592 `??` + 3 `M`:**

    562  docs/history/CURRENT/*.tsv.gz    <- timestamped snapshots, 13 MB total — CHURN
      3  _lanes/assess/  (BRIEF/OUT/pair.err)
      1  _lanes/design/pair-additive.err
    ~24  embed2_<subject>-<lang>.stdout.log / .stderr.log
      3  M  docs/CARD-SOURCE-CANDIDATES-hindi.tsv
         M  docs/CARD-SOURCE-CANDIDATES-latin.tsv
         M  docs/CURRENT.tsv

**efus3 (198) — 120 `??` + 71 `M` + 7 `D`:**

    128  _logs/_history/            <- CHURN
     13  _logs/agent-crons/crontab-backups/   <- CHURN (12 untracked + 7 deleted)
      8  _logs/zwrit/
      6  _logs/best-practices/
      5  _lanes/trunk-inbox/
      4  _logs/sweep/
    ~25  zfable/ops/pk-console/*.ndjson / *.md  <- live cron output — CHURN
      3  zfable/sau/*.tsv            <- live (sau-keeper cron)
      2  zrac-refreshed-analysis-for-completion/_computed/
      1  M zcons1                    <- submodule pointer
      1  M ssot_engine_efus3/dc/intake.ndjson
      1  M ssot_engine_efus3/rc/intake.ndjson

**zdash222 (65) — 41 `??` + 24 `M`:**

    32  _logs/gates/                <- CHURN
     6  _logs/abuse-digest/         <- CHURN
     3  docs/registers/{clones,machines,repos}.tsv  <- real
     2  docs/dod/                   <- real
     2  _logs/zk/, _logs/zgate2/    <- CHURN
     1  seats/zdash22v7-publish/REPORTS.md
     1  public/data/catalog.json
     1  docs/protocols/CLOUDFLARE-PUBLISH.ledger.tsv

**zprotocols (19) — REAL WORK, and it is cron-staged:**

    A  ctx/CONTEXT-v1.0.16.md  +  .sha256
    A  ctx/CONTEXT-v1.0.17.md  +  .sha256
    ...  through v1.0.23
    M  ctx/LATEST.md

The `zctx-refresh` cron (every 5 min) runs `zctx.sh compile && git -C .../zprotocols add ctx && git -C .../zprotocols commit && git push -q github main`. It **stages but does not commit** these 9 files. So they accumulate in the index waiting for a commit that never comes. This is a latent inconsistency, not a human's uncommitted work.

**ztemp33 (6) — REAL:** `API-MAP-dashboard.md`, `REPORT-1-dashboard.md`, `brief-cred-protocols.md`, `brief-gate-matrix.md`, `lanes/`, `launch-lanes.sh`.

**Trivial (1 each):** `twdpt1` → `attempted`; `eduz-claude-home` → `M settings/settings.json`; `zdash223` → `?? HANDOVER.md`; `zcdx1/z02-gating` → `?? docs/01-zgate-ecosystem-discovery.md`; `Desktop/notespk` → `M npk017-pkgit-scripts/syncpkall_scope.txt`.

### 4.3 The 7 deletions in efus3 — look at these before anything else

    D _logs/agent-crons/crontab-backups/crontab.pk33.20260916T060256Z.3081326.bak
    D _logs/agent-crons/crontab-backups/crontab.pk33.20260916T062039Z.3452119.bak
    D _logs/agent-crons/crontab-backups/crontab.pk33.20260916T062039Z.3452208.bak
    D _logs/agent-crons/crontab-backups/crontab.pk33.20260916T071559Z.144458.bak
    D _logs/agent-crons/crontab-backups/crontab.pk33.20260916T073412Z.466343.bak
    D _logs/agent-crons/crontab-backups/crontab.pk33.20260916T084744Z.1437790.bak
    D _logs/sweep/.downstreak-aw

Six are crontab backups (the manager prunes to the newest 40, and those six fell off the end — expected, benign). One is `_logs/sweep/.downstreak-aw` — a hidden marker file, unexplained. **Deletions are the one dirty state that can lose content**, so confirm before `git checkout`/`git restore` anything.

### 4.4 The conclusion that matters

**~250 of ~893 dirty entries are machine-generated log churn that the crons rewrite every cycle.** Pausing the crons (Phase 5) dissolves most of the dirty-tree problem on its own — the churn stops accumulating and the tree converges on whatever the last real write was.

**The genuine uncommitted work is small and identifiable:**

| Repo | Real uncommitted work |
|---|---|
| zcontent1 | 3 TSVs (`CARD-SOURCE-CANDIDATES-{hindi,latin}`, `CURRENT`) + 562 history snapshots (archive or ignore) |
| zprotocols | 9 staged ctx files + `LATEST.md` — cron-staged, need one commit |
| ztemp33 | 6 briefs + `lanes/` + `launch-lanes.sh` |
| zdash222 | ~8 real files (3 registers, 2 dod, 1 publish report, 1 catalog.json, 1 ledger) |
| efus3 | ~15 real (pk-console excluded), 1 submodule pointer, 2 intake logs, 7 deletions |

**So item (c) — "cleaning up dirty git tree" — is really item (5) — "pause the crons" — followed by a small, surgical commit pass.** The order matters: cleaning before pausing means it all gets dirty again within one cycle.

---

## 5. Storage and CPU: consolidated plan

### 5.1 Where the storage is (SSD 915 G usable, 655 G used / 215 G free)

From audit #2, confirmed:

| Item | Size | Action |
|---|---|---|
| `~/.local/share/opencode/opencode.db` | **244 G** | **delete** — dead tool's event log, no consumer |
| `zsm-ncert-20260919/ncert-pdfs` | **131 G** | **shift to HDD** — downloaded corpus, no live consumer |
| `zncert-w1/library` | 14 G | keep on SSD (see §5.4) |
| `efus3-demoguest-merge` | 11 G | shift to HDD once merge is closed |
| `graphify-out` across 43 repos | 8.0 G | delete (git-ignored, regenerable) |
| `tzerp/zsm1-tzerp/.claude/worktrees` | 6.9 G | prune 24 worktrees — **after checking merge status** |
| `zk/` (vec + index sqlite) | 6.5 G | see §5.3 |
| `node_modules` | 5.7 G | delete if repos paused |
| npm/pnpm/gradle/pip caches | 7.3 G | safe |
| `/var/log/journal` | 4.0 G | `journalctl --vacuum-size=500M` |
| `~/.adal/versions/1.8.24` | 968 M | superseded |
| `.claude/projects` | 906 M | prune selectively |

### 5.2 The HDD has room and a proven pattern

    /mnt/hdd1t/ssot_engine_efus3/dc  ->  /home/pk33/zexp1/efus3/ssot_engine_efus3/dc   (bind)
    /mnt/hdd1t/ssot_engine_efus3/rc  ->  /home/pk33/zexp1/efus3/ssot_engine_efus3/rc   (bind)

Both are already bind-mounted in `/etc/fstab` with `nofail` + `x-systemd.requires-mounts-for`. This is why the 126 GB + 60 GB pre-fix quarantine files are on the HDD, not the SSD. **The pattern is established and proven in production.**

HDD: **645 G free**. Sufficient for everything on the shift list.

### 5.3 On `zk/index.sqlite` — a judgement call, and I'd argue delete

`zdash222/_logs/zk/index.sqlite` = 2.27 GB, plus a stale `index.sqlite.prev` (898 MB, Sep 15).

It holds `docs=35919 chunks=737128`. But it is **the old KG** — the one you're reworking. And it is **regenerable in 125 seconds** (`zk embed` measured). It also contains 11 duplicate `claude-memory` scopes and, until `fd5fa4a83` fixed it, 17,701 graphify-derived files.

I'd **delete it and the `.prev`**, not shift it — but only you know whether you want a reference baseline for comparison. If you do, shift the 2.27 GB and delete the 898 MB `.prev`.

Also worth noting: **a vector store living under `_logs/` is a structural smell.** Nothing distinguishes "important derived data" from "rotten logs" in that path. Whatever the rework lands on, give it a home outside `_logs/`.

### 5.4 The one place HDD will bite

`zncert-w1/library/` (14 GB) is the **source** corpus that `out/` is built from, and the content engine is being reworked. Random reads of 2,569 PDFs during a build, on a spinning disk, will be slow.

**Recommendation: keep `library/` on SSD.** Shift the 131 GB `zsm-ncert-20260919` zips instead — those are the genuinely cold, language-split raw downloads with no live consumer.

### 5.5 CPU: what to free

| Consumer | Cost | Action |
|---|---|---|
| **graphify / zk family** | **100% of a core, continuously; 738 s per 10-min pass; 89–100 °C** | **disable — Phase 1** |
| `idle_inject` kernel threads | 32 threads ≈ 99% of one core | container-runtime (k3s/docker); stops only if you stop those |
| Docker healthchecks | ~50 execs/min, 70k ctx switches/sec | raise intervals; replace `rabbitmq-diagnostics` with a lighter probe |
| `fast_pulse` (every 2 min) | spawns `verify efus3` | disable during rework |
| `sau_keeper` (every 10 min) | `zcons-dc.py rc ingest` — the O(n) rescan | disable during rework |
| k3s | ~9% of a core + 62–73 pod restarts | stop if not needed for the rework |
| `organ_pulse` (every 30 min) | ~85 s per run | disable during rework |

---

## 6. The record — a single ledger, and where it should live

You asked for a record of relocation **and** destructive changes. The pieces already exist; what's missing is a single index.

### 6.1 What already exists

| Record | Where | Format |
|---|---|---|
| Cron registry (103 pk33 rows, state + purpose + installed_by) | `efus3/_logs/agent-crons/registry.tsv` | TSV, 13 cols |
| Crontab write backups (40 newest, auto-rotated) | `efus3/_logs/agent-crons/crontab-backups/` | timestamped `.bak` |
| Pre-pause crontab snapshots | `~/crontab.bak-before-graphify-pause.20261014-*` | flat |
| Hand-written pause note | `~/GRAPHIFY-PAUSED-2026-10-14.md` | markdown |
| efus3 commit log (the `close-down sweep` precedent) | git history | — |
| efus3 `_logs/_history` (152 M) | `efus3/_logs/_history/` | rotated logs |
| zcontent1 `docs/history/CURRENT/*.tsv.gz` | `zcontent1/docs/history/` | timestamped snapshots |

The pattern is established. What's missing is one place that ties a **destructive action** to **why**, **when**, and **how to reverse it**.

### 6.2 Proposed ledger

One file, `efus3/_logs/rework-ledger/2026-10-30.md` (see §6.4 on placement).

Columns:

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|

`class` ∈ {`delete`, `shift`, `pause`, `commit`, `prune`, `config`}.

`recovery` is the important one — for every destructive row, the exact command or path that undoes it. For shifts, the bind-mount line. For pauses, the `pcl_selfcron.sh enable <name>`. For deletes, either "regenerable: `<command>`" or "gone — source at `<path>`".

### 6.3 Recording the crons specifically

Two layers, and they compose:

1. **Machine truth** — `pcl_selfcron.sh disable <name>` per entry. The registry row survives with `state=DISABLED`, schedule, command, purpose, installed_by, installed_at. This is authoritative and machine-readable.
2. **Human ledger** — one row per disabled entry in the ledger, with the *reason* (which the registry's `purpose` field doesn't capture for a disable).

Do **not** use raw `crontab -e` to comment things out during the rework. The `bc13b9fc5` post-mortem is explicit that unbypassed writes are how the crontab went to 1,294 lines. If you must hand-edit, do it *through* the script's `crontab_write` path so the shrink guard and the backups apply.

### 6.4 One recommendation on the record's home

Put the ledger **inside efus3**, not on the Desktop. efus3 is a git repo with an existing `_logs/` discipline, and the ledger is exactly the kind of thing that should survive a machine rebuild. The Desktop copies of audits #1–#3 are working documents; the ledger is the record of fact.

---

## 7. Recommended sequence

### Phase 0 — The record (do this first, it's free)

1. Create the ledger file with the column set above.
2. `crontab -l > ~/crontab.pre-rework.$(date -u +%Y%m%dT%H%M%SZ).txt`
3. `cp efus3/_logs/agent-crons/registry.tsv ~/registry.pre-rework.tsv`
4. `df -h / /mnt/hdd1t` — the "before" numbers.

Nothing destructive. Everything else references this.

### Phase 1 — Pause the KG family (fixes the thermal problem)

    cd /home/pk33/zexp1/efus3
    bash scripts/pcl_selfcron.sh disable zk-embed-tunnel      # longest tag first — see §1.5
    bash scripts/pcl_selfcron.sh disable zk-index-zdash222
    bash scripts/pcl_selfcron.sh disable zk-index
    bash scripts/pcl_selfcron.sh disable zk-graph
    bash scripts/pcl_selfcron.sh disable zk-embed
    bash scripts/pcl_selfcron.sh disable zk-report
    bash scripts/pcl_selfcron.sh verify

**Stops: a full core, continuously, at 89–100 °C.** Reversible via `enable`. Registry records it.

Also kill the currently-running `graphify` processes (`pkill -f 'graphify update'`) — they're mid-pass and will keep the CPU hot until they finish.

### Phase 2 — Prune the KG scope list

Edit `zdash222/docs/registers/clones.tsv` (the source of truth for `bin/zk scopes`) to remove:

- `3.24.5` → `fvm/versions/3.24.5` (Flutter SDK)
- `zt331-not-worthy` → `trash/zt331-not-worthy`
- `x0-pk-only` → `trash/x001-trash-copy/x0-pk-only`
- 10 of the 11 `claude-memory` rows (keep one)

Then `bin/zk scopes` regenerates `scopes.tsv` → **60 → 47 scopes**, and the RED count drops structurally.

### Phase 3 — The 244 GB

Delete `~/.local/share/opencode/` (244 G). Optional: copy `opencode.db` to HDD first if you want the 6.1 GB of real session content back — the copy will take hours at HDD speeds; a selective `VACUUM` after dropping the `event` table is likely faster but needs ~244 GB free scratch.

### Phase 4 — Shift to HDD

Using the **existing fstab bind-mount pattern**:

- `zsm-ncert-20260919/ncert-pdfs` → 131 G (cold corpus)
- `efus3-demoguest-merge` → 11 G (once the merge is closed)
- `zk/` → 6.5 G (if keeping the index)

**Keep `zncert-w1/library` on SSD** (§5.4).

### Phase 5 — The broad cron pause

Disable the efus3 and zdash222 blocks via `pcl_selfcron.sh`, one entry at a time. Leave running:

- `zk-embed-tunnel` (if kept) — connectivity probe
- anything that publishes to a live surface (`zdash222-bitz-inbox`, `ztg.sh poll`, `zgitea_ff.sh sync`, `zdash222-env-backup`, `b09-autoclose`) — **if you want the published product to stay up during the rework**
- `convo-collect` / `session-record-all` — session continuity, cheap, no mutation of truth

Disable the rest, with the reason in the ledger.

### Phase 6 — Now the git cleanup is tractable

After Phase 5, re-run `git status` in each repo. Only the real work should remain. Then, per repo:

- **zcontent1** — commit the 3 real TSVs; decide whether `docs/history/CURRENT/*.tsv.gz` (562 files, 13 MB) is history worth keeping in git or belongs in `.gitignore`
- **zprotocols** — one commit for the 9 staged ctx files (or fix the `zctx-refresh` cron to commit what it stages)
- **ztemp33** — commit or archive the 6 briefs
- **zdash222** — commit the ~8 real files
- **efus3** — investigate the 7 deletions (§4.3), commit the ~15 real files, update the `zcons1` submodule pointer
- **5 trivial repos** — one commit each

### Phase 7 — Reinitiate

Order: k3s (if needed) → Docker services → `pcl_selfcron.sh enable` in stages (fast_pulse → sau_keeper → the rest) → KG rebuild from the pruned scope list → ZNCERT Stage 3 (the 5,786 AMBIGUOUS rows) → zcontent1 KLO ledger build.

**Consider a thermal guard** before Phase 7 (§2.7) — the graphify pause has now been reverted once. A cron that checks `sensors` and pauses graphify above 85 °C closes that loop permanently.

---

## 8. Two things I would push back on

**Don't shift `zk/index.sqlite` and expect the KG rework to use it.** It is the *old* KG — 2.27 GB, 35,919 docs, 737,128 chunks, built from a scope list that included a Flutter SDK and two trash directories. It regenerates in 125 seconds. I'd delete it rather than shift it — unless you want a reference baseline, in which case shift the 2.27 GB and delete the 898 MB `.prev`.

**Don't treat zsm1 as live work.** `RESUME-HERE.md` reads like a handoff to resume from, and it names three RED arcs. Two of them are now GREEN and the third is at 99.3%. Resuming zsm1 would be re-doing finished work and would disturb the one thing in this entire assessment that is genuinely done, clean, and authoritative. The live work is **zcontent1's KLO ledger** and **ZNCERT Stage 3** — and those are exactly where the rework should land.

---

## 9. Numbers, for the record

### Storage

    SSD  /            915 G usable   655 G used   215 G free   76%
    HDD  /mnt/hdd1t   916 G          226 G used   645 G free   26%

### CPU (sampled live during this assessment)

    graphify update /home/pk33/fvm/versions/3.24.5        100% CPU
    graphify update .../efus3-demoguest-merge             100% CPU, RES 3.3g (10.9% RAM)
    Package id 0:  +100.0 degC  (high +80, crit +100)    <- during graphify
    Package id 0:   +89.0 degC  (high +80, crit +100)    <- again
    idle_inject threads (32)                             ~99% of one core, continuous
    zk_graph.sh pass                                     738 s avg over 20 runs, every 10 min
    ZK GRAPH result                                     ok=3 red=45, every run

### Crons

    live effective lines                                99
    registry rows (pk33)                                103
    registry rows (all machines)                        169
    dead-but-registered (pk33)                          3   (zk-graph, zk-index-zdash222, zdash222-wifi-5g-guard)
    DEAD-UNTRIAGED (all on stale machine 'pk')          8
    PAUSED-BY-ADAL-GRAPHIFY markers in live crontab     5
    zk scopes                                           60 rows / 49 unique, 15 junk

### Git

    zcontent1   595 dirty  (562 = docs/history snapshots, 3 real TSVs)
    efus3       198 dirty  (128 = _logs churn, 15 real, 7 deletions)
    zdash222     65 dirty  (41 ?? mostly _logs/gates, 24 M mostly churn, ~8 real)
    zprotocols   19 dirty  (9 staged ctx files — cron-staged, needs 1 commit)
    ztemp33       6 dirty  (real briefs)
    5 repos       1 each   (trivial)

### zsm1 checkers (re-run, not inherited)

    depth_status.py       1961/1961 PERIOD-READY (100.0%)   0 KCs under target on cards/wex/items
    indianization_status  0 faults / 30,164 artefacts / 149 strands   GREEN
    g6_hindi_status       T1 144/145 (99.3%)  T3 144/144  T4 144/144  T2 140/140   G6-100 RED
    boundary_gate.py      L1-L4 PASS                                          GREEN
    git status            clean

---

## 10. Method and limitations

- All figures are direct `du` / `ls` / `stat` / `ps` / `git` output captured live on 2026-10-30.
- The `opencode.db` internals (from audit #2) were measured by rowid-band sampling because a full `dbstat` scan times out; cross-checked against `PRAGMA page_count`.
- zsm1 checkers were **re-run** rather than read from `RESUME-HERE.md`, per that document's own instruction. The test suite (`pnpm test`) was not re-run — out of scope for a disk/CPU audit.
- `sensors` was sampled at two moments, both during active `graphify` runs. The 89–100 °C range is therefore the loaded figure, not a continuous average.
- **No writes, deletes, moves, cron changes, or config changes were made during this assessment.** All commands were read-only except `git status` / `git log` / `git worktree list` and the zsm1 checker scripts, which write only to their own log/derived paths.

*Assessment performed 2026-10-30. Documentation only — no changes made.*
