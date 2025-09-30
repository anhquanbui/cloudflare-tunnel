# Cloudflare Tunnel — Multi-App on One VPS (example.com)

## Introduction
This guide shows how to put multiple self-hosted apps (Jupyter, VS Code Server, Portainer, and anything else on your VPS) behind **Cloudflare Tunnel** and serve them on friendly subdomains like `jupyter.example.com` and `vscode.example.com` — **without opening any inbound ports** on your server.
- **What Cloudflare Tunnel does:** a small daemon (`cloudflared`) makes an **outbound-only** connection from your VPS to Cloudflare’s edge. Traffic from the public Internet hits Cloudflare first and is then proxied down the tunnel to your local services.
- **Why it’s great:** free SSL/TLS, no public firewall holes, works behind NAT/CGNAT, easy to add/remove subdomains, and optional **Zero Trust / Access** for SSO (Google/GitHub/One-time PIN) in front of sensitive dashboards.
- **What you’ll build:** one tunnel + a single config file (`/etc/cloudflared/config.yml`) that routes multiple hostnames to different local ports (`localhost:8888`, `localhost:12000`, `localhost:9000`, …). You’ll also set up Jupyter to run as a non‑root systemd service.
- **Service config gotcha:** running `cloudflared` manually reads `~/.cloudflared/config.yml`, while the **systemd service** typically reads `/etc/cloudflared/config.yml`. This guide standardizes on **`/etc/cloudflared/config.yml`** to avoid confusion.

---

## 0) Prerequisites
- Cloudflare account (Free plan is fine) and a domain **added to Cloudflare** (nameservers set to Cloudflare).
- Linux VPS with `sudo`.
- Your apps already running **locally** on the VPS:
  - Jupyter: `http://localhost:8888`
  - VS Code Server: `http://localhost:12000`
  - Portainer: `http://localhost:9000` *(or HTTPS `https://localhost:9443`)*
- (Optional) `dnsutils` to check DNS: `sudo apt install -y dnsutils`

---

## 1) Install `cloudflared`
```bash
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb
cloudflared --version
```

---

## 2) Log in and create a Tunnel
```bash
cloudflared tunnel login               # opens browser; pick your domain (example.com)
cloudflared tunnel create my-main-tunnel
```
> This prints a **Tunnel ID** and creates credentials JSON at `~/.cloudflared/<TUNNEL_ID>.json`.
> You can leave it there or move it to `/etc/cloudflared/`.

---

## 3) Create the service config (`/etc/cloudflared/config.yml`)
```bash
sudo mkdir -p /etc/cloudflared
sudo tee /etc/cloudflared/config.yml >/dev/null <<'YAML'
tunnel: <TUNNEL_ID>
credentials-file: /etc/cloudflared/<TUNNEL_ID>.json   # or keep ~/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: jupyter.example.com
    service: http://localhost:8888

  - hostname: vscode.example.com
    service: http://localhost:12000

  # Portainer — PICK ONE:
  # A) Portainer HTTP 9000
  - hostname: portainer.example.com
    service: http://localhost:9000

  # B) Portainer HTTPS 9443 (self-signed)
  # - hostname: portainer.example.com
  #   service: https://localhost:9443
  #   originRequest:
  #     noTLSVerify: true

  - service: http_status:404
YAML
```

Validate the config:
```bash
sudo /usr/bin/cloudflared --config /etc/cloudflared/config.yml tunnel ingress validate
```

If you kept the credentials JSON in `~/.cloudflared/`, either change the `credentials-file` path above or copy it:
```bash
sudo cp ~/.cloudflared/<TUNNEL_ID>.json /etc/cloudflared/
sudo chmod 600 /etc/cloudflared/<TUNNEL_ID>.json
```

---

## 4) Create DNS records for your subdomains
```bash
cloudflared tunnel route dns <TUNNEL_ID> jupyter.example.com
cloudflared tunnel route dns <TUNNEL_ID> vscode.example.com
cloudflared tunnel route dns <TUNNEL_ID> portainer.example.com
```
> If you prefer a wildcard CNAME (`*.example.com → <TUNNEL_ID>.cfargotunnel.com`), you can skip per-subdomain DNS routes.

---

## 5) Run `cloudflared` as a systemd service
Install the service and make sure it points to `/etc/cloudflared/config.yml`:
```bash
sudo cloudflared service install
systemctl cat cloudflared     # verify ExecStart uses --config /etc/cloudflared/config.yml
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared
```

*(If needed, override the unit to force the config path:)*
```bash
sudo systemctl edit cloudflared
# Add:
# [Service]
# ExecStart=
# ExecStart=/usr/bin/cloudflared --no-autoupdate --config /etc/cloudflared/config.yml tunnel run
sudo systemctl daemon-reload
sudo systemctl restart cloudflared
```

