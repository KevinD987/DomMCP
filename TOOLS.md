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
- `get_guidance` 🔍 — task-oriented guidance (DXL shapes, access paths, pitfalls) straight from the server,
  so an AI client does not have to guess the conventions.

## Read — documents

- `get_document` — one document by UNID.
- `get_documents_batch` — many documents, projected fields.
- `get_document_schema` — inferred field schema for a form/database.
- `search_documents` — full-text search.
- `get_document_attachments` 🔍 — list a document's file attachments, or download one as base64.
- `fetch_snapshot_chunk` — page through a large, size-capped result.
- `dommcp_render_view` / `dommcp_render_form` 🔍 — render a view or form the way the Notes client shows it
  (HTML), for a quick look without a client.
- `dommcp_export_view` 🔍 — export a view as CSV/JSON for reporting or a hand-over.
- `dommcp_import_csv` ✏️ — bulk-import documents from CSV, with a preview pass before anything is written.
- `dommcp_list_calendar_entries` 🔍 — read calendar entries. Repeating meetings are reported as
  `recurrence: "not_expanded"` rather than silently returning only the first occurrence.

## DQL & aggregation

- `query_documents_dql` — DQL query (use the `query` argument).
- `query_documents_dql_explain` — preview the DQL execution plan.
- `aggregate_documents` — group/aggregate document fields.
- `count_documents` — count matching documents.
- `aggregate_text_matches` — aggregate full-text matches.
- `extract_log_lines` — pull lines from log views.
- `count_log_events` — count log events.
- `aggregate_log_messages` — group/aggregate log messages.

## Retrieval / RAG — **ENTERPRISE only**

- `dommcp_rag_sync` 🔍 — hand documents to an index page by page, in a shape you can embed: stable identity
  (database + UNID), form, timestamps, a `content_hash`, the fields you selected, and a flat `text` that
  keeps the field name in front of each value — "active" on its own is not interpretable. Pages are capped
  in **bytes**, not only in document count, and every answer carries the call that fetches the next one. A
  delta run reports **deletions** (via Domino's deletion stubs, the only way to learn that a document is
  gone) and states plainly when that list cannot be complete, because Domino purges stubs after the
  replication cut-off. Attachments come along as metadata — name, size, type where it can be determined —
  never as bytes.
- `dommcp_rag_check_access` 🔍 — re-check, at answer time, whether *this* user may actually see a document
  the index returned. An index is a copy, and a copy does not age with your ACL.

> Embeddings, chunking, vector storage and reranking are deliberately **not** part of this. The server hands
> out documents; which model indexes them is your choice.
>
> Both tools sit in the **ENTERPRISE** package even though they are read-only. The package boundary follows
> blast radius, not difficulty: `dommcp_rag_sync` is the most convenient way there is to carry a complete
> customer database out of the building, page by page.

## Write — documents ✏️

- `create_document` — create a document (`form` + typed `fields` array).
- `update_document` — update fields of a document (target by UNID).
- `delete_document` — delete a document.
- `create_document_attachment` ✏️ — attach a file to a document (`$FILE`, from base64 or text).
- `dommcp_write_richtext` ✏️ — write a real rich-text field (CD records): styled runs, colours, fonts.
- `dommcp_folder` ✏️ — create a folder or file/move/remove documents in it, addressed by UNID.
- `dommcp_upsert_design_elements` ✏️ — apply several design changes in one call, in dependency order.

> **`status` means RESULT, not intent.** A write is `ok` only after the server read the document back and
> the stored values matched. `partial` means written but not verifiable, `error` means a proven loss — and
> then the response *names the fields*. Never treat `ok` as the only thing worth checking.

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
- `dommcp_eval_lotusscript` ✏️ — run a LotusScript snippet and get its `Print` output back, without
  hand-building a throwaway agent. The server creates one, runs it and removes it again — also when the run
  fails. A runtime error is reported as such, with the line number in *your* snippet.
- `dommcp_recompile_lotusscript` ✏️ — recompile the agents that bind a given script library. Use it after
  changing what a library **exports**: a change to the *body* of an existing function reaches consumers
  immediately, a change to its *interface* only after a recompile. It also names the consumers that no
  longer compile — the failure you would otherwise meet at runtime.
- `dommcp_search_design` 🔍 — full-text search across the design source (LotusScript, formulas, JavaScript)
  with element, line and matching text. A broken pattern is reported as an error, never as "0 hits".
- `dommcp_copy_design` ✏️ — copy **named** design elements from one database into another, in dependency
  order, without a master-template binding. Identical elements are left alone, so a second run is harmless.
