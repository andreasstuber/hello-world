# Self-Hosted Vaultwarden on GCP — Complete Setup Guide

A practical guide for setting up a personal, self-hosted [Vaultwarden](https://github.com/dani-garcia/vaultwarden) instance on Google Cloud Platform using a free-tier VM. This covers everything from infrastructure to client apps, password imports, Google SSO, automated backups, family sharing, and MFA migration.

---

## What Is Vaultwarden?

Vaultwarden is an open-source, self-hosted implementation of the Bitwarden password manager server. You run it on your own infrastructure — your passwords never leave your control. All official Bitwarden client apps (web, desktop, browser extensions, mobile) connect to it.

**Why self-host?**
- Full data ownership
- No subscription cost (beyond hosting)
- Works with the free Bitwarden clients on all platforms

---

## Prerequisites

- A Google account (personal Gmail is fine)
- A domain name you control (or a subdomain from a domain you manage)
- About 30–60 minutes

---

## Part 1 — GCP Infrastructure

### 1.1 Create a GCP Project

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (e.g., `my-vaultwarden`)
3. Enable billing (the e2-micro VM is free-tier eligible in `us-east1`, `us-west1`, or `us-central1`)

### 1.2 Create the VM

In **Compute Engine → VM instances → Create instance**:

| Setting | Value |
|---------|-------|
| Name | `vaultwarden` |
| Region | `us-east1` (or any free-tier region) |
| Machine type | `e2-micro` |
| Boot disk | Debian 12, 20 GB |
| Firewall | Allow HTTP and HTTPS traffic |

**Reserve a static IP:** VPC Network → IP addresses → Reserve static address → attach to the VM.

**Open ports in the firewall:** VPC Network → Firewall → Create rule:
- Name: `allow-web`
- Direction: Ingress
- Targets: All instances
- Source: `0.0.0.0/0`
- Protocols/ports: `tcp:80,443`

### 1.3 Point Your Domain

Create an **A record** in your DNS provider pointing your chosen subdomain to the VM's static IP.

> **Cloudflare users:** Set the record to **DNS-only (grey cloud)**, NOT proxied. Cloudflare's Flexible SSL proxying will cause an infinite redirect loop with Caddy's automatic HTTPS. You must bypass the Cloudflare proxy for this to work.

Allow 1–5 minutes for DNS to propagate before proceeding.

---

## Part 2 — Server Setup

SSH into your VM:

```bash
gcloud compute ssh vaultwarden --zone=us-east1-b --project=YOUR_PROJECT_ID
```

### 2.1 Install Docker and docker-compose

```bash
sudo apt-get update
sudo apt-get install -y docker.io

# Install docker-compose v1 binary (simpler than the v2 plugin for Debian 12)
sudo curl -L "https://github.com/docker/compose/releases/download/1.29.2/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

> **Note:** If you installed docker-compose this way, always use `docker-compose` (hyphenated), not `docker compose`.

### 2.2 Create the Vaultwarden Directory

```bash
sudo mkdir -p /opt/vaultwarden
cd /opt/vaultwarden
```

### 2.3 Generate an Admin Token

Vaultwarden has an admin panel that requires a token. Generate a secure one:

```bash
openssl rand -base64 48
```

Save this value — you'll need it in the config and to store it somewhere safe.

### 2.4 Create docker-compose.yml

```bash
sudo nano /opt/vaultwarden/docker-compose.yml
```

```yaml
version: '3'

services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    volumes:
      - vaultwarden-data:/data
    environment:
      - DOMAIN=https://your-vaultwarden.example.com
      - SIGNUPS_ALLOWED=false
      - INVITATIONS_ALLOWED=true
      - INVITATION_EXPIRATION_HOURS=720
      - ADMIN_TOKEN=YOUR_GENERATED_TOKEN_HERE
      - WEBSOCKET_ENABLED=true
      - LOG_LEVEL=warn
      - EMERGENCY_ACCESS_ALLOWED=true
      - SENDS_ALLOWED=true

volumes:
  vaultwarden-data:

networks:
  default:
    name: vaultwarden-net
```

> Set `SIGNUPS_ALLOWED=true` initially to create your account, then set it back to `false`.

### 2.5 Create the Caddyfile

[Caddy](https://caddyserver.com) handles HTTPS automatically — no certificate management needed.

```bash
sudo nano /opt/vaultwarden/Caddyfile
```

```
your-vaultwarden.example.com {
    encode gzip

    @notifications {
        path /notifications/hub
        not path /notifications/hub/negotiate
    }
    reverse_proxy @notifications vaultwarden:3012

    reverse_proxy vaultwarden:80
}
```

> The `@notifications` block routes WebSocket traffic for real-time vault sync. In Vaultwarden 1.36+, port 3012 is deprecated (integrated into port 80), but the config is harmless to keep.

### 2.6 Start Vaultwarden

```bash
cd /opt/vaultwarden
sudo docker-compose up -d vaultwarden
```

### 2.7 Start Caddy

Caddy is started separately (not in docker-compose) so it persists independently:

```bash
sudo docker run -d \
  --name caddy \
  --restart unless-stopped \
  --network vaultwarden-net \
  -p 80:80 -p 443:443 \
  -v /opt/vaultwarden/Caddyfile:/etc/caddy/Caddyfile \
  -v caddy_data:/data \
  -v caddy_config:/config \
  caddy:latest
```

### 2.8 Create Your Account

1. Navigate to `https://your-vaultwarden.example.com/#/register`
2. Create your account with a strong master password
3. Once created, disable signups:

```bash
cd /opt/vaultwarden
sudo sed -i 's/SIGNUPS_ALLOWED=true/SIGNUPS_ALLOWED=false/' docker-compose.yml
sudo docker-compose up -d vaultwarden
```

---

## Part 3 — Client Apps

All official Bitwarden apps work with Vaultwarden. When logging in, look for a **"Self-hosted"** or **"Custom server"** option and enter your URL (`https://your-vaultwarden.example.com`).

| Client | Download |
|--------|----------|
| Web vault | `https://your-vaultwarden.example.com` |
| Browser extension | Chrome Web Store / Edge Add-ons (Bitwarden) |
| Desktop | [bitwarden.com/download](https://bitwarden.com/download) |
| iOS / Android | App Store / Google Play (Bitwarden) |

**Browser extension:** Click the extension icon → Settings → Self-hosted environment → enter your URL.

**Desktop app:** On the login screen, click the region selector (usually shows "bitwarden.com") → Self-hosted → enter your URL.

**Mobile app:** On the login screen → Region → Self-hosted → Server URL → enter your URL.

---

## Part 4 — Importing Passwords

### 4.1 From Chrome / Edge

1. In Chrome/Edge: `Settings → Passwords → Export passwords` → save as CSV
2. In your web vault: `Tools → Import data` → select **Chrome (csv)** or **Microsoft Edge (csv)**
3. Upload the file

### 4.2 From KeePass

1. In KeePass: `File → Export → KeePass XML (2.x)` → save the file
2. In your web vault: `Tools → Import data` → select **KeePass 2 (xml)**
3. Upload the file

> Delete the exported files after importing — they contain all your passwords in plaintext.

---

## Part 5 — Google SSO (Optional)

This allows you to log in with your Google account instead of (or in addition to) your email/password.

> **Note:** After SSO login, Vaultwarden still prompts for your master password to decrypt the vault. This is by design — the SSO only authenticates your identity; your encryption key remains your master password.

### 5.1 Create a Google OAuth App

1. Go to [console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials
2. Create OAuth 2.0 Client ID → Web application
3. Add authorized redirect URI: `https://your-vaultwarden.example.com/identity/connect/oidc-signin`
4. Note the Client ID and Client Secret

> **Common mistake:** Using `/sso/callback` as the redirect URI. The correct path for Vaultwarden is `/identity/connect/oidc-signin`.

### 5.2 Add SSO Config to docker-compose.yml

```yaml
      - SSO_ENABLED=true
      - SSO_AUTHORITY=https://accounts.google.com
      - SSO_CLIENT_ID=YOUR_GOOGLE_CLIENT_ID
      - SSO_CLIENT_SECRET=YOUR_GOOGLE_CLIENT_SECRET
      - SSO_AUTHORIZE_EXTRA_PARAMS=access_type=offline&prompt=consent
```

Restart: `sudo docker-compose up -d vaultwarden`

---

## Part 6 — Automated Backups

Vaultwarden stores its data in a SQLite database inside a Docker named volume. The following PowerShell script copies it to your Windows machine daily.

Create `C:\Backups\Vaultwarden\backup.ps1`:

```powershell
$BackupDir = "C:\Backups\Vaultwarden"
$Date      = Get-Date -Format "yyyy-MM-dd"
$Dest      = "$BackupDir\vaultwarden-$Date.sqlite3"
$GcloudArgs = @("--zone=us-east1-b", "--project=YOUR_PROJECT_ID", "--account=YOUR_GMAIL", "--quiet")

# Stage the DB to /tmp (volume files need sudo)
& gcloud compute ssh vaultwarden @GcloudArgs `
    --command="sudo cp /var/lib/docker/volumes/vaultwarden_vaultwarden-data/_data/db.sqlite3 /tmp/vw_backup.sqlite3 && sudo chmod 644 /tmp/vw_backup.sqlite3"

if ($LASTEXITCODE -ne 0) { Write-Output "[$Date] Backup FAILED — stage error"; exit 1 }

& gcloud compute scp "vaultwarden:/tmp/vw_backup.sqlite3" $Dest @GcloudArgs

if ($LASTEXITCODE -eq 0) {
    Write-Output "[$Date] Backup saved to $Dest"
} else {
    Write-Output "[$Date] Backup FAILED — SCP error"; exit 1
}

& gcloud compute ssh vaultwarden @GcloudArgs --command="sudo rm /tmp/vw_backup.sqlite3"

# Retain last 30 days
Get-ChildItem "$BackupDir\vaultwarden-*.sqlite3" |
    Sort-Object LastWriteTime -Descending |
    Select-Object -Skip 30 |
    Remove-Item -Force
```

> **Why stage to /tmp?** Docker named volume files are owned by root. `gcloud compute scp` runs as your user and cannot read them directly. Staging to /tmp with `sudo chmod 644` makes the file readable for SCP.

### Schedule It (Windows Task Scheduler)

Run this once in an elevated PowerShell:

```powershell
$action  = New-ScheduledTaskAction -Execute "powershell.exe" `
             -Argument "-NonInteractive -File C:\Backups\Vaultwarden\backup.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At "2:00AM"
$settings = New-ScheduledTaskSettingsSet -StartWhenAvailable
Register-ScheduledTask -TaskName "VaultwardenBackup" `
  -Action $action -Trigger $trigger -Settings $settings `
  -RunLevel Highest -Force
```

---

## Part 7 — Family Sharing

### 7.1 Invite Family Members

With `SIGNUPS_ALLOWED=false`, invited users can still register via their invitation email link:

1. Set `INVITATIONS_ALLOWED=true` and `INVITATION_EXPIRATION_HOURS=720` in docker-compose.yml
2. Restart: `sudo docker-compose up -d vaultwarden`
3. In your web vault: click the grid icon → **New organisation** → create it
4. Inside the org → **Members** → **Invite member** → enter each email → role: Member

Each person receives an email with a registration link valid for 30 days. They create their own vault and join the organisation.

### 7.2 Share Passwords

Once members have accepted:
- Create **Collections** inside the org (e.g., "Shared Netflix", "Household Accounts")
- Move items from your personal vault into a collection
- Members can see only the collections you share with them

### 7.3 Emergency Access

This allows designated people to request access to your entire vault if you are unreachable for a set period.

In your vault: **Settings → Emergency Access → Add emergency contact**

- Enter their email address
- Set access type: **View** (read-only) or **Takeover** (can change master password)
- Set waiting period: e.g., 30 days

They receive an invitation. After accepting:
1. If something happens to you, they click **Request access**
2. You have the waiting period to deny it
3. If you don't respond, access is granted automatically

> Emergency Access and organisation sharing are independent — you need both if you want them to both see shared passwords AND access your personal vault in an emergency.

---

## Part 8 — Authy Migration (TOTP tokens)

If you currently use Authy for two-factor authentication and want to move those tokens into Vaultwarden's built-in TOTP authenticator, here's what you need to know.

### What Works

**Gradual migration (recommended):** When you visit a site that uses Authy for 2FA, go to that site's security settings, disable the existing TOTP, and re-enroll using Vaultwarden's built-in authenticator. Scan the new QR code, save the TOTP secret in your Vaultwarden item, and you're done for that site. Repeat over time as you naturally visit each service.

### What Doesn't Work

**Direct export from Authy:** Authy has no export feature. There is no official way to bulk-export your TOTP secrets.

**Android emulator approach (BlueStacks + old Authy APK):** A previously documented trick involved installing an old version of the Authy Android app on an Android emulator and extracting tokens via ADB. **This no longer works.** Authy now detects emulator environments and blocks login, even with older APK versions. Do not spend time on this path.

**Authy desktop app export:** The Authy desktop app stores tokens locally in an encrypted file. The encryption key is derived from your Authy backup password and is not directly extractable without reverse engineering — not practical for most users.

### Practical Advice

- Keep Authy running in parallel while you migrate gradually
- Prioritise high-value accounts (email, banking, cloud providers) first
- Each migration takes about 2 minutes per site
- 60 tokens migrated over a few months is realistic without any rush

---

## Part 9 — Troubleshooting

### ERR_TOO_MANY_REDIRECTS

**Cause:** Cloudflare proxy (orange cloud) is enabled. Cloudflare's Flexible SSL sends HTTP to Caddy, which redirects to HTTPS, causing an infinite loop.

**Fix:** In Cloudflare DNS, change the A record to **DNS-only (grey cloud)**.

### "Failed to fetch" on login

**Cause:** Stale browser DNS cache or Cloudflare cache after a DNS change.

**Fix:** Flush DNS (`ipconfig /flushdns` on Windows), clear browser cache, or use an incognito window.

### docker-compose changes not taking effect

`docker-compose start` restarts existing containers without re-reading environment variables. Always use:

```bash
sudo docker-compose up -d vaultwarden
```

This recreates the container with the new config.

### docker cp doesn't work for volume data

`docker cp` writes to the container's overlay filesystem, not to named volumes. To access named volume data, use the volume path directly:

```
/var/lib/docker/volumes/vaultwarden_vaultwarden-data/_data/
```

Files there require `sudo` to read.

---

## Part 10 — Admin Panel

Vaultwarden ships with an admin panel at `/admin`. Access it with your admin token.

Useful for:
- Viewing all users
- Resending invitation emails
- Disabling/deleting accounts
- Checking configuration

> Store your admin token somewhere safe and separate from your Vaultwarden vault (e.g., printed and stored physically, or in a secrets manager). If you lose vault access, the admin token is how you recover.

---

## Resources

- [Vaultwarden GitHub](https://github.com/dani-garcia/vaultwarden)
- [Vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki)
- [Bitwarden clients download](https://bitwarden.com/download)
- [Caddy documentation](https://caddyserver.com/docs)
- [GCP Free Tier details](https://cloud.google.com/free/docs/free-cloud-features)
