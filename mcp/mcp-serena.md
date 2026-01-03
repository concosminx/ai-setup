# Serena MCP

1. Official [docs](https://github.com/oraios/serena)

2. Install uv
```cmd
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```
3. Setup in Visual Studio Code (remote):
```json
"mcp": {
  "servers": {
    "serena": {
      "command": "uvx",
      "args": [
        "--from", 
        "git+https://github.com/oraios/serena", 
        "serena-mcp-server",
        "--context", 
        "desktop-app"
      ]
    }
  }
}
```

4. For Any Project:
" Please activate my project at [drag and drop your project folder here] and perform onboarding to understand the codebase."

5. What You Can Ask After Setup:

- "Summarize this project's architecture"
- "Help me implement user authentication"
- "Find and fix bugs in the payment logic"
- "Refactor the database connection code"
- "Add tests for the API endpoints"
- "What are the main security concerns?"


