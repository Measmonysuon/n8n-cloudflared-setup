
# n8n + PostgreSQL + Cloudflare Tunnel Setup

This setup allows you to run n8n with PostgreSQL behind a secure Cloudflare Tunnel on a Linux server.

## Prerequisites

- Docker & Docker Compose
- A domain name (e.g. `example.com`) managed by Cloudflare
- cloudflared installed and authenticated (`cloudflared tunnel login`)

## Folder Structure

```
.
├── docker-compose.yml
├── cloudflared.service
├── config.yml
└── README.md
```

---

## 1. Setup PostgreSQL + n8n via Docker

**docker-compose.yml**
```yaml
version: '3.1'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: n8n
      POSTGRES_PASSWORD: n8npass
      POSTGRES_DB: n8n
    volumes:
      - pgdata:/var/lib/postgresql/data

  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      DB_TYPE: postgresdb
      DB_POSTGRESDB_HOST: postgres
      DB_POSTGRESDB_PORT: 5432
      DB_POSTGRESDB_DATABASE: n8n
      DB_POSTGRESDB_USER: n8n
      DB_POSTGRESDB_PASSWORD: n8npass
      N8N_BASIC_AUTH_ACTIVE: true
      N8N_BASIC_AUTH_USER: admin
      N8N_BASIC_AUTH_PASSWORD: adminpass
    depends_on:
      - postgres
    volumes:
      - ./n8n_data:/home/node/.n8n

volumes:
  pgdata:
```

Run with:
```bash
docker compose up -d
```

---

## 2. Configure Cloudflare Tunnel

### config.yml

```yaml
tunnel: <your-tunnel-id>
credentials-file: /home/user/.cloudflared/<your-tunnel-id>.json

ingress:
  - hostname: n8n.example.com
    service: http://localhost:5678
  - service: http_status:404
```

Replace:
- `<your-tunnel-id>` with your actual tunnel ID
- `n8n.example.com` with your subdomain

### cloudflared systemd service

**cloudflared.service**
```ini
[Unit]
Description=cloudflared
After=network-online.target
Wants=network-online.target

[Service]
User=user
WorkingDirectory=/home/user
TimeoutStartSec=0
ExecStart=/usr/local/bin/cloudflared --no-autoupdate tunnel run n8n
Restart=on-failure
RestartSec=5s

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable cloudflared
sudo systemctl start cloudflared
```

---

## 3. Test Your Setup

Check if the tunnel is live:
```bash
cloudflared tunnel list
```

Access your n8n instance at:
```
https://n8n.example.com
```

If you get a 502/530 error, verify that n8n is running and tunnel DNS is correct.

---

✅ You're done!
