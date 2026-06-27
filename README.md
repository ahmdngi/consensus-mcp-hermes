# Consensus MCP for Hermes Agent

Connect [Consensus](https://consensus.app) — 200M+ peer-reviewed research papers — to [Hermes Agent](https://hermes-agent.nousresearch.com) via the Model Context Protocol (MCP).

## How It Works

Consensus's MCP server at `https://mcp.consensus.app/mcp` uses OAuth for authentication. This guide uses the [`mcp-remote`](https://www.npmjs.com/package/mcp-remote) bridge package, which handles the OAuth flow and proxies MCP calls between Hermes and Consensus.

```
Hermes → mcp-remote (stdio) → OAuth token → Consensus MCP (https)
                                            → 200M+ papers
```

## Prerequisites

- **Hermes Agent** with MCP client enabled (`pip install mcp`)
- **Node.js / npx** (ships with Node.js)
- **Consensus account** — free tier works (10 papers/search with OAuth, 3/search without)

## Setup

### 1. Add to Hermes Config

Edit `~/.hermes/config.yaml`:

```yaml
mcp_servers:
  consensus:
    command: npx
    args:
      - -y
      - mcp-remote@latest
      - https://mcp.consensus.app/mcp
    timeout: 180
    connect_timeout: 60
```

**⚠️ Critical:** `args` must be a YAML **list** (as above), not a string. Using `hermes config set` stores it as a string and silently breaks the connection. Verify with:

```bash
grep -A 5 'consensus:' ~/.hermes/config.yaml
```

### 2. Trigger OAuth

```bash
hermes mcp test consensus
```

This starts the mcp-remote bridge, connects to Consensus, and generates an authorization URL.

### 3. Complete OAuth (headless workflow)

The agent shares an authorization URL with you:

```
https://consensus.app/oauth/authorize?response_type=code&client_id=...&redirect_uri=http://localhost:6761/oauth/callback&state=YYYY&...
```

**The redirect goes to `localhost:6761` on the server** — your browser will fail to reach it. This is expected.

1. Open the URL in your browser (desktop or phone)
2. Sign in with your Consensus credentials
3. Click **Authorize**
4. Your browser tries to redirect to `http://localhost:6761/oauth/callback?code=XXXX&state=YYYY` and fails — **copy the `code=XXXX` value** from the failing URL
5. Paste the code back. The agent delivers it to the callback server:

```bash
curl "http://localhost:6761/oauth/callback?code=XXXX&state=YYYY"
```

The callback server exchanges the code for a Bearer token and caches it at `~/.mcp-auth/`.

### 4. Verify

```bash
hermes mcp test consensus
```

Expected output:
```
✓ Connected (2800ms)
✓ Tools discovered: 1
  search    Search over 200 million peer-reviewed academic papers.
```

```bash
hermes mcp ls
```

Should show `consensus` with status `✓ enabled`.

## Usage

Once connected, the `search` tool is available with these parameters:

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | yes | Research question; use academic terminology |
| `year_min` | integer | no | Exclude papers before this year |
| `year_max` | integer | no | Exclude papers after this year |
| `study_types` | string[] | no | Filter: `rct`, `meta-analysis`, `systematic review`, etc. |
| `sjr_max` | integer | no | Journal quartile (1=Q1 highest, 4=Q4) |
| `human` | boolean | no | Human subjects only |
| `sample_size_min` | integer | no | Minimum sample size |
| `medical_mode` | boolean | no | Top medical journals only |
| `exclude_preprints` | boolean | no | Peer-reviewed only |
| `duration_min` | integer | no | Minimum study duration in days |

## Re-authentication

Tokens are cached by mcp-remote at `~/.mcp-auth/mcp-remote-<version>` and include a refresh token (4-hour access token expiry). To re-authenticate manually:

```bash
rm -rf ~/.mcp-auth/mcp-remote-*
hermes mcp test consensus
# Repeat OAuth flow
```

Or if available:

```bash
hermes mcp reauth consensus
```

## OAuth Endpoints

Discovered from `https://consensus.app/.well-known/oauth-authorization-server`:

| Field | Value |
|-------|-------|
| Authorization endpoint | `https://consensus.app/oauth/authorize` |
| Token endpoint | `https://consensus.app/oauth/token/` |
| Grant types | `authorization_code`, `refresh_token` |
| PKCE | S256 (required) |
| Client type | Public (no client secret) |
| Scopes | `search` |

## Common Pitfalls

1. **args stored as string** — `hermes config set` writes `args` as a YAML string, not a list. Edit config file directly.
2. **OAuth callback misses** — opening the authorize URL on a different machine redirects to *that* machine's localhost. Use the manual code delivery via curl.
3. **mcp-remote exits** — if the process dies before OAuth completes, the PKCE challenge is invalidated. Kill stale processes and restart.
4. **Cloudflare blocking** — consensus.app uses Cloudflare managed challenge. Use a real browser for the sign-in step (headless Playwright is blocked).

## Related

- [mcp-remote package](https://www.npmjs.com/package/mcp-remote) — stdio-to-HTTP MCP proxy with OAuth
- [Hermes Agent MCP docs](https://hermes-agent.nousresearch.com/docs) — native MCP client setup
- [Consensus API docs](https://docs.consensus.app/docs/mcp) — official MCP documentation
