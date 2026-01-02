# SQLcl MCP Server

1. Read the [docs for SQLcl](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/working-sqlcl.html) and for [SQLcl MCP Server](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/25.2/sqcug/using-oracle-sqlcl-mcp-server.html)

2. Download SQLcl from [here](https://www.oracle.com/database/sqldeveloper/technologies/sqlcl/download/)

3. [Optional] Install an Oracle Database - check [Docker images](https://github.com/oracle/docker-images/tree/main/OracleDatabase) and [Oracle repository](https://container-registry.oracle.com)

```cmd
docker run --name xe11 -p 1521:1521 -e ORACLE_PWD=admin oracle/database:11.2.0.2
```
   
4. [Optional] Install an Oracle Sample Schema - check [schemas here](https://github.com/oracle-samples/db-sample-schemas)

5. Create an MCP compatible connection using the `SQLcl`
```sh
# Launch SQLcl
sql /nolog

# Create a new connection 'tom' and save the password
conn -save -savepwd tom tom/tom@localhost:1521/orclpdb1

#Test the saved connection
cm test tom

# Connect using the saved connection
conn -name tom

# Modify the connection if you change the user / password
alter user tom identified by tommy;
conn -save -savepwd -replace tom tom/tommy@localhost:1521/orclpdb1

# Optionally delete duplicate connections if needed
cm delete -conn tom

```

6. Setup in Visual Studio Code:
- install `Oracle SQL Developer Extension for VSCode` and create a connection

- add the MCP server
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
