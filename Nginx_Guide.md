# Nginx — What It Is and How It Works

## What is Nginx?

Nginx (pronounced "engine-x") is a web server that sits **in front of your app**. When someone visits your site, the request hits Nginx first — Nginx then decides where to send it.

For your stack:
- **Next.js** runs on port `3000`
- **FastAPI (Python)** runs on port `8000`
- **Nginx** listens on port `80` (HTTP) or `443` (HTTPS) and routes traffic to the right service

---

## Why Do You Need It?

Without Nginx, users would have to type:
```
http://yourserver.com:3000   → frontend
http://yourserver.com:8000   → backend API
```

With Nginx, everything goes through one clean address:
```
http://yourserver.com        → frontend (Next.js)
http://yourserver.com/api/   → backend  (FastAPI)
```

This is called a **reverse proxy** — Nginx proxies requests *on behalf of* the client to your internal services.

---

## How It Works (Your Stack)

```
User Browser
     |
     | HTTP request to yourserver.com
     v
  [ Nginx :80 ]
     |            |
     |            |--- /api/*  ---->  FastAPI :8000
     |
     |--- /*  --------->  Next.js :3000
```

1. Request comes in from the internet
2. Nginx checks the URL path
3. If the path starts with `/api/`, it forwards the request to FastAPI
4. Everything else goes to Next.js
5. Nginx sends the response back to the user

---

## Basic Nginx Config for Your Stack

File location on your server: `/etc/nginx/sites-available/myapp`

```nginx
server {
    listen 80;
    server_name yourserver.com;

    # Forward all /api/ requests to FastAPI
    location /api/ {
        proxy_pass http://localhost:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Forward everything else to Next.js
    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t          # test config for errors
sudo systemctl reload nginx
```

---

## Key Concepts

| Term | Meaning |
|---|---|
| **Reverse proxy** | Nginx forwards client requests to backend services on their behalf |
| **proxy_pass** | Tells Nginx which internal service to forward the request to |
| **location block** | Matches URL paths to decide where to route |
| **upstream** | The internal app server (your Next.js or FastAPI process) |
| **Port 80/443** | Standard web ports — users don't have to type a port number |

---

## What Nginx Also Handles

Beyond routing, Nginx gives you:

- **SSL/TLS (HTTPS)** — terminates HTTPS so your apps don't have to deal with certificates
- **Load balancing** — split traffic across multiple app instances if you scale up
- **Static file serving** — can serve images/CSS directly without touching your app
- **Rate limiting** — block abusive clients
- **Gzip compression** — compresses responses to make your site faster

---

## Deployment Flow Summary

```
1. Start FastAPI    →  uvicorn main:app --port 8000
2. Start Next.js   →  npm run start  (runs on port 3000)
3. Nginx routes    →  /api/* to FastAPI, /* to Next.js
4. (Optional) Certbot adds HTTPS to your Nginx config
```

---

## HTTPS with Certbot (Free SSL)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourserver.com
```

Certbot automatically edits your Nginx config to redirect HTTP → HTTPS and handles certificate renewal.
