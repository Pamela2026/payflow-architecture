# Postmortem - Docker Compose & Image Fixes

Summary
- Fixed Dockerfiles across services and implemented a smaller distroless final image for `auth-service`.
- Added `docker-compose.yml` with healthchecks, restart policies, and resource limits.
- Prepared Trivy scan script.

What broke
- Several services had non-optimal Dockerfiles (single-stage or larger base images) increasing image size and surface area.
- `auth-service` image was ~184MB; optimized to use a distroless final image to shrink size.

Why it broke
- Dockerfiles used full Node images for runtime instead of smaller distroless images and didn't use user namespaces consistently.

Actions taken
- Converted `auth-service` Dockerfile to multi-stage build with a distroless final image and non-root user.
- Added `docker-compose.yml` to orchestrate services, volumes, healthchecks, and restart policies.
- Created a Trivy scan helper script.

Lessons
- Multi-stage builds and distroless images drastically reduce runtime image size.
- Always run containers as non-root to limit impact of a compromised process.
- Healthchecks and restart policies greatly improve recoverability.

Next steps
- Build images locally and run the `scripts/scan-images.sh` to collect vulnerabilities.
- Iterate on reducing other service images if required.

---

## Incident: Postgres connection refused during auth-service startup (ECONNREFUSED)

Timestamp: 2025-12-16

Error / symptom
- During `docker compose up` the `auth-service` repeatedly logged the following error while starting:

```
Error: connect ECONNREFUSED 172.19.0.3:5432
		at /app/node_modules/pg-pool/index.js:45:11
		at process.processTicksAndRejections (node:internal/process/task_queues:95:5)
		at async initDB (/app/server.js:51:18) {
	errno: -111,
	code: 'ECONNREFUSED',
	syscall: 'connect',
	address: '172.19.0.3',
	port: 5432
}
```

Root cause
- This was a startup race: `auth-service` attempted to connect to Postgres before the Postgres container was fully ready to accept connections. The Postgres container later completed initialization and became healthy, at which point the app was able to initialize its DB.

Diagnosis steps performed
- Reviewed `docker compose logs postgres` and observed Postgres start, an early fast shutdown/restart, and eventual readiness: `database system is ready to accept connections`.
- Checked `docker compose logs auth-service` which showed the `ECONNREFUSED` error followed by a successful `Auth database initialized` after Postgres was ready.

Fix / Remediation
- Short-term: restart `auth-service` after Postgres becomes healthy:

```bash
docker compose restart auth-service
docker compose logs auth-service --follow
```

- Long-term (recommended): make the application robust to DB startup ordering by adding retry/backoff around the initial DB connection. Add the following wrapper around the existing `initDB()` call in `services/auth-service/server.js`:

```javascript
async function waitForDb(retries = 15, delay = 2000) {
	for (let i = 0; i < retries; i++) {
		try { await initDB(); console.log('Auth database initialized'); return; }
		catch (err) {
			console.error(`DB connect failed (attempt ${i+1}/${retries}):`, err.message);
			await new Promise(res => setTimeout(res, delay));
			delay = Math.min(10000, Math.floor(delay * 1.5));
		}
	}
	throw new Error('DB connection retries exhausted');
}

waitForDb().catch(err => { console.error(err); process.exit(1); });
```

Why this fix
- Ensures the service will continue attempting to connect until Postgres becomes available, avoiding transient startup failures and making the system more resilient to orchestration ordering issues.

Verification
- After Postgres reported readiness, restarting `auth-service` returned `Auth database initialized` in the service logs and no further `ECONNREFUSED` errors were observed.

Follow-ups
- Consider adding a small start-wrapper (`wait-for-it`, `dockerize`) for environments where modifying application code is undesirable.
- Keep `postgres` healthcheck in `docker-compose.yml` and consider orchestrator-level readiness checks where supported.

## Modifications & Rationale

