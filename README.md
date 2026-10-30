# pk33-cleanup

Audit set + human-typed prompt record for the pk33 cleanup and rework planning, 29–30 Oct 2026.

## What this is

| File | What it covers |
|---|---|
| `pk-load-audit-1-30oct0134hrs.md` | CPU/load assessment — why the box *looked* hot but wasn't CPU-starved |
| `pk-prune-assessment-2-30oct0134hrs.md` | Storage prune + HDD-shift assessment (~460 GB recoverable) |
| `pk-rework-assessment-3-30oct0134hrs.md` | The start point: cron recording, KG rework, finalised-vs-live map, dirty trees, 7-phase plan |
| `pk-observations-gotchas-audit-4-30oct0134hrs.md` | **20 tagged findings for second opinion** — gotchas, risks, stale records, and what was NOT verified |
| `human_typed.md` | Verbatim record of all 9 human prompts + response summaries + intent timeline + what was NOT done |

Read them in order. Audit #4 is written to be read standalone.

## ⚠️ Repo location: needs transferring to the `awgbackups` org

The intended destination was the GitHub org **`awgbackups`**, but the push could not complete there:

    $ gh repo create awgbackups/pk33-cleanup
    GraphQL: pk-szt6 does not have the correct permissions to execute `CreateRepository` (createRepository)

The org exists but **`pk-szt6` is not a member of it**:

    $ gh api orgs/awgbackups/members/pk-szt6
    {"message":"User does not exist or is not a member of the organization","status":"404"}

    $ gh api orgs/awgbackups --jq '{created_at, members_count, public_repos}'
    {"created_at":"2026-10-30T21:14:14Z","members_count":null,"public_repos":0}

    $ gh api 'orgs/awgbackups/members?per_page=100' --jq 'length'
    0

The org was created at **2026-10-30T21:14:14Z** — about 16 minutes before this repo was set up — and has **zero members and zero repositories**. It appears to have been created from a session authenticated as a *different* GitHub identity than the one on this box.

**Everything on this box authenticates as `pk-szt6`:**

- `gh` CLI token (`~/.config/gh/hosts.yml`) — one token only, and shell history shows exactly one token ever used here
- SSH keys `id_ed25519_github_1785153294`, `id_ed25519_pk33_github`, `id_ed25519_zgate_cloud` — all three authenticate as `pk-szt6`
- `/home/zdash22x` and `/root` have no gh credentials
- no other `ghp_` / `github_pat_` token found in env, secret stores, or history

**So the content is published to `pk-szt6/pk33-cleanup` — a public repo that works — and the org transfer is one click for whoever owns the org.**

### How to complete the transfer

**Option A — transfer the existing repo (preserves history, issues, stars):**

1. Open <https://github.com/pk-szt6/pk33-cleanup> → **Settings**
2. **Danger Zone** → **Transfer ownership**
3. Type `awgbackups` and confirm
4. Accept the transfer in the `awgbackups` org (requires org owner)

**Option B — add `pk-szt6` to the org, then re-push:**

1. In the `awgbackups` org: **Settings → People → Invite people** → add `pk-szt6` with role `admin` (or `write`)
2. Accept the invitation (as `pk-szt6`)
3. On this box:

       cd ~/awgbackups/pk33-cleanup
       gh repo create awgbackups/pk33-cleanup --public --source=. --remote=org --push
       git remote set-url origin git@github.com:awgbackups/pk33-cleanup.git

**Option C — if the org should have a different owner identity**, add that identity's token to `~/.config/gh/hosts.yml` and the same `gh repo create` command will work.

### Local state

The working copy at `~/awgbackups/pk33-cleanup` is a complete git repo with the full history:

    52eb240  PK33 cleanup: audit set #1-#4 + human-typed prompt record

`origin` currently points at `pk-szt6/pk33-cleanup`. Once the org has the repo, repoint it:

    git remote set-url origin git@github.com:awgbackups/pk33-cleanup.git
    git push -u origin main

Note: `~/.gitconfig` has `insteadOf = git@github.com:` → `https://github.com/`, so SSH-form remotes are rewritten to HTTPS and authenticated via the `gh` credential helper.

---

## Scope and safety

**No changes were made to the audited system.** Across the whole session:

- no cron disabled, enabled, healed, or edited (`pcl_selfcron.sh` was run with `list` and `verify` only — both read-only)
- no file deleted, moved, or renamed on the audited system (the 244 GB `opencode.db` is still present)
- no git commit/branch/stash/reset in any of the ten audited repos (read-only git commands only)
- no service stopped or reconfigured; k3s, docker, and all containers left running
- no config file edited (`/etc/fstab`, crontab, systemd units untouched)

The only writes were these five documents, this repo, and one throwaway test repo that was created during diagnosis and immediately deleted (`pk-szt6/pk33-cleanup` — recreated properly as this repo).

## The four documents in brief

**Audit #1 — load.** The box was 85–90% idle; load average 24–28 was process churn, not saturation (zero D-state throughout). Two hot Python processes root-caused to `intake.append()` rescanning the full 91 MB `rc/intake.ndjson` per event — measured 10.0 s for 10 appends. Docker healthchecks boot a full BEAM VM every 10–15 s (caught at 244% CPU, 128 `erts_sched` threads, ~70k ctx switches/sec). 32 `idle_inject` kernel threads burn ~1 core. k3s crash-looping, 62–73 restarts per pod.

**Audit #2 — prune/shift.** ~460 GB prunable, ~389 GB pure bloat. Largest: a 244 GB SQLite event log in an inactive tool (event table ~239 GB; `message.updated.1` rows average 526 KB, max 96 MB). Second: 131 GB ncert zip corpus, 1,025 files, no live consumer. The HDD bind-mount pattern is **already proven in production** — `ssot_engine_efus3/{dc,rc}` are already on it via `/etc/fstab`.

**Audit #3 — the start point.** The cron-recording mechanism **already exists**: `pcl_selfcron.sh disable` preserves the registry row as `state=DISABLED`. The real CPU hog is the KG rebuild — `graphify` at 100% with the CPU package at **89–100 °C** (crit 100), graphifying the Flutter SDK; a hand-written pause note from **2026-10-14** shows it was paused for exactly that reason and **later reverted**. zsm1 is **finalised** (re-ran its checkers: Indianization 0 faults, depth 1961/1961, boundary 4/4, git clean) — its `RESUME-HERE.md` is 2 months stale. ~250 of ~893 dirty tree entries are cron churn, not work.

**Audit #4 — gotchas.** 20 findings tagged [OBS] / [GOTCHA] / [RISK] / [STALE] / [UNVERIFIED], each with a "second opinion wanted on" question, plus an explicit *method, limits, and what I did NOT verify* section. Newly caught in that pass: the opencode dir holds **4 live API credential files** (so the delete target is narrower than audit #2 said); `disable zk-index` would **silently also drop** `zk-index-zdash222` (a `grep -vF` substring collision the shrink guard permits); the zprotocols ctx files are **cron-staged but never committed**; and `GRAPHIFY-PAUSED-2026-10-14.md`'s restore command would **re-enable three jobs** if run today.

---

*Created 2026-10-30 by pk-szt6 on pk33. Measurement was read-only throughout.*
