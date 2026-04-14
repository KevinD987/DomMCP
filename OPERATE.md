# DomMCP Operation Guide

## Paths
- Binary: `/opt/hcl/domino/notes/latest/linux/dommcp_addin`
- Config: `/local/notesdata/dommcp-default-config.json`

## Start / Stop / Reload
```bash
# Stop
domino cmd "tell dommcp quit"

# Start
domino cmd "load dommcp_addin /local/notesdata/dommcp-default-config.json"

# Reload config without restart
domino cmd "tell dommcp reload-config"
```

## Runtime Status
```bash
domino cmd "tell dommcp status"
domino cmd "tell dommcp show-metrics"
```

## Health Checks (HTTP/MCP)
```bash
curl -sS -X POST http://<host>:8088/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

## License Activation
1. Set `license.license_code` in config.
2. Set `license.status` to `active`.
3. Restart add-in:
```bash
domino cmd "tell dommcp quit"
domino cmd "load dommcp_addin /local/notesdata/dommcp-default-config.json"
```

## Notes
- Without a valid license, server runs in Starter baseline (`starter_default_unlicensed`).
- Professional/Enterprise tools require valid signed entitlements.
