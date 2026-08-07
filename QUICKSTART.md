# DomMCP Quickstart

This guide installs DomMCP on a Domino server, mints the first admin token, installs a license, and connects
a client. Pick the section for your platform, then continue with **First start → Admin → License → Client**.

Placeholders: `<host>` = your Domino server, `<token>` = a Bearer token you mint below, `<version>` = the
release version.

**Requirements:** HCL **Domino 12.0.2, 14.0 or 14.5** — one binary covers all three. Linux x86_64,
Windows x64, or the HCL domino-container.

All downloads are on the [Releases page](https://github.com/KevinD987/DomMCP/releases).

On Linux, match the binary to your **glibc** — check with `ldd --version | head -n 1`. The current build
(v0.0.671) needs **≥ 2.38** (Ubuntu 24.04, Debian 13, RHEL/Rocky 10); v0.0.272 still runs on **2.34**
(RHEL/Rocky 9, Ubuntu 22.04). **Below 2.34 — e.g. RHEL/Rocky 8 — nothing works**, because HCL's own
`libnotes.so` for Domino 14 requires glibc 2.34; that host cannot run Domino 14 at all. Use the container
add-on, a newer host OS, or ask us for a Domino-12.0.2-specific build (that runtime needs only glibc 2.17).

---

## 1) Install the add-in

### Linux

```bash
# Use the build that matches your glibc — check with: ldd --version | head -n 1
cp dommcp_addin-linux-x64-glibc2.38 /opt/hcl/domino/notes/latest/linux/dommcp_addin
chmod 755 /opt/hcl/domino/notes/latest/linux/dommcp_addin
chown notes:notes /opt/hcl/domino/notes/latest/linux/dommcp_addin
# Verify integrity first:
shasum -a 256 -c SHA256SUMS
```

### Windows

```powershell
# Copy the EXE into the Domino program directory (where nserver.exe lives), e.g.:
Copy-Item dommcp_addin-windows-x64.exe "C:\Program Files\HCL\Domino\dommcp_addin.exe"
# Verify integrity (compare against the entry in SHA256SUMS):
Get-FileHash "C:\Program Files\HCL\Domino\dommcp_addin.exe" -Algorithm SHA256
```

### HCL domino-container — option A: build-time Custom Add-on `.taz` (persistent, recommended)

The `.taz` is an HCL "Custom Add-on" tarball: register it when you **build** the container image. It installs
the binary **and** the `dommcp/` templates, and survives image rebuilds. It does **not** start the task and
does **not** ship a license.

```bash
# 1) place dommcp-<version>.taz on the build host (or host it over HTTPS)
# 2) add to the domino-container build command (sha256 is printed in addon-manifest.json):
#    -custom-addon=dommcp-<version>.taz#<sha256>
```

### HCL domino-container — option B: copy into a RUNNING container (`docker cp`)

For a quick install into an already-running container (e.g. a demo), copy the files in with **`docker cp`** —
the plain `cp` above runs on the host and does **not** reach into the container. `<c>` = your container name
(find it with `docker ps --format '{{.Names}}'`).

```bash
# Binary → program dir (NOTE: /opt is NOT persistent — survives restarts, not an image rebuild):
docker cp dommcp_addin-linux-x64-glibc2.38  <c>:/opt/hcl/domino/notes/latest/linux/dommcp_addin
docker exec -u 0 <c> sh -lc 'chown notes:notes /opt/hcl/domino/notes/latest/linux/dommcp_addin && chmod 755 /opt/hcl/domino/notes/latest/linux/dommcp_addin'

# Whole dommcp/ folder (config + audit templates) → persistent data dir:
docker cp dommcp                <c>:/local/notesdata/
# License next to them (see step 4 for issuing it):
docker cp dommcp-license.json   <c>:/local/notesdata/dommcp/dommcp-license.json
docker exec -u 0 <c> sh -lc 'chown -R notes:notes /local/notesdata/dommcp && chmod 640 /local/notesdata/dommcp/*'
```

> `docker cp <source-on-host> <container>:<path>` — left is the host, right is the container. If `-u 0` is
> blocked on your host, use `-u root`. With option B you've already done step 1b (the `dommcp/` copy) — skip it.

The console commands in the steps below run via `docker exec <c> domino cmd '<verb>'`. After the container is
up, continue with **First start** (`load dommcp_addin`).

---

## 1b) Provide the config NSF (binary installs — Linux & Windows)

A binary install needs its config NSF in the Domino **data** directory before the first load. The
**container `.taz` already includes one**; for a manual binary install, place it yourself.

### Recommended: ship the prebuilt DB templates (with design)

Every release carries **`dommcpcfg.ntf`** (config) and **`dommcpaudit.ntf`** (browsable audit views) as
assets — download both from the [Releases page](https://github.com/KevinD987/DomMCP/releases). These are
Domino **master templates**: they are **token-less and carry no secrets** — only the **design** (forms +
views). Put both files into the `dommcp/` subfolder of the data directory. On its first load the add-in
creates the two databases *from* these templates and records the design inheritance, so a later update is a
newer `.ntf` plus Domino's design task. You mint the first admin token in step 3.

Leave the templates out and the add-in still starts — but it then creates **empty** databases that the Notes
client cannot open. It says so at startup; `tell dommcp repair-design` fixes it afterwards.

```bash
# Linux — both templates belong in the dommcp/ subfolder of the data directory:
mkdir -p /local/notesdata/dommcp
cp dommcpcfg.ntf dommcpaudit.ntf /local/notesdata/dommcp/
chown -R notes:notes /local/notesdata/dommcp
```

```powershell
# Windows (data dir is the "Directory=" value in notes.ini, e.g. C:\Program Files\HCL\Domino\Data):
Copy-Item -Recurse dommcp "C:\Program Files\HCL\Domino\Data\dommcp"
```

> The container `.taz` already contains this `dommcp/` folder (installed on first container setup) — you only
> need this manual copy for a **binary** install.

> If an older install left a `dommcpcfg.nsf` in the data-dir **root**, remove it so the subfolder copy is used.

> `dommcpcfg.nsf` is design-only (zero documents). On the first load the add-in reports
> `grants_loaded=0` until you run `provision-admin` (step 3), which creates the admin grant + token in it.

### Alternative: JSON seed (no bundled NSF design)

Instead of the NSF you may drop the [`default-config.json`](default-config.json) JSON seed into the data
directory as `dommcp-default-config.json`. The add-in then builds the config NSF from it on first load
(`grants_loaded=1` with the `super-admin` grant template), but that NSF has **no design** (it looks empty in
the Notes client). Prefer the prebuilt NSF above unless you have a reason not to.

```bash
cp default-config.json /local/notesdata/dommcp-default-config.json   # Linux
```

---

## 2) First start (NSF = source of truth)

Start the add-in with **no path argument** so it auto-discovers its config NSF (the `dommcpcfg.nsf` you placed
in step 1b, or the `dommcp/dommcpcfg.nsf` subfolder). Confirm with `tell dommcp status`: the prebuilt NSF
reports `grants_loaded=0` (design-only — you mint the admin next, step 3); the JSON-seed path reports
`grants_loaded=1`.

```text
# On the Domino console (Linux: domino cmd "<verb>" / Windows: at the live console):
load dommcp_addin
```

Confirm it is listening:

```bash
curl -sS http://<host>:8088/healthz
# -> {"ok":true,"status":"up","license":{...}}
```

> Do **not** pass a config path to `load dommcp_addin` — a path forces file-based config and defeats
> NSF = source of truth.

---

## 3) Mint the first admin token

A fresh install is **token-less** on purpose. Bootstrap the first admin from the privileged server console:

```text
tell dommcp provision-admin <secret>        # <secret> must be >= 8 chars
```

This mints an admin token bound to the built-in super-admin grant:

- **Bearer token** = `<secret>`
- **write-intent token** = `<secret>-write` (needed for every write/design/admin call)

Re-running `provision-admin` rotates the token.

Smoke-test the token:

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <secret>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

---

## 3b) Sign the template databases (avoids a client warning)

The shipped template NSFs are signed by the **build** server. On your server the Notes client may warn
*"document has been modified or corrupted since signed"* when opening a config/audit document. Re-sign them
once with **your** server's ID — DomMCP does this itself (no Admin client needed):

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer <secret>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"dommcp_sign_database",
    "arguments":{"database_path":"dommcp/dommcpcfg.nsf","scope":"all","reason":"install sign",
      "change_ticket":"INSTALL","requested_by":"admin","idempotency_key":"sign-cfg",
      "write_intent_token":"<secret>-write"}}}'
