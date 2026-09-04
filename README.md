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

### ⚠️ Prerequisite for the hosted connectors: public HTTPS

The claude.ai and ChatGPT connectors need DomMCP reachable over **public HTTPS**, behind a reverse proxy
**you run**, forwarding `Host` and `X-Forwarded-Proto: https`. DomMCP derives the OAuth redirect URLs, the
`issuer` and the `resource_metadata` hint in its `WWW-Authenticate` challenge from those two headers — if
they do not arrive correctly, it builds addresses that are unreachable from outside and the sign-in fails
with an error that does not point at the cause.

Verify it in one call, **from outside**, through the public name:

```bash
curl -s https://dommcp.example.com/.well-known/oauth-protected-resource
# expect: {"resource":"https://dommcp.example.com/mcp", …}
#   scheme MUST be https, host MUST be your public name — an http:// or an internal IP
#   means the proxy is not passing the headers through.
```

Ready-made nginx and Caddy configurations plus a full verification checklist ship with the handbook.
A Bearer client (n8n, a script) does **not** need any of this — it can talk to `http://<host>:8088/mcp`
directly on the internal network.

See [`QUICKSTART.md`](QUICKSTART.md) for connecting each client and
[`examples/`](examples/) for a worked n8n workflow.

## Security model

- **Token → Grant → scope.** Each Bearer token resolves (SHA-256) to one or more *grants*; a grant decides
  which databases, tools, views, and fields the token may touch, plus rate/size limits.
- **Write-intent.** A token carries a separate write-intent hash; without it, writes are refused even when a
  write tool is called. Over MCP HTTP the value is **derived server-side from the same session** — the client
  never sends it. It is therefore a per-token capability flag, not a second factor the caller supplies.
- **Run-as ACL.** A token can be mapped to a Domino user so reads are additionally enforced by the database
  ACL (a no-access user sees nothing).
- **Audit.** Every request is recorded to an audit NSF.

## License & editions

DomMCP uses an **offline Ed25519** license — no phone-home. The verifying public key is **compiled into the
binary**, so a license must be signed by the genuine key (a swapped public key cannot self-sign).

| Edition | Capability |
|---|---|
| **Essentials** | Free forever. Explore a Domino estate: which databases exist, how they are built, their views and fields, and the state of the server. |
| **Analyst** | Read, query and analyse: documents, DQL, aggregations, log analysis, bulk export, effective access rights, design inspection. |
| **PRO** | Analyst plus write and design authoring. |
| **ENTERPRISE** | Full toolset incl. administration, higher limits. |
| **Trial** | 14 days of the full PRO feature set. |

Runtime behavior without a valid license:

- **Unlicensed or expired:** the server settles into **Essentials**, permanently. Nothing is deleted and
  nothing is locked.

There is no hard block and no evaluation deadline. What separates the free tier from a paid one is the tool
ceiling, not a timer — documents, DQL and aggregations begin at Analyst.

License installation is described in [`QUICKSTART.md`](QUICKSTART.md).

## Platforms & artifacts

**Requirements:** HCL **Domino 12.0.2, 14.0 or 14.5** (server task add-in) — one binary covers all three,
verified on all three. Linux x86_64, Windows x64, or the HCL domino-container.

