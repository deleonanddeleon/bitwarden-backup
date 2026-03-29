# Bitwarden Vault Backup — Tier 2 Raspberry Pi Architecture
## Implementation Specification

**Version:** 1.0
**Date:** 2026-03-29
**Status:** Authoritative — implement directly from this document

---

## Table of Contents

1. [Overview](#1-overview)
2. [System Requirements](#2-system-requirements)
3. [Security Model](#3-security-model)
4. [Component Specifications](#4-component-specifications)
   - 4.1 [Bitwarden CLI](#41-bitwarden-cli)
   - 4.2 [Encryption](#42-encryption)
   - 4.3 [Backup Script Logic](#43-backup-script-logic)
   - 4.4 [GFS Rotation](#44-gfs-rotation)
   - 4.5 [rclone pCloud Crypt Remote](#45-rclone-pcloud-crypt-remote)
   - 4.6 [ntfy.sh Notifications](#46-ntfysh-notifications)
   - 4.7 [systemd Timer and Service](#47-systemd-timer-and-service)
   - 4.8 [Logging](#48-logging)
   - 4.9 [Integrity Verification](#49-integrity-verification)
5. [File and Directory Layout](#5-file-and-directory-layout)
6. [Backup Script](#6-backup-script)
7. [Operational Procedures](#7-operational-procedures)
8. [Known Limitations](#8-known-limitations)

---

## 1. Overview

### 1.1 Purpose

This document specifies a fully automated, encrypted, off-site backup system for a Bitwarden password vault. The system runs on a Raspberry Pi as a headless service, performs nightly exports of the vault, encrypts the export before it touches disk, applies a Grandfather-Father-Son (GFS) rotation scheme locally, and replicates the encrypted archive set to pCloud via an rclone crypt remote.

### 1.2 Scope

This specification covers:

- Authentication to the Bitwarden CLI using API key credentials
- GPG symmetric encryption of vault exports
- Local GFS rotation (daily/weekly/monthly)
- Encrypted replication to pCloud via rclone
- Failure and success notifications via ntfy.sh
- systemd timer-based scheduling
- Structured logging and log rotation
- Post-backup integrity verification

This specification does **not** cover:

- Bitwarden server setup or self-hosting
- pCloud account creation or billing
- Raspberry Pi OS installation
- Network router or firewall configuration upstream of the Pi

### 1.3 Constraints

| Constraint | Value | Rationale |
|---|---|---|
| Platform | Raspberry Pi OS Bookworm (Debian 12) | Target hardware |
| Full-disk encryption | None (no LUKS) | Accepted trade-off — see §1.4 |
| Cloud storage | pCloud via rclone crypt remote | Specified requirement |
| Notifications | ntfy.sh | Specified requirement |
| Plaintext vault on disk | Never | Core security requirement |

### 1.4 No-LUKS Decision

LUKS full-disk encryption is **not used** in this architecture. The rationale is operational: a headless Raspberry Pi with LUKS requires either a network unlock mechanism (e.g., Mandos, Clevis/Tang) or physical keyboard access at boot. Both options add complexity and failure modes that are inappropriate for a single-purpose backup appliance that must recover automatically after power loss.

The accepted risk is physical access. If an attacker gains physical possession of the SD card or storage medium, the operating system, scripts, and local backup files are readable. This risk is mitigated by:

1. All vault exports are GPG-encrypted before being written to disk. A raw vault JSON file never exists on disk at any point.
2. The GPG passphrase is stored in a `chmod 600` file readable only by the `bwbackup` service user.
3. The Bitwarden API key and vault master password are stored in a `chmod 600` environment file readable only by `bwbackup`.
4. Physical access to the Pi should be controlled as part of site security.

Operators who require encryption at rest for the SD card must implement LUKS with a network unlock solution independently; that is out of scope for this tier.

### 1.5 Threat Model

| Threat | Mitigation |
|---|---|
| Remote attacker reading vault exports | GPG AES-256 encryption; no plaintext ever on disk |
| Bitwarden account credentials leaked via backup system | API key + password stored in `chmod 600` env file; not logged |
| pCloud breach exposing vault data | rclone crypt remote encrypts filenames and file contents before upload |
| Backup silently failing | ntfy.sh failure notification; SHA-256 sidecar verification |
| Corrupt or truncated backup | `gpg --list-packets` check after every backup; monthly full-decrypt + item count |
| Physical theft of Pi | Plaintext vault never on disk; credentials only accessible to `bwbackup` user |

---

## 2. System Requirements

### 2.1 Hardware

Any of the following Raspberry Pi models are supported:

- Raspberry Pi 3 Model B+
- Raspberry Pi 4 Model B (any RAM variant)
- Raspberry Pi 5

Minimum storage: 8 GB SD card or USB drive. Recommended: 16 GB or larger to accommodate log rotation and 12 months of local monthly backups. A typical encrypted vault export is 10–100 KB; local storage is not a practical constraint.

### 2.2 Operating System

- **Raspberry Pi OS Bookworm** (Debian 12 base), 64-bit or 32-bit
- Kernel: 6.x (as shipped with Bookworm)
- The system must have internet access on port 443/TCP for Bitwarden API, pCloud, and ntfy.sh

### 2.3 Required Packages

Install via `apt` unless noted otherwise:

| Package | Version | Source | Purpose |
|---|---|---|---|
| `gnupg` | 2.2+ (Bookworm ships 2.2.40) | apt | GPG encryption |
| `rclone` | 1.65+ recommended | apt or rclone.org | pCloud sync |
| `curl` | 7.88+ (Bookworm default) | apt | ntfy.sh notifications |
| `python3` | 3.11 (Bookworm default) | apt | JSON item-count verification |
| `jq` | 1.6+ | apt | JSON parsing in scripts |
| `nodejs` | 18 LTS or 20 LTS | NodeSource PPA | Bitwarden CLI runtime |
| `npm` | 9+ (bundled with Node) | NodeSource PPA | Bitwarden CLI install |

Install commands:

```bash
# System packages
sudo apt update
sudo apt install -y gnupg rclone curl python3 jq

# Node.js 20 LTS via NodeSource
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Bitwarden CLI (global, accessible to all users)
sudo npm install -g @bitwarden/cli

# Verify
bw --version
rclone --version
gpg --version
```

### 2.4 Service User

A dedicated system user with no login shell:

```bash
sudo useradd -r -m -d /home/bwbackup -s /usr/sbin/nologin bwbackup
```

The `-m` flag creates the home directory at `/home/bwbackup`. This is required because the Bitwarden CLI stores its data directory under the user's home.

---

## 3. Security Model

### 3.1 Secrets Inventory

The following secrets exist in the system:

| Secret | Description | Storage Location | Permissions |
|---|---|---|---|
| `BW_CLIENTID` | Bitwarden API key client ID | `/etc/bw-backup/env` | `600`, owner `bwbackup` |
| `BW_CLIENTSECRET` | Bitwarden API key client secret | `/etc/bw-backup/env` | `600`, owner `bwbackup` |
| `BW_PASSWORD` | Bitwarden vault master password | `/etc/bw-backup/env` | `600`, owner `bwbackup` |
| GPG passphrase | Passphrase for symmetric encryption | `/etc/bw-backup/passphrase.key` | `600`, owner `bwbackup` |
| rclone config | pCloud OAuth token + crypt remote config | `/home/bwbackup/.config/rclone/rclone.conf` | `600`, owner `bwbackup` |
| ntfy.sh topic | Topic name (acts as a shared secret) | `/etc/bw-backup/env` | `600`, owner `bwbackup` |

### 3.2 Credential Storage

**Environment file** (`/etc/bw-backup/env`):

```
BW_CLIENTID=user.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
BW_CLIENTSECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
BW_PASSWORD=your_vault_master_password
NTFY_TOPIC=your-private-topic-name
```

Set ownership and permissions immediately after creation:

```bash
sudo chown bwbackup:bwbackup /etc/bw-backup/env
sudo chmod 600 /etc/bw-backup/env
```

**GPG passphrase file** (`/etc/bw-backup/passphrase.key`):

Contains only the raw passphrase, no trailing newline required (GPG handles both). Generate a strong passphrase:

```bash
openssl rand -base64 48 | sudo tee /etc/bw-backup/passphrase.key
sudo chown bwbackup:bwbackup /etc/bw-backup/passphrase.key
sudo chmod 600 /etc/bw-backup/passphrase.key
```

### 3.3 Process Isolation

- The backup script runs exclusively as user `bwbackup`
- `bwbackup` has no sudo privileges
- `bwbackup` has no login shell (`/usr/sbin/nologin`)
- `umask 077` is set at the top of the backup script so all created files default to `600`
- The Bitwarden CLI session token (`BW_SESSION`) exists only in memory as an environment variable for the duration of the script run; it is never written to disk

### 3.4 No Plaintext Vault Policy

The vault JSON export is piped directly from `bw export` through GPG and written as ciphertext. The pipeline is:

```
bw export --format json --output /dev/stdout | gpg [flags] --output "$OUTFILE"
```

At no point does a plaintext `.json` file exist on disk. This is a hard requirement that must not be relaxed.

### 3.5 ufw Firewall

Configure the host firewall to allow only the minimum required outbound traffic:

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow out 443/tcp comment 'HTTPS for Bitwarden, pCloud, ntfy.sh'
sudo ufw allow out 53/udp comment 'DNS resolution'
sudo ufw allow in on lo
sudo ufw allow out on lo
sudo ufw enable
```

Adjust SSH access rules according to your operational needs (e.g., `sudo ufw allow in 22/tcp from 192.168.1.0/24`).

### 3.6 SSH Hardening

In `/etc/ssh/sshd_config`:

```
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
```

Restart sshd after changes: `sudo systemctl restart sshd`

---

## 4. Component Specifications

### 4.1 Bitwarden CLI

#### 4.1.1 Auth Flow

Authentication is two-step. Step one logs in using the API key, which bypasses two-factor authentication and is required for unattended operation. Step two unlocks the vault to obtain a session token.

```bash
# Step 1: Login with API key
# Reads BW_CLIENTID and BW_CLIENTSECRET from environment
bw login --apikey

# Step 2: Unlock vault and capture session token
BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)
export BW_SESSION
```

The `--raw` flag causes `bw unlock` to output only the session token string with no surrounding text, making it safe to capture directly.

#### 4.1.2 Required Environment Variables

| Variable | Set by | Description |
|---|---|---|
| `BW_CLIENTID` | `EnvironmentFile` in systemd service | API key client ID |
| `BW_CLIENTSECRET` | `EnvironmentFile` in systemd service | API key client secret |
| `BW_PASSWORD` | `EnvironmentFile` in systemd service | Vault master password |
| `BW_SESSION` | Set by `bw unlock --raw` at runtime | Session token (in-memory only) |

#### 4.1.3 Sync Before Export

Always sync before exporting to ensure the local CLI cache reflects the current server state:

```bash
bw sync --session "$BW_SESSION"
```

Failure of `bw sync` must abort the backup with a failure notification.

#### 4.1.4 Export Command

```bash
bw export \
    --format json \
    --output /dev/stdout \
    --session "$BW_SESSION" \
    | gpg --batch --pinentry-mode loopback \
          --passphrase-fd 3 \
          --symmetric \
          --cipher-algo AES256 \
          --s2k-digest-algo SHA512 \
          --s2k-count 65011712 \
          --output "$OUTFILE" \
    3< "$GPG_PASSPHRASE_FILE"
```

The exit code of the pipeline must be checked. Use `set -o pipefail` so that a failure in `bw export` is not masked by GPG's exit code.

#### 4.1.5 Not Exported

The following Bitwarden data is **not** included in the `bw export --format json` output and cannot be backed up by this system:

- File attachments
- Bitwarden Send items
- Items in the vault trash

Operators requiring attachment backup must implement a separate mechanism outside the scope of this specification.

#### 4.1.6 Lock and Logout

After export, release the session and log out:

```bash
bw lock
bw logout
```

Both commands should be run in the script's `EXIT` trap to ensure cleanup even on failure.

---

### 4.2 Encryption

#### 4.2.1 Algorithm Selection

GPG symmetric encryption with the following parameters:

| Parameter | Value | Rationale |
|---|---|---|
| Cipher | AES-256 | Strong, widely supported |
| S2K digest | SHA-512 | Stronger key derivation hash |
| S2K count | 65,011,712 | Maximum standard iteration count; resist brute-force |
| Compression | None (not specified; GPG default) | JSON already low-entropy; compression before encryption can leak information via size oracle |

#### 4.2.2 GPG Encryption Command

```bash
gpg \
    --batch \
    --pinentry-mode loopback \
    --passphrase-fd 3 \
    --symmetric \
    --cipher-algo AES256 \
    --s2k-digest-algo SHA512 \
    --s2k-count 65011712 \
    --output "$OUTFILE" \
    3< "$GPG_PASSPHRASE_FILE"
```

The `--passphrase-fd 3` with file descriptor 3 opened from the passphrase file avoids passing the passphrase via command-line arguments (which would be visible in `ps` output) or via `--passphrase` (which also appears in process listings).

`--batch` and `--pinentry-mode loopback` are both required for non-interactive operation.

#### 4.2.3 Output File Naming

```
bitwarden_YYYY-MM-DD_HHMMSS.json.gpg
```

Example: `bitwarden_2026-03-29_020015.json.gpg`

The timestamp uses the local system time at the moment the script runs. The `.json.gpg` double extension clearly identifies the content type and encryption wrapper.

#### 4.2.4 Passphrase File Specification

- Path: `/etc/bw-backup/passphrase.key`
- Content: raw passphrase string (UTF-8, no shell special characters recommended)
- Permissions: `600`
- Owner: `bwbackup:bwbackup`
- The passphrase file must exist and be non-empty before the script runs; the script must validate this and abort if the check fails

#### 4.2.5 Decryption Command (for reference/restore)

```bash
gpg \
    --batch \
    --pinentry-mode loopback \
    --passphrase-file /etc/bw-backup/passphrase.key \
    --output vault_restored.json \
    --decrypt bitwarden_2026-03-29_020015.json.gpg
```

---

### 4.3 Backup Script Logic

The script executes steps in the following order. Any step that fails must trigger the failure notification and exit non-zero, except lock/logout which run unconditionally via the EXIT trap.

1. Set `umask 077` and `set -euo pipefail`
2. Load configuration from environment (provided by systemd `EnvironmentFile`)
3. Validate that required variables are set and passphrase file exists
4. Determine backup tier (daily / weekly / monthly) based on current date
5. Construct output filename with timestamp
6. Log start
7. Login to Bitwarden with API key (`bw login --apikey`)
8. Unlock vault and capture `BW_SESSION`
9. Sync vault (`bw sync`)
10. Export vault and encrypt to output file in the appropriate tier directory
11. Write SHA-256 sidecar file (`.sha256`)
12. Verify the GPG container with `gpg --list-packets`
13. Verify SHA-256 checksum
14. Lock and logout Bitwarden (also in EXIT trap)
15. Apply GFS rotation pruning in the appropriate tier directory
16. Sync local backup directory to pCloud crypt remote with rclone
17. Send ntfy.sh success notification
18. Log completion

On any failure (caught by `trap ... ERR` or `set -e`):

1. Run lock/logout (via EXIT trap)
2. Send ntfy.sh failure notification with step identification
3. Log failure
4. Exit non-zero

---

### 4.4 GFS Rotation

#### 4.4.1 Directory Layout

```
/var/backups/bitwarden/
├── daily/
├── weekly/
└── monthly/
```

Create with correct permissions:

```bash
sudo mkdir -p /var/backups/bitwarden/{daily,weekly,monthly}
sudo chown -R bwbackup:bwbackup /var/backups/bitwarden
sudo chmod -R 700 /var/backups/bitwarden
```

#### 4.4.2 Tier Assignment

Each backup run produces exactly one backup placed in exactly one tier directory:

| Condition | Tier | Directory |
|---|---|---|
| Day of month is 1 | Monthly | `/var/backups/bitwarden/monthly/` |
| Day of week is Sunday (and not 1st) | Weekly | `/var/backups/bitwarden/weekly/` |
| All other days | Daily | `/var/backups/bitwarden/daily/` |

In bash:

```bash
DAY_OF_MONTH=$(date +%-d)
DAY_OF_WEEK=$(date +%u)   # 1=Monday, 7=Sunday

if [[ "$DAY_OF_MONTH" -eq 1 ]]; then
    TIER="monthly"
elif [[ "$DAY_OF_WEEK" -eq 7 ]]; then
    TIER="weekly"
else
    TIER="daily"
fi

BACKUP_DIR="/var/backups/bitwarden/$TIER"
```

#### 4.4.3 Retention Policy

| Tier | Retention |
|---|---|
| Daily | Last 7 files |
| Weekly | Last 4 files |
| Monthly | Last 12 files |

#### 4.4.4 Prune Logic

After the new backup is written and verified, prune old files in the tier directory. Files are sorted by name (which sorts by timestamp due to the `YYYY-MM-DD_HHMMSS` naming convention). Both the `.json.gpg` file and its `.sha256` sidecar must be deleted together.

```bash
prune_tier() {
    local dir="$1"
    local keep="$2"

    # List all .json.gpg files sorted oldest-first
    mapfile -t all_backups < <(ls -1 "$dir"/*.json.gpg 2>/dev/null | sort)

    local total="${#all_backups[@]}"
    if [[ "$total" -le "$keep" ]]; then
        return 0
    fi

    local to_delete=$(( total - keep ))
    for (( i=0; i<to_delete; i++ )); do
        local f="${all_backups[$i]}"
        log_info "Pruning old backup: $f"
        rm -f "$f" "${f%.json.gpg}.sha256"
    done
}
```

Call after successful backup:

```bash
case "$TIER" in
    daily)   prune_tier "$BACKUP_DIR" 7  ;;
    weekly)  prune_tier "$BACKUP_DIR" 4  ;;
    monthly) prune_tier "$BACKUP_DIR" 12 ;;
esac
```

---

### 4.5 rclone pCloud Crypt Remote

#### 4.5.1 Remote Configuration

Two remotes must be configured in `/home/bwbackup/.config/rclone/rclone.conf`:

**Step 1 — pCloud remote (named `pcloud`):**

Run interactively as `bwbackup` once during setup:

```bash
sudo -u bwbackup rclone config
```

Select: New remote → name `pcloud` → type `pcloud` → follow OAuth flow. This generates an OAuth token stored in the rclone config file.

**Step 2 — Crypt remote (named `bw-crypt`):**

Add to `/home/bwbackup/.config/rclone/rclone.conf` (or via `rclone config`):

```ini
[bw-crypt]
type = crypt
remote = pcloud:bitwarden-backups
filename_encryption = standard
directory_name_encryption = true
password = <rclone-obscured-password>
password2 = <rclone-obscured-salt>
```

Where:
- `remote = pcloud:bitwarden-backups` — the path on pCloud where encrypted backups are stored
- `filename_encryption = standard` — filenames are encrypted
- `directory_name_encryption = true` — directory names are also encrypted
- `password` and `password2` — generated via `rclone obscure your-crypt-password`

Generate the obscured passwords:

```bash
sudo -u bwbackup rclone obscure 'your-crypt-password'
sudo -u bwbackup rclone obscure 'your-crypt-salt'
```

Store the plaintext crypt password and salt in `/etc/bw-backup/env` or separately secured, as they are needed for recovery.

Config file permissions:

```bash
sudo chown bwbackup:bwbackup /home/bwbackup/.config/rclone/rclone.conf
sudo chmod 600 /home/bwbackup/.config/rclone/rclone.conf
```

#### 4.5.2 Sync Command

```bash
rclone sync \
    /var/backups/bitwarden/ \
    bw-crypt: \
    --checksum \
    --log-file /var/log/rclone-bitwarden.log \
    --log-level INFO
```

`--checksum` instructs rclone to use content checksums rather than modification time + size for change detection. This is more reliable for backup verification.

`rclone sync` makes the destination match the source, including deletions. This means remote files corresponding to locally pruned backups will also be removed on the next sync.

The sync command runs as user `bwbackup`. If `rclone sync` exits non-zero, the script must treat this as a failure and send a failure notification.

---

### 4.6 ntfy.sh Notifications

#### 4.6.1 Topic Naming

The ntfy.sh topic acts as a shared secret — anyone who knows the topic name can read and send messages. Use a long, random, unguessable topic name:

```bash
openssl rand -hex 16
# Example output: a3f8c2e1b4d97650f1e2a3b4c5d6e7f8
```

Recommended format: `bwbackup-<random-hex>` (e.g., `bwbackup-a3f8c2e1b4d97650f1e2a3b4c5d6e7f8`)

Store in `/etc/bw-backup/env` as `NTFY_TOPIC`.

#### 4.6.2 Notification Functions

**Success notification:**

```bash
notify_success() {
    local message="$1"
    curl --silent --fail \
        -H "Title: Bitwarden Backup OK" \
        -H "Priority: low" \
        -H "Tags: white_check_mark" \
        -d "$message" \
        "https://ntfy.sh/${NTFY_TOPIC}" \
        || true   # notification failure must not fail the backup job
}
```

**Failure notification:**

```bash
notify_failure() {
    local message="$1"
    curl --silent --fail \
        -H "Title: Bitwarden Backup FAILED" \
        -H "Priority: high" \
        -H "Tags: rotating_light" \
        -d "$message" \
        "https://ntfy.sh/${NTFY_TOPIC}" \
        || true
}
```

The `|| true` on the curl call prevents a notification failure from masking the original backup failure or triggering recursive failure handling.

#### 4.6.3 Message Format

**Success:**
```
Backup completed: bitwarden_2026-03-29_020015.json.gpg (tier: daily)
Synced to pCloud. Local: 7 daily, 4 weekly, 3 monthly backups retained.
```

**Failure:**
```
Backup FAILED at step: bw sync
Host: raspberrypi | Time: 2026-03-29T02:00:45+00:00
Check /var/log/bitwarden-backup.log for details.
```

#### 4.6.4 Self-Hosted ntfy.sh

If using a self-hosted ntfy.sh instance, replace `https://ntfy.sh` with your instance URL in all curl calls. The topic name and authentication approach remain the same.

---

### 4.7 systemd Timer and Service

#### 4.7.1 Service Unit

File: `/etc/systemd/system/bw-backup.service`

```ini
[Unit]
Description=Bitwarden Vault Backup
After=network-online.target
Wants=network-online.target
Documentation=file:///home/bwbackup/docs/spec.md

[Service]
Type=oneshot
User=bwbackup
Group=bwbackup
EnvironmentFile=/etc/bw-backup/env
ExecStart=/home/bwbackup/scripts/bw-backup.sh
StandardOutput=append:/var/log/bitwarden-backup.log
StandardError=append:/var/log/bitwarden-backup.log

# Security hardening
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=/var/backups/bitwarden /var/log /home/bwbackup/.config /tmp
PrivateTmp=true
```

Notes:
- `After=network-online.target` ensures the backup does not start before the network is fully up, which is important on Pi hardware where network acquisition can take several seconds after boot.
- `ProtectSystem=strict` and `ProtectHome=read-only` limit write access to explicitly listed paths.
- `PrivateTmp=true` gives the service an isolated `/tmp`.

#### 4.7.2 Timer Unit

File: `/etc/systemd/system/bw-backup.timer`

```ini
[Unit]
Description=Bitwarden Vault Backup — Daily Timer
Documentation=file:///home/bwbackup/docs/spec.md

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
Unit=bw-backup.service

[Install]
WantedBy=timers.target
```

`Persistent=true` means if the system was off at 02:00, the timer will fire as soon as the system next boots, provided the job has not run today. This ensures no backup is silently skipped due to downtime.

#### 4.7.3 Enabling

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bw-backup.timer
sudo systemctl list-timers bw-backup.timer
```

---

### 4.8 Logging

#### 4.8.1 Log Format

Every log line follows this format:

```
[ISO8601_TIMESTAMP] [LEVEL] message
```

Example:

```
[2026-03-29T02:00:15+00:00] [INFO] Starting backup run. Tier: daily
[2026-03-29T02:00:16+00:00] [INFO] bw login successful
[2026-03-29T02:00:17+00:00] [INFO] bw unlock successful, session token acquired
[2026-03-29T02:00:20+00:00] [INFO] bw sync completed
[2026-03-29T02:00:22+00:00] [INFO] Export and encryption complete: bitwarden_2026-03-29_020022.json.gpg
[2026-03-29T02:00:22+00:00] [INFO] SHA-256 sidecar written
[2026-03-29T02:00:22+00:00] [INFO] GPG packet check passed
[2026-03-29T02:00:22+00:00] [INFO] Checksum verification passed
[2026-03-29T02:00:23+00:00] [INFO] Pruned 0 old backups from daily
[2026-03-29T02:00:45+00:00] [INFO] rclone sync completed
[2026-03-29T02:00:45+00:00] [INFO] Backup run complete. Duration: 30s
```

Implemented in the script:

```bash
log() {
    local level="$1"; shift
    printf '[%s] [%s] %s\n' "$(date --iso-8601=seconds)" "$level" "$*" \
        | tee -a /var/log/bitwarden-backup.log
}
log_info()  { log INFO  "$@"; }
log_error() { log ERROR "$@"; }
log_warn()  { log WARN  "$@"; }
```

Secrets (passwords, session tokens) must never appear in log output.

#### 4.8.2 Log File Path

```
/var/log/bitwarden-backup.log
```

Create and set permissions:

```bash
sudo touch /var/log/bitwarden-backup.log
sudo chown bwbackup:bwbackup /var/log/bitwarden-backup.log
sudo chmod 640 /var/log/bitwarden-backup.log
```

#### 4.8.3 rclone Log

rclone writes its own log separately:

```
/var/log/rclone-bitwarden.log
```

Create:

```bash
sudo touch /var/log/rclone-bitwarden.log
sudo chown bwbackup:bwbackup /var/log/rclone-bitwarden.log
sudo chmod 640 /var/log/rclone-bitwarden.log
```

#### 4.8.4 logrotate Configuration

File: `/etc/logrotate.d/bitwarden-backup`

```
/var/log/bitwarden-backup.log /var/log/rclone-bitwarden.log {
    weekly
    rotate 52
    compress
    delaycompress
    missingok
    notifempty
    create 640 bwbackup bwbackup
}
```

This retains one year of weekly-rotated logs. `delaycompress` keeps the most recent rotated file uncompressed for easy inspection.

---

### 4.9 Integrity Verification

#### 4.9.1 Post-Backup Checks (Every Run)

**SHA-256 sidecar creation:**

Immediately after GPG encryption, write a sidecar checksum file:

```bash
sha256sum "$OUTFILE" | awk '{print $1}' > "${OUTFILE%.json.gpg}.sha256"
```

The sidecar file contains only the hex digest with no filename.

**GPG packet structure check:**

Verify the GPG container is structurally valid without requiring the passphrase:

```bash
gpg --list-packets "$OUTFILE" > /dev/null 2>&1
```

Exit code 0 confirms the file has valid GPG structure (correct magic bytes, valid packet headers). This will catch truncated writes or filesystem errors but does not attempt decryption.

**Checksum verification:**

Re-verify the sidecar immediately after writing:

```bash
EXPECTED=$(cat "${OUTFILE%.json.gpg}.sha256")
ACTUAL=$(sha256sum "$OUTFILE" | awk '{print $1}')
if [[ "$EXPECTED" != "$ACTUAL" ]]; then
    log_error "Checksum mismatch for $OUTFILE"
    exit 1
fi
```

#### 4.9.2 Monthly Full Verification (Spot Check)

On the 1st of each month, after the regular backup completes, the script performs a full decryption and JSON validation:

```bash
if [[ "$TIER" == "monthly" ]]; then
    TMPDIR=$(mktemp -d)
    TMPFILE="$TMPDIR/verify.json"

    gpg --batch \
        --pinentry-mode loopback \
        --passphrase-file "$GPG_PASSPHRASE_FILE" \
        --output "$TMPFILE" \
        --decrypt "$OUTFILE"

    # Validate JSON and count items
    ITEM_COUNT=$(python3 -c "
import json, sys
with open('$TMPFILE') as f:
    data = json.load(f)
items = data.get('items', [])
print(len(items))
sys.exit(0 if len(items) > 0 else 1)
")

    log_info "Monthly spot-check: decryption successful, item count: $ITEM_COUNT"
    rm -rf "$TMPDIR"
fi
```

The temporary file is created in a `mktemp -d` directory under `/tmp` (which is private to the service due to `PrivateTmp=true`). It is always removed in the `EXIT` trap.

---

## 5. File and Directory Layout

```
/
├── etc/
│   ├── bw-backup/
│   │   ├── env                          # rw------- bwbackup:bwbackup  (secrets env file)
│   │   └── passphrase.key               # rw------- bwbackup:bwbackup  (GPG passphrase)
│   ├── systemd/system/
│   │   ├── bw-backup.service            # rw-r--r-- root:root
│   │   └── bw-backup.timer              # rw-r--r-- root:root
│   └── logrotate.d/
│       └── bitwarden-backup             # rw-r--r-- root:root
├── home/
│   └── bwbackup/
│       ├── scripts/
│       │   └── bw-backup.sh             # rwx------ bwbackup:bwbackup  (backup script)
│       └── .config/
│           └── rclone/
│               └── rclone.conf          # rw------- bwbackup:bwbackup  (rclone remotes)
├── var/
│   ├── backups/
│   │   └── bitwarden/                   # rwx------ bwbackup:bwbackup
│   │       ├── daily/                   # rwx------ bwbackup:bwbackup
│   │       │   ├── bitwarden_YYYY-MM-DD_HHMMSS.json.gpg   # rw------- bwbackup:bwbackup
│   │       │   └── bitwarden_YYYY-MM-DD_HHMMSS.sha256     # rw------- bwbackup:bwbackup
│   │       ├── weekly/                  # rwx------ bwbackup:bwbackup
│   │       │   └── (same file pattern)
│   │       └── monthly/                 # rwx------ bwbackup:bwbackup
│   │           └── (same file pattern)
│   └── log/
│       ├── bitwarden-backup.log         # rw-r----- bwbackup:bwbackup
│       └── rclone-bitwarden.log         # rw-r----- bwbackup:bwbackup
└── usr/
    └── local/
        └── bin/
            └── bw -> (npm global install location)
```

---

## 6. Backup Script

File: `/home/bwbackup/scripts/bw-backup.sh`
Owner: `bwbackup:bwbackup`
Permissions: `700`

```bash
#!/usr/bin/env bash
# /home/bwbackup/scripts/bw-backup.sh
# Bitwarden Vault Backup — Tier 2 Raspberry Pi Architecture
# See docs/spec.md for full specification.

set -euo pipefail
umask 077

# ---------------------------------------------------------------------------
# Configuration (sourced from EnvironmentFile=/etc/bw-backup/env via systemd)
# Required variables: BW_CLIENTID, BW_CLIENTSECRET, BW_PASSWORD, NTFY_TOPIC
# ---------------------------------------------------------------------------
readonly GPG_PASSPHRASE_FILE="/etc/bw-backup/passphrase.key"
readonly BACKUP_BASE="/var/backups/bitwarden"
readonly LOG_FILE="/var/log/bitwarden-backup.log"
readonly RCLONE_REMOTE="bw-crypt:"
readonly NTFY_URL="https://ntfy.sh"

# ---------------------------------------------------------------------------
# Logging
# ---------------------------------------------------------------------------
log() {
    local level="$1"; shift
    printf '[%s] [%s] %s\n' "$(date --iso-8601=seconds)" "$level" "$*" \
        | tee -a "$LOG_FILE"
}
log_info()  { log INFO  "$@"; }
log_warn()  { log WARN  "$@"; }
log_error() { log ERROR "$@"; }

# ---------------------------------------------------------------------------
# Notifications
# ---------------------------------------------------------------------------
notify_success() {
    local message="$1"
    curl --silent --max-time 10 \
        -H "Title: Bitwarden Backup OK" \
        -H "Priority: low" \
        -H "Tags: white_check_mark" \
        -d "$message" \
        "${NTFY_URL}/${NTFY_TOPIC}" \
        || log_warn "ntfy.sh success notification failed (non-fatal)"
}

notify_failure() {
    local message="$1"
    curl --silent --max-time 10 \
        -H "Title: Bitwarden Backup FAILED" \
        -H "Priority: high" \
        -H "Tags: rotating_light" \
        -d "$message" \
        "${NTFY_URL}/${NTFY_TOPIC}" \
        || log_warn "ntfy.sh failure notification failed"
}

# ---------------------------------------------------------------------------
# Cleanup and failure trap
# ---------------------------------------------------------------------------
SCRIPT_START=$(date +%s)
BACKUP_FILE=""          # set once the target filename is known
TMPDIR_VERIFY=""        # set during monthly spot-check
FAILURE_STEP="unknown"

cleanup() {
    local exit_code=$?

    # Always attempt to lock/logout regardless of exit code
    if bw status 2>/dev/null | grep -q '"status":"unlocked"'; then
        bw lock    2>/dev/null || true
    fi
    bw logout  2>/dev/null || true

    # Clean up temporary verification directory if present
    if [[ -n "$TMPDIR_VERIFY" && -d "$TMPDIR_VERIFY" ]]; then
        rm -rf "$TMPDIR_VERIFY"
    fi

    if [[ $exit_code -ne 0 ]]; then
        local host
        host=$(hostname)
        local timestamp
        timestamp=$(date --iso-8601=seconds)
        log_error "Backup FAILED at step: ${FAILURE_STEP}"
        notify_failure "Backup FAILED at step: ${FAILURE_STEP}
Host: ${host} | Time: ${timestamp}
Check ${LOG_FILE} for details."
    fi
}
trap cleanup EXIT

# ---------------------------------------------------------------------------
# Validation
# ---------------------------------------------------------------------------
validate_environment() {
    FAILURE_STEP="environment-validation"
    local errors=0

    for var in BW_CLIENTID BW_CLIENTSECRET BW_PASSWORD NTFY_TOPIC; do
        if [[ -z "${!var:-}" ]]; then
            log_error "Required environment variable not set: $var"
            (( errors++ )) || true
        fi
    done

    if [[ ! -f "$GPG_PASSPHRASE_FILE" ]]; then
        log_error "GPG passphrase file not found: $GPG_PASSPHRASE_FILE"
        (( errors++ )) || true
    fi

    if [[ ! -s "$GPG_PASSPHRASE_FILE" ]]; then
        log_error "GPG passphrase file is empty: $GPG_PASSPHRASE_FILE"
        (( errors++ )) || true
    fi

    if [[ $errors -gt 0 ]]; then
        exit 1
    fi
}

# ---------------------------------------------------------------------------
# GFS tier determination
# ---------------------------------------------------------------------------
determine_tier() {
    local day_of_month
    day_of_month=$(date +%-d)
    local day_of_week
    day_of_week=$(date +%u)   # 1=Monday, 7=Sunday

    if [[ "$day_of_month" -eq 1 ]]; then
        echo "monthly"
    elif [[ "$day_of_week" -eq 7 ]]; then
        echo "weekly"
    else
        echo "daily"
    fi
}

# ---------------------------------------------------------------------------
# GFS pruning
# ---------------------------------------------------------------------------
prune_tier() {
    local dir="$1"
    local keep="$2"

    mapfile -t all_backups < <(ls -1 "$dir"/*.json.gpg 2>/dev/null | sort)
    local total="${#all_backups[@]}"

    if [[ "$total" -le "$keep" ]]; then
        log_info "No pruning needed in $dir ($total backup(s), keep $keep)"
        return 0
    fi

    local to_delete=$(( total - keep ))
    log_info "Pruning $to_delete old backup(s) from $dir (have $total, keep $keep)"
    for (( i=0; i<to_delete; i++ )); do
        local f="${all_backups[$i]}"
        log_info "Pruning: $(basename "$f")"
        rm -f "$f" "${f%.json.gpg}.sha256"
    done
}

# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------
main() {
    log_info "========================================================"
    log_info "Bitwarden backup starting"

    # Validation
    validate_environment

    # Determine tier
    FAILURE_STEP="tier-determination"
    local tier
    tier=$(determine_tier)
    local backup_dir="${BACKUP_BASE}/${tier}"
    log_info "Backup tier: $tier | Directory: $backup_dir"

    # Construct output filename
    local timestamp
    timestamp=$(date +%Y-%m-%d_%H%M%S)
    local filename="bitwarden_${timestamp}.json.gpg"
    BACKUP_FILE="${backup_dir}/${filename}"

    # ---------------------------------------------------------------------------
    # Step 1: Bitwarden login
    # ---------------------------------------------------------------------------
    FAILURE_STEP="bw-login"
    log_info "Logging in to Bitwarden with API key"
    bw login --apikey 2>&1 | grep -v -i "secret\|password\|token" | while read -r line; do
        log_info "bw login: $line"
    done || true
    # Check login succeeded
    if ! bw status 2>/dev/null | grep -qE '"status":"(unlocked|locked)"'; then
        log_error "bw login failed — vault status not locked or unlocked after login"
        exit 1
    fi
    log_info "bw login successful"

    # ---------------------------------------------------------------------------
    # Step 2: Unlock vault
    # ---------------------------------------------------------------------------
    FAILURE_STEP="bw-unlock"
    log_info "Unlocking vault"
    BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)
    export BW_SESSION
    if [[ -z "$BW_SESSION" ]]; then
        log_error "bw unlock returned empty session token"
        exit 1
    fi
    log_info "bw unlock successful, session token acquired"

    # ---------------------------------------------------------------------------
    # Step 3: Sync
    # ---------------------------------------------------------------------------
    FAILURE_STEP="bw-sync"
    log_info "Syncing vault"
    bw sync --session "$BW_SESSION"
    log_info "bw sync completed"

    # ---------------------------------------------------------------------------
    # Step 4: Export and encrypt (no plaintext on disk)
    # ---------------------------------------------------------------------------
    FAILURE_STEP="export-encrypt"
    log_info "Exporting and encrypting vault to $BACKUP_FILE"
    bw export \
        --format json \
        --output /dev/stdout \
        --session "$BW_SESSION" \
        | gpg \
              --batch \
              --pinentry-mode loopback \
              --passphrase-fd 3 \
              --symmetric \
              --cipher-algo AES256 \
              --s2k-digest-algo SHA512 \
              --s2k-count 65011712 \
              --output "$BACKUP_FILE" \
        3< "$GPG_PASSPHRASE_FILE"

    if [[ ! -f "$BACKUP_FILE" ]]; then
        log_error "Output file not created: $BACKUP_FILE"
        exit 1
    fi
    if [[ ! -s "$BACKUP_FILE" ]]; then
        log_error "Output file is empty: $BACKUP_FILE"
        exit 1
    fi
    log_info "Export and encryption complete: $(basename "$BACKUP_FILE")"

    # ---------------------------------------------------------------------------
    # Step 5: SHA-256 sidecar
    # ---------------------------------------------------------------------------
    FAILURE_STEP="checksum-write"
    local sidecar="${BACKUP_FILE%.json.gpg}.sha256"
    sha256sum "$BACKUP_FILE" | awk '{print $1}' > "$sidecar"
    log_info "SHA-256 sidecar written: $(basename "$sidecar")"

    # ---------------------------------------------------------------------------
    # Step 6: GPG packet structure check
    # ---------------------------------------------------------------------------
    FAILURE_STEP="gpg-packet-check"
    if ! gpg --list-packets "$BACKUP_FILE" > /dev/null 2>&1; then
        log_error "GPG packet check failed for $BACKUP_FILE — file may be corrupt"
        exit 1
    fi
    log_info "GPG packet check passed"

    # ---------------------------------------------------------------------------
    # Step 7: Checksum verification
    # ---------------------------------------------------------------------------
    FAILURE_STEP="checksum-verify"
    local expected actual
    expected=$(cat "$sidecar")
    actual=$(sha256sum "$BACKUP_FILE" | awk '{print $1}')
    if [[ "$expected" != "$actual" ]]; then
        log_error "Checksum mismatch for $(basename "$BACKUP_FILE")"
        log_error "  expected: $expected"
        log_error "  actual:   $actual"
        exit 1
    fi
    log_info "Checksum verification passed"

    # ---------------------------------------------------------------------------
    # Step 8: Lock and logout (also in EXIT trap, belt-and-suspenders)
    # ---------------------------------------------------------------------------
    FAILURE_STEP="bw-lock-logout"
    bw lock
    bw logout
    log_info "Bitwarden session locked and logged out"
    unset BW_SESSION

    # ---------------------------------------------------------------------------
    # Step 9: Monthly spot-check (full decryption + item count)
    # ---------------------------------------------------------------------------
    if [[ "$tier" == "monthly" ]]; then
        FAILURE_STEP="monthly-spot-check"
        log_info "Performing monthly full-decryption spot-check"
        TMPDIR_VERIFY=$(mktemp -d)
        local verify_file="${TMPDIR_VERIFY}/verify.json"

        gpg \
            --batch \
            --pinentry-mode loopback \
            --passphrase-file "$GPG_PASSPHRASE_FILE" \
            --output "$verify_file" \
            --decrypt "$BACKUP_FILE"

        local item_count
        item_count=$(python3 -c "
import json, sys
try:
    with open('${verify_file}') as f:
        data = json.load(f)
    items = data.get('items', [])
    count = len(items)
    print(count)
    sys.exit(0 if count > 0 else 1)
except Exception as e:
    print('ERROR: ' + str(e), file=sys.stderr)
    sys.exit(1)
")
        log_info "Monthly spot-check passed: $item_count item(s) in decrypted vault"
        rm -rf "$TMPDIR_VERIFY"
        TMPDIR_VERIFY=""
    fi

    # ---------------------------------------------------------------------------
    # Step 10: GFS rotation
    # ---------------------------------------------------------------------------
    FAILURE_STEP="gfs-rotation"
    case "$tier" in
        daily)   prune_tier "$backup_dir" 7  ;;
        weekly)  prune_tier "$backup_dir" 4  ;;
        monthly) prune_tier "$backup_dir" 12 ;;
    esac

    # ---------------------------------------------------------------------------
    # Step 11: rclone sync to pCloud crypt remote
    # ---------------------------------------------------------------------------
    FAILURE_STEP="rclone-sync"
    log_info "Syncing to pCloud crypt remote ($RCLONE_REMOTE)"
    rclone sync \
        "$BACKUP_BASE/" \
        "$RCLONE_REMOTE" \
        --checksum \
        --log-file /var/log/rclone-bitwarden.log \
        --log-level INFO
    log_info "rclone sync completed"

    # ---------------------------------------------------------------------------
    # Step 12: Success notification
    # ---------------------------------------------------------------------------
    local end_time
    end_time=$(date +%s)
    local duration=$(( end_time - SCRIPT_START ))

    # Count local backups across tiers
    local daily_count weekly_count monthly_count
    daily_count=$(ls -1 "${BACKUP_BASE}/daily/"*.json.gpg 2>/dev/null | wc -l)
    weekly_count=$(ls -1 "${BACKUP_BASE}/weekly/"*.json.gpg 2>/dev/null | wc -l)
    monthly_count=$(ls -1 "${BACKUP_BASE}/monthly/"*.json.gpg 2>/dev/null | wc -l)

    log_info "Backup run complete. Duration: ${duration}s"
    log_info "========================================================"

    notify_success "Backup completed: $(basename "$BACKUP_FILE") (tier: ${tier})
Synced to pCloud. Retained: ${daily_count} daily, ${weekly_count} weekly, ${monthly_count} monthly.
Duration: ${duration}s"
}

main "$@"
```

---

## 7. Operational Procedures

### 7.1 Manual Backup Run

To run a backup immediately outside the scheduled timer:

```bash
sudo systemctl start bw-backup.service
```

Monitor progress:

```bash
sudo journalctl -u bw-backup.service -f
tail -f /var/log/bitwarden-backup.log
```

Check exit status:

```bash
sudo systemctl status bw-backup.service
```

### 7.2 Verify the Latest Backup

**List all local backups:**

```bash
ls -lh /var/backups/bitwarden/daily/
ls -lh /var/backups/bitwarden/weekly/
ls -lh /var/backups/bitwarden/monthly/
```

**Verify a backup's checksum:**

```bash
# Set BACKUP_FILE to the .json.gpg file to verify
BACKUP_FILE="/var/backups/bitwarden/daily/bitwarden_2026-03-29_020015.json.gpg"
SIDECAR="${BACKUP_FILE%.json.gpg}.sha256"

EXPECTED=$(cat "$SIDECAR")
ACTUAL=$(sha256sum "$BACKUP_FILE" | awk '{print $1}')
echo "Expected: $EXPECTED"
echo "Actual:   $ACTUAL"
[[ "$EXPECTED" == "$ACTUAL" ]] && echo "MATCH" || echo "MISMATCH"
```

**Verify GPG structure without passphrase:**

```bash
gpg --list-packets /var/backups/bitwarden/daily/bitwarden_2026-03-29_020015.json.gpg
```

**Verify rclone remote contents:**

```bash
sudo -u bwbackup rclone ls bw-crypt:
```

### 7.3 Restore Procedure

**Step 1: Identify the backup to restore.**

```bash
ls -lh /var/backups/bitwarden/daily/
# Or from remote:
sudo -u bwbackup rclone ls bw-crypt:daily/
```

**Step 2: Retrieve from remote if local copy is unavailable.**

```bash
sudo -u bwbackup rclone copy \
    bw-crypt:daily/bitwarden_2026-03-29_020015.json.gpg \
    /tmp/restore/
```

**Step 3: Decrypt the backup.**

```bash
BACKUP_FILE="/var/backups/bitwarden/daily/bitwarden_2026-03-29_020015.json.gpg"
OUTPUT_FILE="/tmp/vault_restored.json"

gpg \
    --batch \
    --pinentry-mode loopback \
    --passphrase-file /etc/bw-backup/passphrase.key \
    --output "$OUTPUT_FILE" \
    --decrypt "$BACKUP_FILE"

echo "Restored to: $OUTPUT_FILE"
```

**Step 4: Inspect the vault JSON.**

```bash
python3 -c "
import json
with open('/tmp/vault_restored.json') as f:
    data = json.load(f)
items = data.get('items', [])
print(f'Total items: {len(items)}')
for item in items[:5]:
    print(f'  - {item.get(\"name\", \"(unnamed)\")} [{item.get(\"type\")}]')
"
```

**Step 5: Import into Bitwarden (if needed).**

Use the Bitwarden web vault or the CLI:

```bash
bw login
bw import bitwardenjson /tmp/vault_restored.json
```

Note: importing will create duplicate items if any items already exist in the vault. Consider creating a fresh vault or using the Bitwarden web interface's import tool for more control.

**Step 6: Securely delete the decrypted file after use.**

```bash
shred -u /tmp/vault_restored.json
```

### 7.4 Checking Timer Status

```bash
# View upcoming timer fire time
sudo systemctl list-timers bw-backup.timer

# View last service run result
sudo systemctl status bw-backup.service

# View last 50 log lines
sudo tail -50 /var/log/bitwarden-backup.log
```

### 7.5 Rotating the GPG Passphrase

1. Generate a new passphrase: `openssl rand -base64 48`
2. Decrypt all existing local backups with the old passphrase
3. Re-encrypt each with the new passphrase
4. Update `/etc/bw-backup/passphrase.key`
5. Sync to pCloud: `sudo -u bwbackup rclone sync /var/backups/bitwarden/ bw-crypt: --checksum`
6. Verify at least one backup decrypts successfully with the new passphrase

There is no incremental passphrase rotation; all local backups must be re-encrypted.

---

## 8. Known Limitations

### 8.1 No File Attachments

The `bw export --format json` command exports vault items (logins, notes, identities, cards) but does **not** export file attachments. Attachments are stored separately in Bitwarden's object storage and are not accessible via the standard export API. Any files attached to vault items will be lost if the only recovery mechanism is this backup.

Workaround: manually download attachments periodically via the Bitwarden web vault. Attachment backup automation is not in scope for this architecture.

### 8.2 No Bitwarden Send Items

Bitwarden Send (temporary file/text sharing) is not included in vault exports. Send items cannot be backed up via this mechanism.

### 8.3 No Vault Trash

Items in the Bitwarden trash are not included in the export. Permanently deleted items cannot be recovered from these backups.

### 8.4 Physical Access Risk (No LUKS)

Because LUKS full-disk encryption is not used (see §1.4), physical possession of the Raspberry Pi's storage medium grants access to:

- The script at `/home/bwbackup/scripts/bw-backup.sh`
- Encrypted vault backups at `/var/backups/bitwarden/`
- The GPG passphrase at `/etc/bw-backup/passphrase.key`
- The Bitwarden credentials at `/etc/bw-backup/env`
- The rclone config (including pCloud OAuth token) at `/home/bwbackup/.config/rclone/rclone.conf`

An attacker with the GPG passphrase and an encrypted backup can decrypt the vault. Physical security of the Raspberry Pi is a required control for this architecture.

### 8.5 pCloud Dependency

The off-site copy depends on pCloud's availability and the operator's pCloud account. If pCloud becomes unavailable or the account is suspended, the rclone sync step will fail and ntfy.sh will send a failure notification. Local backups continue to be created regardless. Operators requiring higher off-site availability should consider adding a secondary rclone remote as an additional sync target.

### 8.6 ntfy.sh Topic Security

The ntfy.sh topic name is effectively a shared secret. The free tier of ntfy.sh provides no authentication beyond knowledge of the topic name. Anyone who knows the topic name can read backup status notifications. Use a long random topic name (see §4.6.1) and treat it as a secret. Operators with stricter requirements should self-host ntfy.sh and enable authentication.

### 8.7 Single Point of Failure

This architecture runs on a single Raspberry Pi with no redundancy. If the Pi fails, backups stop until it is replaced. The pCloud copy provides a recovery path for the vault data, but the backup system itself must be manually rebuilt. Document the setup procedure and store it (along with the GPG passphrase) securely out-of-band (e.g., in a separate password manager or printed and stored in a safe).
