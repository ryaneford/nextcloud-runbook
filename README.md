# Nextcloud Talk Server — Installation Runbook

Self-hosted Nextcloud 33 with Talk, Music, Telegram bridge, hardware video transcoding, fail2ban hardening, and a Talk bot that auto-archives paywalled links.
Designed for a Proxmox VM behind NGINX Proxy Manager and Cloudflare.

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
| Reverse proxy | NGINX Proxy Manager → Cloudflare |

---

## 1. Base System

```bash
apt-get update && apt-get upgrade -y
apt-get install -y apache2 postgresql redis-server php8.4-fpm \
  php8.4-{cli,curl,gd,mbstring,xml,zip,bcmath,gmp,intl,pgsql,apcu,redis,imagick} \
  libapache2-mod-xsendfile inotify-tools ffmpeg smbclient php8.4-smbclient \
  python3 curl wget git unzip

a2enmod rewrite headers env dir mime proxy_fcgi setenvif remoteip
a2enconf php8.4-fpm
systemctl enable --now apache2 postgresql redis-server php8.4-fpm
```

### PostgreSQL

```bash
sudo -u postgres psql <<EOF
CREATE USER ncuser WITH PASSWORD 'CHANGE_ME';
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
  --database-pass 'DB_PASSWORD' \
  --admin-user admin \
  --admin-pass 'ADMIN_PASSWORD' \
  --data-dir /var/ncdata"
```

---

## 2. Apache Configuration

`/etc/apache2/sites-available/nextcloud.conf`:

```apache
<VirtualHost *:11000>
    ServerName your.domain.com
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

    # Behind NGINX Proxy Manager
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

All set via occ — do not edit config.php directly.

```bash
OCC="su -s /bin/sh www-data -c 'php /home/nextcloud/occ'"

# Domain
$OCC config:system:set trusted_domains 0 --value='your.domain.com'
$OCC config:system:set overwrite.cli.url --value='https://your.domain.com'
$OCC config:system:set overwriteprotocol --value='https'
$OCC config:system:set overwritehost --value='your.domain.com'

# Cloudflare trusted proxies (full list at cloudflare.com/ips)
$OCC config:system:set trusted_proxies 0 --value='127.0.0.1'
$OCC config:system:set trusted_proxies 1 --value='YOUR_NPM_LAN_IP'
# ... add Cloudflare IP ranges

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

# Misc
$OCC config:system:set default_phone_region --value='CA'
$OCC config:system:set default_timezone --value='America/Toronto'
$OCC config:system:set defaultapp --value='spreed'
$OCC config:system:set maintenance_window_start --value=1 --type=integer
$OCC config:system:set upgrade.disable-web --value=true --type=boolean

# Preview providers — requires php8.4-imagick + libheif1 for HEIC/TIFF/WebP/PDF
# ghostscript required for PDF previews (apt-get install -y ghostscript)
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
for user in ryan mike jimm dave; do
  su -s /bin/sh www-data -c "php /home/nextcloud/occ user:add \
    --display-name='${user^}' --password-from-env $user" <<< "USER_PASSWORD"
done
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

## 6. SMTP (ProtonMail)

```bash
OCC="su -s /bin/sh www-data -c 'php /home/nextcloud/occ'"
$OCC config:system:set mail_smtpmode --value='smtp'
$OCC config:system:set mail_smtphost --value='smtp.protonmail.ch'
$OCC config:system:set mail_smtpport --value='587'
$OCC config:system:set mail_smtpsecure --value='tls'
$OCC config:system:set mail_smtpauth --value=1 --type=integer
$OCC config:system:set mail_smtpauthtype --value='LOGIN'
$OCC config:system:set mail_smtpname --value='noreply@yourdomain.com'
$OCC config:system:set mail_smtppassword --value='YOUR_SMTP_TOKEN'
$OCC config:system:set mail_from_address --value='noreply'
$OCC config:system:set mail_domain --value='yourdomain.com'
```

Test via Admin → Basic Settings → Send test email.

---

## 7. Hardware Video Transcoding (Plex LXC)

Offloads high-bitrate Talk video uploads to an Intel iGPU in a Plex LXC container.
Only transcodes files above 10 Mbps. Uses VA-API with CQP mode (QP=38 ≈ 8-10 Mbps output).