**All downloads are on the [Releases page](https://github.com/KevinD987/DomMCP/releases).** They used to sit
in version folders in this repository; those folders are gone — every artifact, including the older versions,
is now a release asset. That keeps a clone small and gives each download a stable URL and a checksum.

### Current release — [v0.0.1278](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.1278)

Linux and Windows come from the **same commit** (`ea5d7cc`) in this release. Each binary ships with its own
`manifest-*.json` naming that commit and the artifact's SHA-256, so the claim is checkable rather than
promised.

| Artifact | Asset |
|---|---|
| **Linux x86_64 add-in** (glibc ≥ 2.38) | `dommcp_addin` |
| **Windows x64 add-in** | `dommcp_addin.exe` |
| **HCL domino-container Custom Add-on** | `dommcp-0.0.1278.taz` (`-custom-addon=<file>.taz#<sha256>`) |
| **DB master templates** (config + audit, token-less) | `dommcpcfg.ntf`, `dommcpaudit.ntf` |
| **Handbook** (German / English, PDF) | `DomMCP-Installationsanleitung.pdf`, `DomMCP-Installation-Guide.pdf` |
| Build provenance | `manifest-linux.json`, `manifest-windows.json`, `addon-manifest.json` |
| Integrity checksums | `SHA256SUMS`, `SHA256SUMS-windows.txt` |

Always verify after download: `shasum -a 256 -c SHA256SUMS`.

**What changed.** This release is mostly about speaking the MCP specification exactly, because a client that
disagrees with the server on a field name does not report a mismatch — it simply stops.

- `server/discover` now uses the field names the specification prescribes, and **every** result in the
  modern protocol era carries `resultType`. A client that reached the modern era previously got a formally
  valid answer it could not accept, and went quiet after a single successful call.
- All published protocol revisions are accepted (`2024-11-05` through `2026-11-25`), and `initialize`
  answers with the version the client asked for instead of a fixed one.
- The OAuth protected-resource metadata now advertises `scopes_supported`, which a client reads between the
  token exchange and its first call.
- **View access answers one question.** `list_views`, `profile_database` and `read_view_entries` used to
  disagree: a view could be reported as readable and then be refused. They now share a single rule, the
  right to read views is settable and visible through the admin tools, and a view outside your scope stays
  hidden rather than appearing readable.
- The request log records every call, including the ones the server refuses — a rejected request used to
  leave no trace at all, which made "nothing arrived" and "we turned it away" look identical.

> **Windows is current again.** It had not been since v0.0.809: a POSIX-only header had been breaking the
> MSVC build for three weeks, followed by two more of the same family once that was cleared. The Windows
> build now runs on every change rather than only when a release is cut, so the next such break surfaces
> with the change that causes it instead of three weeks later.

### ⚠️ Check your glibc first (Linux)

The add-in links against the libnotes of the target Domino, and that sets a **minimum glibc**. Detect yours
with `ldd --version | head -n 1`, then:

| Distribution | glibc | v0.0.1236 (≥ 2.38) | v0.0.272 (≥ 2.34) |
|---|---|---|---|
| Ubuntu 24.04 LTS, Debian 13, RHEL/Rocky/Alma 10 | 2.38–2.41 | ✅ | ✅ |
| Ubuntu 22.04 LTS | 2.35 | ❌ | ✅ |
| RHEL / Rocky / Alma 9 | 2.34 | ❌ | ✅ |
| RHEL / Rocky / Alma 8 | 2.28 | ❌ | ❌ |

**For glibc below 2.34 there is no working binary, and for Domino 14 there cannot be one.** The floor is not
ours to choose: HCL's own `libnotes.so` — the library the add-in links against — requires it. Measured with
`objdump -T` on the shipped runtimes:

| Domino | `libnotes.so` requires |
|---|---|
| 12.0.2 | GLIBC_2.17 |
| 14.0 | GLIBC_2.34 |
| 14.5 | GLIBC_2.34 |

So on RHEL/Rocky/Alma 8 (glibc 2.28), **Domino 14 itself does not run** — with or without DomMCP. An earlier
version of this page pointed glibc-2.28 hosts at v0.0.272 and offered to build against an older glibc; both
were wrong, and we corrected them rather than leave them standing.

If you run **Domino 12.0.2** on such a host, a build is possible — that runtime only needs glibc 2.17. Ask,
and we will produce a 12.0.2-specific artifact. For Domino 14 the answer is the container add-on, or a newer
host OS.

The container add-on sidesteps the issue entirely: the `.taz` runs inside the HCL domino-container image.

### Older releases

[v0.0.809](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.809),
[v0.0.671](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.671),
[v0.0.650](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.650),
[v0.0.615](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.615) (Linux only),
[v0.0.272](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.272) and
[v0.0.249](https://github.com/KevinD987/DomMCP/releases/tag/v0.0.249) remain available. Use v0.0.272 only if
your glibc rules out the current build — it is many months behind.

**New in v0.0.1236** — three months of work, and most of it is about answers you can trust: what the
server hands back, and what it now refuses to do.

*Rich text is readable.* Until this release, a rich text item was skipped on **every** read path — a
document with a contract clause came back looking exactly like one without it, and nothing said so. The
reader existed; it was simply never wired up. Line breaks were lost in both directions as well, on writing
and on reading, by two separate defects that hid each other. Expect responses to carry content that was
always there: if you want narrow answers, project the fields you need.

*The server stays up.* A handful of DXL payloads could take down the whole Domino server process — not the
add-in, the server — and one of them hung a worker thread forever with no crash, no log and no recovery.
They are bugs in HCL's DXL importer, reachable by anyone with design write access; the fix is ours: the
payloads are refused **before** the import, each with a message naming what is wrong and which route does
work. A second family sat in our own code, where an oversized paragraph, a long query term or a long log
line overflowed a recursive regex — and that one needed only a **read-only** token. All of them are now
linear scanners rather than size limits, so nothing valid was made invalid to buy the fix.

*Writes no longer truncate quietly.* Several places in the native writer described a record length in a
16-bit field and cut the payload once it grew past that: a form body, an action bar, a compiled selection
formula. A truncated formula can still be valid and simply select different documents — a view that looks
complete and is not. Oversized input is now rejected with the limit and the way around it, instead of being
silently shortened and reported as success. The same rule reached the tools: a script library too large for
its storage is refused *before* the write, with the real reason, rather than leaving an empty note behind
and advising you to try again.

*An answer no longer claims more than was checked.* The audit chain reported `chain_intact: true` over zero
verified events, and an audit database that could not be opened looked like a clean one. An unreadable ACL
was reported as an empty ACL — on a rights question, the most expensive kind of wrong answer. "Cannot be
opened" was reported as "has no design" and as "not found". A dry run reported a compile result it never
computed, and one reported a rejection that never happened. Each of these now says which of the two it is,
or omits the verdict entirely — because "not checked" is not "checked and fine".

*Tool schemas a model can actually follow.* Measured against a local model rather than assumed: an argument
offered but not required gets left out, the call fails, and the model repeats it verbatim until it gives
up. Advice in the error text does not steer a weak model; the schema does. So the canonical argument is now
required (aliases still work, they are just no longer advertised), arrays of objects declare their keys, and
a closed vocabulary is an `enum` instead of a sentence — each list counted against the implementation, which
turned up several documented values that no code ever accepted. On the measured task set this moved whole
categories from "never succeeds" to "succeeds in two calls".

*Retrieval.* `dommcp_rag_sync` hands documents to an index page by page with stable identity, a content
hash, byte-capped pages, deletion reporting via Domino's deletion stubs, and attachment metadata — and it
says when a deletion list cannot be complete instead of implying it is. `dommcp_rag_check_access` re-checks
at answer time whether the user may see what the index returned, because an index is a copy and a copy does
not age with your ACL. Embeddings, chunking and vector storage are deliberately out of scope. Both are
ENTERPRISE-only: read-only, but the most convenient way there is to carry a database out of the building.

*Java, and things that quietly did nothing.* A crashed Java agent reported `ok` with an empty result while
its stack trace went to the server log; the exception now comes back with the call. A Java library is its
own design type — writing LotusScript over one used to destroy two megabytes of third-party code and report
full success. Libraries can be **referenced** instead of embedded (the same agent went from 2.9 MB to 1.6 kB
per call), and a file already on the server can be named by path instead of being sent as base64. Elsewhere:
a permission switch that was read, stored and shipped in every configuration but never enforced now is
(with a migration that distinguishes an inherited default from an actual decision), a timeout the server
reported but never applied was removed rather than left as decoration, and guardrail obligations that only
set a flag are now enforced.

*Smaller, but you will notice.* Copying a design used to drop the entire application shell — framesets,
outlines, pages — and report success; promoting an app from test to production lost its navigation. Every
scaffolded application shipped an empty placeholder view that was the only one visible in the Notes menu.
`dommcp_export_database_dxl` now distinguishes "too large for this profile" from "unreadable" instead of
offering both and letting you guess. Steering fields such as `has_more` and `status` are serialized
**before** the payload, so a client that truncates long answers still sees them — they used to sit behind
30 KB of documents. And a text field named `Readers` protects nothing in Domino unless the item carries the
reader flag; writing one without it is now called out, since a protection that silently does nothing is the
one nobody discovers.

**New in v0.0.809** — the largest release so far, and most of it is about the server telling you what it
actually did.

*Design work that no longer fails quietly.* A LotusScript snippet can be run directly with
`dommcp_eval_lotusscript`, and a crash is now reported as `runtime_error` with the error and line instead of
`ok` with truncated output. `dommcp_import_dxl` compiles LotusScript but does not sign it — an unsigned
script library makes every agent that binds it abort silently, so unsigned code-bearing notes are now named
in the response. A library imported without `Option Public` exports nothing, and an agent using it receives
an empty value with no error anywhere; that is now flagged too, and `dommcp_run_agent` explains an
empty-output run by checking exactly these two causes. `dommcp_copy_design` and `dommcp_apply_package`
predict what a run cannot do **before** writing anything, and what the DXL path cannot carry (Java agents,
oversized script modules) is copied note by note instead. New alongside them:
`dommcp_recompile_lotusscript`, `dommcp_search_design`, `dommcp_set_database_title`.

*Changes you can review and undo.* Every design write takes a snapshot first
(`dommcp_list_design_snapshots` / `dommcp_get_design_snapshot` / `dommcp_restore_design_snapshot`, and the
restore is itself reversible). Related edits can be planned, diffed, approved by a second grant and applied
as one unit with rollback (`dommcp_create_change_set` …). `dommcp_design_dependencies` answers what uses
what before you delete it — and says plainly that "no references found" is not the same as "safe to delete".
`dommcp_export_package` / `inspect` / `apply` promote the *same* package to test and production instead of
having a model rebuild it twice.

*Knowing your estate.* `dommcp_assess_database` is read-only and in the READ tier, so it can run before you
grant write access; it reports findings with checkable evidence and no invented overall score. A browser
status page at `GET /status` needs no MCP client. `dommcp_run_tests` runs a declarative suite **once per
grant**, because a view returning rows for an administrator says nothing about a clerk.

*New capabilities.* Mail and calendar (`dommcp_send_mail` is off by default, enterprise only, and reports
`handed_to_router` rather than claiming delivery), CSV import and view export, application templates for
reproducible scaffolding, multi-server queries that state which peers answered, attachment text extraction
that distinguishes "no text layer" from "empty", a tamper-evident audit chain with SIEM export,
customer-written guardrails, and long calls as background tasks so they survive a proxy timeout.

*Smaller, but you will notice.* The delivery seed now ships `read` and `developer` role templates next to
`super-admin` — picking the smallest one cuts the tool list sent at every session start to roughly a third.
`search_documents` returns the same page twice in a row (it did not before). A config reload that would
return zero tokens is refused instead of revoking every credential at once. And a configuration database
that exists but cannot be opened is never overwritten: the server says so and stops, because replacing the
wrong file is quiet, not loud.

The Windows binary is current again in this release: a `gmtime_r` call — POSIX-only, and absent from MSVC —
had been breaking the Windows build, so the platform guard now lives in one shared helper instead of being
repeated at every call site.

**New in v0.0.671:** a grant's database list is now bound to `allow_all_databases` instead of being inferred
from `allowed_tools: "*"` — a token scoped to one database could otherwise see the server's whole inventory.
Grants can be scoped to a **directory** (`DatabasePathPrefix`) with a limit on how many databases may live
there and a default size quota. A design write that loses code now repairs itself instead of failing the
call, and a scheduled agent whose LotusScript does not compile is rolled back to the version that did — it
used to be left broken. Writes that had to fall back to the lower-fidelity path say so in one sentence, as
does a truncated server list. Several tools got cheaper to call: the batch design upsert writes many
elements in one round trip, and arguments the server fills in itself are no longer advertised.

**New in v0.0.650:** Windows binary current again (it had silently stopped being built for four weeks);
database titles with umlauts no longer break the JSON response; patching an agent keeps its UNID and note id;
MCP spec `2026-07-28` is served alongside the older revision, so existing clients are unaffected.

**New in v0.0.615:** a database created without a design reports it at startup and can be repaired with
`tell dommcp repair-design`; `dommcp_upsert_html_page` publishes a web UI from plain HTML; scheduled agents
keep their schedule when patched; patched file resources keep their MIME type; keyword choice fields are
clickable and verified after the write.

**New in v0.0.272:** browsable audit database, automatic re-signing of every DomMCP-written document, new
`dommcp_sign_database` tool, per-identity grant dedup, license doc kept in sync with the active license,
UTF-8 numeric-entity decode fix.

## Documentation

| File | Contents |
|---|---|
| [`QUICKSTART.md`](QUICKSTART.md) | Install per platform (Linux / Windows / container), first start, admin bootstrap, license, client setup. |
| [`OPERATE.md`](OPERATE.md) | Console verbs, backup/restore, monitoring, license renewal. |
| [`TOOLS.md`](TOOLS.md) | The MCP tool catalog by category (read / write / design / admin / DQL). |
| Handbook PDFs (release assets) | **Administrator handbook**, German and English: install (Linux, container, Windows), first administrator, licence, functional test, configuration database, AI client, operation, backup, troubleshooting, acceptance checklist. |
| `dommcpcfg.ntf` / `dommcpaudit.ntf` (release assets) | **Recommended fresh-install DB master templates** — token-less, no secrets: config (Token/Grant/License/Settings forms + views) and a browsable audit database. Put both into `<data dir>/dommcp/`; the add-in creates the databases from them on first load — see QUICKSTART step 1b. |
| [`CONFIG_EXAMPLE.json`](CONFIG_EXAMPLE.json) | Annotated config template (placeholders only). |
| [`examples/`](examples/) | n8n workflow + connector notes. |

## Contact

- Website: [it-dallmann.de](https://it-dallmann.de)
- Product page: [it-dallmann.de/dommcp.html](https://it-dallmann.de/dommcp.html) (German)
- Background article: [How DomMCP works](https://it-dallmann.de/blog/en/dommcp-domino-mcp-server.html) (English)
- HCL Domino Marketplace: [domino-dommcp](https://marketplace.hcl-software.com/domino/catalog/domino-dommcp)
- Please test in your environment and open a GitHub issue for bugs, questions, or ideas.
