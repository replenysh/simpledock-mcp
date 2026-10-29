# Installing the SimpleDock MCP server (for AI agents)

SimpleDock is a **hosted remote** MCP server. Do not clone, build or run anything from this repo.

1. Make sure the user has a SimpleDock account (free plan works). If not, they can sign up at https://simpledock.ai.
2. Add the remote server to the MCP settings file (Cline: `cline_mcp_settings.json`):
```json
{
  "mcpServers": {
    "simpledock": {
      "url": "https://simpledock.ai/api/v1/mcp",
      "type": "streamableHttp"
    }
  }
}
```
   - Server name: `simpledock`
   - URL: `https://simpledock.ai/api/v1/mcp`
   - Transport: Streamable HTTP
   - No API key or environment variables are needed.
3. Save the settings. On the first connection the server returns `401` with OAuth metadata; the client performs OAuth 2.1 Dynamic Client Registration with PKCE and opens a browser for the user to sign in to SimpleDock and approve access.
4. Verify by calling `list_locations`. It should return the user's warehouse locations.
5. Typical flow: `list_locations` → `list_appointment_types` → `get_availability` → `create_appointment`.

Troubleshooting:
- 401 after sign-in: remove and re-add the server to re-run OAuth.
- If the client only supports stdio, use `{"command": "npx", "args": ["-y", "mcp-remote", "https://simpledock.ai/api/v1/mcp"]}`.
- Sign-in is the OAuth page the client opens. Email is confirmed via the link in the SimpleDock email. API keys are created in the SimpleDock app under Settings → API. Webhook signing secrets are copied from Settings → Integrations. Leave passwords, email codes, API keys, and signing secrets in the app: do not ask for them in chat, and do not read the user's inbox or any other connector.
- Help: support@simpledock.ai, https://simpledock.ai/developers/mcp
