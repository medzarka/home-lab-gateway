# 🌐 Homelab Ingress Gateway (Traefik v3 Edge Router)

The **Homelab Gateway** is the centralized **Edge Ingress Controller** deployed on the primary Cloud VPS (**`zap-vps`**). It manages SSL certificate provisioning, traffic termination, rate limiting, and secure reverse proxy routing across all cluster nodes.

---

## 🏛️ Architecture

```
                                 [ Public Internet ]
                                         │
                             https://*.bluewave.work
                                  (Ports 80/443)
                                         ▼
                 ┌───────────────────────────────────────────────┐
                 │                VM 1: zap-vps                  │
                 │                                               │
                 │   ┌───────────────────────────────────────┐   │
                 │   │         TRAEFIK V3 (Edge Ingress)     │   │
                 │   │  * Wildcard SSL (*.bluewave.work)     │   │
                 │   │  * Read-Only Docker Socket Proxy      │   │
                 │   │  * Security Headers & Rate Limiting   │   │
                 │   └───────┬───────────────────────┬───────┘   │
                 │           │                       │           │
                 │           ▼                       ▼           │
                 │       [ Arcane ]              [ Seafile ]     │
                 │       (:3552)                 (:80)           │
                 └───────────┼───────────────────────┼───────────┘
                             │                       │
                (Encrypted Mesh Overlay: homelab_swarm_net / Tailscale)
                             │                       │
              ┌──────────────┴────┐            ┌─────┴─────────────┐
              │  VM 2: oci01-flex │            │VM 3: orangepi5plus│
              │                   │            │                   │
              │  * LiteLLM AI     │            │* Homelab Dashboard│
              │  * Open-WebUI     │            │* Media Stack      │
              │  (No Traefik)     │            │(No Traefik)       │
              └───────────────────┘            └───────────────────┘
```

---

## 🚀 Key Features

1. **Traefik v3.1 Ingress Engine:**
   - **Automated Wildcard SSL:** Provisions and auto-renews Let's Encrypt wildcard certificates (`*.bluewave.work` and `bluewave.work`) via Cloudflare DNS-01 ACME challenges.
   - **Automatic HTTPS Redirection:** All port 80 HTTP requests are automatically upgraded to port 443 HTTPS.
   - **Docker Swarm & File Discovery:** Discovers services via Swarm container labels and file-based dynamic routes (`dynamic/`).
2. **Socket Security Hardening:**
   - Uses `tecnativa/docker-socket-proxy` to isolate the host Docker daemon with a read-only API (`tcp://socket-proxy:2375`).
3. **Fail2Ban In-Memory Defense (`tomMoulard/fail2ban`):**
   - Automatically bans malicious scanners and brute-force IPs after 5 failed requests while whitelisting the Tailscale mesh.
4. **GoAccess Real-Time Traffic & Bandwidth Dashboard:**
   - Visualizes live and historical data transfer (hourly, daily, monthly) and top subdomains at `https://traffic.bluewave.work`.
5. **OWASP Security Middleware:**
   - Enforces HSTS, XSS protection, anti-clickjacking, and brute-force rate-limiting (100 req/s).

---

## 📁 Repository Structure

```
homelab-gateway/
├── docker-compose.yaml        # Traefik v3 + Socket Proxy stack
├── .env.example               # Clean configuration template
└── dynamic/
    ├── middleware.yaml        # HSTS, OWASP headers, rate limiting & /arcane shortcut
    └── routes.yaml            # Dynamic routing to local & Tailscale mesh services
```

---

## 🔒 Activating Wildcard SSL with Cloudflare DNS

### 1. Cloudflare API Token Prerequisites
In your [Cloudflare Dashboard](https://dash.cloudflare.com/profile/api-tokens), create an API token with:
- `Zone -> DNS -> Edit`
- `Zone -> Zone -> Read`
- Target: Specific zone -> `bluewave.work`

### 2. Configure `.env`
```bash
cp .env.example .env
# Edit .env and enter your CLOUDFLARE_API_TOKEN and ACME_EMAIL
```

### 3. Set Up Cloudflare Wildcard DNS
In Cloudflare DNS, add an **`A`** record:
- **Name:** `*` *(and `@`)*
- **IPv4 Address:** `<YOUR_VPS_PUBLIC_IP>` (e.g. `147.189.174.15`)
- **Proxy Status:** Proxied (Orange Cloud) or DNS Only

---

## ⚙️ Deployment via Arcane / Docker Compose

You can deploy the Gateway stack directly in **Arcane**:
1. Open **Arcane** -> **Projects** -> **+ New Project**.
2. Name: `homelab-gateway`.
3. Environment: `Local Docker (zap-vps)`.
4. Connect this Git repository: `https://github.com/medzarka/home-lab-gateway.git`.
5. Enter your `.env` variables and click **Deploy**.

Or via CLI on `zap-vps`:
```bash
docker compose up -d
```
