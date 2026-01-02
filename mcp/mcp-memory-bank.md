# Memory Bank MCP Server

1. Official [docs](https://github.com/alioshr/memory-bank-mcp)

2. Setup in Visual Studio Code (remote):
```json
"mcp": {
  "servers": {
    "allPepper-memory-bank": {
      "type": "stdio",
      "command": "npx",
      "args": [
        "-y",
        "@allpepper/memory-bank-mcp@latest"
      ],
      "env": {
        "MEMORY_BANK_ROOT": "YOUR PATH"
      }
    }
  }
}
```
