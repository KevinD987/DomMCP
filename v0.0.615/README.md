# DomMCP v0.0.615 — Linux x86_64

Linux add-in, the database templates and the administrator handbooks. The Windows binary and the container
add-on tarball (`.taz`) are **not** part of this drop — the previous full release
[`v0.0.272/`](../v0.0.272) still carries those.

| Artifact | File |
|---|---|
| Linux x86_64 add-in | [`linux-x86_64/dommcp_addin-linux-x86_64-glibc2.38`](linux-x86_64) |
| Database templates (config + audit) | [`dommcp/`](dommcp) → copy the whole folder into `/local/notesdata/` |
| Handbook (German) | [`docs/DomMCP-Installationsanleitung.pdf`](docs) |
| Handbook (English) | [`docs/DomMCP-Installation-Guide.pdf`](docs) |
| Checksums | `SHA256SUMS` in each folder |

**Put the templates in place before the first start.** If `dommcpcfg.nsf` and `dommcpaudit.nsf` are missing,
the add-in creates empty ones: the server runs, but those databases contain no form and no view and cannot be
opened in the Notes client at all. Since this build the task warns about that at startup and
`tell dommcp repair-design` installs the design into an existing database without touching its documents.

The templates in this drop also fix two things you would otherwise see in the client: the choice fields
(status, the 1|0 rights) are real dropdowns again instead of a flat, unclickable list, and the configuration
database finally carries its own icon.

```bash
shasum -a 256 -c SHA256SUMS
```

## ⚠️ Check your glibc before downloading

This build needs **glibc ≥ 2.38** — higher than the previous public build, which ran on **2.34**. The floor
comes from the build container, not from DomMCP itself.

```bash
ldd --version | head -n 1
```

| Distribution | glibc | This build |
|---|---|---|
| Ubuntu 24.04 LTS | 2.39 | ✅ |
| Debian 13 | 2.41 | ✅ |
| RHEL / Rocky / Alma 10 | 2.39 | ✅ |
| Ubuntu 22.04 LTS | 2.35 | ❌ — use [`v0.0.272`](../v0.0.272) |
| RHEL / Rocky / Alma 9 | 2.34 | ❌ — use [`v0.0.272`](../v0.0.272) |
| RHEL / Rocky / Alma 8 | 2.28 | ❌ — use [`v0.0.272`](../v0.0.272) |

A build for older glibc is planned. If you need one now, say so — it is a build-container question, not a
code change.

## Installation

Both handbooks in `docs/` cover the whole path: install, first administrator, licence, functional test,
configuration database, connecting an AI client, day-to-day operation, backup, troubleshooting and an
acceptance checklist. Short version for Linux:

```bash
cp dommcp_addin-linux-x86_64-glibc2.38 /opt/hcl/domino/notes/latest/linux/dommcp_addin
chown notes:notes /opt/hcl/domino/notes/latest/linux/dommcp_addin
chmod 755         /opt/hcl/domino/notes/latest/linux/dommcp_addin
```

On the Domino console — **without a path argument**, so the add-in finds its configuration NSF in the
`dommcp/` subfolder itself:

```
load dommcp_addin
tell dommcp provision-admin <yourSecret>
```

## What changed since v0.0.272

Several hundred commits; the ones that change how you operate it:

- **A database without a design says so.** If the shipped, designed `dommcpcfg.nsf` / `dommcpaudit.nsf` were
  not in place at install time, the add-in used to create empty ones silently: the server worked, but the
  databases could not be opened in the Notes client at all. The task now warns about this at every start and
  the new console verb **`tell dommcp repair-design`** installs the design into an existing database without
  touching its documents.
- **`dommcp_upsert_html_page`** — publish a web UI from an NSF by handing over plain HTML; the server does the
  pass-through plumbing. Pages written through the DXL importer are now served byte-identically (the importer
  left `$WebFlags` unset, so Domino wrapped them in its own page frame and dropped the line breaks).
- **Scheduled agents keep their schedule.** A `dommcp_patch_design` whose replacement text contained `<` broke
  the import and silently downgraded a scheduled agent to manual. The replacement is escaped for its context
  now, the fallback no longer runs silently, and the scheduled path compiles like the native one.
- **File resources keep their MIME type** when patched — a patched `.js` used to fall back to
  `application/octet-stream`, which browsers refuse to execute.
- **Choice fields are clickable.** Keyword combobox/listbox fields are written with `usenotesstyle='false'`,
  and the result is verified after the write instead of only being enforced during it.
- OAuth for claude.ai custom connectors, run-as ACL enforcement, DQL, aggregations, audit database with
  browsable views, asynchronous audit writing.

Full detail: the handbooks in `docs/`.
