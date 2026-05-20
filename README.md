# Nextcloud Talk Server — Installation Runbook

A fully self-hosted Nextcloud 33 stack with Talk, Music, a Telegram bridge with native media relay, fail2ban hardening, a TURN server for video calls, and a Talk bot that auto-archives paywalled links.

Designed for a Debian VM behind NGINX Proxy Manager and Cloudflare, but adaptable to any reverse-proxy setup.

---

## Configuration Variables

Replace these placeholders throughout this guide before running any commands.

| Variable | Description | Example |
|---|---|---|
| `YOUR_DOMAIN` | Nextcloud public domain | `next.example.com` |
| `YOUR_TURN_DOMAIN` | TURN server domain (can be same host) | `turn.example.com` |
| `YOUR_PUBLIC_IP` | Server public IP | `203.0.113.10` |
| `YOUR_LAN_IP` | Server LAN IP | `192.168.1.50` |
| `YOUR_NC_URL` | Full Nextcloud URL | `https://next.example.com` |
| `YOUR_DB_PASSWORD` | PostgreSQL password for ncuser | — |
| `YOUR_REDIS_PASSWORD` | Redis requirepass value | — |
| `YOUR_ADMIN_PASSWORD` | Nextcloud admin account password | — |
| `YOUR_TURN_SECRET` | coturn shared secret (`openssl rand -hex 32`) | — |
| `YOUR_TELEGRAM_BOT_TOKEN` | Bot token from @BotFather | `123456:ABC...` |
| `YOUR_TELEGRAM_GROUP_ID` | Telegram group/channel ID (negative number) | `-1001234567890` |
| `YOUR_TALK_TOKEN` | Nextcloud Talk room token | `abc12xyz` |
| `YOUR_SMB_HOST` | NAS/SMB server IP | `192.168.1.10` |
| `YOUR_SMB_SHARE` | Share name | `data` |
| `YOUR_SMB_PATH` | Path inside the share | `media/music` |
| `YOUR_SMB_USER` | SMB username | — |
| `YOUR_SMB_PASSWORD` | SMB password | — |
| `YOUR_LASTFM_API_KEY` | Last.fm API key | — |
| `YOUR_LASTFM_SECRET` | Last.fm shared secret | — |
| `YOUR_SMTP_HOST` | SMTP server hostname | `smtp.example.com` |
| `YOUR_SMTP_PORT` | SMTP port | `587` |
| `YOUR_SMTP_USER` | SMTP username/address | — |
| `YOUR_SMTP_PASSWORD` | SMTP password or app token | — |
| `YOUR_SMTP_FROM` | From address | `noreply@example.com` |
| `YOUR_BOT_SECRET` | Archive bot HMAC secret (`openssl rand -hex 32`) | — |
| `YOUR_ADMIN_APP_PASS` | Admin app password for archive bot message deletion | — |

---

## Stack

| Component | Value |
|---|---|
| OS | Debian 13 |
| Web server | Apache 2.4 on port 11000 |
| PHP | 8.4-FPM |
| Database | PostgreSQL 17 |
| Cache | Redis 8.0 |
| Nextcloud | 33.x |
| Reverse proxy | NGINX Proxy Manager → Cloudflare (or any reverse proxy) |

---

## 1. Base System

```bash
apt-get update && apt-get upgrade -y
apt-get install -y apache2 postgresql redis-server php8.4-fpm \
  php8.4-{cli,curl,gd,mbstring,xml,zip,bcmath,gmp,intl,pgsql,apcu,redis,imagick} \
  libapache2-mod-xsendfile inotify-tools ffmpeg smbclient php8.4-smbclient \
  ghostscript python3 curl wget git unzip fail2ban

a2enmod rewrite headers env dir mime proxy_fcgi setenvif remoteip
a2enconf php8.4-fpm
systemctl enable --now apache2 postgresql redis-server php8.4-fpm
```

### PostgreSQL

```bash
sudo -u postgres psql <<EOF
CREATE USER ncuser WITH PASSWORD 'YOUR_DB_PASSWORD';
CREATE DATABASE nextcloud OWNER ncuser;
GRANT ALL PRIVILEGES ON DATABASE nextcloud TO ncuser;
EOF
```

### Redis

