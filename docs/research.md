# Bitwarden Vault Backup — Research

*Generated: 2026-03-29*

---

## Table of Contents

1. [Bitwarden CLI — Export & Headless Authentication](#1-bitwarden-cli--export--headless-authentication)
2. [Environment Comparison: PowerShell vs WSL vs Raspberry Pi](#2-environment-comparison-powershell-vs-wsl-vs-raspberry-pi)
3. [Encryption: GPG and Alternatives](#3-encryption-gpg-and-alternatives)
4. [Storage, Rotation & Logging](#4-storage-rotation--logging)
5. [Recommended Architecture](#5-recommended-architecture)

---

## 1. Bitwarden CLI — Export & Headless Authentication

### Installation

```bash
# npm (cross-platform)
npm install -g @bitwarden/cli

# Chocolatey (Windows)
choco install bitwarden-cli

# Scoop (Windows)
scoop install bitwarden-cli

# Homebrew (macOS/Linux)
brew install bitwarden-cli

# Or download standalone binary from https://bitwarden.com/download/
bw --version
```

### Authentication Flow

Authentication is a two-step process: **login** (verifies identity with the server) + **unlock** (decrypts the local vault).

**Step 1 — Login with API key (required for headless/automated use)**

API keys bypass 2FA entirely — this is the only supported path for automated logins. Generate them in the Bitwarden web vault under Account Settings > Security > Keys.

```bash
export BW_CLIENTID="user.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export BW_CLIENTSECRET="xxxxxxxxxxxxxxxxxxxxxxxxxxxx"
bw login --apikey
```

**Step 2 — Unlock and capture session token**

```bash
export BW_PASSWORD="your_master_password"
export BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)
# --raw outputs only the session key with no surrounding text
```

`BW_SESSION` is an in-memory key. It does not persist to disk, and expires when `bw lock` is called or the process ends.

**Environment variables reference:**

| Variable | Purpose |
|---|---|
| `BW_SESSION` | Active session key (from `bw unlock --raw`) |
| `BW_CLIENTID` | API key client ID (for `bw login --apikey`) |
| `BW_CLIENTSECRET` | API key client secret |
| `BW_PASSWORD` | Master password (used with `--passwordenv`) |
| `BITWARDENCLI_APPDATA_DIR` | Override CLI config/data directory |

### Export Command

```bash
bw export [--output <filepath>] [--format <format>] [--password <password>] [--organizationid <orgId>]
```

**Formats:**

| Flag | Description |
|---|---|
| `json` | Unencrypted JSON (default). Full fidelity: logins, TOTP seeds, custom fields, secure notes, cards, identities. |
| `csv` | Unencrypted CSV. Reduced fields — no TOTP, no custom fields. |
| `encrypted_json` | Bitwarden-native encrypted JSON, protected by a separate export password. Re-importable. |

```bash
# Unencrypted JSON
bw export --format json --output ./vault-backup.json

# Bitwarden encrypted JSON (use --password to avoid interactive prompt)
bw export --format encrypted_json --password "$EXPORT_PASSWORD" --output ./vault-backup.json

# Pipe directly to an encryption tool
bw export --format json --output /dev/stdout | gpg --symmetric --cipher-algo AES256 > vault.json.gpg
```

**What is NOT included in any export:**
- File attachments (must be fetched individually via `bw get attachment`)
- Bitwarden Send items
- Trash / deleted items

### Full Headless Backup Script Pattern

```bash
#!/usr/bin/env bash
set -euo pipefail
umask 077   # new files are owner-read-only by default

# Credentials injected via environment or secrets manager — never hardcoded
BACKUP_DIR="/var/backups/bitwarden/daily"
TIMESTAMP=$(date +%Y-%m-%d_%H%M%S)
OUTFILE="${BACKUP_DIR}/bitwarden_${TIMESTAMP}.json.gpg"

# 1. Authenticate
bw login --apikey         # reads BW_CLIENTID + BW_CLIENTSECRET

# 2. Unlock
export BW_SESSION
BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)

# 3. Sync to get latest vault state
bw sync

# 4. Export and encrypt (piped to GPG — no plaintext touches disk)
bw export --format json --output /dev/stdout \
  | gpg --batch --pinentry-mode loopback \
        --passphrase-fd 3 \
        --symmetric --cipher-algo AES256 --s2k-digest-algo SHA512 \
        --output "$OUTFILE" \
    3< /etc/bitwarden-backup/passphrase.key

# 5. Checksum for integrity verification
sha256sum "$OUTFILE" > "${OUTFILE}.sha256"

# 6. Lock and logout
bw lock
bw logout

echo "[$(date -Iseconds)] [INFO] Backup written: $OUTFILE"
```

### Key Limitations

- **No attachment export via `bw export`** — attachments need per-item fetching.
- **`bw sync` is required** before exporting; the local cache may be stale.
- **API key login is mandatory** for fully automated scripts where 2FA is enabled.
- **Rate limiting** applies on Bitwarden cloud — self-hosting (Vaultwarden) is recommended for high-frequency testing.
- **Node.js 18+** required for the npm-installed CLI; the standalone binary has no dependency.

---

## 2. Environment Comparison: PowerShell vs WSL vs Raspberry Pi

### Side-by-Side Summary

| Dimension | PowerShell + Task Scheduler | WSL + Task Scheduler | Raspberry Pi |
|---|---|---|---|
| **Scheduling reliability** | High (if PC is on/awake) | Medium-High (Windows must be on) | **Highest** (always-on) |
| **Missed run handling** | `StartWhenAvailable` flag | Same via Task Scheduler | `systemd Persistent=true` |
| **Credential security (default)** | **Strong** (DPAPI/WCM) | Weak (plaintext file) | Medium (`chmod 600` env file) |
| **Credential security (hardened)** | **Strong** (SecretManagement + signing) | Medium (GPG + WCM interop) | Strong (LUKS + `systemd-creds`) |
| **Physical security risk** | Low (Windows login required) | Low (same as Windows) | High (SD card accessible) |
| **Setup complexity** | Medium | High (two environments) | Medium (basic) / High (FDE) |
| **Independence from main PC** | None | None | **Full** |
| **Linux tooling** | Limited | Full | Full |
| **Script portability** | Windows-only | Portable to Linux | Portable to other Linux |
| **Upfront cost** | $0 | $0 | $35–80 USD hardware |

---

### 2a. PowerShell + Windows Task Scheduler

**Scheduling:**

```powershell
$trigger  = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At "02:00AM"
$action   = New-ScheduledTaskAction -Execute "powershell.exe" `
              -Argument "-NonInteractive -ExecutionPolicy Bypass -File C:\Scripts\bw-backup.ps1"
$settings = New-ScheduledTaskSettingsSet -WakeToRun -RunOnlyIfNetworkAvailable -StartWhenAvailable
Register-ScheduledTask -TaskName "BitwardenBackup" -Trigger $trigger -Action $action -Settings $settings
```

Key flags:
- `WakeToRun` — wakes from sleep at scheduled time (requires BIOS RTC wake support).
- `StartWhenAvailable` — runs the missed task when the machine next comes online.
- `RunOnlyIfNetworkAvailable` — skips if offline.

**Credential storage via SecretManagement (recommended):**

```powershell
Install-Module Microsoft.PowerShell.SecretManagement
Install-Module Microsoft.PowerShell.SecretStore
Register-SecretVault -Name LocalStore -ModuleName Microsoft.PowerShell.SecretStore

# Store once:
Set-Secret -Name BwClientId     -Secret "user.xxxxxxxx"
Set-Secret -Name BwClientSecret -Secret "your-secret"
Set-Secret -Name BwPassword     -Secret "your-master-password"

# Retrieve at runtime in the backup script:
$env:BW_CLIENTID     = Get-Secret -Name BwClientId     -AsPlainText
$env:BW_CLIENTSECRET = Get-Secret -Name BwClientSecret -AsPlainText
$env:BW_PASSWORD     = Get-Secret -Name BwPassword     -AsPlainText
```

DPAPI ties these secrets to the Windows user account + machine — they cannot be read by other users or transferred to another machine without re-entering credentials.

**Script signing (integrity protection):**

```powershell
$cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject "CN=BwBackup"
Set-AuthenticodeSignature -FilePath C:\Scripts\bw-backup.ps1 -Certificate $cert
# Set execution policy to AllSigned to enforce this:
Set-ExecutionPolicy AllSigned -Scope LocalMachine
```

**Pros:** Native Windows; DPAPI is robust; Task Scheduler handles missed runs; rich Event Log auditing.
**Cons:** Machine must be on/awake; DPAPI breaks on profile deletion; execution policy is a deterrent, not a security boundary; WakeToRun unreliable on laptops with modern sleep states (S0ix/Modern Standby).

---

### 2b. WSL (Windows Subsystem for Linux) + Task Scheduler

**The critical constraint:** WSL does not run autonomously — it requires Windows to be on and a process to start it. Windows Task Scheduler must still be used as the scheduling backbone.

**Recommended pattern — Task Scheduler launches WSL:**

```powershell
$action = New-ScheduledTaskAction -Execute "wsl.exe" `
  -Argument "-d Ubuntu -- bash -c '/home/user/scripts/bw-backup.sh >> /home/user/logs/bw-backup.log 2>&1'"
$trigger = New-ScheduledTaskTrigger -Weekly -DaysOfWeek Sunday -At "02:00AM"
Register-ScheduledTask -TaskName "BitwardenWSLBackup" -Trigger $trigger -Action $action `
  -Settings (New-ScheduledTaskSettingsSet -StartWhenAvailable)
```

**Credential storage options (weakest to strongest):**

1. `chmod 600` env file in WSL — acceptable for low threat models.
2. Call Windows Credential Manager from WSL via PowerShell interop:
   ```bash
   BW_PASSWORD=$(powershell.exe -Command "Get-Secret -Name BwPassword -AsPlainText" | tr -d '\r')
   ```
3. GPG-encrypted file in WSL keyring.

**Writing output to Windows filesystem (for cross-environment access):**

```bash
OUTPUT="/mnt/c/Backups/bitwarden_$(date +%Y-%m-%d_%H%M%S).json.gpg"
bw export --format json --output /dev/stdout | gpg ... --output "$OUTPUT"
```

**Pros:** Full Linux toolchain (GPG, rclone, etc.); bash scripts portable to Raspberry Pi; can combine Windows scheduling with Linux execution.
**Cons:** Windows must still be on; credential storage weaker out-of-the-box; two environments to maintain; WSL2 requires Hyper-V; startup overhead.

---

### 2c. Raspberry Pi (Dedicated Linux Device)

**Scheduling via systemd timer (recommended over cron):**

```ini
# /etc/systemd/system/bw-backup.timer
[Unit]
Description=Weekly Bitwarden Backup

[Timer]
OnCalendar=Sun 02:00:00
Persistent=true          # run missed jobs on next boot
Unit=bw-backup.service

[Install]
WantedBy=timers.target
```

```ini
# /etc/systemd/system/bw-backup.service
[Unit]
Description=Bitwarden Vault Backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=bwbackup
EnvironmentFile=/etc/bw-backup/env
ExecStart=/home/bwbackup/scripts/bw-backup.sh
```

```bash
systemctl enable --now bw-backup.timer
systemctl list-timers bw-backup.timer
```

`After=network-online.target` ensures internet is available before the script runs.

**Credential storage — dedicated env file:**

```bash
# /etc/bw-backup/env
BW_CLIENTID=user.xxxxxxxx
BW_CLIENTSECRET=xxxxxxxx
BW_PASSWORD=your_master_password

chmod 600 /etc/bw-backup/env
chown bwbackup:bwbackup /etc/bw-backup/env
```

**For stronger security — systemd-creds (Pi OS Bookworm / systemd v250+):**

```bash
systemd-creds encrypt --name=bw-password - /etc/bw-backup/bw-password.cred
# In service unit: LoadCredential=bw-password:/etc/bw-backup/bw-password.cred
```

**Physical security — LUKS full-disk encryption:**

Without FDE, anyone with physical access can read the SD card directly. Enable LUKS on the root partition if the Pi is in a location that could be physically accessed by untrusted parties. Note: FDE requires a passphrase at boot (complicates headless reboot) or a network key server (Clevis/Tang).

**Hardening checklist:**

```bash
# Dedicated user with no sudo
useradd -r -s /usr/sbin/nologin bwbackup

# Automatic security updates
apt install unattended-upgrades

# Firewall — outbound 443 only needed
ufw default deny incoming
ufw default deny outgoing
ufw allow out 443/tcp
ufw enable

# Disable SSH password auth (key-based only)
# /etc/ssh/sshd_config: PasswordAuthentication no
```

**Pros:** Always-on; most reliable scheduling; `Persistent=true` handles missed runs; full Linux toolchain; independent of primary PC; low power (~3–5 W).
**Cons:** Hardware cost ($35–80); physical security risk without FDE; extra device to maintain; SD card wear (use high-endurance card or USB boot).

---

### Recommendation

| Use case | Best choice |
|---|---|
| Desktop PC always on, Windows-native preference | PowerShell + Task Scheduler |
| Want Linux tools but stay on Windows | WSL + Task Scheduler |
| Maximum reliability, independent of primary PC | **Raspberry Pi** |
| Highest security + reliability combined | Raspberry Pi + LUKS + systemd-creds + rclone to cloud |

---

## 3. Encryption: GPG and Alternatives

### Option Comparison

| Tool | Public-key | Symmetric | Script-friendly | Pre-installed | Cross-platform |
|---|---|---|---|---|---|
| **GPG** | Yes | Yes | Yes (with flags) | Usually | Yes |
| **age** | Yes | Yes | Yes (public-key) | No | Yes |
| **openssl enc** | No | Yes | Yes | Yes | Yes |
| **7-Zip** | No | Yes | Partial | No | Yes |

---

### 3a. GPG — Symmetric (Recommended for Personal Use)

Passphrase-based. No key infrastructure. Simplest to set up.

```bash
# Encrypt
gpg --batch \
    --pinentry-mode loopback \
    --passphrase-fd 3 \
    --symmetric \
    --cipher-algo AES256 \
    --s2k-digest-algo SHA512 \
    --s2k-count 65011712 \
    --output vault_backup.json.gpg \
    bitwarden_export.json \
    3< /root/.gpg_backup_passphrase

# Decrypt
gpg --batch --pinentry-mode loopback \
    --passphrase-fd 3 \
    --output bitwarden_export.json \
    --decrypt vault_backup.json.gpg \
    3< /root/.gpg_backup_passphrase
```

Key flags:
- `--batch` — disables all interactive prompts.
- `--pinentry-mode loopback` — required for GPG 2.1+ in non-interactive/cron contexts; routes passphrase handling internally instead of delegating to the gpg-agent/pinentry dialog.
- `--passphrase-fd 3` — reads passphrase from file descriptor 3 (not stdin, not command args — avoids exposure in `ps aux`).
- `--s2k-count 65011712` — maximum PBKDF2-style iteration count, slows brute-force.

**Passphrase file setup:**

```bash
install -m 600 -o root /dev/null /etc/bitwarden-backup/passphrase.key
echo "YourStrongPassphrase" > /etc/bitwarden-backup/passphrase.key
```

**Verify encrypted file without decrypting:**

```bash
gpg --list-packets vault_backup.json.gpg
# Shows cipher algo, S2K params — proves it's a valid GPG container
```

---

### 3b. GPG — Asymmetric (Recommended When Backup Destination Is Untrusted)

The backup script only needs the **public key** — it can encrypt without ever being able to decrypt. The private key lives offline or on a separate machine.

```bash
# Generate key pair (one-time setup)
gpg --batch --generate-key <<EOF
Key-Type: RSA
Key-Length: 4096
Name-Real: Bitwarden Backup
Name-Email: backup@local
Expire-Date: 0
%no-passphrase
%commit
EOF

# Export public key (put this on the backup machine)
gpg --export --armor backup@local > bitwarden_backup_public.asc

# Export private key (store OFFLINE — NOT on the backup machine)
gpg --export-secret-keys --armor backup@local > bitwarden_backup_private.asc

# Encrypt using only the public key (no prompts, no private key needed)
gpg --encrypt \
    --recipient backup@local \
    --trust-model always \
    --output vault_backup.json.gpg \
    bitwarden_export.json

# Decrypt (requires private key — run on the offline machine)
gpg --decrypt --output bitwarden_export.json vault_backup.json.gpg
```

`--trust-model always` bypasses the trust database warning in non-interactive scripts. Alternatively, locally sign the key once: `gpg --lsign-key backup@local`.

---

### 3c. GPG on Windows (Gpg4win)

- Install from https://www.gpg4win.org/ — includes GnuPG + Kleopatra GUI.
- Binary: `C:\Program Files (x86)\GnuPG\bin\gpg.exe`
- Pinentry uses a Windows GUI dialog by default — add `--pinentry-mode loopback --batch` to suppress it in scripts.
- GPG file format is fully cross-platform (OpenPGP RFC 4880) — files encrypted on Windows decrypt on Linux and vice versa.

```powershell
# PowerShell — non-interactive symmetric encryption
"YourPassphrase" | & "C:\Program Files (x86)\GnuPG\bin\gpg.exe" `
    --batch --pinentry-mode loopback --passphrase-fd 0 `
    --symmetric --cipher-algo AES256 `
    --output vault_backup.gpg bitwarden_export.json
```

---

### 3d. age (Modern GPG Alternative)

`age` is a minimal, modern encryption tool — no daemon, no keyring, no trust model.

```bash
# Install
curl -L https://github.com/FiloSottile/age/releases/latest/download/age-linux-amd64.tar.gz | tar xz

# Generate key pair
age-keygen -o backup_key.txt          # private key file
age-keygen -y backup_key.txt          # print public key (age1ql3z7hjy...)

# Encrypt with public key (no prompts)
age --recipient age1ql3z7hjy... \
    --output vault_backup.json.age \
    bitwarden_export.json

# Can also reuse existing SSH keys
age --recipient "$(cat ~/.ssh/id_ed25519.pub)" \
    --output vault_backup.json.age \
    bitwarden_export.json

# Decrypt
age --decrypt --identity backup_key.txt \
    --output bitwarden_export.json \
    vault_backup.json.age
```

**Advantages over GPG:** Single binary, no daemon, no keyring, public key is a short readable string, designed for scripting, modern crypto (X25519, ChaCha20-Poly1305).
**Disadvantages:** Not pre-installed, no signing, passphrase mode less mature for non-interactive use.

---

### 3e. openssl enc (Fallback — Pre-installed Everywhere)

```bash
# Encrypt
openssl enc -aes-256-cbc -pbkdf2 -iter 600000 \
    -in bitwarden_export.json \
    -out vault_backup.json.enc \
    -pass file:/etc/bitwarden-backup/passphrase.key

# Decrypt
openssl enc -d -aes-256-cbc -pbkdf2 -iter 600000 \
    -in vault_backup.json.enc \
    -out bitwarden_export.json \
    -pass file:/etc/bitwarden-backup/passphrase.key
```

`-pbkdf2` is mandatory — without it, openssl uses a weak MD5-based key derivation. `-iter 600000` meets NIST recommendations for PBKDF2-HMAC-SHA256.

---

### Key Management Best Practices

- Store the backup passphrase (or private key) in your primary password manager (Bitwarden itself is fine as a secondary location — you're backing it up, not solely relying on it).
- Additionally store it somewhere physically: printed paper in a sealed envelope, or a hardware security key.
- For asymmetric keys: the **private key must never reside on the backup machine**. Store it on an encrypted USB drive, air-gapped machine, or YubiKey (OpenPGP card mode).
- Rotate passphrases/keys annually or after suspected compromise.
- Set GNUPG permissions: `chmod 700 ~/.gnupg && chmod 600 ~/.gnupg/*`.
- Never store keys/passphrases in the same cloud bucket where encrypted backups are written.

---

## 4. Storage, Rotation & Logging

### File Naming

```bash
# Bash
TIMESTAMP=$(date +%Y-%m-%d_%H%M%S)
BACKUP_FILE="bitwarden_${TIMESTAMP}.json.gpg"
CHECKSUM_FILE="${BACKUP_FILE}.sha256"

# PowerShell
$timestamp  = Get-Date -Format "yyyy-MM-dd_HHmmss"
$backupFile = "bitwarden_$timestamp.json.gpg"
```

ISO 8601 timestamp ensures lexicographic sort = chronological sort — critical for rotation scripts. Always generate a `.sha256` sidecar immediately after encryption.

---

### Rotation: Simple Keep-Last-N

```bash
BACKUP_DIR="/var/backups/bitwarden/daily"
KEEP=7

# Name-sort (safer than mtime-sort)
ls -1 "${BACKUP_DIR}"/bitwarden_*.json.gpg | sort | head -n -"$KEEP" | xargs -r rm -f

# Prune orphaned checksums
for chk in "${BACKUP_DIR}"/*.sha256; do
  [ -f "${chk%.sha256}" ] || rm -f "$chk"
done
```

```powershell
$keep = 7
Get-ChildItem "C:\Backups\Bitwarden\daily" -Filter "bitwarden_*.json.gpg" |
  Sort-Object Name |
  Select-Object -SkipLast $keep |
  Remove-Item -Force
```

---

### Rotation: Grandfather-Father-Son (GFS)

GFS maintains three tiers covering a full year in 23 files:

| Tier | Run frequency | Files kept |
|---|---|---|
| Daily (Son) | Every day | Last 7 |
| Weekly (Father) | Every Sunday | Last 4 |
| Monthly (Grandfather) | 1st of month | Last 12 |

**Directory layout:**

```
backups/bitwarden/
├── daily/
├── weekly/
└── monthly/
```

**GFS implementation (Bash):**

```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/var/backups/bitwarden"
DAILY_DIR="${BACKUP_DIR}/daily"
WEEKLY_DIR="${BACKUP_DIR}/weekly"
MONTHLY_DIR="${BACKUP_DIR}/monthly"
KEEP_DAILY=7 KEEP_WEEKLY=4 KEEP_MONTHLY=12

mkdir -p "$DAILY_DIR" "$WEEKLY_DIR" "$MONTHLY_DIR"

TIMESTAMP=$(date +%Y-%m-%d_%H%M%S)
NEW_BACKUP="${DAILY_DIR}/bitwarden_${TIMESTAMP}.json.gpg"

# --- Create backup here (export + encrypt) ---
# ...

# Promote to weekly (Sunday = day 7 in %u)
[ "$(date +%u)" -eq 7 ] && cp "$NEW_BACKUP" "${WEEKLY_DIR}/$(basename "$NEW_BACKUP")"

# Promote to monthly (1st of month)
[ "$(date +%-d)" -eq 1 ] && cp "$NEW_BACKUP" "${MONTHLY_DIR}/$(basename "$NEW_BACKUP")"

# Prune each tier
prune_dir() {
  ls -1 "${1}"/bitwarden_*.json.gpg 2>/dev/null | sort | head -n -"$2" | xargs -r rm -f
}
prune_dir "$DAILY_DIR"   $KEEP_DAILY
prune_dir "$WEEKLY_DIR"  $KEEP_WEEKLY
prune_dir "$MONTHLY_DIR" $KEEP_MONTHLY
```

**GFS implementation (PowerShell):**

```powershell
$now        = Get-Date
$dayOfWeek  = [int]$now.DayOfWeek   # 0=Sun
$dayOfMonth = $now.Day
$backupDir  = "C:\Backups\Bitwarden"

foreach ($sub in "daily","weekly","monthly") {
  New-Item -ItemType Directory -Force -Path "$backupDir\$sub" | Out-Null
}

$timestamp = $now.ToString("yyyy-MM-dd_HHmmss")
$newBackup = "$backupDir\daily\bitwarden_$timestamp.json.gpg"
# ... create backup ...

if ($dayOfWeek -eq 0)  { Copy-Item $newBackup "$backupDir\weekly\"  }
if ($dayOfMonth -eq 1) { Copy-Item $newBackup "$backupDir\monthly\" }

function Prune-Dir($dir, $keep) {
  Get-ChildItem $dir -Filter "bitwarden_*.json.gpg" | Sort-Object Name |
    Select-Object -SkipLast $keep | Remove-Item -Force
}
Prune-Dir "$backupDir\daily"   7
Prune-Dir "$backupDir\weekly"  4
Prune-Dir "$backupDir\monthly" 12
```

---

### Local Storage Permissions

```bash
# Linux: dedicated user + strict permissions
useradd -r -s /usr/sbin/nologin bwbackup
chmod 700 /var/backups/bitwarden
chown -R bwbackup:bwbackup /var/backups/bitwarden
umask 077    # in script: new files auto-get 600

# Immediately after creating each file:
chmod 600 "$BACKUP_FILE"
```

```powershell
# Windows: remove inheritance, grant only the backup account
$acl = Get-Acl "C:\Backups\Bitwarden"
$acl.SetAccessRuleProtection($true, $false)
$rule = New-Object Security.AccessControl.FileSystemAccessRule(
    "BUILTIN\Administrators", "FullControl", "ContainerInherit,ObjectInherit", "None", "Allow")
$acl.AddAccessRule($rule)
Set-Acl -Path "C:\Backups\Bitwarden" -AclObject $acl
```

---

### Remote / Offsite Storage

**rclone (recommended — supports S3, B2, GDrive, OneDrive, and 40+ others):**

```bash
# Sync to Backblaze B2
rclone sync /var/backups/bitwarden/ b2:my-bucket/bitwarden/ \
  --checksum \
  --b2-hard-delete \
  --log-file /var/log/rclone-bitwarden.log \
  --log-level INFO

# Sync to S3
rclone sync /var/backups/bitwarden/ s3:my-bucket/bitwarden/ \
  --checksum \
  --transfers 4

# rclone crypt remote — additionally hides filenames from cloud provider
rclone sync /var/backups/bitwarden/ bw-crypt:
```

**rsync to self-hosted server:**

```bash
rsync -avz --delete \
  -e "ssh -i ~/.ssh/backup_key" \
  /var/backups/bitwarden/ \
  backupuser@offsite.example.com:/backups/bitwarden/
```

Restrict the SSH key to rsync-only in `~/.ssh/authorized_keys` on the remote:
```
command="rsync --server --delete ...",no-port-forwarding,no-X11-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA...
```

**S3 Object Lock — prevents ransomware from deleting cloud backups:**

```bash
aws s3api put-object-lock-configuration \
  --bucket my-backup-bucket \
  --object-lock-configuration '{"ObjectLockEnabled":"Enabled","Rule":{"DefaultRetention":{"Mode":"GOVERNANCE","Days":30}}}'
```

---

### Logging

**Structured log format (Bash):**

```bash
LOG_FILE="/var/log/bitwarden-backup.log"
log() { echo "[$(date -Iseconds)] [$1] ${*:2}" | tee -a "$LOG_FILE"; }

log INFO  "Backup started"
log INFO  "Encrypted backup created: $OUTFILE ($(stat -c%s "$OUTFILE") bytes)"
log ERROR "Encryption failed — exit code $?"
```

**logrotate configuration (`/etc/logrotate.d/bitwarden-backup`):**

```
/var/log/bitwarden-backup.log {
    weekly
    rotate 52
    compress
    delaycompress
    missingok
    notifempty
    create 640 bwbackup adm
}
```

**Dead man's switch with Healthchecks.io (detects silent cron failures):**

```bash
HEALTHCHECK_URL="https://hc-ping.com/your-uuid-here"
curl -fsS -m 10 "${HEALTHCHECK_URL}/start" > /dev/null 2>&1
# ... run backup ...
if [ $? -eq 0 ]; then
  curl -fsS -m 10 "$HEALTHCHECK_URL"        > /dev/null 2>&1
else
  curl -fsS -m 10 "${HEALTHCHECK_URL}/fail" > /dev/null 2>&1
fi
```

---

### Backup Verification

**1. Checksum (no passphrase needed):**

```bash
# Generate at backup time:
sha256sum "$OUTFILE" > "${OUTFILE}.sha256"

# Verify later:
sha256sum -c "${OUTFILE}.sha256"   # output: "bitwarden_....json.gpg: OK"
```

**2. GPG packet header inspection (no passphrase needed):**

```bash
gpg --list-packets vault_backup.json.gpg
# Valid output shows: ":symkey enc packet:" and ":encrypted data packet:"
# Truncated/corrupted files will show errors
```

**3. Full decryption spot-check (periodic — monthly recommended):**

```bash
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

gpg --batch --pinentry-mode loopback \
    --passphrase-file /etc/bitwarden-backup/passphrase.key \
    --output "$TEMP_DIR/check.json" \
    --decrypt "$OUTFILE"

# Validate JSON and count items
python3 -c "
import json
with open('$TEMP_DIR/check.json') as f:
    data = json.load(f)
items = data.get('items', data.get('Items', []))
assert len(items) > 0, 'No items in export'
print(f'PASS: {len(items)} items found')
"

# Securely erase plaintext temp file
shred -u "$TEMP_DIR/check.json" 2>/dev/null || rm -f "$TEMP_DIR/check.json"
```

**4. File size sanity check (cheapest first pass):**

```bash
MIN_SIZE=1024  # a valid encrypted vault export will always exceed 1 KB
SIZE=$(stat -c%s "$OUTFILE")
[ "$SIZE" -lt "$MIN_SIZE" ] && { log ERROR "File too small (${SIZE}B)"; exit 1; }
```

---

## 5. Recommended Architecture

### Tier 1: Simple (Windows desktop, personal use)

- **Environment:** PowerShell + Windows Task Scheduler
- **Auth:** Bitwarden API key stored in Windows SecretStore (DPAPI)
- **Export:** `bw export --format encrypted_json` (Bitwarden's own encryption)
- **Extra encryption:** GPG symmetric (AES-256) or 7-Zip AES-256 as a second layer
- **Rotation:** Keep last 7 daily + 4 weekly (simple PowerShell `Get-ChildItem | Sort | Skip`)
- **Offsite:** rclone to OneDrive or Google Drive
- **Monitoring:** Windows Event Log + Healthchecks.io

### Tier 2: Robust (always-on backup, Raspberry Pi)

- **Environment:** Raspberry Pi 4/5 + systemd timers (`Persistent=true`)
- **Auth:** Bitwarden API key in `systemd-creds` encrypted env file; dedicated `bwbackup` user
- **Export:** `bw export --format json` piped directly to GPG (no plaintext on disk)
- **Encryption:** GPG symmetric AES-256 with `--s2k-count 65011712`; passphrase in `chmod 600` file
- **Rotation:** GFS (7 daily / 4 weekly / 12 monthly) = 23 files covering 1 year
- **Offsite:** rclone to Backblaze B2 (cost ~$0.006/GB/month)
- **Integrity:** SHA-256 sidecar + `gpg --list-packets` check after each backup; monthly full decryption test
- **Monitoring:** Healthchecks.io dead man's switch + email on failure; logrotate (52 weekly rotations)

### Tier 3: Maximum Security (untrusted backup destination)

- **Encryption:** GPG asymmetric (public key only on backup machine; private key offline/YubiKey)
- **OR:** `age` with X25519 public key
- **Storage:** rclone crypt remote + S3 Object Lock (GOVERNANCE mode, 30-day retention)
- **Physical:** Raspberry Pi with LUKS FDE + Clevis/Tang network unlock
- **Credentials:** No master password stored — API key only; revocable from Bitwarden web vault

---

### Complete Script Skeleton (Linux/Raspberry Pi)

```bash
#!/usr/bin/env bash
# bitwarden-backup.sh
set -euo pipefail
umask 077

### Configuration (all secrets via environment / EnvironmentFile) ###
BACKUP_ROOT="/var/backups/bitwarden"
LOG_FILE="/var/log/bitwarden-backup.log"
GPG_PASSPHRASE_FILE="/etc/bitwarden-backup/passphrase.key"
KEEP_DAILY=7; KEEP_WEEKLY=4; KEEP_MONTHLY=12
HEALTHCHECK_URL="${HEALTHCHECK_URL:-}"  # optional

log() { echo "[$(date -Iseconds)] [$1] ${*:2}" | tee -a "$LOG_FILE"; }

TIMESTAMP=$(date +%Y-%m-%d_%H%M%S)
DAILY_DIR="${BACKUP_ROOT}/daily"
OUTFILE="${DAILY_DIR}/bitwarden_${TIMESTAMP}.json.gpg"

mkdir -p "${BACKUP_ROOT}/daily" "${BACKUP_ROOT}/weekly" "${BACKUP_ROOT}/monthly"

[ -n "$HEALTHCHECK_URL" ] && curl -fsS -m 10 "${HEALTHCHECK_URL}/start" > /dev/null 2>&1 || true

### 1. Authenticate ###
log INFO "Logging in via API key"
bw login --apikey
export BW_SESSION
BW_SESSION=$(bw unlock --passwordenv BW_PASSWORD --raw)

### 2. Sync ###
log INFO "Syncing vault"
bw sync

### 3. Export + Encrypt (no plaintext on disk) ###
log INFO "Exporting and encrypting to $OUTFILE"
bw export --format json --output /dev/stdout \
  | gpg --batch --pinentry-mode loopback \
        --passphrase-fd 3 \
        --symmetric --cipher-algo AES256 \
        --s2k-digest-algo SHA512 --s2k-count 65011712 \
        --output "$OUTFILE" \
    3< "$GPG_PASSPHRASE_FILE"

### 4. Checksum ###
sha256sum "$OUTFILE" > "${OUTFILE}.sha256"
log INFO "Backup created: $OUTFILE ($(stat -c%s "$OUTFILE") bytes)"

### 5. Verify (fast — no decryption) ###
gpg --list-packets "$OUTFILE" > /dev/null 2>&1 \
  && log INFO "GPG packet check: PASS" \
  || { log ERROR "GPG packet check: FAIL"; exit 1; }

### 6. Lock + logout ###
bw lock; bw logout

### 7. GFS rotation ###
DOW=$(date +%u); DOM=$(date +%-d)
[ "$DOW" -eq 7 ] && cp "$OUTFILE" "${BACKUP_ROOT}/weekly/"
[ "$DOM" -eq 1 ] && cp "$OUTFILE" "${BACKUP_ROOT}/monthly/"

prune() { ls -1 "${1}"/bitwarden_*.json.gpg 2>/dev/null | sort | head -n -"$2" | xargs -r rm -f; }
prune "${BACKUP_ROOT}/daily"   $KEEP_DAILY
prune "${BACKUP_ROOT}/weekly"  $KEEP_WEEKLY
prune "${BACKUP_ROOT}/monthly" $KEEP_MONTHLY

### 8. Sync to cloud ###
rclone sync "${BACKUP_ROOT}/" b2:my-bucket/bitwarden/ --checksum \
  --log-file /var/log/rclone-bitwarden.log --log-level INFO

log INFO "Backup job complete"
[ -n "$HEALTHCHECK_URL" ] && curl -fsS -m 10 "$HEALTHCHECK_URL" > /dev/null 2>&1 || true
```

---

*Sources: Bitwarden CLI documentation, GnuPG documentation, rclone documentation, age-encryption.org, systemd documentation.*
