# surfbuddy-server

PHP 8.4 provisioning/config-management code for the single Debian VPS (`s001.surfbuddy.dk`) that hosts everything for Surfbuddy. It is infrastructure-as-code, not an application — running `scripts/install.sh` (→ `scripts/install.php`) idempotently configures the box from scratch: packages, Apache vhosts + PHP-FPM pools, MariaDB, firewall (iptables), WireGuard VPN, TLS certs (certbot), DKIM/Postfix mail, phpMyAdmin, Netdata monitoring, Syncthing.

No README/INSTALL exists in the repo itself; `scripts/install.php` is the source of truth.

## This server is not Surfbuddy-only

The box also hosts unrelated personal sites (`taust.dk`, a "todo" app with live/staging envs). Don't assume every vhost or DB user in `install.php` is part of Surfbuddy.

## Layout

- `classes/` — small support classes: `config.php` (constants, local MariaDB bootstrap, file logger), `firewall.php` (writes iptables rules), `surfbuddyDbObj.php`, `surfbuddyPsrLogger.php`.
- `modified_files/{etc,root,var}/` — literal overlay files copied onto the server's real `/etc`, `/root`, `/var` by `install.php`'s `overwriteFiles()`. This is where the actual server config lives (vhosts, FPM pools, cron, Postfix/DKIM, sysctl, SSH keys).
- `scripts/install.php` — the ~650-line provisioner (see below).
- `scripts/cron.php` — see [cron](../../architecture/cron.md).
- `scripts/getgrib.py`, `readgrib.py`, `parsegrib.php` — early/experimental GRIB2 prototyping (pygrib/eccodes, ECMWF data). Not load-bearing themselves, but they anticipated the approach `surfbuddy-v3`'s `ECMWF.php`/`scripts/decode_grib.py` actually ended up using — see [ECMWF map fetch](../../architecture/ecmwf-map-fetch.md).
- `misc/` — `.gpg` public signing keys for apt repos (Google Cloud, Syncthing) — not secret.

## How it relates to the other repos

- `modified_files/etc/apache2/sites-available/` defines vhosts for `backend.surfbuddy.dk`/`api.surfbuddy.dk` (docroot `/var/www/backend.surfbuddy.dk/web/public`, PHP-FPM) — this serves **surfbuddy-v3**.
- Same directory defines vhosts for `frontend.surfbuddy.dk`/`surfbuddy.dk`/`app.surfbuddy.dk` (docroot `/var/www/frontend.surfbuddy.dk/web`) — this serves the built **surfbuddy-vue** static output.
- `modified_files/etc/cron.d/cron` runs `scripts/cron.php` every minute, which just hits `https://backend.surfbuddy.dk/cron` — i.e. this repo triggers surfbuddy-v3's scheduled forecast-fetch, it doesn't run any forecast logic itself. See [cron](../../architecture/cron.md).

## Secrets committed in this repo

Not read in detail, not to be exposed in docs or chat — locations only, so future work doesn't accidentally leak them:
- `modified_files/etc/apache2/.htpasswd`
- `modified_files/etc/todo_live.env`, `modified_files/etc/todo_staging.env`
- `modified_files/root/.ssh/authorized_keys`
- `modified_files/var/www/backend.surfbuddy.dk/.ssh/id_ed25519` (private key) and `.pub`
- Plaintext DB passwords hardcoded in `scripts/install.php` and `classes/config.php`
