# wthost API Reference

## Overview
- Base path: http://<bind_ip>:<port>/api/v1
- Authentication: Bearer token in Authorization header for all endpoints except /check/{pin}
- Protocols: REST (JSON over HTTP) and WebSocket
- Response envelope:
```json
{
  "success": true,
  "data": "...",
  "status_message": "...",
  "timestamp": "RFC3339"
}
```

## Authentication
- All main endpoints under /api/v1 require Authorization: Bearer <token> except /check/{pin}.
- Token types (entity/token.go):
  - device — format: <token>:<deviceId>, used by a device to authenticate itself. The deviceId part identifies the device.
  - user — format: <token>:<deviceId>, issued after PIN validation to allow a user/client app to act on behalf of that device.
- Validation flow (impl/core/core.go):
  - Middleware extracts the Bearer token; Core.AuthenticateByToken verifies it (via DB when enabled).
  - For device token (type=device): deviceId parsed from the token is authorized/registered (created if missing).
  - For user token (type=user): deviceId is read from the token record; last_seen is updated.

## Linking Flow (PIN → user token)
1) A device requests a PIN by sending a WebSocket message with command "pin_request".
2) Core generates a PIN and stores it (Mongo required for persistence).
3) The client app performs GET /api/v1/check/{pin} (no auth).
4) On success, the service returns a user bearer token in the form <token>:<deviceId>.
5) The client then uses this Bearer token for authenticated REST calls.

## REST API
Base URL: `http://<bind_ip>:<port>/api/v1`
Headers: All authenticated requests must include Authorization: Bearer <token>

1) GET `/device/{filter}`
- Purpose: Fetch device data for the specified device id (filter acts as device id).
- Response: { success, data, status_message, timestamp }
- Example:
```bash
curl -H "Authorization: Bearer <token>" \
     http://127.0.0.1:9700/api/v1/device/123456
```

2) POST `/op`
- Purpose: Initiate a payment-related operation for the authenticated device.
- Request (application/json):
```json
{
  "type": "pay",
  "data": "15000",
  "transaction_id": "<required for refund>",
  "description": "<optional description>"
}
```
- Notes:
  - `type` must be one of: `pay`, `refund`
  - `data` is required and is sent to the device as-is, it contains a payment value in cents
  - For refund, `transaction_id` must be provided
- Success response data: operation id (string)
- Example:
```bash
curl -X POST http://127.0.0.1:9700/api/v1/op \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{
           "type": "pay",
           "data": "{\"amount\":1000,\"currency\":\"USD\"}",
           "description": "Online order #42"
         }'
```

3) GET `/op/{id}`
- Purpose: Read operation details back for the authenticated device.
- Response: Operation object with sensitive fields hidden.
- Example:
```bash
curl -H "Authorization: Bearer <token>" \
     http://127.0.0.1:9700/api/v1/op/<operationId>
```

4) POST `/secrets`
- Purpose: Set bank credentials for the authenticated device.
- Request (application/json):
```json
{
  "bank_clid": "...",
  "bank_secret": "...",
  "bank_token": "..."
}
```
- Example:
```bash
curl -X POST http://127.0.0.1:9700/api/v1/secrets \
     -H "Authorization: Bearer <token>" \
     -H "Content-Type: application/json" \
     -d '{"bank_clid":"demo","bank_secret":"demo","bank_token":"demo"}'
```

5) GET `/check/{pin}`
- Auth: none. Verifies a PIN issued for device linking and returns a user token on success.
- Example:
```bash
curl http://127.0.0.1:9700/api/v1/check/123456
# Response data: "<token>:<deviceId>"
```

## WebSocket API
- URL: `ws://<bind_ip>:<port>/api/v1/ws`
- Auth: same Bearer token as REST; middleware authenticates and attaches device to context.
- On connect: the device is registered in the pool under its device ID and can receive targeted messages.

### Messages: Service → Device (initiated by POST /op)
```json
{
  "operation_id": "...",
  "command": "pay",
  "transaction_id": "...", 
  "description": "...",
  "bank_clid": "...",
  "bank_secret": "...",
  "bank_token": "...",
  "data": "<opaque JSON string>"
}
```

### Messages: Device → Service
- PIN request
```json
{ "device": "<deviceId>", "command": "pin_request" }
```
  → Response (service → device):
```json
{ "operation_id":"", "command":"pin_request", "data":"<pin>" }
```

- Operation result
```json
{
  "device": "<deviceId>",
  "operation_id": "<id>",
  "command": "result",
  "data": "<JSON string with result>",
  "description": "<optional message>"
}
```
  → Service updates operation status via Core.completeOperation

## Operation Lifecycle (entity/operation.go)
- Created: status=pending, timestamps set
- Completed/Failed: status updated, closed timestamp set; result is parsed to extract known fields:
  - pay (object) and rid (string) extracted from result JSON when present
  - success=false toggles status to failed
  - message extracted into Operation.Message

## Notes
- All timestamps are RFC3339 strings in response envelope.
- For production, enable MongoDB for token, PIN, device, and operation persistence.
