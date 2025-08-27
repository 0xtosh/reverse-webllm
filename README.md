# Setup guide for litellm/webui web frontend
This guide is for setting up your own litellm/webui frontend for your own local or cloud llm models via their APIs. Runs on a local machine with a cloud based web front end / reverse proxy using an outbound reverse ssh tunnel.

---

## 🌩️ Cloudnode

### 1. Prepare SSH tunnel receiver

```bash
sudo apt-get update
sudo apt install autossh
useradd tunnel
cd /home/tunnel
mkdir .ssh
cd .ssh
ssh-keygen -t rsa -b 4096 -f tunnel
mv tunnel.pub authorized_keys
```

Restrict key usage to tunneling only:

```bash
sudo sed -i 's/^/no-pty,no-X11-forwarding,no-agent-forwarding,command="sleep infinity" /' authorized_keys
chown -R tunnel:tunnel /home/tunnel/.ssh
```

✅ Copy the **`tunnel`** private key to the localnode at `/home/tunnel/.ssh/tunnel` then get rid of the private key

```bash
shred tunnel
rm tunnel
```

---

### 2. Configure Nginx reverse proxy

Create `/etc/nginx/sites-available/chat.mydomain.com`:

```nginx
server {
    if ($host = chat.mydomain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    listen [::]:80;
    server_name chat.mydomain.com;

    location /.well-known/acme-challenge/ {
        root /var/www/html;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;

    server_name chat.mydomain.com;
    ssl_certificate /etc/letsencrypt/live/chat.mydomain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/chat.mydomain.com/privkey.pem; # managed by Certbot

    location / {
        proxy_pass http://localhost:8081;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable and reload Nginx:

```bash
sudo ln -s /etc/nginx/sites-available/chat.mydomain.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo certbot --nginx
sudo systemctl reload nginx
```

---

## 💻 Localnode

### 1. Outbound SSH tunnel setup

```bash
useradd tunnel
cd /home/tunnel
mkdir .ssh
cd .ssh
# Copy "tunnel" private key from cloudnode here
chown -R tunnel:tunnel /home/tunnel/.ssh
chmod 400 /home/tunnel/.ssh/tunnel
```

Create service file `/etc/systemd/system/chatbot-tunnel.service`:

```ini
[Unit]
Description=SSH Tunnel for Chatbot
After=network.target

[Service]
User=tunnel
ExecStart=/usr/bin/autossh -M 0 -N -i /home/tunnel/.ssh/tunnel   -o "ServerAliveInterval=30"   -o "ServerAliveCountMax=3"   -o "ExitOnForwardFailure=yes"   -R 8081:localhost:8080 tunnel@cloudnode
RestartSec=5
Restart=always

[Install]
WantedBy=multi-user.target
```
Do the first login manually in order to make sure you can accept the key:

```ssh -i /home/tunnel/.ssh/tunnel -u tunnel tunnel@cloudnode```

Enable service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable chatbot-tunnel.service
sudo systemctl start chatbot-tunnel.service
```

---

### 2. Install Docker & Chatbot stack

Install Docker:

```bash
curl -sSL https://get.docker.com | sh
sudo usermod -aG docker ubuntu
```

Setup chatbot directory:

```bash
mkdir /opt/chatbot
cd /opt/chatbot
echo "GEMINI_API_KEY=YOUR_API_KEY_HERE" > .env
```

#### `config.yaml`
```yaml
model_list:
  - model_name: gemini-2.5-pro
    litellm_params:
      model: gemini/gemini-2.5-pro
      api_key: os.environ/GEMINI_API_KEY

general_settings:
  master_key: sk-123456789-abcdefg
```

#### `docker-compose.yml`
```yaml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-stable
    container_name: litellm_proxy
    command: --config /config.yaml --port 8000
    ports:
      - "8000:8000"
    environment:
      - GEMINI_API_KEY=${GEMINI_API_KEY}
    volumes:
      - ./config.yaml:/config.yaml
    restart: unless-stopped

  open-webui:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open_webui
    ports:
      - "8080:8080"
    volumes:
      - ./open-webui:/app/backend/data
    environment:
      - 'OPENAI_API_BASE_URL=http://litellm:8000/v1'
      - 'OPENAI_API_KEY=sk-123456789-abcdefg'
    depends_on:
      - litellm
    restart: unless-stopped
```

Run services:

```bash
docker compose up -d
```

---

## ✅ Testing

- Local test: [http://<LOCALNODE_IP>:8080](http://<LOCALNODE_IP>:8080)  
- Remote access: [https://chat.mydomain.com](https://chat.mydomain.com) → forwards to localnode WebUI  

---
