# 🛡️ Homelab Edge Gateway (Traefik v3 + Authelia SSO + Socket Proxy + Fail2Ban)

Enterprise-grade, automated edge reverse proxy and authentication gateway for the cluster.

---

## 🏛️ Ingress Architecture & Components

```
                                  [ Public Internet Ingress ]
                                               │
                              https://*.bluewave.work (Ports 80/443)
                                               │
                                               ▼
                              ┌──────────────────────────────────┐
                              │       TRAEFIK V3.1 (zap-vps)     │
                              │  - Wildcard TLS (Cloudflare DNS) │
                              │  - Automated Port 80 -> 443      │
                              │  - Fail2Ban Brute-Force Defense  │
                              └────────────────┬─────────────────┘
                                               │
                              Is User Authenticated on *.bluewave.work?
                                               │
                      ┌────────────────────────┴────────────────────────┐
                      │ (ForwardAuth)                                   │
                      ▼                                                 ▼
          ┌───────────────────────┐                         ┌───────────────────────┐
          │  AUTHELIA SSO (9091)  │                         │    BACKEND ROUTING    │
          │  - Visual Login Page  │                         │  - Homepage (3000)    │
          │  - 2FA TOTP / WebAuthn│                         │  - Dozzle Logs (8080) │
          │  - Argon2id Passwords │                         │  - Beszel Hub (8090)  │
          │  - Session Cookies    │                         │  - Remote Swarm Nodes │
          └───────────────────────┘                         └───────────────────────┘
```

---

## 💾 Standard Data & Storage Template

Persistent state is stored cleanly outside the Git repository in the standard homelab hierarchy:

```
/home/${SYSTEM_USER}/DATA/gateway/data/
├── traefik/
│   ├── acme.json              # Let's Encrypt Wildcard SSL certificate store (0600)
│   └── logs/                  # Traefik access & error logs
└── authelia/
    ├── db.sqlite3             # Authelia user 2FA and session database
    └── notification.txt       # Local identity verification codes
```

---

## 🚀 Deployment via Arcane GitOps

1. Open **Arcane Cockpit** at [`https://arcane.bluewave.work`](https://arcane.bluewave.work).
2. Click **Projects** $\rightarrow$ **New Project**.
3. Set:
   * **Name:** `gateway`
   * **Git Repository:** `https://github.com/medzarka/home-lab-gateway.git`
   * **Branch:** `main`
4. Add Environment Variables (from `.env.example`):
   ```env
   SYSTEM_USER=mgrsys
   DATA_DIR=/home/mgrsys/DATA
   CLOUDFLARE_API_TOKEN=your_token_here
   ACME_EMAIL=medzarka@gmail.com
   ROOT_DOMAIN=bluewave.work
   ```
5. Click **Deploy**.

---

## 🔑 Authelia User, Group & Access Control Management

All credentials are encrypted with **Argon2id** and stored in [`authelia/users_database.yml`](authelia/users_database.yml).

### 1. Adding a New User:

#### Step 1: Generate an Argon2id Password Hash
```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'YourNewPassword'
```

#### Step 2: Add User to [`authelia/users_database.yml`](authelia/users_database.yml)
```yaml
users:
  admin:
    displayname: "Homelab Administrator"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..." # Paste hash here
    email: "medzarka@gmail.com"
    groups:
      - admins
      - devops

  john:
    displayname: "John Doe"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..."
    email: "john@bluewave.work"
    groups:
      - users
      - family
```

---

### 2. Restricting Services by User or Group (ACLs)

In [`authelia/configuration.yml`](authelia/configuration.yml), you can create granular role-based rules under `access_control.rules`:

```yaml
access_control:
  default_policy: "deny"
  rules:
    # --- Public Bypass (APIs & Static Assets) ---
    - domain: "auth.bluewave.work"
      policy: "bypass"

    # --- Only 'admins' can access Traefik, Arcane, and Cluster Metrics (2FA) ---
    - domain:
        - "traefik.bluewave.work"
        - "metrics.bluewave.work"
        - "arcane.bluewave.work"
      subject:
        - "group:admins"
      policy: "two_factor"

    # --- Only 'devops' and 'admins' can access container logs ---
    - domain:
        - "logs.bluewave.work"
      subject:
        - "group:admins"
        - "group:devops"
      policy: "one_factor"

    # --- All logged-in users (admins, users, family) can access Homepage ---
    - domain:
        - "homelab.bluewave.work"
        - "hub.bluewave.work"
      subject:
        - "group:admins"
        - "group:users"
        - "group:family"
      policy: "one_factor"
```

> **⚡ Zero Downtime:** Authelia watches configuration and user database files automatically. Changes take effect **instantly without restarting containers**.

---

### 3. Custom Branding & Logo

Custom branding assets reside in `authelia/assets/`:
* `authelia/assets/logo.png` $\rightarrow$ Custom login card logo.
* `authelia/assets/favicon.ico` $\rightarrow$ Custom browser tab icon.

