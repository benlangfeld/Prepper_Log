# PrepTrack Sync Backend

The optional sync service stores household changes in SQLite. Keep it on a trusted
LAN and use HTTPS with a certificate trusted by every device; do not expose its
HTTP endpoint to the internet.

## Secure LAN deployment

Follow [the deployment and backup/restore drill](../docs/SECURE_LAN_SYNC.md).
The Compose configuration requires a TLS certificate and key in
`sync-backend/certs/server.crt` and `sync-backend/certs/server.key`, and uses
`CORS_ORIGINS` as an exact, comma-separated browser-origin allowlist. An empty
allowlist denies browser origins. Do not commit certificate private keys.

## Environment variables

- `PORT` (default `8787`)
- `DB_PATH` (default `/data/sync.db`)
- `TLS_CERT_PATH` and `TLS_KEY_PATH` (must be set together)
- `CORS_ORIGINS` (comma-separated exact origin allowlist; empty denies browser origins)

Pairing codes must be 16–128 characters; each source IP is limited to 10 attempts
per 15 minutes. Pairing rate limits are process-local and reset on server restart.

## API

- `POST /v1/pair` `{ syncCode, deviceName }`
- `POST /v1/sync/push` (device bearer token and `x-household-id` required)
- `GET /v1/sync/pull?cursor=<n>` (same authentication headers)

Push batches are validated as a whole; invalid batches receive HTTP 400 and are
not applied.

## Tests

```bash
npm run test
```