```ini
# /etc/redis/redis.conf — add/set:
unixsocket /run/redis/redis-server.sock
unixsocketperm 770
requirepass YOUR_REDIS_PASSWORD
```

```bash
usermod -aG redis www-data
systemctl restart redis-server
```

### Nextcloud Download

```bash
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
tar -xjf latest.tar.bz2 -C /home/
chown -R www-data:www-data /home/nextcloud
mkdir -p /var/ncdata && chown www-data:www-data /var/ncdata
```

### occ Install

```bash
su -s /bin/sh www-data -c "php /home/nextcloud/occ maintenance:install \
  --database pgsql \
  --database-name nextcloud \
  --database-host 127.0.0.1 \
  --database-user ncuser \
  --database-pass 'YOUR_DB_PASSWORD' \
  --admin-user admin \
  --admin-pass 'YOUR_ADMIN_PASSWORD' \
  --data-dir /var/ncdata"
```

---

## 2. Apache Configuration

`/etc/apache2/sites-available/nextcloud.conf`:

```apache
<VirtualHost *:11000>
    ServerName YOUR_DOMAIN
    DocumentRoot /home/nextcloud

    <Directory /home/nextcloud>
        Options +FollowSymlinks
        AllowOverride All
        Require all granted
        <IfModule mod_dav.c>
            Dav off
        </IfModule>
    </Directory>

    # Fast file streaming — PHP authenticates, Apache serves bytes
    XSendFile On
    XSendFilePath /var/ncdata
    XSendFilePath /home/ncdata

    # Matterbridge media relay (public, no auth required)
    Alias /matterbridge-media /var/matterbridge-media
    <Directory /var/matterbridge-media>
        Options -Indexes
        AllowOverride None
        Require all granted
    </Directory>

    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.4-fpm.sock|fcgi://localhost"
    </FilesMatch>

    # Behind a reverse proxy — adjust RemoteIPInternalProxy to match your proxy's LAN IP
    SetEnvIf X-Forwarded-Proto https HTTPS=on
    RemoteIPHeader X-Forwarded-For
    RemoteIPInternalProxy 127.0.0.1

    Header always set Strict-Transport-Security "max-age=15552000; includeSubDomains"
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set X-XSS-Protection "1; mode=block"
    Header always set Referrer-Policy "no-referrer"

    ErrorLog ${APACHE_LOG_DIR}/nextcloud_error.log
    CustomLog ${APACHE_LOG_DIR}/nextcloud_access.log combined
</VirtualHost>
```

```bash
a2ensite nextcloud
apache2ctl configtest && systemctl reload apache2
```

---

## 3. Nextcloud config.php

All set via occ — do not edit config.php directly (PHP-FPM opcache will serve a stale compiled version).

> **Note:** If you must edit config.php directly, run `systemctl restart php8.4-fpm` afterwards and immediately `chown www-data:www-data /home/nextcloud/config/config.php` — editing as root changes file ownership and breaks the web app.