# repeat with "database_path":"dommcp/dommcpaudit.nsf" and idempotency_key "sign-audit"
```

> Only needed once per copied template. Documents DomMCP creates/updates afterwards are auto-signed, so the
> warning does not recur.

---

## 4) Install a license

DomMCP verifies an offline Ed25519 license with a public key compiled into the binary. Without one it runs a
7-day read-only evaluation, then blocks. To activate:

1. Obtain your signed license file (`dommcp-license.json`) — it contains `license_code` + `license_domain`
   issued for your domain.
2. Drop it **into the `dommcp/` subfolder, next to the config NSF** (the add-in auto-finds it there — no env
   var needed):

   ```bash
   # Linux / container:
   cp dommcp-license.json /local/notesdata/dommcp/dommcp-license.json
   chown notes:notes /local/notesdata/dommcp/dommcp-license.json
   chmod 640 /local/notesdata/dommcp/dommcp-license.json
   ```

   ```powershell
   # Windows:
   Copy-Item dommcp-license.json "C:\Program Files\HCL\Domino\Data\dommcp\dommcp-license.json"
   ```

3. Reload so the license file is read — a plain `reload-config` does **not** re-read it.

   **Which command depends on whether `dommcp_addin` is listed in `ServerTasks=`:**

   ```bash
   grep -i '^ServerTasks' /local/notesdata/notes.ini
   ```

   *Not listed* (the state right after a first install):

   ```text
   tell dommcp quit
   load dommcp_addin
   ```

   *Listed* — which we recommend, so the task survives a server start:

   ```bash
   systemctl restart domino          # in a container: restart the container
   ```

   Do **not** use `quit` in that case: Domino starts the task again immediately, the old and the new process
   fight over port 8088, and the result looks like a broken program even with a perfectly good one.

4. Verify:

   ```bash
   curl -sS http://<host>:8088/healthz
   # license.status should be "valid", read_only false, with your edition + expiry
   ```

---

## 5) Connect a client

### Anthropic Claude (custom connector)

Claude connects over OAuth 2.1 + Streamable HTTP. You need DomMCP reachable over **public HTTPS** (a reverse
proxy that forwards `Host` and `X-Forwarded-Proto: https` to `http://<host>:8088`).

