# ForkYouToo — Frontend

> Discover, explore, and fork real GitHub repositories from ALU students and alumni — all in one place.

[![Demo Video](https://img.shields.io/badge/Watch-Demo%20Video-red?style=for-the-badge&logo=youtube)](https://your-video-link-here)

[![Live App](https://img.shields.io/badge/Live-wwwpapa.tech-blue?style=for-the-badge&logo=globe)](https://wwwpapa.tech)

---

## Backend

This frontend talks to the ForkYouToo Django backend API.
Backend repo: https://github.com/Oluwatimilehin2004/ForkYouToo-Backend

By default the API URLs point to `http://127.0.0.1:8000` for local development.
For production, it was updated to the API URL constants at the top of each HTML file to point to your load balancer address(https://wwwpapa.tech).

---

## Table of Contents

- [Overview](#overview)
- [Pages](#pages)
- [Features](#features)
- [Running Locally](#running-locally)
- [Infrastructure & Deployment](#infrastructure--deployment)
  - [Architecture Overview](#architecture-overview)
  - [Request Flow](#request-flow)
  - [Web Server Setup](#web-server-setup-web01--web02)
  - [Load Balancer Setup](#load-balancer-setup-lb01)
  - [SSL/TLS Certificate](#ssltls-certificate)
  - [Custom Response Header](#custom-response-header)
- [Security](#security)
- [API Credits](#api-credits)
- [Challenges & Solutions](#challenges--solutions)

---

## Overview

ForkYouToo is a full-stack web application that solves a real problem in the ALU community — finding, exploring, and reusing high-quality student GitHub projects without manually searching through GitHub.

Users can browse all ALU-related repos updated in the last year, filter by language, sort by recency/stars/forks, search live, and fork any repo directly into their own GitHub account with one click — including optional renaming and automatic attribution.

This repository contains the **static frontend** — four plain HTML files with all CSS and JavaScript written inline. No build tools, no frameworks, no dependencies.

For the backend API, see: **[ForkYouToo-Backend](https://github.com/Oluwatimilehin2004/ForkYouToo-Backend)**

---

## Pages

| File | Purpose |
|------|---------|
| `index.html` | Main explorer — browse, search, filter, sort ALU repos with infinite scroll |
| `login.html` | JWT-based login form with validation |
| `signup.html` | User registration with ALU email validation |
| `imports.html` | View and manage all your imported/forked repositories |

---

## Features

- **Browse** — paginated infinite-scroll grid of ALU repos (updated in the last year)
- **Live Search** — debounced search across repo name, description, and language
- **Filter** — filter by programming language (Python, JavaScript, Java, TypeScript, Rust)
- **Sort** — sort by Most Recent, Stars, or Forks
- **Fork/Import** — fork any repo to your GitHub account in one click
- **Rename** — optionally rename the forked repo on import
- **Import History** — dedicated page showing all your imports with direct GitHub links
- **JWT Authentication** — secure register/login with token-based sessions
- **Auto URL Detection** — automatically uses local or production API based on where the app is accessed from

---

## Running Locally

No installation needed for the frontend.

### Prerequisites
- The backend must be running. See [ForkYouToo-Backend](https://github.com/Oluwatimilehin2004/ForkYouToo-Backend) for setup instructions.

### Steps

**1. Clone this repo**
```bash
git clone https://github.com/Oluwatimilehin2004/ForkYouToo-Frontend.git
cd ForkYouToo-Frontend
```

**2. Start the backend** (in a separate terminal)
```bash
cd ForkYouToo-Backend
source venv/bin/activate      # or env\Scripts\activate on Windows
python manage.py runserver
```

**3. Open the frontend**

Open `index.html` in your browser or use VS Code Live Server.

The frontend automatically detects whether it is running locally or in production:

```js
const _h = window.location.hostname;
const BASE_URL = (_h === '127.0.0.1' || _h === 'localhost' || _h === '')
  ? 'http://127.0.0.1:8000'       // local Django server
  : 'https://wwwpapa.tech';       // production server
```

No manual URL changes needed — it works in both environments automatically.

---

## Infrastructure & Deployment

### Architecture Overview

```
                    ┌─────────────────────────────────┐
                    │         CLIENT BROWSER           │
                    │     https://wwwpapa.tech         │
                    └────────────────┬────────────────┘
                                     │ HTTPS (port 443)
                                     ▼
                    ┌─────────────────────────────────┐
                    │         LB01 — HAProxy           │
                    │  SSL Termination (Let's Encrypt) │
                    │  Round-Robin Load Balancing      │
                    │  X-Served-By custom header       │
                    └──────────┬──────────────┬───────┘
                               │              │
               HTTP (port 80)  │              │  HTTP (port 80)
                               ▼              ▼
          ┌─────────────────────────┐  ┌─────────────────────────┐
          │   WEB01 — Nginx         │  │   WEB02 — Nginx         │
          │   Serves static HTML    │  │   Serves static HTML    │
          │   Proxies /api/ to      │  │   Proxies /api/ to      │
          │   Gunicorn :8000        │  │   Gunicorn :8000        │
          └────────────┬────────────┘  └────────────┬────────────┘
                       │                            │
                       ▼                            ▼
          ┌─────────────────────┐       ┌─────────────────────┐
          │  Gunicorn           │       │  Gunicorn           │
          │  (systemd service)  │       │  (systemd service)  │
          └──────────┬──────────┘       └──────────┬──────────┘
                     │                             │
                     ▼                             ▼
          ┌─────────────────────┐       ┌─────────────────────┐
          │  Django Application │       │  Django Application │
          │  GitHub REST API    │       │  GitHub REST API    │
          └─────────────────────┘       └─────────────────────┘
```

---

### Request Flow

```
1. User opens https://wwwpapa.tech in their browser
         │
         ▼
2. HAProxy on LB01 receives the request on port 443
   → Terminates SSL using the Let's Encrypt certificate
   → Decrypts the HTTPS request into plain HTTP
   → Adds X-Served-By header identifying which backend server will respond
         │
         ▼
3. HAProxy forwards plain HTTP to Web01 OR Web02 (round-robin, alternating)
         │
         ▼
4. Nginx on the selected web server receives the request
   → Path starts with /api/  →  proxy_pass to Gunicorn on port 8000
   → All other paths         →  serve static HTML files from /var/www/
         │
         ▼
5. Gunicorn receives the API request and forwards it to Django
   → Django processes the request
   → Calls the GitHub API if needed (results cached 30 mins)
   → Returns JSON response
         │
         ▼
6. Response travels back through Nginx → HAProxy → Client (encrypted)
```

---

### Web Server Setup (Web01 & Web02)

Both web servers are configured identically. Repeat all steps on each server.

**1. Clone the frontend into the web root**
```bash
cd /var/www
sudo git clone https://github.com/Oluwatimilehin2004/ForkYouToo.git forkyoutoo
sudo chown -R ubuntu:ubuntu /var/www/forkyoutoo
```

**2. Clone the backend**
```bash
cd /home/ubuntu
git clone https://github.com/Oluwatimilehin2004/ForkYouToo-Backend.git
cd ForkYouToo-Backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
nano .env   # fill in SECRET_KEY and GITHUB_TOKEN
python manage.py migrate
python manage.py collectstatic --noinput
```

**3. Configure Nginx**

Create `/etc/nginx/sites-available/forkyoutoo`:
```nginx
server {
    listen 80;
    server_name wwwpapa.tech www.wwwpapa.tech _;

    # Serve static frontend files
    root /var/www/forkyoutoo-frontend;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # Proxy all API requests to Gunicorn
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Django collected static files
    location /static/ {
        alias /home/ubuntu/ForkYouToo-Backend/staticfiles/;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/forkyoutoo /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

**4. Backend startup script**

Create `/home/ubuntu/ForkYouToo-Backend/startup.sh`:
```bash
#!/bin/bash
cd /home/ubuntu/ForkYouToo-Backend
source venv/bin/activate
gunicorn --workers 3 --bind 0.0.0.0:8000 fork_you_too.wsgi:application
```

```bash
chmod +x /home/ubuntu/ForkYouToo-Backend/startup.sh
```

**5. Backend systemd service**

The backend runs as a systemd service so it starts automatically on boot and never needs to be started manually.

Create `/etc/systemd/system/fork_you_too_backend.service`:
```ini
[Unit]
Description=ForkYouToo-Backend Service
After=network.target auditd.service

[Service]
User=ubuntu
ExecStart=/home/ubuntu/ForkYouToo-Backend/startup.sh > /var/log/fork_you_too_backend.log
KillMode=process

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable fork_you_too_backend
sudo systemctl start fork_you_too_backend
sudo systemctl status fork_you_too_backend   # must show: active (running)
```

---

### Load Balancer Setup (LB01)

HAProxy distributes incoming traffic between Web01 and Web02 using the **round-robin** algorithm — each request alternates between the two servers, ensuring even load distribution.

`/etc/haproxy/haproxy.cfg`:
```
global
    log /dev/log local0
    maxconn 2048
    daemon

defaults
    log global
    mode http
    option httplog
    option dontlognull
    timeout connect 5s
    timeout client  30s
    timeout server  30s

# Redirect all HTTP → HTTPS
frontend http_front
    bind *:80
    redirect scheme https code 301 if !{ ssl_fc }

# HTTPS — SSL terminates here at the load balancer
frontend https_front
    bind *:443 ssl crt /etc/ssl/forkyoutoo/forkyoutoo.pem
    default_backend web_servers

# Backend pool
backend web_servers
    balance roundrobin
    option httpchk GET /
    http-response set-header X-Served-By %[srv_name]
    server web01 <WEB01_IP>:80 check
    server web02 <WEB02_IP>:80 check
```

```bash
sudo haproxy -c -f /etc/haproxy/haproxy.cfg   # validate — must say: Configuration file is valid
sudo systemctl restart haproxy
sudo systemctl status haproxy                   # must show: active (running)
```

**Verify round-robin is working:**
```bash
curl -k -I https://wwwpapa.tech/
# Run 4 times — X-Served-By header should alternate between web01 and web02
```

---

### SSL/TLS Certificate

SSL is terminated at the load balancer (LB01). The certificate was obtained from **Let's Encrypt** — a free, trusted, globally recognised Certificate Authority.

```bash
# Install Certbot on LB01
sudo apt install certbot -y

# Obtain certificate covering both www and non-www
sudo certbot certonly --standalone \
    -d wwwpapa.tech \
    -d www.wwwpapa.tech

# Combine cert + private key into a single PEM file for HAProxy
sudo cat /etc/letsencrypt/live/wwwpapa.tech/fullchain.pem \
         /etc/letsencrypt/live/wwwpapa.tech/privkey.pem \
    | sudo tee /etc/ssl/forkyoutoo/forkyoutoo.pem
sudo chmod 600 /etc/ssl/forkyoutoo/forkyoutoo.pem

sudo systemctl restart haproxy
```

All traffic between the client and LB01 is encrypted via TLS 1.2/1.3.
Traffic between LB01 and the web servers travels over the private internal network as plain HTTP — this is standard SSL termination architecture and is secure because the internal network is not publicly accessible.

---

### Custom Response Header

Every response includes a custom `X-Served-By` header added by HAProxy, identifying which backend server handled the request:

```
X-Served-By: web01
X-Served-By: web02
```

Verify in browser DevTools → Network tab → any request → Response Headers, or:

```bash
curl -k -I https://wwwpapa.tech/
```

---

## Security

| Measure | Implementation |
|---------|---------------|
| HTTPS everywhere | Let's Encrypt TLS certificate on HAProxy, HTTP redirects to HTTPS |
| JWT Authentication | All API endpoints require a valid Bearer token |
| XSS Prevention | All repo data passed through `escapeHtml()` before DOM insertion |
| No secrets in repo | No API keys, tokens, or `.env` files committed — `.gitignore` enforced |
| Input validation | Frontend and backend both validate all user inputs |
| CORS | Django configured to only accept requests from known origins |
| SQL Injection | Django ORM used for all DB queries — no raw SQL |

---

## API Credits

| API | Used For | Documentation |
|-----|----------|---------------|
| **GitHub REST API v3** | Search repositories, fork repos, update file contents | [docs.github.com/en/rest](https://docs.github.com/en/rest) |
| **GitHub Search API** | Find ALU-related repos by name, topic, description | [docs.github.com/en/rest/search](https://docs.github.com/en/rest/search) |
| **GitHub Contents API** | Read and write README files for attribution | [docs.github.com/en/rest/repos/contents](https://docs.github.com/en/rest/repos/contents) |

*GitHub API is provided by GitHub, Inc. — [github.com](https://github.com)*

---

## Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| Loading 500 repos caused 30+ second page load | Parallel GitHub API fetching with `ThreadPoolExecutor` + 30-min in-memory cache |
| Infinite scroll did not help load time — all data fetched before anything rendered | Implemented true server-side pagination (`?page=&per_page=&sort=`) |
| Frontend used same hardcoded URL locally and in production | Auto-detect via `window.location.hostname` — works in both environments with no manual changes |
| Gunicorn had to be restarted manually after every server reboot | Created `fork_you_too_backend.service` systemd service — starts automatically on boot |
| HAProxy SSL certificate not covering `www` subdomain | Obtained Let's Encrypt cert with SAN covering both `wwwpapa.tech` and `www.wwwpapa.tech` |
| `.env` file accidentally committed with real GitHub token | Rewrote git history with `git filter-branch`, immediately revoked and rotated the compromised token |

---

## License

MIT — built for educational purposes as part of ALU coursework.



