# Server Access & Deployment Guide

A step-by-step guide for securely accessing a Linux server and deploying a FastAPI + React application.

---

## Prerequisites

- Your SSH private key file (e.g. `id_ed25519`) provided by the server admin
- The server's IP address and your username (provided by admin)
- Git, Node.js, and Python installed on your local machine

---

## 1. Connecting to the Server

### Fix key file permissions (required — SSH will reject the key otherwise)

**PowerShell (Windows):**
```powershell
icacls "C:\path\to\id_ed25519" /inheritance:r /grant:r "YOUR_WINDOWS_USERNAME:R"
```

**Bash/Linux/Mac:**
```bash
chmod 600 /path/to/id_ed25519
```

### Connect via SSH
```bash
ssh -i /path/to/id_ed25519 username@server-ip
```

> If prompted "Are you sure you want to continue connecting?" type `yes`.

### If you land in a restricted shell (rbash)
```bash
bash
```

---

## 2. Uploading Your Project

Run this from your **local machine** (not the SSH session):

```bash
scp -O -i /path/to/id_ed25519 -r /path/to/your/project username@server-ip:~/project-name
```

- `-O` forces the legacy SCP protocol (more compatible)
- `-r` copies the entire folder recursively

---

## 3. Installing Python Dependencies

On the server:

```bash
cd ~/project-name
python3 -m venv venv          # create isolated Python environment
source venv/bin/activate       # activate it
pip install -r requirements.txt  # install dependencies
```

---

## 4. Creating the .env File

Your app needs secret keys (API keys, database credentials). Create the file manually on the server — never commit it to git:

```bash
nano ~/project-name/.env
```

Add your variables:
```
OPENAI_API_KEY=your_key_here
DATABASE_URL=your_db_url_here
```

Save with `Ctrl+O`, exit with `Ctrl+X`.

> **Security tip:** Never share or commit your `.env` file. If a key is accidentally exposed, rotate it immediately.

---

## 5. Building the Frontend (React/Vite)

If the server lacks Node.js, build locally and upload:

**Build locally:**
```bash
cd /path/to/project/frontend
VITE_API_URL=http://server-ip:8001 npm run build
```

**Upload the built files:**
```bash
scp -O -i /path/to/id_ed25519 -r /path/to/project/frontend/dist username@server-ip:~/project-name/frontend/
```

---

## 6. Running the Backend

### Run in foreground (for testing):
```bash
cd ~/project-name
source venv/bin/activate
uvicorn api:app --host 0.0.0.0 --port 8001
```

### Run in background (persistent):
```bash
nohup uvicorn api:app --host 0.0.0.0 --port 8001 > server.log 2>&1 &
```

### Check logs:
```bash
cat ~/project-name/server.log
```

### Stop the backend:
```bash
pkill -f "uvicorn api:app"
```

---

## 7. Serving the Frontend

### Temporary (for testing, no Nginx):
```bash
cd ~/project-name/frontend/dist
nohup python3 -m http.server 3000 > ~/project-name/frontend.log 2>&1 &
```

### Stop the frontend server:
```bash
pkill -f "http.server 3000"
```

---

## 8. Nginx Configuration (requires sudo/admin)

To serve at `http://server-ip/chat`, the server admin needs to:

1. Install Nginx:
```bash
sudo apt install nginx
```

2. Create a config file at `/etc/nginx/sites-available/app`:
```nginx
server {
    listen 80;

    location /chat {
        alias /home/username/project-name/frontend/dist;
        index index.html;
        try_files $uri $uri/ /chat/index.html;
    }

    location /api {
        proxy_pass http://localhost:8001;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

3. Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

4. Open port 80 in the AWS security group (via AWS console).

---

## Security Checklist

- [ ] Never commit `.env` files to git
- [ ] Keep your SSH private key secure — never share it
- [ ] Rotate API keys immediately if accidentally exposed
- [ ] Use strong, unique passwords
- [ ] Only open necessary ports in the firewall
- [ ] Run the app as a non-root user (default SSH user is fine)
