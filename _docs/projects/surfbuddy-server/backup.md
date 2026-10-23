# Backup

Encrypted restic backups of selected site data, pushed to a Hetzner Storage Box (`u694361`, SSH port 23, SFTP). Configured per site via the `backup` key in `$sites` in `scripts/install.php`; only the Scan Organizer sites use it so far.

## What is backed up

- Per site: every database in `db.dbs` (`mariadb-dump --single-transaction --routines --events`) and the listed `paths` (relative to the site home; `secret/` for Scan Organizer).
- Databases are dumped first, then files, so every file a dump references is in the snapshot.
- `web/`, `tmp/` and `log/` are not backed up.

## Setup

- `scripts/backup.php <config>` runs one site's backup: dump, `restic backup`, `restic forget --prune`, and on Sundays `restic check --read-data-subset=5%`. Dumps live in `/var/backups/surfbuddy/<domain>/` (0700) and are deleted afterwards.
- `scripts/backup-check.php` runs daily at 09:00 and fails when a site's last success stamp is older than its `max_age_hours`.
- `install.php` (`configureBackups()`) installs restic, writes `/etc/cron.d/surfbuddy-backup`, and creates per site:
  - `/etc/surfbuddy/backup/<user>.json` and `<user>.password` (root-only, 0600).
  - One restic repository per site at `backups/<domain>` on the Storage Box, with its own password.
- One dedicated SSH key `/root/.ssh/backup_ed25519` is shared by all sites. On first creation the installer prints the public key; add it to the Storage Box (`ssh-copy-id -s -p 23 -i <key>.pub u694361@u694361.your-storagebox.de`; SSH support must be enabled on the Storage Box).
- On first creation the installer prints each repository password once. They exist only on the server otherwise, so keep a copy elsewhere - without the password the backups are unreadable.

## Schedule and retention

| Site | Schedule | Retention | Alert after |
|---|---|---|---|
| `docs.surfbuddy.dk` | nightly 03:15 | 14 daily, 8 weekly, 6 monthly | 36 h |
| `docsstaging.surfbuddy.dk` | Sundays 04:00 | 4 weekly | 192 h |

## Alerts

- `backup.php` writes only errors to stderr; cron mails them to root, which `/root/.forward` sends on to the owner.
- Run output (restic summary with sizes) is appended to `/var/log/surfbuddy-backup.log`; success stamps are `/var/lib/surfbuddy-backup/<domain>.ok`.
- `scripts/backup-report.php` looks at the backups from the Storage Box side (restic snapshot lists and `df` over SSH), not at local stamps:
  - `check` runs daily at 08:30 through monit (`/etc/monit/conf.d/backups`) and alerts when a repository has no snapshot newer than its `max_age_hours`, cannot be read, or the Storage Box is more than 20 % full. The 20 % limit is not a capacity limit; passing it means something unexpected is filling the box.
  - `report --mail` runs Mondays 08:00 from cron and mails the snapshot lists and Storage Box usage to root.
- Monit (`/etc/monit/monitrc`, alerts to the owner) has no other checks and no web interface beyond localhost:2812.
- No alert is possible if the whole server is down.

## Restore

Set the repository environment (values are in `/etc/surfbuddy/backup/<user>.json`, or reconstruct them on another machine from the Storage Box path and the saved password):

```bash
export RESTIC_REPOSITORY=sftp:u694361@u694361.your-storagebox.de:backups/docs.surfbuddy.dk
export RESTIC_PASSWORD_FILE=/etc/surfbuddy/backup/scanorg.password
restic_sb() { restic -o "sftp.args=-p 23 -i /root/.ssh/backup_ed25519" "$@"; }
restic_sb snapshots
restic_sb restore latest --target /root/restore
```

The restore contains `var/www/<domain>/secret/` and `var/backups/surfbuddy/<domain>/<database>.sql`.

- Files: copy `secret/` back into the site home, then `chown -R <site user>:<site user> secret` and `chmod 0700 secret`. Files inside keep the owner/permissions restic stored, but verify `users.json`, `anthropic.key` and `ai.json` are 0600 and owned by the site user.
- Database: `mariadb <database> < <database>.sql`. The app applies schema migrations itself on the first request.
- Verify by restoring into a throwaway database and directory (never into staging), then compare the row count of `pages` with the number of files in `pages/`.
