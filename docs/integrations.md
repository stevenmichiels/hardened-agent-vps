# Private Integrations

Optional services are isolated from live applications, repositories, and each
other. Keep their host ports on loopback or a Tailscale address unless a public
exposure change has been reviewed explicitly.

## VPS Zones

The playbook manages parent zones rather than putting every workload in one
root-owned directory:

```text
/srv/apps
|-- production-site
`-- staging-site

/home/<agent_user>
|-- repos/project
`-- agent-workspaces
    |-- claude-code
    `-- codex-cli

/opt
`-- firecrawl

/etc
`-- firecrawl

/var
`-- backups/agent-vps
```

Concrete app and repository directories are created only when their names are
known. Coding agents work in Git repositories or disposable workspaces, not in
production runtime directories, env files, or databases.

## Firecrawl

Firecrawl is a separate private service zone. Its compose files live under
`/opt/firecrawl`, its secret-bearing env file lives under `/etc/firecrawl`, and
its host API defaults to `http://127.0.0.1:3002`.

Redis, RabbitMQ, and Postgres stay on the private Firecrawl backend network;
only the API port is published, on loopback.

The VPS backup includes `/etc/firecrawl` but not live Firecrawl Docker volumes.
Use a separate cold volume backup if job history must survive a rebuild.

## Codex CLI and Firecrawl MCP

Codex CLI running on the VPS can reach the private host endpoint without
exposing Firecrawl publicly:

```sh
codex mcp add firecrawl \
  --env FIRECRAWL_API_URL=http://127.0.0.1:3002 \
  -- npx -y firecrawl-mcp
```

This writes user-local state to `~/.codex/config.toml`. Do not commit it or copy
a macOS Codex config directly to the VPS. Keep writable roots limited to the
repository and agent-workspace paths, and validate the integration with:

```sh
codex mcp list
curl -fsS http://127.0.0.1:3002
```

For non-interactive `codex exec`, MCP approval can cancel a tool call. Use an
approval bypass only for a controlled smoke test in an empty workspace, never
as the normal operating mode.

## LangChain Documentation MCP

Add the official hosted documentation MCP for LangChain, LangGraph, and
LangSmith lookups:

```sh
codex mcp add langchain-docs --url https://docs.langchain.com/mcp
```

If the subcommand is unavailable, use the equivalent user-local config:

```toml
[mcp_servers.langchain-docs]
url = "https://docs.langchain.com/mcp"
```

Prefer the official documentation MCP. Third-party code-search MCP packages can
introduce separate login, data-handling, and credit models that need explicit
review.

## Remote Desktop

The optional NoMachine/XFCE profile is Tailscale-only and disabled by default.
See [Remote Desktop](remote-desktop.md).

## Agent Review Helpers

The workbench can install a read-only Codex review helper for Claude Code work.
See [Agent Workflows](agent-workflows.md).
