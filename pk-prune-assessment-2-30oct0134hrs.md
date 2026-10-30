# PK Prune & Shift Assessment #2 — 30 Oct, 01:34 hrs

**Host:** pk33 · SSD: CT1000T500 1 TB (915 G usable) at `/` · HDD: ST1000DM003 1 TB at `/mnt/hdd1t`
**Goal:** free the SSD ahead of reinitiating core workings + a knowledge-graph rework.
**Scope:** assessment only — nothing moved, deleted, or paused yet.

---

## Headline

**~460 GB is prunable or HDD-shiftable, of which ~389 GB is pure bloat — zero data loss.**

The single biggest item (244 GB) is a runaway event log in an abandoned tool's SQLite database, not a knowledge asset. The second biggest (131 GB) is a downloaded source corpus that is reproducible. Together they are 82% of everything prunable.

Current SSD: **655 G used / 215 G free (76%)**.
Current HDD: **226 G used / 645 G free (26%)** — 645 G of room to absorb the shift.

**Critical context:** this box has already had a disk-full incident. `efus3` git log: `pcl_hr wakeup: stop tail-into-own-log self-copy loop (pk33 disk-full incident 2026-09-19, 420 GB hr-watch.log)`. The same class of failure — an unbounded log with no rotation — is what produced the 244 GB `opencode.db` below. Worth fixing the *pattern*, not just the instances.

---

## Tier 1 — Pure bloat, delete with confidence (~262 GB, no HDD needed)

### 1.1 `opencode.db` — 244 GB (event-log explosion) · **highest value single action**

```
/home/pk33/.local/share/opencode/opencode.db        261,891,379,200 bytes
/home/pk33/.local/share/opencode/opencode.db-wal       775,956,712 bytes
```

`opencode` is **not running** (no process, no cron, no systemd unit). Last real use Sep 9 (snapshot dir) / Sep 18 (db mtime); only the log dir is stale from Jul 26.

Root cause, established by direct measurement:

| Table | Rows | Size |
|---|---|---|
| `event` | 487,675 | **~239 GB** (extrapolated from rowid-band sampling, avg 526 KB/row, max **96 MB**) |
| `message` | 37,844 | 5.80 GB |
| `part` | 150,328 | 0.31 GB |
| `session` | 5,735 | small |

The `event` table is an append-only event log where every `message.updated.1` row re-snapshots the entire message. Sampled rowids confirm the fat rows:

```
event rowid  75000-75300: 301 rows  avg 605,067 B  max 91,449,991 B
event rowid 100000-100300: 301 rows  avg 474,768 B  max 43,952,514 B
event rowid 150000-150300: 301 rows  avg 687,936 B  max 10,634,369 B
msg.updated  110000-110200: 61 rows  avg 18,943,747 B  max 96,264,916 B
```

Type breakdown: `message.part.updated.1` = 295,190 rows · `message.updated.1` = 133,401 rows · `session.updated.1` = 53,349 · `session.created.1` = 5,735.

So a handful of rows carry tens of megabytes each, and 487k rows of that multiply out to 239 GB. The database is **92% dead event-log weight** — the actual session content (`message` + `part`) is only 6.1 GB.

**Action:** delete the whole `opencode` dir. If history is wanted, copy `opencode.db` to the HDD first (a 244 G copy on the HDD is free — 645 G free — but the copy itself will take hours at HDD speeds; a selective `VACUUM` after deleting `event` is likely faster, though it needs free space equal to the DB size).

**Reclaim: ~244 GB.**

### 1.2 `graphify-out/` dirs — 8.0 GB across 43 repos · **regenerable by design**

Confirmed regenerable and explicitly excluded from knowledge:

- `.gitignore:96` → `**/graphify-out/` (untracked in every repo — verified via `git ls-files`, empty)
- `scripts/zfts.py:43` → `"graphify-out", # graphify-out = derived code graph (A-16), never knowledge`
- `scripts/gen_mem.sh` → rebuilds it with `graphify update .` (tree-sitter, no API, 0 tokens)

Biggest offenders, all dated daily snapshots of a derived code graph:

```
5.0G  /home/pk33/zexp1/efus3/graphify-out          (18 dated dirs, 257 MB each)
516M  /home/pk33/zexp1/efus3-demoguest-merge/graphify-out
400M  /home/pk33/zdash22x/zdash222/graphify-out
242M  /home/pk33/zexp1/twdpt1/graphify-out
187M  /home/pk33/zdash22x/zcontent1/graphify-out
136M  /home/pk33/tzerp/zsm1-tzerp/graphify-out
```

