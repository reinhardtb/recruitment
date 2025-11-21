# MCP Confluence Integration Guide

## Overview
This guide explains how to configure and use Model Context Protocol (MCP) for Confluence integration to publish your architecture documentation.

## Prerequisites
- Confluence account and access
- Confluence API token
- VS Code with MCP support

## Configuration Steps

### 1. Install Confluence MCP Server
You'll need to configure the Confluence MCP server. This is typically done in your MCP settings.

### 2. Confluence API Token
1. Log in to your Confluence instance
2. Navigate to Account Settings → Security → API Tokens
3. Create a new API token
4. Save the token securely (you won't be able to see it again)

### 3. MCP Configuration
Create or update your MCP configuration file with Confluence server details:

```json
{
  "mcpServers": {
    "confluence": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-confluence"
      ],
      "env": {
        "CONFLUENCE_URL": "https://your-domain.atlassian.net",
        "CONFLUENCE_USERNAME": "your-email@domain.com",
        "CONFLUENCE_API_TOKEN": "your-api-token"
      }
    }
  }
}
```

### 4. Environment Variables (Alternative)
Alternatively, set these environment variables:
- `CONFLUENCE_URL`: Your Confluence instance URL
- `CONFLUENCE_USERNAME`: Your Confluence username/email
- `CONFLUENCE_API_TOKEN`: Your API token

## Using Confluence MCP Tools

Once configured, you'll have access to Confluence operations through MCP:

### Available Operations
- Create new pages
- Update existing pages
- Search for pages
- Get page content
- Manage page hierarchy
- Add comments
- Attach files

### Publishing Documentation Workflow

#### 1. Prepare Documentation
- Write your architecture documentation in Markdown
- Include diagrams (as images or Mermaid/PlantUML)
- Review and organize content

#### 2. Create Confluence Space Structure
- Determine page hierarchy
- Plan parent-child relationships
- Organize by architecture domains

#### 3. Publish Pages
Use MCP tools to:
- Create parent page for architecture documentation
- Create child pages for each architecture domain
- Upload diagrams and attachments
- Set appropriate permissions

#### 4. Maintain Documentation
- Version control in Git
- Sync updates to Confluence
- Track changes and reviews

## Best Practices

### Documentation Organization in Confluence
```
📄 [System Name] Architecture Documentation
├── 📄 System Overview
├── 📄 Architecture Decisions
│   ├── 📄 ADR-001: [Decision]
│   ├── 📄 ADR-002: [Decision]
│   └── 📄 ADR-003: [Decision]
├── 📄 Component Architecture
├── 📄 Data Architecture
├── 📄 Security Architecture
└── 📄 Integration Architecture
```

### Markdown to Confluence
- Confluence uses its own markup language
- MCP handles conversion automatically
- Test with a small page first
- Review formatting after publishing

### Diagrams
- Export diagrams as PNG/SVG
- Include alt text for accessibility
- Store original diagram sources in Git
- Upload as attachments to Confluence pages

### Version Control
- Keep source documentation in Git
- Use Confluence for presentation/collaboration
- Sync regularly
- Tag releases

## Troubleshooting

### Common Issues

#### Authentication Fails
- Verify API token is correct
- Check username/email is correct
- Ensure Confluence URL includes protocol (https://)

#### Permission Errors
- Verify you have space permissions
- Check page permissions
- Contact Confluence admin if needed

#### Content Formatting Issues
- Test Markdown conversion
- Simplify complex Markdown
- Use Confluence-native tables if needed

## Security Considerations

### API Token Security
- Never commit tokens to Git
- Use environment variables
- Rotate tokens regularly
- Limit token permissions

### Sensitive Information
- Review docs before publishing
- Redact sensitive data
- Use appropriate Confluence permissions
- Consider separate spaces for internal/external docs

## Next Steps
1. Set up Confluence space
2. Configure MCP with credentials
3. Test with a sample page
4. Publish architecture documentation
5. Share with interview panel

---
*Note: For Booking.com interview, ensure documentation is professional and complete before publishing.*
