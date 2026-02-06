# AI Workflow Stack

A complete local AI automation stack featuring **n8n** for workflow automation, **Ollama** for running local LLMs, **OpenWebUI** for a chat interface, and **PostgreSQL** for data persistence.

## Overview

This project provides a containerized environment for building AI-powered workflows with the following components:

| Service | Description | Port |
|---------|-------------|------|
| [n8n](https://n8n.io/) | Workflow automation platform | 5678 |
| [Ollama](https://ollama.com/) | Local LLM runner | 11434 |
| [OpenWebUI](https://github.com/open-webui/open-webui) | Chat interface for LLMs | 3000 |
| PostgreSQL | Database for n8n persistence | 5432 |

## Getting Started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (v20.10 or later)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.0 or later)
- At least 8GB of free RAM (16GB+ recommended for running larger models)
- 10GB+ of free disk space for models and data

### Quick Start

1. **Clone or navigate to the project directory:**
  ```bash
   cd /Users/robertomagallanes/Developer/Docker/n8nTest
  ```

2. **Create environment variables (optional but recommended):**
  ```bash
   export POSTGRES_USER=n8n
   export POSTGRES_PASSWORD=your_secure_password
   export N8N_BASIC_AUTH_USER=admin
   export N8N_BASIC_AUTH_PASSWORD=your_secure_password
   export WEBUI_SECRET_KEY=your_secret_key
   export TIMEZONE=America/Los_Angeles
   ```

3. **Start all services:**
   ```bash
   docker-compose up -d
  ```

4. **Access the services:**
   - n8n: http://localhost:5678
   - OpenWebUI: http://localhost:3000
   - Ollama API: http://localhost:11434

## Cloudflare Tunnel
Quick tunnels create a temporary, short-lived public URL that forwards traffic to your local n8n instance — great for ad-hoc testing, but NOT suitable for production webhooks because the URL changes when the tunnel restarts.

This project includes a `cloudflared` service in `docker-compose.yml` that can run a persistent (named) Cloudflare Tunnel. The recommended flow for production-like stability is:

1. Create a named tunnel locally (requires Cloudflare account access):

```bash
# create a named tunnel and capture the generated tunnel id
cloudflared tunnel create n8n-tunnel
```

2. Copy the generated credentials JSON into the project folder (local-only):

```bash
# create the folder and copy the credentials file produced by the create step
mkdir -p .cloudflared
# move or copy ~/.cloudflared/<TUNNEL-UUID>.json -> .cloudflared/
```

3. Create a tunnel config file (use the provided template `.cloudflared/config.yml.template`):

```yaml
# .cloudflared/config.yml (example)
tunnel: <TUNNEL-UUID>
credentials-file: /etc/cloudflared/<TUNNEL-UUID>.json
ingress:
  - hostname: n8n.lation.com.mx
    service: http://n8n:5678
  - service: http_status: 404
```

4. Start the `cloudflared` container (the service mounts `.cloudflared` at `/etc/cloudflared`):

```bash
docker compose up -d cloudflared n8n
docker compose logs -f cloudflared
```

5. In the Cloudflare dashboard DNS for `lation.com.mx` create a CNAME for the hostname you want to expose (for example `n8n`) and point it to the tunnel target shown by `cloudflared` (usually `<tunnel-uuid>.cfargotunnel.com`). Follow the Cloudflare docs for mapping a tunnel to a hostname if you prefer the UI.

6. Update your `.env` so n8n builds stable webhook URLs (example):

```env
N8N_PROTOCOL=https
N8N_HOST=n8n.lation.com.mx
WEBHOOK_URL=https://n8n.lation.com.mx/
```

Security notes:
- Keep `.cloudflared/` and credentials out of git (this repo already gitignores `.cloudflared`).
- Protect `n8n.lation.com.mx` with Cloudflare Access (Zero Trust) and keep n8n basic auth enabled for defense-in-depth.
- Avoid exposing database ports (Postgres) to the public host in production.

See `.cloudflared/config.yml.template` for a starting config. Do NOT commit your actual credentials JSON into the repository.

## Development Setup

### Environment Variables

Create a `.env` file in the project root for easier configuration:

```env
# Database Configuration
POSTGRES_USER=n8n
POSTGRES_PASSWORD=your_secure_password_here
POSTGRES_DB=n8n

# n8n Authentication
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_password_here

# OpenWebUI Secret
WEBUI_SECRET_KEY=your_random_secret_key

# Timezone
TIMEZONE=America/Los_Angeles
```

### Data Persistence

All service data is persisted using Docker volumes:

- `postgres_data` - PostgreSQL database files
- `n8n_data` - n8n workflows and credentials
- `ollama_data` - Downloaded LLM models
- `openwebui_data` - OpenWebUI configuration and chat history

Data is stored in the `data/` directory on the host for easy backup and migration.

## Usage

### Managing Services

```bash
# Start all services
docker-compose up -d

# Stop all services
docker-compose down

# Stop and remove all data (WARNING: deletes all data)
docker-compose down -v

# View logs
docker-compose logs -f

# View logs for a specific service
docker-compose logs -f n8n

# Restart a specific service
docker-compose restart ollama
```

### Working with Ollama

Pull and run LLM models:

```bash
# Pull a model
docker exec -it ollama ollama pull llama3.2

# List available models
docker exec -it ollama ollama list

# Run a model interactively
docker exec -it ollama ollama run llama3.2
```

### n8n Workflow Automation

1. Access n8n at http://localhost:5678
2. Log in with your configured credentials
3. Create workflows that can:
   - Call Ollama models via HTTP requests
   - Integrate with external APIs
   - Automate data processing tasks

### OpenWebUI Chat Interface

1. Access OpenWebUI at http://localhost:3000
2. Create an account on first run
3. Select from available Ollama models
4. Start chatting with local LLMs

## Network Configuration

All services communicate through a dedicated Docker network `stack-ai`:

- n8n → PostgreSQL (port 5432)
- OpenWebUI → Ollama (port 11434)
- External access via mapped host ports

## Troubleshooting

### Services won't start

Check Docker logs for specific errors:
```bash
docker-compose logs
```

### n8n database connection issues

Ensure PostgreSQL is healthy before n8n starts:
```bash
docker-compose ps
```

### Ollama models not loading

Check available disk space and model downloads:
```bash
docker exec ollama ollama list
df -h
```

### Port conflicts

If ports are already in use, modify the port mappings in `docker-compose.yml`:
```yaml
ports:
  - "8080:5678"  # Use 8080 instead of 5678
```

## Security Considerations

- Change default passwords in production
- Use strong, unique passwords for all services
- Consider using a reverse proxy (nginx/traefik) with SSL for external access
- The n8n webhook URL is configured for Cloudflare tunnel - update for your domain
- Keep Docker images updated regularly

## Backup and Restore

### Backup

```bash
# Backup all data volumes
tar -czvf backup-$(date +%Y%m%d).tar.gz data/
```

### Restore

```bash
# Stop services
docker-compose down

# Restore data
tar -xzvf backup-20240101.tar.gz

# Restart services
docker-compose up -d
```

## Updating

### Update All Services

```bash
# Pull latest images for all services
docker-compose pull

# Recreate containers with new images (preserves data)
docker-compose up -d
```

### Update Individual Services

```bash
# Update only n8n
docker-compose pull n8n
docker-compose up -d n8n

# Update only Ollama
docker-compose pull ollama
docker-compose up -d ollama

# Update only OpenWebUI
docker-compose pull openwebui
docker-compose up -d openwebui

# Update only PostgreSQL
docker-compose pull postgres
docker-compose up -d postgres
```

### Update Ollama Models

```bash
# List currently installed models
docker exec ollama ollama list

# Update/pull latest version of a specific model
docker exec ollama ollama pull llama3.2
docker exec ollama ollama pull mistral
docker exec ollama ollama pull codellama

# Update all models (pull latest versions)
docker exec ollama ollama pull llama3.2
docker exec ollama ollama pull mistral
# Add other models you have installed
```

### Check Current Versions

```bash
# Check n8n version
docker exec n8n n8n --version

# Check Ollama version
docker exec ollama ollama --version

# Check PostgreSQL version
docker exec n8n-postgres postgres --version

# Check OpenWebUI version (via container image)
docker inspect openwebui | grep -i version

# Check all image versions
docker-compose images
```

### Clean Up Old Images

```bash
# Remove unused/old images after updates
docker image prune -f

# Remove all unused volumes (WARNING: be careful!)
docker volume prune -f

# Full system cleanup (unused images, containers, networks)
docker system prune -f
```

## Resources

- [n8n Documentation](https://docs.n8n.io/)
- [Ollama Documentation](https://github.com/ollama/ollama/blob/main/README.md)
- [OpenWebUI Documentation](https://docs.openwebui.com/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)

## License

This project is provided as-is for educational and development purposes.