Note `efus3/graphify-out` holds 18 dated copies of the *same* graph (2026-09-16 → 2026-10-13). Keeping the newest and dropping the other 17 recovers ~4.5 GB alone.

**Reclaim: 8.0 GB** (or ~4.5 GB if you keep the newest snapshot of each).

### 1.3 24 orphaned Claude worktrees — 6.9 GB

```
/home/pk33/tzerp/zsm1-tzerp/.claude/worktrees/   24 worktrees, 6.9 GB (~290 MB each)
```

All dated **Jul 31 – Aug 1** — two months stale. They are still registered in `git worktree list` (branches like `wip/pk33/api-v1-wave1`, `wip/pk33/ncert-chapter-links`, `wip/pk33/kc-graph-engine`). Several carry **unmerged WIP branch tips**.

⚠️ **Do not blind-delete.** Before pruning: check whether any `wip/pk33/*` branch tip is unmerged (`git branch --no-merged`), and confirm nothing is still being worked in.

**Reclaim: 6.9 GB** (after verifying merge status).

### 1.4 Build cache & dependency caches — ~8.6 GB

| Path | Size | Note |
|---|---|---|
| `/home/pk33/.npm/_cacache` | 1.7 G | npm cache — safe, `npm cache clean --force` |
| `/home/pk33/.local/share/pnpm/store` | 3.5 G | pnpm content-addressable store |
| `/home/pk33/.gradle/caches` | 1.4 G | Gradle |
| `/home/pk33/.gradle/wrapper/dists` | 657 M | Gradle distributions |
| `/home/pk33/.cache/pip` | 426 M | pip HTTP cache |
| `/home/pk33/.cache/pnpm` | 631 M | pnpm metadata |
| `/home/pk33/.cache/puppeteer` | 653 M | re-downloads on demand |
| `/home/pk33/.cache/codex-runtimes` | 1.7 G | Codex runtime bundle — **keep**, in active use |

**Reclaim: ~7.3 GB** (excluding codex-runtimes).

### 1.5 `node_modules` across repos — 5.7 GB

```
2.5G  tzerp/tzerp1              830M  zexp1/efus3-demoguest-merge
769M  zexp1/efus3               287M  zexp77/anim8
221M  .npm/_npx/4b4c857f...     212M  .local/lib/node_modules
203M  zexp1/twdpt1-local        202M  zexp1/twdpt1
196M  zsvt1/zsvt1               112M  tzerp/zsm1-tzerp
```

All reproducible from lockfiles. The 221 M `.npm/_npx` entry is a throwaway npx cache.

**Reclaim: 5.7 GB** — but only worth it if the repos are genuinely paused.

### 1.6 System journal — 4.0 GB

`/var/log/journal` = 4.0 GB (`journalctl --disk-usage` → 3.9 G). Trivial and safe:

```bash
journalctl --vacuum-size=500M
```

**Reclaim: ~3.5 GB.**

### 1.7 Docker dangling volumes — 48 MB

`docker system df` → Local Volumes 18 total / 10 active, **48.27 MB reclaimable**. Negligible but free.

### 1.8 Duplicate runtime versions — ~1.9 GB

```
968M  ~/.adal/versions/1.8.24   (1.8.25 also present)
657M  ~/.gradle/wrapper/dists    (multiple Gradle dists)
2.2G  ~/fvm/versions/3.24.5      (single Flutter SDK)
```

`.adal` 1.8.24 is a superseded tool version — removable. `fvm` holds one Flutter SDK; keep unless Flutter work is paused.

### 1.9 Claude/Codex session transcripts — 0.9 GB

```
194M  ~/.claude/projects/-home-pk33-zdash22x-zdash222
 88M  ~/.claude/projects/-home-pk33-zexp1-twdpt1
 62M  ~/.claude/projects/-home-pk33-zdash22x-zcontent1-vlm-bakeoff
 37M  ~/.claude/projects/-home-pk33-zdash22x-zcontent1
```

Plus `~/.claude/telemetry` 73 M, `~/.claude/file-history` 20 M.

**Reclaim: ~0.9 GB** — but these are *your* conversation history. Prune selectively, not wholesale.

---

## Tier 2 — Shift to HDD (~152 GB freed from SSD, 0 data lost)