### SSH Key Setup

```bash
ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519_transcode -N "" -C "nextcloud-transcode@yourdomain"
# Append public key to Plex LXC's /root/.ssh/authorized_keys
ssh-copy-id -i /root/.ssh/id_ed25519_transcode root@PLEX_LXC_IP
```

### Transcode Script

`/usr/local/bin/nc-transcode-video`:

```bash
#!/bin/bash
FILE="$1"
NC_USER="$2"
PLEX_HOST="PLEX_LXC_IP"
PLEX_SSH="ssh -i /root/.ssh/id_ed25519_transcode -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@$PLEX_HOST"
PLEX_SCP="scp -i /root/.ssh/id_ed25519_transcode -o StrictHostKeyChecking=no"
TMPDIR_REMOTE="/tmp/nc-transcode"
OCC=/home/nextcloud/occ
AUDIO_BITRATE=128k

prev=0
for i in $(seq 1 30); do
    cur=$(stat -c%s "$FILE" 2>/dev/null || echo 0)
    [ "$cur" -eq "$prev" ] && [ "$cur" -gt 0 ] && break
    prev=$cur; sleep 3
done

[ ! -s "$FILE" ] && exit 0

bitrate=$(ffprobe -v quiet -print_format json -show_format "$FILE" 2>/dev/null \
  | python3 -c "import json,sys; d=json.load(sys.stdin); print(int(d['format'].get('bit_rate',0)))" 2>/dev/null)
[ -z "$bitrate" ] || [ "$bitrate" -le 10000000 ] && exit 0

BASENAME=$(basename "$FILE")
REMOTE_IN="$TMPDIR_REMOTE/$BASENAME"
REMOTE_OUT="$TMPDIR_REMOTE/out_$BASENAME"

$PLEX_SSH "mkdir -p $TMPDIR_REMOTE" || exit 1
$PLEX_SCP "$FILE" "root@$PLEX_HOST:$REMOTE_IN" || exit 1

# Intel 6th gen iHD only supports CQP — QP 38 ≈ 8-10 Mbps
$PLEX_SSH "ffmpeg -y \
  -vaapi_device /dev/dri/renderD128 \
  -i '$REMOTE_IN' \
  -vf 'format=nv12,hwupload' \
  -c:v h264_vaapi -qp 38 \
  -c:a aac -b:a $AUDIO_BITRATE \
  -movflags +faststart -map_metadata 0 \
  '$REMOTE_OUT' 2>/dev/null"

[ $? -ne 0 ] && { $PLEX_SSH "rm -f '$REMOTE_IN' '$REMOTE_OUT'"; exit 1; }

TMPLOCAL="${FILE%.mp4}_new.mp4"
$PLEX_SCP "root@$PLEX_HOST:$REMOTE_OUT" "$TMPLOCAL"

if [ $? -eq 0 ] && [ -s "$TMPLOCAL" ]; then
    chown www-data:www-data "$TMPLOCAL"
    mv "$TMPLOCAL" "$FILE"
    chown www-data:www-data "$FILE"
    su -s /bin/sh www-data -c "php $OCC files:scan --path='/$NC_USER/files' -q" 2>/dev/null
fi

$PLEX_SSH "rm -f '$REMOTE_IN' '$REMOTE_OUT'"
rm -f "$TMPLOCAL"
```

### Video Watcher

`/usr/local/bin/nc-video-watcher`:

```bash
#!/bin/bash
DATA_DIR=/var/ncdata
USERS=(ryan mike jimm dave)
VIDEO_EXTS="mp4|mkv|avi|mov|wmv|webm|flv|m4v"

WATCH_PATHS=()
for u in "${USERS[@]}"; do
    dir="$DATA_DIR/$u/files/Talk"
    mkdir -p "$dir" && chown www-data:www-data "$dir"
    WATCH_PATHS+=("$dir")
done

inotifywait -m -e close_write --format '%w%f' "${WATCH_PATHS[@]}" 2>/dev/null \
| while read -r filepath; do
    if echo "$filepath" | grep -qiE "\.($VIDEO_EXTS)$"; then
        nc_user=$(echo "$filepath" | sed "s|$DATA_DIR/||;s|/.*||")
        /usr/local/bin/nc-transcode-video "$filepath" "$nc_user" &
    fi
done
```