- `dommcp_design_dependencies` 🔍 — dependency graph and impact analysis (`dependencies_of` | `used_by` |
  `impact_analysis`), each edge with its source location and a confidence level. "0 references" means
  *nothing was found*, never "safe to delete" — the response says so.
- `dommcp_list_design_capabilities` 🔍 — what this build can write per design type, measured, not promised.

## Design safety net — snapshots, change sets, promotion

- `dommcp_list_design_snapshots` / `dommcp_get_design_snapshot` 🔍 — before **every** design write the server
  stores the previous state; these read the history and verify a snapshot's integrity.
- `dommcp_restore_design_snapshot` ✏️ — roll a design element back. The restore is itself reversible and
  returns an `undo_snapshot_id`.
- `dommcp_create_change_set` ✏️ — plan several related design changes as ONE unit (writes nothing yet).
- `dommcp_get_change_set` 🔍 — diff and conflict view for a planned change set.
- `dommcp_approve_change_set` ✏️ — four-eyes approval: the grant that planned it may not approve it.
- `dommcp_apply_change_set` ✏️ — apply it. All baselines are checked before the first write; a failure
  mid-way rolls back what was already applied and the state is never reported as `applied`.
- `dommcp_export_package` 🔍 / `dommcp_inspect_package` 🔍 / `dommcp_apply_package` ✏️ — reproducible
  promotion: build a package once, dry-run it against the target, then apply the **same** package to test
  and production instead of having a model generate it twice.

## Assessment, testing & automation

- `dommcp_assess_database` 🔍 — inventory plus prioritised findings with checkable evidence (element, line,
  matching text). **Read-only**, so it can run before anyone grants write access.
- `dommcp_run_tests` ✏️ — run a declarative test suite; the same cases run **once per grant**, because rows
  an administrator sees say nothing about what a caseworker sees. A failure carries evidence, not just
  `false`.
- `dommcp_list_app_templates` 🔍 / `dommcp_create_app_from_template` ✏️ — a small gallery of reviewed
  starting points, so "build me a helpdesk" yields the same application twice.
- `dommcp_policy_explain` 🔍 — dry-run your own guardrail policy: which rule would decide, for which grant,
  at which time — without executing anything.
- `dommcp_start_task` / `dommcp_get_task` / `dommcp_cancel_task` — run a long call as a background task so
  it survives a proxy timeout. Only calls that can genuinely take minutes are accepted.
- `dommcp_server_task` ✏️ — control a Domino server task.
- `dommcp_federated_query` 🔍 — ask several DomMCP servers at once (`find_database` | `replica_status` |
  `server_health`). The answer names its participants and flags itself incomplete when one stays silent —
  a missing row is never proof that something does not exist.
- `dommcp_send_mail` ✏️ — send mail through the server. Off by default, bounded by an explicit policy, with
  a preview mode — it is the one action DomMCP cannot take back.

## Audit trail

- `dommcp_query_audit` 🔍 — query the audit trail (who did what, with which tool, against which database).
- `dommcp_export_audit` 🔍 — export it for reporting or an auditor.
- `dommcp_verify_audit_chain` 🔍 — verify the trail's integrity chain.

## Database lifecycle ✏️

- `dommcp_create_database_empty` — create an empty NSF (auto-creates a default `Main` view).
- `dommcp_create_database_from_template` — create from a template.
- `delete_database` — delete a database.
- `db_compact` — compact a database.
- `db_get_quota` 🔍 — read quota/size limits.
- `db_updall` ✏️ — rebuild/refresh view indexes.
- `dommcp_set_database_title` ✏️ — rename an existing database. Only the title is touched; the stored value
  is read back and compared.

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
- `dommcp_list_grants` 🔍 / `dommcp_list_tokens` 🔍 — the configured grants and tokens (hashes only, never
  a secret).
- `notesini_set` ✏️ — set a `notes.ini` variable from an allow-listed set.

---

### Edition gating (summary)

- **Essentials** — free forever: which databases exist, their views and fields, server state.
- **Analyst** — read / DQL / aggregation / design-inspection tools.
- **PRO** — adds the write ✏️ and design-authoring tools.
- **ENTERPRISE** — full set incl. administration, higher limits.
- **Trial** — 14 days of the full PRO set.

Beyond the edition, a token only ever sees the tools its **grant** allows (`allowed_tools`), scoped to the
databases, views, and fields the grant permits.

> The edition names are the licensing tiers. Actual capability is enforced by the **entitlements carried in
> the signed license** plus the **grant** — not by the edition label alone.
