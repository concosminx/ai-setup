# Shannon: Autonomous AI Pentester

How to install: `npx skills add unicodeveloper/shannon`

How to run: 

```sh
# Full pentest of a local app

/shannon http://localhost:3000 myapp

# Target specific vulnerability categories
/shannon - scope=xss,injection http://localhost:8080 frontend

# Named workspace (for resuming if interrupted)
/shannon - workspace=audit-q1 http://staging.example.com backend-api

# Check status of a running pentest
/shannon status

# View the latest report
/shannon results
```