`/etc/systemd/system/nc-video-watcher.service`:

```ini
[Unit]
Description=Nextcloud Talk video transcoder
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/nc-video-watcher
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
chmod +x /usr/local/bin/nc-transcode-video /usr/local/bin/nc-video-watcher
systemctl daemon-reload
systemctl enable --now nc-video-watcher
```

---

## 8. Nextcloud Music + SMB Library

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
php /home/nextcloud/occ files_external:config 1 host SMB_SERVER_IP
php /home/nextcloud/occ files_external:config 1 share SHARE_NAME
php /home/nextcloud/occ files_external:config 1 root 'path/to/music'
php /home/nextcloud/occ files_external:config 1 user SMB_USERNAME
php /home/nextcloud/occ files_external:config 1 password SMB_PASSWORD
php /home/nextcloud/occ files_external:config 1 domain ''
php /home/nextcloud/occ files_external:option 1 read_only 1
"

# Grant access to each user
for user in ryan mike jimm dave; do
  su -s /bin/sh www-data -c "php /home/nextcloud/occ files_external:applicable --add-user $user 1"
done

# Verify connection
su -s /bin/sh www-data -c "php /home/nextcloud/occ files_external:verify 1"
```

### Scan & Index

```bash
# Run per user — takes ~20 minutes for a large library
su -s /bin/sh www-data -c "php /home/nextcloud/occ files:scan --path='/ryan/files/Music Library'"
su -s /bin/sh www-data -c "php /home/nextcloud/occ music:scan ryan"
# Repeat for other users
```

### Last.fm Scrobbling

Register a free API app at https://www.last.fm/api/account/create (only Contact email and App name required).

```bash
su -s /bin/sh www-data -c "
php /home/nextcloud/occ config:system:set music.lastfm_api_key --value='YOUR_LASTFM_API_KEY'
php /home/nextcloud/occ config:system:set music.lastfm_api_secret --value='YOUR_LASTFM_API_SECRET'
"
```

Each user then connects their own Last.fm account via Music → Settings → Connect.

### Music Admin Settings

```bash
su -s /bin/sh www-data -c "
php /home/nextcloud/occ config:system:set music.cover_size --value=500 --type=integer
php /home/nextcloud/occ config:system:set music.ampache_session_expiry_time --value=86400 --type=integer
php /home/nextcloud/occ config:system:set music.podcast_auto_update_interval --value=12 --type=float
"
```

---

## 9. Telegram ↔ Talk Bridge (Matterbridge)

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
  '[{\"type\":\"telegram\",\"token\":\"BOT_TOKEN\",\"channel\":\"TELEGRAM_GROUP_ID\"}]',
  1, 0
);"
```

Telegram group IDs are negative numbers. Get yours by adding your bot to the group and calling:
`https://api.telegram.org/botTOKEN/getUpdates`

### Activate via Talk API

```bash
curl -s -X PUT \
  -u "USERNAME:APP_PASSWORD" \
  -H "OCS-APIREQUEST: true" \
  -H "Content-Type: application/json" \
  "https://your.domain.com/ocs/v2.php/apps/spreed/api/v1/bridge/ROOM_TOKEN" \
  -d '{"parts":[{"type":"telegram","token":"BOT_TOKEN","channel":"TELEGRAM_GROUP_ID"}],"enabled":true}'
```

### Patch MatterbridgeManager.php for Native Formatting

The Talk app generates the Matterbridge config. Two edits improve message appearance and enable media relay.

**File:** `/home/nextcloud/apps/spreed/lib/MatterbridgeManager.php`

**1. Add `[general]` media config** — insert before the `foreach` loop in `generateConfig()`:

