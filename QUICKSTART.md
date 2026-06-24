# DomMCP Quickstart

This guide installs DomMCP on a Domino server, mints the first admin token, installs a license, and connects
a client. Pick the section for your platform, then continue with **First start → Admin → License → Client**.

Placeholders: `<host>` = your Domino server, `<token>` = a Bearer token you mint below, `<version>` = the
release version.

---

## 1) Install the add-in

### Linux

```bash
# Use the provided Linux build (glibc2.34). Check your host glibc with: ldd --version | head -n 1
# (if your target Domino needs a different glibc floor, request a matching build)
cp dommcp_addin-linux-x64-glibc2.34 /opt/hcl/domino/notes/latest/linux/dommcp_addin
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

### HCL domino-container (Custom Add-on `.taz`)

The `.taz` is an HCL "Custom Add-on" tarball: register it when you **build** the container image. It installs
the binary and a clean, token-less seed config; it does **not** start the task and does **not** ship a license.

```bash
# 1) place dommcp-<version>.taz on the build host (or host it over HTTPS)
# 2) add to the domino-container build command (sha256 is printed in addon-manifest.json):
#    -custom-addon=dommcp-<version>.taz#<sha256>
```

After the container is up, continue with **First start** below (`load dommcp_addin`).

---

## 2) First start (NSF = source of truth)

Start the add-in with **no path argument** so it auto-discovers its config NSF (`dommcp/dommcpcfg.nsf`). On a
fresh server the first load seeds that NSF from the clean, token-less seed.

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

## 4) Install a license

DomMCP verifies an offline Ed25519 license with a public key compiled into the binary. Without one it runs a
7-day read-only evaluation, then blocks. To activate:

1. Obtain your signed license file (`dommcp-license.json`) — it contains `license_code` + `license_domain`
   issued for your domain.
2. Drop it next to the config NSF:

   ```bash
   # Linux / container:
   cp dommcp-license.json /local/notesdata/dommcp/dommcp-license.json
   chown notes:notes /local/notesdata/dommcp/dommcp-license.json
   chmod 640 /local/notesdata/dommcp/dommcp-license.json
   ```

3. Reload the add-in so the license file is read (a plain `reload-config` does **not** re-read the license
   file):

   ```text
   tell dommcp quit
   load dommcp_addin
   ```

4. Verify:

   ```bash
   curl -sS http://<host>:8088/healthz
   # license.status should be "valid", read_only false, with your edition + expiry
   ```

---

## 5) Connect a client

### Anthropic Claude (custom connector)

Claude connects over OAuth 2.1 + Streamable HTTP. You need DomMCP reachable over **public HTTPS** (a reverse
proxy that forwards `Host` and `X-Forwarded-Proto: https` to `http://<host>:8088`). Then in claude.ai add a
**custom connector** pointing at your HTTPS `/mcp` URL and complete the consent (it asks for a DomMCP token).

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