```bash
OCC="su -s /bin/sh www-data -c 'php /home/nextcloud/occ'"

# Domain
$OCC config:system:set trusted_domains 0 --value='YOUR_DOMAIN'
$OCC config:system:set overwrite.cli.url --value='YOUR_NC_URL'
$OCC config:system:set overwriteprotocol --value='https'
$OCC config:system:set overwritehost --value='YOUR_DOMAIN'

# Trusted proxies — add your reverse proxy LAN IP and Cloudflare ranges if applicable
$OCC config:system:set trusted_proxies 0 --value='127.0.0.1'
$OCC config:system:set trusted_proxies 1 --value='YOUR_PROXY_LAN_IP'

# If behind Cloudflare, use CF-Connecting-IP for real client IP
$OCC config:system:set forwarded_for_headers 0 --value='HTTP_CF_CONNECTING_IP'
$OCC config:system:set forwarded_for_headers 1 --value='HTTP_X_FORWARDED_FOR'

# Performance
$OCC config:system:set memcache.local --value='\OC\Memcache\APCu'
$OCC config:system:set memcache.distributed --value='\OC\Memcache\Redis'
$OCC config:system:set memcache.locking --value='\OC\Memcache\Redis'
$OCC config:system:set redis host --value='/run/redis/redis-server.sock'
$OCC config:system:set redis port --value=0 --type=integer
$OCC config:system:set redis password --value='YOUR_REDIS_PASSWORD'

# Logging
$OCC config:system:set loglevel --value=2 --type=integer
$OCC config:system:set logfile --value='/var/ncdata/nextcloud.log'

# XSendFile
$OCC config:system:set enable_xsendfile --value=true --type=boolean

# Locale — adjust to your region
$OCC config:system:set default_phone_region --value='US'
$OCC config:system:set default_timezone --value='America/New_York'
$OCC config:system:set defaultapp --value='spreed'
$OCC config:system:set maintenance_window_start --value=1 --type=integer
$OCC config:system:set upgrade.disable-web --value=true --type=boolean

# Preview providers — requires php8.4-imagick + libheif1 for HEIC/TIFF/WebP/PDF
# ghostscript required for PDF previews (included in base install above)
$OCC config:system:set enabledPreviewProviders 0 --value='OC\Preview\PNG'
$OCC config:system:set enabledPreviewProviders 1 --value='OC\Preview\JPEG'
$OCC config:system:set enabledPreviewProviders 2 --value='OC\Preview\GIF'
$OCC config:system:set enabledPreviewProviders 3 --value='OC\Preview\BMP'
$OCC config:system:set enabledPreviewProviders 4 --value='OC\Preview\XBitmap'
$OCC config:system:set enabledPreviewProviders 5 --value='OC\Preview\MP3'
$OCC config:system:set enabledPreviewProviders 6 --value='OC\Preview\TXT'
$OCC config:system:set enabledPreviewProviders 7 --value='OC\Preview\MarkDown'
$OCC config:system:set enabledPreviewProviders 8 --value='OC\Preview\Movie'
$OCC config:system:set enabledPreviewProviders 9 --value='OC\Preview\MKV'
$OCC config:system:set enabledPreviewProviders 10 --value='OC\Preview\MP4'
$OCC config:system:set enabledPreviewProviders 11 --value='OC\Preview\AVI'
$OCC config:system:set enabledPreviewProviders 12 --value='OC\Preview\HEIC'
$OCC config:system:set enabledPreviewProviders 13 --value='OC\Preview\TIFF'
$OCC config:system:set enabledPreviewProviders 14 --value='OC\Preview\WebP'
$OCC config:system:set enabledPreviewProviders 15 --value='OC\Preview\PDF'
$OCC config:system:set enabledPreviewProviders 16 --value='OC\Preview\ICO'
$OCC config:system:set preview_max_x --value=2048 --type=integer
$OCC config:system:set preview_max_y --value=2048 --type=integer
$OCC config:system:set preview_max_filesize_image --value=50 --type=integer
```

---

## 4. Users

```bash
# Add users — repeat for each person
su -s /bin/sh www-data -c "php /home/nextcloud/occ user:add \
  --display-name='Alice' --password-from-env alice" <<< "THEIR_PASSWORD"
```

---

## 5. Scheduled Maintenance

`/etc/cron.d/nextcloud-maintenance`:

```cron
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# PostgreSQL VACUUM — prevents table bloat
0 1 * * 0 postgres vacuumdb --all --analyze -q

# Orphaned filecache cleanup
15 1 * * 0 www-data php /home/nextcloud/occ files:cleanup

# Empty trash (all users)
30 1 * * 1 www-data php /home/nextcloud/occ trashbin:cleanup --all-users

# Prune old file versions
0 2 * * 2 www-data php /home/nextcloud/occ versions:cleanup

# Missing DB indices (safe after upgrades)
0 2 1 * * www-data php /home/nextcloud/occ db:add-missing-indices

# Full repair
30 2 1 * * www-data php /home/nextcloud/occ maintenance:repair --quiet

# Matterbridge media cleanup (older than 24h)
0 4 * * * root find /var/matterbridge-media -type f -mtime +1 -delete
```

---

## 6. SMTP

```bash
OCC="su -s /bin/sh www-data -c 'php /home/nextcloud/occ'"
$OCC config:system:set mail_smtpmode --value='smtp'
$OCC config:system:set mail_smtphost --value='YOUR_SMTP_HOST'
$OCC config:system:set mail_smtpport --value='YOUR_SMTP_PORT'
$OCC config:system:set mail_smtpsecure --value='tls'
$OCC config:system:set mail_smtpauth --value=1 --type=integer
$OCC config:system:set mail_smtpauthtype --value='LOGIN'
$OCC config:system:set mail_smtpname --value='YOUR_SMTP_USER'
$OCC config:system:set mail_smtppassword --value='YOUR_SMTP_PASSWORD'
$OCC config:system:set mail_from_address --value='noreply'
$OCC config:system:set mail_domain --value='YOUR_DOMAIN'
```

