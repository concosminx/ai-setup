# SQLcl MCP Server

1. Read the [docs for SQLcl](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/working-sqlcl.html) and for [SQLcl MCP Server](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/using-oracle-sqlcl-mcp-server.html)

2. Setup in Visual Studio Code (remote):
```json
"mcp": {
  "servers": {
    "sqlcl": {
      "command": "path_to/sqlcl/bin/sql",
      "args": [
        "-mcp"
      ],
      "type": "stdio"
    }
  }
}
```
