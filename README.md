# Property Mitras — server access

How to reach the WordPress install on the server and work with it safely.

**No credentials in this file.** The password lives in `deploy.conf`, which is
gitignored. This document is safe to commit.

| | |
|---|---|
| Site | https://propertymitras.seoxpert360.com |
| WP admin | https://propertymitras.seoxpert360.com/wp-login.php |
| Host | AWS EC2 `13.232.54.1` (Elastic IP, static), ap-south-1 Mumbai |
| Panel | CloudPanel — https://13.232.54.1:8443 |
| Stack | WordPress 7.1.2 · PHP 8.3 · MySQL 8.4 (Percona) · nginx |
| Table prefix | `pm_` (not the default `wp_`) |

---

## Connecting

Two accounts exist. **Use the site user for anything to do with this website.**

### Site user — `propmitras` (use this one)

```bash
ssh propmitras@13.232.54.1
# password: see SSH_PASS in deploy.conf
```

Confined to `/home/propmitras`. No sudo. **Cannot read the other sites on this
server** — verified. That confinement is the point: a mistake here cannot reach
True Marketing or the Laravel CMS.

### Admin user — `ubuntu` (server work only)

```bash
ssh -i /d/Storage/AWS-Free/laravel-cms-key.pem ubuntu@13.232.54.1
```

Key auth, passwordless sudo, full access. Use it for nginx, PHP config,
certificates and logs — not for editing this site.

### If the connection times out

SSH is firewalled to `103.203.146.0/24`, the admin ISP range. From any other
network it hangs and then times out — that is the firewall, not the server.

Add your current address (needs AWS credentials, **one line** — backslash
continuation does not work in Windows CMD):

```bash
aws ec2 authorize-security-group-ingress --group-id sg-0dda218a6600246e0 --ip-permissions "IpProtocol=tcp,FromPort=22,ToPort=22,IpRanges=[{CidrIp=YOUR.IP.HERE/32,Description=temp}]" --region ap-south-1 --profile aws-free
```

---

## Where things are

```
/home/propmitras/htdocs/propertymitras.seoxpert360.com/   <- WordPress root
├── wp-config.php          DB credentials + salts (640, do not commit)
├── wp-content/
│   ├── themes/
│   ├── plugins/
│   └── uploads/           the only user data not held in the database
└── wp-admin/ wp-includes/ core - replaceable, never edit
/home/propmitras/logs/     nginx + php logs for this site
/home/propmitras/bin/wp    WP-CLI wrapper pinned to PHP 8.3 (see below)
```

---

## WP-CLI

Installed and on PATH. **Plain `wp` runs PHP 8.3**, matching the website, via
`~/bin/wp`.

```bash
SITE=~/htdocs/propertymitras.seoxpert360.com

wp --path=$SITE core version
wp --path=$SITE plugin list
wp --path=$SITE user list
wp --path=$SITE plugin update --all
wp --path=$SITE cache flush
```

### Changing the site URL

Never edit `siteurl` alone — serialised data in options and page builders stores
the old URL and breaks. Use search-replace, dry run first, every time:

```bash
wp --path=$SITE search-replace 'https://propertymitras.seoxpert360.com' 'https://newdomain.com' --all-tables --dry-run
```

---

## Database

WP-CLI reads credentials from `wp-config.php`, so you rarely need them directly.

```bash
wp --path=$SITE db check
wp --path=$SITE db size --tables
wp --path=$SITE db export ~/manual-backup-$(date +%F).sql   # before anything risky
wp --path=$SITE db query "SELECT option_name FROM pm_options LIMIT 5"
```

---

## Moving files

SFTP is enabled; same credentials as SSH.

```bash
sftp propmitras@13.232.54.1
scp -r ./my-theme propmitras@13.232.54.1:~/htdocs/propertymitras.seoxpert360.com/wp-content/themes/
```

Then fix permissions so PHP can read it:

```bash
chmod -R 755 ~/htdocs/propertymitras.seoxpert360.com/wp-content/themes/my-theme
find ~/htdocs/propertymitras.seoxpert360.com/wp-content/themes/my-theme -type f -exec chmod 644 {} \;
```

