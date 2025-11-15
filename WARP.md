# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Repository Overview

This repository manages a multi-container Docker infrastructure designed to run alongside Plesk. Each service is isolated in its own directory with a dedicated `docker-compose.yml` file. All services use Traefik as a reverse proxy to avoid port conflicts with Plesk.

### Key Architecture Decisions

- **Traefik runs on non-standard ports**: Uses ports 8080 (HTTP) and 8443 (HTTPS) instead of standard 80/443 to avoid conflicts with Plesk
- **External networks are required**: Services communicate via two pre-created external networks: `traefik_proxy` and `mysql_net`
- **MySQL runs on port 3307**: Remapped to avoid conflict with Plesk's MySQL on 3306
- **Environment-based configuration**: Each service directory contains a `.env` file with service-specific configuration (git-ignored)
- **SSL handled by Plesk**: Let's Encrypt is disabled in Traefik; SSL certificates are managed by Plesk

## Network Prerequisites

Before deploying any service, ensure the external Docker networks exist:

```bash
docker network create traefik_proxy
docker network create mysql_net
```

## Service Management

Each service is managed independently from its own directory.

### Starting a service
```bash
cd <service-name>
docker-compose up -d
```

### Stopping a service
```bash
cd <service-name>
docker-compose down
```

### Viewing service logs
```bash
cd <service-name>
docker-compose logs -f
```

### Restarting a service
```bash
cd <service-name>
docker-compose restart
```

## Services in this Stack

- **traefik/**: Reverse proxy (ports 8080, 8443 on localhost only)
- **mysql/**: MySQL 8.0 database server (port 3307)
- **portainer/**: Docker management UI
- **netdata/**: System monitoring dashboard
- **dozzle/**: Real-time Docker log viewer
- **uptime/**: Uptime Kuma service monitoring
- **bugsink/**: Error tracking application

## Environment Configuration

Each service requires a `.env` file in its directory. Common environment variables include:

- `SERVICE_URL_WEB`: The domain/subdomain for Traefik routing (e.g., `portainer.example.com`)
- `SERVICE_USER_*`: Service-specific usernames
- `SERVICE_PASSWORD_*`: Service-specific passwords
- `MYSQL_ROOT_PASSWORD`: MySQL root password (in mysql/.env)

When adding a new service that needs Traefik routing, ensure it has:
1. A `.env` file with `SERVICE_URL_WEB` defined
2. Traefik labels in docker-compose.yml:
   - `traefik.enable=true`
   - `traefik.http.routers.<name>.rule=Host(\`${SERVICE_URL_WEB}\`)`
   - `traefik.http.routers.<name>.entrypoints=websecure`
   - `traefik.http.services.<name>.loadbalancer.server.port=<port>`
3. Connection to `traefik_proxy` network

## Common Operations

### Adding a new service

1. Create a new directory for the service
2. Create `docker-compose.yml` with proper Traefik labels and network connections
3. Create `.env` file with required variables (especially `SERVICE_URL_WEB`)
4. Add `.gitignore` to exclude `.env` files
5. Ensure service connects to `traefik_proxy` network (and `mysql_net` if needed)

### Checking MySQL connectivity

From any service that needs database access:
```bash
docker exec -it <container-name> mysql -h mysql_server -P 3307 -u <user> -p
```

### Inspecting Traefik configuration

View Traefik dashboard (requires authentication setup):
```bash
curl http://127.0.0.1:8080/api/http/routers
```

### Troubleshooting service routing

Check if service is registered with Traefik:
```bash
docker exec traefik wget -qO- http://localhost:8080/api/http/services
```

Verify container is on correct network:
```bash
docker inspect <container-name> | grep -A 10 Networks
```

## Important Constraints

- Never use ports 80, 443, or 3306 (reserved by Plesk)
- All web services must go through Traefik on ports 8080/8443
- Always use external networks (`traefik_proxy`, `mysql_net`) - never create them in compose files
- Environment files (`.env`) must never be committed to git
- Services requiring databases should reference `mysql_server:3307` as the host
