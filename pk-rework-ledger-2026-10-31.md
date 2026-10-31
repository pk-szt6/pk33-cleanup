<!-- aristotle
category: record
as-of: 2026-10-31
owner: pk33
-->
# pk33 rework ledger — 2026-10-31

The record of fact for the pk33 cleanup/rework. Every relocation, pause, and
destructive action gets a row here. The registry
(`efus3/_logs/agent-crons/registry.tsv`) is the *machine truth* for cron state;
this file is the *human ledger* — it carries the reason and the recovery path,
which the registry's `purpose` field does not.

Plan of record: `pk-rework-assessment-3-30oct0134hrs.md` (audit #3) and
`pk-observations-gotchas-audit-4-30oct0134hrs.md` (audit #4), both in
`~/awgbackups/pk33-cleanup` and pushed to https://github.com/pk-szt6/pk33-cleanup
(the `awgbackups` org transfer is still blocked — `pk-szt6` is not a member).

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|

## Phase 0 — the record (non-destructive, done first)

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 0.1 | 2026-10-31T21:57Z | config | `crontab` (user pk33) | snapshot to `~/crontab.pre-rework.20261031T215718Z.txt` | 104 lines | 104 lines | machine truth before any cron write | yes | the snapshot file itself |
| 0.2 | 2026-10-31T21:57Z | config | `efus3/_logs/agent-crons/registry.tsv` | snapshot to `~/registry.pre-rework.tsv` | 49149 B | 49149 B | machine truth before any state write | yes | the snapshot file itself |
| 0.3 | 2026-10-31T21:57Z | config | `/` and `/mnt/hdd1t` | "before" disk numbers to `~/df.pre-rework.20261031T215718Z.txt` | `/` 657G used of 915G (76%) | unchanged | baseline for the prune/shift phases | yes | the snapshot file itself |
| 0.4 | 2026-10-31T21:58Z | config | this file | created `efus3/_logs/rework-ledger/2026-10-31.md` | — | — | single index of relocation + destructive changes (audit #3 §6) | yes | git history |

## Phase 1 — pause the KG family (audit #3 §7)

Reason for all six: the `zk` family drives `graphify update`, which pins a core
at ~100% and drives the CPU package to 89–100 °C (critical = 100 °C). On
2026-10-31T21:56Z the package read **97 °C** with `graphify update` at 99% CPU.
This pause was applied on 2026-10-14 and **reverted**; the live duplicate
`zk-index` cron line is what brought it back.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 1.1 | 2026-10-31 | pause | `zk-embed-tunnel` | `pcl_selfcron.sh disable` | live, */5 | pending | audit #3 Phase 1 — probes pk33:18095 -> pk31:8095 bge-m3 embed tunnel; Phase 5 would keep it if the tunnel is wanted | yes | `cd /home/pk33/zexp1/efus3 && bash scripts/pcl_selfcron.sh enable zk-embed-tunnel` |
| 1.2 | 2026-10-31 | pause | `zk-index-zdash222` | `pcl_selfcron.sh disable` | live, 4,14,24,34,44,54 | pending | KG index rebuild every 10 min — CPU hog | yes | `bash scripts/pcl_selfcron.sh enable zk-index-zdash222` |
| 1.3 | 2026-10-31 | pause | `zk-index` | `pcl_selfcron.sh disable` | live, */10 + 1 paused duplicate | pending | the CPU hog: `zskill zk index --if-stale` + `bin/zk_graph.sh --missing` (audit #1 measured 738 s per pass) | yes | `bash scripts/pcl_selfcron.sh enable zk-index` |
| 1.4 | 2026-10-31 | pause | `zk-graph` | `pcl_selfcron.sh disable` | live, 41 */6 | pending | `bin/zk_graph.sh` -> `graphify update` per clone scope | yes | `bash scripts/pcl_selfcron.sh enable zk-graph` |
| 1.5 | 2026-10-31 | pause | `zk-embed` | `pcl_selfcron.sh disable` | live, 23 * | pending | `zskill zk embed --batch 128` — embedding pass | yes | `bash scripts/pcl_selfcron.sh enable zk-embed` |
| 1.6 | 2026-10-31 | pause | `zk-report` | `pcl_selfcron.sh disable` | live, 3 * | pending | hourly COMPUTED report -> REPORTS.md (+commit/push) | yes | `bash scripts/pcl_selfcron.sh enable zk-report` |
| 1.7 | 2026-10-31 | pause | running `graphify update` processes | kill in-flight passes | 1 core @ 99% | 0 processes | mid-pass runs keep the CPU hot until they finish (audit #3 Phase 1) | yes | they re-run on next schedule; nothing deleted |
| 1.8 | 2026-10-31 | config | CPU package temperature | measured before/after | 97 °C (crit 100) | 45 °C | the whole point of Phase 1 | — | — |

### Ordering note (gotcha from audit #4)

`pcl_selfcron.sh disable` removes crontab lines with `grep -vF` on the tag, and
`# pcl-selfcron:zk-index` is a **substring** of `# pcl-selfcron:zk-index-zdash222`.
So `disable zk-index` silently removes the `zk-index-zdash222` line *without*
marking its registry row DISABLED. `zk-index-zdash222` is therefore disabled
**first** (1.2 before 1.3), so the registry stays truthful.

### Not done in Phase 1

- No file was deleted, moved, or renamed.
- No other cron was touched (the broad pause is Phase 5, pending).
- No registry row was removed; only `state` flipped ACTIVE -> DISABLED.
- The `#PAUSED-BY-ADAL-GRAPHIFY:` commented lines for `zk-graph`, `zk-embed`,
  `zk-index`, `zk-index-zdash222` were consumed by the tag removal. The
  original crontab is preserved in `~/crontab.pre-rework.20261031T215718Z.txt`
  and in `~/GRAPHIFY-PAUSED-2026-10-14.md`.

### Validated: the pause will hold

Checked after the pause, because a pause that a heal cron silently undoes is
not a pause:

- `pcl_selfcron.sh verify` reports all six as **`DISABLED`**, not `DEAD`
  (103 registered, 1 dead — the one dead entry is pre-existing and unrelated:
  `zdash222-wifi-5g-guard`).
- `pcl_selfcron.sh heal` skips `DISABLED` rows explicitly
  (`skipped (DISABLED): <name>`), and `zfable/ops/warden.sh` — which runs every
  5 minutes and calls `verify` — therefore cannot reinstall them.

This is why the pause was done through `pcl_selfcron.sh disable` and **not** by
commenting the lines out by hand. The hand-commented line in the box right now
(`#PAUSED-BY-ADAL-NONROOT-WIFI: ... zdash222-wifi-5g-guard`) is the counter
example: it has no registry state change, so `verify` counts it `DEAD`, and
`warden.sh` calls `heal` every 5 minutes to try to put it back. The 2026-10-14
ADAL-GRAPHIFY pause was the same pattern, and it is how the graphify jobs came
back after being stopped.

## Phase 2 — prune the KG scope list (audit #3 §7)

**Audit #3's Phase 2 instruction was mechanically wrong and has been corrected.**
It said to edit `zdash222/docs/registers/clones.tsv`. Two problems:

1. `clones.tsv` is **regenerated by `bin/recon_fleet.sh` on every beat**, so a
   hand-edit there is undone within the hour. The scope list's real sources are
   `clones.tsv` (rows), `bin/zk`'s hardcoded `~/.claude/projects/*/memory` glob,
   and `docs/knowledge/zk-scopes-extra.tsv`.
2. The 11 `claude-memory` rows are **not in `clones.tsv` at all** — they are
   appended by `bin/zk` itself.

The fix was therefore applied at the **scope layer** (`bin/zk`), which is where
the prune actually belongs.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 2.1 | 2026-10-31 | prune | `bin/zk` — scope generation | skip clone roots under `fvm/` and `trash/` | 61 scopes | 58 scopes | `fvm/versions/3.24.5` is the Flutter SDK (1.7 GB of vendor Dart, 49011 commits behind) — the scope that pinned a core at 100% and drove the package to 89-100 °C; `trash/*` is discarded content | yes | `git revert c3fa3e7` (zdash222) |
| 2.2 | 2026-10-31 | prune | `bin/zk` — memory corpus | skip empty `~/.claude/projects/*/memory` dirs | 58 scopes / 12 memory rows | 53 scopes / 7 memory rows | 5 of the 12 dirs hold 0 files — they contributed 0 documents but a scope row each | yes | `git revert c3fa3e7` (zdash222) |
| 2.3 | 2026-10-31 | commit | zdash222 | commit `c3fa3e7` (bin/zk + regenerated `_logs/zk/scopes.tsv`) | dirty | committed | the scope prune must survive `recon_fleet.sh` and a rebuild | yes | `git revert c3fa3e7` |
| 2.4 | 2026-10-31 | config | zdash222 `_logs/zk/scopes.tsv` | regenerated by `bin/zk scopes` | 61 rows | 53 rows | follows 2.1 + 2.2 | yes | regenerate |

**Graphify population (repo scopes only): 49 -> 46.** RED baseline at the time
of the prune was `ok=4 red=45`; the RED count falls only once a full
`zk_graph.sh` pass completes, which is exactly what the prune makes affordable.

### Correction to audit #4's open question 2

Audit #4 asked: *"Is the KG scope name the primary key? If yes, the 11
`claude-memory` rows are already colliding and the index's 35,919 docs may be
partially lost."* — **Checked: no documents are at risk.**

`scripts/zfts.py` `_load_scopes_tsv()` does `scopes.setdefault(scope, []).append(root)`:
same-named rows are **merged by appending their roots**, so all 7 memory dirs
are walked and indexed under one scope name. The merge is the intended shape
for the memory corpus, and `bin/zk_embed.py` filters on exactly that name
(`scope != 'claude-memory' OR path LIKE '%/memory/%'`) — renaming the scope
would have silently broken that filter and started embedding non-memory files.
The collision was cosmetic (scope count), not destructive.

## Bonus — repair the zdash222 git state (not in the audit plan)

Found while committing 2.3: **zdash222 had been stuck in an interactive rebase
since 2026-10-07 22:03** — a conflict on `seats/zdash22v1-knowledge/REPORTS.md`
that was never resolved. `main` was frozen at `39497bc` while every hourly cron
commit since landed on the **detached HEAD** (81 commits in 24 days). The repo
reported "interactive rebase in progress" for all of that time.

| # | date | class | target | action | before_size | after_size | reason | reversible | recovery |
|---|---|---|---|---|---|---|---|---|---|
| 3.1 | 2026-10-31 | config | zdash222 refs | create `backup/pre-rebase-main-2026-10-31` -> old `main` (39497bc) and `backup/detached-line-2026-10-31` -> detached HEAD (c3fa3e7) | — | 2 backup branches | both tips preserved before anything moved | yes | `git branch -f backup/pre-rebase-main-2026-10-31 39497bc` |
| 3.2 | 2026-10-31 | config | zdash222 rebase state | `git rebase --quit` | rebase in progress since 2026-10-07 | rebase state cleared | clears the 24-day-old stuck state; HEAD, index and working tree are left exactly as they are, and the Oct-7 autostash is saved to the stash list rather than applied | yes | `git stash list` -> `stash@{0}: autostash` (81d7ddaa) |
| 3.3 | 2026-10-31 | commit | zdash222 `main` | `git branch -f main HEAD` + `git checkout main` | main = 39497bc (stale, 24 d) | main = c3fa3e7 (82 commits ahead of a269bf4) | the detached line is the *newer* history; the 2 dropped commits are the redundant `v1-knowledge: hourly computed report` commits the rebase was trying to eliminate | yes | `git branch -f main backup/pre-rebase-main-2026-10-31` |

Working tree was **not** touched by 3.2/3.3: dirty-file count is 65 before and
after. `origin/main` (gitea) still points at the old tip, so local `main` and
`origin/main` now diverge (82 local vs 2 remote commits) — **a push would need
`--force-with-lease` and has NOT been done.**

### Not done

- No push to `origin` for zdash222 (divergence would need a force push).
- The 65 dirty files in zdash222 are still dirty — Phase 6 (git cleanup) is
  still ahead. The cron churn is what keeps them dirty, and the broad cron
  pause (Phase 5) has not happened.
- The `stash@{0}` autostash from 2026-10-07 has not been popped or dropped.

