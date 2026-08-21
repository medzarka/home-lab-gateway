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
2. **Arcane Docker Cockpit:**
   - Modern SvelteKit & Go Docker management platform replacing legacy tools.
   - Full support for Compose stacks, image scanning, container terminals, and GitOps.
3. **Multi-Node Agent (`arcane-agent`):**
   - Connects remote nodes (`oci01-flex` and `orangepi5plus`) directly to the Arcane UI.

---

## 📁 Repository Structure

```
homelab-gateway/
├── docker-compose.yaml        # Traefik v3 + Arcane + Backward Compatibility Bridge
├── .env.example               # Documented environment variables template
├── dynamic/
│   ├── middleware.yaml        # HSTS, OWASP headers, rate limiting & /arcane shortcut
│   └── routes.yaml            # Cluster-wide reverse proxy routing rules
└── arcane-agent/
    └── docker-compose.yaml    # Remote agent template for worker nodes
```

---

## ⚙️ Deployment & Configuration

### 1. Environment Setup
Copy `.env.example` to `.env` and fill in your Cloudflare API Token and Arcane secrets:

```bash
cp .env.example .env
```

Generate 32-byte encryption keys:
```bash
openssl rand -base64 32 # Set as ARCANE_ENCRYPTION_KEY
openssl rand -base64 32 # Set as ARCANE_JWT_SECRET
```

### 2. Launch Gateway
```bash
docker compose up -d
```

### 3. Deploy Arcane Agents on Worker Nodes (VM2 & VM3)
In Arcane, navigate to **Environments -> Add Remote Host**, copy the generated token, and run on the remote node:

```bash
cd arcane-agent
ARCANE_AGENT_TOKEN="your_token_here" docker compose up -d
```