**Check that first — it is the step people miss**, and the failure shows up much later as a sign-in that
just does not complete:

```bash
curl -s https://dommcp.example.com/.well-known/oauth-protected-resource
# expect: {"resource":"https://dommcp.example.com/mcp", …}
#   http:// or an internal IP here = the proxy is not forwarding the headers.
```

If the proxy is out of your hands, set **`PublicBaseUrl`** in GlobalSettings instead — DomMCP then ignores
`Host`/`X-Forwarded-Proto` for address building entirely. Using `X-Forwarded-*`? Then `TrustedProxies` must
list the proxy's address, otherwise DomMCP deliberately discards those headers (anti-spoofing).

Then in claude.ai add a **custom connector** pointing at your HTTPS `/mcp` URL and complete the consent
(it asks for a DomMCP token).

### OpenAI / ChatGPT

Add DomMCP as an MCP connector pointing at the same HTTPS `/mcp` endpoint and authorize with a DomMCP token.

### n8n / scripts / any Bearer client

Point an MCP client (or plain HTTP) at `http://<host>:8088/mcp` with header
`Authorization: Bearer <token>`. See [`examples/n8n/`](examples/n8n/) for a complete workflow.

---

## API smoke calls

List tools:

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

Read view entries:

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":2,"method":"tools/call","params":{
    "name":"read_view_entries",
    "arguments":{"database_path":"names.nsf","view":"People","limit":25}}}'
```

Create a document (write — note the `write_intent_token`):

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":3,"method":"tools/call","params":{
    "name":"create_document",
    "arguments":{
      "database_path":"app.nsf",
      "form":"Memo",
      "fields":[{"name":"Subject","type":"text","value":"Hello from DomMCP"}],
      "write_intent_token":"<token>-write",
      "idempotency_key":"demo-001"
    }}}'
```

DQL query (PRO / ENTERPRISE):

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":4,"method":"tools/call","params":{
    "name":"query_documents_dql",
    "arguments":{"database_path":"app.nsf","query":"Form = '\''Memo'\''","return_mode":"ids_only","limit":10}}}'
```

> **Write/design/admin tools** always need `write_intent_token` = `<token>-write` and a unique
> `idempotency_key`. `preview_only` defaults to `false` (the call performs the real write).