Test via Admin → Basic Settings → Send test email.

---

## 7. Nextcloud Music + SMB Library

### Install

```bash
su -s /bin/sh www-data -c "php /home/nextcloud/occ app:install music"
su -s /bin/sh www-data -c "php /home/nextcloud/occ app:enable files_external"
```

### SMB Mount

```bash
# Create mount (ID will be 1 if first mount)
su -s /bin/sh www-data -c "php /home/nextcloud/occ files_external:create 'Music Library' smb password::password"

su -s /bin/sh www-data -c "
php /home/nextcloud/occ files_external:config 1 host YOUR_SMB_HOST
php /home/nextcloud/occ files_external:config 1 share YOUR_SMB_SHARE
php /home/nextcloud/occ files_external:config 1 root 'YOUR_SMB_PATH'
php /home/nextcloud/occ files_external:config 1 user YOUR_SMB_USER
php /home/nextcloud/occ files_external:config 1 password YOUR_SMB_PASSWORD
php /home/nextcloud/occ files_external:config 1 domain ''
php /home/nextcloud/occ files_external:option 1 read_only 1
"

# Grant access to each user
su -s /bin/sh www-data -c "php /home/nextcloud/occ files_external:applicable --add-user alice 1"

# Verify connection
su -s /bin/sh www-data -c "php /home/nextcloud/occ files_external:verify 1"
```

### Scan & Index

```bash
# Run per user — may take a while for large libraries
su -s /bin/sh www-data -c "php /home/nextcloud/occ files:scan --path='/alice/files/Music Library'"
su -s /bin/sh www-data -c "php /home/nextcloud/occ music:scan alice"
```

### Last.fm Scrobbling

Register a free API app at https://www.last.fm/api/account/create (only contact email and app name required).

```bash
su -s /bin/sh www-data -c "
php /home/nextcloud/occ config:system:set music.lastfm_api_key --value='YOUR_LASTFM_API_KEY'
php /home/nextcloud/occ config:system:set music.lastfm_api_secret --value='YOUR_LASTFM_SECRET'
"
```

Each user connects their own Last.fm account via Music → Settings → Connect.

---

## 8. Telegram ↔ Talk Bridge (Matterbridge)

Bridges a Nextcloud Talk room to a Telegram group. Text and media relay in both directions.

### Prerequisites

- Nextcloud Talk app installed and configured
- A Telegram bot created via @BotFather
- Bot added to the Telegram group as admin
- Bot privacy mode **disabled** (BotFather → /mybots → Group Privacy → Turn off)

### Install Talk App

```bash
su -s /bin/sh www-data -c "php /home/nextcloud/occ app:install spreed"
```

The Matterbridge binary ships with the Talk app at:
`/home/nextcloud/apps/talk_matterbridge/bin/matterbridge-*-linux-64bit`

### Configure Bridge via Database

```bash
# Get the Talk room's numeric ID
sudo -u postgres psql nextcloud -c "SELECT id, token, name FROM oc_talk_rooms;"

# Insert bridge config (replace room_id, bot_token, chat_id)
sudo -u postgres psql nextcloud -c "
INSERT INTO oc_talk_bridges (room_id, json_values, enabled, pid)
VALUES (
  ROOM_ID,
  '[{\"type\":\"telegram\",\"token\":\"YOUR_TELEGRAM_BOT_TOKEN\",\"channel\":\"YOUR_TELEGRAM_GROUP_ID\"}]',
  1, 0
);"
```

Telegram group IDs are negative numbers. Get yours by adding your bot to the group and calling:
`https://api.telegram.org/botYOUR_TELEGRAM_BOT_TOKEN/getUpdates`

### Activate via Talk API

