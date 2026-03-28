# ForkYouToo — ALU Repository Explorer

> Discover, explore, and fork real GitHub repositories from ALU students and alumni.

ForkYouToo solves a real problem in the ALU community: finding high-quality, recent student projects to learn from, build on, or contribute to — without manually searching GitHub. Users can browse, filter, sort, search, and fork repos directly into their own GitHub account in one click.

---

## 🎯 Purpose

ALU students often want to:
- Find reference implementations for coursework
- Learn from peers' code style and architecture
- Fork and adapt existing student projects

ForkYouToo aggregates all ALU-related GitHub repositories (updated in the last year), presents them in a clean searchable interface, and lets authenticated users fork any repo directly to their GitHub account with optional renaming and attribution.

---

## ✨ Features

- **Browse** — paginated infinite-scroll grid of ALU repos
- **Search** — live search across name, description, and language (debounced)
- **Filter** — filter by programming language
- **Sort** — sort by Most Recent, ⭐ Stars, or 🍴 Forks
- **Fork/Import** — fork any repo to your GitHub account with one click
- **Rename** — optionally rename the forked repo
- **Attribution** — automatically adds an import note to the README
- **Import History** — dedicated page showing all your imported repos with GitHub links
- **Authentication** — JWT-based register/login with ALU email validation
- **Caching** — GitHub API results cached for 30 minutes (Django in-memory cache)
- **Parallel fetching** — 14 GitHub search queries run concurrently using ThreadPoolExecutor

---

## 🔌 External APIs Used

| API | Purpose | Documentation |
|-----|---------|---------------|
| [GitHub REST API v3](https://docs.github.com/en/rest) | Search repositories, fork repos, update files | https://docs.github.com/en/rest |
| GitHub Search API | Find ALU-related repos by name, topic, description | https://docs.github.com/en/rest/search |
| GitHub Contents API | Read/write README and settings files | https://docs.github.com/en/rest/repos/contents |

*Credit: GitHub API provided by GitHub, Inc.*

---

## 🚀 Running Locally

### Prerequisites

- Python 3.10+
- pip
- Git

### 1. Clone the repository

```bash
git clone https://github.com/Oluwatimilehin2004/ForkYouToo-Backend.git
cd ForkYouToo-Backend
```

### 2. Create and activate virtual environment

```bash
python -m venv env

# Windows
env\Scripts\activate

# macOS / Linux
source env/bin/activate
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Set up environment variables

```bash
cp .env.example .env
```

Edit `.env` and fill in your values:

```
SECRET_KEY=your-django-secret-key
DEBUG=True
GITHUB_TOKEN=your-github-personal-access-token
```

**Getting a GitHub token:**
1. Go to https://github.com/settings/tokens
2. Generate new token (classic)
3. Select scopes: `repo`, `user`
4. Copy the token into your `.env`

### 5. Run migrations

```bash
python manage.py migrate
```

### 6. Start the development server

```bash
python manage.py runserver
```

### 7. Open the frontend

Open `index.html` in your browser (or serve it with Live Server in VS Code).

Make sure the API URLs in the HTML files point to `http://127.0.0.1:8000`.

### 8. Settings.py cache configuration

Add this to your `settings.py` (no extra installs required):

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    }
}
```

---

## 🗂️ Project Structure

```
ForkYouToo-Backend/
├── userapp/
│   ├── models.py          # UserProfile, ImportHistory
│   ├── views.py           # All API endpoints
│   ├── urls.py            # URL routing
│   ├── admin.py
│   └── apps.py
├── services/
│   └── import_service.py  # GitHub fork/customize logic
├── frontend/
│   ├── index.html         # Main repo explorer
│   ├── login.html         # Login page
│   ├── signup.html        # Registration page
│   └── imports.html       # Import history page
├── .env.example           # Environment variable template
├── .gitignore
├── manage.py
└── requirements.txt
```

---

## 🔐 Security

- API keys stored in `.env`, never committed to version control
- `.env` is listed in `.gitignore`
- JWT tokens used for all authenticated endpoints
- All user inputs validated on both frontend and backend
- Django ORM used for all database queries (prevents SQL injection)
- `escapeHtml()` used on all repo data before DOM insertion (prevents XSS)
- GitHub tokens verified against the GitHub API before being stored

---

## 🚢 Deployment (Part Two)

### Architecture

```
Internet → Load Balancer (Lb01) → Web01
                                → Web02
```

### Server Setup (Web01 and Web02)

Run on each server:

```bash
# Install dependencies
sudo apt update
sudo apt install python3-pip python3-venv nginx -y

# Clone project
git clone https://github.com/Oluwatimilehin2004/ForkYouToo-Backend.git /var/www/forkyoutoo
cd /var/www/forkyoutoo

# Virtual environment
python3 -m venv env
source env/bin/activate
pip install -r requirements.txt
pip install gunicorn

# Environment variables
cp .env.example .env
nano .env  # fill in real values

# Migrate
python manage.py migrate

# Collect static files
python manage.py collectstatic --noinput
```

### Gunicorn Service

Create `/etc/systemd/system/forkyoutoo.service`:

```ini
[Unit]
Description=ForkYouToo Gunicorn
After=network.target

[Service]
User=ubuntu
WorkingDirectory=/var/www/forkyoutoo
ExecStart=/var/www/forkyoutoo/env/bin/gunicorn \
    --workers 3 \
    --bind 0.0.0.0:8000 \
    forkyoutoo.wsgi:application
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable forkyoutoo
sudo systemctl start forkyoutoo
```

### Nginx Configuration (Web01 and Web02)

`/etc/nginx/sites-available/forkyoutoo`:

```nginx
server {
    listen 80;
    server_name _;

    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location / {
        root /var/www/forkyoutoo/frontend;
        index index.html;
        try_files $uri $uri/ /index.html;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/forkyoutoo /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Load Balancer Configuration (Lb01)

`/etc/nginx/sites-available/forkyoutoo-lb`:

```nginx
upstream forkyoutoo_backend {
    server <WEB01_IP>:80;
    server <WEB02_IP>:80;
}

server {
    listen 80;

    location / {
        proxy_pass http://forkyoutoo_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/forkyoutoo-lb /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### Verifying Load Balancing

```bash
# Hit the load balancer multiple times and check which server responds
curl http://<LB01_IP>/api/alu-repos/
```

Both Web01 and Web02 should alternately handle requests.

---

## ⚠️ Challenges & Solutions

| Challenge | Solution |
|-----------|----------|
| GitHub API returns 500 repos but page load took 30+ seconds | Switched to parallel `ThreadPoolExecutor` + 30-minute cache |
| Infinite scroll loaded all 500 items before showing anything | Moved to true server-side pagination (`?page=&per_page=`) |
| Redis not installed but was configured as cache backend | Switched to Django's built-in `LocMemCache` (zero setup) |
| GitHub push protection blocked `.env` with real token | Rewrote commit history with `git filter-branch`, revoked old token |
| `import_status` and `my_imports` queried wrong FK | Fixed `user=request.user.profile` → `user=request.user` |

---

## 📦 Dependencies

```
django
djangorestframework
djangorestframework-simplejwt
requests
Pillow
python-decouple
gunicorn
```

---

## 📄 License

MIT — built for educational purposes as part of ALU coursework.