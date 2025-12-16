# Run and Test (local)

Purpose
- A concise runbook for developers and operators to build, run, verify, and troubleshoot the PayFlow stack locally. Includes quick commands for builds, healthchecks, vulnerability scanning, and basic incident/chaos exercises.

Prerequisites
- Docker and Docker Compose installed
- (Optional) Trivy installed for scanning

Build images and start stack

```bash
# From repo root
docker compose build --pull --progress=plain
docker compose up -d

# View logs
docker compose logs -f
```

Verify auth-service readiness and health

```bash
# Rebuild and start only auth-service (uses updated Dockerfile and server.js)
docker compose up -d --no-deps --build auth-service

# Follow logs while the service initializes
docker compose logs -f auth-service

# Check container health status
docker ps
docker inspect --format='{{json .State.Health}}' $(docker ps -q --filter name=payflow-wallet-auth-service)

# If healthchecks fail with "node: executable file not found", rebuild after applying PATH fix in Dockerfile
docker compose up -d --no-deps --build auth-service
```

Run Trivy scans (after images built)

```bash
# Build images with explicit names (optional). By default compose tags images as <folder>_service
# Run the provided helper (make it executable first):
chmod +x ./scripts/scan-images.sh
./scripts/scan-images.sh

# Or scan a specific image
trivy image payflow_auth-service:latest
```

Basic Incident Response commands

- Check container status:

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

- Check service logs:

```bash
docker compose logs auth-service --tail=200
# or
docker logs <container_id>
```

- Restart a single service:

```bash
docker compose restart auth-service
```

- Recreate service (pulls new image/builds):

```bash
docker compose up -d --no-deps --build auth-service
```

- Bring down the stack and bring back up:

```bash
docker compose down
docker compose up -d
```

Memory throttling test (simulate low memory):

```bash
# Limit container to 120MB runtime
docker update --memory 120m <container_id>
```

Network removal (chaos):

```bash
docker network rm payflow-wallet_default
# Recover by recreating stack
docker compose up -d
```
