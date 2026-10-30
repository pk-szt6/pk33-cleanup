# PK Load Audit #1 — 30 Oct, 01:34 hrs

**Host:** pk33 · 32 cores · Linux 7.0.0-38-generic · up ~25 min at time of scan
**Scope:** CPU assessment only — no changes made.

---

## Headline

**The machine is not CPU-starved.** 32 cores, CPU **85–90% idle** at scan time.

Load average (10–28) is **not** CPU saturation — it is process churn. Zero D-state
(I/O-blocked) processes were observed at any point during the scan. The load number is
misleading: a quick `top` glance would suggest nothing is wrong.

---

## 1. What actually burned CPU

### 1.1 Two Python processes in `/home/pk33/zexp1/efus3` — the real hotspot (was 90%+ each)

| PID | Command |
|---|---|
| 246592 | `zcons1/bin/zcons-dc.py --home ssot_engine_efus3 --centre rc ingest --file .../.rc-feed-intake.ndjson` |
| 248866 | `python3 -m consolidation_engine.verify.verify efus3` |

Both trace to the same root cause in `zcons1/ssot_engine/intake.py`:

- `append()` (line 69) calls `_scan_max_seq(f)` on every single event — a **full
  re-scan of the entire 91 MB `rc/intake.ndjson`** just to find the next sequence number.
- `append_many()` calls `append()` once per event, so a 76-event batch performs
  **76 full rescans of 91 MB**.
- Measured: `append_many` of 10 events = **10.0 s**. One `_scan_max_seq` pass = **1.9 s**.

That is O(events x filesize) per run. The file grows every run, so the cost degrades
monotonically over time.

`verify` then repeats the same shape of work: `iter_events()` parses all **221,634**
lines, plus `max_seq()` does another full parse.

**Cron drivers:**

```
*/10 * * * *  sau_keeper.sh  -> sau_engine feed -> feed_rc.py -> zcons-dc.py rc ingest
*/2  * * * *  fast_pulse.sh  -> verify efus3
```

The 2-minute `fast_pulse` cadence is why a fresh `verify` process was observed
spawning on each inspection during the audit.

### 1.2 Docker healthchecks — the churn behind the fake load average

~**50 `docker exec` calls per minute** across 11 containers, on aggressive intervals:

| Container | Interval | Timeout |
|---|---|---|
| tzerp1-local-redis | 5s | 3s |
| tzerp1-local-rabbit | 10s | 5s |
| eduz-pk33-dev-pg | 10s | 5s |
| eduz-pk33-dev-redis | 10s | 3s |
| eduz-pk33-prod-pg | 10s | 5s |
| eduz-pk33-prod-redis | 10s | 3s |
| eduz-pk33-dev-rabbit | 15s | 10s |
| eduz-pk33-prod-rabbit | 15s | 10s |

Each `rabbitmq-diagnostics ping` spawns a **full BEAM VM**. One was caught at
**244% CPU** with **128 `erts_sched` threads**. Context switches ran at
**~70,000/sec**. The RabbitMQ `check_running` healthcheck is the worst offender.

### 1.3 `idle_inject` kernel threads — 32 of them, ~99% of one core combined

3.1% each, 39 s CPU time over 25 min, one per core. This is CPU-idle-time injection
(typically cgroup / CPU-quota throttling). It is the single largest **steady**
consumer, and explains why "idle" CPU is not 100%.

### 1.4 `k3s server` — ~9% steady, pods crash-looping

| Pod | Restarts |
|---|---|
| kube-system/local-path-provisioner-58d557dc48-6mdtn | 73 |
| kube-system/metrics-server-7c86f97b8d-fprlw | 72 |
| kube-system/coredns-5f5694d56b-zwzpz | 62 |

All three restarted at the 19:14 boot. `coredns` last state: exit 255 (Unknown);
`metrics-server` exit 2 (Error); `local-path-provisioner` exit 1 (Error).

Also **~500+ `zpool-*` completed pods** accumulated in `default` over 93 days.

### 1.5 Everything else

| Item | CPU |
|---|---|
| `ChatGPT` / `codex` (Electron app + renderers) | ~20–30% combined, transient |
| `uvicorn` (sysnotifier, :8321) | ~2% |
| `tzerp1-local-redis` | ~7–11% |
| `sysnotifier` composed-server (node, pid 131543) | ~2.5% |

Memory is healthy: **16 GB free**, swap entirely untouched (0 used of 84 GB).
Disk I/O: `nvme0n1` ~4.2 MB/s read / ~2.9 MB/s write, ~2% utilization — no bottleneck.

---

## 2. Bottom line

Nothing was pathologically wrong at the time of the scan. The one genuine efficiency
defect is the **O(n) seq-scan inside `intake.append()`** — the only real CPU sink —
and it degrades as the intake log grows. `verify`'s full-file parse of
`intake.ndjson` is the same shape of problem.

### Recommended fixes, in order of payoff

1. **Cache max-seq in memory** (or read it from the checkpoint) instead of rescanning
   the whole log per append. Turns 76 rescans into 1.
2. **Raise the RabbitMQ healthcheck interval** and use a lighter probe than
   `rabbitmq-diagnostics`, which boots a BEAM VM on every check.

### Incidental findings (not CPU-related, noted in passing)

- **126 GB** `dc/intake.quarantine.ndjson.pre-fix-20260916` still on disk, plus a
  **60 GB** `rc/` equivalent.
- `sau_keeper` logs show a steady stream of dropped SAU cells for out-of-vocabulary
  bucket tokens — `d`, `g`, `p`, `v`, `r`, `b` against an allowed set of `c`/`s`/`w`.
  A data bug, not a resource one.

---

## Appendix — commands used

```bash
top -b -n 1 | head -30
ps -eo pid,ppid,user,pcpu,pmem,etime,stat,comm --sort=-pcpu | head -25
cat /proc/loadavg
vmstat 1 3
iostat -x 1 2
docker stats --no-stream
docker events --since 15m --until 0s
docker inspect -f '{{.Config.Healthcheck}}' <container>
k3s kubectl get pods -A
```

Key measurements taken:

- `_scan_max_seq` over `rc/intake.ndjson`: **1.9 s**
- `append_many` of 10 events: **10.0 s**
- `iter_events` over `rc/`: **221,634 events in 1.4 s**
- `max_seq` over `rc/`: **1.0 s**
- Fork rate: **~100–166/sec** sustained (healthcheck churn)
- Context switches: **~70,000/sec**
- Threads on box: **4,315** (128 of them `erts_sched`)
- System-wide busy over 10 s: **33% of all cores**

---

*Audit performed 2026-10-29/30. Assessment only — no changes, kills, or edits made.*
