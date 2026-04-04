# Vyakt Vultr Deployment Guide (Ubuntu)

This guide deploys Vyakt on a Vultr Ubuntu server with:

- Gunicorn (app server)
- Nginx (reverse proxy)
- systemd (auto-start)
- SSL via Certbot (Let's Encrypt)

It assumes MongoDB is hosted externally (Atlas) and your app already has MONGO_URI.

## 1. Prerequisites

- Vultr instance created and running
- Ubuntu selected
- SSH key added to Vultr and attached to server
- Firewall allows inbound ports: 22, 80, 443
- Domain optional (required for HTTPS cert with domain)
- GitHub repo pushed with latest code

## 2. Connect To Server

From your local machine:

```bash
ssh root@YOUR_SERVER_IP
```

## 3. Update OS And Install Dependencies

```bash
apt update && apt upgrade -y
apt install -y git python3 python3-venv python3-pip nginx ffmpeg libgl1 libglib2.0-0 build-essential
```

## 4. Clone Project

```bash
mkdir -p /opt
cd /opt
git clone YOUR_GITHUB_REPO_URL vyakt
cd /opt/vyakt
```

Example URL:

```text
https://github.com/YOUR_USERNAME/YOUR_REPO.git
```

## 5. Create Production .env

Create env file:

```bash
nano /opt/vyakt/.env
```

Paste and fill real values:

```env
FLASK_SECRET_KEY=change_this_to_a_long_random_secret
FLASK_DEBUG=False
FLASK_HOST=127.0.0.1
FLASK_PORT=8000

MONGO_URI=your_mongo_uri
MONGO_DB_NAME=gestura_learning
MONGO_APP_NAME=GesturaAI-Learning

GEMINI_API_KEY=your_gemini_api_key

VITE_EMAILJS_SERVICE_ID=
VITE_EMAILJS_TEMPLATE_ID=
VITE_EMAILJS_PUBLIC_KEY=
```

Save: Ctrl+O, Enter, Ctrl+X

## 6. Python Environment And Packages

```bash
cd /opt/vyakt
python3 -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install gunicorn
```

## 7. Quick Gunicorn Test

```bash
cd /opt/vyakt
source venv/bin/activate
gunicorn -w 3 -b 127.0.0.1:8000 app:app
```

In another SSH session, verify:

```bash
curl -I http://127.0.0.1:8000/
```

Stop test server with Ctrl+C.

## 8. Create systemd Service

```bash
nano /etc/systemd/system/vyakt.service
```

Paste:

```ini
[Unit]
Description=Vyakt Gunicorn Service
After=network.target

[Service]
User=root
WorkingDirectory=/opt/vyakt
EnvironmentFile=/opt/vyakt/.env
ExecStart=/opt/vyakt/venv/bin/gunicorn -w 3 -b 127.0.0.1:8000 app:app
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
systemctl daemon-reload
systemctl enable vyakt
systemctl start vyakt
systemctl status vyakt
```

## 9. Configure Nginx Reverse Proxy

Create site file:

```bash
nano /etc/nginx/sites-available/vyakt
```

### If using domain

```nginx
server {
	listen 80;
	server_name yourdomain.com www.yourdomain.com;

	location / {
		proxy_pass http://127.0.0.1:8000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

### If testing with server IP only

```nginx
server {
	listen 80;
	server_name _;

	location / {
		proxy_pass http://127.0.0.1:8000;
		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;
	}
}
```

Enable config:

```bash
ln -sf /etc/nginx/sites-available/vyakt /etc/nginx/sites-enabled/vyakt
nginx -t
systemctl reload nginx
```

## 10. Enable HTTPS (Domain Required)

Point domain DNS A records to server IP first, then run:

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

Auto-renew test:

```bash
certbot renew --dry-run
```

## 11. Verify Deployment

Open in browser:

- http://YOUR_SERVER_IP
- or https://yourdomain.com (if SSL done)

Check endpoints:

- /learning
- /api/v1/learning/state
- /api/v1/quests/today

## 12. Deploy Future Updates

When you push new code to GitHub:

```bash
cd /opt/vyakt
git pull origin main
source venv/bin/activate
pip install -r requirements.txt
systemctl restart vyakt
systemctl status vyakt
```

## 13. Logs And Troubleshooting

App service logs:

```bash
journalctl -u vyakt -f
```

Nginx error logs:

```bash
tail -f /var/log/nginx/error.log
```

Nginx access logs:

```bash
tail -f /var/log/nginx/access.log
```

### Common Issues

1. Port conflict:
   - Ensure Gunicorn binds only to 127.0.0.1:8000
2. 502 Bad Gateway:
   - `systemctl status vyakt`
   - check app env values in /opt/vyakt/.env
3. Module import errors:
   - activate venv and reinstall requirements
4. Mongo connection failures:
   - verify MONGO_URI and Atlas IP/network allowlist

## 14. Production Hardening (Recommended)

1. Create non-root deploy user and run service as that user
2. Disable password SSH login; keep key-based login only
3. Keep FLASK_DEBUG=False
4. Rotate exposed API keys/secrets
5. Add GitHub Actions auto-deploy after manual deploy is stable

