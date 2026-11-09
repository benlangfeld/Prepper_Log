# Secure LAN sync and recovery

The inventory remains usable offline without this server. Sync is optional and
requires the devices and server to share a trusted LAN. The server stores bearer
tokens in the browser's `localStorage`; HTTPS protects them in transit but does
not protect a device already compromised by script injection or theft.

## HTTPS setup

The app is HTTPS-hosted, so browsers (including iOS Safari) block requests from
it to an `http://` LAN server as mixed content. Give the sync server a stable LAN
address and use a certificate trusted by every device:

1. Install [mkcert](https://github.com/FiloSottile/mkcert) on a trusted
   administrator computer. Generate a local CA and install its root certificate
   as trusted on every phone and computer that will use sync. On iOS, install the
   root certificate profile and enable full trust in **Settings → General →
   About → Certificate Trust Settings**.
2. Replace `192.168.0.20` below with the server's reserved LAN IP. Keep the
   private key private and do not commit the `sync-backend/certs/` directory:

   ```sh
   mkdir -p sync-backend/certs
   mkcert -cert-file sync-backend/certs/server.crt \
     -key-file sync-backend/certs/server.key 192.168.0.20 localhost 127.0.0.1
   ```

3. Set `CORS_ORIGINS` to the exact HTTPS origin serving the app (origin only,
   with no path), in the ignored root `.env` file:

   ```dotenv
   CORS_ORIGINS=https://your-app-host.example
   ```

   For local development, list each needed origin explicitly, separated by
   commas. An unset or empty allowlist denies browser origins.
4. Start and check the TLS-enabled server:

   ```sh
   docker compose -f docker-compose.sync.yml up -d --build
   curl --cacert "$(mkcert -CAROOT)/rootCA.pem" \
     https://192.168.0.20:8787/health
   ```

5. In PrepTrack settings, enter `https://192.168.0.20:8787`. Use a random
   household pairing code of at least 16 characters, for example generated with
   `openssl rand -hex 16`; enter it only on trusted devices. The server rejects
   shorter codes and limits each source IP to 10 attempts per 15 minutes.

Do not forward port 8787 from the router or expose the service to the public
internet. Pairing rate limits are process-local and reset when the server
restarts; they are not a substitute for LAN isolation. Keep the application
origin and device operating systems up to date.

## Backup and restore drill

Keep both an encrypted app export and a server-database backup. They protect
different things: app exports are end-to-end encrypted on the device; the SQLite
database preserves sync history and pairing records. Store backups offline or
on encrypted storage, and keep the encryption passphrase separately.

1. In **Settings → Export**, create an encrypted `.ptbak` export with a strong
   passphrase. Confirm that the file is downloaded and retain the passphrase.
2. Back up the server database consistently by stopping the service first:

   ```sh
   mkdir -p backups
   docker compose -f docker-compose.sync.yml stop
   docker compose -f docker-compose.sync.yml cp preptrack-sync:/data/sync.db \
     "backups/sync-$(date +%F-%H%M%S).db"
   docker compose -f docker-compose.sync.yml start
   ```

3. Periodically verify the backup by restoring it before starting the service:

   ```sh
   docker compose -f docker-compose.sync.yml stop
   docker compose -f docker-compose.sync.yml cp backups/sync-YYYY-MM-DD-HHMMSS.db \
     preptrack-sync:/data/sync.db
   docker compose -f docker-compose.sync.yml exec -T preptrack-sync \
     sh -c 'rm -f /data/sync.db-wal /data/sync.db-shm'
   docker compose -f docker-compose.sync.yml start
   curl --cacert "$(mkcert -CAROOT)/rootCA.pem" \
     https://192.168.0.20:8787/health
   ```

4. Pair a spare device and confirm that it can pull the expected inventory. Test
   encrypted export restoration separately on a spare app profile before relying
   on either backup.

The procedure above stops sync during the database copy, avoiding an inconsistent
copy of an active SQLite WAL database. Do not copy only `sync.db` while the
container is running.

## Known limitations

- Sync conflicts use whole-entity last-write-wins with client timestamps. If two
  offline devices change the same product quantity independently, one value can
  replace the other; stock movements are not merged as independent events.
- Household readiness settings are device-local. Cats, utility outage targets,
  and evacuation-kit targets are not modeled as shared/synced configuration.
- Device tokens are stored in browser `localStorage`. Pair only trusted devices
  and clear the pairing from a device before giving it away.
- Browser local storage/IndexedDB may be evicted by the operating system. Keep
  periodic encrypted exports; neither PWA installation nor LAN sync is a backup.