```php
private function generateConfig(array $bridge): string {
    $content = '[general]' . "\n";
    $content .= '	MediaDownloadPath = "/var/matterbridge-media"' . "\n";
    $content .= '	MediaServerDownload = "https://your.domain.com/matterbridge-media"' . "\n";
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

**3. Replace the `nctalk` nick/media lines** (find the `PrefixMessagesWithNick` + `RemoteNickFormat` block inside the `nctalk` if-branch):

```php
$content .= '	PrefixMessagesWithNick = false' . "\n";
$content .= '	RemoteNickFormat="{NICK}"' . "\n";
$content .= '	MediaDownloads = ["telegram"]' . "\n\n";
```

After patching, restart PHP-FPM and re-PUT the bridge via the Talk API to regenerate the TOML:

```bash
systemctl restart php8.4-fpm
# Then re-run the curl PUT command from above
```

### Media Directory

```bash
mkdir -p /var/matterbridge-media
chown www-data:www-data /var/matterbridge-media
chmod 755 /var/matterbridge-media
```

---

## 10. Telegram Media Relay Service

Watches for files Matterbridge downloads from Telegram, uploads them to Nextcloud natively, and shares them in the Talk room as inline image attachments. Deletes the raw URL message Matterbridge posts first.

**Variables to update:** `TOML`, `NC_URL`, `TALK_TOKEN`

`/usr/local/bin/mb-media-to-talk`:

```bash
#!/bin/bash
MEDIA_DIR=/var/matterbridge-media
BRIDGE_USER="bridge-bot"
TOML="/tmp/bridge-ROOM_TOKEN.toml"
NC_URL="https://your.domain.com"
TALK_TOKEN="ROOM_TOKEN"
NC_FOLDER="TelegramMedia"

get_password() {
    grep 'Password' "$TOML" 2>/dev/null | grep -oP '(?<=")[^"]+(?=")'
}

get_sender_name() {
    grep "POST.*chat/$TALK_TOKEN.*actorDisplayName=.*Go-http-client" \
        /var/log/apache2/nextcloud_access.log 2>/dev/null \
        | tail -1 \
        | grep -oP '(?<=actorDisplayName=)[^& ]+' \
        | python3 -c "import sys,urllib.parse; print(urllib.parse.unquote(sys.stdin.read().strip()))" 2>/dev/null
}

