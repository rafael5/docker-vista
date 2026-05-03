# FOIA VistA on IRIS Community Edition — Docker Install Guide

A step-by-step recipe for installing FOIA VistA on InterSystems IRIS
Community Edition under Docker, with all persistent state on the host
at `~/data/foia-iris/` so the install survives any Docker teardown.

Verified end-to-end on Linux (Debian-family, Docker 29.4.0) on
2026-05-03 with `DBA_VISTA_FOIA_20260409.zip`.

## What you get

- FOIA VistA running in the IRIS namespace `VISTA`
- RPC Broker on `localhost:9430` (CPRS-reachable)
- VistALink on `localhost:8001`
- IRIS superserver on `localhost:1972` (ODBC/JDBC, VS Code)
- IRIS Management Portal on `localhost:52773`
- All persistent state on host at `~/data/foia-iris/`
- Zero-loss restore: any Docker object can be wiped and rebuilt on
  top of `~/data/foia-iris/`

## What this guide does NOT cover

- The licensed/Enterprise IRIS path (`autoInstaller.sh -c` + `iris-files/` staging)
- GT.M / YottaDB
- RPMS
- Production multi-user use — Community Edition has a hard ~5-LU
  concurrency cap (see §9)

For the licensed-kit, YottaDB, GT.M, or RPMS paths, see the upstream
README's Quick Reference.

---

## 0. Prerequisites

- Docker installed and running
- ~6 GB free disk
- Outbound HTTPS to `foia-vista.worldvista.org` and
  `containers.intersystems.com`
- A GitHub account if you intend to contribute changes back via PR

## 1. Fork and clone the repo

```sh
gh repo fork WorldVistA/docker-vista --clone --remote
cd docker-vista
```

This sets `origin` to your fork and adds `upstream` for WorldVistA's
copy. If you don't have `gh` or don't plan to contribute, just clone
upstream directly:

```sh
git clone https://github.com/WorldVistA/docker-vista.git
cd docker-vista
```

The build assets we'll use live at `IRIS/docker-iris/`:

| File         | Purpose                                                       |
| ------------ | ------------------------------------------------------------- |
| `Dockerfile` | `FROM iris-community:latest-em` — copies `IRIS.DAT` into `/usr/irissys/mgr/VISTA/` and runs `iris merge` against `merge.cpf` |
| `merge.cpf`  | Creates the `VISTA` database, namespace, and `%DB_VISTA` resource; maps `%Z*`, `%ut`, `%Serenj*` globals/routines into VISTA; maps `HLTMP`, `TMP`, `UTILITY`, `XTMP`, `XUTL` to `IRISTEMP` |

## 2. Fetch the FOIA distribution