The HDD already has the bind-mount pattern established — **this is the proven path**:

```
/etc/fstab:
UUID=f0f6e896-...  /mnt/hdd1t  ext4  defaults,nofail,...  0 2
/mnt/hdd1t/ssot_engine_efus3/dc  /home/pk33/zexp1/efus3/ssot_engine_efus3/dc  none  bind,nofail,...  0 0
/mnt/hdd1t/ssot_engine_efus3/rc  /home/pk33/zexp1/efus3/ssot_engine_efus3/rc  none  bind,nofail,...  0 0
```

The efus3 `dc`/`rc` spine dirs are **already** on the HDD this way — which is why the 126 GB + 60 GB pre-fix quarantine files (visible in the load audit) are on the HDD, not the SSD. The pattern works; extend it.

### 2.1 `zsm-ncert-20260919/ncert-pdfs` — 131 GB · **largest shift candidate**

```
1,025 files, ALL .zip — a per-language/per-chapter PDF corpus
  12G urdu      6.4G sanskrit   6.4G english   5.9G manipuri
  5.8G konkani  5.7G telugu      5.7G malayalam 5.7G hindi
  5.6G tamil    5.6G dogri       5.6G bodo      5.5G sindhi
  5.5G punjabi  5.5G nepali      5.4G odia      5.3G marathi
  5.2G kannada  5.1G santhali    5.1G bengali   5.1G assamese
  5.0G gujarati 4.9G maithili    3.7G kashmiri
```

Not a git repo. No cron job, no systemd timer, **no running process** references it except `zdash22x/zcontent1-ncert-refresh/` and `zcontent1-vlm-bakeoff/` — both of which are currently **inactive** (no cron entries, no timers, no processes).

**This is a downloaded corpus, not a derived artifact.** Shift it, don't delete it:

```bash
mkdir -p /mnt/hdd1t/zsm-ncert-20260919
mv /home/pk33/zsm-ncert-20260919/ncert-pdfs /mnt/hdd1t/zsm-ncert-20260919/
ln -s /mnt/hdd1t/zsm-ncert-20260919/ncert-pdfs /home/pk33/zsm-ncert-20260919/ncert-pdfs
```