---

## 6) Health checks
**Local backends:**
```bash
curl -I http://localhost:8888/lab      # Jupyter (expect 302/200)
curl -I http://localhost:12000         # VS Code Server
curl -I http://localhost:9000          # Portainer HTTP
# or: curl -kI https://localhost:9443
```

**Public (through Cloudflare):**
```bash
dig +short jupyter.example.com         # expect Cloudflare IPs 104.* / 172.*
curl -I https://jupyter.example.com
curl -I https://vscode.example.com
curl -I https://portainer.example.com
```

---

## 7) Add / Remove apps quickly

**Add `app1.example.com` → `http://localhost:8001`:**
1) Edit `/etc/cloudflared/config.yml` and add:
   ```yaml
   - hostname: app1.example.com
     service: http://localhost:8001
   ```
2) `cloudflared tunnel route dns <TUNNEL_ID> app1.example.com`
3) `sudo systemctl restart cloudflared`

**Remove an app:**
1) Delete its ingress block from `/etc/cloudflared/config.yml`
2) `sudo systemctl restart cloudflared`
3) Remove the DNS record from Cloudflare

---

## 8) JupyterLab under a non-root user (recommended)

**Create venv & install Jupyter:**
```bash
sudo adduser --disabled-password --gecos "" jupyter   # if not present
sudo -u jupyter -H python3 -m venv /home/jupyter/jupyterenv
sudo -u jupyter -H /home/jupyter/jupyterenv/bin/pip install --upgrade pip jupyterlab
```

**Generate an argon2 password hash:**
```bash
sudo -u jupyter -H /home/jupyter/jupyterenv/bin/python3 -c \
"from jupyter_server.auth import passwd; print(passwd('YourStrongPass'))"
# copy the printed 'argon2:...' string
```

**Systemd unit (stores the hash in the unit for simplicity):**
```bash
sudo tee /etc/systemd/system/jupyter.service >/dev/null <<'UNIT'
[Unit]
Description=Jupyter Lab (user: jupyter)
After=network.target

[Service]
Type=simple
User=jupyter
Group=jupyter
WorkingDirectory=/home/jupyter
ExecStart=/home/jupyter/jupyterenv/bin/jupyter lab \
  --ServerApp.ip=0.0.0.0 \
  --ServerApp.port=8888 \
  --ServerApp.open_browser=False \
  --ServerApp.port_retries=0 \
  --IdentityProvider.token='' \
  --ServerApp.identity_provider_class=jupyter_server.auth.identity.PasswordIdentityProvider \
  --PasswordIdentityProvider.hashed_password='argon2:PUT_HASH_HERE'
Restart=always
RestartSec=3s

[Install]
WantedBy=multi-user.target
UNIT

sudo systemctl daemon-reload
sudo systemctl enable --now jupyter
sudo systemctl status jupyter
```

*(Alternative: store the hash in `/home/jupyter/.jupyter/jupyter_server_config.py` and run Jupyter with `--config=...`.)*

---

## 9) Security tips
- **Do not** expose databases or internal-only services via Tunnel.
- Protect sensitive dashboards (e.g., Portainer, Grafana) with **Cloudflare Access** (Zero Trust → Access → Applications) to require Google/GitHub/OTP before the app.
- If your apps run in Docker and you run `cloudflared` natively, make sure the containers **publish** their ports to the host (`-p HOST:CONTAINER`). If you don’t want to publish, run `cloudflared` **in Docker** on the same bridge network and point services to container names.

---

## 10) Troubleshooting

- **NXDOMAIN** when opening a subdomain  
  → The DNS record doesn’t exist. Create it with:  
  `cloudflared tunnel route dns <TUNNEL_ID> sub.example.com`  
  Also clear client DNS cache if needed.

- **HTTP 404** from Cloudflare  
  → `cloudflared` didn’t match any ingress rule (likely the service is reading a different config).  
  Check which config the service uses: `systemctl cat cloudflared` (look for `--config /etc/cloudflared/config.yml`).  
  Validate: `cloudflared --config /etc/cloudflared/config.yml tunnel ingress validate`.

- **HTTP 502**  
  → Backend isn’t listening or not reachable (e.g., Docker container not published).  
  Check: `ss -lntp | grep :PORT`, `curl -I http://localhost:PORT`.

- **Works when running manually, fails as a service**  
  → Paths differ. Running manually reads `~/.cloudflared/config.yml`; the service typically reads `/etc/cloudflared/config.yml`. Align them or override the unit.

---

> Replace every `<TUNNEL_ID>` with your real Tunnel ID. Whenever you edit `/etc/cloudflared/config.yml`, run:  
> `sudo systemctl restart cloudflared`

Happy tunneling! 🚀
