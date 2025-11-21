# MCP Configuration

## Setup Instructions

1. Copy `.mcp-config.json.example` to your VS Code settings or MCP configuration location
2. Fill in your Confluence credentials:
   - `CONFLUENCE_URL`: Your Confluence instance (e.g., https://your-domain.atlassian.net)
   - `CONFLUENCE_USERNAME`: Your Confluence email
   - `CONFLUENCE_API_TOKEN`: Generate from Confluence Account Settings → Security → API Tokens

3. Never commit the actual configuration with credentials to Git

## Environment Variables (Alternative)

You can also set these as environment variables:

```powershell
# PowerShell
$env:CONFLUENCE_URL = "https://your-domain.atlassian.net"
$env:CONFLUENCE_USERNAME = "your-email@domain.com"
$env:CONFLUENCE_API_TOKEN = "your-api-token"
```

## Verifying MCP Configuration

Once configured, you should have access to Confluence MCP tools for:
- Creating and updating pages
- Searching Confluence
- Managing documentation

See `confluence/mcp-confluence-setup.md` for detailed usage instructions.
