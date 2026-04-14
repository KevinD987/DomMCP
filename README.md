# DomMCP Public Binary (Starter / Professional / Enterprise)

## What is DomMCP
- DomMCP is a read-only MCP server add-in for HCL Domino.
- It provides MCP tools for Domino retrieval, inspection, and operations used by agents and workflow systems.
- This publication is a **closed-source binary distribution**.

## Public editions
- **Starter**: curated core read-only toolset with tighter runtime limits.
- **Professional** (**licensed**): extended read/analysis toolset (query + aggregation + broader capabilities).
- **Enterprise** (**licensed**): Professional superset with enterprise entitlement profile and rollout policy.

## License model / default runtime behavior
- Without a valid license code, DomMCP runs in Starter baseline mode.
- Technical runtime status for that mode is `starter_default_unlicensed`.
- In that mode, Starter toolset and Starter caps are enforced permanently.
- Professional and Enterprise capabilities require a valid signed license and remain blocked otherwise.

## Release artifacts
- Versioned artifacts are published under `release/public/<version>/`.
- Current package files:
  - `release/public/v0.0.81/linux-x86_64/dommcp_addin`
  - `release/public/v0.0.81/linux-x86_64/SHA256SUMS`
  - `release/public/v0.0.81/RELEASE.md`
  - `release/public/CONFIG_EXAMPLE.json`
  - `release/public/OPERATE.md`
  - `release/public/QUICKSTART.md`
  - `release/public/TOOLS.md`

## Minimal installation
- Copy binary and set executable bit.
- Provide runtime JSON config.
- Restart add-in with target config.
- Validate MCP endpoint.

```bash
cp dommcp_addin /opt/hcl/domino/notes/latest/linux/dommcp_addin
chmod +x /opt/hcl/domino/notes/latest/linux/dommcp_addin
domino cmd "tell dommcp quit"
domino cmd "load dommcp_addin /local/notesdata/<config>.json"
```

```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

## License activation
- Insert signed license code into config key `license.license_code`.
- Restart add-in.
- Validate runtime/licensing tools.

```bash
domino cmd "tell dommcp quit"
domino cmd "load dommcp_addin /local/notesdata/<config>.json"
```

## Supported platform
- Linux x86_64 (`ELF 64-bit` add-in binary).
- Built against Domino C API on Linux.
- **Requires HCL Domino 14.5** (runtime and API compatibility target for this release line).

## Known prerequisites / constraints
- Domino server runtime must match released binary requirements.
- Domain binding in runtime config must match signed license claims.
- MCP listen host/port must be reachable from clients.
- Public toolset is read-only by design.
