# Connecting AI clients to DomMCP

DomMCP speaks standard MCP over `POST /mcp` (JSON-RPC 2.0) on port 8088. Three integration styles:

| Client | Transport | Auth |
|---|---|---|
| Anthropic Claude (claude.ai) | OAuth 2.1 + Streamable HTTP | OAuth consent → DomMCP token |
| OpenAI / ChatGPT | MCP connector (HTTP) | DomMCP token |
| n8n / scripts | Plain HTTP / MCP client node | `Authorization: Bearer <token>` |

## Prerequisite for Claude & ChatGPT: public HTTPS

Both hosted assistants require a **public HTTPS** endpoint. Put a reverse proxy (nginx, Caddy, Traefik, …) in
front of DomMCP that terminates TLS and forwards to `http://<dommcp-host>:8088`, passing through:

- `Host` — the public hostname
- `X-Forwarded-Proto: https`

Example nginx location:

```nginx
location /mcp {
    proxy_pass http://127.0.0.1:8088;
    proxy_set_header Host              $host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Forwarded-For   $remote_addr;
    proxy_http_version 1.1;
    proxy_buffering off;          # keep SSE streaming responsive
}
```

Expose `/.well-known/oauth-protected-resource` and `/.well-known/oauth-authorization-server` through the same
proxy as well — DomMCP serves the OAuth discovery documents itself.

## Anthropic Claude — custom connector

1. In claude.ai, add a **custom connector** with the URL `https://<your-public-host>/mcp`.
2. Claude performs OAuth dynamic client registration and redirects to DomMCP's consent page.
3. On the consent page, paste a **DomMCP Bearer token** — the access token is bound to that token's grant.
4. Approve. Claude can now call the tools the grant allows.

Notes:
- Access tokens are long-lived; the refresh token is stable. You re-authorize only if the server's OAuth
  state is reset (e.g. an add-in redeploy).
- Write/design tools work without you supplying a write-intent value — the connector binding derives it.

## OpenAI / ChatGPT — MCP connector

1. Add an MCP connector pointing at `https://<your-public-host>/mcp`.
2. Authorize with a DomMCP token.
3. Tool availability follows the token's grant and the license edition.

## n8n — Bearer

Use the MCP client node (or an HTTP Request node) against `http://<dommcp-host>:8088/mcp` with header
`Authorization: Bearer <token>`. A complete workflow is in [`../n8n/`](../n8n/).

## Quick check (any client / curl)

```bash
curl -sS -X POST https://<your-public-host>/mcp \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer <token>' \
  --data '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

> Tokens are secrets. Use a least-privilege grant per client (see [`../../CONFIG_EXAMPLE.json`](../../CONFIG_EXAMPLE.json)),
> and prefer a read-only grant for clients that only need to read.
