# DomMCP — HCL Domino as an MCP server

DomMCP is an **HCL Domino server add-in** that exposes Domino NSF databases as a
[Model Context Protocol](https://modelcontextprotocol.io) (MCP) server over HTTP. It lets AI agents and
automation platforms **read, write, and build** Domino applications — documents *and* design elements
(forms, views, agents, pages, script libraries, ACL) — through a single governed endpoint.

This repository is a **closed-source binary distribution**. Source code is not published here.

> **Not read-only.** Earlier public builds were retrieval-only. Current DomMCP performs full read **and**
> write **and** design authoring **and** server administration, gated by a per-token permission model.

## What it does

- **Read / query:** discover databases, profile schema, read views, fetch documents, full-text search,
  **DQL** queries, and aggregations.
- **Write:** create / update / delete documents (typed fields, rich text, names/authors/readers).
- **Design authoring:** create and patch forms, subforms, views, folders, pages, outlines, framesets,
  navigators, agents (formula + LotusScript), script libraries, database scripts, shared fields, and
  image/file/stylesheet resources — driven entirely by DXL/parameters from the AI side.
- **Administration:** ACL and roles, group and person management, database lifecycle (create / compact /
  quota / delete), and scoped server-console commands.

## How it works

```
AI agent / automation  ──HTTP POST /mcp (JSON-RPC 2.0)──▶  DomMCP add-in  ──Notes C API──▶  Domino NSF
                          Bearer token                       (load dommcp_addin, :8088)
```

The add-in runs as a Domino server task (`load dommcp_addin`) and listens on port **8088**, serving
JSON-RPC 2.0 over `POST /mcp`. Configuration lives in an NSF on the server (**NSF = source of truth**);
a JSON config file is used only to seed a fresh install.

### Runtime layout (data directory)

Both the **config NSF** and the **license file** live in the **`dommcp/` subfolder** of the Domino data
directory — that is where the add-in looks for them (no env var or `notes.ini` entry needed):

```
<Domino data dir>/
  dommcp/
    dommcpcfg.nsf          # config NSF (tokens, grants, settings, license doc) — token-less when shipped
    dommcp-license.json    # signed Ed25519 license (drop it here; auto-found next to the config NSF)
    dommcpaudit.nsf        # audit log (created automatically next to the config NSF)
```

The add-in binary itself goes in the Domino **program** directory (where `nserver`/`nserver.exe` lives).

## Clients

DomMCP speaks standard MCP and works with:

- **Anthropic Claude** — via the OAuth 2.1 / Streamable-HTTP **custom connector** (claude.ai).
- **OpenAI / ChatGPT** — MCP-compatible connector.
- **n8n / scripts / any Bearer client** — plain `Authorization: Bearer <token>` against `POST /mcp`.

See [`QUICKSTART.md`](QUICKSTART.md) for connecting each client and
[`examples/`](examples/) for a worked n8n workflow.

## Security model

- **Token → Grant → scope.** Each Bearer token resolves (SHA-256) to one or more *grants*; a grant decides
  which databases, tools, views, and fields the token may touch, plus rate/size limits.
- **Write-intent.** Every write additionally requires a `write_intent_token` (the token secret + `-write`),
  so read tokens cannot mutate even if a write tool is invoked.
- **Run-as ACL.** A token can be mapped to a Domino user so reads are additionally enforced by the database
  ACL (a no-access user sees nothing).
- **Audit.** Every request is recorded to an audit NSF.

## License & editions

DomMCP uses an **offline Ed25519** license — no phone-home. The verifying public key is **compiled into the
binary**, so a license must be signed by the genuine key (a swapped public key cannot self-sign).

| Edition | Capability |
|---|---|
| **READ** | Read / query / aggregation tools. |
| **PRO** | READ plus write + design authoring. |
| **ENTERPRISE** | Full toolset incl. administration, higher limits. |
| **TRIAL** | Time-boxed evaluation. |

Runtime behavior without a valid license:

- **Unlicensed:** a **7-day read-only evaluation** window, after which the server is **blocked** until a
  license is installed.
- **Expired license:** the server degrades to **permanent read-only**.

License installation is described in [`QUICKSTART.md`](QUICKSTART.md).

## Platforms & artifacts

**Requirements:** HCL **Domino 14.5** (server task add-in). Linux x86_64 (current build: **glibc2.34** —
check yours with `ldd --version`), Windows x64, or the HCL domino-container.

DomMCP ships for **Linux** and **Windows**, plus an **HCL domino-container Custom Add-on** tarball. The current
build is in [`v0.0.272/`](v0.0.272) (previous: [`v0.0.249/`](v0.0.249)):

| Artifact | Path |
|---|---|
| Linux x86_64 add-in (glibc2.34) | [`v0.0.272/linux-x86_64/dommcp_addin-linux-x86_64-glibc2.34`](v0.0.272/linux-x86_64) |
| Windows x64 add-in | [`v0.0.272/windows-x64/dommcp_addin-windows-x64.exe`](v0.0.272/windows-x64) |
| HCL domino-container Custom Add-on | [`v0.0.272/dommcp-0.0.272.taz`](v0.0.272) (`-custom-addon=<file>.taz#<sha256>`) |
| DB templates (config + audit, in data-dir layout) | [`v0.0.272/dommcp/`](v0.0.272/dommcp) → copy the whole `dommcp/` folder into `/local/notesdata/` |
| Integrity checksums | `SHA256SUMS` in each folder |

**New in v0.0.272:** browsable audit database (auto-created, human-readable views), automatic re-signing of
every DomMCP-written document (no more "modified since signed" client warnings), new `dommcp_sign_database`
tool (bulk re-sign a copied template NSF with your server's ID), per-identity grant dedup, license doc kept
in sync with the active license, UTF-8 numeric-entity decode fix.

Always verify checksums after download (`shasum -a 256 -c SHA256SUMS`).

**glibc note (Linux):** the binary's glibc floor follows the libnotes of the target Domino. Match the
artifact to your host — detect with `ldd --version | head -n 1`.

## Documentation

| File | Contents |
|---|---|
| [`QUICKSTART.md`](QUICKSTART.md) | Install per platform (Linux / Windows / container), first start, admin bootstrap, license, client setup. |
| [`OPERATE.md`](OPERATE.md) | Console verbs, backup/restore, monitoring, license renewal. |
| [`TOOLS.md`](TOOLS.md) | The MCP tool catalog by category (read / write / design / admin / DQL). |
| [`v0.0.272/dommcp/`](v0.0.272/dommcp) | **Recommended fresh-install DB templates** — token-less, no secrets: `dommcpcfg.nsf` (config: Token/Grant/License/Settings forms + views) and `dommcpaudit.nsf` (browsable audit views). Copy the folder to `/local/notesdata/dommcp/` — see QUICKSTART step 1b. |
| [`default-config.json`](default-config.json) | Alternative JSON **seed** (token-less, `super-admin` grant template, placeholders; builds a config NSF *without* design). |
| [`CONFIG_EXAMPLE.json`](CONFIG_EXAMPLE.json) | Annotated config template (placeholders only). |
| [`examples/`](examples/) | n8n workflow + connector notes. |

## Contact

- Website: [it-dallmann.de](https://it-dallmann.de)
- Product page: [it-dallmann.de/domino-mcp](https://it-dallmann.de/domino-mcp/)
- Please test in your environment and open a GitHub issue for bugs, questions, or ideas.
