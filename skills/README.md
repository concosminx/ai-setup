## Useful AI/Automation Skills Repositories

- [obra/superpowers](https://github.com/obra/superpowers): A collection of scripts and tools to enhance developer productivity and automation, providing "superpowers" for various workflows.
- [GWUDCAP/cc-sessions](https://github.com/GWUDCAP/cc-sessions): Resources and session materials from the GWU Data, Code, and Policy (DCAP) group, focused on collaborative coding and data science practices.
- [ayoubben18/ab-method](https://github.com/ayoubben18/ab-method): Implementation and resources for the AB-Method, a technique for data analysis and machine learning experimentation.
- [alonw0/web-asset-generator](https://github.com/alonw0/web-asset-generator): Tools for generating and managing web assets, streamlining the process of creating optimized resources for web projects.
- [undeadlist/claude-code-agents](https://github.com/undeadlist/claude-code-agents): A collection of Claude-based code agents and automation scripts for advanced AI-driven workflows.
- [akin-ozer/cc-devops-skills](https://github.com/akin-ozer/cc-devops-skills): DevOps skills, templates, and automation resources for Claude Code and related AI projects.
- [sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills): A curated set of advanced skills and tools for the Antigravity platform, focused on automation and productivity.
- [affaan-m/everything-claude-code](https://github.com/affaan-m/everything-claude-code): Comprehensive resources, guides, and tools for working with Claude Code and AI-powered development.
- [glittercowboy/taches-cc-resources](https://github.com/glittercowboy/taches-cc-resources): A resource collection for Claude Code, featuring tools, guides, and assets to support automation and AI workflows.
- [browser-use/browser-use](https://github.com/browser-use/browser-use): Tools and scripts for browser automation, scraping, and web interaction workflows.
- [anthropics/skills](https://github.com/anthropics/skills): A collection of skills, templates, and resources for enhancing Claude and other AI agent capabilities.


## Install skills

- Code Reviewer: `npx claude-code-templates@latest --skill development/code-reviewer`
- Excalidraw Diagram Generator: `npx skills add https://github.com/coleam00/excalidraw-diagram-skill --skill excalidraw-diagram`

How to use: 

```html
Create an Excalidraw diagram showing how a request flows through
our API gateway, auth middleware, and downstream services
Generate an architecture diagram for a multi-tenant SaaS with
separate database schemas per tenant and a shared analytics layer
Draw a sequence diagram for our OAuth2 PKCE flow including
the browser, authorization server, and resource server
```

- Google Workspace (GWS) Skills

```sh
# Install
npm install -g @googleworkspace/cli

# Start MCP server with selected APIs
gws mcp -s drive,gmail,calendar,sheets

npx skills add https://github.com/googleworkspace/cli

# Claude now has direct access to these APIs
```

- Shannon - Autonomous AI Pentester: `npx skills add unicodeveloper/shannon`

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

- Anthropic Skills:
  - pdf `npx skills add https://github.com/anthropics/skills --skill pdf`
  - docx `npx skills add https://github.com/anthropics/skills --skill docx`
  - pptx `npx skills add https://github.com/anthropics/skills --skill pptx`
  - xlsx `npx skills add https://github.com/anthropics/skills --skill xlsx`
  - canvas-design `npx skills add https://github.com/anthropics/skills --skill canvas-design`
  - internal-comms `npx skills add https://github.com/anthropics/skills --skill internal-comms`
  - frontend-design `npx skills add https://github.com/anthropics/skills --skill frontend-design`
  - doc-coauthoring `npx skills add https://github.com/anthropics/skills --skill doc-coauthoring`
  - skill-creator `npx skills add https://github.com/anthropics/skills --skill skill-creator`
  - web-artifacts-builder `npx skills add https://github.com/anthropics/skills --skill web-artifacts-builder`
  - slack-gif-creator `npx skills add https://github.com/anthropics/skills --skill slack-gif-creator`

- Awesome-Skills
  - javascript-mastery: `npx skills add https://github.com/sickn33/antigravity-awesome-skills --skill javascript-mastery`
 
- Matt Pocock skills
  - `npx skills@latest add mattpocock/skills`