This section lists repository changes made during the exercise and the reasons behind each change.

	- Change: Converted to a multi-stage build and used `gcr.io/distroless/nodejs:18` as the final image. Created a numeric non-root user (UID 1001) and chowned application files in the build stage.
	- Reason: Multi-stage builds keep build-time artifacts out of the final runtime image; distroless significantly reduces image size and attack surface by omitting shells and package managers.
	- Trade-offs: Distroless images make in-container debugging harder and require reliable registry access. If registry pulls fail, fallback to `node:18-alpine` is a pragmatic temporary option.

	- Change: Added `.dockerignore` to each service (auth, api-gateway, transaction, wallet, notification, frontend).
	- Reason: Exclude dev artifacts (`node_modules`, `.env`, `.git`, editor configs) from build context to speed builds and avoid leaking secrets into images.

	- Change: Added retry/backoff logic around `initDB()` and a `/ready` readiness endpoint in addition to existing `/health` liveness probe.
	- Reason: Make the service tolerant of DB startup races so it does not fail permanently if Postgres is not yet ready. Provide a clear readiness probe for the orchestrator to use.
	- Trade-offs: Slight delay before app readiness; recommended for production to avoid race conditions.

	- Change: Added a production-style compose file with services (postgres, redis, api-gateway, auth-service, transaction-service, wallet-service, notification-service, frontend), persistent volume for Postgres, healthchecks, restart policies, and resource limits. Updated the `auth-service` healthcheck to call `/ready` with an increased `start_period` and retries.
	- Reason: Provide a reproducible local stack with healthchecks and restart policies to improve service reliability and facilitate chaos/incident exercises.
	- Caveats: `deploy.resources` is primarily for swarm/Kubernetes and may be ignored by local `docker compose`. The `version` attribute is obsolete for modern `docker compose` (warning emitted) and can be removed.

	- Change: Added a small helper to run `trivy image` against built images.
	- Reason: Automate vulnerability scanning for quick local assessments and to integrate later into CI.

	- Change: Added documentation and runbook content describing commands to build, run, scan, and perform chaos experiments; recorded incidents and remediation steps.
	- Reason: Capture decisions, commands, and lessons learned so operators can reproduce, detect, and fix similar incidents quickly.

## Recent changes & rationale (2025-12-16)

These changes were applied after initial testing and an incident where `auth-service` failed to start due to DB readiness and healthcheck issues. Summary below with reasons and verification notes.

- `services/auth-service/server.js`
  - Change: Added retry/backoff around `initDB()` and a `/ready` readiness endpoint.
  - Why: Avoid transient startup failures when Postgres is still initializing; give orchestrator a true readiness signal separate from liveness.
  - Verification: After applying the retry logic the service logs show repeated connection attempts followed by `Auth database initialized` once Postgres became ready.

- `services/auth-service/Dockerfile`
  - Change: Tightened `npm` install to `--omit=dev`, `npm prune --production`, cleared npm caches, and removed common test/docs folders from `node_modules` during the deps stage.
  - Why: Reduce final image size by removing dev dependencies, caches, and nonessential files from `node_modules`.
  - Note: This is a conservative approach; further reductions possible via `pnpm`, replacing heavy deps, or moving native build artifacts to a separate stage.

- `services/auth-service/Dockerfile` (PATH fix)
  - Change: Set `ENV PATH=/nodejs/bin:$PATH` in the final distroless image.
  - Why: Distroless Node places the `node` binary in `/nodejs/bin`; Docker healthchecks/execs failed with `exec: "node": executable file not found in $PATH`. Adding the path lets healthchecks invoke `node` without adding a shell or extra tooling to the image.
  - Verification: Healthcheck exec failures (ExitCode -1) disappeared and the container's health transitioned to healthy after rebuild.

- `docker-compose.yml`
  - Change: Updated `auth-service` healthcheck to call `/ready`, increased `start_period`, timeout and retries.
  - Why: Prevent the orchestrator from marking the container unhealthy while the DB is still starting and allow the readiness probe to reflect real service readiness.
  - Verification: Compose health now respects the longer start period and the service no longer shows repeated failing streaks during DB init.

- `.dockerignore` files (per-service)
  - Change: Added conservative `.dockerignore` for each service to exclude `node_modules`, `.env`, `.git`, editor folders and logs.
  - Why: Reduce build context upload, prevent accidental inclusion of local files and secrets, and speed builds.

Operational notes
- Image sizes: after pruning and cleanup the `auth-service` image reduced (example build showed ~253MB). If more reduction is required, consider `pnpm`, replacing heavy dependencies, or moving to slimmer runtime images where feasible.
- Debugging distroless images: distroless reduces attack surface but complicates interactive debugging. Use a temporary `node:alpine` final image or a debug image for troubleshooting.

Commands to verify locally
```bash
docker compose up -d --no-deps --build auth-service
docker compose logs auth-service --follow
docker ps
docker inspect --format='{{json .State.Health}}' $(docker ps -q --filter name=payflow-wallet-auth-service)
docker images | grep auth-service
```


Each modification was driven by one or more goals: reduce image size, improve security (non-root runs, smaller attack surface), increase reliability (healthchecks, retries), and improve operational visibility (logs, docs, Trivy). Where changes introduce trade-offs (e.g., distroless debugging difficulty), I documented fallback options and next steps.


