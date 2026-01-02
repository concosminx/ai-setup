# Supabase MCP Server

1. Official [docs](https://supabase.com/blog/mcp-server)

2. Setup in Visual Studio Code (remote):
```json
"mcp": {
  "servers": {
    "supabase": {
      "command": "npx",
      "args": [
        "-y",
        "@supabase/mcp-server-supabase@latest",
        "--access-token",
        "<personal-access-token>"
      ]
    }
  }
}
```

3. Get the [personal-access-token](https://supabase.com/dashboard/account/tokens) from your account 
