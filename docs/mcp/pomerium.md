---
description: "Put a Pydantic AI MCP server behind Pomerium to authenticate MCP clients and pass upstream OAuth tokens, such as GitHub's, to your agent's tools."
---

# Pomerium

[Pomerium](https://pomerium.com) is an open source identity-aware proxy that can sit in front of an MCP server and handle OAuth on both sides: it authenticates MCP clients, and it can obtain tokens for upstream APIs such as GitHub or Google Drive. Your server doesn't implement any OAuth itself.

## Putting your server behind Pomerium

Serve your MCP server over the Streamable HTTP transport, for example by replacing `server.run()` in the [MCP server example](server.md#mcp-server) with `server.run(transport='http', host='0.0.0.0', port=8080)`. Then add a Pomerium route with an `mcp.server` block that points at the server's `/mcp` endpoint. MCP clients, including Pydantic AI's [MCP client](client.md), connect to the route's URL and sign in through Pomerium.

See Pomerium's [Protect an MCP Server](https://www.pomerium.com/docs/capabilities/mcp/protect-mcp-server) guide for route, policy, and MCP client configuration, and the [MCP reference](https://www.pomerium.com/docs/capabilities/mcp/reference) for all MCP settings.

## Using upstream tokens in your agent

When a route has an `upstream_oauth2` block, Pomerium runs the upstream provider's OAuth flow for each user and sends the user's access token to your server in the `Authorization: Bearer` header of every proxied request. See Pomerium's [MCP + Upstream OAuth](https://www.pomerium.com/docs/capabilities/mcp/mcp-upstream-oauth) guide for the configuration for GitHub, Google, Notion, and other providers.

Read the token from the incoming request and pass it to the agent as [dependencies](../dependencies.md), so the agent's tools can call the upstream API on the user's behalf:

```py {title="mcp_server_github.py"}
from dataclasses import dataclass

import httpx
from fastmcp import FastMCP
from fastmcp.server.dependencies import get_http_request

from pydantic_ai import Agent, RunContext


@dataclass
class GitHubDeps:
    token: str
    client: httpx.AsyncClient


server = FastMCP('GitHub Assistant')
github_agent = Agent(
    '{{ openai:medium }}',
    deps_type=GitHubDeps,
    instructions="Answer questions about the user's GitHub repositories.",
)


@github_agent.tool
async def list_repos(ctx: RunContext[GitHubDeps]) -> list[str]:
    """List repositories the authenticated user can access."""
    response = await ctx.deps.client.get(
        'https://api.github.com/user/repos',
        headers={'Authorization': f'Bearer {ctx.deps.token}'},
    )
    response.raise_for_status()
    return [repo['full_name'] for repo in response.json()]


@server.tool()
async def github_assistant(question: str) -> str:
    """Answer a question about the caller's GitHub repositories."""
    # Pomerium sets this header to the user's upstream GitHub token.
    authorization = get_http_request().headers.get('authorization', '')
    token = authorization.removeprefix('Bearer ')
    async with httpx.AsyncClient() as client:
        result = await github_agent.run(
            question, deps=GitHubDeps(token=token, client=client)
        )
    return result.output


if __name__ == '__main__':
    server.run(transport='http', host='0.0.0.0', port=8080)
```

Before going to production, review the [OAuth security checklist](oauth.md#production-security-checklist).
