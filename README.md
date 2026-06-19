# whathurts.today

A minimalist joke site about waking up sore.

## Local preview

```bash
python3 -m http.server 8000
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

Copy site files to the web root:

```bash
sudo mkdir -p /var/www/whathurts
rsync -av --delete index.html css/ user@your-server:/var/www/whathurts/
```

Or from the server after cloning the repo:

```bash
sudo mkdir -p /var/www/whathurts
sudo cp -r index.html css /var/www/whathurts/
```

## nginx

Enable the site config:

```bash
sudo ln -s /path/to/whathurts/deploy/nginx/whathurts.today.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

The committed config is HTTP-only. Certbot adds TLS on the server.

## SSL with Certbot (Let's Encrypt)

After DNS resolves and nginx is serving the site on port 80:

```bash
sudo certbot --nginx -d whathurts.today -d www.whathurts.today
```

Certbot will:

- Obtain a certificate from Let's Encrypt
- Modify the nginx config on the server to add HTTPS
- Set up HTTP → HTTPS redirect

Verify auto-renewal:

```bash
sudo certbot renew --dry-run
```

Certbot installs a systemd timer to renew automatically. No action needed unless renewal fails.

### What Certbot adds (on the server, not in git)

- `listen 443 ssl` and certificate paths under `/etc/letsencrypt/live/whathurts.today/`
- `include /etc/letsencrypt/options-ssl-nginx.conf`
- Port 80 redirect to HTTPS

Private keys stay in `/etc/letsencrypt/` on the server — never commit them.

## Updating the site

Edit files locally, then rsync again:

```bash
rsync -av --delete index.html css/ user@your-server:/var/www/whathurts/
```

No build step required.
