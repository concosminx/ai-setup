# Sequential Thinking MCP Server

1. Official [docs](https://github.com/modelcontextprotocol/servers/tree/main/src/sequentialthinking)

2. Setup in Visual Studio Code (Docker):
```json
"mcp":{
   "servers":{
      "sequentialthinking":{
         "command":"docker",
         "args":[
            "run",
            "--rm",
            "-i",
            "mcp/sequentialthinking"
         ]
      }
   }
}
```