The latest builds are published at
[`https://foia-vista.worldvista.org/DBA_VistA_FOIA_System_Files/DBA_VISTA_FOIA_2026/`](https://foia-vista.worldvista.org/DBA_VistA_FOIA_System_Files/DBA_VISTA_FOIA_2026/).
At time of writing the most recent is **`DBA_VISTA_FOIA_20260409.zip`** (~919 MB).

```sh
cd IRIS/docker-iris
wget -c "https://foia-vista.worldvista.org/DBA_VistA_FOIA_System_Files/DBA_VISTA_FOIA_2026/DBA_VISTA_FOIA_20260409.zip"
unzip -o DBA_VISTA_FOIA_20260409.zip
ls -lh IRIS.DAT          # should be ~4.6 GB
rm -f DBA_VISTA_FOIA_20260409.zip
cd ../..
```

> **Important:** paste the URL on a single line. If your shell wraps
> the paste, the second half becomes a separate command and `wget`
> 404s on a truncated path.
>
> **Important:** delete the `.zip` after unzipping. The Dockerfile
> only uses `IRIS.DAT`; leaving the zip in the build context sends
> ~919 MB of unused data to the daemon and slows the build.
>
> **One-time use only:** this `IRIS.DAT` is consumed by the §3 build.
> After §7 the canonical copy lives at
> `~/data/foia-iris/mgr/VISTA/IRIS.DAT` and you can `rm -f IRIS/docker-iris/IRIS.DAT`
> to reclaim 4.6 GB.

## 3. Build the image

```sh
docker build -t foia IRIS/docker-iris/
```

What this does:

1. Pulls `iris-community:latest-em` (~3 GB, first time only)
2. Creates `/usr/irissys/mgr/VISTA/` owned by user `51773` (the
   `irisowner` user inside the Community image)
3. Copies your `IRIS.DAT` and `merge.cpf` into the image
4. Starts IRIS, runs `iris merge IRIS /tmp/merge.cpf` (creates the
   VISTA database, namespace, and global mappings), stops IRIS

This is the *structural* setup only. The RPC Broker, VistALink, and
Taskman are **not** running yet — that's §5.

Expected build time: ~2 min after the base pull.

## 4. Run the container

```sh
docker run --name foia -d \
    -p 1972:1972 -p 52773:52773 \
    -p 9430:9430 -p 8001:8001 \
    foia
```

| Port  | Purpose                                       | Listening after step |
| ----- | --------------------------------------------- | -------------------- |
| 1972  | IRIS superserver (ODBC/JDBC, VS Code)         | 4                    |
| 52773 | IRIS Management Portal                        | 4                    |
| 9430  | RPC Broker (CPRS)                             | 5                    |
| 8001  | VistALink                                     | 5                    |

Ports 9430 and 8001 are published *now* even though the listeners
don't bind them until §5 — this avoids needing a container restart
with new port flags later.

**First-time Mgmt Portal login.** User `_SYSTEM`, password `SYS`
(Community image default). You'll be forced to change it on first
login; VS Code IRIS extensions also won't connect until you do.

## 5. Run the FOIA post-install

This step:
- Recompiles the routines that came in via `IRIS.DAT`
- Runs `^pvPostInstall` (DSI* cleanup, etc.)
- Runs `KBANTCLN` to initialize Taskman cleanly
- Saves a `^ZSTU` routine in `%SYS` that auto-starts the RPC Broker,
  Taskman, and VistALink on every boot

The Community Dockerfile doesn't include `Common/`, so we copy the
two routine sources in by hand:

```sh
docker cp Common/pvPostInstall.m foia:/tmp/
docker cp Common/KBANTCLN.m      foia:/tmp/
```

Then drive `iris session` from the host. **Run this entire block as
a single heredoc** so the routine load + execute happens in one IRIS
session:

```sh
docker exec -i foia iris session IRIS -U VISTA <<'END'
W "=== Recompiling routines ===",!
D Compile^%R("*.*")
D Compile^%R("%*.*")
W "=== Loading pvPostInstall ===",!
D $SYSTEM.Process.SetZEOF(1)
ZR  ZS pvPostInstall
S F="/tmp/pvPostInstall.m"
O F U F ZL  ZS pvPostInstall C F
W "=== Running pvPostInstall ===",!
D ^pvPostInstall
W "=== Loading KBANTCLN ===",!
S F="/tmp/KBANTCLN.m"
ZR  ZS KBANTCLN
O F U F ZL  ZS KBANTCLN C F
W "=== Cleaning Taskman ===",!
S U="^"
D GETENV^%ZOSV S UCI=$P(Y,U),VOL=$P(Y,U,2)
D START^KBANTCLN(VOL,UCI,999,"SANDBOX","SANDBOX.OSEHRA.ORG",1)
W "=== Done ===",!
HALT
END
```

You should see a long `INITIALIZATION COMPLETED IN 0 SECONDS` block
from KBANTCLN listing every Kernel Taskman option, ending with
`Routine ZUONT was renamed to ZU` and `=== Done ===`.

Then save the boot-autostart routine in `%SYS`:

```sh
docker exec -i foia iris session IRIS -U %SYS <<'END'
ZR  ZS ZSTU
ZSTU	;Boot up stuff
	;
	J ZISTCP^XWBTCPM1(9430):"VISTA"
	;
	J ^ZTMB:"VISTA"
	;
	J START^XOBVLL(8001):"VISTA"
	QUIT
ZS ZSTU
HALT
END
```

> **Tab indentation matters.** The four lines under `ZSTU` MUST be
> tab-indented (not spaces). The heredoc above preserves the tabs;
> if you retype it, use literal tabs.

Restart the container so `^ZSTU` fires on boot:

```sh
docker restart foia
```

## 6. Verify

After ~25 s the autostart should have all four listeners up:

```sh
sleep 25
docker exec foia ss -ltn | grep -E ':(1972|52773|9430|8001)'
```

You should see four `LISTEN` lines. Also confirm the autostart fired:

```sh
docker exec foia tail -20 /usr/irissys/mgr/messages.log
```

Expected lines from this boot:
- `Licensed for N users.`
- `Executing ^ZSTU routine`
- `Starting TASKMGR`

To open an interactive VistA terminal:

```sh
docker exec -it foia iris session IRIS -U VISTA
VISTA> D ^XUP                ; Kernel signon
```

Default FOIA user from the upstream kit: `fakedoc1` / `1Doc!@#$`. Full
list at `^VA(200,`.

> If `iris session` returns `<LICENSE LIMIT EXCEEDED>`, see §9.

## 7. Migrate to host-bind volumes (zero-loss persistence)

Up to this point everything lives in the container's writable layer.
`docker rm -f foia` would erase the install. To survive any teardown,
move all persistent state to the host at `~/data/foia-iris/` and
bind-mount it back.

```sh
# Stop cleanly so journals flush
docker stop foia

# Export everything that needs to persist
mkdir -p ~/data/foia-iris
docker cp foia:/usr/irissys/mgr      ~/data/foia-iris/
docker cp foia:/usr/irissys/iris.cpf ~/data/foia-iris/iris.cpf

# Set ownership to UID 51773 (irisowner inside the Community image)
# via a privileged helper container — no host sudo needed
docker run --rm --user 0 \
    -v ~/data/foia-iris:/data alpine \
    chown -R 51773:51773 /data

# Replace the running container with a volume-backed one
docker rm foia
docker run --name foia -d \
    -v ~/data/foia-iris/mgr:/usr/irissys/mgr \
    -v ~/data/foia-iris/iris.cpf:/usr/irissys/iris.cpf \
    -p 1972:1972 -p 52773:52773 -p 9430:9430 -p 8001:8001 \
    foia
```

After ~25 s the listeners are back up — re-run the §6 verification.

**What's on the volume:**

| Path                                  | Purpose                                        |
| ------------------------------------- | ---------------------------------------------- |
| `~/data/foia-iris/iris.cpf`           | IRIS config — VISTA namespace + mappings (from `merge.cpf`) |
| `~/data/foia-iris/mgr/VISTA/IRIS.DAT` | The VistA database (~4.6 GB)                   |
| `~/data/foia-iris/mgr/IRIS.DAT`       | `%SYS` database — your `^ZSTU` lives here      |
| `~/data/foia-iris/mgr/irissecurity/`  | Security database                              |
| `~/data/foia-iris/mgr/irisaudit/`     | Audit database                                 |
| `~/data/foia-iris/mgr/journal/`       | IRIS recovery journals                         |
| `~/data/foia-iris/mgr/messages.log`   | IRIS log; survives `docker rm`                 |

After this step, `~/data/foia-iris/` is the system of record. You can
`rm -f IRIS/docker-iris/IRIS.DAT` to reclaim 4.6 GB from the build
context — restores in §8 read from `~/data/foia-iris/`.

## 8. Restoring after teardown

The volume contents alone are sufficient to bring the install back,
no matter what Docker objects survive.

**Container removed (most common — `docker rm -f foia`):**

```sh
docker run --name foia -d \
    -v ~/data/foia-iris/mgr:/usr/irissys/mgr \
    -v ~/data/foia-iris/iris.cpf:/usr/irissys/iris.cpf \
    -p 1972:1972 -p 52773:52773 -p 9430:9430 -p 8001:8001 \
    foia
```

**Container *and* the `foia` image both removed.** The volume's
`iris.cpf` already declares the VISTA namespace (the `iris merge`
from §3 is persisted on disk), so you don't need a custom image —
the upstream IRIS Community image works directly:

```sh
docker run --name foia -d \
    -v ~/data/foia-iris/mgr:/usr/irissys/mgr \
    -v ~/data/foia-iris/iris.cpf:/usr/irissys/iris.cpf \
    -p 1972:1972 -p 52773:52773 -p 9430:9430 -p 8001:8001 \
    containers.intersystems.com/intersystems/iris-community:latest-em
```

The image just provides the IRIS binaries; everything VistA-specific
comes from the volume.

**`docker system prune -a` (everything wiped including the base image).**
Same recipe as above — Docker re-pulls `iris-community:latest-em`
automatically on first `docker run` (~3 GB, one-time).

**Lost the host data?** No restore — start over from §2. Back the
directory up regularly (recipe below).

### Verifying a restore

```sh
sleep 25
docker exec foia ss -ltn | grep -E ':(1972|52773|9430|8001)'
docker exec foia tail -5 /usr/irissys/mgr/messages.log
```

The `messages.log` tail should include the current boot's
`Executing ^ZSTU routine` and `Starting TASKMGR`.

### Backups

`~/data/foia-iris/` is owned UID `51773` mode `700`, so `tar` from
your host user fails. Run backup through a helper container with
matching UID:

```sh
docker stop foia
docker run --rm --user 0 \
    -v ~/data/foia-iris:/data:ro \
    -v ~/data:/out \
    alpine tar czf /out/foia-iris-$(date -u +%Y%m%d).tgz -C / data
docker start foia
```

Stopping first guarantees a consistent journal state in the snapshot.

### Inspecting host data

The same permissions block `du`/`ls`. Use a helper container:

```sh
docker run --rm -v ~/data/foia-iris:/data alpine du -sh /data /data/mgr/VISTA
```

## 9. Community Edition license cap (important caveat)

`iris-community` is licensed for ~5 concurrent **logical users (LU)**.
Once `^ZSTU` runs at boot the slot budget is consumed:

| Consumer                                          | ≈ slots |
| ------------------------------------------------- | ------- |
| IRIS internals (System Monitor, WorkQueue daemon) | 1–2     |
| RPC Broker (`ZISTCP^XWBTCPM1`)                    | 1       |
| Taskman (`^ZTMB` + spawned `ZTMS*` workers)       | 2+      |
| VistALink (`START^XOBVLL`)                        | 1       |

That is at-or-over the cap. Consequences:

- A *new* `iris session` lands `<LICENSE LIMIT EXCEEDED>` and
  `messages.log` records `License limit exceeded N times since instance start`
- The image's healthcheck probes IRIS via a session, so `docker ps`
  may show `(unhealthy)` even though the listeners are reachable.
  The four published ports are the source of truth, not the Docker
  health flag.
- A single CPRS / RPC client from outside *can* connect, but only
  one or two simultaneously
- The instance itself is healthy; `iris list` reports `state: warn`,
  not `down`

**Freeing a slot for interactive shell debugging.** From a `%SYS`
session: `D STOP^%ZTMSH` halts Taskman. Or stop the XWB listener
job. Restart the container afterward to bring them back.

For sustained multi-user use, switch to a licensed IRIS install
(not covered by this guide).

## 10. Stop and clean up

```sh
docker stop foia            # SIGTERM → IRIS shuts down cleanly
docker rm foia              # remove container; data on volume persists
```

To completely wipe and start over, also delete the host data (mode 700,
needs a helper container):

```sh
docker run --rm --user 0 -v ~/data/foia-iris:/data alpine \
    rm -rf /data/mgr /data/iris.cpf
rmdir ~/data/foia-iris
docker rmi foia
```

---

## Troubleshooting

- **`wget` returns 404** — URL was line-wrapped on paste. Re-run with
  the URL on a single line, double-quoted, with `-c` to resume the
  partial download.
- **Build context huge / build slow** — the FOIA `.zip` is still in
  `IRIS/docker-iris/`. Delete it; only `IRIS.DAT` is needed.
- **CPRS / VistALink doesn't connect after build** — §5 wasn't run,
  or `^ZSTU` didn't fire on the restart. Check `messages.log` for
  `Executing ^ZSTU routine` near the most recent boot timestamp.
- **`<LICENSE LIMIT EXCEEDED>` on `iris session`** — see §9. Not a
  bug; expected once Taskman + listeners are up.
- **`docker ps` says `(unhealthy)`** — same root cause as above; the
  external listeners are the source of truth.
- **First Mgmt Portal login refuses to continue** — change the
  `_SYSTEM` password to something other than `SYS`. Required before
  VS Code or any other portal-authenticated tool will connect.
- **Routines don't recompile / `^pvPostInstall` errors** — make sure
  the `docker cp` commands in §5 placed the files at `/tmp/` *before*
  the heredoc runs, and that the heredoc was typed/pasted with
  literal `<<'END'` (single-quoted) so `$SYSTEM` and `$P` aren't
  shell-expanded.

---

## What this guide diverges from upstream

The bare `IRIS/docker-iris/Dockerfile` does the structural CPF merge
but stops there. This guide adds two layers on top:

1. **§5 FOIA post-install** — recompile + `^pvPostInstall` + `KBANTCLN`
   + `^ZSTU` autostart. The licensed-kit path's `Common/pvPostInstall.sh`
   does the same work but isn't wired into the Community Dockerfile.
2. **§7 host-volume migration** — moves all state to `~/data/foia-iris/`
   so the install survives any Docker teardown.

Both are candidates for upstreaming back into `IRIS/docker-iris/` once
proven across more environments.
