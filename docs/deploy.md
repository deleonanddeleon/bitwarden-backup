# Tier 2 Raspberry Pi Bitwarden Backup — Deployment Guide

This guide walks through deploying the Tier 2 Raspberry Pi Bitwarden Backup system from scratch. Follow each phase in order. Commands are written to be copy-pasted directly into a terminal.

**System overview:**
- Raspberry Pi 3B+, 4, or 5 running Raspberry Pi OS Bookworm Lite (64-bit)
- Bitwarden CLI (`bw`) for vault export
- GPG symmetric encryption (AES-256)
- GFS rotation: 7 daily / 4 weekly / 12 monthly
- rclone with pCloud crypt remote for offsite sync
- ntfy.sh push notifications
- systemd timer (daily at 02:00)

---

## Phase 1 — Initial Pi Setup

### Step 1. Flash the OS

1. Download and open [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Choose device: your Pi model (3B+, 4, or 5).
3. Choose OS: **Raspberry Pi OS Lite (64-bit)** — under "Raspberry Pi OS (other)".
4. Choose storage: your SD card or USB drive.
5. Click the gear icon (or "Edit Settings") before writing:
   - Set **hostname** (e.g., `pi-backup`).
   - Set **username** and a strong password.
   - Paste your **SSH public key** under "Enable SSH / Allow public-key authentication only".
   - Set your **Wi-Fi credentials** if not using Ethernet.
   - Set **locale and timezone**.
6. Write the image.

### Step 2. First Boot and SSH In

Insert the SD card, power on the Pi, and wait ~60 seconds for first boot.

```bash
ssh <your-username>@<pi-hostname>.local
# or use the IP address if mDNS is not available
```

### Step 3. Update the System

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 4. Set Timezone

> **Customize:** Replace `<your/zone>` with your actual timezone (e.g., `America/New_York`, `Europe/London`, `Australia/Sydney`).
> Run `timedatectl list-timezones` to see all options.

```bash
sudo timedatectl set-timezone <your/zone>
timedatectl   # confirm it is correct
```

### Step 5. Enable Automatic Security Updates

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

When prompted, select **Yes** to enable automatic updates.

---

## Phase 2 — Create Dedicated Service User

All backup operations run as a non-login system user (`bwbackup`) to limit privilege exposure.

```bash
sudo useradd -r -m -s /usr/sbin/nologin bwbackup
sudo mkdir -p /home/bwbackup/scripts /home/bwbackup/.config
sudo chown -R bwbackup:bwbackup /home/bwbackup
```

Verify:

```bash
id bwbackup
ls -la /home/bwbackup/
```

---

## Phase 3 — Install Dependencies

### Step 1. Install Node.js 20.x

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version   # should print v20.x.x or later
```

### Step 2. Install Bitwarden CLI

```bash
sudo npm install -g @bitwarden/cli
bw --version   # confirm installation
```

### Step 3. Install rclone

```bash
curl https://rclone.org/install.sh | sudo bash
rclone --version   # confirm installation
```

### Step 4. Install Remaining Tools

```bash
sudo apt install -y gpg curl python3 ufw logrotate
gpg --version   # confirm gpg is present
```

### Step 5. Final Verification

```bash
bw --version
rclone --version
gpg --version
node --version
```

All four commands should print version strings without errors.

---

## Phase 4 — Configure Firewall

Lock down outbound traffic to only what the backup process needs (HTTPS and DNS), plus inbound SSH.

> **Note:** If you run SSH on a non-standard port, replace `22` with your port number.

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow out 443/tcp      # HTTPS (Bitwarden API, pCloud, ntfy.sh)
sudo ufw allow out 53/udp       # DNS resolution
sudo ufw allow in 22/tcp        # SSH — adjust port if needed
sudo ufw enable
```

Verify:

```bash
sudo ufw status verbose
```

> **Warning:** Do not close your current SSH session until you have confirmed SSH is still accessible from a second terminal window.

---

## Phase 5 — Bitwarden API Key

The backup script uses Bitwarden's API key login (non-interactive) rather than interactive username/password login.

### Step 1. Generate Your API Key

1. Log into your Bitwarden web vault at [vault.bitwarden.com](https://vault.bitwarden.com).
2. Navigate to: **Account Settings** → **Security** → **Keys** → **API Key** → **View API Key**.
3. Note down:
   - `client_id` — begins with `user.` followed by a UUID
   - `client_secret` — a short alphanumeric string
   - Your master password (you already know this)

### Step 2. Create the Secrets Directory and Environment File

```bash
sudo mkdir -p /etc/bw-backup
sudo install -m 600 -o bwbackup -g bwbackup /dev/null /etc/bw-backup/env
```

### Step 3. Write Credentials Into the Environment File

> **Customize:** Replace all placeholder values with your actual credentials and chosen ntfy topic name.

```bash
sudo tee /etc/bw-backup/env <<'EOF'
BW_CLIENTID=user.xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
BW_CLIENTSECRET=xxxxxxxxxxxxxxxxxxxxxxxxxxxx
BW_PASSWORD=your_master_password
NTFY_TOPIC=your-chosen-topic-name
EOF
```

### Step 4. Lock Down Permissions

```bash
sudo chmod 600 /etc/bw-backup/env
sudo chown bwbackup:bwbackup /etc/bw-backup/env
```

Verify:

```bash
sudo ls -la /etc/bw-backup/env
# Should show: -rw------- 1 bwbackup bwbackup
```

---

## Phase 6 — GPG Passphrase

The backup files are encrypted with GPG symmetric encryption using AES-256. The passphrase is stored in a key file readable only by the `bwbackup` user.

### Step 1. Create the Config Directory and Key File

```bash
sudo mkdir -p /etc/bitwarden-backup
sudo chown bwbackup:bwbackup /etc/bitwarden-backup
sudo install -m 600 -o bwbackup -g bwbackup /dev/null /etc/bitwarden-backup/passphrase.key
```

### Step 2. Generate and Write a Strong Passphrase

Generate a strong random passphrase (at least 32 characters). One approach:

```bash
# Generate a random passphrase using openssl:
openssl rand -base64 32
```

Copy the output, then write it to the key file:

> **Customize:** Replace `YourStrongRandomPassphrase` with the generated value.

```bash
sudo -u bwbackup bash -c 'echo "YourStrongRandomPassphrase" > /etc/bitwarden-backup/passphrase.key'
sudo chmod 600 /etc/bitwarden-backup/passphrase.key
```

Verify:

```bash
sudo ls -la /etc/bitwarden-backup/passphrase.key
# Should show: -rw------- 1 bwbackup bwbackup
```

> **Important:** Store this passphrase in at least two places:
> 1. In your Bitwarden vault itself (under a secure note), and
> 2. On a physical piece of paper stored somewhere safe.
>
> Without this passphrase, encrypted backups cannot be decrypted.

---

## Phase 7 — Create Backup Storage Directories

```bash
sudo mkdir -p /var/backups/bitwarden/{daily,weekly,monthly}
sudo chown -R bwbackup:bwbackup /var/backups/bitwarden
sudo chmod 700 /var/backups/bitwarden
```

Verify:

```bash
sudo ls -la /var/backups/bitwarden/
# Should show daily/, weekly/, monthly/ owned by bwbackup
```

---

## Phase 8 — Configure rclone for pCloud with Encryption

Backups are synced to pCloud through an rclone `crypt` remote, which encrypts filenames and content before upload. The underlying pCloud remote is configured first, then the crypt layer sits on top of it.

### Step 1. Create the rclone Config Directory

```bash
sudo mkdir -p /home/bwbackup/.config/rclone
sudo chown -R bwbackup:bwbackup /home/bwbackup/.config
```

### Step 2. Configure the pCloud Remote

The pCloud OAuth flow requires a browser. If you are running the Pi headlessly, run the `rclone authorize` step on your desktop machine and paste the resulting token back into the Pi.

**On your desktop machine** (where rclone is also installed):

```bash
rclone authorize pcloud
```

A browser window will open. Authorize rclone to access your pCloud account. Copy the token JSON string that is printed in the terminal.

**On the Pi**, run the interactive config as the `bwbackup` user:

```bash
sudo -u bwbackup rclone config
```

In the interactive prompt:

1. Press `n` for new remote.
2. Name: `pcloud`
3. Type: choose `pcloud` (type the number shown for pCloud).
4. For client_id and client_secret: leave blank (press Enter) to use the built-in app credentials.
5. When asked about the OAuth token, paste the token JSON you obtained from your desktop.
6. Press `q` to quit config.

### Step 3. Configure the Crypt Remote

> **Customize:** Choose a strong password and a separate salt value. These are different from the GPG passphrase — they are used by rclone to encrypt filenames and file contents in pCloud. Store these values in your Bitwarden vault.

Still in `rclone config` (or re-enter it):

```bash
sudo -u bwbackup rclone config
```

1. Press `n` for new remote.
2. Name: `bw-crypt`
3. Type: choose `crypt`.
4. Remote: `pcloud:bitwarden-backup`
5. Filename encryption: `standard`
6. Directory name encryption: `true`
7. Enter a password when prompted (this encrypts file contents and names).
8. Enter a password salt when prompted (adds additional entropy).
9. Press `q` to quit.

### Step 4. Alternatively — Write the Config File Directly

If you prefer to write the config file without the interactive wizard, place the following in `/home/bwbackup/.config/rclone/rclone.conf`:

> **Customize:** Replace all placeholder values. Use `rclone obscure <yourpassword>` on the Pi to generate the obscured password and salt values.

```bash
# Generate obscured values first:
sudo -u bwbackup rclone obscure YourRemotePassword
sudo -u bwbackup rclone obscure YourRemoteSalt
```

Then write the config:

```
[pcloud]
type = pcloud
token = {"access_token":"...","token_type":"bearer","expiry":"0001-01-01T00:00:00Z"}

[bw-crypt]
type = crypt
remote = pcloud:bitwarden-backup
filename_encryption = standard
directory_name_encryption = true
password = <rclone-obscured-password>
password2 = <rclone-obscured-salt>
```

Set permissions on the config file:

```bash
sudo chmod 600 /home/bwbackup/.config/rclone/rclone.conf
sudo chown bwbackup:bwbackup /home/bwbackup/.config/rclone/rclone.conf
```

### Step 5. Test the Remote

```bash
sudo -u bwbackup rclone lsd bw-crypt:
```

This should return an empty listing (no error) on first run. If you see an authentication error, recheck the token in the pcloud remote config.

---

## Phase 9 — Install the Backup Script

The backup script is the core of the system. It handles: Bitwarden login, vault sync, JSON export, GPG encryption, SHA-256 checksum, verification, lock/logout, GFS rotation, rclone upload, and ntfy.sh notification.

### Step 1. Create the Script File

```bash
sudo install -m 750 -o bwbackup -g bwbackup /dev/null /home/bwbackup/scripts/bw-backup.sh
```

### Step 2. Write the Script Content

Write the script content from `spec.md` into `/home/bwbackup/scripts/bw-backup.sh`. The script must be owned by `bwbackup` with mode `750`.

Verify permissions after writing:

```bash
ls -la /home/bwbackup/scripts/bw-backup.sh
# Should show: -rwxr-x--- 1 bwbackup bwbackup
```

### Step 3. Smoke Test the Script Syntax

```bash
sudo -u bwbackup bash -n /home/bwbackup/scripts/bw-backup.sh
# No output means no syntax errors
```

---

## Phase 10 — ntfy.sh Push Notifications

ntfy.sh provides free, serverless push notifications. The backup script posts a message to a topic URL after each run (success or failure).

### Step 1. Choose a Topic Name

Pick a topic name that is hard to guess — ntfy.sh topics are public by default. A good approach:

```bash
# Generate a random suffix:
cat /proc/sys/kernel/random/uuid
```

Use a name like `bw-backup-a1b2c3d4` where the suffix is the first 8 characters of the UUID.

### Step 2. Install the Mobile App

- **Android:** Available on [F-Droid](https://f-droid.org/en/packages/io.heckel.ntfy/) and [Google Play](https://play.google.com/store/apps/details?id=io.heckel.ntfy).
- **iOS:** Available on the [App Store](https://apps.apple.com/us/app/ntfy/id1625396347).

Open the app, tap the `+` button, and subscribe to your chosen topic name.

### Step 3. Set the Topic in the Environment File

> **Customize:** Edit `/etc/bw-backup/env` and update `NTFY_TOPIC` to your chosen topic name.

```bash
sudo nano /etc/bw-backup/env
# Update: NTFY_TOPIC=bw-backup-a1b2c3d4
```

### Step 4. Test a Notification

> **Customize:** Replace `your-topic-name` with your actual topic.

```bash
curl -d "test notification from pi-backup" https://ntfy.sh/your-topic-name
```

You should receive a push notification on your phone within a few seconds.

---

## Phase 11 — Log Rotation

Configure logrotate to keep 52 weeks of compressed logs and prevent the log file from growing indefinitely.

```bash
sudo tee /etc/logrotate.d/bitwarden-backup <<'EOF'
/var/log/bitwarden-backup.log {
    weekly
    rotate 52
    compress
    delaycompress
    missingok
    notifempty
    create 640 bwbackup adm
}
EOF
```

Create the log file with correct ownership so the service can write to it from first run:

```bash
sudo touch /var/log/bitwarden-backup.log
sudo chown bwbackup:adm /var/log/bitwarden-backup.log
sudo chmod 640 /var/log/bitwarden-backup.log
```

Test logrotate config:

```bash
sudo logrotate --debug /etc/logrotate.d/bitwarden-backup
```

---

## Phase 12 — systemd Timer and Service

### Step 1. Create the Service Unit

```bash
sudo tee /etc/systemd/system/bw-backup.service <<'EOF'
[Unit]
Description=Bitwarden Vault Backup
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot
User=bwbackup
EnvironmentFile=/etc/bw-backup/env
ExecStart=/home/bwbackup/scripts/bw-backup.sh
StandardOutput=append:/var/log/bitwarden-backup.log
StandardError=append:/var/log/bitwarden-backup.log
EOF
```

### Step 2. Create the Timer Unit

The timer runs the backup daily at 02:00. `Persistent=true` means if the Pi was off at 02:00, the backup runs as soon as it boots.

```bash
sudo tee /etc/systemd/system/bw-backup.timer <<'EOF'
[Unit]
Description=Daily Bitwarden Backup Timer

[Timer]
OnCalendar=*-*-* 02:00:00
Persistent=true
Unit=bw-backup.service

[Install]
WantedBy=timers.target
EOF
```

### Step 3. Enable and Start the Timer

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bw-backup.timer
```

Verify the timer is active and shows the next trigger time:

```bash
sudo systemctl list-timers bw-backup.timer
```

---

## Phase 13 — First Manual Run and Verification

Run the backup manually to confirm the full pipeline works end to end before relying on the timer.

### Step 1. Run the Service

```bash
sudo systemctl start bw-backup.service
```

### Step 2. Watch the Logs in Real Time

```bash
sudo journalctl -u bw-backup.service -f
```

Wait for the service to complete. Press `Ctrl+C` to exit the log follower once done.

### Step 3. Confirm Backup Files Were Created

```bash
ls -la /var/backups/bitwarden/daily/
```

You should see at minimum one `.gpg` file and one `.sha256` file.

### Step 4. Verify the Checksum

```bash
sudo -u bwbackup sh -c 'cd /var/backups/bitwarden/daily && sha256sum -c *.sha256'
```

Expected output: `filename.json.gpg: OK`

### Step 5. Verify GPG Encryption

```bash
sudo -u bwbackup gpg --list-packets /var/backups/bitwarden/daily/*.gpg 2>&1 | head -20
```

You should see output referencing AES encryption. You should NOT see any plaintext vault data.

### Step 6. Confirm the Full Log

```bash
cat /var/log/bitwarden-backup.log
```

### Step 7. Confirm rclone Synced to pCloud

```bash
sudo -u bwbackup rclone ls bw-crypt:
```

You should see the encrypted backup file listed in the crypt remote.

### Step 8. Confirm ntfy Notification Was Received

Check your phone for the push notification from the backup run.

---

## Phase 14 — SSH Hardening

> **Warning:** Complete this step last. Confirm your SSH key login works in a separate terminal window before disabling password authentication. If you lock yourself out, you will need physical console access to the Pi.

### Step 1. Verify SSH Key Login Works

Open a second terminal and confirm you can SSH in with your key:

```bash
ssh <your-username>@<pi-hostname>.local
```

If this succeeds, proceed.

### Step 2. Disable Password Authentication

```bash
sudo nano /etc/ssh/sshd_config
```

Ensure the following lines are set (add or update them):

```
PasswordAuthentication no
PubkeyAuthentication yes
ChallengeResponseAuthentication no
```

### Step 3. Restart SSH

```bash
sudo systemctl restart sshd
```

### Step 4. Test From a New Terminal

Open a third terminal and confirm SSH key login still works:

```bash
ssh <your-username>@<pi-hostname>.local
```

Also confirm that password login is now rejected (try adding `-o PubkeyAuthentication=no` to force it):

```bash
ssh -o PubkeyAuthentication=no <your-username>@<pi-hostname>.local
# Should be rejected: "Permission denied (publickey)"
```

---

## Verification Checklist

Use this checklist to confirm the deployment is complete and functioning correctly.

### System Setup
- [ ] Raspberry Pi OS Bookworm Lite (64-bit) is running
- [ ] `sudo apt update && sudo apt upgrade -y` has been run
- [ ] Correct timezone is set (`timedatectl` confirms)
- [ ] Unattended upgrades are enabled

### Users and Permissions
- [ ] `bwbackup` system user exists (`id bwbackup`)
- [ ] `/home/bwbackup/scripts/` exists and is owned by `bwbackup`
- [ ] `/etc/bw-backup/env` exists, is owned by `bwbackup`, mode `600`
- [ ] `/etc/bitwarden-backup/passphrase.key` exists, is owned by `bwbackup`, mode `600`
- [ ] `/var/backups/bitwarden/{daily,weekly,monthly}` exist, owned by `bwbackup`, mode `700`

### Dependencies
- [ ] `bw --version` prints a version string
- [ ] `rclone --version` prints a version string
- [ ] `gpg --version` prints a version string
- [ ] `node --version` prints v20.x or later

### Firewall
- [ ] `sudo ufw status verbose` shows: deny incoming, deny outgoing, allow 443/tcp out, allow 53/udp out, allow 22/tcp in

### Backup Script
- [ ] `/home/bwbackup/scripts/bw-backup.sh` exists, mode `750`, owned by `bwbackup`
- [ ] `bash -n /home/bwbackup/scripts/bw-backup.sh` reports no syntax errors

### rclone / pCloud
- [ ] `sudo -u bwbackup rclone lsd bw-crypt:` returns without error
- [ ] `/home/bwbackup/.config/rclone/rclone.conf` mode `600`, owned by `bwbackup`

### ntfy.sh
- [ ] Topic name is set in `/etc/bw-backup/env`
- [ ] Test `curl` to the topic delivers a notification to your phone

### logrotate
- [ ] `/etc/logrotate.d/bitwarden-backup` exists
- [ ] `/var/log/bitwarden-backup.log` exists, owned by `bwbackup:adm`, mode `640`
- [ ] `sudo logrotate --debug /etc/logrotate.d/bitwarden-backup` reports no errors

### systemd Timer
- [ ] `sudo systemctl is-enabled bw-backup.timer` returns `enabled`
- [ ] `sudo systemctl list-timers bw-backup.timer` shows a next trigger time
- [ ] `sudo systemctl is-active bw-backup.timer` returns `active`

### First Run
- [ ] `sudo systemctl start bw-backup.service` exits successfully
- [ ] `sudo journalctl -u bw-backup.service` shows no errors
- [ ] At least one `.gpg` and one `.sha256` file exist in `/var/backups/bitwarden/daily/`
- [ ] `sha256sum -c` on the `.sha256` file returns `OK`
- [ ] `gpg --list-packets` on the `.gpg` file shows AES encryption (no plaintext)
- [ ] `sudo -u bwbackup rclone ls bw-crypt:` shows the encrypted backup file
- [ ] Push notification was received on your phone

### SSH Hardening
- [ ] SSH key login confirmed working
- [ ] `PasswordAuthentication no` is set in `/etc/ssh/sshd_config`
- [ ] `sudo systemctl restart sshd` completed without error
- [ ] Password login is rejected (`Permission denied (publickey)`)

### Passphrases and Secrets Stored Securely
- [ ] GPG passphrase is saved in Bitwarden vault
- [ ] GPG passphrase has a physical backup copy stored securely
- [ ] rclone crypt password and salt are saved in Bitwarden vault
- [ ] Bitwarden API client_id and client_secret are noted (they can be regenerated from the web vault if lost)
