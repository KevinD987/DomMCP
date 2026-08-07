# DomMCP Operation Guide

Day-2 operations: console verbs, health/monitoring, backup & restore, and license renewal.

Console commands are issued at the Domino console. On Linux that is typically:

```bash
domino cmd "tell dommcp status"
```

On Windows, type the same `tell dommcp <verb>` at the live server console.

## Paths (Linux / container)

| What | Path |
|---|---|
| Add-in binary | `/opt/hcl/domino/notes/latest/linux/dommcp_addin` |
| Config NSF (source of truth) | `/local/notesdata/dommcp/dommcpcfg.nsf` |
| License file | `/local/notesdata/dommcp/dommcp-license.json` |
| Audit NSF | `/local/notesdata/dommcp/dommcpaudit.nsf` |
| Seed (first boot only) | `/local/notesdata/dommcp-default-config.json` |

On Windows the data directory is your Domino `notesdata` folder; the relative layout (`dommcp\dommcpcfg.nsf`,
`dommcp\dommcp-license.json`, `dommcp\dommcpaudit.nsf`) is the same.

## Start / stop / reload

```text
load dommcp_addin                 # start (no path → NSF=Truth auto-discovery)
tell dommcp quit                  # stop
tell dommcp reload-config         # re-read config from the NSF (NOT the license file)
```

> A new **license file** is only picked up on a full reload — `tell dommcp quit` + `load dommcp_addin` when
> the task is *not* in `ServerTasks=`, otherwise a restart of the whole server (see the next note).
> `reload-config` re-reads the config NSF, not the license file. Check afterwards that the `fingerprint=…`
> in `tell dommcp status` has changed.

> **If `dommcp_addin` is listed in `ServerTasks=`**, Domino may restart the task immediately after `quit` —
> the old and the new instance then fight over port 8088 and neither comes up properly. That looks like a
> broken binary but is only a race. When swapping the program file, **replace it and restart the whole
> Domino server**; never kill the task hard (`kill -9`), which leaves shared memory behind and blocks the
> next start.

## Console verbs

| Verb | Purpose |
|---|---|
| `status` | Current state, license summary, listener. |
| `show-metrics` | Request/throughput/error counters. |
| `reload-config` | Re-read config from the NSF. |
| `provision-admin <secret>` | Mint/rotate the first admin token (≥ 8 chars; write token = `<secret>-write`). |
| `DEBUG ON` / `DEBUG OFF` | Toggle verbose console logging. |
| `repair-design [<target.nsf> <source.nsf>]` | Install a **missing design** into the config/audit database — see below. Documents, ACL and icon are left untouched. |
| `quit` | Stop the add-in. |
| `help` | List available verbs. |

### „The database cannot be opened in the Notes client"

If the shipped databases were not in place at first start, DomMCP created **empty** ones: the server works,
but they hold no form and no view, so the client cannot open them. Since v0.0.615 the task says so at every
start:

```text
DomMCP WARNING: the config database '…/dommcpcfg.nsf' has NO DESIGN (no forms, no views).
```

Put the delivered database next to it as `<name>-master.nsf` (or hand over both paths explicitly) and run:

```text
tell dommcp repair-design
```

From v0.0.621 on, a **master template** (`dommcp/dommcpcfg.ntf`, `dommcp/dommcpaudit.ntf`) shipped next to
the databases prevents the situation altogether: the add-in creates them from the template on first start —
design without documents — and marks them to inherit, so an update is a newer `.ntf` plus Domino's Design
task.

## Health & monitoring

```bash
# Liveness + license at a glance (no auth):
curl -sS http://<host>:8088/healthz
# -> {"ok":true,"status":"up","license":{"edition":"...","status":"valid","read_only":false,...}}

# Tool inventory (auth):
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Monitoring tips:

- Poll `/healthz` for liveness; alert if `ok` is false or `license.read_only` flips to `true` unexpectedly.
- Watch `license.status` and the expiry — see **License renewal** below.
- Use `tell dommcp show-metrics` (or the `get_server_stats` / `get_server_health` tools) for counters.
- The audit NSF records every request; review it for access patterns and failures.

## Backup & restore

Back up two things together — they are the server's identity and entitlement:

1. **Config NSF** `dommcp/dommcpcfg.nsf` — holds tokens, grants, global settings (and a license copy).
2. **License file** `dommcp/dommcp-license.json`.

```bash
# Backup (server stopped or via Domino-consistent backup):
cp /local/notesdata/dommcp/dommcpcfg.nsf      /backup/dommcpcfg.nsf
cp /local/notesdata/dommcp/dommcp-license.json /backup/dommcp-license.json
# Optional: audit history
cp /local/notesdata/dommcp/dommcpaudit.nsf     /backup/dommcpaudit.nsf
```

Restore by copying the files back into `/local/notesdata/dommcp/` (preserve `notes:notes` ownership and
`640` on the license file), then `load dommcp_addin`.

> Treat the config NSF and license file as **secrets** — they contain token hashes and your signed license.
> Do not commit them to version control or share them publicly.

## License renewal

The license carries an expiry. On expiry the server degrades to **permanent read-only** until a new license
is installed; an unlicensed server runs a 7-day read-only evaluation and then blocks.

1. Check the current expiry: `curl -sS http://<host>:8088/healthz` → `license.expires_at_unix` /
   `valid_until`.
2. Obtain a renewed `dommcp-license.json` (new `license_code`, same domain).
3. Replace the license file (keep `notes:notes`, `640`). **Make sure you replace the file the server
   actually reads** — the path is resolved in this order: `LicenseFilePath` in the GlobalSettings document
   → the `DOMMCP_LICENSE_FILE` variable in `notes.ini` → the default `dommcp-license.json` **next to the
   config NSF** (`/local/notesdata/dommcp/`). If one of the first two is set, it wins, even if it points at
   a different directory. One command settles it:

   ```bash
   grep -i '^DOMMCP_LICENSE_FILE=' /local/notesdata/notes.ini    # empty output → the default applies
   ```

   Editing the wrong file is quiet, not loud: the config NSF also holds a `License` document, so the server
   stays licensed on the old terms and simply never picks up the new code.
4. Reload — **how depends on `ServerTasks=`** (see the note further up): if `dommcp_addin` is listed there,
   restart the whole server (`systemctl restart domino`, or restart the container); only if it is *not*
   listed is `tell dommcp quit` + `load dommcp_addin` the right pair.
5. Verify `license.status` = `valid` with the new expiry, and that `fingerprint=…` in `tell dommcp status`
   has actually changed. An unchanged fingerprint means the new file was not read.

**Renewal reminder:** set a calendar reminder ~30 days before `valid_until`. A renewal is a file swap plus a
reload — no reinstall, no data migration.
