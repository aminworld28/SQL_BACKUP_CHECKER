# SQL Backup Checker (PowerShell)

A lightweight PowerShell script that checks the latest SQL Server backup file in each store's network share and emails an alert when something looks wrong. Built for multi-site environments (retail, restaurants, branch offices) where logging into dozens of servers every morning doesn't scale.

> **Version 1 (share-folder check).** All IPs, store names, and credentials in this repo are dummy values. IPs use the `192.0.2.0/24` documentation range (RFC 5737).

## What it does

For every store in the store map, the script:

1. Checks that the network share is reachable
2. Finds the newest file in the folder (by `LastWriteTime`)
3. Applies two rules:
   - Was it written in the last 24 hours?
   - Is it larger than 1 MB?
4. Assigns a status per store
5. Writes a dated text report to a log folder
6. Sends an HTML email **only if** at least one store is not OK
7. Deletes log files older than 30 days

### Status meanings

| Status    | Meaning                                              |
|-----------|------------------------------------------------------|
| `OK`      | Recent file found and larger than 1 MB               |
| `WARNING` | Recent file found, but suspiciously small (< 1 MB)   |
| `FAILED`  | No recent file, or the share path is unreachable     |

## Requirements

- Windows PowerShell 5.1 or PowerShell 7+
- Read access to each store's backup share from the machine running the script
- An SMTP server (and credentials) for email alerts

## Setup

1. Clone the repo:
   ```powershell
   git clone https://github.com/aminworld28/SQL_BACKUP_CHECKER.git
   cd <your-repo>
   ```
2. Open `Check-SqlBackups.ps1` and edit the configuration blocks:
   - **`$StoreMap`**: one line per store, `"Store Name" = "\\ip-or-host\share"`
   - **Email settings**: SMTP server, port, from/to addresses, username
   - **`$AppPassword`**: your SMTP app password (see the security note below)
   - **`$LogPath`**: where reports are saved (default `C:\BackupLogs`)
3. Run it:
   ```powershell
   .\Check-SqlBackups.ps1
   ```

## Example log output

```
SQL BACKUP REPORT - 10/06/2026 08:00:00
============================================================
STORE NAME                | STATUS     | DETAILS
ST001-Demo Store A        | OK         | Size: 412.37 MB
ST002-Demo Store B        | WARNING    | File too small (640 KB)
ST003-Demo Store C        | FAILED     | Unreachable Path
ST004-Demo Store D        | FAILED     | File not found/outdated
```

## Security note

**Never commit real credentials, IP addresses, or internal store names.** The password is stored in plain text in this version for simplicity. For real use, store credentials securely, for example with `Export-Clixml` (tied to the user and machine) or a secrets manager, and keep your configured copy out of version control.

## Known limitations (v1)

This version answers *"did a recent file land in the folder?"* It does **not** answer *"can I restore this database?"* Specifically:

- It doesn't verify backup integrity (no `RESTORE VERIFYONLY`, no test restores)
- It uses a fixed 1 MB threshold rather than comparing against the previous backup size
- It checks the share only, not the SQL Server backup history
- It takes the newest file in the folder, regardless of file type or database
- It runs manually and sends no email when everything is OK, so silence is ambiguous
- It checks stores one by one, so a slow or unreachable path can stall the run
- `Send-MailMessage` is deprecated, and the `Read-Host` in the error handler will hang a scheduled task

## Roadmap

- [x] **v1**: share-folder check with log and email alert
- [ ] **v2**: dbatools-based checks against SQL Server backup history, `CHECKSUM` + `RESTORE VERIFYONLY`, size-trend detection, daily "all OK" summary, unattended scheduling
- [ ] **v3**: rotating test restores with `DBCC CHECKDB`, central reporting, real-world rollout notes

## Contributing

Issues and pull requests are welcome. If you manage backups across many sites, I'd like to hear how you monitor them.


