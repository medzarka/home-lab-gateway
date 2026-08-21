# 🌐 Homelab Gateway Stack

A dedicated, highly secure, containerized ingress gateway and network router featuring **Traefik v3**, **Tailscale Mesh VPN**, **Cloudflare Zero Trust Tunnel**, and a read-only **Docker Socket Proxy**.

This stack acts as the universal network ingress layer for all services and applications across your homelab.

---

## 🏛️ Architecture Overview

```mermaid
graph TD
    PublicUser([Public Internet / Browser]) -->|HTTPS / DDoS Protected| Cloudflare[Cloudflare Edge / Zero Trust]
    Cloudflare -->|Outbound Encrypted Tunnel| Cloudflared[cloudflared Container]
    
    TailnetUser([Tailnet Client / Admin]) -->|WireGuard Encrypted| Tailscale[Tailscale Gateway]
    
    Cloudflared -->|shared_net:80| Traefik[Traefik v3 Ingress]
    Tailscale -->|Shares Network Stack| Traefik
    
    Traefik -->|Read-Only Socket Events| SocketProxy[Docker Socket Proxy]
    Traefik -->|Routes to Containers| Stacks[Homelab Stacks: Dockhand, Seafile, AI, Dev]
```

---

## 🔒 Security Highlights

1. **Dedicated Infrastructure Layer:** Operates independently of application containers. Upgrading, restarting, or deploying application stacks will never interrupt the gateway or drop network connectivity.
2. **Dual Ingress (Tailscale + Cloudflare):**
   - **Private Mesh:** Direct encrypted access via Tailscale (`https://<service>.homelab-gw.ts.net`).
   - **Public Edge:** Secure public access via Cloudflare Tunnel without opening any router ports or exposing your home IP.
3. **Docker Socket Proxy Isolation:** Traefik interacts with Docker through a locked-down, read-only TCP proxy (`socket-proxy:2375`) rather than mounting the raw `/var/run/docker.sock`.
4. **Universal Routing via `shared_net`:** Connected to `shared_net` so Traefik automatically discovers and routes to any container on the host with `traefik.*` labels.

---

## 🚀 Quick Start

### 1. Configure Environment
```bash
cp .env.example .env
nano .env
```

Set your configuration values:
- **`TS_AUTHKEY`**: Your Tailscale auth key from the [Tailscale Admin Console](https://login.tailscale.com/admin/settings/keys).
- **`TS_HOSTNAME`**: Hostname for the gateway on your tailnet (default: `homelab-gw`).
- **`CLOUDFLARE_TUNNEL_TOKEN`**: Your Tunnel token from [Cloudflare Zero Trust](https://one.dash.cloudflare.com/) (Networks $\rightarrow$ Tunnels).

---

### 2. Cloudflare Zero Trust Tunnel Setup

1. In the **Cloudflare Zero Trust Dashboard**, navigate to **Networks $\rightarrow$ Tunnels**.
2. Create a new Cloudflare Tunnel (e.g. `homelab-tunnel`) and paste the **Tunnel Token** into `CLOUDFLARE_TUNNEL_TOKEN` in `.env`.
3. Under the **Public Hostnames** tab of the tunnel:
   - **Public Hostname:** `*.yourdomain.com` (wildcard) or specific subdomains (`dockhand.yourdomain.com`, `seafile.yourdomain.com`, etc.)
   - **Service Type:** `HTTP`
   - **URL:** `gateway_tailscale:80` (or `gateway_traefik:80`)

---

### 3. Deploy the Gateway
```bash
docker compose up -d
```
The `init-volumes` container will automatically create all persistent storage paths under `${GATEWAY_HOME}/volumes` with correct permissions.

---

## 🔌 How to Expose Any Container with Traefik

To have Traefik automatically detect, route traffic to, and secure any Docker container in your homelab, follow these two simple requirements in the application's `docker-compose.yaml`:

### 1. Attach the Container to `shared_net`
The target container must join `shared_net` so Traefik can send HTTP packets to its internal IP.

### 2. Add Traefik Labels
Add Docker labels specifying the router name, hostname rule, and internal container port.

---

### 📋 Copy-Paste Template for Any Application

```yaml
services:
  my-app:
    image: my-app:latest
    container_name: my_app
    restart: unless-stopped
    
    # 1. Connect to the gateway shared network
    networks:
      - shared_net
      
    # 2. Configure Traefik Ingress Labels
    labels:
      # Enable Traefik discovery for this container
      - "traefik.enable=true"
      
      # Routing Domain Rule (Match Tailscale domain and/or Cloudflare public domain)
      - "traefik.http.routers.my-app.rule=Host(`app.homelab-gw.ts.net`) || Host(`app.yourdomain.com`)"
      
      # Entrypoint (websecure = HTTPS / 443)
      - "traefik.http.routers.my-app.entrypoints=websecure"
      - "traefik.http.routers.my-app.tls=true"
      
      # Internal port the container listens on (e.g. 80, 3000, 8080, 5000)
      - "traefik.http.services.my-app.loadbalancer.server.port=8080"
      
      # Explicitly specify the network Traefik should use to reach this container
      - "traefik.docker.network=shared_net"

networks:
  shared_net:
    name: shared_net
    external: true
```

---

### 💡 Label Configuration Explained

| Label | Description | Example |
| :--- | :--- | :--- |
| `traefik.enable=true` | Informs Traefik to create a reverse proxy route for this container. | `true` |
| `traefik.http.routers.<app>.rule` | Domain/Host matching rule. Can combine multiple domains with `\|\|`. | `Host(\`seafile.homelab-gw.ts.net\`) \|\| Host(\`seafile.domain.com\`)` |
| `traefik.http.routers.<app>.entrypoints` | Which port entrypoint to listen on (`websecure` for HTTPS :443, `web` for HTTP :80). | `websecure` |
| `traefik.http.routers.<app>.tls=true` | Enables TLS termination for the router. | `true` |
| `traefik.http.services.<app>.loadbalancer.server.port` | The **internal port** inside the container where the web server is listening. | `80`, `3000`, `8080`, `9085` |
| `traefik.docker.network=shared_net` | Directs Traefik to route over `shared_net` (vital if the container has multiple networks). | `shared_net` |

---

### 🛡️ Optional: Adding Middlewares (Basic Auth, Path Stripping)

#### A. Protect with Basic Authentication:
```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-app.rule=Host(`admin.yourdomain.com`)"
      - "traefik.http.routers.my-app.entrypoints=websecure"
      - "traefik.http.routers.my-app.tls=true"
      - "traefik.http.routers.my-app.middlewares=my-app-auth"
      # Generate hash via: htpasswd -nb user password (escape $ with $$ in compose)
      - "traefik.http.middlewares.my-app-auth.basicauth.users=admin:$$apr1$$xyz$$abcdefg"
      - "traefik.http.services.my-app.loadbalancer.server.port=80"
      - "traefik.docker.network=shared_net"
```

#### B. Strip Path Prefix (e.g. expose service at `/api`):
```yaml
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.my-app.rule=Host(`yourdomain.com`) && PathPrefix(`/api`)"
      - "traefik.http.routers.my-app.middlewares=strip-api"
      - "traefik.http.middlewares.strip-api.stripprefix.prefixes=/api"
      - "traefik.http.services.my-app.loadbalancer.server.port=8000"
      - "traefik.docker.network=shared_net"
```

---

## ⚙️ Log Rotation & Management

All containers are pre-configured with JSON log rotation (`max-size: 10m`, `max-file: 3`) to prevent host storage bloat.

To view logs:
```bash
docker compose logs -f traefik
```
