# Mac Mini Deployment Runbook

## 1. Target

V1 runs on the household Mac mini at:

```text
https://financials.local:8443
```

The Mac advertises local hostname `financials` through mDNS. Reserve its private IP in the home router so firewall rules and diagnostics remain stable. The canonical application URL is the hostname, not the IP.

Port 8443 avoids conflict with Laravel Herd's Nginx on ports 80/443. There is no plain-HTTP authenticated endpoint. A failed HTTP connection is preferable to an insecure fallback; members bookmark the HTTPS URL.

## 2. Runtime Components

- Native FrankenPHP/Caddy in classic mode; Laravel Octane is not installed.
- Laravel database queue worker.
- Laravel scheduler worker.
- SQLite operational database.
- Private Laravel storage.
- A separate encrypted backup destination.

FrankenPHP is the selected household server because it provides a production-capable PHP server, native macOS service support, and Caddy's internal CA. Herd remains the development environment.

## 3. Host Preparation

Perform once from an administrator account:

```bash
sudo scutil --set ComputerName "Financials"
sudo scutil --set LocalHostName "financials"
sudo scutil --set HostName "financials.local"
brew install dunglas/frankenphp/frankenphp
```

Verify `financials.local` resolves from another LAN device before continuing. Keep macOS automatic security updates and firewall enabled. Do not configure router port forwarding, UPnP exposure, Expose/ngrok, or public DNS.

## 4. Application Environment

Production `.env` uses:

```dotenv
APP_NAME=Financials
APP_ENV=production
APP_DEBUG=false
APP_URL=https://financials.local:8443

DB_CONNECTION=sqlite
QUEUE_CONNECTION=database
SESSION_DRIVER=database
CACHE_STORE=database

SESSION_SECURE_COOKIE=true
SESSION_HTTP_ONLY=true
SESSION_SAME_SITE=lax
```

Generate `APP_KEY` once. File permissions allow only the owning Mac account and runtime process to read `.env`, SQLite, logs, private files, Caddy keys, and backups. Nothing under `storage`, `database`, or the repository root is served directly.

Before first start:

```bash
frankenphp php-cli artisan key:generate
frankenphp php-cli artisan migrate --force
frankenphp php-cli artisan optimize
```

## 5. FrankenPHP Caddyfile

Homebrew runs the configuration at `$(brew --prefix)/etc/Caddyfile`. Use the absolute application path:

```caddyfile
{
    frankenphp
    admin localhost:2019
}

https://financials.local:8443 {
    tls internal
    root * /Users/cardhouse/Herd/financials/public
    encode zstd br gzip

    header {
        X-Content-Type-Options nosniff
        X-Frame-Options SAMEORIGIN
        Referrer-Policy strict-origin-when-cross-origin
        -Server
    }

    php_server {
        try_files {path} index.php
    }
}
```

Validate and start:

```bash
frankenphp validate --config "$(brew --prefix)/etc/Caddyfile"
brew services start dunglas/frankenphp/frankenphp
curl --fail --silent --show-error https://financials.local:8443/up
```

Use `brew services restart dunglas/frankenphp/frankenphp` after configuration or application deployment. Do not enable Octane worker mode in V1; classic mode avoids long-lived application-state concerns.

## 6. Device Certificate Trust

Caddy's `tls internal` creates a local CA and server certificates. Trust the CA on the Mac using FrankenPHP/Caddy's trust command or macOS Keychain. Locate the public root certificate in Caddy's data directory and transfer it directly with AirDrop or another physically verified method.

For each iPhone/iPad:

1. Transfer and install the CA certificate profile.
2. Open Settings → General → About → Certificate Trust Settings.
3. Enable full trust for the household CA.
4. Open the canonical URL and verify the certificate is trusted before signing in.

For each Mac, import the root into the System keychain and mark it trusted for SSL. Do not send the CA root through an unverified link. The public root may be shared; the CA private key never leaves the Mac mini except in the encrypted administrative backup.

When a device is retired, remove its CA trust. If the CA private key is suspected compromised, replace the CA, reissue the site certificate, and reinstall trust on approved devices.

## 7. Queue and Scheduler Services

Create two per-user `launchd` agents or equivalent supervised services using absolute paths.

Queue command:

```bash
frankenphp php-cli /Users/cardhouse/Herd/financials/artisan queue:work database --sleep=1 --tries=3 --timeout=120 --max-time=3600
```

Scheduler command:

