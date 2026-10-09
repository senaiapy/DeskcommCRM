## ADDED Requirements

### Requirement: The worker health port is published on loopback only
`docker-compose.yml` and `docker-compose.local.yml` SHALL publish the worker's `HEALTH_PORT` as `127.0.0.1:8787:8787`, so `/healthz` and `/metrics` are reachable from the host but not from other machines.

#### Scenario: Another machine on the network
- **WHEN** a host on the same network connects to port 8787 of the machine running the local stack
- **THEN** the connection is refused because the port is bound to 127.0.0.1
