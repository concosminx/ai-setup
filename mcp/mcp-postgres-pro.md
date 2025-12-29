# Postgres MCP Pro

1. Official [docs](https://github.com/crystaldba/postgres-mcp)

2. Setup in Visual Studio Code (Docker):
```json
"mcp": {
  "servers": {
    "postgres": {
      "command": "uv",
      "args": [
        "run",
        "postgres-mcp",
        "--access-mode=unrestricted"
      ],
      "env": {
        "DATABASE_URI": "postgresql://username:password@localhost:5432/dbname"
      }
    }
  }
}
```