```bash
frankenphp php-cli /Users/cardhouse/Herd/financials/artisan schedule:work
```

Required service behavior:

- starts at login and restarts after failure;
- runs from the project directory;
- writes stdout/stderr to private rotating logs;
- receives `SIGTERM` during deployment and restarts afterward;
- queue workers recycle at least hourly through `--max-time`;
- never run more than one scheduler worker for this installation.

After every deployment, run `queue:restart` and verify both service labels are healthy.

## 8. Scheduled Tasks

Laravel scheduler owns the schedule:

- bill occurrence generation daily at 00:15;
- income occurrence generation daily at 00:20;
- next budget-period preparation on the 25th at 00:30, idempotently;
- database backup daily at 02:00;
- backup integrity verification immediately after backup;
- weekly application health summary;
- prune framework sessions/cache/jobs according to Laravel defaults without pruning financial history.

Commands use `withoutOverlapping()` and stable natural/explicit idempotency keys. A missed schedule catches up safely on next run where business behavior requires it.

## 9. Backups

Backups are stored on a separate encrypted destination, not merely another folder on the same disk. The backup command:

1. Uses SQLite's online `.backup` mechanism against the live database.
2. Copies required private import/export attachments still under retention.
3. Captures a redacted configuration manifest, not `.env` secrets.
4. Calculates SHA-256 checksums.
5. Runs `PRAGMA integrity_check` on the copied database.
6. Records success/failure in application health state.

Retention:

- 7 daily backups;
- 8 weekly backups;
- 12 monthly backups.

The Caddy CA private material and an offline copy of `APP_KEY` belong in a separate encrypted administrative backup with recovery instructions. They are not included in ordinary financial exports.

Once per month, restore the latest backup to a temporary path, run migrations in pretend/status mode as appropriate, boot the app against the restored copy, and verify record counts plus a known dashboard month. Delete the temporary restore securely afterward.

## 10. Restore Procedure

1. Stop web, queue, and scheduler services.
2. Copy the damaged database aside; never overwrite the only copy.
3. Verify the selected backup checksum and SQLite integrity.
4. Restore database and required private files with correct ownership/permissions.
5. Run `frankenphp php-cli artisan migrate --force` only after checking migration status and application version compatibility.
6. Run `frankenphp php-cli artisan optimize`.
7. Start web, scheduler, and queue.
8. Verify `/up`, authentication, current dashboard, latest balances, and pending jobs.
9. Record the restore event and retained damaged-copy location.

## 11. Application Update Procedure

1. Confirm clean source state and review release notes/migrations.
2. Create and verify an immediate backup.
3. Enable maintenance mode.
4. Stop queue/scheduler services after active jobs finish.
5. Install locked Composer/NPM dependencies and build assets.
6. Run migrations with `--force`.
7. Run the full test/format/static-analysis gate when updating from source on the host.
8. Run `artisan optimize` and `queue:restart`.
9. Restart all services.
10. Disable maintenance mode and verify `/up` plus the primary workflow.

Never deploy uncommitted source, use `composer update` during deployment, or discard the pre-update backup until verification succeeds.

## 12. Health and Troubleshooting

`/up` fails when the application cannot boot. The enhanced health check reports unhealthy when:

- SQLite is missing, read-only, corrupt, or foreign keys are disabled;
- private storage is not writable;
- queue heartbeat is older than five minutes;
- scheduler heartbeat is older than five minutes;
- latest verified backup is older than 26 hours;
- certificate expiration/renewal state requires action.

The UI shows an administrative health banner without exposing paths, SQL, stack traces, or secrets. Detailed diagnostics remain available only on the Mac mini through logs and `artisan financials:health`.

## 13. Security Checklist

- No public router forwarding or tunnel.
- HTTPS certificate trusted and hostname verified on each device.
- `APP_DEBUG=false`.
- Public registration/email reset disabled.
- Strong unique passwords; optional TOTP enabled per member.
- macOS firewall and security updates enabled.
- `.env`, DB, backups, logs, CA keys, and private files excluded from Git and web root.
- Restore test completed within the last month.
- Both runtime workers healthy after reboot.

## 14. Source References

- [Laravel deployment requirements](https://laravel.com/docs/13.x/deployment)
- [FrankenPHP Laravel configuration](https://frankenphp.dev/docs/laravel/)
- [FrankenPHP macOS production service](https://frankenphp.dev/docs/production/)
- [Caddy internal TLS](https://caddyserver.com/docs/caddyfile/directives/tls)
