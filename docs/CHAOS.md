# Chaos Engineering Notes - PayFlow

Planned experiments

1) Stop a container suddenly
- Command: `docker-compose stop auth-service` or `docker stop <container_id>`
- Expected: Dependent services fail healthchecks and may retry; `api-gateway` should return 5xx for auth calls.
- Recovery: `docker-compose start auth-service` or `docker-compose up -d auth-service`

2) Remove the network
- Command: `docker network rm payflow-wallet_default` (compose default network)
- Expected: Containers lose connectivity to DB/redis; healthchecks fail.
- Recovery: `docker-compose down && docker-compose up -d`

3) Simulate low memory
- Command: run compose with memory limits or update container: `docker update --memory 100m <container>`
- Expected: OOM on memory-heavy processes; container may be killed and restarted.
- Recovery: Increase memory limit or optimize service to use less memory.

Documentation should include exact commands used, timestamps, and observed logs.
