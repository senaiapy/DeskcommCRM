## ADDED Requirements

### Requirement: Backups are readable only by their owner
`hostgator-setup-kit/backup.sh` and `scripts/backup-db.sh` SHALL run with `umask 077`, and `backup.sh` SHALL `chmod 700` the `BACKUP_DIR` (warning and continuing when the filesystem refuses) and write the WAHA and storage archives through host-side redirection, so dumps and archives are created owner-only.

#### Scenario: Another user on the VPS
- **WHEN** a second Unix user lists `backups/` after a backup
- **THEN** the directory and the new `db-*`, `waha-*` and `storage-*` files are not readable by that user

### Requirement: The storage archive is renamed into place only after tar succeeds
With `SINGLE_SERVER=1`, `backup.sh` SHALL write the storage archive to `.storage-<ts>.tgz.parcial`, delete it and abort when tar fails, and `mv` it to `storage-<ts>.tgz` only on success.

#### Scenario: Tar fails
- **WHEN** the storage tar command exits non-zero
- **THEN** no `storage-<ts>.tgz` exists and the script says the backup is not complete
