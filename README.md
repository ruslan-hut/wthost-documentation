# wthost — Payment Bridge Backend Service

## Overview
- Purpose: wthost is a backend bridge between client applications (e.g., web back-office) and mobile devices that execute payment operations. It exposes a simple REST API for initiating and tracking payment/refund operations and a WebSocket channel to communicate with mobile devices in real time.
- Core idea: Clients request a pay/refund via REST. The service forwards the command to a specific device over WebSocket. The device processes the request and reports results back, which are persisted and retrievable via REST.

## Key Features
- REST API to initiate and read operations
- WebSocket for device communication (commands + results)
- Token-based authentication (device and user tokens)
- Optional MongoDB persistence (devices, tokens, PINs, operations)
- Structured logging with request correlation (X-Request-ID)
- Pluggable client connection pool with device presence tracking

## Architecture (high level)
- HTTP API: chi router under /api/v1
  - Auth middleware parses Bearer token and attaches device to request context
  - JSON responses wrapped in a unified envelope { success, data, status_message, timestamp }
- Core service (impl/core): business logic
  - Token authentication (device/user), device authorization, PIN-based linking
  - Operation lifecycle: create → send to device → complete/fail → read
- WebSocket handler: registers device connection and exchanges messages
- Client pool: tracks active WebSocket clients and allows targeted send by device ID
- Database (optional): MongoDB for storing devices, tokens, PINs, operations

## Quick Start
Prerequisites
- Go 1.22+
- MongoDB (optional, enable for persistence)

Build and Run
- Local default config uses in-memory behavior (Mongo disabled). You can run without Mongo to explore flows.

1) Run with defaults
```bash
go run ./cmd/wthost -conf config.yml -log ./logs
```

2) Build a binary
```bash
go build -o bin/wthost ./cmd/wthost
./bin/wthost -conf config.yml -log ./logs
```

## Configuration
- Config is a YAML file loaded with -conf flag. Two examples are provided:
  - config.yml — local defaults (Mongo disabled)
  - wt-config.yml — templated for environments via env vars

### Config schema (internal/config/config.go):
```yaml
  env: local
  listen:
    bind_ip: 127.0.0.1
    port: 9700
  mongo:
    enabled: false
    host: 127.0.0.1
    port: 27017
    user: admin
    password: pass
    database: wthost
```

## Logging
- -log flag specifies a directory for logs. Logger level depends on env (see internal/lib/logger).
- Requests receive an X-Request-ID header; logs include request_id.

## Authentication
- All main endpoints under /api/v1 require Authorization: Bearer <token> except /check/{pin}.
- Token types (entity/token.go):
  - device — format: <token>:<deviceId>, used by a device to authenticate itself. The deviceId part is used to identify the connecting device.
  - user — format: <token>:<deviceId>, issued after PIN validation to allow a user/client app to act on behalf of that device.
- Validation flow (impl/core/core.go):
  - Middleware extracts the Bearer token, Core.AuthenticateByToken verifies it via DB when enabled.
  - For device token (type=device): deviceId is parsed from token and authorized/registered (created if missing).
  - For user token (type=user): deviceId is read from token record; last_seen is updated.

## Linking Flow (PIN → user token)
- A device requests a PIN by sending a WebSocket message with command "pin_request".
- Core generates a PIN and stores it (Mongo required for persistence)
- The client app performs GET /api/v1/check/{pin} (no auth)
- On success, the service returns a user bearer token in the form <token>:<deviceId>
- The client then uses this Bearer token for authenticated REST calls.

## API documentation
For detailed HTTP and WebSocket API, including endpoints, request/response formats, and examples, see [API.md](./API.md).

## Persistence and No-DB mode
- If Mongo is disabled (mongo.enabled=false):
  - AuthenticateByToken accepts any "token:deviceId" and creates a dummy device object in memory
  - PIN flow returns a random token without verification storage
  - Operation storage and reads require Mongo; relevant endpoints will return errors where DB is needed
- For production, enable Mongo and supply credentials/DB in config.

## Ports and Networking
- Service binds to `listen.bind_ip:listen.port` (default 127.0.0.1:9700). To expose externally, set `bind_ip` to 0.0.0.0.
- Reverse proxies should pass X-Forwarded-For; middleware honors it for remote address logging.

## CI/CD
- GitHub Actions workflows are present in `.github/workflows/`:
  - deploy.yml, deploy-dev.yml, deploy-prod-tag.yml
- They can be adapted to your environment for building and deploying the service.

## Development Notes
- Go version: see go.mod (go 1.22)
- Main entrypoint: cmd/wthost/main.go
- HTTP setup: internal/http-server/api/api.go
- Middleware: internal/http-server/middleware/authenticate
- Handlers: internal/http-server/handlers/*
- Core logic: impl/core
- WebSocket client handling: internal/http-server/handlers/websocket
- Mongo adapter: internal/database/mongo-client.go

## Troubleshooting
- 401 Unauthorized: ensure Authorization header is set and token format is correct ("<token>:<deviceId>").
- 400 on POST /op: validate request body; refund requires transaction_id.
- Device not receiving operations: ensure device is connected over WebSocket and its deviceId matches the token suffix; check active client presence via logs.
- DB-related errors: verify Mongo connection settings and credentials.

## License
- Proprietary or TBD