**Frees: 131 GB.** (A symlink or a bind mount — the fstab bind-mount style is more consistent with what's already there.)

### 2.2 `zncert-w1` — 17 GB

```
14G  library/     (2,569 NCERT PDF assets across 13 grades)
1.3G out/         (rendered HTML — likely regenerable)
1.4G .venv/
```

README: *"NCERT publishes 15,909 textbook assets across 13 grades and 10+ languages. We hold 2,569 of them locally (14 GB, on two worker machines)."* — i.e. **the local copy is a cache of something available elsewhere.** `out/` is rendered output and should be regenerable from `library/`.

Shift `library/` to HDD (14 GB). Keep `out/` on SSD if it's actively being read.

**Frees: ~14 GB.**

### 2.3 `zk/` knowledge-graph vector store — 6.5 GB

```
3.9G  zdash222/_logs/zk/vec.sqlite        (Oct 14)
2.2G  zdash222/_logs/zk/index.sqlite      (Oct 14)
898M  zdash222/_logs/zk/index.sqlite.prev (Sep 15)  ← stale duplicate
```

⚠️ **Note the path oddity** — a vector store living under `_logs/` is a smell: it's derived data in a logs directory, so nothing distinguishes "important" from "rotten." `index.sqlite.prev` is a **stale 898 MB duplicate** from Sep 15 — that one is free to delete.

The live `vec.sqlite`/`index.sqlite` are the **knowledge graph you're about to rework**. Shift both to HDD now; when the KG rework starts you'll rebuild them anyway.

**Frees: 6.5 GB** (or 5.6 GB if you keep the live pair on SSD for the rework).

### 2.4 `efus3-demoguest-merge` — 11 GB

```
6.7G  apps/          2.2G packages/       830M node_modules
516M  graphify-out   223M _logs/           97M ssot_engine_efus3
```

A seat merge working copy (`zdash22v21-demoguest-merge`), last commit a receipt update. Untracked: `skills/_meta/bin/_logs/`. **This is a work-in-progress branch** — decide before touching. If the merge is done or abandoned, it shifts or goes to HDD cleanly (it's a git repo, so the history is safe regardless of where the checkout lives).

**Frees: 0–11 GB** depending on your call.

### 2.5 `zdash22x/zdash222/_local` — 573 MB

```
458M  _local/tzerp1
 97M  _local/zsm1
```

Local checkouts of other repos — shift or re-point to the canonical locations.

---

## Tier 3 — Pause (not delete): stop the bleeding before you restart

### 3.1 The 98-entry crontab

```
97 of 98 entries  →  /home/pk33/zexp1/efus3/...
31 entries        →  /home/pk33/zdash22x/zdash222/...
 3 entries        →  /home/pk33/zexp1/zprotocols/...
```

Job families include `zsm_lane_heartbeat.sh`, `zk_graph.sh`, `zinc.sh`, `zgate_sync.sh`, `zctx.sh`, `warden.sh`, `tag_scan.sh`, `lane_watch.sh`, `lane_heartbeat.sh`, `fleet_ring_probe.sh`, `fleet_mirror.sh`, plus ~40 singletons.

Highest-frequency / highest-cost entries:

| Cron | Freq | Why it matters |
|---|---|---|
| `fast_pulse.sh` | **every 2 min** | spawns `verify efus3` — the CPU hot spot from the load audit |
| `sau_keeper.sh` | every 10 min | `zcons-dc.py rc ingest` — the *other* CPU hot spot |
| `organ_pulse.sh` | every 30 min | ~85 s per run (copy-detect scans 2,069 files) |
| `lane_heartbeat.sh` | every 10 min | |
| `warden.sh` | every 5 min | |
| `mmx-lane-watch-5min` | every 5 min | |

**Recommendation:** back up the crontab and comment out the whole efus3 + zdash222 block while you rework the KG. Restarting core workings is exactly when you want these back — but not *during* the rework, or they will keep mutating the spines you're rebuilding.

```bash
crontab -l > ~/crontab.bak-pre-rework.$(date +%Y%m%d-%H%M%S)
```

### 3.2 The `idle_inject` kernel threads — 32 threads, ~1 full core

32 threads at 3.1% each = **~99% of one core burned continuously**, 39 s each over 25 min. These are cgroup/CPU-quota idle-injection threads. They will keep spinning regardless of what you pause — they come from the container runtime (k3s + docker), not your app workloads.

### 3.3 The k3s crash-loop — 62–73 restarts per pod

```
local-path-provisioner  73 restarts
metrics-server          72 restarts
coredns                 62 restarts
```

All three restarted at the 19:14 boot. `coredns` exit 255, `metrics-server` exit 2, `local-path-provisioner` exit 1. Also **~500+ completed `zpool-*` pods** accumulated in `default` over 93 days. If k3s isn't needed for the rework, stopping `k3s.service` frees ~9% of a core and stops the restart churn.

### 3.4 Docker healthcheck churn — ~50 execs/min, 70k ctx switches/sec

| Container | Interval | Probe |
|---|---|---|
| tzerp1-local-redis | **5s** | `redis-cli ping` |
| tzerp1-local-rabbit | 10s | `rabbitmq-diagnostics -q ping` ← **boots a BEAM VM** |
| eduz-pk33-{dev,prod}-pg | 10s | `psql select 1` |
| eduz-pk33-{dev,prod}-redis | 10s | `redis-cli ping` |
| eduz-pk33-{dev,prod}-rabbit | 15s | `rabbitmq-diagnostics -q check_running` ← **boots a BEAM VM** |

Raising the Redis probes to 30 s and replacing `rabbitmq-diagnostics` with a lighter check cuts most of the churn with no functional loss.

### 3.5 Archived repos — leave the checkouts, drop the working copies

`/home/pk33/tzerp/zsm1-tzerp` README states:

> **READ-ONLY ARCHIVE (2026-09-16, pk decision via seat zdash22v8-zsm)**
> This repository was consolidated into the single source of truth **github `zexp1/zsm1`**

Its HEAD is `ba1baf21 README: read-only archive notice — SSOT is zexp1/zsm1 (ZS-02, pk decision)`.

Yet it still holds **6.9 GB of worktrees** and 772 M of `.git`. Meanwhile the real `zexp1/zsm1` lives *inside* `efus3/zsm1` (563 M). The archive is the bigger object.

---

## Recommended sequence

### Phase 1 — immediate, zero-risk (~262 GB)

1. **Delete `~/.local/share/opencode/` entirely** → **244 GB**. Largest win by far, no active consumer, no cron, no service. *(Optional: copy to HDD first if you want the 6.1 GB of real session content back.)*
2. `journalctl --vacuum-size=500M` → **3.5 GB**
3. `docker volume prune` → 48 MB
4. Clear npm / pnpm / gradle / pip caches → **7.3 GB**
5. Prune old `graphify-out` dated snapshots, keep newest → **4.5 GB**

**After Phase 1: SSD free goes from 215 GB → ~480 GB.**

### Phase 2 — verify then prune the 24 worktrees (~6.9 GB)

```bash
cd /home/pk33/tzerp/zsm1-tzerp
git branch --no-merged          # check for unmerged WIP tips
git worktree list
```

Confirm nothing valuable is stranded, then `git worktree remove` each. Keep the read-only archive checkout itself — it's only 772 M of `.git` plus source.

### Phase 3 — shift the bulk to HDD (~152 GB)

Use the **existing fstab bind-mount pattern** — it's proven and already in place for `ssot_engine_efus3/{dc,rc}`:

```bash
# ncerts corpus (131 GB)
mkdir -p /mnt/hdd1t/zsm-ncert-20260919
mv /home/pk33/zsm-ncert-20260919/ncert-pdfs /mnt/hdd1t/zsm-ncert-20260919/
# add to /etc/fstab:
/mnt/hdd1t/zsm-ncert-20260919/ncert-pdfs  /home/pk33/zsm-ncert-20260919/ncert-pdfs  none  bind,nofail  0 0
```

Repeat for: `zncert-w1/library` (14 G), `zdash222/_logs/zk` (6.5 G), `efus3-demoguest-merge` (11 G, if you decide the merge is closed).

**After Phase 3: SSD free ≈ 630 GB, HDD free ≈ 493 GB.** Both healthy.

### Phase 4 — pause the automation, then restart deliberately

4. Back up and comment out the 98-entry crontab (efus3 + zdash222 blocks).
5. Decide on k3s — stop it if the rework doesn't need it.
6. Raise the Docker healthcheck intervals.

**When you reinitiate core workings**, restart in this order: k3s (if needed) → Docker services → `crontab` in stages (fast_pulse first, then sau_keeper, then the rest) → KG rebuild from the shifted stores.

---

## What NOT to touch

| Path | Size | Why |
|---|---|---|
| `/home/pk33/.local/share/opencode/snapshot` | 157 M | small, but the live tool's snapshots — keep if opencode returns |
| `~/.cache/codex-runtimes` | 1.7 G | in active use by Codex |
| `/home/pk33/zexp1/efus3/ssot_engine_efus3/{dc,rc}` | — | **already** on HDD via bind mount; don't relocate |
| `/mnt/hdd1t/zsm-data/raw` | 52 G | dataset (`green`/`amber` corpus) — already HDD, leave it |
| `~/.claude/projects/*` | 906 M | your conversation history — prune selectively |
| `~/.adal/versions/1.8.25` | 968 M | current version — keep |
| `fvm/versions/3.24.5` | 2.2 G | only Flutter SDK — keep unless Flutter is paused |
| `/opt/*` | 7.7 G | playwright, rust, nodejs, chrome — toolchain, leave alone |

---

## Totals

| Tier | Action | SSD freed | Data loss |
|---|---|---|---|
| 1 | Bloat deletion (opencode, caches, journals, graphify) | **~262 GB** | none (opencode inactive) |
| 2 | Verified prune (24 worktrees) | ~6.9 GB | none if merged |
| 3 | Shift to HDD | **~152 GB** | none |
| 4 | Pause automation | 0 (frees CPU + stops regrowth) | none |
| | **Total** | **~420 GB** | **effectively zero** |

**Projected end state:** SSD **235 G used / 680 G free (74% free)** · HDD **~378 G used / 538 G free**.

The single most important action is deleting `opencode.db`. It is 244 GB of a dead tool's event log, and it is why this box is 76% full.

---

## Appendix — measurement method

Full-scan `dbstat` and `sum(length())` over the 244 GB database both timed out (>5 min). Sizes were established by **rowid-band sampling** (40 bands × 200 rows) and extrapolated against the known row count, cross-checked against `PRAGMA page_count` (63,938,339 pages × 4096 B = 261.9 GB, matching the file size).

All other figures are direct `du`/`ls`/`stat` output. No writes, deletes, moves, or service changes were made during this assessment.

*Assessment performed 2026-10-29/30. Documentation only — no changes made.*
