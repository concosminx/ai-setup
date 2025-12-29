# n8n-MCP

1. Official [docs](https://github.com/czlonkowski/n8n-mcp)

2. Create an API key from n8n - [docs](https://docs.n8n.io/api/authentication/#call-the-api-using-your-key)

3. Setup in Visual Studio Code (NPX):
```json
"mcp": {
  "servers": {
    "n8n-mcp": {
      "command": "npx",
      "args": ["n8n-mcp"],
      "env": {
        "MCP_MODE": "stdio",
        "LOG_LEVEL": "error",
        "DISABLE_CONSOLE_OUTPUT": "true",
        "N8N_API_URL": "https://your-n8n-instance.com",
        "N8N_API_KEY": "your-api-key"
      }
    }
  }
}
```
