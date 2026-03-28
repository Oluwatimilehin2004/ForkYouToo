# ForkYouToo — Frontend

> The browser interface for ForkYouToo, an ALU GitHub repository explorer.

This repository contains the complete frontend — four HTML files with all CSS and JavaScript written inline. No build tools, no frameworks, no dependencies to install.

---

## 📄 Pages

| File | Purpose |
|------|---------|
| `index.html` | Main repo explorer — browse, search, filter, sort ALU repos |
| `login.html` | JWT login form |
| `signup.html` | User registration with ALU email validation |
| `imports.html` | View and manage your imported repositories |

---

## 🔗 Backend

This frontend talks to the ForkYouToo Django backend API.
Backend repo: https://github.com/Oluwatimilehin2004/ForkYouToo-Backend

By default the API URLs point to `http://127.0.0.1:8000` for local development.
For production, update the API URL constants at the top of each HTML file to point to your load balancer address.

---

## 🚀 Running Locally

No installation needed. Just open `index.html` in your browser, or use VS Code Live Server.

Make sure the backend server is running first:
```
http://127.0.0.1:8000
```

---

## 🌐 Production API URL

In each HTML file, find and update this line to your load balancer's IP or domain:

```js
// index.html, imports.html
const API_URL = 'http://<LB01_IP>/api/alu-repos/';
const IMPORT_API_URL = 'http://<LB01_IP>/api/import/';

// login.html
const API_LOGIN_URL = 'http://<LB01_IP>/api/login/';

// signup.html
const API_URL = 'http://<LB01_IP>/api/register/';
```

---

## 🚢 Deployment

The frontend is served as static files by Nginx on Web01 and Web02.
The load balancer (Lb01 with HAProxy) distributes traffic between them.

See the backend README for full deployment instructions:
https://github.com/Oluwatimilehin2004/ForkYouToo-Backend

---

## 🔐 Security Notes

- No API keys or secrets exist in this repo
- All authentication is handled via JWT tokens stored in `localStorage`
- All repo data rendered to the DOM is passed through `escapeHtml()` to prevent XSS

---

## 📄 License

MIT — built for educational purposes as part of ALU coursework.