# Database

Application data lives in one MySQL/MariaDB database, `SB`, owned by `surfbuddy-v3`. Access goes through the author's own `mikk3lro/atomix-databases` PDO wrapper (`Mikk3lRo\atomix\databases\Db`) — raw SQL in `src/datastores/`, no ORM. Connection details (host, credentials) are hardcoded in `surfbuddy-v3/src/SBconfig.php`, not loaded from environment/`.env`.

Datastores, one per domain: `observationsDatastore` (weather station observations/locations), `settingsDatastore` (key/value app settings, e.g. cron bookkeeping timestamps), `shareDatastore` (share tokens/images), `errorsDatastore` (fetch-error logging for alerting).

`surfbuddy-server` also runs its own local MariaDB, but that's server-provisioning bookkeeping (per-site DB users, a throwaway test DB, etc.) — a separate concern from the `SB` application database above.
