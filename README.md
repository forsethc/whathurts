# whathurts.today

A minimalist joke site about waking up sore.

## Repo layout

```
site/           # web root — symlink this to /var/www/whathurts
  hurt.txt      # what hurts today (edit this)
  og-image.svg  # source for link preview image
  og-image.png  # Open Graph / iMessage preview (generated from svg)
deploy/nginx/   # nginx config template
```

## Local preview

```bash
cd site && python3 -m http.server 8000
```

Open http://localhost:8000

## DNS (Namecheap)

In **Advanced DNS** for `whathurts.today`:

| Type  | Host | Value              |
|-------|------|--------------------|
| A     | `@`  | Your server IP     |
| CNAME | `www`| `whathurts.today`  |

Wait for propagation, then verify:

```bash
dig +short whathurts.today
dig +short www.whathurts.today
```

Both should return your server IP.

## Server setup

Install nginx and Certbot (Debian/Ubuntu):

```bash
sudo apt update
sudo apt install nginx certbot python3-certbot-nginx
```

Ensure ports **80** and **443** are open in your firewall.

## Deploy

Clone the repo on the server, then symlink only the site files to the web root:

```bash
git clone git@github.com:forsethc/whathurts.git
sudo ln -s /path/to/whathurts/site /var/www/whathurts
```

nginx `root` points at `/var/www/whathurts`, which resolves to `site/` — not the whole repo.

To update after pushing changes:

```bash
cd /path/to/whathurts && git pull
```

No rsync or copy step needed; the symlink picks up changes immediately.

## Updating what hurts

The center text lives in [`site/hurt.txt`](site/hurt.txt) — edit that file only, never the HTML.

**From laptop:**

```bash
echo "My knee" > site/hurt.txt
git add site/hurt.txt && git commit -m "knee today" && git push
ssh server 'cd /opt/whathurts && git pull'
```

**From phone:** edit `site/hurt.txt` in the GitHub app → commit → `git pull` on the server.

nginx should send `Cache-Control: no-cache` for `/hurt.txt` so browsers pick up changes on refresh (see [`deploy/nginx/whathurts.today.conf`](deploy/nginx/whathurts.today.conf)). Add the same `location = /hurt.txt` block to your live server config if it's not there yet, then `sudo nginx -t && sudo systemctl reload nginx`.

Verify:

```bash
curl -I https://whathurts.today/hurt.txt
# Cache-Control: no-cache
```

### Optional: auto-pull on the server

If you don't want to SSH for `git pull` after phone edits, add a cron job on the server:

```bash
# /etc/cron.d/whathurts-pull
*/15 * * * * root cd /opt/whathurts && git pull --quiet
```

Edits pushed to GitHub will show up within 15 minutes.

## Link preview (Open Graph / iMessage)

iOS and other apps scrape Open Graph meta tags from [`site/index.html`](site/index.html). The preview image is [`site/og-image.png`](site/og-image.png), generated from [`site/og-image.svg`](site/og-image.svg).

To update the preview image:

```bash
# 1. Edit site/og-image.svg, then regenerate the PNG
npx @resvg/resvg-js-cli site/og-image.svg site/og-image.png

# 2. Commit and deploy
git add site/og-image.svg site/og-image.png
git commit -m "update link preview"
git push
ssh server 'cd /opt/whathurts && git pull'
```

iMessage caches link previews — if the old image persists after deploy, share `https://whathurts.today/?v=2` once to bust the cache.

## nginx

Enable the site config:

```bash
sudo ln -s /path/to/whathurts/deploy/nginx/whathurts.today.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

The committed config is HTTP-only. Certbot adds TLS on the server.

## SSL with Certbot (Let's Encrypt)

Run Certbot on the host that terminates HTTP for the domain (port 80 must be reachable for the ACME challenge). If a reverse proxy sits in front of nginx, ensure it forwards `/.well-known/acme-challenge/` to the same box where Certbot runs.

### Initial setup (one-time)

After DNS resolves and nginx is serving the site on port 80:

```bash
sudo certbot --nginx -d whathurts.today -d www.whathurts.today
```

Certbot will:

- Obtain a certificate from Let's Encrypt (valid 90 days)
- Modify the nginx config **on the server** to add HTTPS and HTTP→HTTPS redirect
- Reload nginx

Choose **redirect** when asked about HTTP → HTTPS.

Verify HTTPS:

```bash
curl -I https://whathurts.today/
sudo certbot certificates
```

### Auto-renewal

Certbot installs a **systemd timer** that checks ~twice daily and renews when the cert has **≤30 days** left. No cron job or manual calendar reminder needed if renewal works.

Verify auto-renewal once after initial setup:

```bash
sudo certbot renew --dry-run
```

Confirm the timer is active:

```bash
sudo systemctl list-timers | grep certbot
```

Check expiry anytime:

```bash
sudo certbot certificates
```

### Manual renewal (when auto-renew fails)

Certbot emails the address given at signup if renewal fails. If you notice expiry warnings or `renew --dry-run` fails, fix the underlying issue first (see troubleshooting below), then:

```bash
sudo certbot renew
sudo nginx -t && sudo systemctl reload nginx
```

Renew a single cert by name:

```bash
sudo certbot renew --cert-name whathurts.today
```

Force renewal before the usual window (e.g. after fixing a broken config):

```bash
sudo certbot renew --force-renewal --cert-name whathurts.today
sudo nginx -t && sudo systemctl reload nginx
```

### Adding or changing domains

After changing `server_name` or adding hostnames, re-run:

```bash
sudo certbot --nginx -d whathurts.today -d www.whathurts.today
```

### Troubleshooting renewal failures

| Symptom | Things to check |
|---------|-----------------|
| Challenge fails | Port 80 open; nginx running; proxy forwards `/.well-known/acme-challenge/` |
| `nginx -t` fails after renew | Certbot may have edited config — inspect `/etc/nginx/sites-enabled/whathurts.today.conf` |
| Site works but renew fails | `sudo certbot renew --dry-run -v` for verbose output |
| Wrong nginx config | Don't overwrite the live server config with the HTTP-only file from this repo — Certbot's SSL lines live only on the server |

Useful logs:

```bash
sudo journalctl -u certbot.timer
sudo journalctl -u certbot.service
sudo tail /var/log/letsencrypt/letsencrypt.log
```

### What Certbot adds (on the server, not in git)

- `listen 443 ssl` and certificate paths under `/etc/letsencrypt/live/whathurts.today/`
- `include /etc/letsencrypt/options-ssl-nginx.conf`
- Port 80 redirect to HTTPS

Private keys stay in `/etc/letsencrypt/` on the server — never commit them.

The committed [`deploy/nginx/whathurts.today.conf`](deploy/nginx/whathurts.today.conf) is an HTTP-only baseline for fresh installs. After Certbot runs, the authoritative config is on the server.