delete_url_message() {
    local pw="$1"
    local response msg_id
    response=$(curl -s \
        -u "$BRIDGE_USER:$pw" \
        -H "OCS-APIREQUEST: true" \
        "$NC_URL/ocs/v2.php/apps/spreed/api/v1/chat/$TALK_TOKEN?lookIntoFuture=0&limit=10&format=json")
    msg_id=$(echo "$response" | python3 -c "
import sys, json
try:
    for msg in json.load(sys.stdin).get('ocs',{}).get('data',[]):
        if msg.get('actorId') == 'bridge-bot' and 'matterbridge-media' in msg.get('message',''):
            print(msg['id']); break
except: pass
" 2>/dev/null)
    [ -n "$msg_id" ] && curl -s -X DELETE \
        -u "$BRIDGE_USER:$pw" -H "OCS-APIREQUEST: true" \
        "$NC_URL/ocs/v2.php/apps/spreed/api/v1/chat/$TALK_TOKEN/$msg_id" -o /dev/null
}

upload_and_share() {
    local filepath="$1"
    local filename; filename=$(basename "$filepath")
    local pw; pw=$(get_password)
    [ -z "$pw" ] && return 1
    [ ! -s "$filepath" ] && return 1

    curl -s -X MKCOL -u "$BRIDGE_USER:$pw" \
        "$NC_URL/remote.php/dav/files/$BRIDGE_USER/$NC_FOLDER/" -o /dev/null

    local http_code
    http_code=$(curl -s -o /dev/null -w "%{http_code}" \
        -T "$filepath" -u "$BRIDGE_USER:$pw" \
        "$NC_URL/remote.php/dav/files/$BRIDGE_USER/$NC_FOLDER/$filename")

    if [ "$http_code" = "201" ] || [ "$http_code" = "204" ]; then
        sleep 2
        delete_url_message "$pw"

        local sender; sender=$(get_sender_name)
        [ -n "$sender" ] && curl -s -X POST \
            -u "$BRIDGE_USER:$pw" -H "OCS-APIREQUEST: true" \
            "$NC_URL/ocs/v2.php/apps/spreed/api/v1/chat/$TALK_TOKEN" \
            --data-urlencode "message=📷 $sender (Telegram):" -o /dev/null

        curl -s -X POST \
            -u "$BRIDGE_USER:$pw" -H "OCS-APIREQUEST: true" \
            "$NC_URL/ocs/v2.php/apps/files_sharing/api/v1/shares" \
            -d "path=/$NC_FOLDER/$filename&shareType=10&shareWith=$TALK_TOKEN" -o /dev/null
    fi
}

# Matterbridge creates a new subdir per file — watch parent for CREATE events,
# then poll up to 5s for the file (avoids inotifywait race condition)
inotifywait -m -e create "$MEDIA_DIR" --format '%w%f' 2>/dev/null \
| while read -r dirpath; do
    [ ! -d "$dirpath" ] && continue
    (
        for i in $(seq 1 5); do
            filepath=$(find "$dirpath" -maxdepth 1 -type f ! -name '.*' 2>/dev/null | head -1)
            if [ -n "$filepath" ]; then
                upload_and_share "$filepath"; break
            fi
            sleep 1
        done
    ) &
done
```

`/etc/systemd/system/mb-media-to-talk.service`:

```ini
[Unit]
Description=Matterbridge media relay to Nextcloud Talk
After=network.target nc-video-watcher.service

[Service]
Type=simple
ExecStart=/usr/local/bin/mb-media-to-talk
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
chmod +x /usr/local/bin/mb-media-to-talk
systemctl daemon-reload
systemctl enable --now mb-media-to-talk
```

---

## 11. NGINX Proxy Manager (NPM) Settings

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

## 12. coturn TURN Server

Required for Talk video/audio calls through NAT (cellular, strict home ISPs).

```bash
apt-get install -y coturn
```

`/etc/turnserver.conf`:

```ini
listening-port=3478
tls-listening-port=5349
listening-ip=0.0.0.0
cert=/etc/letsencrypt/live/turn.your.domain.com/fullchain.pem
pkey=/etc/letsencrypt/live/turn.your.domain.com/privkey.pem
relay-ip=YOUR_LAN_IP
external-ip=YOUR_PUBLIC_IP/YOUR_LAN_IP
min-port=49152
max-port=65535
realm=turn.your.domain.com
server-name=turn.your.domain.com
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

> **Port range:** `49152–65535` — each call uses ~4 ports. The range `49152–49162` (10 ports) only handles ~2 concurrent calls. Forward the full range TCP+UDP on your router.

Register in Nextcloud Talk admin panel (Settings → Talk → TURN servers) or via occ:

```bash
# Generate a random secret
openssl rand -hex 32

# Add TURN server in Talk admin UI or directly via DB
# Talk admin: Settings → Talk → TURN servers → add turns:turn.your.domain.com:5349
```

```bash
systemctl enable --now coturn
```

---

## 13. Fail2ban

Protects against brute-force login attempts on Nextcloud and SSH.

```bash
apt-get install -y fail2ban
```

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

# Verify all 4 jails loaded
fail2ban-client status

# Test filter against your log
fail2ban-regex /var/ncdata/nextcloud.log /etc/fail2ban/filter.d/nextcloud.conf
```

---

## 14. Archive Bot (Talk)

A Nextcloud Talk bot that intercepts paywalled links, fetches the archive.ph snapshot (or Wayback Machine fallback), posts the direct archive URL as a threaded reply, and deletes the original message.

Works for links posted from both Talk and Telegram (via the bridge).

### Prerequisites

```bash
# Generate bot secret and admin app password
openssl rand -hex 32   # use as BOT_SECRET below

# Generate admin app password (needed to delete messages as moderator)
su -s /bin/bash www-data -c \
  "php8.4 /home/nextcloud/occ user:auth-tokens:add --name 'archive-bot-delete' admin"
```

### Bot Script

See `scripts/archive-bot`. Update these variables at the top:

| Variable | Description |
|---|---|
| `SECRET` | 32-byte hex secret generated above |
| `NC_URL` | Your Nextcloud URL |
| `NC_ADMIN_PASS` | App password generated above |

```bash
cp scripts/archive-bot /usr/local/bin/archive-bot
chmod +x /usr/local/bin/archive-bot
```

### Add Admin to Rooms (required for message deletion)

Admin must be a moderator in every room where the bot is active:

```bash
OCC="su -s /bin/bash www-data -c 'php8.4 /home/nextcloud/occ'"

# Add and promote admin in each room (repeat per room token)
$OCC talk:room:add --user admin ROOM_TOKEN
$OCC talk:room:promote ROOM_TOKEN admin
```

### Systemd Service

See `systemd/archive-bot.service`:

```bash
cp systemd/archive-bot.service /etc/systemd/system/archive-bot.service
systemctl daemon-reload
systemctl enable --now archive-bot
```

### Register and Enable Bot

```bash
# Register system-wide (run once)
php8.4 /home/nextcloud/occ talk:bot:install \
  "archive-bot" \
  "BOT_SECRET" \
  "http://127.0.0.1:9876" \
  "Replies with archive.ph links for paywalled URLs"

# Check assigned ID
php8.4 /home/nextcloud/occ talk:bot:list

# Enable in each room (repeat per room token)
php8.4 /home/nextcloud/occ talk:bot:setup BOT_ID ROOM_TOKEN
```

### How It Works

1. Talk sends a webhook to `http://127.0.0.1:9876` for every message
2. Bot verifies HMAC-SHA256 signature using the shared secret
3. Extracts URLs and checks domain against paywall list
4. Strips tracking query parameters (`utm_*`, `srnd`, `fbclid`, etc.) before archive lookup
5. Tries `archive.ph/newest/{url}` — returns direct timestamped URL if found
6. Falls back to Wayback Machine availability check
7. Posts archive URL as a threaded reply using the bot API
8. Deletes the original message using admin credentials

### Adding Paywall Domains

Edit the `PAYWALL_DOMAINS` set in `/usr/local/bin/archive-bot` and restart:

```bash
systemctl restart archive-bot
```

---

## 15. Post-Install Checklist

- [ ] Test video upload in Talk — confirm auto-transcode triggers for files > 10 Mbps
- [ ] Test SMTP — Admin → Basic Settings → Send test email
- [ ] Connect Last.fm — Music → Settings → Connect (each user does this individually)
- [ ] Test Telegram bridge — send a message both ways
- [ ] Send a photo from Telegram — confirm it appears as inline image in Talk
- [ ] Share a photo in Talk — confirm it appears in Telegram
- [ ] Verify SMB music mount is read-only (`files_external:verify 1`)
- [ ] Post a paywalled link — confirm archive-bot replies with direct archive URL and deletes original
- [ ] Verify fail2ban jails are active: `fail2ban-client status`
- [ ] Confirm all services are running

```bash
systemctl status nc-video-watcher mb-media-to-talk archive-bot fail2ban coturn
```

---

## Key File Locations

| File | Purpose |
|---|---|
| `/home/nextcloud/config/config.php` | Nextcloud config (edit via occ only) |
| `/etc/apache2/sites-available/nextcloud.conf` | Apache vhost |
| `/etc/cron.d/nextcloud-maintenance` | Scheduled maintenance |
| `/usr/local/bin/nc-transcode-video` | Hardware transcode script |
| `/usr/local/bin/nc-video-watcher` | Talk video upload watcher |
| `/usr/local/bin/mb-media-to-talk` | Telegram media relay |
| `/usr/local/bin/archive-bot` | Paywalled link archive bot |
| `/var/matterbridge-media/` | Temp media storage for bridge |
| `/var/ncdata/` | Nextcloud data directory |
| `/tmp/bridge-ROOM_TOKEN.toml` | Active Matterbridge config (auto-generated) |
| `/tmp/bridge-ROOM_TOKEN.log` | Matterbridge live log |
| `/home/nextcloud/apps/spreed/lib/MatterbridgeManager.php` | Patched for native message formatting |
| `/var/log/apache2/nextcloud_access.log` | Apache access log (used by relay service) |
| `/var/ncdata/nextcloud.log` | Nextcloud application log (used by fail2ban) |
| `/etc/fail2ban/filter.d/nextcloud.conf` | Fail2ban filter for Nextcloud login failures |
| `/etc/fail2ban/jail.d/nextcloud.conf` | Fail2ban jails (Nextcloud, Apache, SSH) |
| `/etc/turnserver.conf` | coturn TURN server config |
