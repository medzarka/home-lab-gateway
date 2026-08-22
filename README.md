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

## 🔑 Authelia User & Password Management

All credentials are encrypted with **Argon2id** and stored in [`authelia/users_database.yml`](authelia/users_database.yml).

### Generate a New Password Hash:
```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'YourNewPassword'
```

### Update User in `authelia/users_database.yml`:
```yaml
users:
  admin:
    displayname: "Homelab Administrator"
    password: "$argon2id$v=19$m=65536,t=3,p=4$..." # Paste hash here
    email: "medzarka@gmail.com"
    groups:
      - admins
```
Authelia watches file changes and reloads credentials **instantly with zero downtime**.
