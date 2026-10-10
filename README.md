# PostgreSQL Backup, PITR, and Streaming Replication Lab

## Objective

Create and verify a logical backup, configure WAL archiving, perform
point-in-time recovery (PITR), and configure a streaming standby replica.

## Environment

- PostgreSQL: 17.11
- Operating system: Windows
- Primary: 127.0.0.1:5433
- Standby: 127.0.0.1:5435
- Tools: psql, pg_dump, pg_restore, pg_basebackup, pg_verifybackup, pg_ctl

## 1. Logical Backup and Restore

Created a custom-format logical backup:

```powershell
pg_dump -h 127.0.0.1 -p 5433 -U postgres -d bootcamp -Fc -f "$HOME\backups\bootcamp.dump"
```

Restored the archive into the separate `bootcamp_check` database using
`pg_restore`. Verified the archive and restored tables, including
`audit_log`, `categories`, and `students`.

The students table initially contained three rows.

## 2. WAL Archiving and Physical Backup

Configured WAL archiving on the primary PostgreSQL instance.

- `wal_level = replica`
- `archive_mode = on`
- WAL archive directory: `C:\Program Files\PostgreSQL\17\data\wal_archive`

Created a physical base backup using `pg_basebackup` with WAL streaming.
Validated the backup with `pg_verifybackup`.

Result: `backup successfully verified`.

## 3. Point-in-Time Recovery (PITR)

Created a separate recovery copy of the base backup and configured:

- A separate recovery port: `5434`
- `restore_command` to retrieve archived WAL files
- `recovery_target_time` set to the saved target timestamp
- `recovery_target_action = 'promote'`

Simulated accidental deletion by deleting rows from the test database
`bootcamp_check`, not the original `bootcamp` database.

Recovered the database to the selected point in time.

Verification:

```sql
SELECT count(*) FROM public.students;
```

Result: 3 rows recovered.

The recovery log confirmed that archive recovery completed.

## 4. Streaming Replication

Built a standby using `pg_basebackup` with `-X stream -R`.

- Primary: `127.0.0.1:5433`
- Standby: `127.0.0.1:5435`
- Replication user: `replicator`
- Replication mode: asynchronous

Verified the standby was in recovery:

```sql
SELECT pg_is_in_recovery();
```

Result: `true`.

Verified the standby WAL receiver:

```sql
SELECT status, sender_host, sender_port, latest_end_lsn
FROM pg_stat_wal_receiver;
```

Result: `streaming`, sender `127.0.0.1`, port `5433`.

Verified the primary connection:

```sql
SELECT application_name, client_addr, state, sync_state
FROM pg_stat_replication;
```

Result: `streaming`, `async`.

## 5. Replication Data Test

Created a test table and inserted a test row in the `postgres` database
on the primary. Queried the standby and confirmed the row appeared:

```text
id | message
1  | Streaming replication works
```

Final replay lag check:

```sql
SELECT application_name, state, sync_state,
       pg_size_pretty(pg_wal_lsn_diff(sent_lsn, replay_lsn)::bigint)
       AS replay_lag
FROM pg_stat_replication;
```

Observed result: `0 bytes`.

## 6. Lessons Learned

- Logical backups help restore selected databases or objects.
- Physical base backups provide a foundation for cluster recovery.
- WAL archiving supports point-in-time recovery.
- Streaming replication maintains a standby using WAL records.
- Asynchronous replication may lose the latest transactions during an abrupt
  primary failure if those records have not reached or been replayed by the
  standby.
- Backups must be stored independently and restores tested regularly.
- Passwords and other secrets must never be committed to Git.

## Conclusion

The logical restore, physical backup verification, PITR exercise, and
streaming replication test completed successfully in the local lab.
