# DomMCP MCP Tools

The authoritative list is always `tools/list` against your server — what a given **token** sees depends on
its grant and the active license edition. This catalog groups the tools by category.

**Legend:** 🔍 = read-only · ✏️ = mutating (needs a unique `idempotency_key`; the write intent is added by
the server — see below). Unmarked tools in a category inherit that category's nature.

> Read tools require only a Bearer token. Write / design / admin tools additionally need a **write intent**
> — but over MCP the server derives it from your authenticated session, so **an MCP client sends nothing
> extra**. Only direct HTTP callers that bypass the MCP session hand over `<token>-write` themselves. Every
> mutating call does need a unique `idempotency_key`, which the client generates.

## Read — discovery & views

- `list_allowed_databases` — databases this token may access.
- `get_database_info` — title, size, counts, metadata.
- `profile_database` — shape/size and best access path.
- `suggest_best_access_path` — recommend how to read a database efficiently.
- `list_views` — views/folders in a database.
- `probe_view_shape` — column/structure probe of a view.
- `probe_log_view` — structure probe for log views.
- `read_view_entries` — read rows from a view (paged).
- `list_fields_for_view` — fields/columns backing a view.

## Read — documents

- `get_document` — one document by UNID.
- `get_documents_batch` — many documents, projected fields.
- `get_document_schema` — inferred field schema for a form/database.
- `search_documents` — full-text search.
- `get_document_attachments` 🔍 — list a document's file attachments, or download one as base64.
- `fetch_snapshot_chunk` — page through a large, size-capped result.
- `dommcp_render_view` / `dommcp_render_form` 🔍 — render a view or form the way the Notes client shows it
  (HTML), for a quick look without a client.

## DQL & aggregation

- `query_documents_dql` — DQL query (use the `query` argument).
- `query_documents_dql_explain` — preview the DQL execution plan.
- `aggregate_documents` — group/aggregate document fields.
- `count_documents` — count matching documents.
- `aggregate_text_matches` — aggregate full-text matches.
- `extract_log_lines` — pull lines from log views.
- `count_log_events` — count log events.
- `aggregate_log_messages` — group/aggregate log messages.

## Write — documents ✏️

- `create_document` — create a document (`form` + typed `fields` array).
- `update_document` — update fields of a document (target by UNID).
- `delete_document` — delete a document.
- `create_document_attachment` ✏️ — attach a file to a document (`$FILE`, from base64 or text).
- `dommcp_write_richtext` ✏️ — write a real rich-text field (CD records): styled runs, colours, fonts.
- `dommcp_folder` ✏️ — create a folder or file/move/remove documents in it, addressed by UNID.

## Design authoring

- `dommcp_upsert_design_element` ✏️ — create/replace a design element: `form | subform | page | view |
  folder | outline | frameset | navigator | agent_formula | agent_lotusscript | script_library |
  database_script | shared_field | image | file | stylesheet`.
- `dommcp_patch_design` ✏️ — lossless component-level patch (field/action/column/event/entry/frame).
- `dommcp_upsert_lotusscript_agent_source` ✏️ — author + compile a LotusScript agent from structured source.
- `dommcp_delete_design_element` ✏️ — delete a design element.
- `dommcp_get_design_element` 🔍 — read a design element (DXL/details).
- `dommcp_list_design_elements` 🔍 — list a database's design elements.
- `dommcp_export_database_dxl` 🔍 — export database/design as DXL.
- `dommcp_import_dxl` ✏️ — the official Domino DXL importer, byte-faithful: the right choice whenever the
  DXL carries visual styling (frameset, outline, styled form or view, image).
- `dommcp_upsert_html_page` ✏️ — publish a **web UI from plain HTML**: hand over the complete markup, the
  server does the pass-through plumbing and can point the database's web launch at the page.
- `dommcp_upsert_java_agent` ✏️ — author a Java agent (class files / JARs, with byte and SHA-256 echo).
- `dommcp_run_agent` ✏️ — run an agent and return its `Print` output.
- `dommcp_sign_database` ✏️ — re-sign a database's design with the server ID (needed after copying a
  template NSF).
- `dommcp_refresh_design` ✏️ — refresh a database's design from its master template.
- `dommcp_scaffold_app` ✏️ — generate a complete client application (forms, views, sidebar navigation,
  frameset) from a compact content model.
- `dommcp_validate_formula` 🔍 — compile-check a Domino formula before writing it into a design element.

## Database lifecycle ✏️

- `dommcp_create_database_empty` — create an empty NSF (auto-creates a default `Main` view).
- `dommcp_create_database_from_template` — create from a template.
- `delete_database` — delete a database.
- `db_compact` — compact a database.
- `db_get_quota` 🔍 — read quota/size limits.

## ACL & security

- `dommcp_set_database_acl` ✏️ — add/update/delete ACL entries, define & assign roles, set admin server.
  (Always use this tool for ACL — not `dommcp_upsert_design_element`.)
- `get_database_acl` 🔍 — read the ACL.
- `get_database_acl_effective` 🔍 — effective access for a name.
- `list_database_roles` 🔍 — roles defined in a database.
- `show_database_users` 🔍 — users/principals with access.

## Administration — users, groups, server

- `provision_person_token` ✏️ — mint a run-as token mapped to a Domino user.
- `dommcp_register_user` ✏️ — register a Domino person.
- `dommcp_manage_group` ✏️ — create/update groups & membership.
- `server_console_command` ✏️ — run a scoped Domino console command.
- `get_server_health` 🔍 — server health.
- `get_server_stats` 🔍 — server statistics.
- `get_license_status` 🔍 — current license edition, status, expiry, and runtime mode.
- `self_test` 🔍 — internal self-test.
- `list_server_processes` 🔍 — running server processes.
- `list_active_users` 🔍 — active users/sessions.
- `list_replica_status` 🔍 — replication status.
- `list_server_groups` 🔍 — server/directory groups.
- `list_server_databases` 🔍 — databases on the server.

---

### Edition gating (summary)

- **READ** — read / DQL / aggregation tools.
- **PRO** — adds the write ✏️ and design-authoring tools.
- **ENTERPRISE** — full set incl. administration, higher limits.
- **TRIAL** — time-boxed evaluation.

Beyond the edition, a token only ever sees the tools its **grant** allows (`allowed_tools`), scoped to the
databases, views, and fields the grant permits.

> The edition names are the licensing tiers. Actual capability is enforced by the **entitlements carried in
> the signed license** plus the **grant** — not by the edition label alone.
