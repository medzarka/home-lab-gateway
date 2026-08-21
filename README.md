# 🌐 Homelab Ingress Gateway & Arcane Management Cockpit

The **Homelab Gateway** serves as the central **Edge Ingress Controller** and **Cluster Management Cockpit** for the multi-node homelab infrastructure.

---

## 🏛️ Architecture

```
                                 [ Public Internet / Cloudflare ]
                                                │
                                    (Ports 80 / 443 / DNS ACME)
                                                ▼
                         ┌─────────────────────────────────────────────┐
                         │              VM 1: zap-vps                  │
                         │                                             │
                         │   ┌─────────────────────────────────────┐   │
                         │   │             TRAEFIK V3              │   │
                         │   │  * Wildcard SSL (*.bluewave.work)   │   │
                         │   │  * Automatic HTTPS Redirection      │   │
                         │   │  * Swarm & File Dynamic Discovery   │   │
                         │   └──────┬──────────────┬───────────────┘   │
                         │          │              │                   │
                         │          ▼              ▼                   │
                         │      [ ARCANE ]    [ Local Stacks ]         │
                         │      (:3552 GUI)   (Seafile/Immich)         │
                         └──────────┼──────────────┼───────────────────┘
                                    │              │
                   (Encrypted Swarm Mesh Overlay / Tailscale0)
                                    │              │
                    ┌───────────────┴────┐   ┌─────┴────────────────┐
                    │  VM 2: oci01-flex  │   │ VM 3: orangepi5plus  │
                    │                    │   │                      │
                    │  * LiteLLM AI      │   │  * Homelab Dashboard │
                    │  * Open-WebUI      │   │  * Media Stack       │
                    │  * Arcane Agent    │   │  * Arcane Agent      │
                    └────────────────────┘   └──────────────────────┘
```

---

## 🚀 Key Features

1. **Traefik v3.1 Ingress Engine:**
   - **Automated Wildcard SSL:** Resolves certificates for `*.bluewave.work` and `bluewave.work` using Cloudflare DNS ACME challenges.
   - **Zero Config Reloads:** Adding Docker labels to any container automatically provisions routes and SSL certs.
   - **Dynamic Routing:** Routes traffic to local stacks and remote Tailscale nodes across the cluster.
2. **Socket Security Hardening:**
   - Uses `tecnativa/docker-socket-proxy` to restrict Traefik to a read-only Docker socket interface.
3. **Arcane Docker Cockpit:**
   - Modern SvelteKit & Go Docker management platform replacing legacy tools.
   - Full support for Compose stacks, image scanning, container terminals, and GitOps.
4. **Multi-Node Agent (`arcane-agent`):**
   - Connects remote nodes (`oci01-flex` and `orangepi5plus`) directly to the Arcane UI.

---

## 🔒 Activating Wildcard SSL with Cloudflare DNS

Traefik automatically provisions and renews trusted **Let's Encrypt Wildcard SSL certificates** (`*.bluewave.work` and `bluewave.work`) via the Cloudflare DNS-01 ACME challenge.

### 1. Cloudflare API Token Prerequisites
In your [Cloudflare Dashboard](https://dash.cloudflare.com/profile/api-tokens), create a scoped API Token with:
- **Permissions:**
  - `Zone -> DNS -> Edit`
  - `Zone -> Zone -> Read`
- **Zone Resources:**
  - `Include -> Specific zone -> bluewave.work`

### 2. Configure `.env`
Add your token and email to `.env`:
```bash
CLOUDFLARE_API_TOKEN=your_generated_cloudflare_token_here
ACME_EMAIL=your_email@domain.com
ROOT_DOMAIN=bluewave.work
```

### 3. Set Up Cloudflare DNS Record
In Cloudflare DNS, add a single **Wildcard DNS record** pointing to your VPS public IP:
- **Type:** `A`
- **Name:** `*` *(or `@` for root)*
- **IPv4 Address:** `147.189.174.15`
- **Proxy Status:** Proxied (Orange Cloud) or DNS Only

### 4. How SSL is Applied to Services
In `dynamic/routes.yaml` (or via Docker container labels), declare `tls.certResolver: "cloudflare"`:

```yaml
# In dynamic/routes.yaml:
routers:
  arcane:
    rule: "Host(`arcane.bluewave.work`)"
    entryPoints: ["websecure"]
    service: "arcane-svc"
    tls:
      certResolver: "cloudflare"

# Or via Docker Compose container labels:
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.myapp.rule=Host(`myapp.bluewave.work`)"
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls.certresolver=cloudflare"
```

Certificates are securely stored in `/srv/data/gateway/volumes/traefik/acme.json` (`chmod 600`) and auto-renew 30 days before expiration.

---

## 👤 User & Password Management in Arcane

### 1. Default Admin Credentials
When Arcane is first started on a fresh database, it bootstraps a default global administrator:
- **Username:** `arcane`
- **Password:** `arcane-admin`

> [!IMPORTANT]
> Upon your first login at [`https://arcane.bluewave.work`](https://arcane.bluewave.work), change this default password immediately.

---

### 2. Adding & Updating Users via the Web UI
1. Log in to **[`https://arcane.bluewave.work`](https://arcane.bluewave.work)** as an administrator.
2. Navigate to **Settings ⚙️ -> Users**.
3. **To Add a New User:**
   - Click **+ Add User / Invite User**.
   - Enter their Username, Email, and initial Password.
   - Assign their role (**Global Administrator** or **Standard User**).
4. **To Change Your Password / Profile:**
   - Click on your avatar/username in the bottom-left corner -> **Account Settings**.
   - Enter your current password and set a new password.
   - Optionally register a **FIDO2 / WebAuthn Passkey (Touch ID, YubiKey, Windows Hello)** for passwordless 2FA login.

---

### 3. Resetting Admin Password via CLI (Emergency Recovery)
If you ever get locked out of Arcane, you can reset any global administrator's password from the host terminal:

1. **Verify `ALLOW_CLI_PASSWORD_RESET=true` is set:**
   In `/home/mgrsys/DATA/compose/arcane/.env`:
   ```bash
   ALLOW_CLI_PASSWORD_RESET=true
   ```

2. **Execute the Password Reset Command:**
   ```bash
   ssh mgrsys@zap-vps
   sudo docker exec -it arcane ./arcane admin reset-password --username arcane
   ```
   *(Enter and confirm your new password when prompted).*

3. **Reset MFA / Passkeys (If MFA device is lost):**
   ```bash
   sudo docker exec -it arcane ./arcane admin reset-mfa --username arcane
   ```

---

## 📁 Repository Structure

```
homelab-gateway/
├── docker-compose.yaml        # Traefik v3 + Arcane + Socket-Proxy
├── .env.example               # Documented environment variables template
├── dynamic/
│   ├── middleware.yaml        # HSTS, OWASP headers, rate limiting & /arcane shortcut
│   └── routes.yaml            # Cluster-wide reverse proxy routing rules
└── arcane-agent/
    └── docker-compose.yaml    # Remote agent template for worker nodes
```

---

## ⚙️ Deployment & Lifecycle Commands

### 1. Launch / Update Gateway
```bash
docker compose up -d
```

### 2. Check Service Logs
```bash
docker compose logs -f traefik
docker compose logs -f arcane
```

### 3. Deploy Arcane Agents on Remote Nodes (`oci01-flex` & `orangepi5plus`)
In Arcane, navigate to **Environments -> Add Remote Host**, copy the generated token, and run on the remote node:

```bash
cd arcane-agent
ARCANE_AGENT_TOKEN="your_token_here" docker compose up -d
```
