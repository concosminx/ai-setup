# How to Install Open WebUI with Docker and Docker Compose

This guide explains how to install Open WebUI using Docker and Docker Compose, with links to the official documentation.

## Docker Installation

1. Pull the Open WebUI image and run it:
   ```sh
   docker run -d -p 8080:8080 --name open-webui open-webui/open-webui
   ```
2. Access Open WebUI at [http://localhost:8080](http://localhost:8080)

- Official Docker instructions: [Open WebUI Docker Guide](https://github.com/open-webui/open-webui?tab=readme-ov-file#quick-start-with-docker-)

## Docker Compose Installation

1. Create a `docker-compose.yml` file with the following content:
   ```yaml
   version: '3'
   services:
     open-webui:
       image: open-webui/open-webui
       ports:
         - "8080:8080"
   ```
2. Start the service:
   ```sh
   docker compose up -d
   ```
3. Access Open WebUI at [http://localhost:8080](http://localhost:8080)

- Official Docker Compose instructions: [Open WebUI Docker Compose Guide](https://github.com/open-webui/open-webui?tab=readme-ov-file#other-installation-methods)

---
For more details, visit the [Open WebUI GitHub Repository](https://github.com/open-webui/open-webui).
