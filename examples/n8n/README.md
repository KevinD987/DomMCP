# n8n Example: Domino + Mattermost

File:
- `Workflow_Domino_Mattermost.example.json`

What it does:
- Receives a webhook message.
- Optionally filters to one allowed Mattermost user.
- Calls DomMCP via MCP client node.
- Sends the answer back to Mattermost.

Before import/use:
- Replace all placeholder values:
  - `<webhook-path>`
  - `<allowed_user>`
  - `<dommcp-host>`
  - `<mattermost-channel-id>`
  - credential placeholders (`<...-credential-id>`, `<...-credential-name>`)
- Configure n8n credentials:
  - Mattermost API
  - MCP bearer token
  - LLM provider credentials

Security note:
- This example is sanitized for public release.
- Do not commit real webhook IDs, channel IDs, bearer tokens, or credential IDs.