---

## Logs

Nothing grows dangerously here, but this is where to look and how to clear it.

| Log | Path |
|---|---|
| nginx access | `/home/propmitras/logs/nginx/access.log` |
| nginx error | `/home/propmitras/logs/nginx/error.log` |
| PHP errors | `/home/propmitras/logs/php/error.log` |
| Varnish purge | `/home/propmitras/logs/varnish-cache/purge.log` |

All root-owned — reading or clearing needs the **`ubuntu`** account.

```bash
ssh -i /d/Storage/AWS-Free/laravel-cms-key.pem ubuntu@13.232.54.1

sudo tail -50 /home/propmitras/logs/php/error.log
sudo tail -50 /home/propmitras/logs/nginx/error.log

# clear without deleting
sudo truncate -s 0 /home/propmitras/logs/nginx/access.log
sudo truncate -s 0 /home/propmitras/logs/nginx/error.log
sudo truncate -s 0 /home/propmitras/logs/php/error.log
```

Use `truncate -s 0`, not `rm`. Deleting leaves nginx writing to an unnamed
handle and the space is not freed until it restarts.

**There is no WordPress `debug.log`** — `WP_DEBUG` is `false`, correct for a live
site. To debug, set `WP_DEBUG` and `WP_DEBUG_LOG` to `true` in `wp-config.php`,
reproduce, read `wp-content/debug.log`, then turn it off again.

### If WP-CLI prints a wall of PHP notices

The cause is a **PHP version mismatch**. The website runs PHP 8.3 (nginx proxies
to `127.0.0.1:18002`), but `/usr/bin/wp` is the raw phar and its
`#!/usr/bin/env php` shebang resolves to **8.4**.

Fixed on 2026-10-04 with `~/bin/wp`:

```sh
#!/bin/sh
exec /usr/bin/php8.3 /usr/bin/wp "$@"
```

`~/bin` is prepended to PATH in `.bashrc` **above** the interactive-only return,
so it applies to `ssh user@host 'cmd'` as well as interactive logins. That
matters because deploy scripts use non-login shells.

Setting `WP_CLI_PHP` does **not** work — it is only honoured by WP-CLI's bash
wrapper, not the phar. For unattended scripts, call the full path
(`/usr/bin/php8.3 /usr/bin/wp`), which is what `REMOTE_WP` in `deploy.conf` uses.

---

## Things that will catch you out

**`blog_public` is 0.** Search engines are told not to index this site — correct
while it is empty, but it stays that way until changed. At launch:

```bash
wp --path=$SITE option update blog_public 1
```

**PHP-FPM is capped at `pm.max_children = 6`.** The box has 2 GB RAM and
CloudPanel defaults new pools to 250, which lets PHP exhaust memory and trigger
the OOM killer, taking down every site. Check `free -m` before raising it.
Changing it needs the `ubuntu` account.

**File editing is disabled in wp-admin** (`DISALLOW_FILE_EDIT`). Deliberate —
edit over SFTP or locally, not through the browser.

**all-in-one-wp-migration is active**, and its import size is bounded by PHP's
`upload_max_filesize` / `post_max_size` (both 32M). For a larger restore, use
WP-CLI and a database dump rather than the plugin.

**Imagick, redis and memcached are all loaded** (since 2026-10-04), so WordPress
uses `WP_Image_Editor_Imagick` rather than GD. Redis and memcached servers run on
the host if you ever want object caching.

**Core fails `wp core verify-checksums`** with ~21 "file should not exist"
warnings under `wp-includes/php-ai-client/`. Known false positive in WordPress
7.1 — the same warnings appear on True Marketing. Not tampering.

---

## Backups

`/usr/local/bin/site-backup.sh` runs nightly at **02:15 UTC** and discovers this
site automatically. It captures the database, `wp-content` and `wp-config.php`,
keeping 7 days in `/var/backups/sites` (root-only — use the `ubuntu` account).

Encrypted off-server copies live in the **seoxpert360-infra** repo, which also
holds the rebuild and migration runbooks.

Before anything risky, take your own snapshot:

```bash
wp --path=$SITE db export ~/before-change-$(date +%F).sql
```
