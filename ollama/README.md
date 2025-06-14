# How to Install Ollama

This guide covers installing Ollama on Windows, Linux (Ubuntu), and using Docker.

## Windows Installation
1. Visit the official Ollama website: [https://ollama.com/download](https://ollama.com/download)
2. Download the Windows installer and run it.
3. Follow the on-screen instructions to complete the installation.

- Official documentation: [Ollama Windows Install Guide](https://github.com/ollama/ollama/blob/main/docs/windows.md)

## Linux (Ubuntu) Installation
1. Open a terminal and run:
   ```sh
   curl -fsSL https://ollama.com/install.sh | sh
   sudo systemctl start ollama
   ```
2. To enable Ollama to start on boot:
   ```sh
   sudo systemctl enable ollama
   ```

- Official documentation: [Ollama Linux Install Guide](https://github.com/ollama/ollama/blob/main/docs/linux.md)

## Docker Installation

### Using `docker run`
1. Run the following command:
   ```sh
   docker run -d -p 11434:11434 --name ollama ollama/ollama
   ```

### Using Docker Compose
1. Create a `docker-compose.yml` file with the following content:
   ```yaml
   version: '3'
   services:
     ollama:
       image: ollama/ollama
       ports:
         - "11434:11434"
   ```
2. Start the service:
   ```sh
   docker compose up -d
   ```

- Official documentation: [Ollama Docker Guide](https://github.com/ollama/ollama/blob/main/docs/docker.md)

---
For more details, visit the [Ollama Documentation](https://github.com/ollama/ollama/tree/main/docs).
