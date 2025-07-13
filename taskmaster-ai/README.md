# How to Install TaskMaster AI

TaskMaster AI is an AI-powered task management and automation tool. Follow the steps below to install it on your system.

## Prerequisites

- Node.js 14 or higher
- npm (Node Package Manager)

## Installation Methods

### Global Installation

1. Install TaskMaster AI globally from npm:
   ```
   npm install -g task-master-ai
   ```

### Project Installation

1. Install TaskMaster AI for current project from npm:
   ```
   npm install task-master-ai
   ```

## Getting Started

After installation, you can start using TaskMaster AI by running:
```
task-master init
```

This will create the necessary configuration files and guide you through the initial setup.

## Adding TaskMaster AI as an MCP Server in VS Code

To use TaskMaster AI as a Model Context Protocol (MCP) server in VS Code:

1. Open VS Code settings (Ctrl+,) and search for "MCP"

2. Add TaskMaster AI to your MCP servers configuration:
   ```json
   {
     "mcpServers": {
       "taskmaster-ai": {
         "command": "npx",
           "args": ["-y", "--package=task-master-ai", "task-master-ai"],
           "env": {
             "ANTHROPIC_API_KEY": "YOUR_ANTHROPIC_API_KEY_HERE",
             "PERPLEXITY_API_KEY": "YOUR_PERPLEXITY_API_KEY_HERE",
             "MODEL": "claude-3-7-sonnet-20250219",
             "PERPLEXITY_MODEL": "sonar-pro",
             "MAX_TOKENS": 64000,
             "TEMPERATURE": 0.2,
             "DEFAULT_SUBTASKS": 5,
             "DEFAULT_PRIORITY": "medium"
           }
         }
       }
   }
   ```

3. Restart VS Code or reload the window (Ctrl+Shift+P → "Developer: Reload Window")

4. TaskMaster AI should now be available as an MCP server in your VS Code environment

## Official Documentation

- [TaskMaster AI ](https://www.task-master.dev/)
- [GitHub Repository](https://github.com/eyaltoledano/claude-task-master) 
- [Usage example](https://devopstoolkit.live/ai/from-shame-to-fame-how-i-fixed-my-lazy-vibe-coding-habits-with-taskmaster/)

