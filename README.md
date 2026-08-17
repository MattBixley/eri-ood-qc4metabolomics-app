# eri-ood-qc4metabolomics-app

OOD Batch Connect app for [QC4Metabolomics](https://github.com/stanstrup/QC4Metabolomics)
on eRI (first build / beta).

Modeled on [nesi/eri-ood-rstudio-server-app](https://github.com/nesi/eri-ood-rstudio-server-app),
adapted for a Shiny app that runs as a 4-container Apptainer stack instead of
a single `rserver` process. This is a first draft, built and tested from
scratch in this repo before any move to the `nesi` org - see "Known caveats"
before relying on it for anything beyond a demo/eval.

## What this app does

Submitting the form starts a whole-node (`--exclusive`) SLURM job that:

1. Starts `mariadb`, `qc_process` (cron file-watcher), `ms_converter`
   (raw -> mzML conversion) and `qc_shiny` (the Shiny UI) as Apptainer
   instances, using the project's own images at
   `/agr/persist/projects/2026_metabolomics/apps/QC4Metabolomics/sif/`.
2. Waits for the Shiny UI to answer on the port OOD assigned.
3. Gives you a "Connect to QC4Metabolomics" button that opens it.

`db_backup` (the fifth container in the project's own `run_qc.sh`) is
intentionally **not** started - it backs up the team's shared production
database, which doesn't apply to a private, per-session sandbox that goes
away when the session ends.

## Design choice: private per-session stack, not one shared team instance

QC4Metabolomics is really meant as a shared, continuously-running team QC
dashboard (one database, everyone's results) - not a personal sandbox. A
"connect everyone to one already-running shared instance" design would match
that better, but is architecturally a different kind of app (a thin
connector to a stack started separately, outside OOD, rather than something
that submits its own job) and needs someone to actually stand up and own that
always-on instance first.

This first build instead mirrors the RStudio app's model exactly: **every
session gets its own private stack** - its own MariaDB database and its own
copy of the project's demo dataset, staged under the job's own directory
(`session.staged_root`), not the shared project one. That means:

- concurrent sessions (different users, or the same user twice) never race
  each other over the same database or data files
- but also: each session starts from an empty QC history, not the team's
  actual accumulated results - it's a demo/eval sandbox, not a window onto
  production data

If/when this moves toward real team use, revisit this - the shared-instance
model is probably the right end state, this first build just isn't it.

## Why nothing here uses `--fakeroot`

Unlike the project's own `run_qc.sh`, this app never passes `--fakeroot` to
`apptainer instance start`. On eRI, no real user account has an
`/etc/subuid`/`/etc/subgid` range (only template accounts like
`cloud-user`/`podman`/`testuser` do), and `newuidmap`/`newgidmap` aren't
setuid-root either - so `apptainer instance start --fakeroot` silently dies
moments after printing "instance started successfully" (a one-shot
`apptainer exec --fakeroot ... whoami` misleadingly still works and prints
`root`, but a persistent instance never does). This was confirmed identically
on both a login node and a compute node under the same account, so it's a
cluster-wide config gap for eRI admins to fix, not a per-node quirk -
`run_qc.sh` would hit the exact same wall.

Until that's fixed, `template/script.sh.erb` runs every instance as your own
regular user instead, which needs three workarounds (each confirmed working
end-to-end - real `settings_demo.env`, mariadb provisioned, Shiny UI
returning HTTP 200 - before this app was written):

- **Real bind-mounted host directories**, not `--writable-tmpfs`'s overlay,
  for any path a container writes to outside its own data-dir binds (e.g.
  `/run/mysqld`, `/run`, shiny's log/lib dirs). Apptainer's `--writable-tmpfs`
  overlay upper layer throws ENODATA/EINVAL for a lot of writes (unix socket
  binds, chown to a uid other than your own) when it isn't backed by a real
  filesystem under a genuine (non-fakeroot) user namespace.
- **mariadb**: its container's normal entrypoint never runs here, so the
  script does first-run bootstrap by hand (`mariadb-install-db`, then
  creates the database/user and deletes the default anonymous user, which
  otherwise intercepts localhost auth ahead of the named user). Drops
  `--user=root` from `mariadbd` (we aren't root). Client tools
  (`mariadb-admin`, `mariadb`) need `--skip-ssl` too, not just the server.
- **qc_shiny**: its `/init` (s6-overlay) entrypoint hard-requires real root
  to chown files to the `shiny` user, so the script bypasses `/init` entirely
  and execs `shiny-server` directly, after substituting a `shiny-server.conf`
  with `run_as` pointed at your own user (shiny-server refuses to start
  otherwise) and an `Renviron.site` built from `settings_demo.env`
  (shiny-server resets the environment for each app's R session, so without
  this the app sees none of its DB config - normally done by the image's own
  s6 cont-init script, skipped along with the rest of `/init`).

See also the sibling `q/QC4Metabolomics/` module in
[eri-easyconfigs](https://github.com/nesi/eri-easyconfigs), which documents
the same findings for a standalone (non-OOD) module wrapper.

## No login on the Shiny UI

Unlike the RStudio app (which auto-signs you in via a per-session password
and a PAM auth helper, because `rserver` requires that), `view.html.erb` here
is just a plain link. Shiny Server (the open-source edition this project
ships) has no built-in authentication at all - the only access control is
that the URL is only reachable through your own already-authenticated OOD
session/proxy.

## Known caveats (first build - check these before relying on this)

- **Not yet tested through a real OOD web front-end.** Everything in
  `template/script.sh.erb` was validated by hand, piece by piece, outside of
  OOD (see the eri-easyconfigs conversation this was built from) - but the
  actual OOD form submission, job templating, and reverse-proxy/websocket
  behavior for Shiny's websocket-based reactivity have not been exercised
  end-to-end through a live OnDemand instance.
- **`bc_queue: interactive`** in `form.yml` is copied from the known-working
  RStudio app; `submit.yml.erb` additionally hard-codes
  `--partition=compute` (matching `run_qc.sh`) as a native SLURM arg, which
  may double up with (or need reconciling against) whatever partition
  `interactive` maps to in this cluster's `/etc/ood/config/clusters.d/slurm.yml`
  - check that on deploy.
  - No custom `icon.png` yet - falls back to OOD's default app icon.
- `qc_process`/`ms_converter`/`db_backup` got less scrutiny than
  `mariadb`/`qc_shiny` even in the original hand-validation this is based on
  - if file conversion/processing misbehaves in a session, check their logs
    under `<session dir>/state/*.log`.
