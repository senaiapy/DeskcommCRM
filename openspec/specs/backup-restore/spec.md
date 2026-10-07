# backup-restore Specification

## Purpose
Backup and restore of a self-hosted DeskcommCRM: a verified database dump plus the WhatsApp session volume (and, on single-server installs, the attachment storage volume), restore that only runs against an empty database, and the standalone `scripts/backup-db.sh` for operators who run their own cron. Backup and restore are host-side shell operations of `hostgator-setup-kit/`; there is no `/api/v1` route for them. The update flow that calls the backup is covered by `selfhost-install-update`; deletion of stored files by policy is covered by `data-retention` and `file-storage`.

## Requirements

### Requirement: Database dump through the schema connection
`hostgator-setup-kit/backup.sh` SHALL write `db-<YYYYmmdd-HHMMSS>.sql.gz` into `BACKUP_DIR` (default `$PROJECT_DIR/backups`) by running `pg_dump --no-owner --no-privileges` from a `postgres:17-alpine` container against `url_do_schema` (the kit's DDL connection, not the app's lower-privilege role).

#### Scenario: Nightly backup
- **WHEN** `bash hostgator-setup-kit/backup.sh` runs with a reachable database
- **THEN** a file `backups/db-<timestamp>.sql.gz` exists and the script prints the verified size

### Requirement: A dump is renamed into place only after it is proven readable
`backup.sh` SHALL write the dump to a hidden `.db-<ts>.sql.gz.parcial` file, SHALL delete it and abort when the `pg_dump | gzip` pipeline fails or `gzip -t` rejects the file, and SHALL `mv` it to its final `db-<ts>.sql.gz` name only after both checks pass.

#### Scenario: Disk fills during the dump
- **WHEN** `gzip` exits non-zero midway through the dump
- **THEN** the partial file is removed, no `db-<ts>.sql.gz` is created and the script exits with an error telling the operator not to proceed with an update

### Requirement: WhatsApp session snapshot kept only when it contains a session
`backup.sh` SHALL tar the WAHA session volume into `waha-<ts>.tgz` and SHALL keep the archive only when `tar_tem_sessao` finds a session inside it, otherwise removing it and warning that the WhatsApp pairing is not in this backup without failing the database backup.

#### Scenario: Wrong or empty session volume
- **WHEN** the resolved WAHA volume contains no session files
- **THEN** no `waha-<ts>.tgz` is kept and a warning says the restore will require scanning the QR code again

### Requirement: Attachments are part of a single-server backup
When `SINGLE_SERVER=1`, `backup.sh` SHALL archive the Supabase storage volume into `storage-<ts>.tgz` and SHALL abort with an error when that archive cannot be written.

#### Scenario: Single-server storage archive fails
- **WHEN** `SINGLE_SERVER=1` and the storage tar command fails
- **THEN** the script exits with "this backup is NOT complete" instead of reporting success

### Requirement: Rolling retention of 14 backups per kind
`backup.sh` SHALL delete all but the 14 most recent files of each of `db-*.sql.gz`, `waha-*.tgz` and `storage-*.tgz` in `BACKUP_DIR` after a successful run.

#### Scenario: Fifteenth daily backup
- **WHEN** a 15th `db-*.sql.gz` is written
- **THEN** the oldest `db-*.sql.gz` is removed and 14 remain

### Requirement: Restore only into an empty database
`hostgator-setup-kit/restore.sh <db-*.sql.gz>` SHALL exit with usage when the file argument is missing, SHALL count `pg_tables` in schema `public` before asking for confirmation, and SHALL abort without changing anything when that count is greater than zero or cannot be read as a number.

#### Scenario: Restore over a live database
- **WHEN** `restore.sh` targets a database whose `public` schema already has tables
- **THEN** it exits with an error naming the table count and no SQL from the dump is applied

### Requirement: Restore requires typed confirmation
`restore.sh` SHALL require the operator to type `RESTAURAR` before piping the dump into `psql` (without `ON_ERROR_STOP`), and SHALL exit with "Cancelado." on any other input.

#### Scenario: Operator types something else
- **WHEN** the confirmation prompt receives `yes`
- **THEN** the script exits with "Cancelado." and the database is untouched

### Requirement: Paired session and attachment archives are restored with the dump
`restore.sh` SHALL restore the `waha-<ts>.tgz` that shares the dump's timestamp into the WAHA session volume when it exists, and, when `SINGLE_SERVER=1`, SHALL restore `storage-<ts>.tgz` into the storage volume or warn that attachments did not come back when it is missing.

#### Scenario: Single-server restore without the storage archive
- **WHEN** `SINGLE_SERVER=1` and no `storage-<ts>.tgz` sits next to the dump
- **THEN** the database is restored and a warning says the attachments were not restored

### Requirement: Updates take a backup first
`hostgator-setup-kit/update.sh` SHALL run `backup.sh` before `git checkout` of the target tag unless `--skip-backup` is passed, and SHALL abort the update when the backup fails in a non-interactive run (agent-driven or no TTY).

#### Scenario: Agent-driven update with a failing backup
- **WHEN** `update.sh` runs under `DESKCOMM_AGENT_REPORT` and `backup.sh` exits non-zero
- **THEN** the update stops with "O backup preventivo falhou" before touching code or database

### Requirement: Standalone public-schema dump script
`scripts/backup-db.sh [dir]` SHALL dump schema `public` in custom format to `deskcomm-<UTC stamp>.dump` using `SUPABASE_DB_ADMIN_URL`, else `SUPABASE_DB_URL` (environment first, then `.env.local`/`.env`), SHALL exit 1 when neither is set, and SHALL delete dumps older than `RETENTION_DAYS` (default 14).

#### Scenario: No connection string configured
- **WHEN** neither `SUPABASE_DB_ADMIN_URL` nor `SUPABASE_DB_URL` is available
- **THEN** the script prints `FATAL` and exits 1 without creating a file