```bash
curl -s -X PUT \
  -u "admin:YOUR_ADMIN_PASSWORD" \
  -H "OCS-APIREQUEST: true" \
  -H "Content-Type: application/json" \
  "YOUR_NC_URL/ocs/v2.php/apps/spreed/api/v1/bridge/YOUR_TALK_TOKEN" \
  -d '{"parts":[{"type":"telegram","token":"YOUR_TELEGRAM_BOT_TOKEN","channel":"YOUR_TELEGRAM_GROUP_ID"}],"enabled":true}'
```

### Patch MatterbridgeManager.php for Native Formatting

The Talk app auto-generates the Matterbridge config. These edits improve message appearance and enable media relay.

**File:** `/home/nextcloud/apps/spreed/lib/MatterbridgeManager.php`

> **Warning:** This file is overwritten on Talk app updates. Re-apply the patch after each update.

**1. Add `[general]` media config** — insert before the `foreach` loop in `generateConfig()`:

```php
private function generateConfig(array $bridge): string {
    $content = '[general]' . "\n";
    $content .= '	MediaDownloadPath = "/var/matterbridge-media"' . "\n";
    $content .= '	MediaServerDownload = "YOUR_NC_URL/matterbridge-media"' . "\n";
    $content .= '	MediaDownloadSize = 52428800' . "\n\n";
    foreach ($bridge['parts'] as $k => $part) {
```

**2. Replace the `telegram` section block:**

```php
} elseif ($type === 'telegram') {
    $content .= sprintf('[%s.%s]', $type, $k) . "\n";
    $content .= sprintf('	Token = "%s"', $part['token']) . "\n";
    $content .= '	PrefixMessagesWithNick = true' . "\n";
    $content .= '	RemoteNickFormat = "{NICK}: "' . "\n";
    $content .= '	UseFirstName = true' . "\n";
    $content .= '	MediaConvertWebPToPNG = true' . "\n\n";
```

**3. Replace the `nctalk` nick/media lines:**

```php
$content .= '	PrefixMessagesWithNick = false' . "\n";
$content .= '	RemoteNickFormat="{NICK}"' . "\n";
$content .= '	MediaDownloads = ["telegram"]' . "\n\n";
```

After patching, restart PHP-FPM and re-PUT the bridge via the Talk API to regenerate the TOML:

```bash
systemctl restart php8.4-fpm
# Then re-run the curl PUT command above
```

### Media Directory

```bash
mkdir -p /var/matterbridge-media
chown www-data:www-data /var/matterbridge-media
chmod 755 /var/matterbridge-media
```

---

## 9. Telegram Media Relay Service

Watches for files Matterbridge downloads from Telegram, uploads them to Nextcloud natively, and shares them in the Talk room as inline image attachments. Deletes the raw URL message Matterbridge posts first.

See `scripts/mb-media-to-talk`. Update `TOML`, `NC_URL`, and `TALK_TOKEN` at the top of the script.

```bash
cp scripts/mb-media-to-talk /usr/local/bin/mb-media-to-talk
chmod +x /usr/local/bin/mb-media-to-talk
cp systemd/mb-media-to-talk.service /etc/systemd/system/mb-media-to-talk.service
systemctl daemon-reload
systemctl enable --now mb-media-to-talk
```

---

## 10. NGINX Proxy Manager Settings

In NPM's Advanced tab for the Nextcloud proxy host, add:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $http_connection;
proxy_set_header Range $http_range;
proxy_set_header If-Range $http_if_range;
proxy_max_temp_file_size 0;
proxy_buffering off;
gzip off;
```

These settings fix video streaming buffering and range request handling.

---

## 11. coturn TURN Server

Required for Talk video/audio calls through NAT (cellular, strict home ISPs). Can run on the same host or a separate one.

```bash
apt-get install -y coturn
```

`/etc/turnserver.conf`:

```ini
listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
cert=/etc/letsencrypt/live/YOUR_TURN_DOMAIN/fullchain.pem
pkey=/etc/letsencrypt/live/YOUR_TURN_DOMAIN/privkey.pem
relay-ip=YOUR_LAN_IP
external-ip=YOUR_PUBLIC_IP/YOUR_LAN_IP
min-port=49152
max-port=65535
realm=YOUR_TURN_DOMAIN
server-name=YOUR_TURN_DOMAIN
use-auth-secret
static-auth-secret=YOUR_TURN_SECRET
no-multicast-peers
denied-peer-ip=10.0.0.0-10.255.255.255
denied-peer-ip=192.168.0.0-192.168.255.255
denied-peer-ip=172.16.0.0-172.31.255.255
denied-peer-ip=127.0.0.0-127.255.255.255
fingerprint
log-file=/var/log/turnserver.log
simple-log
```

> **Port range:** `49152–65535` gives headroom for many concurrent calls. Each WebRTC call uses ~4 ports. Forward the full range TCP+UDP on your router/firewall.

Register in Nextcloud Talk admin panel: Settings → Talk → TURN servers → add `turns:YOUR_TURN_DOMAIN:5349` with your secret.

```bash
# Generate secret
openssl rand -hex 32

