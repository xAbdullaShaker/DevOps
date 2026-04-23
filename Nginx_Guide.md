# Nginx — What It Is and How It Works

![Nginx Reverse Proxy Architecture](https://miro.medium.com/1*Gm_q3hi9cBRFeGA1NoxowQ.png)

## What is Nginx?

Nginx (pronounced "engine-x") is a web server that sits **in front of your app**. When someone visits your site, the request hits Nginx first — Nginx then decides where to send it.

For your stack:
- **React/Next.js** runs on port `3000`
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
http://yourserver.com        → frontend (React/Next.js)
http://yourserver.com/api/   → backend  (FastAPI)
```

This is called a **reverse proxy** — Nginx proxies requests *on behalf of* the client to your internal services.

---

## How It Works (Your Stack)

```
User Browser
     |
     | HTTP/HTTPS request to yourserver.com
     v
  [ Nginx :80 / :443 ]
     |            |
     |            |--- /api/*  ---->  FastAPI :8000
     |
     |--- /*  --------->  React/Next.js :3000
```

1. Request comes in from the internet
2. Nginx checks the URL path
3. If the path starts with `/api/`, it forwards the request to FastAPI
4. Everything else goes to React/Next.js
5. Nginx sends the response back to the user

---

## Production Nginx Config for Your Stack

File location on your server: `/etc/nginx/sites-available/myapp`

```nginx
# ── Rate limiting zone (declare outside the server block) ────────────────────
# Allows 10 requests/sec per IP. Declared once, used in location blocks below.
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

# ── Redirect HTTP → HTTPS ─────────────────────────────────────────────────────
server {
    listen 80;
    server_name yourserver.com;
    return 301 https://$host$request_uri;
}

# ── Main HTTPS server ─────────────────────────────────────────────────────────
server {
    listen 443 ssl;
    server_name yourserver.com;

    # SSL certificates (Certbot fills these in automatically)
    ssl_certificate     /etc/letsencrypt/live/yourserver.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourserver.com/privkey.pem;

    # ── Security headers ──────────────────────────────────────────────────────
    add_header X-Frame-Options           "SAMEORIGIN";
    add_header X-Content-Type-Options    "nosniff";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

    # ── Gzip compression ──────────────────────────────────────────────────────
    # Compresses JSON and text responses before sending — reduces bandwidth
    gzip on;
    gzip_types text/plain application/json text/event-stream text/css application/javascript;

    # ── FastAPI — all /api/* requests ─────────────────────────────────────────
    location /api/ {
        proxy_pass         http://127.0.0.1:8000/;
        proxy_http_version 1.1;              # required for keepalive and SSE

        # Disable buffering — CRITICAL for Server-Sent Events (SSE) streaming.
        # Without this, Nginx holds all tokens in memory and flushes at once,
        # breaking the typewriter/streaming effect in the frontend.
        proxy_buffering    off;
        proxy_cache        off;
        add_header         X-Accel-Buffering no;  # belt-and-suspenders for SSE

        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;  # lets FastAPI know it's HTTPS

        # Raise timeout — LLM streaming responses can run for many seconds
        proxy_read_timeout 300s;

        # Rate limiting — allow bursts of 20, no delay on burst
        limit_req zone=api_limit burst=20 nodelay;
    }

    # ── React/Next.js — everything else ──────────────────────────────────────
    location / {
        proxy_pass       http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t          # test config for errors — always run this before reload
sudo systemctl reload nginx
```

---

## Key Concepts

| Term | Meaning |
|---|---|
| **Reverse proxy** | Nginx forwards client requests to backend services on their behalf |
| **proxy_pass** | Tells Nginx which internal service to forward the request to |
| **location block** | Matches URL paths to decide where to route |
| **upstream** | The internal app server (your React or FastAPI process) |
| **Port 80/443** | Standard web ports — users don't have to type a port number |
| **proxy_buffering off** | Disables response buffering — required for SSE/streaming APIs |
| **limit_req_zone** | Defines a rate-limiting bucket by IP address |

---

## What Nginx Also Handles

Beyond routing, Nginx gives you:

- **SSL/TLS (HTTPS)** — terminates HTTPS so your apps don't have to deal with certificates
- **Load balancing** — split traffic across multiple app instances if you scale up
- **Static file serving** — can serve images/CSS directly without touching your app
- **Rate limiting** — block abusive clients at the network level, before they hit Python
- **Gzip compression** — compresses responses to make your site faster
- **Security headers** — adds HTTP security headers to every response automatically

---

## SSE Streaming — Why `proxy_buffering off` Matters

If your backend uses **Server-Sent Events (SSE)** to stream tokens to the browser (e.g. a chat typewriter effect), Nginx will silently break it unless you disable buffering.

**Default behaviour (buffering on):**
```
FastAPI sends token → Nginx buffers it → ... waits ... → flushes all at once
```
The user sees nothing, then gets the full answer in one block — no streaming.

**Correct behaviour (buffering off):**
```
FastAPI sends token → Nginx forwards immediately → Browser renders it instantly
```

Always add these three lines inside any `location` block that proxies an SSE endpoint:
```nginx
proxy_buffering    off;
proxy_cache        off;
add_header         X-Accel-Buffering no;
```

And raise the read timeout so long-running streams aren't cut off:
```nginx
proxy_read_timeout 300s;
```

---

## Deployment Flow Summary

```
1. Start FastAPI    →  uvicorn api:app --host 127.0.0.1 --port 8000
2. Start React      →  npm run start  (runs on port 3000)
3. Nginx routes     →  /api/* to FastAPI, /* to React
4. Certbot adds HTTPS (see below)
```

> Bind your app servers to `127.0.0.1`, not `0.0.0.0`. This ensures only Nginx can reach them — they are never exposed directly to the internet.

---

## HTTPS with Certbot (Free SSL)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourserver.com
```

Certbot automatically edits your Nginx config to add the SSL certificate paths and redirects HTTP → HTTPS. It also sets up a cron job to auto-renew the certificate before it expires.
