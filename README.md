# cluster-status

A single-file, stdlib-only Python CLI that checks your SLURM clusters over
`ssh` and prints a colorful fairshare / queue / running-job dashboard. No
daemon, no server, no deps beyond Python 3 and an `ssh` config that can
reach your clusters without a password prompt.

```
cluster-status  2026-10-09 15:59:13
──────────────────────────────────────────────────────────
TOTAL   cpu  1666 core-hr/hr   gpu 18.57/hr (H100-eq)
  trace, last 3h (347 samples)
    cpu ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▂▃▄▄▅▇█▇▇▆▅▅▄▄  (8 - 3621 core-hr/hr)
    gpu ▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▁▂▂▂▂▁▁▁▁▁▁▁▁▁▁▁▁▃▄▆▆▆▆▇▇▇▇▇▇▇█  (0.29 - 18.82/hr)

cluster1
───────
  account               fairshare      24h       7d
  myaccount_cpu              0.534     1044     3254
  myaccount_gpu              0.093       81     4317
  running now
    myaccount_cpu          2 jobs      4 core-hr/hr cpu    0.00/hr gpu
  walltime, most urgent first (bar = elapsed/limit; array jobs grouped)
    CPU 12345678_* x2  ████████████████░░░░  82%  1h38m/2h00m      queued 10m before start
  queue  0 pending
```

## What it does

One `ssh` round-trip per cluster bundles `squeue` + `sshare` + `sreport` into
a single remote script, so a check is fast enough to run on a watch loop.
Per cluster, it shows:

- **Fairshare** per account, color-coded (green = healthy, red = depressed
  by recent usage), plus usage hours over the last 24h/7d.
- **Running jobs**: current burn rate per account (CPU core-hours/hr, GPU in
  full-GPU equivalents — a SLURM MIG slice like `1g.10gb` correctly counts
  as `1/7`, not `1`).
- **Walltime progress bars** for the jobs closest to hitting their time
  limit, with queue-wait time annotated. Array-job tasks (`123_1`, `123_2`,
  ...) are grouped into one row instead of flooding the list with clones,
  and each row is tagged `CPU`/`GPU` based on its actual GPU allocation.
- **Pending queue**: ETA for the next few jobs to start, computed entirely
  on the remote host (avoids local/remote timezone bugs).
- **A trace/sparkline** of total CPU and GPU burn rate over a rolling
  window, persisted to `~/.local/state/cluster-status/history.csv`.
- Clusters are listed busiest-first (by running-job count), so the cluster
  you're actually hammering right now shows up on top.

Clusters are listed most-heavily-used-first automatically.

## Setup

1. Make sure you can `ssh <host>` into each cluster without a password
   prompt (key-based auth, or a `~/.ssh/config` entry with the right
   `User`/`IdentityFile`). The remote side needs `squeue`, `sshare`,
   `sreport`, `date`, and `bash` — i.e. any normal SLURM login node.
2. Copy `cluster-status` onto your `PATH` and make it executable:
   ```bash
   cp cluster-status ~/.local/bin/
   chmod +x ~/.local/bin/cluster-status
   ```
3. Edit the two constants near the top of the file to your own hosts and
   SLURM accounts:
   ```python
   DEFAULT_CLUSTERS = ["cluster1", "cluster2"]
   DEFAULT_ACCOUNTS = ["myaccount_cpu", "myaccount_gpu"]
   ```
   (or skip editing and just pass `--clusters` / `--accounts` every time.)

## Usage

```bash
cluster-status                                   # one-shot check
cluster-status --clusters cluster1                # just one cluster
cluster-status --accounts myaccount_cpu           # just one account
cluster-status --watch 60                         # refresh every 60s
cluster-status --html status.html                 # also write a static HTML snapshot
cluster-status --history-hours 12                 # widen the trace window (default 3h)
cluster-status --no-record                        # don't log this run into the trace history
```

`--html` writes a static file each run — not a server. Open it in a browser
and reload, or pair `--watch` with `--html` in a background job if you want
it to stay current.

## Notes

- Everything is read-only: `squeue`, `sshare`, `sreport`, nothing that
  submits, cancels, or modifies jobs.
- GPU accounting assumes NVIDIA MIG-style gres names like
  `nvidia_h100_80gb_hbm3_1g.10gb` (7 "g" units per physical chip). Plain
  `gpu:<model>:<n>` gres is counted as `n` whole GPUs. Adjust `_gpu_equiv`
  if your cluster's gres naming differs.
- No external dependencies — just the Python 3 standard library.
