# DomMCP v0.1.0

This is a major step beyond the v0.0.81 read-only line: DomMCP is now a full **read + write + design +
admin** MCP server for HCL Domino, on **Linux and Windows** and as an **HCL domino-container add-on**, with
**Claude** and **ChatGPT** connector support.

> Pre-release builds for this line are tagged `v0.1.0-rcN`. v0.0.81 remains the current "Latest" until v0.1.0
> is finalized.

## Highlights vs v0.0.81

### Write & build (was: read-only)
- **Document writes** — `create_document` / `update_document` / `delete_document` with typed fields
  (text, number, datetime, rich text, names/authors/readers).
- **Design authoring** — create and patch forms, subforms, views, folders, pages, outlines, framesets,
  navigators, agents (formula + LotusScript), script libraries, database scripts, shared fields, and
  image/file/stylesheet resources. Everything is DXL/parameter-driven from the AI side.
- **Component-level patching** — `dommcp_patch_design` edits a single field/action/column/event/entry/frame
  losslessly.
- **Database lifecycle** — create (empty or from template), compact, quota, delete.
- **ACL & directory** — `dommcp_set_database_acl` (entries + roles + admin server), group and person
  management, and run-as person tokens.

### New platforms
- **Windows** add-in binary (`dommcp_addin-windows-x64.exe`) — same MCP surface as Linux.
- **HCL domino-container Custom Add-on** (`dommcp-<ver>.taz`) — register at image build with
  `-custom-addon=<file>.taz#<sha256>`; installs the binary + clean token-less seed.
- **Linux** binaries are glibc-tagged so you can match your Domino host.

### More clients
- **Anthropic Claude** custom connector over **OAuth 2.1 + Streamable HTTP**.
- **OpenAI / ChatGPT** MCP-compatible connector.
- **n8n / Bearer** clients unchanged.

### Governance
- **Token → Grant → scope** model (databases, tools, views, fields, limits).
- **Write-intent token** required for every mutation (read tokens can't write).
- **Run-as ACL** enforcement and per-request **audit**.

### Licensing & editions
- Offline **Ed25519** license; verifying public key **compiled into the binary** (no self-sign).
- Editions **READ / PRO / ENTERPRISE / TRIAL**.
- Unlicensed = **7-day read-only evaluation**, then blocked; expired = **permanent read-only**.

## Artifacts

| File | Platform / use |
|---|---|
| `dommcp_addin-linux-x64-glibc<ver>` | Linux Domino add-in (match host glibc). |
| `dommcp_addin-windows-x64.exe` | Windows Domino add-in. |
| `dommcp-<ver>.taz` | HCL domino-container Custom Add-on. |
| `SHA256SUMS` / `SHA256SUMS-windows.txt` | Integrity checksums. |

## Integrity

```bash
shasum -a 256 -c SHA256SUMS
```

## Install

See [`QUICKSTART.md`](../QUICKSTART.md) for per-platform install (Linux / Windows / container), first start,
`tell dommcp provision-admin`, license installation, and client setup. Operations are covered in
[`OPERATE.md`](../OPERATE.md); the tool catalog is in [`TOOLS.md`](../TOOLS.md).

## Platform requirements

- **HCL Domino 14.5** target line.
- Linux x86_64: pick the binary matching your host glibc (`ldd --version | head -n 1`).
- Windows x64.
- For Claude/ChatGPT connectors: DomMCP reachable over public **HTTPS** (reverse proxy forwarding `Host` +
  `X-Forwarded-Proto: https`).

## Notes

- The internal build/version string of a binary (e.g. shown in `/healthz`) is the build number and may differ
  from the release tag.
- A pre-release is for evaluation. Please report issues on GitHub.