systemctl enable --now coturn
```

---

## 12. Fail2ban

Protects against brute-force login attempts on Nextcloud, Apache, and SSH.

`/etc/fail2ban/filter.d/nextcloud.conf` (see `fail2ban/filter.d/nextcloud.conf`):

```ini
[Definition]
failregex = .*"remoteAddr":"<HOST>".*"message":"Login failed:.*
            .*"remoteAddr":"<HOST>".*"message":"Trusted domain error.*
ignoreregex =
datepattern = "time"\s*:\s*"%%Y-%%m-%%dT%%H:%%M:%%S
```

`/etc/fail2ban/jail.d/nextcloud.conf` (see `fail2ban/jail.d/nextcloud.conf`):

```ini
[nextcloud]
enabled  = true
port     = http,https,11000
filter   = nextcloud
logpath  = /var/ncdata/nextcloud.log
maxretry = 5
findtime = 10m
bantime  = 1h
action   = iptables-multiport[name=nextcloud, port="http,https,11000", protocol=tcp]

[apache-auth]
enabled  = true
port     = http,https,11000
filter   = apache-auth
logpath  = /var/log/apache2/nextcloud_error.log
maxretry = 5
findtime = 10m
bantime  = 1h

[apache-badbots]
enabled  = true
port     = http,https,11000
filter   = apache-badbots
logpath  = /var/log/apache2/nextcloud_access.log
maxretry = 2
findtime = 10m
bantime  = 24h

[sshd]
enabled  = true
port     = ssh
filter   = sshd
logpath  = /var/log/auth.log
maxretry = 5
findtime = 10m
bantime  = 1h
```

```bash
systemctl enable --now fail2ban

# Verify all jails loaded
fail2ban-client status

# Whitelist your own IP to avoid locking yourself out
# Add to /etc/fail2ban/jail.d/nextcloud.conf under [DEFAULT]:
# ignoreip = 127.0.0.1/8 ::1 YOUR.HOME.IP

# Unban an IP manually
fail2ban-client set nextcloud unbanip 1.2.3.4
```

---

## 13. Archive Bot (Talk)

A Nextcloud Talk bot that intercepts paywalled links, fetches the archive.ph snapshot (or Wayback Machine fallback), posts the direct archive URL as a threaded reply, and deletes the original message.

Works in any Talk room it's enabled in, including rooms with the Telegram bridge — links posted from Telegram are caught too.

### Prerequisites

```bash
# Generate bot HMAC secret
openssl rand -hex 32   # → YOUR_BOT_SECRET

# Generate admin app password (needed to delete messages)
su -s /bin/bash www-data -c \
  "php8.4 /home/nextcloud/occ user:auth-tokens:add --name 'archive-bot-delete' admin"
# → YOUR_ADMIN_APP_PASS
```

### Bot Script

See `scripts/archive-bot`. Update these values at the top:

| Variable | Description |
|---|---|
| `SECRET` | `YOUR_BOT_SECRET` as bytes |
| `NC_URL` | `YOUR_NC_URL` |
| `NC_ADMIN_PASS` | `YOUR_ADMIN_APP_PASS` |

```bash
cp scripts/archive-bot /usr/local/bin/archive-bot
chmod +x /usr/local/bin/archive-bot
cp systemd/archive-bot.service /etc/systemd/system/archive-bot.service
systemctl daemon-reload
systemctl enable --now archive-bot
```

### Add Admin to Rooms

Admin must be a moderator in every room where the bot is active so it can delete messages:

```bash
OCC="su -s /bin/bash www-data -c 'php8.4 /home/nextcloud/occ'"

