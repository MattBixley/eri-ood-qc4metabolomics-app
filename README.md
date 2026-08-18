# eri-ood-qc4metabolomics-app

OOD Batch Connect app for [QC4Metabolomics](https://github.com/stanstrup/QC4Metabolomics)
on eRI (first build / beta).

Modeled on [nesi/eri-ood-rstudio-server-app](https://github.com/nesi/eri-ood-rstudio-server-app),
adapted for a Shiny app that runs as a 4-container Apptainer stack instead of
a single `rserver` process. This is a first draft, built and tested from
scratch in this repo before any move to the `nesi` org - see "Known caveats"
before relying on it for anything beyond a demo/eval.

## What this app does

Submitting the form starts a normal (shared-node) SLURM job, sized by the
`partition`/`num_cores`/`num_mem` fields you pick, that:

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

## Using the dashboard

The tabs aren't all equally important - there's a clear path for a
first-time visit (per the upstream project's own docs,
[stanstrup.github.io/QC4Metabolomics](http://stanstrup.github.io/QC4Metabolomics/)):

1. **Contaminants** - start here. Shows a plot of the most frequently
   detected contaminants across whatever files have been processed so far,
   with a "minimum intensity" threshold slider. It auto-refreshes about
   every minute as new files come in, so it's the quickest way to check the
   pipeline is actually doing something.
2. **Track Compounds** (`TrackCmp`) - two sub-tabs:
   - *Compound Settings*: define specific analytes to watch over time (name,
     instrument, ionization mode, m/z, retention time). Nothing shows up in
     *Compound Stats* until at least one compound is defined here - seeing
     `No compounds defined. Quitting.` in `state/qc_process.log` is expected
     on a fresh session, not an error.
   - *Compound Stats*: the tracked values for whatever you configured, once
     data has come in.
3. **Warning Rules** (`Warner`) - threshold rules (e.g. "m/z deviation >
   X") that trigger an email alert when a new file violates them.

The rest (`Files`, `FileInfo`, `Productivity`, `Log`, `FileSchedule`,
`Debug`) are more operational/supporting views - file status, parsed
filename metadata, throughput, and internal logs.

## Importing your own data

This is a filesystem workflow, not a Shiny upload button - confirmed from
the project's own ["Moving raw files
automatically"](http://stanstrup.github.io/QC4Metabolomics/articles/file_move.html)
doc: the pipeline is driven by two plain text files, `raw_filelist.txt` and
`mzML_filelist.txt`, that `ms_converter` and `qc_process` each poll once a
minute (`script.sh.erb`'s `run_cron_replacement()` - see below).

Your session's own private data directory is:

```
<session output dir>/state/data/
```

e.g. `~/ondemand/data/sys/dashboard/batch_connect/dev/qc4metabolomics/output/<session-id>/state/data/`
(find `<session-id>` from the OOD dashboard's session card, or `ls -t` that
`output/` directory for the newest one).

To add your own files, from a terminal while the session is running (OOD's
Shell app, or SSH to whichever compute node the session landed on - shown
on the session card):

- **Already `.mzML`**: copy the file into `state/data/`, then append its
  full path as a new line to `state/data/mzML_filelist.txt` - `qc_process`
  picks it up within a minute.
- **Raw instrument files** (e.g. `.raw`): copy the folder into
  `state/data/`, append its path to `state/data/raw_filelist.txt` instead -
  `ms_converter` converts it to `.mzML` first, then writes the result into
  `mzML_filelist.txt` for you.

Filenames need to follow the mask this deployment's `settings_demo.env`
expects: `project_date_instrument_batchseq_mode_sampleid` (e.g. the seeded
demo file `Mcourse_20230509_Sold_031_pos_5A-0424.mzML`) - that's how the
dashboard extracts instrument/date/mode for filtering and the Contaminants/
TrackCmp tabs.

To confirm processing actually happened, check
`state/qc_process.log`/`state/qc_process_varlog/QC_cron.log` (or the
`ms_converter` equivalents) after a minute or two - a real run looks like
several `Modules/*/schedule.R` scripts running in sequence with only
benign R deprecation warnings, ending in the Contaminants module printing
`"Starting next batch of files"`. If it errors instead, that log is where
to look first.

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

## Schema initialization (`setup/init_db.R`)

`CREATE DATABASE` alone gives you an *empty* database - the app's own tables
(one set per enabled module, defined by `Modules/*/init_db_create.sql`) are
created by the project's `setup/init_db.R`, which `script.sh.erb` now runs
once via `apptainer exec` into the `qc_shiny` instance right after it starts
(that image is what has R, the `MetabolomiQCsR` package, and the `Modules/`
SQL files - `mariadb`'s image doesn't). Skipping this step doesn't fail the
job or even fail `wait_until_port_used` - shiny-server comes up and answers
HTTP requests just fine - but every visit to the app crashes with:

```
Error : No tables in the DB. Probably not done initializing.
```

which reads as "doesn't connect" from the OOD dashboard even though the
job, the containers, and the Shiny UI's HTTP listener are all fine. If this
resurfaces, check `<session dir>/state/init_db.log` first.

## Fixed: qc_process/ms_converter's cron didn't actually run

Both containers' own `/setup/cron_with_env.sh` entrypoint starts
Debian/Ubuntu's `cron`, which failed immediately:

```
Starting cron
seteuid: Invalid argument
```

Confirmed by hand this is cron's own startup behavior under a rootless user
namespace - it happens even with an *empty* crontab, so it's not about a
specific job's owning user (reinstalling the baked-in `root` crontab under
your own user doesn't help). Since each container only ever has ONE job on a
fixed 1-minute schedule, `script.sh.erb`'s `run_cron_replacement()` now just
extracts that one command from the image's baked-in
`/var/spool/cron/crontabs/root` and runs it in a plain loop instead of using
cron at all. Two things this needed, both confirmed by hand:

- a real bind-mounted `/var/log` (same ENODATA-on-`--writable-tmpfs`-overlay
  story as everywhere else in this README - the job's own `>> /var/log/*.log`
  redirect needs a real filesystem underneath).
- an explicit `--pwd` into the image's own app directory
  (`/srv/shiny-server/QC4Metabolomics` for `qc_process`, `/converter_scripts`
  for `ms_converter`). Without it, `apptainer exec` mirrors the *calling host
  process's* cwd inside the container - and since this script's own cwd is
  `$APP_SRC` (the project's GPFS checkout, which apptainer on this cluster
  transparently binds into every container), landing there by accident makes
  R's implicit `.Rprofile`-sourcing at startup try to bootstrap and download
  an entire `renv` environment from scratch (that GPFS checkout has its own
  *never-built* `renv.lock`/`.Rprofile`, distinct from the one baked into the
  image) instead of quietly activating the image's already-fully-installed
  one.

A follow-up real session initially showed both `qc_process.log` and
`ms_converter.log` completely empty (not even an error) - `run_cron_replacement()`
got hardened with `set +e` inside the loop (a single failed iteration was
silently killing it forever) and unconditional diagnostic logging of the
extracted command. Confirmed in a subsequent real session: `qc_process` ran
all five `Modules/*/schedule.R` scripts successfully against the seeded
demo data (a few minutes end-to-end, since Contaminants loads the full
xcms/Bioconductor stack), and `ms_converter` polls and exits cleanly every
minute (nothing to convert - the demo dataset is already `.mzML`).

## Fixed: the form always showed "1 node | 256 cores" regardless of selection

An earlier version of this app requested `--exclusive` in `submit.yml.erb`
(a whole node, no matter what `num_cores`/`num_mem` were set to) - which is
also why SLURM/OOD always reported the whole node's actual core count in the
session card, ignoring the form's selection entirely. `--exclusive` is gone;
`num_cores`/`num_mem`/`partition` (`form.yml`) are now real SLURM cgroup
limits, and the node is genuinely shared with other jobs.

That reintroduces the reason `--exclusive` was there in the first place:
Apptainer instances bind directly onto the node's real (shared) network
namespace, not a private per-job one, so a *fixed* port collides the moment
two sessions land on the same node at once. Shiny's port was already
dynamic (`before.sh.erb`'s `port=$(find_port)`) - mariadb's wasn't
(`settings_demo.env` hard-codes `MYSQL_PORT=3306`). `before.sh.erb` now
picks a second random `db_port` the same way, and `script.sh.erb` builds
`$SETTINGS_ENV` - a session-local copy of `settings_demo.env` with
`MYSQL_PORT` overridden - used everywhere instead of the original.
`mariadbd` itself needs an explicit `--port="${db_port}"` too: `MYSQL_PORT`
only tells *client* libraries what port to connect to, it doesn't tell the
server what to listen on. Confirmed end-to-end by hand (mariadb bound to a
random port via TCP, `init_db.R` and the Shiny UI both connecting through
`$SETTINGS_ENV`/`Renviron.site`, HTTP 200) before this landed.

## No login on the Shiny UI

Unlike the RStudio app (which auto-signs you in via a per-session password
and a PAM auth helper, because `rserver` requires that), `view.html.erb` here
is just a plain link. Shiny Server (the open-source edition this project
ships) has no built-in authentication at all - the only access control is
that the URL is only reachable through your own already-authenticated OOD
session/proxy.

## Known caveats (first build - check these before relying on this)

- **Confirmed working through a real OOD session** (2026-08-18): form
  submission, job templating, the reverse proxy, and Shiny's websocket-based
  reactivity all work end-to-end - `qc_process` successfully ran all five
  `Modules/*/schedule.R` scripts (Files, FileInfo, FileSchedule,
  Contaminants, TrackCmp) against the seeded demo data, and `ms_converter`
  polls cleanly every minute. Earlier revisions of this README said this
  was untested; it no longer is, though it's still worth treating as a
  first build rather than production-hardened.
- **Partition options** (`form.yml`'s `partition` select: `compute`,
  `interactive`, `hugemem`) were taken from this cluster's actual
  `sinfo`/`scontrol show partition` output on 2026-08-18 (`gpu`/`vgpu`
  deliberately excluded - this app has no GPU workload). Re-check that list
  if partitions are ever renamed/added/removed.
  - No custom `icon.png` yet - falls back to OOD's default app icon.
- `db_backup` got less scrutiny than the other four containers even in the
  original hand-validation this is based on (though it isn't started here at
  all - see above).
