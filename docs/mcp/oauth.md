---
description: "Secure a Pydantic AI MCP server with OAuth by placing it behind an identity-aware proxy or gateway, and review a production security checklist."
---

# OAuth

Production MCP servers that access upstream services, such as Google Drive, GitHub, or Slack, need to authenticate users and authorize tool calls. The [MCP Authorization specification](https://modelcontextprotocol.io/specification/latest/basic/authorization) defines an OAuth 2.1-based flow for this, and the MCP project publishes complementary [security best practices](https://modelcontextprotocol.io/docs/tutorials/security/security_best_practices) that all implementations should follow.

Pydantic AI itself does not implement an OAuth authorization server; instead, you should place your MCP server behind a dedicated gateway that handles authentication, token management, and access control. Known solutions for doing this with a Pydantic AI MCP server:

- [Pomerium](pomerium.md), an open source identity-aware proxy that handles both downstream client authentication and upstream OAuth for third-party APIs.

!!! note "Have another approach?"
    If you are using a different identity-aware proxy, API gateway, or OAuth provider to secure a Pydantic AI MCP server, contributions documenting the setup are welcome.

## Production security checklist

OAuth passthrough is not blanket tool authorization. A token scoped for `repo` access authorizes *all* GitHub repo operations — the MCP server still decides which tools to expose and to whom. Keep the following in mind before deploying:

- **Use least-privilege scopes.** Request only the scopes your tools actually need. Prefer narrower scopes where the provider supports them.
- **Enforce tool-level policies at the gateway.** Most gateways let you block specific tools by name at the proxy layer, before a request reaches your server — use this to prevent destructive tools from being called even if a client holds a valid token. See Pomerium's [Limit MCP Tool Calling](https://www.pomerium.com/docs/capabilities/mcp/limit-mcp-tools) for an example.
- **Don't forward arbitrary inbound headers to tools.** Your server receives a gateway-injected `Authorization` header containing the upstream token. Only extract the headers you expect — avoid passing raw request headers to tool implementations without validation.
- **Redact tokens from traces and logs.** If you use [Pydantic AI's Logfire integration](../logfire.md) or any other tracing, ensure the `Authorization` header and token values are scrubbed before they reach your observability backend.
- **Know where the trust boundary sits.** Your gateway authenticates the MCP client and manages the upstream token. Your MCP server trusts that the gateway has validated the request — it should not re-implement auth, but it also should not assume the injected token grants more access than its scopes allow. The upstream API is the final authority on what that token can do.