# Repeat for each room token
$OCC talk:room:add --user admin YOUR_TALK_TOKEN
$OCC talk:room:promote YOUR_TALK_TOKEN admin
```

### Register and Enable Bot

```bash
# Register system-wide (run once)
php8.4 /home/nextcloud/occ talk:bot:install \
  "archive-bot" \
  "YOUR_BOT_SECRET" \
  "http://127.0.0.1:9876" \
  "Replies with archive.ph links for paywalled URLs"

# Get the assigned bot ID
php8.4 /home/nextcloud/occ talk:bot:list

# Enable in each room (repeat per room token)
php8.4 /home/nextcloud/occ talk:bot:setup BOT_ID YOUR_TALK_TOKEN
```

### How It Works

1. Talk sends a signed webhook to `http://127.0.0.1:9876` for every message
2. Bot verifies HMAC-SHA256 signature using the shared secret
3. Extracts URLs and checks domain against the paywall list (edit `PAYWALL_DOMAINS` in the script)
4. Strips tracking query parameters (`utm_*`, `srnd`, `fbclid`, etc.) before archive lookup
5. Tries `archive.ph/newest/{url}` — returns direct timestamped URL if found
6. Falls back to Wayback Machine availability check
7. Posts archive URL as a threaded reply using the bot API
8. Deletes the original message using admin credentials

---

## 14. Two-Factor Authentication (TOTP)

```bash
su -s /bin/bash www-data -c "php8.4 /home/nextcloud/occ app:enable twofactor_totp"
su -s /bin/bash www-data -c "php8.4 /home/nextcloud/occ app:enable twofactor_backupcodes"
```

Users enable it themselves: Profile → Settings → Security → Two-Factor Authentication → TOTP.

To **enforce** 2FA for all users:

```bash
su -s /bin/bash www-data -c "php8.4 /home/nextcloud/occ twofactorauth:enforce --on"
```

To disable 2FA for a locked-out user:

```bash
su -s /bin/bash www-data -c "php8.4 /home/nextcloud/occ twofactorauth:disable USERNAME totp"
```

---

## 15. Post-Install Checklist

- [ ] Test SMTP — Admin → Basic Settings → Send test email
- [ ] Connect Last.fm — Music → Settings → Connect (each user does this individually)
- [ ] Test Telegram bridge — send a message both ways
- [ ] Send a photo from Telegram — confirm it appears as inline image in Talk
- [ ] Share a photo in Talk — confirm it appears in Telegram
- [ ] Verify SMB music mount is read-only (`files_external:verify 1`)
- [ ] Post a paywalled link — confirm archive-bot replies with a direct archive URL and deletes the original
- [ ] Verify fail2ban jails are active: `fail2ban-client status`
- [ ] Test a Talk video call — confirm TURN server is used for NAT traversal
- [ ] Enable TOTP for admin account before sharing server with users
- [ ] Confirm all services are running:

```bash
systemctl status mb-media-to-talk archive-bot fail2ban coturn
```

---

## Key File Locations

| File | Purpose |
|---|---|
| `/home/nextcloud/config/config.php` | Nextcloud config (edit via occ only) |
| `/etc/apache2/sites-available/nextcloud.conf` | Apache vhost |
| `/etc/cron.d/nextcloud-maintenance` | Scheduled maintenance |
| `/usr/local/bin/mb-media-to-talk` | Telegram media relay |
| `/usr/local/bin/archive-bot` | Paywalled link archive bot |
| `/var/matterbridge-media/` | Temp media storage for bridge |
| `/var/ncdata/` | Nextcloud data directory |
| `/var/ncdata/nextcloud.log` | Nextcloud application log (used by fail2ban) |
| `/tmp/bridge-YOUR_TALK_TOKEN.toml` | Active Matterbridge config (auto-generated, rotates on restart) |
| `/tmp/bridge-YOUR_TALK_TOKEN.log` | Matterbridge live log |
| `/home/nextcloud/apps/spreed/lib/MatterbridgeManager.php` | Patched for native message formatting — re-patch after Talk updates |
| `/var/log/apache2/nextcloud_access.log` | Apache access log |
| `/etc/fail2ban/filter.d/nextcloud.conf` | Fail2ban filter for Nextcloud login failures |
| `/etc/fail2ban/jail.d/nextcloud.conf` | Fail2ban jails (Nextcloud, Apache, SSH) |
| `/etc/turnserver.conf` | coturn TURN server config |
