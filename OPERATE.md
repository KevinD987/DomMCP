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
| Audit NSF | `/local/notesdata/dommcpaudit.nsf` |
| Seed (first boot only) | `/local/notesdata/dommcp-default-config.json` |

On Windows the data directory is your Domino `notesdata` folder; the relative layout (`dommcp\dommcpcfg.nsf`,
`dommcp\dommcp-license.json`, `dommcpaudit.nsf`) is the same.

## Start / stop / reload

```text
load dommcp_addin                 # start (no path → NSF=Truth auto-discovery)
tell dommcp quit                  # stop
tell dommcp reload-config         # re-read config from the NSF (NOT the license file)
```

> A new **license file** is only picked up on a full reload: `tell dommcp quit` then `load dommcp_addin`.
> `reload-config` re-reads the config NSF, not the license file.

## Console verbs

| Verb | Purpose |
|---|---|
| `status` | Current state, license summary, listener. |
| `show-metrics` | Request/throughput/error counters. |
| `reload-config` | Re-read config from the NSF. |
| `provision-admin <secret>` | Mint/rotate the first admin token (≥ 8 chars; write token = `<secret>-write`). |
| `DEBUG ON` / `DEBUG OFF` | Toggle verbose console logging. |
| `quit` | Stop the add-in. |
| `help` | List available verbs. |

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
cp /local/notesdata/dommcpaudit.nsf            /backup/dommcpaudit.nsf
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
3. Replace `/local/notesdata/dommcp/dommcp-license.json` (keep `notes:notes`, `640`).
4. Full reload: `tell dommcp quit` then `load dommcp_addin`.
5. Verify `license.status` = `valid` with the new expiry.

**Renewal reminder:** set a calendar reminder ~30 days before `valid_until`. A renewal is a file swap plus a
reload — no reinstall, no data migration.
