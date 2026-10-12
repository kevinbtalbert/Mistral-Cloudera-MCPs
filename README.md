# Impala MCP Server

Read-only [Model Context Protocol](https://modelcontextprotocol.io/) server that exposes Cloudera Data Warehouse (Impala) tables to MCP clients over **Streamable HTTP**.

It is meant to run as a **Cloudera AI Application**. The workbench ingress terminates TLS and forwards to `app.py` on `127.0.0.1:$CDSW_APP_PORT`; MCP clients use the application’s **public HTTPS URL**, not a machine on your laptop.

Two tools:

| Tool | Description |
|------|-------------|
| `get_schema` | Lists tables in the configured database (`SHOW TABLES`) |
| `execute_query` | Runs read-only Impala SQL (`SELECT`, `SHOW`, `DESCRIBE`, `WITH`) |

`app.py` is intentionally a single file (PBJ application runtimes do not reliably support local package imports).

## Requirements

- Cloudera AI workbench with **Applications** and a Python **3.11+** ML runtime
- CDW Impala coordinator hostname and CDP **workload** credentials (read access to the target database)
- A workbench URL reachable by your MCP client (typically public HTTPS, off VPN)

## Unauthenticated application access

Remote MCP clients cannot complete Cloudera SSO. For them to call `/mcp`, the application must allow unauthenticated HTTP access to its URL (or you must use `MCP_BEARER_TOKEN` and configure the client accordingly—SSO is still bypassed for the app URL when unauthenticated access is enabled).

1. **Workbench admin:** enable **Allow applications to be configured with unauthenticated access** (Admin → Security). Without this, the “Enable Unauthenticated Access” option does not appear on the application.
2. **Application:** when creating or editing the app, turn on **Enable Unauthenticated Access** unless you rely only on bearer auth and your client supports it.

If unauthenticated access is off, `/healthz` and `/mcp` often return a login page or redirect; MCP `initialize` fails against HTML instead of JSON.

When the endpoint is open, anyone who knows the URL can run read-only queries as your Impala user. Use a read-only workload account and consider `MCP_BEARER_TOKEN` for production.

## Deploy

1. **New Project → Git** — connect this repository (or push it to your Git remote and use that URL).
2. Open a **project session** and install dependencies once:

   ```bash
   python scripts/reinstall_deps.py
   ```

   Use this instead of a bare `pip install` when upgrading fastmcp major versions on an existing project.

3. **Applications → New Application**

   | Field | Value |
   |-------|--------|
   | Script | `app.py` |
   | Runtime | Python **3.11+** |
   | Subdomain | e.g. `impala-mcp` (becomes part of the public hostname) |
   | **Enable Unauthenticated Access** | **On** (required for typical remote MCP clients; see above) |
   | Resources | 1 vCPU / 2 GB is usually enough |

4. Set **application environment variables** (see below). Do not rely on `.env` in the repo—it is not deployed. After start, check application logs for `[startup]` lines; `<UNSET>` for `IMPALA_HOST` or `IMPALA_USER` means variables were not applied.

5. Leave **`MCP_HOST`** and **`MCP_PORT`** unset so the process binds `127.0.0.1` on **`CDSW_APP_PORT`** (set by the platform).

Public endpoints:

- MCP: `https://<subdomain>.<workbench-domain>/mcp`
- Health: `https://<subdomain>.<workbench-domain>/healthz`

## Verify

From a machine that can reach the workbench (often **off VPN**):

```bash
curl -i https://<subdomain>.<workbench-domain>/healthz   # expect: 200 ok
```

In a project session (with `requirements.txt` dependencies available):

```bash
python scripts/smoke_test.py https://<subdomain>.<workbench-domain>/mcp \
  --query "SELECT 1"
```

If `healthz` returns HTML or a 302 to SSO, fix unauthenticated access before debugging Impala or MCP.

## Environment variables

Set these on the **Cloudera AI Application**. `.env.example` lists the same names for reference when filling in the UI.

### Impala connection (required)

| Variable | Description |
|----------|-------------|
| `IMPALA_HOST` | Impala coordinator hostname |
| `IMPALA_PORT` | Coordinator port (often `443`) |
| `IMPALA_USER` | CDP workload username |
| `IMPALA_PASSWORD` | CDP workload password |
| `IMPALA_DATABASE` | Default database for `SHOW TABLES` and unqualified table names |
| `IMPALA_AUTH_MECHANISM` | Default `LDAP` |
| `IMPALA_USE_HTTP_TRANSPORT` | Default `true`; use `false`, `0`, `no`, or `off` to disable |
| `IMPALA_HTTP_PATH` | Default `cliservice` |
| `IMPALA_USE_SSL` | Default `true`; use `false`, `0`, `no`, or `off` to disable |

String values like `"false"` are **not** treated as false unless they match the patterns above.

### HTTP server (usually leave defaults)

| Variable | Description |
|----------|-------------|
| `CDSW_APP_PORT` | Set by Cloudera AI when the application runs; do not override |
| `MCP_HOST` | Default `127.0.0.1`; keep default in Applications |
| `MCP_PATH` | Default `/mcp` |
| `LOG_LEVEL` | Default `info` |

### Optional auth

| Variable | Description |
|----------|-------------|
| `MCP_BEARER_TOKEN` | If set, `/mcp` requires `Authorization: Bearer <token>`. `/healthz`, `/`, and `/icon.png` stay public. |

## HTTP routes

| Path | Purpose |
|------|---------|
| `GET /healthz` | Liveness probe (no Impala connection) |
| `GET /` | Service metadata JSON |
| `GET /icon.png` | Connector icon |
| `POST /mcp` | MCP Streamable HTTP (stateless JSON responses) |

## Troubleshooting

| Symptom | Likely cause |
|---------|----------------|
| `/healthz` returns login HTML or 302 | Unauthenticated application access not enabled (admin + app checkbox) |
| `get_schema` → `Error: ...` | Wrong `IMPALA_*`, suspended virtual warehouse, or network from the app pod |
| `[startup]` shows `<UNSET>` for Impala vars | Variables set in `.env` or project files but not on the **application** |
| `ImportError: FastMCP` from `fastmcp` | Broken in-place upgrade — run `python scripts/reinstall_deps.py` in a session |
| Client handshake fails on `initialize` | Client probe quirks; server includes compatibility middleware for common cases |

## License

Derived from [`cloudera/iceberg-mcp-server`](https://github.com/cloudera/iceberg-mcp-server) (Apache-2.0). See `NOTICE` and `LICENSE` when present in the repository.
